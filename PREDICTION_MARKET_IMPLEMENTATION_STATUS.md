# 预测市场实现状态

本文档明确区分**白皮书中的完整设计**与**当前已实现功能**。

---

## 测试网已实现（超越 MVP）

**当前上测试网的合约已超越 MVP 白皮书**，包含 POC 核心的冷静期、一次性挑战与挑战期（再质押/移仓）。

| 功能 | 状态 | 说明 |
|------|------|------|
| 金库投票决定结果 | ✅ | `OCPVault`：三态 YES/NO/INVALID，占比决定 outcome |
| 预测市场交易 | ✅ | 恒定乘积 AMM：`buyYes` / `buyNo` / `sellYes` / `sellNo` |
| 手续费进金库 | ✅ | 0.3% 交易费 donate 到金库 |
| 分红按本金占比 | ✅ | 标准 O(1) 分配 |
| 锁定至结算 | ✅ | 金库存款在终局前不可提款 |
| 解析 | ✅ | `resolve()` 读取金库 `getOutcome()` |
| 赎回 | ✅ | `redeem()` 赢方 1:1 兑付 |
| **冷静期** | ✅ | t0 后时长 Δ，仅劣势方或 INVALID 可发起一次挑战 |
| **挑战保证金** | ✅ | 挑战需支付 `minStake`（b），逻辑与白皮书一致 |
| **挑战期 / 再质押期** | ✅ | 挑战后开放时长 Δ，可重新投票、整边移仓 |
| **一次性挑战** | ✅ | 每场仅允许一次挑战，挑战期结束即终局 |

### 合约

- `OCPVault`（`core/OCPVault.sol`）— 含冷静期、挑战、再质押、终局
- `OCPVaultFactory`（`factory/OCPVaultFactory.sol`）
- `PredictionMarket`（`market/PredictionMarket.sol`）— AMM 交易 + resolve + redeem

---

## 未实现（协议/应用规格中描述）

| 功能 | 文档 | 说明 |
|------|------|------|
| 躺平惩罚 | PM 应用规格 §2.3 | 本周期无交易者仅获 30% 分红，70% 再分配 |
| （已实现）结局未定处理 | PM 应用规格 §4.3 | 50:50 或参与不足时 Invalid 退款（见 OCPVault 终局规则与市场 INVALID 赎回） |

---

## 文档结构（2025 年 2 月）

| 文档 | 定位 |
|------|------|
| **OCP_WHITEPAPER_CN.md** | 共识上链协议，总领白皮书 |
| **POC_WHITEPAPER_CN.md** | POC 机制，质押—挑战—共识 |
| **PREDICTION_MARKET_APPLICATION_SPEC.md** | PM 应用规格 |
| **PREDICTION_MARKET_MVP.md** | 预测市场基线实现说明（MVP 为历史版本；测试网已超越） |
| **PREDICTION_MARKET_L2_ARCHITECTURE.md** | 三分架构：事件账本(主网) / 金库+预测市场(L2)，L2 部署设计理由 |
| **本文档** | 实现状态对照 |

---

## 路线图建议

1. **MVP**（已过）：金库 + 交易 + 解析 + 分红
2. **当前测试网**：上述 + 一次性挑战 + 冷静期 + 挑战期（再质押/移仓）
3. **V2**：躺平惩罚
4. **V3**：Invalid 边界处理、参数调优

**TODO**
- 增加 `totalFees`（金库累计手续费）链上字段与事件，便于前端展示与索引统计
