# 预测市场 MVP 实现说明

> 本 MVP 基于**共识上链协议（OCP）**与**承诺证明（POC）**的预测市场应用。协议设计（一次性挑战）见 [POC_WHITEPAPER_CN.md](./POC_WHITEPAPER_CN.md)。

## 已实现（最小可用链）

1. **金库投票决定结果**：`PredictionMarketVault`，投票=是，不投票=否，占比决定 outcome
2. **预测市场交易**：恒定乘积 AMM，`buyYes` / `buyNo` / `sellYes` / `sellNo`
3. **手续费进金库**：0.3% 交易费 donate 到金库，按本金占比分红
4. **锁定至结算**：金库存款在 `resolutionTime` 之前不可提款

## 新增合约

| 合约 | 路径 | 说明 |
|------|------|------|
| PredictionMarketVault | `core/PredictionMarketVault.sol` | 预测市场专用金库，deposit/vote/donate/withdraw |
| PredictionMarketVaultFactory | `factory/PredictionMarketVaultFactory.sol` | 一次性创建金库 + 市场 |
| IPredictionMarketVault | `interfaces/IPredictionMarketVault.sol` | 金库最小接口 |

## 修改合约

| 合约 | 变更 |
|------|------|
| PredictionMarket | 实现 AMM 交易、resolve、redeem，手续费 donate 到金库 |

## 使用流程

1. 工厂 `createMarket(depositToken, resolutionTime, initialLiquidity)` 创建金库+市场
2. 用户向金库存款、投票（可选）
3. 用户在市场交易 YES/NO
4. `resolutionTime` 后任何人调用 `resolve()`，读取金库投票结果
5. 用户 `redeem()` 赢方份额，`withdraw()` 金库本金+分红

## 待实现（后续版本）

- **一次性挑战 + 最终对决期**（OCP/POC 协议核心）
- 躺平惩罚（30/70 分红）
