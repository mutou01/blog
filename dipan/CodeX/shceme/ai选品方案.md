# ai选品方案

## 1. 功能定位

`ai选品` 是跨境 ERP 系统中的市场商品推荐能力。第一版聚焦东南亚市场，从 Shopee 和 TikTok Shop 获取公开市场商品和爆品数据，经过清洗、入库、趋势分析和语义检索后，根据用户提示词返回最匹配的 10 个商品。

第一版目标不是构建真正意义上的全平台全量商品库，而是构建一个可持续维护的市场商品池：

- 覆盖 Shopee 和 TikTok Shop。
- 覆盖东南亚主要站点：ID、TH、VN、MY、PH、SG。
- 每日更新商品价格、销量、评论、排名等快照。
- 支持按国家、平台、类目、价格、销量趋势等条件查询。
- 接入 LLM，对候选商品做语义匹配、重排和推荐理由生成。

## 2. 核心结论

当前方案可以实现商品数据获取，但需要明确边界：

- 可以稳定获取商品图、商品名称、商品链接、售价、部分描述、榜单排名。
- SKU 不一定能稳定获取。公开市场数据通常没有卖家后台 SKU，第一版应以平台商品 ID、变体 ID 作为 SKU 替代口径。
- 销量可以获取展示销量、估算销量、榜单排名、GMV 或趋势指标，但不应承诺等同于卖家后台真实销量。
- TikTok Shop 爆品数据建议通过第三方数据 API 或授权导出增强。
- Shopee 市场数据可以先通过公开页采集实现，但需要接受页面变化、字段缺失和维护成本。

因此，第一版应定义为：

> 基于公开数据采集和授权第三方数据源，构建东南亚 Shopee + TikTok Shop 市场商品池，并提供 ai选品 推荐能力。

## 3. 数据边界

### 3.1 允许的数据来源

- 平台公开页面中的公开商品数据。
- 第三方数据服务商提供的官方 API、OpenAPI、企业数据包或导出文件。
- 已购买账号权限内允许导出的数据。
- 后续可接入自有店铺订单和商品数据，用于验证推荐效果。

### 3.2 不纳入系统能力的方式

- 绕过登录、验证码、风控或反爬机制。
- 绕过第三方 SaaS 的权限或付费限制。
- 未授权调用第三方内部接口。
- 批量模拟真人账号访问。

## 4. 是否需要第三方 API

技术上不一定必须接入第三方 API，但业务上强烈建议预留并优先接入。

### 4.1 不接第三方 API 的能力

只使用公开页采集时，可以实现：

- 商品列表采集。
- 商品详情采集。
- 价格、图片、链接、标题、部分销量和评论采集。
- 每日快照。
- 基础 ai选品 查询。

但限制明显：

- 爆品榜和增长趋势不稳定。
- 销量口径可能缺失或不统一。
- TikTok Shop 的达人、视频、直播带货关联数据难以完整获取。
- Shopee 页面结构变化会带来持续维护成本。

### 4.2 接入第三方 API 的价值

第三方数据 API 或数据导出可以增强：

- 爆品榜单。
- 销量趋势。
- GMV 估算。
- 达人带货关系。
- 视频和直播关联。
- 类目热度。
- 增长率。
- 竞争度分析。

建议第一版不绑定具体供应商，而是先设计统一的 `DataSourceAdapter` 接口，再逐步接入 FastMoss、EchoTik、Kalodata 等供应商。

## 5. 推荐数据源策略

### 5.1 TikTok Shop

优先级最高。建议采用：

- FastMoss API / OpenAPI。
- EchoTik API / 导出。
- Kalodata API / 导出。
- TikTok Shop 公开页采集作为补充。

TikTok Shop 的爆品发现高度依赖趋势、达人、视频和直播数据，第三方数据源价值较高。

### 5.2 Shopee

建议采用：

- Shopee 公开搜索页、类目页、商品详情页采集。
- 第三方 Shopee 数据服务商补充。
- Shopee 官方 Open Platform 作为后续授权店铺数据补充。

Shopee 官方 API 更适合授权店铺经营数据，不适合作为全站市场爆品池的唯一来源。

### 5.3 Amazon

第一版暂不作为主范围。可以保留适配器扩展口：

- 卖家精灵。
- Helium 10。
- Jungle Scout。
- Amazon SP-API。

后续如果要扩展欧美市场，再纳入 Amazon 数据。

## 6. 总体架构

```text
数据源层
Shopee公开页 / TikTok公开页 / FastMoss / EchoTik / Kalodata / 其他供应商
        |
        v
采集调度层
按国家、平台、类目、关键词、榜单每日采集
        |
        v
数据清洗层
字段标准化、去重、币种统一、销量口径标记、图片入对象存储
        |
        v
商品情报库
商品主表 + 每日指标快照 + 原始记录 + 向量索引
        |
        v
ai选品推荐层
意图解析 + 混合检索 + 规则打分 + LLM重排
        |
        v
ERP前端/API
返回10个商品、推荐理由、风险提示、数据来源
```

## 7. 技术栈建议

结合现有系统：

- 后端：现有 Golang 项目。
- 数据库：PostgreSQL。
- 调度：Go 内置 cron。
- 任务队列：PostgreSQL 任务表。
- 并发执行：Go worker pool。
- 页面采集：优先 Go HTTP 采集；必要时外挂 Playwright 服务。
- 向量检索：PostgreSQL + pgvector。
- 图片存储：MinIO、S3 或 OSS。
- LLM：通过统一 LLM Gateway 调用，避免业务代码绑定具体模型供应商。

第一版不建议引入：

- Airflow。
- Temporal。
- Kafka。
- RabbitMQ。
- Elasticsearch。
- 独立向量数据库。
- 数据湖。

## 8. 调度方案

第一版采用最简任务调度：

```text
Go Cron
   |
   v
生成 crawl_jobs
   |
   v
Worker 使用 PostgreSQL 抢任务
   |
   v
执行采集、清洗、入库
   |
   v
更新任务状态
```

任务抢占建议使用 PostgreSQL：

```sql
FOR UPDATE SKIP LOCKED
```

这样多个 worker 可以并发执行，不会重复处理同一任务。

任务状态：

- pending
- running
- success
- failed
- cancelled

## 9. 采集任务类型

### 9.1 榜单采集任务

按平台、国家、类目抓取热销榜、趋势榜、新品榜。

### 9.2 搜索采集任务

按关键词抓取搜索结果，例如：

- beauty
- phone case
- home storage
- pet supplies
- kitchen tools

### 9.3 商品详情任务

对榜单和搜索结果中的商品链接进入详情页，补齐：

- 商品图。
- 商品名称。
- 商品描述。
- 售价。
- 销量。
- 店铺。
- 评分。
- 评论数。
- 商品链接。

### 9.4 指标快照任务

每天记录：

- 售价。
- 销量。
- 评论数。
- 评分。
- 榜单排名。
- 类目。
- 数据来源。

用于计算趋势和增长率。

## 10. 数据模型

### 10.1 商品主表 market_products

```sql
CREATE TABLE market_products (
    id BIGSERIAL PRIMARY KEY,
    platform VARCHAR(32) NOT NULL,
    country_code VARCHAR(8) NOT NULL,
    platform_product_id VARCHAR(128) NOT NULL,
    platform_variant_id VARCHAR(128),
    normalized_sku VARCHAR(256),
    title TEXT NOT NULL,
    description TEXT,
    category_path TEXT,
    brand VARCHAR(256),
    shop_id VARCHAR(128),
    shop_name VARCHAR(256),
    product_url TEXT NOT NULL,
    main_image_url TEXT,
    main_image_object_key TEXT,
    currency VARCHAR(8),
    latest_price NUMERIC(18, 4),
    latest_sales_volume BIGINT,
    latest_rating NUMERIC(6, 3),
    latest_review_count BIGINT,
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status VARCHAR(32) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

说明：

- `normalized_sku` 第一版不承诺是真实 seller SKU。
- 优先使用平台商品 ID + 变体 ID 形成稳定唯一标识。
- 建议额外创建唯一索引：`CREATE UNIQUE INDEX uniq_market_products_platform_item_variant ON market_products (platform, country_code, platform_product_id, COALESCE(platform_variant_id, ''));`

### 10.2 每日指标表 market_product_daily_metrics

```sql
CREATE TABLE market_product_daily_metrics (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES market_products(id),
    stat_date DATE NOT NULL,
    price NUMERIC(18, 4),
    sales_volume BIGINT,
    sales_amount NUMERIC(18, 4),
    rating NUMERIC(6, 3),
    review_count BIGINT,
    ranking_position INT,
    ranking_type VARCHAR(64),
    category_path TEXT,
    metric_source VARCHAR(128),
    metric_definition TEXT,
    confidence_score NUMERIC(5, 4),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (product_id, stat_date, ranking_type, metric_source)
);
```

### 10.3 原始数据表 market_product_source_records

```sql
CREATE TABLE market_product_source_records (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT REFERENCES market_products(id),
    source_type VARCHAR(64) NOT NULL,
    source_name VARCHAR(128) NOT NULL,
    source_url TEXT,
    raw_payload_json JSONB NOT NULL,
    collected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    confidence_score NUMERIC(5, 4)
);
```

### 10.4 任务表 crawl_jobs

```sql
CREATE TABLE crawl_jobs (
    id BIGSERIAL PRIMARY KEY,
    source_type VARCHAR(64) NOT NULL,
    source_name VARCHAR(128),
    platform VARCHAR(32) NOT NULL,
    country_code VARCHAR(8) NOT NULL,
    job_type VARCHAR(64) NOT NULL,
    payload_json JSONB NOT NULL,
    status VARCHAR(32) NOT NULL DEFAULT 'pending',
    retry_count INT NOT NULL DEFAULT 0,
    max_retry_count INT NOT NULL DEFAULT 3,
    locked_by VARCHAR(128),
    locked_at TIMESTAMPTZ,
    started_at TIMESTAMPTZ,
    finished_at TIMESTAMPTZ,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 10.5 向量表 market_product_embeddings

```sql
CREATE TABLE market_product_embeddings (
    product_id BIGINT PRIMARY KEY REFERENCES market_products(id),
    embedding_model VARCHAR(128) NOT NULL,
    embedding_text TEXT NOT NULL,
    embedding_vector vector,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

如果第一版暂不启用 pgvector，可以先保留表结构，先用关键词检索和规则排序。

## 11. Go 模块建议

```text
/internal/ai_selection
  handler.go
  service.go
  prompt_parser.go
  retriever.go
  scorer.go
  reranker.go

/internal/market_data
  scheduler.go
  worker.go
  job_repo.go
  product_repo.go
  source_adapter.go

/internal/market_data/sources
  shopee_public.go
  tiktok_public.go
  fastmoss.go
  echotik.go
  kalodata.go
```

核心接口：

```go
type DataSourceAdapter interface {
    Name() string
    Fetch(ctx context.Context, job CrawlJob) ([]MarketProductDTO, error)
}
```

## 12. ai选品推荐流程

```text
用户提示词
   |
   v
意图解析
   |
   v
结构化过滤
   |
   v
关键词检索 + 向量检索
   |
   v
规则评分
   |
   v
LLM 重排
   |
   v
返回 10 个商品
```

示例请求：

```json
{
  "prompt": "帮我找印尼 TikTok Shop 女性家居收纳爆品，价格低一点，最近销量增长快",
  "country_code": "ID",
  "platform": "tiktok_shop",
  "limit": 10
}
```

示例返回：

```json
{
  "products": [
    {
      "title": "Foldable Storage Box",
      "platform": "tiktok_shop",
      "country_code": "ID",
      "price": 3.99,
      "currency": "USD",
      "sales_volume": 12800,
      "growth_rate_7d": 0.34,
      "product_url": "https://example.com/product",
      "image_url": "https://example.com/image.jpg",
      "reason": "近7天销量增长明显，价格带适合印尼低客单测品，家居收纳类目竞争中等。",
      "risk": "同款较多，需要关注差异化图片和组合套装。"
    }
  ]
}
```

## 13. 推荐评分

第一版可以采用可解释规则评分：

```text
final_score =
  0.30 * trend_score
+ 0.25 * sales_score
+ 0.15 * price_fit_score
+ 0.15 * competition_score
+ 0.10 * data_confidence_score
+ 0.05 * freshness_score
```

评分维度：

- `trend_score`：近 7 天或 30 天销量、排名、评论增长。
- `sales_score`：销量或 GMV 规模。
- `price_fit_score`：是否符合用户价格偏好。
- `competition_score`：同类商品数量、评论数、店铺集中度。
- `data_confidence_score`：数据来源可靠度。
- `freshness_score`：数据采集时间是否足够新。

## 14. MVP 范围

第一版建议只做：

- Shopee + TikTok Shop。
- 东南亚 6 个国家。
- 每日采集一次。
- 商品基础字段入库。
- 每日指标快照。
- ai选品 查询接口。
- 返回 10 个商品。
- 每个商品返回推荐理由、风险提示和数据来源。

暂不做：

- 自动刊登。
- 自动下单。
- 广告投放建议。
- 供应链匹配。
- 利润精算。
- Amazon 深度选品。

## 15. 主要风险

### 15.1 数据来源风险

公开页面可能变化，采集器需要维护。

缓解：

- 做字段完整率监控。
- 做采集失败率监控。
- 保存原始记录。
- 为每个平台保留多个数据源。

### 15.2 销量口径风险

公开销量、估算销量和真实销量不是同一概念。

缓解：

- 保存 `metric_source`。
- 保存 `metric_definition`。
- 推荐结果中展示数据来源和置信度。

### 15.3 SKU 缺失风险

市场公开数据不一定包含真实 seller SKU。

缓解：

- 第一版将 SKU 定义为 `platform_product_id + platform_variant_id`。
- 如果后续接入授权店铺 API，再补充真实 seller SKU。

### 15.4 LLM 推荐风险

LLM 不能替代真实数据排序。

缓解：

- LLM 只做意图理解、语义重排和解释生成。
- 候选召回和规则评分由数据库与后端服务完成。

## 16. 推荐实施阶段

### 阶段一：跑通商品池

- 建表。
- 做 crawl_jobs。
- 做 Go cron。
- 做 worker。
- 接一个 Shopee 公开页采集器。
- 接一个 TikTok Shop 数据源或公开页采集器。

### 阶段二：跑通 ai选品 查询

- 商品关键词检索。
- 国家、平台、类目、价格过滤。
- 简单规则评分。
- 返回 Top 10。

### 阶段三：增强爆品能力

- 接入 FastMoss、EchoTik 或 Kalodata。
- 增加趋势指标。
- 增加 GMV、达人、视频、直播关联。
- 引入 pgvector。
- 引入 LLM 重排。

### 阶段四：ERP 闭环

- 接入自有店铺销量。
- 接入成本和利润。
- 评估推荐商品后续转化。
- 优化评分模型。

## 17. 当前建议

第一版最快落地方案：

```text
Golang 后端
+ PostgreSQL
+ Go cron
+ PostgreSQL crawl_jobs
+ Go worker pool
+ Shopee 公开页采集
+ TikTok 第三方 API/导出优先，公开页采集补充
+ 关键词检索和规则评分
+ LLM 推荐理由生成
```

该方案能够实现商品数据获取，并能支撑 ai选品 MVP。但需要在产品说明中明确：

- 商品池是按平台、国家、类目、关键词和榜单构建的市场样本池。
- 销量、GMV、趋势指标必须展示数据来源和口径。
- SKU 在市场公开数据中不保证存在。
- 第三方 API 不是第一天必须接，但应作为正式版本的重要增强。
