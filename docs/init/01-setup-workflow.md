# 搭建整体流程

本文描述从零复刻 / 本地运行 go-cex 的完整步骤，按依赖关系排序。

---

## 阶段 0：理解目标架构

在写代码前先明确：

1. **多进程拆分**：API 只负责接入与校验；撮合、结算、撤单各自独立，通过 MQ 通信。
2. **两阶段下单**：先 DB 事务内写订单 + 冻结资金，再投递 MQ，避免消息先于账务落地。
3. **按市场分配节点**：`markets` 表中的 `matching_node` / `trade_treat_node` / `order_cancel_node` 决定哪个进程消费哪个市场。
4. **Redis 承担热读**：深度、Ticker、K 线供 API 快速查询；MySQL 是账务真相源。

---

## 阶段 1：初始化 Go 工程

### 1.1 创建模块与目录骨架

**若你已经在 GitHub / 本地创建好仓库并 clone 下来，可跳过下面两行**，直接进入该目录，确认根目录已有 `go.mod` 即可。

```bash
# 仅「从零、本地还没有项目目录」时需要：
mkdir cex-backend && cd cex-backend
go mod init github.com/yourname/cex-backend   # 生成 go.mod，声明 Go 模块名
```

| 命令 | 含义 |
|------|------|
| `mkdir cex-backend` | 新建项目文件夹（名字可自定，与仓库名一致即可） |
| `cd cex-backend` | 进入该文件夹 |
| `go mod init github.com/yourname/cex-backend` | 初始化 Go **模块**（类似 Node 的 `npm init`），生成 `go.mod`，后续 `go get` / `go build` 都依赖它 |

> **不是安装框架。** `go mod init` 是 Go 自带的模块管理命令，不引入 Echo、Gorm 等；那些要在后面用 `go get` 或写 `go.mod` 的 `require` 再添加。本项目的 Web 框架是 Echo，ORM 是 Gorm，属于业务依赖，不是这一步完成的。

建议先按原项目划分顶层目录（详见 [03-directory-structure.md](./03-directory-structure.md)）：

```text
cex-backend/
├── api/              # API 入口 + v1 handlers
├── config/           # YAML 配置 + 加载代码
├── initializers/     # 启动初始化（DB/Redis/MQ/Auth/i18n）
├── models/           # Gorm 模型 + AutoMigrate
├── routes/           # 路由注册
├── trade/            # 撮合 + 成交进程
├── order/            # 撤单进程
├── workers/          # 异步 Worker
├── schedules/        # 定时任务（可选）
├── utils/            # DB/Redis/响应封装
├── third_party/      # 撮合引擎等 vendored 依赖
├── scripts/          # 种子数据、运维脚本
├── docker/           # 容器化配置
├── build.sh
├── start.sh
└── stop.sh
```

### 1.2 引入核心依赖

参考 `go.mod`：

- `github.com/labstack/echo` — HTTP
- `github.com/jinzhu/gorm` + `github.com/go-sql-driver/mysql` — 数据库
- `github.com/gomodule/redigo` — Redis
- `github.com/streadway/amqp` — RabbitMQ
- `github.com/shopspring/decimal` — 金额精度
- `github.com/oldfritter/sneaker-go/v3` — Worker 框架
- `github.com/robfig/cron` — 定时任务
- 撮合引擎：可 vendoring 到 `third_party/matching`，并在 `go.mod` 中 `replace`

---

## 阶段 2：基础设施

### 2.1 启动依赖服务

**方式 A：Docker Compose（推荐）**

```bash
cd go-cex
docker compose up --build -d
docker compose ps
```

默认拉起：MySQL、Redis、RabbitMQ、api、matching、cancel、treat、workers。

**方式 B：本地安装**

需自行安装并启动：

- MySQL 8（创建 `goDCE` 与 `goDCE_backup` 库）
- Redis 7
- RabbitMQ 3.13（建议开 management 插件）

### 2.2 准备配置文件

从 example 复制并修改：

```bash
cp config/env.yml.example config/env.yml
cp config/database.yml.example config/database.yml
cp config/amqp.yml.example config/amqp.yml
cp config/redis.yml.example config/redis.yml
cp config/workers.yml.example config/workers.yml
cp config/interfaces.yml.example config/interfaces.yml
```

**必须修改项：**

| 文件 | 关键字段 |
|------|----------|
| `database.yml` | host/port/username/password，database 指向 `goDCE` 与 backup 库 |
| `redis.yml` | server 地址，如 `127.0.0.1:6379` |
| `amqp.yml` | host/port/username/password/vhost |
| `env.yml` | `model: production`，`node: a`（单机演示） |

Docker 模式使用 `docker/config/*.yml`，主机名须与 compose 服务名一致（`mysql` / `redis` / `rabbitmq`）。

### 2.3 数据库初始化

API 启动时会执行 `models.AutoMigrations()` 自动建表（主库 + backup 库）。

项目**不会**自动插入币种、市场、用户，需执行种子脚本：

```bash
./scripts/seed_demo_data.sh
```

种子数据包含：btc/usdt 币种、btcusdt 市场、demo-trader / demo-maker 用户及账户余额。

---

## 阶段 3：编译与启动

### 3.1 编译五个二进制

```bash
./build.sh
```

等价于：

```bash
go build -o ./cmd/api api/api.go
go build -o ./cmd/matching trade/matching.go
go build -o ./cmd/cancel order/cancel.go
go build -o ./cmd/treat trade/treat.go
go build -o ./cmd/workers workers/workers.go
```

### 3.2 启动顺序

```bash
./start.sh
```

启动顺序在脚本中为：api → matching → cancel → treat → workers。

**逻辑依赖：**

1. **api 最先**：执行 AutoMigrate、加载缓存、监听 :9990
2. **matching / treat / cancel**：依赖 MQ 与 DB，订阅各自市场队列
3. **workers**：依赖 MQ，消费 K 线 / Ticker 队列
4. **schedule**（可选）：`go run schedules/schedule.go`

### 3.3 停止与重启

```bash
./stop.sh
./restart.sh
```

进程 PID 写入 `pids/`，日志写入 `logs/`。

---

## 阶段 4：验证链路

### 4.1 公开接口（无需登录）

```bash
curl http://127.0.0.1:9990/api/c2e/v1/markets
curl http://127.0.0.1:9990/api/c2e/v1/currencies
curl 'http://127.0.0.1:9990/api/c2e/v1/depth?market=btcusdt'
curl http://127.0.0.1:9990/api/c2e/v1/tickers
```

### 4.2 登录

```bash
curl -X POST 'http://127.0.0.1:9990/api/web/v1/users/login' \
  -d 'source=email&symbol=demo@c2e.local&password=123456'
```

返回 Token，后续请求带鉴权头（见 `initializers/auth.go`）。

### 4.3 下单 → 撮合 → 成交

1. 用 demo-trader 下买单
2. 用 demo-maker 下卖单（价格匹配）
3. 查 `GET /orders`、`GET /trades/my`、账户余额变化
4. RabbitMQ 管理台（Docker 默认 `http://127.0.0.1:15672`，godce/godce）观察队列消费

### 4.4 重置演示数据

```bash
./scripts/reset_demo_state.sh
```

---

## 阶段 5：复刻时的推荐搭建顺序

若从零手写而非 fork，建议按以下里程碑推进（与 [02-feature-roadmap.md](./02-feature-roadmap.md) 对应）：

| 里程碑 | 交付物 | 验证方式 |
|--------|--------|----------|
| M1 基础设施 | config + utils + models + AutoMigrate | 能连 MySQL/Redis，表自动创建 |
| M2 API 骨架 | Echo + routes + 统一响应 + i18n | `/markets` 返回空数组 |
| M3 基础数据 | Currency / Market / User / Account | 种子数据可查询 |
| M4 用户体系 | 登录、Token、账户查询 | 登录成功，查余额 |
| M5 下单（无撮合） | 写订单 + 冻结资金 | 下单后 DB 有记录、余额 locked |
| M6 MQ 接入 | RabbitMQ 交换机/队列声明 | 下单后 MQ 有消息 |
| M7 撮合引擎 | matching 进程 + 内存订单簿 | 买卖单能成交，MQ 出 treat/cancel 事件 |
| M8 成交结算 | treat 进程 | 订单 done、余额变化、trades 表有记录 |
| M9 撤单 | cancel 进程 + API 撤单 | 冻结释放、深度更新 |
| M10 行情 | depth Redis + KLine/Ticker Worker | 深度/Ticker/K 线接口有数据 |
| M11 定时任务 | schedule | K 线补算、待成交检查 |
| M12 容器化 | Dockerfile + compose | 一键 `docker compose up` |

---

## 常见问题

| 现象 | 可能原因 |
|------|----------|
| 启动即退出 | MySQL/Redis/RabbitMQ 未就绪或配置错误 |
| 市场/币种为空 | 未执行 `seed_demo_data.sh` |
| 下单无成交 | matching/treat 进程未启动，或市场 `matching_able=false` |
| 深度为空 | matching 未运行或未 buildDepth 写 Redis |
| Docker 连不上 DB | `docker/config/database.yml` 主机名应为 `mysql` 而非 `127.0.0.1` |

---

## 参考文件

| 用途 | 路径 |
|------|------|
| API 启动与初始化 | `api/api.go` |
| 路由表 | `routes/v1.go` |
| 编译脚本 | `build.sh` |
| 启动脚本 | `start.sh` |
| Docker 编排 | `docker-compose.yml` |
| 种子 SQL | `scripts/seed_demo_data.sql` |
| 项目 README | `README.md` |
