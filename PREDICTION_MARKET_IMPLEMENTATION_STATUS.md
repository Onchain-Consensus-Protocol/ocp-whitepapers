# 预测市场实现状态

本文档明确区分**白皮书中的完整设计**与**当前已实现功能**。

---

## 已实现（MVP）

| 功能 | 状态 | 说明 |
|------|------|------|
| 金库投票决定结果 | ✅ | `OCPVault`：投票=是，不投票=否，占比决定 outcome |
| 预测市场交易 | ✅ | 恒定乘积 AMM：`buyYes` / `buyNo` / `sellYes` / `sellNo` |
| 手续费进金库 | ✅ | 0.3% 交易费 donate 到金库 |
| 分红按本金占比 | ✅ | 标准 O(1) 分配 |
| 锁定至结算 | ✅ | 金库存款在 `resolutionTime` 前不可提款 |
| 解析 | ✅ | `resolve()` 读取金库 `getOutcome()` |
| 赎回 | ✅ | `redeem()` 赢方 1:1 兑付 |

### 合约

- `OCPVault`（`core/OCPVault.sol`）
- `OCPVaultFactory`（`factory/OCPVaultFactory.sol`）
- `PredictionMarket`（`market/PredictionMarket.sol`）— 已实现交易逻辑

---

## 未实现（协议/应用规格中描述）

| 功能 | 文档 | 说明 |
|------|------|------|
| 挑战机制（一次性） | POC 白皮书 §2 | 默认 24 小时冷静期/挑战期，仅落后方可发起 Y/N 挑战 |
| 保证金 | POC 白皮书 §2 | 挑战需支付 b，赢退输给对手 |
| 最终对决期 | POC 白皮书 §2 | 挑战后 PM+金库开放（时长同冷静期，默认 24 小时），新资金可流入 |
| **一次性挑战** | POC 白皮书 §2 | 每场事件仅允许一次挑战，对决期后终局（无多轮） |
| 躺平惩罚 | PM 应用规格 §2.3 | 本周期无交易者仅获 30% 分红，70% 再分配 |
| 结局未定处理 | PM 应用规格 §4.3 | 50:50 或参与不足时 Invalid 退款 |

---

## 文档结构（2025 年 2 月）

| 文档 | 定位 |
|------|------|
| **OCP_WHITEPAPER_CN.md** | 共识上链协议，总领白皮书 |
| **POC_WHITEPAPER_CN.md** | POC 机制，质押—挑战—共识 |
| **PREDICTION_MARKET_APPLICATION_SPEC.md** | PM 应用规格 |
| **PREDICTION_MARKET_MVP.md** | 已实现 MVP 使用说明 |
| **PREDICTION_MARKET_L2_ARCHITECTURE.md** | 三分架构：事件账本(主网) / 金库+预测市场(L2)，L2 部署设计理由 |
| **本文档** | 实现状态对照 |

---

## 路线图建议

1. **MVP**（当前）：跑通金库 + 交易 + 解析 + 分红
2. **V1**：一次性挑战 + 最终对决期
3. **V2**：躺平惩罚
4. **V3**：Invalid 边界处理、参数调优
