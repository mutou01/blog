# ai选品轻量 MVP 方案

## 1. 方案定位

这一版方案只解决 `ai选品` 的第一阶段核心能力：

> 自动爬取 Shopee / TikTok Shop 公开商品数据，并落库到 PostgreSQL。

第一阶段不追求完整推荐系统，也不引入复杂数据分析能力。目标是先把商品数据稳定采集进来，形成可查询、可持续更新的市场商品池。

## 2. 第一阶段目标

第一阶段只做以下能力：

- 自动爬取公开商品数据。
- 每天定时执行一次。
- 支持 Shopee 和 TikTok Shop。
- 支持东南亚国家维度，例如 ID、TH、VN、MY、PH、SG。
- 落库商品图、商品名称、商品 SKU 或平台商品 ID、商品描述、商品售价、商品销量、商品链接。
- 记录采集任务状态，便于排查失败原因。
- 提供基础商品查询接口。

## 3. 第一阶段不做的事情

为了最快落地，第一阶段不做：

- 不接入 LLM。
- 不做向量检索。
- 不做 pgvector。
- 不做复杂推荐评分。
- 不做趋势分析。
- 不做商品每日指标拆表。
- 不做对象存储。
- 不接第三方数据 API。
- 不做达人、视频、直播关联分析。
- 不做自动刊登和自动下单。

## 4. 技术栈

结合现有系统，采用最简技术栈：

- 后端：现有 Golang 项目。
- 数据库：PostgreSQL。
- 调度：Go 内置定时任务。
- 任务队列：PostgreSQL 任务表。
- 采集执行：Go worker。
- 页面采集：优先使用 Go HTTP 请求和 HTML/JSON 解析；如遇强 JS 渲染页面，再考虑单独接入 Playwright。

第一版不引入 Airflow、Temporal、Kafka、RabbitMQ、Elasticsearch、独立向量数据库。

## 5. 核心流程

```text
Go 定时任务
   |
   v
生成 crawl_jobs
   |
   v
Go worker 抢任务
   |
   v
请求公开页面
   |
   v
解析商品字段
   |
   v
写入 crawled_products
   |
   v
更新任务状态
```

## 6. 数据采集范围

### 6.1 平台

- Shopee
- TikTok Shop

### 6.2 国家

第一版建议支持：

- ID：印尼
- TH：泰国
- VN：越南
- MY：马来西亚
- PH：菲律宾
- SG：新加坡

### 6.3 采集入口

第一版可以先从以下入口采集：

- 搜索关键词页。
- 类目页。
- 榜单页，如果公开可访问。
- 商品详情页。

关键词和目标 URL 可以先写在配置文件中，后续再做后台管理。

## 7. 数据库表结构

第一版只需要两张核心表，外加一张可选配置表。

## 7.1 采集任务表 crawl_jobs

用于记录每一次采集任务。

```sql
CREATE TABLE crawl_jobs (
    id BIGSERIAL PRIMARY KEY,
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

字段说明：

- `platform`：平台，例如 `shopee`、`tiktok_shop`。
- `country_code`：国家，例如 `ID`、`TH`。
- `crawl_type`：采集类型，例如 `search`、`category`、`detail`。
- `target_url`：要采集的页面地址。
- `keyword`：搜索关键词。
- `status`：任务状态。
- `retry_count`：当前重试次数。
- `max_retry_count`：最大重试次数。
- `locked_by`：执行任务的 worker 标识。
- `locked_at`：任务被锁定时间。
- `error_message`：失败原因。

任务状态建议：

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
```

## 7.2 采集商品表 crawled_products

用于存储从公开页面采集到的商品数据。

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

字段说明：

- `product_id`：平台商品 ID，能拿到则填写。
- `sku`：第一版不保证是真实卖家 SKU。如果公开数据没有 SKU，可以使用平台商品 ID 或变体 ID 作为替代。
- `title`：商品名称。
- `description`：商品描述。
- `price`：商品售价。
- `currency`：币种。
- `sales_volume`：公开页面展示销量或可采集销量。
- `product_url`：商品链接。
- `image_url`：商品主图链接。
- `raw_data`：保留原始采集数据，便于后续排查和补字段。

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
```

## 7.3 可选配置表 crawl_sources

如果第一版希望通过数据库维护采集入口，可以加这张表。如果想更快落地，也可以先把采集入口写在配置文件中。

```sql
CREATE TABLE crawl_sources (
    id BIGSERIAL PRIMARY KEY,
    platform VARCHAR(32) NOT NULL,
    country_code VARCHAR(8) NOT NULL,
    source_type VARCHAR(32) NOT NULL,
    keyword VARCHAR(255),
    target_url TEXT,
    enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 8. 任务执行方式

### 8.1 每日生成任务

每天定时根据配置生成采集任务。

例如：

```text
shopee + ID + phone case
shopee + TH + home storage
tiktok_shop + VN + beauty
tiktok_shop + MY + kitchen tools
```

### 8.2 Worker 抢任务

多个 worker 可以并发处理任务。

建议使用 PostgreSQL 的 `FOR UPDATE SKIP LOCKED`：

```sql
SELECT id
FROM crawl_jobs
WHERE status = 'pending'
ORDER BY created_at ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

抢到任务后，将状态更新为 `running`。

### 8.3 写入商品数据

商品写入时使用 `ON CONFLICT` 做更新：

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

## 9. Go 模块建议

```text
/internal/ai_selection
  product_handler.go
  product_service.go

/internal/crawler
  scheduler.go
  worker.go
  job_repository.go
  product_repository.go
  parser.go

/internal/crawler/platforms
  shopee.go
  tiktok_shop.go
```

第一版接口可以保持简单：

```go
type ProductCrawler interface {
    Platform() string
    Crawl(ctx context.Context, job CrawlJob) ([]CrawledProduct, error)
}
```

## 10. 基础查询接口

第一版先提供商品查询接口，不做复杂 AI 推荐。

```text
GET /api/ai-selection/products
```

支持查询参数：

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

返回：

```json
{
  "items": [
    {
      "platform": "shopee",
      "country_code": "ID",
      "product_id": "123456",
      "sku": "123456",
      "title": "Storage Box",
      "price": 3.99,
      "currency": "IDR",
      "sales_volume": 1200,
      "product_url": "https://example.com/product",
      "image_url": "https://example.com/image.jpg",
      "shop_name": "Example Shop",
      "rating": 4.8,
      "review_count": 320,
      "last_crawled_at": "2026-05-19T00:00:00Z"
    }
  ],
  "total": 1
}
```

## 11. 字段获取可行性

| 字段 | 可行性 | 说明 |
|---|---:|---|
| 商品图 | 高 | 公开页面通常可见 |
| 商品名称 | 高 | 基础字段 |
| 商品 SKU | 中低 | 公开数据通常没有真实卖家 SKU，可先使用平台商品 ID |
| 商品描述 | 中 | 详情页通常可获取，但部分页面可能折叠或异步加载 |
| 商品售价 | 高 | 公开页面通常可见 |
| 商品销量 | 中 | 取决于平台展示口径，不保证是真实后台销量 |
| 商品链接 | 高 | 可稳定获取 |

## 12. 风险和取舍

### 12.1 公开页面变化

页面结构变化会导致解析失败。

应对：

- 保留 `raw_data`。
- 记录任务失败原因。
- 定期检查失败率。

### 12.2 销量口径不统一

公开销量不一定是真实后台销量。

应对：

- 第一版只记录能采集到的公开销量。
- 后续如果接入第三方 API，再补充更准确的销量或 GMV。

### 12.3 SKU 不稳定

公开页面不一定包含真实 SKU。

应对：

- 使用平台商品 ID 作为替代 SKU。
- 后续接入授权店铺数据后再补充真实 SKU。

### 12.4 反爬和访问限制

即使只采公开数据，也可能遇到限流或访问失败。

应对：

- 控制采集频率。
- 设置重试次数。
- 避免高并发请求。
- 不绕过验证码、登录和风控。

## 13. 最小落地顺序

建议按以下顺序实现：

1. 建 `crawl_jobs` 表。
2. 建 `crawled_products` 表。
3. 写一个固定关键词和国家的任务生成器。
4. 写 Shopee 或 TikTok Shop 的第一个公开页采集器。
5. 写商品 upsert 入库。
6. 写 worker 状态流转。
7. 写基础商品查询接口。
8. 再扩展更多国家、关键词和平台。

## 14. 第一版验收标准

第一版完成时，应满足：

- 可以每天自动生成采集任务。
- 可以自动执行采集任务。
- 可以把商品基础字段写入 PostgreSQL。
- 同一商品重复采集时会更新，而不是重复插入。
- 采集失败可以看到失败原因。
- 可以通过接口按平台、国家、关键词查询商品。

## 15. 当前推荐结论

当前阶段建议采用轻量 MVP：

```text
Golang 后端
+ PostgreSQL
+ Go 定时任务
+ crawl_jobs
+ crawled_products
+ Go worker
+ Shopee / TikTok Shop 公开数据采集
+ 基础商品查询接口
```

这版最贴近当前核心诉求：自动获取公开商品数据并落库。后续等商品数据稳定后，再逐步加入 LLM、推荐评分、趋势分析和第三方数据 API。
