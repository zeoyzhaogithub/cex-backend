# 功能清单与实现顺序

本文梳理 go-cex **当前已实现的功能**，并给出**复刻时应遵循的实现顺序**（由底向上、由同步到异步）。

---

## 一、当前功能总览

### 1. 基础设施层

| 功能 | 说明 | 关键代码 |
|------|------|----------|
| 多环境配置 | YAML 驱动：env / database / redis / amqp / workers | `config/` |
| 主备双库 | 主库写热数据，backup 库存 K 线、订单/成交备份等 | `utils/gorm.go` |
| Redis 连接池 | 深度、Ticker、K 线缓存 | `utils/redis.go` |
| RabbitMQ 连接 | 交换机、队列、reload 队列 | `initializers/rabbitmq.go`、`config/amqp.go` |
| 国际化 | 错误码中英文 | `initializers/i18n.go`、`initializers/locales/` |
| 启动缓存 | 市场、币种等热数据加载到内存 | `initializers/cacheData.go` |
| 统一 API 响应 | Success / Error / Array 分页结构 | `utils/response.go` |

### 2. 用户与账户

| 功能 | HTTP | 说明 |
|------|------|------|
| 登录 | `POST /api/:platform/v1/users/login` | 通过 Identity 表（email 等）鉴权，签发 Token |
| 当前用户 | `GET /api/:platform/v1/users/me` | 需登录 |
| 账户列表 | `GET /api/:platform/v1/users/accounts` | 按可见币种返回余额 |
| 单币种账户 | `GET /api/:platform/v1/users/accounts/:currency` | 懒创建账户（首次访问时创建） |

相关模型：`User`、`Identity`、`Account`、`Token`、`ApiToken`、`Device`

### 3. 市场与币种（公开）

| 功能 | HTTP | 说明 |
|------|------|------|
| 币种列表 | `GET /api/:platform/v1/currencies` | 可见币种 |
| 市场列表 | `GET /api/:platform/v1/markets` | 含 Ticker 快照等 |
| 深度 | `GET /api/:platform/v1/depth?market=` | 从 Redis 读买卖盘 |
| Ticker 全市场 | `GET /api/:platform/v1/tickers` | 24h 行情摘要 |
| 单市场 Ticker | `GET /api/:platform/v1/tickers/:market` | |
| K 线 | `GET /api/:platform/v1/k` | 多周期 K 线 |
| 图表 | `GET /api/:platform/v1/chart` | 图表数据 |
| 公开成交 | `GET /api/:platform/v1/trades` | 市场最近成交 |

### 4. 订单与交易（需登录）

| 功能 | HTTP | 说明 |
|------|------|------|
| 下单 | `POST /api/:platform/v1/orders?side=buy\|sell` | 限价单；DB 事务 + 冻结 + 投递 MQ |
| 查单笔订单 | `GET /api/:platform/v1/order?id=` | |
| 订单列表 | `GET /api/:platform/v1/orders?market=&state=` | 分页，state: wait/done/cancel |
| 撤单 | `POST /api/:platform/v1/order/delete` | 向撮合发 cancel 消息 |
| 批量撤单 | `POST /api/:platform/v1/orders/clear` | 清空某市场挂单 |
| 我的成交 | `GET /api/:platform/v1/trades/my` | 用户成交历史 |

订单类型：当前为**限价单**（`order_type: limit`），买卖方向通过 `OrderBid` / `OrderAsk` 区分。

### 5. 撮合与结算（后台进程，无 HTTP）

| 功能 | 进程 | 说明 |
|------|------|------|
| 订单撮合 | matching | 内存订单簿，价格优先时间优先 |
| 深度维护 | matching | 撮合后 rebuild depth → Redis |
| 成交事件 | matching → MQ | 推送至 trade treat 队列 |
| 撤单事件 | matching → MQ | 推送至 order cancel 队列 |
| 成交结算 | treat | 更新订单状态、账户余额、写入 Trade |
| 撤单回滚 | cancel | 释放 locked、更新订单为 cancel |
| 历史挂单恢复 | matching 启动时 | state=100 的订单 replay 进引擎 |

撮合引擎：`third_party/matching`（Engine / OrderBook / PriceLevel）

### 6. 异步 Worker

| Worker | 队列路由 | 职责 |
|--------|----------|------|
| KLineWorker | `goDCE.k` | 成交驱动 K 线聚合 |
| TickerWorker | `goDCE.ticker` | 24h Ticker 计算 |
| RebuildKLineToRedisWorker | 配置驱动 | K 线重建到 Redis |
| AccountVersionCheckPointWorker | 配置驱动 | 账户版本一致性检查 |

配置：`config/workers.yml`，注册：`initializers/initializeWorkers.go`

### 7. 定时任务（schedule 进程）

| 任务 | Cron | 说明 |
|------|------|------|
| BackupLogFiles | 23:55 每日 | 日志备份 |
| UploadLogFileToS3 | 23:56 | 上传 S3（需配置） |
| CleanLogs | 23:59 | 清理日志 |
| CleanTokens | 23:57 | 清理过期 Token |
| CreateLatestKLine | 每 5 秒 | DB K 线补算 |
| WaitingOrderCheck | 每 20 秒 | 待成交订单检查 |

### 8. 运维与部署

| 功能 | 说明 |
|------|------|
| build.sh / start.sh / stop.sh | 本地多进程管理 |
| Docker Compose | 一键部署全栈 |
| seed_demo_data | 演示数据注入 |
| reset_demo_state | 恢复演示初始状态 |

### 9. 未包含 / 弱化的能力

以下在完整 CEX 中常见，但 **go-cex 未实现或仅留扩展点**：

- 链上钱包充值 / 提币
- 永续合约、期权、杠杆
- WebSocket 推送（前端需自行轮询或另建 WS 服务）
- 管理后台、KYC、风控规则引擎
- 分布式多节点生产级部署文档

---

## 二、交易主链路（实现核心）

理解实现顺序前，先掌握一条完整链路：

```text
用户 POST /orders
  → API: 校验市场/价格/数量/余额
  → DB 事务: 插入 orders + 冻结 accounts.locked
  → RabbitMQ: 发布到 market.MatchingExchange
  → matching: 消费 → Engine.Submit → 撮合
       ├─ 成交 → TradeTreatExchange → treat: 更新订单/账户/写 trades → 触发 K/Ticker MQ
       └─ 撤单 → OrderCancelExchange → cancel: 解冻 + 订单 state=cancel
  → Redis: 深度 / Ticker / K 线更新
  → API: GET depth / tickers / trades 可读
```

---

## 三、推荐实现顺序（复刻路线图）

按「依赖关系 + 可验证性」排序，每完成一阶段都应有可运行的验证点。

### 第 1 层：工程与数据底座（1～2 周）

| 顺序 | 模块 | 产出 |
|------|------|------|
| 1.1 | `go mod` + 目录骨架 | 空项目可 `go build` |
| 1.2 | `config/` + `utils/gorm.go` | 连接 MySQL，主备双库 |
| 1.3 | `models/common.go` AutoMigrate | 所有表结构就绪 |
| 1.4 | `utils/redis.go` | Redis 连通 |
| 1.5 | `config/amqp` + `initializers/rabbitmq` | RabbitMQ 连通 |

**验证**：写单元测试或 main 函数 ping 三个中间件。

### 第 2 层：领域模型与静态 API（1 周）

| 顺序 | 模块 | 产出 |
|------|------|------|
| 2.1 | `models`: Currency, Market | 币种与市场 CRUD 模型 |
| 2.2 | 种子脚本 | btc/usdt/btcusdt |
| 2.3 | `api/v1/currency.go`, `market.go` | 公开列表接口 |
| 2.4 | `routes/v1.go` + `utils/response` | 路由与统一 JSON |
| 2.5 | `initializers/cacheData` | 市场内存缓存 |

**验证**：`curl /markets`、`/currencies` 返回种子数据。

### 第 3 层：用户与账务（1 周）

| 顺序 | 模块 | 产出 |
|------|------|------|
| 3.1 | User, Identity, Account, Token | 用户体系模型 |
| 3.2 | `initializers/auth.go` | Token 中间件 |
| 3.3 | `api/v1/user.go` | 登录、查账户 |
| 3.4 | Account 冻结/解冻 helper | 为下单做准备 |

**验证**：登录拿到 Token，查询账户余额。

### 第 4 层：下单（同步部分）（1～2 周）

| 顺序 | 模块 | 产出 |
|------|------|------|
| 4.1 | `models/order.go` | 订单状态机 WAIT/DONE/CANCEL |
| 4.2 | `api/v1/order.go` V1PostOrders | 校验 + 写库 + 冻结 |
| 4.3 | 订单查询、列表 API | GET order(s) |
| 4.4 | MQ 发布 `pushMessageToMatching` | 消息进 matching 队列 |

**验证**：下单后 DB 有 WAIT 订单，余额 locked 增加；MQ 可见消息（matching 尚未消费也可）。

### 第 5 层：撮合引擎（2 周，难点）

| 顺序 | 模块 | 产出 |
|------|------|------|
| 5.1 | `third_party/matching` 或自研引擎 | Submit / 价格档位 / 成交输出 |
| 5.2 | `trade/matching/base.go` InitAssignments | 按 node 加载市场 |
| 5.3 | 消费 MQ + doMatching | 撮合逻辑 |
| 5.4 | 启动时 replay 未完成订单 | 进程重启可恢复 |
| 5.5 | `trade/matching/depth.go` | Redis 深度 |

**验证**：启动 matching，双用户挂单可撮合；Redis 深度有变化。

### 第 6 层：成交与撤单（1～2 周）

| 顺序 | 模块 | 产出 |
|------|------|------|
| 6.1 | `trade/treat/` | 消费成交、改账户、写 Trade |
| 6.2 | `order/cancel/` | 消费撤单、解冻 |
| 6.3 | API 撤单 / 批量撤单 | 用户侧撤单入口 |
| 6.4 | `models/trade.go` + `api/v1/trade.go` | 成交查询 |

**验证**：撮合后订单 done、余额正确；撤单后 locked 释放。

### 第 7 层：行情（1～2 周）

| 顺序 | 模块 | 产出 |
|------|------|------|
| 7.1 | `models/k.go`, `ticker.go` | K 线 / Ticker 模型 |
| 7.2 | `workers/sneakerWorkers/kLineWorker` | 成交驱动聚合 |
| 7.3 | `tickerWorker` | 24h 统计 |
| 7.4 | `api/v1/kLine.go`, `ticker.go`, `orderBook.go` | 读 Redis/DB |
| 7.5 | `initializers/latestKLine.go`, `ticker.go` | 启动预热 |

**验证**：`/tickers`、`/k`、`/depth` 有实时数据。

### 第 8 层：可靠性与运维（按需）

| 顺序 | 模块 | 产出 |
|------|------|------|
| 8.1 | `schedules/` | 补算、巡检 |
| 8.2 | AccountVersion 审计 | 账务版本追溯 |
| 8.3 | Docker + 健康检查 | 生产式部署 |
| 8.4 | 日志、S3 备份 | 运维脚本 |

---

## 四、API 路由速查表

`:platform` 常见取值：`web`、`c2e`（路由设计允许多端复用同一套 handler）。

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/api/:platform/v1/currencies` | 否 | 币种 |
| GET | `/api/:platform/v1/markets` | 否 | 市场 |
| GET | `/api/:platform/v1/depth` | 否 | 深度 |
| GET | `/api/:platform/v1/tickers` | 否 | 全部 Ticker |
| GET | `/api/:platform/v1/tickers/:market` | 否 | 单市场 Ticker |
| GET | `/api/:platform/v1/k` | 否 | K 线 |
| GET | `/api/:platform/v1/chart` | 否 | 图表 |
| GET | `/api/:platform/v1/trades` | 否 | 公开成交 |
| POST | `/api/:platform/v1/users/login` | 否 | 登录 |
| GET | `/api/:platform/v1/users/me` | 是 | 当前用户 |
| GET | `/api/:platform/v1/users/accounts` | 是 | 账户列表 |
| GET | `/api/:platform/v1/users/accounts/:currency` | 是 | 单账户 |
| POST | `/api/:platform/v1/orders` | 是 | 下单 |
| GET | `/api/:platform/v1/order` | 是 | 单笔订单 |
| GET | `/api/:platform/v1/orders` | 是 | 订单列表 |
| POST | `/api/:platform/v1/order/delete` | 是 | 撤单 |
| POST | `/api/:platform/v1/orders/clear` | 是 | 批量撤单 |
| GET | `/api/:platform/v1/trades/my` | 是 | 我的成交 |

---

## 五、学习阅读顺序（读懂原仓库）

若目标是读懂而非重写，推荐按 README 顺序：

1. `api/api.go` + `routes/v1.go` — 入口
2. `api/v1/order.go` — 下单与 MQ
3. `trade/matching.go` + `trade/matching/base.go` — 撮合
4. `trade/treat.go` + `order/cancel.go` — 结算闭环
5. `workers/` + `schedules/` — 行情与后台
