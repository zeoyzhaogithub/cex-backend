# 钱包模块 · 代码走读

链上钱包的 Go 实现文档，与 [business/](../business/) 中的业务文档对应。

| 文档 | 业务背景 | 课程 | 状态 |
| :--- | :--- | :--- | :--- |
| [eth-wallet.md](./eth-wallet.md) | [06-eth-wallet.md](../business/06-eth-wallet.md) | 5.4 代码走读、5.5 部署演示 | 待编写 |
| [usdt-wallet.md](./usdt-wallet.md) | [07-usdt-wallet.md](../business/07-usdt-wallet.md) | 6.1 代码走读 | 待编写 |

## 计划目录结构（随代码补充）

```
src/
└── wallet/
    ├── eth/          # ETH 原生：地址池、扫块、归集、出金
    ├── erc20/        # ERC-20：USDT 等代币
    ├── address/      # 地址生成与地址池
    ├── scanner/      # 链上监听（扫链）
    ├── sweep/        # 归集
    └── withdraw/     # 出金
```

## 部署与联调（5.5）

> 待补充：环境变量、RPC 配置、本地链（cex-chain）联调步骤。
