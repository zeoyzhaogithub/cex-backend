# USDT 钱包 · 代码走读

> 业务背景：[07-usdt-wallet.md](../../business/07-usdt-wallet.md)  
> 课程：6.1 EUSDT 代码走读、6.2 USDT 钱包讲解

## 待覆盖内容

- [ ] USDT 合约地址配置（主网 / 测试网）
- [ ] ERC-20 `Transfer` 事件解析
- [ ] `decimals` 与金额换算
- [ ] 与 ETH 扫链模块的协作（同地址、分资产）
- [ ] USDT 归集：`transfer()` 合约调用
- [ ] USDT 出金与手续费
- [ ] Gas 补给：托管地址 ETH 不足时的处理

## 核心接口（示意）

| 模块 | 职责 |
| :--- | :--- |
| `Erc20Scanner` | 监听 USDT Transfer 入账 |
| `UsdtSweep` | 托管地址 USDT → 热钱包 |
| `UsdtWithdraw` | 热钱包 USDT → 用户地址 |

## 与 ETH 钱包的复用

| 复用 | 独立 |
| :--- | :--- |
| 地址池、UID 关联 | 扫链逻辑（原生 vs 事件） |
| Gas 补给机制 | 入账字段（ETH vs USDT 余额） |
| 出金审批流程 | 合约 ABI 调用 |

> 代码就绪后在此补充包路径、关键函数与数据表说明。
