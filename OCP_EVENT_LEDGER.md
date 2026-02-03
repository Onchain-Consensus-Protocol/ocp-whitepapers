# OCP 历史事件账本

## 概述

通过新事件对历史数据的强制依赖，**账本权威性由「最长历史链」上的累积共识决定**——类似比特币中最长链维护账本权威：参与越久、历史终局事件越多，该链上的权重与话语权越大。复制者（从零分叉或新协议）缺乏这段历史链上的资本累积与共识积淀，无法获得同等权威，面临高入场成本与低有效权重。无管理员，代码即法律。

## 核心合约

- **接口**：`solidity/src/interfaces/IOCPEventLedger.sol`
- **实现**：`solidity/src/core/OCPEventLedger.sol`
- **测试**：`solidity/test/OCPEventLedger.t.sol`

## 历史链依赖（权威性来源）

| 机制 | 说明 |
|------|------|
| **质押权重** | `stake()` 时按 `effectiveStakeWeight(user, amount)` 累加 `totalWeightA`/`totalWeightB`；胜负比较按权重，同额下高声誉方权重更高 |
| **资本声誉** | `cumulativeWinningStake` 来自历史赢方质押；`effectiveStakeWeight(user, baseStake)` = 基础权重 × (1 + min(声誉, 纪元上限)) |
| **挑战与再质押** | `challenge(eventId)` 仅劣势方可调用，支付 `challengeBond` 的 ETH，触发再质押期（`RE_STAKE_PERIOD_SECONDS`）；终局时保证金按赢方权重比例分配 |
| **挑战保证金 b** | 按**父链/路径**调整：沿父链向上最多 `MAX_BOND_ANCESTORS` 层统计「被挑战次数」与「挑战方获胜次数」，rate = 路径挑战成功率；bond = base × (2 − rate)，且 **bond ≤ base × 3**（`MAX_CHALLENGE_BOND_MULTIPLIER`）。低路径成功率 → 高保证金；设上限避免历史包袱。 |
| **父系快照** | 创建子事件时对父事件终局共识做 `keccak256(eventId, outcome, totalStakeA, totalStakeB)` 快照；子事件逻辑锁死在此快照上，形成不可篡改的历史链引用 |
| **深度约束** | `depth = parent.depth + 1`，`depth <= MAX_DEPTH(64)`；单父 `MAX_PARENT_COUNT = 1` |
| **难度声誉溢价** | 终局时赢方声誉增量 × `(PRECISION + depth * DEPTH_REPUTATION_BONUS)` / PRECISION，高深度事件赢家获得更多声誉 |

## 时间窗口与终局

- **createEvent(parentEventId, baseChallengeBond, stakeWindowSeconds, challengeWindowSeconds)**：质押期 `[0, stakeWindowEnd)`，挑战窗口 `[stakeWindowEnd, challengeWindowEnd)`；仅劣势方可 `challenge(eventId)` 并支付 ETH 保证金，触发再质押期至 `reStakePeriodEnd`。
- **终局**：未挑战时 `block.timestamp >= challengeWindowEnd` 可终局；已挑战时须 `block.timestamp >= reStakePeriodEnd` 方可终局。挑战方获胜时 `_totalDisputeSuccess` 自增（统计用）；**保证金**由父链/路径的挑战成功率决定，不依赖全局历史。

## 经济模型（中本聪精神）

- **奖励减半**：当前纪元 `E = totalFinalizedEvents / EPOCH_SIZE`（EPOCH_SIZE = 210_000）。终局时发放的声誉增量 = 赢方质押 × `(PRECISION >> E)` × 深度溢价 / PRECISION²，即随纪元 1.0 → 0.5 → 0.25。
- **声誉永恒**：`cumulativeWinningStake` 只增不减；减半仅影响**本事件新赚取的**声誉量。
- **资源约束**：`maxDepth(64)`、单父；创建子事件时父必须已终局。

## 技术约束（EVM 优化）

- **全局状态**：`nextEventId`、`totalFinalizedEvents`、`currentEpoch()` 由 `totalFinalizedEvents / EPOCH_SIZE` 推导。
- **用户映射**：`UserReputation(cumulativeWinningStake, disputeSuccessCount, totalParticipatedEvents)`；按事件存 `_stakeAmount`、`_stakeSideA`、`_participants[eventId]`。
- **惰性更新**：仅在 `stake`、`finalizeEvent` 时写状态；终局时只遍历该事件的 `_participants[eventId]`，无全局循环。

## 账本权威性如何由历史链保证

- **资本声誉**：**从零开始的新链**上用户声誉为 0，同额质押下有效权重仅为基准；**分叉链**继承到分叉点为止的声誉，分叉后各自在新事件上累积。**在最长历史链上**持续参与的一方权重更高——权威向「历史更长的链」倾斜。
- **父系快照**：子事件参数/选项依赖父共识快照，形成链式引用。**从零开始、不包含本链历史的新链**无法伪造本链上的快照序列，因而无法声称拥有本链这段历史链的权威；**分叉链**则继承到分叉点为止的同一快照序列，对共同历史权威一致，分叉后由各自新事件的链式延伸与「最长链」规则竞争。
- **纪元减半**：新赚声誉随纪元减半，已累积声誉只增不减。**从零开始的新链**纪元从 0 起、无历史，受减半压制；**分叉链**继承原链纪元，与老链在减半规则上一视同仁，分叉后权威由最长链竞争决定。

## 计算示例（长链参与者 vs 零历史）

设 `BASE_REPUTATION_CAP = 100e18`，纪元 E=2 时 `cap = 100e18 >> 2 = 25e18`。  
长链参与者 `cumulativeWinningStake = 80e18` → 有效声誉 = min(80e18, 25e18) = 25e18。  
新链/攻击者 = 0。

同额 `baseStake = 10e18`：
- 长链参与者有效权重 = `10e18 * (1e18 + 25e18) / 1e18 = 350e18`
- 新链参与者有效权重 = `10e18`

即同额质押下，**拥有最长历史链参与记录**的一方权重为新链的 35 倍；后者需投入 35 倍本金才能在权重上与之相当——体现「最长链维护账本权威性」的类似逻辑。
