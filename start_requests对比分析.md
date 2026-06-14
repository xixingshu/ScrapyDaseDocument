# Scrapy vs scrapy-redis `start_requests` 对比分析

## 1. Scrapy 的 `start_requests`

### 源码路径

`scrapy/spiders/__init__.py` (spider_base 环境，Scrapy 2.13+)

### 完整源码

```python
class Spider(object_ref):
    name: str
    custom_settings: dict | None = None
    start_urls: list[str]

    def __init__(self, name=None, **kwargs):
        if name is not None:
            self.name = name
        elif not getattr(self, "name", None):
            raise ValueError(f"{type(self).__name__} must have a name")
        self.__dict__.update(kwargs)
        if not hasattr(self, "start_urls"):
            self.start_urls = []

    # 新版入口（2.13+），取代 start_requests
    async def start(self) -> AsyncIterator[Any]:
        with warnings.catch_warnings():
            warnings.filterwarnings(
                "ignore", category=ScrapyDeprecationWarning, module=r"^scrapy\.spiders$"
            )
            for item_or_request in self.start_requests():
                yield item_or_request

    # 旧版入口（已废弃，但仍有兼容）
    def start_requests(self) -> Iterable[Any]:
        warnings.warn(
            "The Spider.start_requests() method is deprecated, use "
            "Spider.start() instead.",
            ScrapyDeprecationWarning,
            stacklevel=2,
        )
        if not self.start_urls and hasattr(self, "start_url"):
            raise AttributeError(
                "Crawling could not start: 'start_urls' not found "
                "or empty (but found 'start_url' attribute instead, "
                "did you miss an 's'?)"
            )
        for url in self.start_urls:
            yield Request(url, dont_filter=True)
```

### 执行流程

```
start_urls → start_requests() → yield Request → 引擎 → 下载器 → parse()
```

### 特点

- 从 `start_urls` 列表读取 URL（代码中硬编码）
- yield 的 Request 禁止去重（`dont_filter=True`）
- 请求队列在**内存**中，爬虫停止后丢失
- 不支持分布式
- 不支持断点续爬

### `start_requests` 弃用说明

Scrapy 2.13+ 引入 `async def start()` 替代 `start_requests()`，目的是统一 Spider 入口的异步化。

**弃用原因：**
1. 让 Spider 入口支持 `async`，与 Scrapy 内部架构对齐（回调、中间件已全部 async）
2. 允许 `start()` 直接 yield Item（dict），旧版只能 yield Request

#### 新旧对比

| 对比 | 旧版 `start_requests()` | 新版 `async start()` |
|---|---|---|
| 类型 | 同步生成器 `def` | 异步生成器 `async def` |
| 返回值 | 只能 yield `Request` | 可 yield `Request` **或** Item（dict） |
| 默认实现 | 直接遍历 `start_urls` | 默认委托给 `start_requests()` 做兼容 |

#### 新版 `start()` 示例

```python
# yield Item（旧版做不到）
async def start(self):
    yield {"foo": "bar"}

# async 原生支持协程
async def start(self):
    data = await fetch_something_async()
    yield Request(data["url"])
```

#### 新版默认实现（与旧版完全等价）

```python
async def start(self):
    with warnings.catch_warnings():
        warnings.filterwarnings("ignore", category=ScrapyDeprecationWarning, ...)
        for item_or_request in self.start_requests():
            yield item_or_request
```

实际功能完全一致，只是包了一层兼容层。对大多数用户无影响 —— 继续用 `start_requests` 或直接覆盖 `start()` 都行。

---

## 2. scrapy-redis 的 `start_requests`

### 源码路径

`scrapy_redis/spiders.py`

### `RedisMixin.start_requests()`

```python
class RedisMixin:
    redis_key = None        # 默认: "<spider.name>:start_urls"
    redis_batch_size = None # 默认: CONCURRENT_REQUESTS
    redis_encoding = None   # 默认: utf-8
    server = None           # Redis 客户端
    max_idle_time = None    # 空闲多久后关闭（0=永不关闭）

    def start_requests(self):
        """Returns a batch of start requests from redis."""
        return self.next_requests()
```

### `next_requests()` — 从 Redis 取数据并生成 Request

```python
def next_requests(self):
    found = 0
    datas = self.fetch_data(self.redis_key, self.redis_batch_size)
    for data in datas:
        reqs = self.make_request_from_data(data)
        if isinstance(reqs, Iterable):
            for req in reqs:
                yield req
                found += 1
        elif reqs:
            yield reqs
            found += 1
        else:
            self.logger.debug(f"Request not made from data: {data}")
    if found:
        self.logger.debug(f"Read {found} requests from '{self.redis_key}'")
```

### `make_request_from_data()` — 将 Redis 数据转为 Request

```python
def make_request_from_data(self, data):
    formatted_data = bytes_to_str(data, self.redis_encoding)

    if is_dict(formatted_data):                 # 支持 JSON 格式
        parameter = json.loads(formatted_data)
    else:                                       # 纯字符串格式（已废弃）
        return FormRequest(formatted_data, dont_filter=True)

    if parameter.get("url", None) is None:
        return []                               # 无 url 则跳过

    url = parameter.pop("url")
    method = parameter.pop("method").upper() if "method" in parameter else "GET"
    metadata = parameter.pop("meta") if "meta" in parameter else {}

    return FormRequest(url, dont_filter=True, method=method, formdata=parameter, meta=metadata)
```

#### Redis 中支持的 JSON 格式示例

```json
{
    "url": "https://example.com",
    "meta": {"job-id": "123xsd", "start-date": "dd/mm/yy"},
    "method": "POST",
    "formdata_key": "formdata_value"
}
```

### `setup_redis()` — 初始化 Redis 连接

```python
def setup_redis(self, crawler=None):
    if self.server is not None:
        return

    crawler = crawler or getattr(self, "crawler", None)
    settings = crawler.settings

    # 确定 Redis key
    if self.redis_key is None:
        self.redis_key = settings.get("REDIS_START_URLS_KEY", defaults.START_URLS_KEY)
    self.redis_key = self.redis_key % {"name": self.name}

    # 确定批处理大小
    if self.redis_batch_size is None:
        self.redis_batch_size = settings.getint("CONCURRENT_REQUESTS", defaults.REDIS_CONCURRENT_REQUESTS)

    # 确定编码
    if self.redis_encoding is None:
        self.redis_encoding = settings.get("REDIS_ENCODING", defaults.REDIS_ENCODING)

    # 连接 Redis
    self.server = connection.from_settings(crawler.settings)

    # 根据配置选则数据读取方式
    if settings.getbool("REDIS_START_URLS_AS_SET", defaults.START_URLS_AS_SET):
        self.fetch_data = self.server.spop           # SET → 随机弹出一个
        self.count_size = self.server.scard
    elif settings.getbool("REDIS_START_URLS_AS_ZSET", defaults.START_URLS_AS_ZSET):
        self.fetch_data = self.pop_priority_queue     # ZSET → 按优先级弹
        self.count_size = self.server.zcard
    else:
        self.fetch_data = self.pop_list_queue         # LIST → 从左出队（默认）
        self.count_size = self.server.llen

    # 确定空闲等待时间
    if self.max_idle_time is None:
        self.max_idle_time = settings.get("MAX_IDLE_TIME_BEFORE_CLOSE", defaults.MAX_IDLE_TIME)

    # 注册 idle 信号 → 闲时继续从 Redis 拉取
    crawler.signals.connect(self.spider_idle, signal=signals.spider_idle)
```

### `spider_idle()` — 闲时从 Redis 补充请求

```python
def spider_idle(self):
    if self.server is not None and self.count_size(self.redis_key) > 0:
        self.spider_idle_start_time = int(time.time())   # 还有任务，重置空闲计时

    self.schedule_next_requests()

    idle_time = int(time.time()) - self.spider_idle_start_time
    if self.max_idle_time != 0 and idle_time >= self.max_idle_time:
        return                                          # 超时，允许关闭
    raise DontCloseSpider                               # 否则阻止关闭
```

### `schedule_next_requests()` — 将请求交给引擎

```python
def schedule_next_requests(self):
    for req in self.next_requests():
        if scrapy_version >= (2, 6):
            self.crawler.engine.crawl(req)
        else:
            self.crawler.engine.crawl(req, spider=self)
```

### 队列读取方式

```python
def pop_list_queue(self, redis_key, batch_size):
    with self.server.pipeline() as pipe:
        pipe.lrange(redis_key, 0, batch_size - 1)
        pipe.ltrim(redis_key, batch_size, -1)
        datas, _ = pipe.execute()
    return datas

def pop_priority_queue(self, redis_key, batch_size):
    with self.server.pipeline() as pipe:
        pipe.zrevrange(redis_key, 0, batch_size - 1)
        pipe.zremrangebyrank(redis_key, -batch_size, -1)
        datas, _ = pipe.execute()
    return datas
```

### `RedisSpider` — 最终类

```python
class RedisSpider(RedisMixin, Spider):
    @classmethod
    def from_crawler(cls, crawler, *args, **kwargs):
        obj = super().from_crawler(crawler, *args, **kwargs)
        obj.setup_redis(crawler)
        return obj


class RedisCrawlSpider(RedisMixin, CrawlSpider):
    @classmethod
    def from_crawler(cls, crawler, *args, **kwargs):
        obj = super().from_crawler(crawler, *args, **kwargs)
        obj.setup_redis(crawler)
        return obj
```

---

## 3. 完整对比表

| 对比维度 | Scrapy `start_requests` | scrapy-redis `start_requests` |
|---|---|---|
| **数据来源** | `start_urls` 列表（代码硬编码） | Redis 队列（运行时动态注入） |
| **生成方式** | 遍历列表 yield `Request(url, dont_filter=True)` | `next_requests()` → `fetch_data()` → `make_request_from_data()` |
| **请求去重** | 内存中的指纹集合（`RFPDupeFilter`） | Redis set 去重（`scrapy_redis.dupefilter.RFPDupeFilter`） |
| **请求队列** | 内存队列 | Redis List / Set / ZSet（`FifoQueue` / `PriorityQueue` / `LifoQueue`） |
| **调度器** | 内置 `scrapy.core.scheduler.Scheduler` | `scrapy_redis.scheduler.Scheduler` |
| **断点续爬** | ❌ 不支持（关闭即丢） | ✅ 支持（`SCHEDULER_PERSIST=True` 保留队列） |
| **分布式** | ❌ 不支持 | ✅ 支持（多实例共享同一 Redis 队列） |
| **闲时策略** | 队列空 → 结束 spider | `spider_idle` 持续轮询 Redis，有数据再补充，否则 `DontCloseSpider` 保持存活 |
| **数据格式** | 只支持 URL 字符串 | 支持 JSON（含 `url`, `method`, `meta`, `formdata`）+ 纯字符串（废弃） |
| **入口方法** | `start_requests()`(旧) / `async start()`(新, 2.13+) | 保持 `start_requests()`，实际委托 `next_requests()` |
| **启动方式** | 直接 `scrapy crawl spider_name` | 需手动 `redis-cli lpush spider_name:start_urls "..."` |

---

## 4. scrapy-redis 核心组件一览

| 文件 | 类/函数 | 作用 |
|---|---|---|
| `spiders.py` | `RedisMixin` | 混入类，提供 `start_requests`、`next_requests`、`spider_idle` 等 |
| `spiders.py` | `RedisSpider` | 最终爬虫基类（`RedisMixin + Spider`） |
| `spiders.py` | `RedisCrawlSpider` | 最终爬虫基类（`RedisMixin + CrawlSpider`） |
| `scheduler.py` | `Scheduler` | Redis 调度器，管理请求的入队、出队、去重 |
| `queue.py` | `FifoQueue` | FIFO 队列（List 实现） |
| `queue.py` | `PriorityQueue` | 优先级队列（ZSet 实现） |
| `queue.py` | `LifoQueue` | LIFO 队列（List 实现） |
| `dupefilter.py` | `RFPDupeFilter` | Redis 去重过滤器 |
| `connection.py` | `get_redis_from_settings` | 根据设置创建 Redis 客户端 |
| `defaults.py` | — | 所有默认配置常量 |
| `pipelines.py` | `RedisPipeline` | 将 Item 推入 Redis 的管道 |
| `stats.py` | `RedisStatsCollector` | 基于 Redis 的统计收集器 |

## 5. 常用配置项

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `REDIS_URL` | `None` | Redis 连接 URL（如 `redis://127.0.0.1:6379`） |
| `REDIS_HOST` | `localhost` | Redis 主机 |
| `REDIS_PORT` | `6379` | Redis 端口 |
| `REDIS_DB` | `0` | Redis 数据库 |
| `REDIS_PARAMS` | `{}` | Redis 额外参数 |
| `SCHEDULER_PERSIST` | `False` | 是否持久化队列（关闭时不清空 Redis） |
| `SCHEDULER_FLUSH_ON_START` | `False` | 启动时是否清空 Redis 队列 |
| `SCHEDULER_QUEUE_CLASS` | `scrapy_redis.queue.PriorityQueue` | 队列类型 |
| `REDIS_START_URLS_KEY` | `%(name)s:start_urls` | 起始 URL 的 Redis key |
| `REDIS_START_URLS_AS_SET` | `False` | 是否用 SET 类型存储起始 URL |
| `REDIS_START_URLS_AS_ZSET` | `False` | 是否用 ZSET 类型存储起始 URL |
| `CONCURRENT_REQUESTS` | `16` | 每批从 Redis 拉取的数量 |
| `DUPEFILTER_CLASS` | `scrapy_redis.dupefilter.RFPDupeFilter` | 去重过滤器 |
| `MAX_IDLE_TIME_BEFORE_CLOSE` | `0` | 空闲多久后关闭（0=永不关闭） |

---

## 6. 扩展：通过 `make_request_from_data` 支持 GET/PUT/DELETE

scrapy-redis 默认的 `make_request_from_data` 统一使用 `FormRequest` 构造请求，但 `FormRequest` 的 `formdata` 参数对 GET/DELETE 无意义。只需在自己的 spider 中 **覆盖** 该方法即可按需支持所有 HTTP 动词：

### 重写示例

```python
import json
from scrapy import FormRequest, Request
from scrapy_redis.spiders import RedisSpider
from scrapy_redis.utils import bytes_to_str, is_dict


class MySpider(RedisSpider):
    name = "myspider"
    redis_key = "myspider:start_urls"

    def make_request_from_data(self, data):
        formatted_data = bytes_to_str(data, self.redis_encoding)

        parameter = json.loads(formatted_data)
        url = parameter.pop("url", None)
        if url is None:
            return []

        method = parameter.pop("method", "GET").upper()
        meta = parameter.pop("meta", {})

        if method in ("GET", "DELETE"):
            return Request(url, dont_filter=True, method=method, meta=meta)
        elif method in ("POST", "PUT", "PATCH"):
            return FormRequest(
                url, dont_filter=True, method=method,
                formdata=parameter, meta=meta
            )
        else:
            return Request(url, dont_filter=True, method=method, meta=meta)
```

### Redis 中推入的 JSON 示例

```bash
# GET
redis-cli lpush myspider:start_urls '{"url": "https://api.example.com/resource/1", "method": "GET", "meta": {"id": 1}}'

# PUT
redis-cli lpush myspider:start_urls '{"url": "https://api.example.com/resource/1", "method": "PUT", "name": "new_name", "meta": {"id": 1}}'

# DELETE
redis-cli lpush myspider:start_urls '{"url": "https://api.example.com/resource/1", "method": "DELETE", "meta": {"id": 1}}'
```

### 原理说明

| 方法 | 使用的类 | body 传参方式 |
|---|---|---|
| GET / DELETE | `scrapy.Request(method=...)` | 无 body |
| POST / PUT / PATCH | `scrapy.FormRequest(formdata=...)` | 自动编码为表单 |

`scrapy.Request` 本身支持 `method` 参数，`FormRequest` 是它的子类，额外增加了 `formdata`。重写 `make_request_from_data` 按方法分派即可干净支持所有 HTTP 动词。无需修改 scrapy-redis 源码。 |
