# OCP 随机结束时间 — TEE + VRF 改造方案

## 背景

当前 commit-reveal 方案有两个根本缺陷：

1. **运营方能提前算出 offset**：运营方自己持有 `seed`，加上 `blockhash` 在 reveal 前几个区块已知，可以在 reveal 前就算出 `randomizedEndTime`。
2. **offset 写进了链上 storage**：`revealRandomizedEnd()` 调用后 `randomizedEndTime` 是 public 变量，任何人都能通过 ABI 或 `eth_getStorageAt` 读出来。

目标：**连运营方自己也不知道精确结束时间，直到结算那一刻才触发。**

---

## 目标架构

```
┌─────────────────────────────────────────────┐
│               TEE Keeper                    │
│  （AWS Nitro Enclave / Intel SGX / Phala）  │
│                                             │
│  1. vault 创建后：创建者发起 VRF 请求       │
│  2. TEE 从 VRF proof 链下推导 offset        │
│     （不依赖链上事件明文）                  │
│  3. 在 enclave 内计算：endTime = base + offset │
│  4. 安静等待到 endTime                      │
│  5. 调用 factory.finalizeVault(vault)       │
│                                             │
│  内部状态从不离开 enclave                   │
└─────────────────────────────────────────────┘
         │ requestRandomWords()     ▲ rawFulfillRandomWords()
         ▼                         │
┌─────────────────────────────────────────────┐
│         OCPVaultFactory                    │
│                                             │
│  - 持有 VRF 订阅 + Coordinator 接口        │
│  - 维护 requestId → vault 映射             │
│  - VRF 回调只 emit keccak256(offset) 承诺  │
│    （不暴露 offset 明文，不写入 storage）   │
│  - finalizeVault() 中转 TEE 结算调用       │
└─────────────────────────────────────────────┘
         │ RandomOffsetCommitted 事件（仅哈希承诺）
         ▼
┌─────────────────────────────────────────────┐
│              OCPVault                       │
│                                             │
│  - 无任何结算时间信息                       │
│  - 随机结束启用时 canResolve() 永远返回 false │
│  - 只有 factory 能调用 finalizeByFactory()  │
└─────────────────────────────────────────────┘
```

---

## 为什么必须 TEE + VRF 两者都要

| 威胁 | 只用 VRF | 只用 TEE | TEE + VRF |
|---|---|---|---|
| 运营方提前算出 offset | ✅ VRF 不由运营方控制 | ❌ TEE 运营方可硬编码逻辑 | ✅ |
| 外部用户从链上读 offset | ❌ offset 仍存链上 | ✅ TEE 不写 offset 到链上 | ✅ |
| 运营方挑选有利的触发时机 | ✅ VRF 混入 blockhash | ✅ TEE 自动触发，无人工干预 | ✅ |
| 声称"TEE 跑对了"无法验证 | ❌ 无证明 | ❌ 随机性无法链上验证 | ✅ VRF 证明在链上 |

---

## 合约改动

### OCPVaultFactory.sol

继承 `VRFConsumerBaseV2Plus`（Chainlink）。

新增状态变量：
```solidity
IVRFCoordinatorV2Plus public immutable vrfCoordinator;
uint256 public vrfSubscriptionId;
bytes32 public vrfKeyHash;
uint32  public vrfCallbackGasLimit;
uint16  public vrfRequestConfirmations;
address public owner;
mapping(uint256 => address) private _requestToVault;
mapping(uint256 => uint32) private _requestToWindowLength;
mapping(address => bool) public vrfRequested;
```

新增函数：
```solidity
/// @notice vault 创建者在创建时调用，发起 VRF 请求
function requestRandomEnd(address vault, uint32 windowLength) external {
    require(_creatorByVault[vault] == msg.sender, "Not creator");
    require(!vrfRequested[vault], "Already requested");
    OCPVault(vault).enableRandomizedEnd();
    uint256 requestId = vrfCoordinator.requestRandomWords(...);
    _requestToVault[requestId] = vault;
    _requestToWindowLength[requestId] = windowLength;
    vrfRequested[vault] = true;
    emit RandomEndRequested(vault, requestId, windowLength);
}

/// @notice Chainlink 回调 — 只 emit keccak256(offset) 承诺值，不暴露明文
function rawFulfillRandomWords(uint256 requestId, uint256[] memory randomWords) external {
    require(msg.sender == address(vrfCoordinator), "Only coordinator");
    address vaultAddr = _requestToVault[requestId];
    uint32 windowLength = _requestToWindowLength[requestId];
    uint32 offset = uint32(randomWords[0] % (uint256(windowLength) + 1));
    // 只 emit 哈希承诺 — 链上日志无法被利用
    emit RandomOffsetCommitted(vaultAddr, keccak256(abi.encodePacked(offset)), requestId);
    delete _requestToVault[requestId];
    delete _requestToWindowLength[requestId];
}

/// @notice TEE Keeper 通过此入口触发 vault 结算
function finalizeVault(address vault) external {
    OCPVault(vault).finalizeByFactory();
}

event RandomEndRequested(address indexed vault, uint256 indexed requestId, uint32 windowLength);
event RandomOffsetCommitted(address indexed vault, bytes32 offsetCommitment, uint256 indexed requestId);
```

> **安全性**：`RandomOffsetCommitted` 只包含 `keccak256(offset)` — 哈希不可逆，外部无法推出 offset。
> TEE Keeper 从 Chainlink 节点的 VRF 证明链下推导 offset，不依赖链上事件。
> 从 VRF 回调到结算之间，**任何外部观察者都无法知道结算时间**（密码学保证，非时间竞争）。

---

### OCPVault.sol

**删除（旧 commit-reveal 方案已全部移除）：**
- `randomizedEndTime`、`randomizedEndRevealed`、`randomizedEndWindowStart`、`randomizedEndWindowLength`、`randomizedEndCommit`、`randomizedEndKeeper`
- `revealRandomizedEnd()`
- `configureRandomizedEnd()`
- `canResolve()` 里的 commit-reveal 逻辑

**保留：**
- `randomizedEndEnabled`（bool，由 factory 调用 `enableRandomizedEnd()` 设置）
- `canResolve()`：随机结束启用时永远返回 `false`，外部无法通过 `finalize()` 触发结算

**新增：**
```solidity
/// @notice 由 factory 调用，启用随机结束模式
function enableRandomizedEnd() external {
    require(msg.sender == factory, "Only factory");
    require(!resolved, "Already finalized");
    require(!randomizedEndEnabled, "Already enabled");
    randomizedEndEnabled = true;
}

/// @notice 由 factory 调用，TEE Keeper 触发结算（唯一结算入口）
function finalizeByFactory() external nonReentrant {
    require(msg.sender == factory, "Only factory");
    require(!resolved, "Already finalized");
    require(randomizedEndEnabled, "Not random-end vault");
    // 终局规则与 finalize() 一致
    uint256 total = _totalPrincipal;
    if (total == 0) outcome = Outcome.INVALID;
    else if (_totalStakeYes * 2 > total) outcome = Outcome.YES;
    else if (_totalStakeNo * 2 > total) outcome = Outcome.NO;
    else outcome = Outcome.INVALID;
    resolved = true;
    emit Finalized(outcome);
}
```

---

### IOCPVault.sol

删除所有 `randomizedEnd*` getter，只保留 `randomizedEndEnabled()`。

---

## TEE Keeper 改动

### 平台选项

| 平台 | 特点 |
|---|---|
| AWS Nitro Enclave | 最容易部署，通过 AWS attestation 验证 |
| Intel SGX | 更通用，DeFi keeper 中广泛使用 |
| Phala Network | Web3 原生 TEE，无需自己维护服务器，最省心 |

### Keeper 逻辑（伪代码）

```typescript
// TEE enclave 内部 — 从 Chainlink VRF proof 链下推导 offset

async function main() {
  const provider = new ethers.JsonRpcProvider(RPC_URL);
  const factoryContract = new Contract(FACTORY_ADDR, FACTORY_ABI, walletInsideEnclave);

  // 监听 VRF 请求事件（获取 vault 和 windowLength）
  factoryContract.on("RandomEndRequested", async (vaultAddr, requestId, windowLength) => {
    // 从 Chainlink VRF 节点获取 proof，链下推导随机数
    const randomWord = await getRandomWordFromVRFProof(requestId);
    const offset = Number(randomWord % (BigInt(windowLength) + 1n));

    const vault = new Contract(vaultAddr, VAULT_ABI, provider);

    // 在 enclave 内计算精确结算时间（不离开 enclave）
    const baseEnd = await getBaseEnd(vault);
    const settlementTime = Number(baseEnd) + offset;

    // 安静等待，外部完全看不到这段逻辑
    const now = Math.floor(Date.now() / 1000);
    if (settlementTime > now) {
      await sleep((settlementTime - now) * 1000);
    }

    // 通过 factory 中转触发结算 — 这是外部第一次感知到"时间到了"
    await factoryContract.finalizeVault(vaultAddr);
  });
}
```

enclave 内持有 keeper EOA 私钥用于签名 `finalizeVault` 交易（付 gas），私钥永不离开 enclave。
Attestation 证明运行的是正确的代码。TEE Keeper 从 VRF proof 链下推导 offset，**不依赖链上事件中的任何明文信息**。

---

## CLI 改动

**删除：**
- `random-end-commit`
- `random-end-reveal`

**新增：**
- `random-end-request <vaultAddr> --window-length <seconds>` — 调用 `factory.requestRandomEnd()`

---

## 费用估算

### VRF LINK 费用（每个 vault）

Chainlink VRF V2+ 在 Base 网络上：
- 每次请求约 **0.001 - 0.003 LINK**（Base 网络 gas 极低）
- 每个 vault 只需请求一次
- 当前 LINK 价格约 $13

→ **每个 vault VRF 费用约 $0.02 - $0.04，可以忽略不计**

### TEE 平台费用

| 平台 | 计费方式 | 月费估算 |
|---|---|---|
| AWS Nitro Enclave | 依附 EC2，Nitro 本身免费；t3.small $0.021/h | **~$15/月** |
| Intel SGX（Azure DCsv3）| DC1s_v3 约 $0.113/h | **~$82/月** |
| Phala Network | 按 CU 计费，轻量任务约 1 CU @ $0.01/CU/h | **~$7/月** |

### 关键：TEE 是一对多，不是一对一

一个 TEE 实例监听所有 vault，不是每个 vault 跑一个 enclave：

```
1 个 TEE 实例 → 监听 N 个 vault → 统一定时结算
```

| 活跃 vault 数 | AWS Nitro 月成本 | 每个 vault 分摊 |
|---|---|---|
| 10 | $15 | $1.50 |
| 100 | $15 | $0.15 |
| 1000 | $15 | $0.015 |

**结论：规模化后每个 vault 边际成本接近零，可以直接写进协议手续费覆盖。**

---

## 实施步骤

1. ✅ 改造 `OCPVaultFactory.sol` — 新增 VRF Coordinator 接口、`requestRandomEnd` + `rawFulfillRandomWords`（只 emit commitment）+ `finalizeVault`
2. ✅ 改造 `OCPVault.sol` — 删除 commit-reveal 字段，新增 `enableRandomizedEnd()` + `finalizeByFactory()`，`canResolve()` 随机模式永远返回 false
3. ✅ 清理 `IOCPVault.sol` — 删除多余 getter，只保留 `randomizedEndEnabled()`
4. ✅ 更新 CLI — `random-end-commit/reveal` 换成 `random-end-request`
5. ✅ 更新 ABI — `abi.ts` 匹配新合约接口
6. 编写 TEE Keeper 服务 — Node.js 进程，在 Nitro Enclave / Phala 内运行
7. Base Sepolia 部署测试 — 充 LINK，验证端到端流程
8. Attestation 验证 — 确认 enclave 二进制哈希与发布代码一致

---

## 待确认事项

- **LINK 谁付**：工厂订阅由协议国库充值，或从 vault 创建费中扣
- **TEE 崩溃兜底**：`t2 + 宽限期` 后任何人可调用 `finalize()`，与当前逻辑一致
- **平台选型**：Phala 最省心（无服务器），AWS Nitro 最成熟；初期建议 AWS，后期迁 Phala
- ✅ ~~**事件日志泄露**：已解决 — 链上只 emit `keccak256(offset)` 承诺值，外部无法反推 offset~~
