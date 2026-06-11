# ETH 钱包

> 本文聚焦 **Ethereum 原生 ETH** 的链上实现。

## 模块概览

| 子模块 | 说明 | 课程 |
| :--- | :--- | :--- |
| 地址创建 | 为用户分配 ETH 托管充值地址 | 5.1 |
| 扫链 | 监听 ETH 充值到账 | 5.2 |
| 归集与提币 | 分散地址归集、热钱包出金 | 5.3 |
| 代码走读 | Go 实现与模块划分 | [architecture/wallets/eth-wallet.md](../architecture/wallets/eth-wallet.md) |
| 部署演示 | 钱包服务启动与联调 | 5.5 |

```
用户充值 ETH
  → 分配托管地址（地址池）
  → 扫链识别到账
  → 更新链下 ETH 余额
  → 归集至热/冷钱包
  → 出金时从热钱包转账
```

---

## 地址创建

### 流程

1. 用户选择 **ETH** 链入金
2. 从**地址池**分配未使用的 ETH 地址（或实时生成）
3. 将 `address` 与 `uid` 关联写入用户地址表
4. 返回地址 / 二维码供用户充值

### 技术要点

| 项 | 说明 |
| :--- | :--- |
| 密钥算法 | secp256k1，与 EVM 生态一致 |
| 地址格式 | `0x` 开头 42 位十六进制 |
| 私钥存储 | 平台加密存储，不对用户暴露 |
| 地址池 | 预生成 + 阈值补货，降低充值延迟 |

### 与通用模型的关系

同 [02-custodial-wallet.md § 地址生成](./02-custodial-wallet.md#2-钱包生成与-uid-关联)，ETH 为 EVM 链上的**原生代币**充值地址，一条地址同时可接收该链上的 ETH 转账。

---

## 扫链（链上监听）

### 职责

监听用户托管地址的 **ETH 入账**，确认后更新链下账务。

### 流程

```
新区块 / Webhook 回调
  → 过滤目标地址（用户托管地址列表）
  → 校验：原生 ETH 转账、金额 > 0
  → 等待确认数（如 12 个区块）
  → 写入充值记录，更新用户 ETH 余额
```

### 实现方式

| 方式 | 说明 |
| :--- | :--- |
| **自建扫块** | 订阅 `newHeads`，解析 block transactions，`to` 匹配托管地址 |
| **第三方 Webhook** | Alchemy / Infura 地址活动通知，回调入账 |

### 注意点

- **原生 ETH**：监听 `tx.Value`，非 ERC-20 的 `Transfer` 事件
- **确认数**：未达确认数前标记 pending，防止链重组
- **重复入账**：txHash + logIndex 唯一索引，幂等处理
- **Gas 来源**：用户充值地址需有少量 ETH 才能后续归集（可归集时由热钱包补 Gas）

---

## 归集与提币

### 归集（Sweep）

用户 ETH 分散在各托管地址。归集将 ETH 转入交易所**归集地址 / 热钱包**：

```
触发条件（余额 > 阈值 且 Gas 合适）
  → 从托管地址 signed tx 转入热钱包
  → 更新链上余额、归集记录
```

- 单地址余额不足以付 Gas 时，可由热钱包**预充 Gas** 再归集
- 大批量可用归集合约批量 transfer，节省操作次数

### 提币（Withdraw）

```
用户发起 ETH 出金
  → 校验链下余额、风控、手续费
  → 从热钱包向用户指定 0x 地址转账
  → 等待确认，更新订单状态
```

| 类型 | 用途 |
| :--- | :--- |
| **热钱包** | 日常出金、归集目标地址 |
| **冷钱包** | 大额储备，人工/策略转入 |

---

## 与 USDT 钱包的区别

| 维度 | ETH 钱包（本章） | USDT 钱包（[07-usdt-wallet.md](./07-usdt-wallet.md)） |
| :--- | :--- | :--- |
| 资产类型 | 链原生 ETH | ERC-20 合约代币 |
| 监听方式 | 交易 `value` 字段 | 合约 `Transfer` 事件 |
| 归集 / 出金 | 直接 ETH 转账 | `transfer()` 合约调用 |
| Gas | 转账本身消耗 ETH | 需地址持有 ETH 作 Gas |

同一 ETH 地址可同时接收 ETH 和 ERC-20 USDT，但扫链、入账逻辑需**分模块**处理。

---

## 相关文档

- [02-custodial-wallet.md](./02-custodial-wallet.md) — 中心化钱包通用设计
- [architecture/wallets/eth-wallet.md](../architecture/wallets/eth-wallet.md) — ETH 钱包代码走读
- [architecture/wallets/usdt-wallet.md](../architecture/wallets/usdt-wallet.md) — USDT 钱包代码走读
