# USDT 钱包（ERC-20）

> ETH 原生币实现见 [06-eth-wallet.md](./06-eth-wallet.md)。

## 模块概览

| 子模块 | 说明 | 课程 |
| :--- | :--- | :--- |
| USDT 钱包讲解 | ERC-20 USDT 充值、入账、归集、出金 | 6.2 |
| 代码走读 | 合约交互与扫链实现 | [architecture/wallets/usdt-wallet.md](../architecture/wallets/usdt-wallet.md) |

**EUSDT** 指 Ethereum 链上的 **ERC-20 USDT**（与 ETH 原生币共用同一套地址体系，扫链与转账逻辑不同）。

---

## ERC-20 与原生 ETH 的差异

| 维度 | ETH（[06-eth-wallet](./06-eth-wallet.md)） | USDT（本章） |
| :--- | :--- | :--- |
| 资产 | 链原生代币 | 智能合约代币 |
| 合约地址 | 无（原生） | 需配置 USDT 合约地址（主网 / 测试网不同） |
| 充值识别 | `tx.Value` | 解析 `Transfer(from, to, value)` 事件 |
| 转账方式 | 普通 ETH 转账 | 调用合约 `transfer(to, amount)` |
| 精度 | 18 位 | USDT 通常 6 位（以合约 `decimals()` 为准） |
| Gas | 转账消耗 ETH | 合约调用消耗 ETH，地址须预留 Gas |

---

## 地址与充值

### 地址复用

用户 **ETH 托管地址** 可同时接收：

- 原生 ETH 充值 → 由 ETH 钱包模块处理
- ERC-20 USDT 充值 → 由 USDT 钱包模块处理

同一 `0x` 地址，**扫链逻辑按资产类型分流**。

### 充值流程

```
用户向托管地址转入 USDT
  → 扫链捕获 USDT 合约 Transfer 事件
  → 校验：to = 托管地址，contract = 配置的 USDT 地址
  → 确认数达标
  → 按 decimals 换算金额，更新链下 USDT 余额
```

### 代币白名单

仅处理配置表中的 USDT 合约地址，防止伪造代币入账。

---

## 扫链（ERC-20）

### 监听目标

- **事件**：`Transfer(address indexed from, address indexed to, uint256 value)`
- **合约**：USDT 官方合约地址（链 ID 对应网络）

### 流程

```
区块 / Webhook
  → 过滤 USDT 合约 logs
  → to 地址 ∈ 用户托管地址集
  → 解析 value，结合 decimals 得人类可读数量
  → 幂等：txHash + logIndex
  → 更新 USDT 账务
```

### 与 ETH 扫链的配合

同一笔区块可能同时包含 ETH 与 USDT 入账，两个模块独立处理、分别入账。

---

## 归集与提币

### 归集

将分散托管地址上的 USDT 归集至热钱包：

```
调用 USDT.transfer(热钱包地址, amount)
Gas 由该地址持有的 ETH 支付
```

- 若托管地址无 ETH，需**Gas 补给**（从热钱包转少量 ETH）后再归集 USDT

### 提币

```
用户发起 USDT 出金
  → 校验链下 USDT 余额
  → 热钱包调用 USDT.transfer(用户地址, amount)
  → 扣除平台手续费（链下或链上逻辑）
```

---

## 账务

链下账务与 [02-custodial-wallet § 账务设计](./02-custodial-wallet.md#中心化钱包账务设计) 一致：

- 用户 UID 下独立 **USDT 余额** 字段
- ETH 充值、USDT 充值分别入账，用户侧统一展示

---

## 相关文档

- [06-eth-wallet.md](./06-eth-wallet.md) — ETH 原生钱包（地址、扫链、归集）
- [architecture/wallets/usdt-wallet.md](../architecture/wallets/usdt-wallet.md) — USDT 代码走读
- [architecture/wallets/eth-wallet.md](../architecture/wallets/eth-wallet.md) — ETH 代码走读
