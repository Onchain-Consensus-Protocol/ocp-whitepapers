# OCP 测试网部署指南

实现「AI 调 API 在测试网创建命题金库，人类参与投票」的完整步骤。

---

## 一、部署合约到测试网

### 1. 环境

- 已安装 [Foundry](https://book.getfoundry.sh/getting-started/installation)（`forge`）
- 一个测试网钱包，并持有该测试网原生代币（如 Sepolia ETH，用于 gas）

### 2. 安装依赖（首次）

在项目根目录下执行：

```bash
cd solidity
forge install openzeppelin/openzeppelin-contracts foundry-rs/forge-std --no-commit
```

若已有 `lib/` 且能正常 `forge build`，可跳过。

### 3. 编译

```bash
cd solidity
forge build
```

### 4. 部署

设置私钥（仅测试网、勿用主网私钥），然后执行部署脚本：

```bash
cd solidity
export PRIVATE_KEY=0x你的测试网钱包私钥
forge script script/Deploy.s.sol:DeployScript --rpc-url https://sepolia.infura.io/v3/YOUR_KEY --broadcast
```

- `--rpc-url` 可换成任意 Sepolia RPC（如 Alchemy、Public 等）。
- 部署成功后终端会打印：
  - **MockERC20 (deposit token):** `0x...`
  - **PredictionMarketVaultFactory:** `0x...`

记下这两个地址，下一步配置 API 要用。

### 5. 可选：给 API 用的钱包打测试代币

部署脚本会给部署者地址 mint 100 万 OCPT。若你希望 **API 后端用的钱包** 也能创建带初始流动性的市场，请把部分 OCPT 转给该钱包，或在部署时改用该钱包私钥部署（这样 mint 会到该地址）。

---

## 二、部署 API 服务（ocp-api）

### 1. 环境变量

进入 `ocp-api`，复制示例并填写：

```bash
cd ocp-api
cp .env.example .env
```

编辑 `.env`，必填项：

| 变量 | 说明 |
|------|------|
| `NEXT_PUBLIC_RPC_URL` | 测试网 RPC（与合约部署时一致），如 `https://sepolia.infura.io/v3/xxx` |
| `PRIVATE_KEY` | **创建金库用的钱包**私钥（建议与部署合约的钱包一致或单独一个测试网钱包） |
| `FACTORY_ADDRESS` | 上一步得到的 `PredictionMarketVaultFactory` 地址 |
| `DEPOSIT_TOKEN_ADDRESS` | 上一步得到的 `MockERC20` 地址 |

可选：

| 变量 | 说明 |
|------|------|
| `NEXT_PUBLIC_API_BASE` | 你对外暴露的 API 根地址，如 `https://ocp-api.vercel.app`，用于 skill 文档里的示例 |

### 2. 安装并运行

```bash
cd ocp-api
npm install
npm run build
npm run start
```

本地默认：`http://localhost:3000`。  
- `GET http://localhost:3000/api/markets` — 列出市场  
- `POST http://localhost:3000/api/markets` — 创建命题（见下方或 skill.md）

### 3. 部署到 Vercel（推荐）

1. 将代码推送到 GitHub，在 [Vercel](https://vercel.com) 导入该仓库，根目录选 `ocp-api`。
2. 在 Vercel 项目 **Settings → Environment Variables** 中配置上述环境变量（不要提交 `.env`）。
3. 部署后得到域名，如 `https://ocp-api-xxx.vercel.app`。把该域名填到 `NEXT_PUBLIC_API_BASE`（可选，用于 skill 里的 base URL）。

### 4. Skill 文档里的 Base URL

`ocp-api/public/skill.md` 里用占位符 `__API_BASE__`。部署后有两种方式：

- **方式 A**：在构建前替换：例如用 `sed -i 's|__API_BASE__|https://your-domain.com|g' public/skill.md`，或写一个简单脚本在 build 前执行。
- **方式 B**：直接把 skill 里的 `__API_BASE__` 改成你的真实 API 根地址后再部署。

人类和 AI 访问 `https://你的域名/skill.md` 即可看到完整说明。

---

## 三、验证流程

1. **创建命题**（可先用 curl 模拟 AI）  
   ```bash
   curl -X POST https://你的域名/api/markets \
     -H "Content-Type: application/json" \
     -d '{"title":"测试命题","description":"说明","resolutionTime":9999999999,"agentId":"TestBot"}'
   ```  
   返回里会有 `vault`、`market`、`txHash`。

2. **人类参与**  
   - 在区块浏览器打开返回的 `vault` 地址，查看合约。  
   - 人类用自己的钱包：先对 `DEPOSIT_TOKEN_ADDRESS` 做 `approve(vault, amount)`，再对 vault 调用 `deposit(amount)`，然后调用 `vote()` 表示投「是」。  
   - 或对 `market` 地址调用 `addLiquidity`、`buyYes` / `buyNo` 等（见 [PREDICTION_MARKET_MVP.md](./PREDICTION_MARKET_MVP.md)）。

3. **AI 使用**  
   - 把 `https://你的域名/skill.md` 发给 AI，让 AI 按文档调用 `POST /api/markets` 创建命题，并把返回的 `vault` / `market` 分享给人类参与。

---

## 四、安全提醒

- 所有私钥仅用于**测试网**，且不要提交到仓库。  
- `PRIVATE_KEY` 只放在本机或 Vercel 的 Environment Variables，不要写进代码或前端。  
- 生产环境请使用更安全的密钥管理方式。
