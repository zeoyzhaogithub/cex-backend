# cex-backend

中心化交易所（CEX）后端。通过链上资产托管完成交易与撮合，涵盖中心化钱包、现货交易、订单簿、撮合引擎等核心模块。

**特点：** 交易性能更优；具备监管、AML 等合规能力。

## 目标

1. 学习中心化交易所的核心业务、算法与系统架构
2. 搭建并部署一个 CEX 平台
3. 衔接钱包、永续合约等扩展模块
4. 掌握现货交易完整生命周期、订单簿结构与撮合引擎原理
5. 完成后端核心模块的代码实现与架构分析

## 快速开始

> 代码实现阶段补充：环境要求、依赖安装、启动命令。

## 文档

| 目录 | 说明 |
| :--- | :--- |
| [docs/business/](./docs/business/) | CEX 领域知识与业务流程（与代码解耦） |
| [docs/architecture/](./docs/architecture/) | 后端模块划分、数据模型、API 与代码走读 |
| [docs/img/](./docs/img/) | 业务文档配图（drawio 源文件与导出图） |

### 业务文档索引

| 文档 | 说明 | 状态 |
| :--- | :--- | :--- |
| [01-cex-overview](./docs/business/01-cex-overview.md) | CEX 概念、生态、与 DEX 对比、现货生命周期 | 已完成 |
| [02-custodial-wallet](./docs/business/02-custodial-wallet.md) | 中心化钱包：入金、监听、归集、出金 | 整理中 |
| [03-spot-trading-flow](./docs/business/03-spot-trading-flow.md) | 现货订单流与资金流 | 待编写 |
| [04-order-book](./docs/business/04-order-book.md) | 订单簿深度解析 | 待编写 |
| [05-matching-engine](./docs/business/05-matching-engine.md) | 撮合引擎核心算法 | 待编写 |

## 实现进度

- [x] 业务文档：CEX 概述
- [x] 业务文档：中心化钱包（结构整理，细节待完善）
- [ ] 业务文档：现货交易 / 订单簿 / 撮合引擎
- [ ] 代码：项目脚手架
- [ ] 代码：中心化钱包模块
- [ ] 代码：订单簿与撮合引擎

## 目录结构

```
cex-backend/
├── README.md                          # 项目入口（本文件）
├── LICENSE
├── docs/
│   ├── business/                      # 业务逻辑文档
│   │   ├── README.md                  # 业务文档索引
│   │   ├── 01-cex-overview.md         # CEX 概述与生态
│   │   ├── 02-custodial-wallet.md     # 中心化钱包
│   │   ├── 03-spot-trading-flow.md    # 现货订单流与资金流
│   │   ├── 04-order-book.md           # 订单簿
│   │   └── 05-matching-engine.md      # 撮合引擎
│   ├── architecture/                  # 架构与实现文档（随代码补充）
│   │   └── README.md
│   └── img/                           # 文档配图
│       ├── 中心化钱包.drawio
│       ├── 中心化钱包.png
│       └── 中心化钱包-提现.png
└── src/                               # 后端源码（待创建）
```
