# OCP 测试网部署指南

实现「AI 调 API 在测试网创建命题金库，人类参与投票」的完整步骤。

> 当前工厂版本只创建金库（vault），不创建预测市场（market）。

---

## 零、Base 测试网（Base Sepolia）添加与水龙头

### 在钱包里添加 Base Sepolia

在 MetaMask（或支持自定义网络的钱包）中：

1. 打开 **设置 → 网络 → 添加网络**（或「添加网络」/「Add network」）。
2. 选择「手动添加网络」并填写下表：

| 字段 | 值 |
|------|-----|
| **网络名称** | Base Sepolia |
| **RPC URL** | `https://sepolia.base.org` |
| **Chain ID** | `84532` |
| **货币符号** | ETH |
| **区块浏览器** | `https://sepolia.basescan.org` |

保存后切换到该网络即可。

### Base Sepolia 水龙头（领测试 ETH）

测试网 gas 需要 Base Sepolia 上的 ETH。**若被 Alchemy 等提示「主网无历史」**，优先用下面「不需主网历史」的那几个。

**不需主网历史 / 不需登录（连钱包即可或极简验证）：**

| 水龙头 | 链接 | 说明 |deployerPrivateKey
|--------|------|------|
| **Bware Labs** | https://bwarelabs.com/faucets | 无需注册，选 Base Sepolia，24 小时一次 |
| **Ponzifun** | https://testnet.ponzi.fun/faucet | 无验证、无注册，约 1 ETH / 48 小时 |
| **Ethereum Ecosystem** | https://www.ethereum-ecosystem.com/faucets/base-sepolia | 无需登录，约 0.5 ETH / 24 小时 |
| **Chainlink** | https://faucets.chain.link/base-sepolia | 连钱包 + 选 Base Sepolia，无主网要求 |
| **QuickNode** | https://faucet.quicknode.com/base/sepolia | 连钱包，每 12 小时一次 |
| **thirdweb** | https://thirdweb.com/base-sepolia-testnet | 支持钱包或社交登录，24 小时一次 |
| **ethfaucet.com** | https://ethfaucet.com/networks/base | 选 Base Sepolia，24 小时限一次 |

**可能需要登录或主网条件：**

| 水龙头 | 链接 | 说明 |
|--------|------|------|
| **Coinbase Developer** | https://portal.cdp.coinbase.com/products/faucet | Coinbase 开发者账号，24 小时一次 |
| **Superchain (Optimism)** | https://app.optimism.io/faucet | 选 Base，有链上身份可领更多 |
| **Alchemy** | https://www.alchemy.com/faucets/base-sepolia | 常要求主网有 0.001 ETH 和交易历史 |
| **LearnWeb3** | https://learnweb3.io/faucets/base_sepolia/ | GitHub 登录，约 0.01 ETH/天 |

领到测试 ETH 后即可在 Base Sepolia 上部署合约或与前端交互。

### 当前已部署（Base Sepolia）

| 合约 | 地址 |
|------|------|
| MockERC20 (deposit token) | `0x7D157084E2A7dE4941F29B873bA14a9286f9BE85` |
| OCPVaultFactory | `0x5D158cd6983b00e2D1367969F3AD0412B23794a5` |

ocp-api 的 `.env` 与前端 `ocp-browser/.env` 已按上表配置（FACTORY_ADDRESS / VITE_FACTORY_ADDRESS、DEPOSIT_TOKEN_ADDRESS / VITE_DEPOSIT_TOKEN_ADDRESS）。

### 重新部署合约并更新前端/API 地址

在本机执行（需已 `source solidity/.env` 且 `PRIVATE_KEY` 已设）：

```bash
cd solidity
source .env
forge script script/Deploy.s.sol:DeployScript --rpc-url https://sepolia.base.org --broadcast
```

终端会输出两行，例如：

- `MockERC20 (deposit token): 0x...`
- `OCPVaultFactory: 0x...`

**更新前端**：把上述两个地址填入 `ocp-browser/.env`：

- `VITE_FACTORY_ADDRESS=<工厂地址>`
- `VITE_DEPOSIT_TOKEN_ADDRESS=<MockERC20 地址>`

**更新 API**：在 `ocp-api/.env` 中填入：

- `FACTORY_ADDRESS=<工厂地址>`
- `DEPOSIT_TOKEN_ADDRESS=<MockERC20 地址>`

前端金库 ABI 已使用 `depositAndVote(uint256 amount, bool isYes)`，无需再改。

**领测试代币（Mock 未验证时）**：前端探索页在「当前存款代币」为上述 Mock OCPT 时会显示「领 OCPT 测试代币」按钮，点击即可给自己 mint 1000 OCPT（需少量 Base Sepolia ETH 付 gas），无需在区块浏览器验证合约。

### 验证 MockERC20 合约（可选）

若希望在 BaseScan 上显示 Read/Write Contract 界面，可用 Foundry 提交验证（需 BaseScan API Key，在 https://sepolia.basescan.org/myapikey 创建）：

```bash
cd solidity
export ETHERSCAN_API_KEY=你的_BaseScan_API_Key
forge verify-contract 0xd6f268AF3C4C4Dd7852f51aedd9De12e7048Ec73 script/Deploy.s.sol:MockERC20 --chain-id 84532 --watch
```

验证通过后，在 https://sepolia.basescan.org/address/0xd6f268AF3C4C4Dd7852f51aedd9De12e7048Ec73#writeContract 即可在浏览器里直接调用 `mint`。

### 验证金库与预测市场合约（Base Sepolia）

在 `solidity` 目录下执行（需已设置 `ETHERSCAN_API_KEY`，如 `source .env`）：

**金库 OCPVault**（地址 `0x<YOUR_VAULT>`）：

```bash
cd solidity && source .env
forge verify-contract 0x<YOUR_VAULT> src/core/OCPVault.sol:OCPVault --chain base-sepolia --constructor-args $(cast abi-encode "constructor(address,address,uint256)" 0x32eB2a7d95301a1B7f42Ad64fd255D2c273f6f8f 0xd6f268AF3C4C4Dd7852f51aedd9De12e7048Ec73 1770436800) --watch
```

**预测市场 PredictionMarket**（地址 `0x<YOUR_MARKET>`）：

```bash
forge verify-contract 0x<YOUR_MARKET> src/market/PredictionMarket.sol:PredictionMarket --chain base-sepolia --constructor-args $(cast abi-encode "constructor(address,address,uint8,bytes,uint256)" 0x<YOUR_VAULT> 0x0000000000000000000000000000000000000000 0 0x 1770436800) --watch
```

验证成功后，在 https://sepolia.basescan.org 搜索上述地址即可看到源码与 Read/Write Contract。

### 部署到 Base Sepolia 时用的 RPC

部署合约时 `--rpc-url` 使用（任选其一）：

```text
https://sepolia.base.org

前端/API 若接 Base Sepolia，环境变量示例：

- `VITE_CHAIN_ID=84532`
- `VITE_RPC_URL=https://sepolia.base.org`
- `VITE_EXPLORER=https://sepolia.basescan.org`
- `VITE_MARKET_ENABLED=true`（开启预测市场功能；不设置则默认关闭）

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

**以太坊 Sepolia：**

```bash
cd solidity
export PRIVATE_KEY=0x你的测试网钱包私钥
forge script script/Deploy.s.sol:DeployScript --rpc-url https://sepolia.infura.io/v3/YOUR_KEY --broadcast
```

**Base Sepolia：**（RPC 无需 API key，先到「零、Base 测试网」领水龙头）

在 `solidity` 目录下，设置私钥后执行（二选一）：

**重要**：Foundry 只读取 **`solidity/.env`**（与 `foundry.toml` 同级），不会读 `solidity/src/.env`。若你把私钥写在 `src/.env`，请改写到 `solidity/.env` 或执行：
`grep PRIVATE_KEY solidity/src/.env >> solidity/.env` 后编辑 `solidity/.env` 只保留一行正确的 `PRIVATE_KEY=0x...`。

```bash
cd solidity
# 方式一：直接导出（仅当前终端有效）
export PRIVATE_KEY=0x你的测试网钱包私钥
# 方式二：使用 .env（在 solidity/.env 中写 PRIVATE_KEY=0x...，不要用 src/.env）
source .env

forge script script/Deploy.s.sol:DeployScript --rpc-url https://sepolia.base.org --broadcast
```

- Sepolia 的 `--rpc-url` 可换成任意 Sepolia RPC（如 Alchemy、Public 等）。
- 部署成功后终端会打印：
  - **MockERC20 (deposit token):** `0x...`
  - **OCPVaultFactory:** `0x...`

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
| `FACTORY_ADDRESS` | 上一步得到的 `OCPVaultFactory` 地址 |
| `DEPOSIT_TOKEN_ADDRESS` | 上一步得到的 `MockERC20` 地址 |

说明：
- `ocp-api` 不再保存统一后端 `PRIVATE_KEY`。
- 每个 AI agent 在请求里携带自己的私钥（`agentPrivateKey` 或 `X-Agent-Private-Key`）完成链上签名。

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
- `GET http://localhost:3000/api/markets` — 列出金库  
- `POST http://localhost:3000/api/markets` — 创建命题（见下方或 skill.md）
- `POST http://localhost:3000/api/markets/:id/stake` — Agent 对 vault 地址参与质押

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
     -d '{"title":"测试命题","description":"说明","resolutionTime":9999999999,"agentId":"TestBot","agentPrivateKey":"0x测试网私钥"}'
   ```  
   返回里会有 `vault`、`txHash`。

2. **人类参与**  
   - 在区块浏览器打开返回的 `vault` 地址，查看合约。  
   - 人类用自己的钱包：先对 `DEPOSIT_TOKEN_ADDRESS` 做 `approve(vault, amount)`，再对 vault 调用 `stake(side, amount)`。  
   - 挑战期可调用 `moveStake(fromSide, toSide, amount)`；可结算时调用 `finalize()`；终局后调用 `withdraw()`。

3. **AI 使用**  
   - 把 `https://你的域名/skill.md` 发给 AI，让 AI 按文档调用 `POST /api/markets` 创建命题，并把返回的 `vault` 分享给人类参与。  
   - AI 可继续调用 `POST /api/markets/{vault地址}/stake|move-stake|finalize|withdraw` 参与同一套玩法（同样需提供 `agentPrivateKey`）。

---

## 四、前端（ocp-browser）发布到 Vercel

**只发主页时**：不用部署合约、不用部署 ocp-api，只把前端直接托管到 Vercel 即可。

### 方式 A：用 GitHub 部署（推荐，推送后自动部署）

1. 把项目推送到 GitHub（若尚未推送）：
   ```bash
   git add -A && git commit -m "your message" && git push origin main
   ```
2. 用 GitHub 登录 [Vercel](https://vercel.com)，**Add New → Project**，选择仓库 `cypherpunk-bc/OCP`（或你的 fork）。
3. **重要**：在 **Configure Project** 里把 **Root Directory** 设为 `ocp-browser`（点 Edit 后输入 `ocp-browser`）。
4. **Framework Preset** 选 Vite；**Build Command** 留空或 `npm run build`；**Output Directory** 留空或 `dist`。`ocp-browser/vercel.json` 已写好，一般无需改。
5. 如需测试网配置，在 **Environment Variables** 里添加 `VITE_FACTORY_ADDRESS`、`VITE_DEPOSIT_TOKEN_ADDRESS`、`VITE_RPC_URL`、`VITE_CHAIN_ID`、`VITE_EXPLORER` 等（与 `ocp-browser/.env.example` 一致）。
6. 点击 **Deploy**。部署完成后会得到 `https://xxx.vercel.app`，之后每次 push 到该仓库会自动重新部署。

### 方式 B：本地用 Vercel CLI 部署（无需 GitHub）

不推前端代码到 GitHub 也可以发布，在本地用 CLI 直接上传构建结果即可。

1. 进入前端目录并**先登录**（否则会报 token 无效）：
   ```bash
   cd ocp-browser
   npx vercel login    # 按提示用浏览器或邮箱登录，完成后再继续
   ```
2. 构建并部署：
   ```bash
   npm run build
   npx vercel --prod   # 首次会问项目名等，选默认即可
   ```
3. 终端会给出预览/生产域名（如 `https://ocp-browser-xxx.vercel.app`）。之后要更新站点时，在 `ocp-browser` 下再执行 `npm run build` 和 `npx vercel --prod` 即可。

绑定自定义域名仍在 Vercel 对应项目的 **Settings → Domains** 里操作，与是否用 GitHub 无关。

---

## 五、在 Vercel 绑定自己的域名

1. 在 Vercel 打开对应项目（ocp-api 或 ocp-browser），进入 **Settings → Domains**。
2. 在 **Domain** 输入框填入你的域名（如 `app.yourdomain.com` 或 `yourdomain.com`），点击 **Add**。
3. 按 Vercel 提示在域名服务商处添加 DNS 记录：
   - **推荐（CNAME）**：  
     类型 `CNAME`，名称填子域名（如 `app` 或 `www`），目标填 `cname.vercel-dns.com`。  
     根域名（如 `yourdomain.com`）若要用 Vercel，类型选 `A`，目标填 `76.76.21.21`（或按 Vercel 页面上显示的 IP）。
   - **或用 Vercel DNS**：若域名在 Vercel 购买或已把 NS 指到 Vercel，在 **Domains** 里可一键验证。
4. 保存 DNS 后等待生效（几分钟到 48 小时不等）。**Domains** 里状态变为 **Valid** 即绑定成功，Vercel 会自动为域名签发 HTTPS。

---

## 六、安全提醒

- 所有私钥仅用于**测试网**，且不要提交到仓库。  
- Agent 私钥仅通过请求动态提供（建议只在 HTTPS + 临时会话下使用），不要写进代码、前端或仓库。  
- 生产环境请使用更安全的密钥管理方式。
