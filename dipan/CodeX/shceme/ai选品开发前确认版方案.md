# ai选品开发前确认版方案

## 1. 当前版本定位

`ai选品` 第一阶段只解决一个核心问题：

> 自动爬取 Shopee 和 TikTok Shop 的公开商品数据，并落库到 PostgreSQL。

这一阶段暂不做复杂 AI 推荐，不接 LLM，不做向量检索，不做趋势分析。先保证公开商品数据可以被自动采集、入库、更新和查询。

## 2. 已确认范围

### 2.1 平台范围

第一版同时支持：

- Shopee
- TikTok Shop

### 2.2 国家范围

第一版覆盖东南亚 6 个国家：

- ID：印尼
- TH：泰国
- VN：越南
- MY：马来西亚
- PH：菲律宾
- SG：新加坡

### 2.3 采集入口

第一版支持四类采集入口：

- 关键词搜索页
- 类目页
- 榜单页
- 商品详情页

### 2.4 采集入口配置方式

采集入口放入数据库表 `crawl_sources` 维护。

不采用写死配置文件的方式，方便后续新增国家、关键词、类目和目标 URL。

### 2.5 商品唯一标识

商品去重优先使用：

```text
platform + country_code + product_url
```

如果后续能从页面中解析到平台商品 ID，则作为补充字段保存，不作为第一版强依赖。

### 2.6 数据更新策略

同一个商品重复采集时：

- 不新增重复记录。
- 直接覆盖最新价格、销量、评分、评论数、描述等字段。
- 更新 `last_crawled_at` 和 `updated_at`。

第一版不做商品历史快照表。

### 2.7 触发方式

第一版支持两种触发方式：

- 手动触发
- 每天定时触发一次

### 2.8 采集技术

第一版优先使用：

- Go HTTP 请求
- HTML / JSON 解析

如果遇到强 JS 渲染、Go HTTP 无法稳定获取数据，再引入 Playwright。

### 2.9 并发和限流

第一版采用保守并发策略：

- 全局 worker 并发数：3
- 单个平台并发数：2
- 单个平台 + 国家维度并发数：1
- 单个 worker 请求间隔：建议 1 到 3 秒随机等待

这样可以降低公开页面访问失败、限流和临时封禁风险。

### 2.10 失败处理

失败任务最多重试 3 次。

当 `retry_count < max_retry_count` 时，任务可以重新进入 `pending` 状态。

当重试次数达到 3 次仍失败：

- 状态置为 `failed`
- 记录 `error_message`
- 不再自动重试

## 3. 核心流程

```text
手动触发 / 每日定时触发
        |
        v
读取 crawl_sources
        |
        v
生成 crawl_jobs
        |
        v
Go worker 抢任务
        |
        v
执行公开页面采集
        |
        v
解析商品字段
        |
        v
Upsert 到 crawled_products
        |
        v
更新 crawl_jobs 状态
```

## 4. 数据库表结构

## 4.1 采集入口表 crawl_sources

`crawl_sources` 用于维护要采集的平台、国家、关键词、类目页、榜单页和详情页入口。

```sql
CREATE TABLE crawl_sources (
    id BIGSERIAL PRIMARY KEY,
    platform VARCHAR(32) NOT NULL,
    country_code VARCHAR(8) NOT NULL,
    source_type VARCHAR(32) NOT NULL,
    keyword VARCHAR(255),
    target_url TEXT,
    enabled BOOLEAN NOT NULL DEFAULT true,
    remark TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

字段说明：

- `platform`：平台，例如 `shopee`、`tiktok_shop`。
- `country_code`：国家，例如 `ID`、`TH`、`VN`、`MY`、`PH`、`SG`。
- `source_type`：入口类型，例如 `search`、`category`、`ranking`、`detail`。
- `keyword`：搜索关键词。`source_type = search` 时使用。
- `target_url`：目标 URL。类目页、榜单页、详情页优先使用。
- `enabled`：是否启用。
- `remark`：备注。

建议索引：

```sql
CREATE INDEX idx_crawl_sources_enabled
ON crawl_sources (enabled);

CREATE INDEX idx_crawl_sources_platform_country
ON crawl_sources (platform, country_code);
```

## 4.2 采集任务表 crawl_jobs

`crawl_jobs` 用于记录每一次实际采集任务。

```sql
CREATE TABLE crawl_jobs (
    id BIGSERIAL PRIMARY KEY,
    source_id BIGINT REFERENCES crawl_sources(id),
    platform VARCHAR(32) NOT NULL,
    country_code VARCHAR(8) NOT NULL,
    crawl_type VARCHAR(32) NOT NULL,
    target_url TEXT,
    keyword VARCHAR(255),
    status VARCHAR(32) NOT NULL DEFAULT 'pending',
    retry_count INT NOT NULL DEFAULT 0,
    max_retry_count INT NOT NULL DEFAULT 3,
    locked_by VARCHAR(128),
    locked_at TIMESTAMPTZ,
    error_message TEXT,
    started_at TIMESTAMPTZ,
    finished_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

任务状态：

- `pending`
- `running`
- `success`
- `failed`
- `cancelled`

建议索引：

```sql
CREATE INDEX idx_crawl_jobs_status_created_at
ON crawl_jobs (status, created_at);

CREATE INDEX idx_crawl_jobs_platform_country
ON crawl_jobs (platform, country_code);

CREATE INDEX idx_crawl_jobs_source_id
ON crawl_jobs (source_id);
```

## 4.3 采集商品表 crawled_products

`crawled_products` 用于存储公开页面采集到的商品数据。

```sql
CREATE TABLE crawled_products (
    id BIGSERIAL PRIMARY KEY,
    platform VARCHAR(32) NOT NULL,
    country_code VARCHAR(8) NOT NULL,
    product_id VARCHAR(128),
    sku VARCHAR(128),
    title TEXT,
    description TEXT,
    price NUMERIC(18, 4),
    currency VARCHAR(8),
    sales_volume BIGINT,
    product_url TEXT NOT NULL,
    image_url TEXT,
    shop_name VARCHAR(255),
    rating NUMERIC(6, 3),
    review_count BIGINT,
    raw_data JSONB,
    first_crawled_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_crawled_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

建议唯一索引：

```sql
CREATE UNIQUE INDEX uniq_crawled_products_platform_country_url
ON crawled_products (platform, country_code, product_url);
```

建议查询索引：

```sql
CREATE INDEX idx_crawled_products_platform_country
ON crawled_products (platform, country_code);

CREATE INDEX idx_crawled_products_sales_volume
ON crawled_products (sales_volume);

CREATE INDEX idx_crawled_products_last_crawled_at
ON crawled_products (last_crawled_at);

CREATE INDEX idx_crawled_products_title
ON crawled_products USING gin (to_tsvector('simple', coalesce(title, '')));
```

## 5. Upsert 策略

商品写入时以 `platform + country_code + product_url` 判断是否重复。

如果不存在，则插入。

如果已存在，则覆盖最新字段。

```sql
INSERT INTO crawled_products (
    platform,
    country_code,
    product_id,
    sku,
    title,
    description,
    price,
    currency,
    sales_volume,
    product_url,
    image_url,
    shop_name,
    rating,
    review_count,
    raw_data,
    last_crawled_at,
    updated_at
)
VALUES (...)
ON CONFLICT (platform, country_code, product_url)
DO UPDATE SET
    product_id = EXCLUDED.product_id,
    sku = EXCLUDED.sku,
    title = EXCLUDED.title,
    description = EXCLUDED.description,
    price = EXCLUDED.price,
    currency = EXCLUDED.currency,
    sales_volume = EXCLUDED.sales_volume,
    image_url = EXCLUDED.image_url,
    shop_name = EXCLUDED.shop_name,
    rating = EXCLUDED.rating,
    review_count = EXCLUDED.review_count,
    raw_data = EXCLUDED.raw_data,
    last_crawled_at = now(),
    updated_at = now();
```

## 6. 任务生成策略

每日定时触发时：

1. 查询 `crawl_sources` 中 `enabled = true` 的记录。
2. 为每条 source 生成一条 `crawl_jobs`。
3. 初始状态为 `pending`。

手动触发时：

1. 支持指定单个 `source_id`。
2. 支持指定平台和国家批量触发。
3. 支持全量触发所有启用 source。

## 7. Worker 执行策略

Worker 通过 PostgreSQL 抢占任务。

建议使用：

```sql
SELECT id
FROM crawl_jobs
WHERE status = 'pending'
ORDER BY created_at ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

抢到任务后：

1. 更新状态为 `running`。
2. 写入 `locked_by`、`locked_at`、`started_at`。
3. 执行采集。
4. 成功则更新为 `success`。
5. 失败则根据重试次数决定重新 pending 或最终 failed。

## 8. 失败重试策略

任务失败时：

```text
如果 retry_count + 1 < max_retry_count:
    retry_count + 1
    status = pending
    error_message = 当前错误

否则:
    retry_count + 1
    status = failed
    error_message = 当前错误
    finished_at = now()
```

默认：

```text
max_retry_count = 3
```

## 9. 并发和限流策略

第一版推荐配置：

```text
global_worker_count = 3
platform_worker_limit = 2
platform_country_worker_limit = 1
request_delay_min_seconds = 1
request_delay_max_seconds = 3
```

含义：

- 整个服务最多同时跑 3 个采集任务。
- 同一个平台最多同时跑 2 个任务。
- 同一个平台 + 国家最多同时跑 1 个任务。
- 每次请求之间随机等待 1 到 3 秒。

这个配置偏保守，适合第一版验证。

## 10. Go 模块建议

```text
/internal/ai_selection
  product_handler.go
  product_service.go

/internal/crawler
  scheduler.go
  source_repository.go
  job_repository.go
  product_repository.go
  worker.go
  limiter.go
  parser.go

/internal/crawler/platforms
  shopee.go
  tiktok_shop.go
```

核心接口：

```go
type ProductCrawler interface {
    Platform() string
    Crawl(ctx context.Context, job CrawlJob) ([]CrawledProduct, error)
}
```

## 11. 基础商品查询接口

第一版只做基础查询接口，不做 AI 推荐。

```text
GET /api/ai-selection/products
```

支持参数：

- `platform`
- `country_code`
- `keyword`
- `min_price`
- `max_price`
- `min_sales_volume`
- `limit`
- `offset`

示例：

```text
GET /api/ai-selection/products?platform=shopee&country_code=ID&keyword=storage&min_sales_volume=1000&limit=10
```

## 12. 字段获取口径

| 字段 | 第一版口径 |
|---|---|
| 商品图 | 采集公开页面主图 URL |
| 商品名称 | 采集公开页面标题 |
| 商品 SKU | 优先平台商品 ID；没有则为空 |
| 商品描述 | 能从详情页拿到则保存 |
| 商品售价 | 保存公开展示价 |
| 商品销量 | 保存公开展示销量，不承诺后台真实销量 |
| 商品链接 | 保存公开商品 URL |

## 13. 第一版验收标准

第一版完成后需要满足：

- 可以在 `crawl_sources` 中配置采集入口。
- 可以手动触发采集。
- 可以每天自动触发采集。
- 可以生成 `crawl_jobs`。
- worker 可以自动执行 pending 任务。
- 失败任务最多重试 3 次。
- 重试 3 次仍失败后记录为 `failed`。
- 商品数据可以写入 `crawled_products`。
- 同一商品重复采集时覆盖最新价格和销量。
- 可以通过接口按平台、国家、关键词查询商品。

## 14. 后续扩展

等第一阶段稳定后，再考虑：

- 商品历史快照表。
- 第三方数据 API。
- LLM 查询和推荐。
- 向量检索。
- 趋势分析。
- 商品推荐评分。
- 图片对象存储。
- 管理后台配置采集源。

## 15. 当前结论

当前开发前确认版采用：

```text
Shopee + TikTok Shop
+ ID / TH / VN / MY / PH / SG
+ 关键词搜索页 / 类目页 / 榜单页 / 商品详情页
+ crawl_sources 管理采集入口
+ crawl_jobs 管理任务
+ crawled_products 存商品
+ Go HTTP 优先
+ 必要时再接 Playwright
+ 手动触发 + 每天一次
+ 保守并发和限流
+ 失败最多重试 3 次
```

这一版是面向最快开发落地的公开数据采集和落库方案。
