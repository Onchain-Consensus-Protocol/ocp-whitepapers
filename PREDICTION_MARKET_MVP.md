# 预测市场实现说明（基线 / 原 MVP）

> 本文档描述预测市场的**基线能力**（原 MVP：金库 + 交易 + 解析 + 分红）。  
> **当前测试网合约已超越 MVP**，金库已包含冷静期、一次性挑战、挑战期（再质押/移仓）等 POC 核心机制，详见 [PREDICTION_MARKET_IMPLEMENTATION_STATUS.md](./PREDICTION_MARKET_IMPLEMENTATION_STATUS.md)。协议设计见 [POC_WHITEPAPER_CN.md](./POC_WHITEPAPER_CN.md)。

## 基线能力（原「最小可用链」）

1. **金库投票决定结果**：`OCPVault`，三态 YES/NO/INVALID，占比决定 outcome
2. **预测市场交易**：恒定乘积 AMM，`buyYes` / `buyNo` / `sellYes` / `sellNo`
3. **手续费进金库**：0.3% 交易费 donate 到金库，按本金占比分红
4. **锁定至结算**：金库存款在终局之前不可提款

## 新增合约

| 合约 | 路径 | 说明 |
|------|------|------|
| OCPVault | `core/OCPVault.sol` | OCP 金库，deposit/vote/donate/withdraw |
| OCPVaultFactory | `factory/OCPVaultFactory.sol` | 一次性创建金库 + 市场 |
| IOCPVault | `interfaces/IOCPVault.sol` | 金库最小接口 |

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

## 测试网已超出本文档（当前合约）

- **一次性挑战 + 挑战期（再质押/移仓）**：已在 `OCPVault` 中实现，见实现状态文档。

## 尚未实现（后续版本）

- 躺平惩罚（30/70 分红）
