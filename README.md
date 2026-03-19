# Freedom Trader 技术文档

> BSC + Solana 双链聚合交易终端 — Chrome 侧边栏扩展
> 版本: 2.1.0 | 合约版本: FreedomRouter v6

---

## 目录

1. [架构概览](#1-架构概览)
2. [BSC 交易流程](#2-bsc-交易流程)
3. [Solana 交易流程](#3-solana-交易流程)
4. [安全模型](#4-安全模型)
5. [多钱包批量交易](#5-多钱包批量交易)
6. [RPC 策略](#6-rpc-策略)
7. [代币检测与自动路由](#7-代币检测与自动路由)
8. [合约源码分析](#8-合约源码分析)
9. [关键代码路径](#9-关键代码路径)

---

## 1. 架构概览

### 1.1 整体架构

Freedom Trader 是一个 Chrome Manifest V3 侧边栏扩展，采用三层架构：

```
┌──────────────────────────────────────────────────────┐
│                    Chrome Side Panel                  │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐ │
│  │ trader.html  │  │settings.html│  │  theme.js    │ │
│  │  (交易UI)    │  │ (设置页面)   │  │ (主题切换)   │ │
│  └──────┬───────┘  └──────┬──────┘  └──────────────┘ │
│         │                 │                           │
│  ┌──────┴─────────────────┴──────────────────────┐   │
│  │             前端模块层 (ES Modules)             │   │
│  │  trader.js → batch.js → trading.js (BSC)      │   │
│  │                      → sol-trading.js (SOL)    │   │
│  │  token.js → token-bsc.js / token-sol.js       │   │
│  │  wallet.js → wallet-bsc.js / wallet-sol.js    │   │
│  │  ui.js / state.js / utils.js / lock.js        │   │
│  │  crypto.js (消息代理层)                         │   │
│  └──────────────────┬────────────────────────────┘   │
│                     │ chrome.runtime.sendMessage      │
│  ┌──────────────────┴────────────────────────────┐   │
│  │        Background Service Worker               │   │
│  │  background.js                                 │   │
│  │  - PBKDF2+AES-256-GCM 加密/解密               │   │
│  │  - BSC: viem WalletClient 签名                 │   │
│  │  - SOL: @solana/web3.js Keypair 签名           │   │
│  │  - Jito Bundle 广播                            │   │
│  │  - 合约地址自动识别 (URL 匹配)                  │   │
│  └───────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────────┐
    │ BSC RPC  │   │ SOL RPC  │   │ Jito Block   │
    │ Nodes    │   │ Nodes    │   │ Engines      │
    └─────┬────┘   └─────┬────┘   └──────────────┘
          ▼               ▼
    ┌──────────┐   ┌──────────────────────┐
    │Freedom   │   │ Pump.fun Program     │
    │Router    │   │ PumpSwap AMM Program │
    │(BSC合约) │   └──────────────────────┘
    └──────────┘
```

### 1.2 模块关系

| 模块 | 文件 | 职责 |
|------|------|------|
| **入口** | `trader.js` | 初始化、链切换、配置加载、事件绑定 |
| **状态** | `state.js` | 全局状态对象，BSC/SOL 钱包、余额、代币信息 |
| **加密代理** | `crypto.js` | 前端→Background 消息代理，所有签名操作转发 |
| **后台** | `background.js` | Service Worker，私钥存储/解密，交易签名发送 |
| **BSC 交易** | `trading.js` | BSC 买入/卖出逻辑，approve 管理，FreedomRouter 调用 |
| **SOL 交易** | `sol-trading.js` → `sol/trading.js` | SOL 买入/卖出，Bonding Curve/PumpSwap 路由 |
| **批量交易** | `batch.js` | 多钱包并行执行，快速买入/卖出 |
| **代币检测** | `token.js` → `token-bsc.js` / `token-sol.js` | 代币状态检测，内外盘判断，UI 更新 |
| **钱包管理** | `wallet.js` → `wallet-bsc.js` / `wallet-sol.js` | 钱包初始化、余额加载、选择器渲染 |
| **UI** | `ui.js` | 报价预估、滑点控制、快捷按钮、链切换 UI |
| **工具** | `utils.js` | 地址验证、金额格式化、超时控制 |
| **锁屏** | `lock.js` | 密码解锁流程 |
| **设置** | `settings.js` | 钱包导入（单个/批量）、RPC 配置、密码管理 |

### 1.3 数据流

```
用户输入代币地址
    ↓
token.js#detectToken() → 根据 currentChain 分发
    ↓                          ↓
token-bsc.js#detectBscToken() token-sol.js#detectSolToken()
    ↓                          ↓
FreedomRouter.getTokenInfo()  sol/trading.js#detectToken()
    ↓                          ↓
返回 routeSource/approveTarget 返回 bonding-curve/pumpswap
    ↓                          ↓
state.lpInfo 更新              state.lpInfo 更新
    ↓
UI 显示内盘/外盘/协议标签
    ↓
用户点击买入/卖出
    ↓
batch.js#executeBatchTrade()
    ↓
Promise.allSettled(多钱包并行)
    ↓                          ↓
trading.js#buy/sell()         sol-trading.js#solBuy/solSell()
    ↓                          ↓
crypto.js#bscWriteContract()  sol/trading.js#buildAndSend()
    ↓                          ↓
background.js 签名+发送        crypto.js#solSignAndSend()
                               → background.js 签名
                               → RPC + Jito 双发
```

### 1.4 构建流程

使用 esbuild 打包（`scripts/build.js`），三个入口点：

- `trader.js` → `dist/src/trader.js` (IIFE)
- `settings.js` → `dist/src/settings.js` (IIFE)
- `background.js` → `dist/background.js` (IIFE)

构建目标 `chrome120`，HTML 中 `type="module"` 在构建时被移除。

---

## 2. BSC 交易流程

### 2.1 FreedomRouter 合约概述

合约地址: `0xCd4D70bb991289b5A8522adB93Cd3C4b93B4Dceb`

FreedomRouterImpl v6 是一个 UUPS 可升级代理合约，继承自：
- `Ownable` — 所有者权限控制
- `ReentrancyGuard` — 防重入攻击
- `Initializable` — 初始化模式
- `UUPSUpgradeable` — 可升级代理

核心外部依赖：
- `tokenManagerV2` (`0x5c952063c7fc8610FFDB798152D69F0B9550762b`) — Four.meme 代币管理合约
- `tmHelper3` (`0xF251F83e40a78868FcfA3FA4599Dad6494E46034`) — Four.meme Helper3 合约
- `flapPortal` (`0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0`) — Flap Portal 合约
- PancakeSwap Router (`0x10ED43C718714eb63d5aA57B78B54704E256024E`) — 外盘 DEX

### 2.2 路由源枚举 (RouteSource)

```solidity
enum RouteSource {
    NONE,                // 0 - 无路由
    FOUR_INTERNAL_BNB,   // 1 - Four.meme 内盘 BNB 池
    FOUR_INTERNAL_ERC20, // 2 - Four.meme 内盘 ERC20 (如 USD1) 池
    FOUR_EXTERNAL,       // 3 - Four.meme 已毕业，走 PancakeSwap
    FLAP_BONDING,        // 4 - Flap 联合曲线（内盘），可买可卖
    FLAP_BONDING_SELL,   // 5 - Flap 联合曲线，ERC20 quote + nativeSwap 未开启，仅可卖
    FLAP_DEX,            // 6 - Flap 已迁移到 DEX，走 PancakeSwap
    PANCAKE_ONLY         // 7 - 纯 PancakeSwap
}
```

### 2.3 路由检测逻辑 (`_detectRoute`)

`FreedomRouter.sol:140-199`

检测优先级：**Four.meme → Flap → PancakeSwap**

```
_detectRoute(token):
  ① 调用 IHelper3.getTokenInfo(token)
     - ver > 0 且 tm != address(0):
       - liquidityAdded = true  → FOUR_EXTERNAL（已毕业，走 PCS）
       - liquidityAdded = false:
         - _mode() != 0:
           - quote == address(0) → FOUR_INTERNAL_BNB
           - quote != address(0) → FOUR_INTERNAL_ERC20

  ② 调用 IFlapPortal.getTokenV7(token)
     - status == 1 (Tradable):
       - quoteToken == address(0) 或 nativeToQuoteSwapEnabled
         → FLAP_BONDING（可买可卖）
       - 否则 → FLAP_BONDING_SELL（仅可通过 Portal 卖，买走 PCS）
     - status == 4 (DEX):
       → FLAP_DEX（走 PancakeSwap V2）

  ③ findBestQuote(token)
     - 遍历 [WBNB, USDT, USD1, USDC, BUSD, FDUSD]
     - 选择流动性最高的配对 → PANCAKE_ONLY

  ④ 以上都未匹配 → NONE
```

### 2.4 内盘/外盘判断标准

**Four.meme 内盘判断**:
- `IHelper3.getTokenInfo()` 返回 `version > 0` 且 `tokenManager != address(0)` 且 `liquidityAdded == false`
- `IFourToken._mode()` 返回值不为 0（代币仍在交易中）

**Four.meme 外盘判断**:
- `liquidityAdded == true`（已添加流动性到 PancakeSwap）

**Flap 内盘判断**:
- `IFlapPortal.getTokenV7()` 返回 `status == 1`（Tradable 状态）

**Flap 外盘判断**:
- `status == 4`（已迁移到 DEX）

### 2.5 买入流程

`FreedomRouter.sol:230-253` — `buy()` 函数

```
buy(token, amountOutMin, tipRate, deadline):
  ① require(block.timestamp <= deadline)
  ② require(msg.value > 0)
  ③ _detectRoute(token) → route
  ④ netValue = _deductTip(msg.value, tipRate)  // 扣除小费
  ⑤ 根据 route 分发:

  FOUR_INTERNAL_BNB / FOUR_INTERNAL_ERC20:
    → _buyFourInternal(token, amountOutMin, netValue)
      - BNB 池: ITMV2.buyTokenAMAP{value}(token, msg.sender, value, amountOutMin)
      - ERC20 池: IHelper3.buyWithEth{value}(0, token, msg.sender, value, amountOutMin)
      - 使用 balanceOf 差值计算实际获得量

  FOUR_EXTERNAL / PANCAKE_ONLY / FLAP_DEX / FLAP_BONDING_SELL:
    → _buyPancake(token, amountOutMin, netValue, deadline)
      - findBestQuote() 找最佳配对
      - 构建路径: [WBNB, token] 或 [WBNB, quote, token]
      - 优先尝试 swapExactETHForTokensSupportingFeeOnTransferTokens
      - 失败则降级到 swapExactETHForTokens
      - 使用 balanceOf 差值兼容 tax token

  FLAP_BONDING:
    → _buyFlap(token, amountOutMin, netValue)
      - IFlapPortal.swapExactInput{value}(inputToken=0, outputToken=token, ...)
      - 路由收到的代币转给用户
      - 退还多余 ETH
```

### 2.6 卖出流程

`FreedomRouter.sol:255-274` — `sell()` 函数

```
sell(token, amountIn, amountOutMin, tipRate, deadline):
  根据 route 分发:

  FOUR_INTERNAL_BNB / FOUR_INTERNAL_ERC20:
    → _sellFourInternal(token, amountIn, amountOutMin, tipRate)
      - BNB 池: ITMV2.sellToken(0, token, msg.sender, amountIn, amountOutMin, rate, DEV)
        (Four.meme 的 sellToken 原生支持 feeRate 参数)
      - ERC20 池: IHelper3.sellForEth(0, token, msg.sender, amountIn, 0, rate, DEV)
      - 使用 msg.sender.balance 差值计算净收益

  FOUR_EXTERNAL / PANCAKE_ONLY / FLAP_DEX:
    → _sellPancakeCompat(token, amountIn, tipRate, deadline)
      - safeTransferFrom 将代币转入路由
      - 使用 balanceOf 差值获取实际到账量（兼容 tax token）
      - forceApprove 给 PancakeRouter
      - 优先 swapExactTokensForETHSupportingFeeOnTransferTokens
      - 扣除小费后将 BNB 发送给用户

  FLAP_BONDING / FLAP_BONDING_SELL:
    → _sellFlap(token, amountIn, amountOutMin, tipRate)
      - safeTransferFrom 代币到路由
      - forceApprove 给 flapPortal
      - IFlapPortal.swapExactInput(inputToken=token, outputToken=0)
      - 扣除小费后发送 BNB 给用户
```

### 2.7 前端调用路径 (BSC)

`trading.js:150-205` — 前端 `buy()` 函数

```javascript
buy(walletId, tokenAddr, amountStr, gasPrice):
  ① normalizeAmount() 规范化金额
  ② parseUnits(amount, quoteDecimals) 转为 BigInt
  ③ getTipRate() 获取小费比率（0-500 bps）
  ④ _getQuoteBuy() 获取预期产出量
  ⑤ amountOutMin = estimated * (100-slippage)% 计算滑点保护
  ⑥ ensureApproved() 确保 quote token 已授权（非 BNB 时）
  ⑦ bscWriteContract() → background.js 签名发送
     调用 FreedomRouter.trade(quoteAddr, tokenAddr, amountIn, amountOutMin, tipRate, deadline)
  ⑧ waitForTransactionReceipt() 等待确认
  ⑨ 买入后自动 approve（为后续卖出做准备）
```

**注意**: 前端 ROUTER_ABI 中使用的是统一的 `trade()` 函数签名，而合约中实际有独立的 `buy()` 和 `sell()` 函数。前端通过 `trade(tokenIn, tokenOut, amountIn, amountOutMin, tipRate, deadline)` 调用，其中 tokenIn/tokenOut 的方向决定买卖：
- 买入: `trade(BNB_sentinel, token, 0, minOut, tip, deadline)` + msg.value
- 卖出: `trade(token, BNB_sentinel, amountIn, minOut, tip, deadline)`

### 2.8 小费机制

```solidity
MAX_TIP = 500;  // 最大 5%

_calcTip(amount, tipRate):
  rate = min(tipRate, MAX_TIP)
  return amount * rate / 10000

_deductTip(amount, tipRate):
  tip = _calcTip(amount, tipRate)
  if tip > 0: _sendBNB(DEV, tip)  // DEV = 0x2De78...
  return amount - tip
```

前端默认 `DEFAULT_TIP_RATE = 0`（完全自愿），用户可在设置中配置 0-5%。

### 2.9 Approve 缓存机制

`trading.js:10-66`

```
ensureApproved(walletId, owner, tokenAddr, spender, gasPrice, minAllowance, source):
  ① 检查内存缓存 state.approvedTokens (Set)
  ② 检查 approvalInFlight (Map) 防止并发重复 approve
  ③ 链上读取 allowance
  ④ 如果 allowance >= MAX_HALF → markApproved() 缓存到 chrome.storage.local
  ⑤ 如果不足 → approve(MAX_UINT256)，等待确认后缓存
```

持久化键: `approvedTokens` (字符串数组: `"owner:token:spender"`)

---

## 3. Solana 交易流程

### 3.1 架构概述

Solana 侧支持两种交易模式:
1. **Pump.fun Bonding Curve** — 内盘，代币未毕业时的联合曲线交易
2. **PumpSwap AMM** — 外盘，代币毕业后迁移到 Constant Product AMM

核心程序地址:
- Pump.fun Program: `6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P`
- PumpSwap AMM: `pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA`
- Pump Fee Program: `pfeeUxB6jkeY1Hxd7CsFCAjcbHA9rWtchMGdZ6VojVZ`

### 3.2 代币检测流程

`sol/trading.js:51-108` — `detectToken()`

```
detectToken(mintAddress):
  ① getBondingCurve(mint) — 读取 Bonding Curve PDA 账户
  ② getTokenProgram(mint) — 判断 SPL Token 还是 Token-2022

  if bc.complete == false:
    → 返回 { type: 'bonding-curve', ... }  // 内盘

  if bc.complete == true:
    ③ findPoolForMint(mint) — 查找 PumpSwap AMM Pool
    ④ getAmmGlobalConfig() — 获取 AMM 全局配置
    ⑤ warmDynamicFeeConfig() — 预热动态费率配置

    if pool found:
      → 返回 { type: 'pumpswap', pool, ... }  // 外盘
    else:
      → 返回 { type: 'completed-no-pool', ... }  // 已毕业但无池子
```

### 3.3 Bonding Curve 账户布局

`sol/accounts.js:7-31`

```
Bonding Curve Account (after 8-byte Anchor discriminator):
  offset 0:  virtual_token_reserves  u64
  offset 8:  virtual_sol_reserves    u64
  offset 16: real_token_reserves     u64
  offset 24: real_sol_reserves       u64
  offset 32: token_total_supply      u64
  offset 40: complete                bool (u8)
  offset 41: creator                 Pubkey (32 bytes)
  offset 73: is_mayhem_mode          bool (u8)
```

### 3.4 Bonding Curve 买入报价

`sol/bonding-curve.js:27-49` — `calcBcBuyQuoteSync()`

```
calcBcBuyQuoteSync(spendableSolIn, bc, fee, slippagePct):
  ① totalFeeBps = protocolFeeBps + creatorFeeBps
  ② netSol = solIn * 10000 / (10000 + totalFeeBps)  // 扣除手续费
  ③ 精确计算 protocolFee 和 creatorFee（向上取整）
  ④ effectiveSol = netSol - 1  // 留 1 lamport 余量
  ⑤ tokensOut = effectiveSol * virtualTokenReserves / (virtualSolReserves + effectiveSol)
     // 恒积公式: x * y = k
  ⑥ minTokensOut = tokensOut * (100 - slippagePct) * 100 / 10000
```

### 3.5 Bonding Curve 卖出报价

`sol/bonding-curve.js:56-82` — `calcBondingCurveSellQuote()`

```
calcBondingCurveSellQuote(tokenAmount, bc, feeConfig, slippagePct):
  ① 如果 amount > realTokenReserves → cap 到 realTokenReserves
  ② 如果 realSolReserves <= 0 → 返回 0（池子空了）
  ③ grossSol = amount * virtualSolReserves / (virtualTokenReserves + amount)
  ④ 如果 grossSol > realSolReserves → cap 到 realSolReserves
  ⑤ protocolFee = (grossSol * protocolFeeBps + 9999) / 10000  // 向上取整
  ⑥ creatorFee = (grossSol * creatorFeeBps + 9999) / 10000
  ⑦ netSol = grossSol - protocolFee - creatorFee
  ⑧ minSolOut = netSol * (100 - slippagePct) * 100 / 10000
```

### 3.6 PumpSwap AMM 买入

`sol/pump-swap.js:45-64` — `calcAmmBuyQuote()`

```
calcAmmBuyQuote(spendableQuoteIn, baseReserve, quoteReserve, totalFeeBps, slippagePct):
  ① netQuote = quoteIn * 10000 / (10000 + feeBps)  // 扣费后净输入
  ② baseOut = netQuote * baseReserve / (quoteReserve + netQuote)  // 恒积公式
  ③ minBaseOut = baseOut * (100 - slippage)%     // 滑点下限（接受更少代币）
  ④ maxQuoteIn = quoteIn * (100 + slippage)%     // 滑点上限（允许支付更多）
```

AMM 动态费率:
- 根据 marketCap 分段查找费率（`lookupFeeTier()`）
- 默认 125bps（1.25%），包含 LP 费 + 协议费 + 创作者费

### 3.7 PumpSwap AMM 卖出

`sol/pump-swap.js:66-78` — `calcAmmSellQuote()`

```
calcAmmSellQuote(baseAmountIn, baseReserve, quoteReserve, totalFeeBps, slippagePct):
  ① grossQuote = amountIn * quoteReserve / (baseReserve + amountIn)
  ② fee = grossQuote * feeBps / 10000
  ③ netQuote = grossQuote - fee
  ④ minQuoteOut = netQuote * (100 - slippage)% / 10000
```

### 3.8 交易指令构建

#### Bonding Curve 买入指令

`sol/bonding-curve.js:84-125` — 17 个账户:

| 索引 | 账户 | 说明 |
|------|------|------|
| 0 | PUMP_GLOBAL | 全局配置 |
| 1 | randomFeeRecipient() | **随机选择**的手续费接收者 |
| 2 | mint | 代币 Mint |
| 3 | bondingCurve PDA | Bonding Curve 状态 |
| 4 | associatedBondingCurve | BC 的代币账户 |
| 5 | associatedUser | 用户的代币 ATA |
| 6 | user | 交易发起者（签名者） |
| 7 | SystemProgram | |
| 8 | tokenProgram | SPL Token / Token-2022 |
| 9 | creatorVault | 创建者保险库 |
| 10 | eventAuthority | 事件权限 PDA |
| 11 | PUMP_PROGRAM | |
| 12 | globalVolumeAccumulator | 全局交易量统计 |
| 13 | userVolumeAccumulator | 用户交易量统计 |
| 14 | feeConfig | 费率配置 PDA |
| 15 | PUMP_FEE | 费率程序 |
| 16 | bondingCurveV2 PDA | V2 版本 |

指令数据: `discriminator(8) + solIn(u64) + minTokensOut(u64) + optionBoolFalse(2)`

#### PumpSwap AMM 买入指令

`sol/pump-swap.js:145-200` — 24 个账户:

核心差异：需要 WSOL ATA 包装、Pool 地址、coinCreator vault、poolV2 等额外账户。

指令数据: `discriminator(8) + baseAmountOut(u64) + maxQuoteAmountIn(u64) + optionBoolFalse(2)`

### 3.9 Jito Bundle 双发机制

`background.js:286-344` 和 `sol/trading.js:354-401`

```
buildAndSend(conn, walletId, user, instructions, t0, opts):
  ① 如果 jitoTip > 0:
     → 添加 buildJitoTipIx(user, jitoTip) 到指令列表
     → 从 8 个 Jito Tip 账户中随机选择一个（负载均衡）

  ② 构建 Transaction，设置 recentBlockhash + feePayer
  ③ 序列化为 unsigned base64
  ④ 调用 crypto.js#solSignAndSend() → background.js

background.js#solSignAndSend():
  ① tx.sign(keypair) — 用密钥对签名
  ② 并行双发:
     - sends[0]: conn.sendRawTransaction(raw, { skipPreflight: true, maxRetries: 2 })
       → 标准 RPC 发送
     - sends[1]: sendToJito(rawBase64)
       → 广播到 6 个 Jito Block Engine:
         - mainnet.block-engine.jito.wtf
         - tokyo.mainnet.block-engine.jito.wtf
         - amsterdam.mainnet.block-engine.jito.wtf
         - frankfurt.mainnet.block-engine.jito.wtf
         - ny.mainnet.block-engine.jito.wtf
         - singapore.mainnet.block-engine.jito.wtf
  ③ Promise.all(sends) — 第一个返回的 signature 作为结果
```

Jito Tip 账户（随机选择）：
```
96gYZGLnJYVFmbjzopPSU6QiEV5fGqZNyN9nmNhvrZU5
HFqU5x63VTqvQss8hp11i4bPHBJkRAt1PSdSHpPxKwBP
Cw8CFyM9FkoMi7K7Crf6HNQqf4uEMzpKw6QNghXLvLkY
ADaUMid9yfUytqMBgopwjb2DTLSLBVCYmRxDAsGTKiGb
DfXygSm4jCyNCybVYYK6DwvWqjKee8pbDmJGcLWNDXjh
ADuUkR4vqLUMWXxW9gh6D6L8pMSawimctcNZ5pGwDcEt
DttWaMuVvTiduZRnguLF7jNxTgiMBZ1hyAumKUiL91KV
3AVi9Tg9Uo68tJfuvoKvqKNWKkC5wPdSSdeBnizKZ6jT
```

### 3.10 确认机制

`sol/trading.js:297-340` — `confirmWithFallback()`

```
confirmWithFallback(conn, sig, timeoutMs=60000):
  并行竞争:
  ① WebSocket 订阅: conn.onSignature(sig, callback, 'confirmed')
  ② 轮询: pollConfirmation(conn, sig, timeoutMs)
     - 每 100ms 调用 getSignatureStatus()
     - 检查 confirmationStatus == 'confirmed' 或 'finalized'
  → Promise.race([wsPromise, pollPromise])
  → 先返回的为准，清理另一个监听器
```

### 3.11 Blockhash 预取

`sol/connection.js:51-103`

```
Blockhash Prefetch:
  - 每 2000ms 自动刷新 (BLOCKHASH_REFRESH_MS)
  - 最大缓存年龄 10000ms (BLOCKHASH_MAX_AGE_MS)
  - getBlockhashFast():
    if 缓存存在且 age < 10s → 立即返回（0ms）
    else → 调用 conn.getLatestBlockhash('confirmed')
```

### 3.12 PDA 派生与缓存

`sol/pda.js` — 所有 PDA 计算带 LRU 缓存（256 上限）:

```
Pump.fun PDAs:
  - deriveBondingCurve(mint)    = findPDA(["bonding-curve", mint], PUMP_PROGRAM)
  - deriveBondingCurveV2(mint)  = findPDA(["bonding-curve-v2", mint], PUMP_PROGRAM)
  - deriveCreatorVault(creator) = findPDA(["creator-vault", creator], PUMP_PROGRAM)

PumpSwap AMM PDAs:
  - derivePool(auth, base, quote) = findPDA(["pool", index_buf, auth, base, quote], PUMP_AMM)
  - derivePoolAuthority(baseMint) = findPDA(["pool-authority", baseMint], PUMP_PROGRAM)
  - derivePoolV2(baseMint)        = findPDA(["pool-v2", baseMint], PUMP_AMM)
  - deriveAmmCreatorVault(creator) = findPDA(["creator_vault", creator], PUMP_AMM)

Fee PDAs (编译时常量):
  - deriveBcFeeConfig()  = findPDA(["fee_config", PUMP_PROGRAM], PUMP_FEE)
  - deriveAmmFeeConfig() = findPDA(["fee_config", PUMP_AMM], PUMP_FEE)

通用:
  - deriveATA(owner, mint, tokenProgram) = findPDA([owner, tokenProgram, mint], ATA_PROGRAM)
  - deriveEventAuthority(program) = findPDA(["__event_authority"], program)
  - deriveGlobalVolumeAccumulator(program) = findPDA(["global_volume_accumulator"], program)
  - deriveUserVolumeAccumulator(user, program) = findPDA(["user_volume_accumulator", user], program)
```

---

## 4. 安全模型

### 4.1 私钥加密存储

`background.js:55-66`

**加密算法: PBKDF2 + AES-256-GCM**

```
密钥派生:
  ① 生成 16 字节随机 salt（crypto.getRandomValues），持久化到 chrome.storage.local["ft_salt"]
  ② PBKDF2 参数:
     - salt: 16 bytes
     - iterations: 100,000
     - hash: SHA-256
     - 输出: AES-GCM 256-bit key
  ③ crypto.subtle.deriveKey('PBKDF2' → 'AES-GCM', length=256)

加密过程 (encrypt):
  ① 生成 12 字节随机 IV (crypto.getRandomValues)
  ② AES-256-GCM 加密明文
  ③ 拼接 IV(12) + ciphertext → base64 编码
  ④ 存储格式: "enc:" + base64

解密过程 (decryptCiphertext):
  ① 去除 "enc:" 前缀
  ② base64 解码
  ③ 分离 IV(前12字节) + encrypted(剩余)
  ④ AES-GCM 解密
```

### 4.2 Service Worker 隔离

```
安全边界:
┌─────────────────────────────┐
│   Side Panel (trader.html)  │  ← 无法访问私钥
│   crypto.js (消息代理)       │  ← 只能发送消息
│   chrome.runtime.sendMessage│
└──────────────┬──────────────┘
               │ 消息通道
┌──────────────┴──────────────┐
│   Background Service Worker │  ← 私钥只在这里解密
│   background.js             │
│   ┌───────────────────────┐ │
│   │ bscClients (Map)      │ │  ← walletId → {client, account}
│   │ solKeypairs (Map)     │ │  ← walletId → Keypair
│   │ cachedKey (CryptoKey) │ │  ← AES-GCM 密钥
│   └───────────────────────┘ │
│   前端永远只能拿到:          │
│   - 地址 (string)           │
│   - txHash / signature      │
└─────────────────────────────┘
```

**关键隔离点**:

1. **`crypto.js` 是消息代理，不是加密模块**
   - `bscWriteContract()` 将参数序列化为 JSON，BigInt 通过 `{__bigint: string}` 传输
   - `solSignAndSend()` 将未签名 TX 序列化为 base64 传给后台
   - 前端永远看不到 private key 明文

2. **sender 验证**: `background.js:349` 检查 `sender.id !== chrome.runtime.id` 防止外部扩展调用

3. **Session Storage 限制**: `background.js:21`
   ```javascript
   chrome.storage.session.setAccessLevel({ accessLevel: 'TRUSTED_CONTEXTS' });
   ```
   密码缓存 `ft_pw` 只有 Service Worker 可读，扩展页面无法访问。

### 4.3 密码验证与会话管理

```
密码哈希:
  hashPassword(password):
    PBKDF2(password, salt, 100000 iterations, SHA-256) → 256 bits → base64
    存储到 chrome.storage.local["ft_pw_hash"]

解锁流程:
  unlock(password):
    ① hash = hashPassword(password)
    ② 对比存储的 ft_pw_hash
    ③ 匹配 → deriveKey(password) → cachedKey
    ④ saveSession(password) → chrome.storage.session["ft_pw"]

活动超时:
  checkExpiry():
    ① 如果 cachedKey 为空 → restoreSession() 从 session storage 恢复
    ② 读取 lockDuration (默认 240 分钟)
    ③ if (now - unlockTime > duration) → clearSession()
    ④ 否则 → unlockTime = now (重置活动计时器)
    → 是"不活动超时"而非"解锁后超时"
```

### 4.4 风险点分析

| 风险 | 缓解措施 |
|------|----------|
| Service Worker 被终止后重启 | Session Storage 持久化密码，`restoreSession()` 自动恢复 |
| XSS 注入扩展页面 | CSP 禁止内联脚本，输出使用 `escapeHtml()` |
| 恶意扩展尝试调用 | `sender.id` 验证排除非本扩展消息 |
| 暴力破解密码 | PBKDF2 100,000 iterations 增加计算成本 |
| 私钥在内存中 | 锁定时 `clearSession()` 清除所有密钥和客户端 |
| Salt 泄漏 | Salt 存在 chrome.storage.local，离线攻击仍需暴力破解密码 |

---

## 5. 多钱包批量交易

### 5.1 batch.js 并行机制

`batch.js:67-120` — `executeBatchTrade()`

```javascript
executeBatchTrade():
  ① 获取当前激活的钱包列表
     - BSC: state.activeWalletIds.filter(id => state.walletClients.has(id))
     - SOL: state.solActiveWalletIds.filter(id => state.solAddresses.has(id))

  ② 为每个钱包创建 Promise:
     const promises = activeWallets.map(id =>
       state.tradeMode === 'buy'
         ? doBuy(id, tokenAddr, normalizedAmount)
         : doSell(id, tokenAddr, normalizedAmount)
     );

  ③ Promise.allSettled(promises)
     → 所有钱包的交易并行执行
     → 不会因单个失败而取消其他

  ④ 统计结果:
     - success / failed 计数
     - 记录 buildMs, sendMs, confirmMs 时间
     - 计算批次总耗时
```

### 5.2 快速买入 (fastBuy)

`batch.js:122-145`

```
fastBuy(amountStr):
  - 所有激活钱包以相同金额并行买入
  - 金额为每个钱包的买入量（不是总量）
  - 例如: 0.1 BNB × 3 钱包 = 总计 0.3 BNB
```

### 5.3 快速卖出 (fastSell)

`batch.js:147-180`

```
fastSell(pct):
  BSC:
    - 读取每个钱包的代币余额 state.tokenBalances.get(id)
    - amt = balance * pct / 100
    - 各钱包可能卖出不同数量

  SOL:
    - 传入 "${pct}%" 字符串
    - sol/trading.js#sell() 内部处理百分比:
      balance = getTokenBalance(user, mint, tokenProgram)
      sellAmount = balance * pct / 100
```

### 5.4 并行度分析

- BSC: 所有钱包的 `bscWriteContract()` 调用通过 `chrome.runtime.sendMessage()` 并行发送到后台
  - 后台顺序处理消息（单线程），但 `wc.client.writeContract()` 是异步的
  - 实际网络请求并行发出
  - 瓶颈: RPC 节点的连接数限制

- SOL: 每个钱包独立构建 TX → 序列化 → 后台签名 → 并行发送
  - Jito 广播进一步增加了并行度（每个 TX 同时发到 6 个 Block Engine）
  - 瓶颈: getAccountInfo 等 RPC 调用的并发量

---

## 6. RPC 策略

### 6.1 BSC RPC 节点

`constants.js:2-13`

```
默认 RPC: https://bsc-dataseed.bnbchain.org

内置 BSC RPC 列表:
  - https://bsc-dataseed.bnbchain.org
  - https://bsc-dataseed1.bnbchain.org
  - https://bsc-dataseed2.bnbchain.org
  - https://bsc-dataseed3.bnbchain.org
  - https://bsc-dataseed4.bnbchain.org
  - https://bsc.publicnode.com
  - https://bsc-dataseed1.defibit.io
  - https://bsc-dataseed1.ninicoin.io
```

`wallet-bsc.js:13-20` — V4 客户端使用 `fallback()` 策略：

```javascript
createV4Client(rpcUrl):
  urls = custom ? [custom, ...BSC_RPCS] : BSC_RPCS
  transport: fallback(urls.map(u => http(u)))
  // 自动在多个 RPC 间切换，某个失败自动降级
```

### 6.2 防夹节点集成

`manifest.json:13-27` — Host permissions:

```
host_permissions 包含:
  - https://*.blockrazor.xyz/*   — BlockRazor 防夹 RPC
  - https://*.48.club/*          — 48 Club MEV 保护 RPC
  - https://*.drpc.org/*         — dRPC
  - https://*.helius-rpc.com/*   — Helius (SOL)
  - https://*.quicknode.pro/*    — QuickNode
  - https://*.rpcpool.com/*      — RPCPool
  - https://*.triton.one/*       — Triton
  - https://*.getblock.io/*      — GetBlock
  - https://*.jito.wtf/*         — Jito Block Engine
```

用户可在设置页面配置自定义 BSC RPC URL，推荐使用 BlockRazor 或 48.club 的防夹节点以避免三明治攻击。

### 6.3 SOL RPC 策略

`sol/constants.js:36-40`

```
默认 SOL RPC: https://solana-rpc.publicnode.com

Fallback RPCs:
  - https://solana-rpc.publicnode.com
  - https://api.mainnet-beta.solana.com
```

`token-sol.js:16-51` — 自动降级机制:

```
detectWithFallback(addr):
  ① 尝试用户配置的 RPC（超时 15s）
  ② 如果是 RPC 错误（403/410/429/网络错误）:
     → 遍历 FALLBACK_SOL_RPCS
     → 每个 fallback 超时 10s
  ③ 检测成功后恢复原始 RPC
```

RPC 错误识别: `isRpcError(msg)` 检测 403/410/429/网络/超时等错误码。

### 6.4 连接管理

```
BSC:
  state.publicClient — 主读取客户端 (viem PublicClient)
  state.v4PublicClient — 备用客户端 (fallback transport)
  background.js 中的 WalletClient — 签名专用

SOL:
  sol/connection.js — 单例 Connection
  setConnection(rpc, wss) — 支持自定义 WebSocket URL
  WebSocket 用于交易确认的实时订阅
```

---

## 7. 代币检测与自动路由

### 7.1 自动合约识别

`background.js:362-415` — URL 监听:

```
chrome.tabs.onUpdated.addListener((tabId, changeInfo, tab) => {
  if (changeInfo.status === 'complete') {
    const detected = extractContractAddress(tab.url);
    if (detected) {
      chrome.runtime.sendMessage({
        type: 'CONTRACT_DETECTED',
        address: detected.address,
        chain: detected.chain,  // 'bsc' 或 'sol'
      });
    }
  }
});
```

**Solana 地址识别模式**:
```
pump.fun/coin/{addr}
dexscreener.com/solana/{addr}
birdeye.so/token/{addr}
gmgn.ai/sol/token/{addr}
solscan.io/token/{addr}
photon.club/*/token/{addr}
```

**BSC 地址识别模式**:
```
debot.ai/(address|token)/*/0x{addr}
gmgn.ai/token/*/0x{addr}
dexscreener.com/*/0x{addr}
dextools.io/*/0x{addr}
poocoin.app/tokens/0x{addr}
bscscan.com/(token|address)/0x{addr}
pancakeswap.finance?outputCurrency=0x{addr}
photons.club/token/*/0x{addr}
birdeye.so/token/0x{addr}
通用: 0x{40 hex chars}
```

链判断: Solana 地址使用 base58（32-44 字符），BSC 地址以 `0x` 开头（40 hex 字符）。

### 7.2 BSC 代币检测

`token-bsc.js:9-104` — `detectBscToken()`

```
detectBscToken(addr):
  ① 并行批量读取:
     - FreedomRouter.getTokenInfo(addr) → routeSource, approveTarget, isInternal, tmFunds...
     - ERC20.symbol(addr)
     - ERC20.decimals(addr)
     - ERC20.balanceOf(addr, walletAddress) × N 个钱包

  ② 根据 routeSource 判断:
     route 1-3 → Four.meme (内盘BNB/内盘ERC20/外盘)
     route 4-6 → Flap (内盘/仅卖/DEX)
     route 7   → PancakeSwap

  ③ UI 标签:
     - "🔥 Four 内盘" (tag-internal)
     - "🥞 Four 外盘" (tag-external)
     - "🦋 Flap 内盘" (tag-internal)
     - "🦋 Flap 内盘(仅卖)" (tag-internal)
     - "🦋 Flap DEX" (tag-external)
     - "🥞 外盘" (tag-external)
     - "⚠️ 无LP"
```

### 7.3 SOL 代币检测

`token-sol.js:53-170` — `detectSolToken()`

```
detectSolToken(addr):
  ① detectWithFallback(addr) — 带 RPC 降级的检测
     → sol/trading.js#detectToken() → type: bonding-curve | pumpswap | completed-no-pool

  ② 并行读取:
     - getTokenBalance() × N 个钱包
     - getTokenMetadata() — Metaplex 元数据 或 Token-2022 扩展元数据

  ③ 储备信息:
     - bonding-curve: realSolReserves + realTokenReserves
     - pumpswap: getPoolReserves() → baseReserve + quoteReserve

  ④ UI 标签:
     - "🔥 内盘" (Pump Bonding Curve)
     - "💧 外盘" (PumpSwap AMM)
     - "⚠️ 无Pool"
```

### 7.4 Token-2022 兼容

`sol/accounts.js:202-209`:

```javascript
getTokenProgram(mint):
  info = await conn.getAccountInfo(mint)
  if info.owner == TOKEN_2022_PROGRAM → return TOKEN_2022_PROGRAM
  else → return SPL_TOKEN_PROGRAM
```

整个交易链路中 `tokenProgram` 参数贯穿传递，确保 ATA 派生、转账指令使用正确的 Token 程序。

---

## 8. 合约源码分析

### 8.1 FreedomRouter 主合约

**文件**: `FreedomRouter/contracts/FreedomRouter.sol` (815 行)

#### 8.1.1 UUPS 升级模式

```solidity
contract FreedomRouterImpl is Ownable, ReentrancyGuard, Initializable, UUPSUpgradeable {
    constructor() Ownable(msg.sender) {
        _disableInitializers();  // 防止 implementation 被直接初始化
    }

    function initialize(address _owner, address _tmV2, address _helper3, address _flapPortal)
        external initializer { ... }

    function _authorizeUpgrade(address) internal override onlyOwner {}
    // 仅 owner 可升级，无需新 implementation 地址验证
}

contract FreedomRouter is ERC1967Proxy {
    constructor(address impl, bytes memory data) ERC1967Proxy(impl, data) {}
    // 标准 ERC1967 代理
}
```

升级流程 (`scripts/upgrade.js`):
1. 验证 deployer 是 proxy owner
2. 读取旧 implementation 地址（ERC1967 slot `0x360894a...`）
3. 部署新 FreedomRouterImpl
4. 调用 `router.upgradeToAndCall(newImplAddr, "0x")`
5. 验证新 implementation 地址

#### 8.1.2 TokenInfo 聚合查询

`getTokenInfo()` 是一个只读聚合函数，一次 RPC 调用返回全部代币信息:

```solidity
struct TokenInfo {
    // 基础 ERC20
    string symbol; uint8 decimals; uint256 totalSupply; uint256 userBalance;
    // 路由
    RouteSource routeSource; address approveTarget;
    // Four.meme 状态
    uint256 mode; bool isInternal; bool tradingHalt; uint256 tmVersion;
    address tmAddress; address tmQuote; uint256 tmStatus;
    uint256 tmFunds; uint256 tmMaxFunds; uint256 tmOffers; uint256 tmMaxOffers;
    uint256 tmLastPrice; uint256 tmLaunchTime; uint256 tmTradingFeeRate;
    bool tmLiquidityAdded;
    // Flap 状态
    uint8 flapStatus; uint256 flapReserve; uint256 flapCirculatingSupply;
    uint256 flapPrice; uint8 flapTokenVersion; address flapQuoteToken;
    bool flapNativeSwapEnabled; uint256 flapTaxRate; address flapPool;
    uint256 flapProgress;
    // PancakeSwap
    address pair; address quoteToken; uint256 pairReserve0; uint256 pairReserve1;
    bool hasLiquidity;
    // Tax Token
    bool isTaxToken; uint256 taxFeeRate;
}
```

#### 8.1.3 Tax Token 兼容

`_sellPancakeCompat()` 使用 **balanceOf 差值** 替代 `amountIn` 来兼容 tax token:

```solidity
function _sellPancakeCompat(address token, uint256 amountIn, ...) internal returns (uint256) {
    uint256 balBefore = IERC20(token).balanceOf(address(this));
    IERC20(token).safeTransferFrom(msg.sender, address(this), amountIn);
    uint256 actualIn = IERC20(token).balanceOf(address(this)) - balBefore;
    // actualIn 可能小于 amountIn（税收代币扣除了一部分）
    IERC20(token).forceApprove(PANCAKE_ROUTER, actualIn);
    // ... 使用 actualIn 进行 swap
}
```

Tax Token 检测:
```solidity
// tmVersion == 2 时检查 template 字段
uint256 creatorType = (tmInfo.template >> 10) & 0x3F;
info.isTaxToken = (creatorType == 5);  // creatorType 5 = Tax Token
if (info.isTaxToken) {
    info.taxFeeRate = ITaxToken(token).feeRate();
}
```

#### 8.1.4 findBestQuote — 最优配对搜索

```solidity
function findBestQuote(address token) internal view returns (address bestQuote, address bestPair) {
    address[6] memory quotes = [WBNB, USDT, USD1, USDC, BUSD, FDUSD];
    uint256 bestLiquidity;
    for (uint256 i = 0; i < quotes.length; i++) {
        address p = IPancakeFactory.getPair(token, quotes[i]);
        if (p != address(0)) {
            (uint112 r0, uint112 r1,) = IPancakePair(p).getReserves();
            uint256 liq = uint256(r0) * uint256(r1);  // 以乘积作为流动性度量
            if (liq > bestLiquidity) {
                bestLiquidity = liq;
                bestQuote = quotes[i];
                bestPair = p;
            }
        }
    }
}
```

### 8.2 VanityDeployer

`FreedomRouter/contracts/VanityDeployer.sol` (22 行)

使用 CREATE2 操作码部署合约到可预测地址（vanity address）:

```solidity
function deploy2(bytes32 salt, bytes calldata initCode) external returns (address addr) {
    assembly {
        addr := create2(0, ptr, initCode.length, salt)
    }
}
```

配合 `scripts/prepare-vanity.js` 和 `scripts/deploy-vanity.js` 可以暴力搜索特定前缀的合约地址。

### 8.3 FreedomHook (PancakeSwap Infinity)

`hooklimit/src/FreedomHook.sol` (489 行)

PancakeSwap V4 (Infinity) CL Hook，实现三个功能:

1. **限价单 (Limit Orders)**: 通过单边流动性实现
   - `place()` — 在指定 tick 放置限价单
   - `kill()` — 取消限价单
   - `withdraw()` — 提取已成交订单
   - 使用 Epoch 机制追踪成交状态

2. **方向限制**: `setDirection(key, Direction.BuyOnly/SellOnly/Both)`
   - `_beforeSwap()` 中检查方向

3. **Pool 创建**: `createPool(tokenA, tokenB, sqrtPriceX96)`

Hook bitmap: `afterInitialize + beforeSwap + afterSwap`

### 8.4 FourMemeRouter (早期版本)

`trader-extension/contracts/FourMemeRouter.sol` (322 行)

这是 FreedomRouter 的前身/简化版本，只支持 Four.meme:

```
PoolType: NOT_FOUND / FOUR_MEME_BNB / FOUR_MEME_USD1 / FOUR_MEME_EXTERNAL / PANCAKE_ONLY
```

- 内盘 BNB: 直接调用 Four.meme Proxy (`0x593445503aca66cc316a313b6f14a1639da1e484`)，手动编码 calldata
- 内盘 USD1: Helper3.buyWithEth
- 外盘: PancakeSwap

---

## 9. 关键代码路径

### 9.1 BSC 买入完整调用链

```
用户操作: 点击 "🚀 买入" 按钮
    │
    ▼
ui.js:376-379 — click event → 'tradeBtn'
    │ import('./batch.js') → executeBatchTrade()
    ▼
batch.js:67-120 — executeBatchTrade()
    │ ① tokenAddr = $('tokenAddress').value.trim()
    │ ② normalizedAmount = normalizeAmount(amountStr, ...)
    │ ③ activeWallets = getActiveWallets()
    │ ④ Promise.allSettled(activeWallets.map(id => doBuy(id, tokenAddr, normalizedAmount)))
    ▼
batch.js:40-49 — doBuy(id, tokenAddr, amountStr)
    │ return buy(id, tokenAddr, normalizedAmount, getPriorityFee())
    ▼
trading.js:150-205 — buy(walletId, tokenAddr, amountStr, gasPrice)
    │ ① amt = parseUnits(normalizedAmount, quoteDecimals)
    │ ② tipRate = getTipRate()  // 0-500 bps
    │ ③ slipBps = (100 - slippage) * 100
    │ ④ amountOutMin = _getQuoteBuy(tokenAddr, amt, slipBps)
    │     → publicClient.readContract(FREEDOM_ROUTER, 'quote', [quoteAddr, amt, tokenAddr])
    │ ⑤ if !native: ensureApproved(walletId, ownerAddress, quoteAddr, FREEDOM_ROUTER, ...)
    │ ⑥ deadline = now + 10s
    ▼
trading.js:180-186 — bscWriteContract(walletId, {...})
    │ address: FREEDOM_ROUTER
    │ functionName: 'trade'
    │ args: [quoteAddr, tokenAddr, native ? 0 : amt, amountOutMin, tipRate, deadline]
    │ value: native ? amt : undefined
    │ gas: 800000n
    │ gasPrice: parseUnits(gasPrice, 9)
    ▼
crypto.js:79-88 — bscWriteContract(walletId, params)
    │ ① 序列化 BigInt 为 {__bigint: string}
    │ ② chrome.runtime.sendMessage({action: 'bscWriteContract', ...})
    ▼
background.js:261-283 — handlers.bscWriteContract()
    │ ① checkExpiry() — 验证会话有效
    │ ② wc = bscClients.get(walletId)
    │     如果不存在 → 重新 initWallets()
    │ ③ 反序列化 BigInt
    │ ④ wc.client.writeContract(params)
    │     → viem 签名 + 发送到 BSC RPC
    │ ⑤ return { txHash }
    ▼
trading.js:191-194 — 等待确认
    │ receipt = publicClient.waitForTransactionReceipt({ hash: txHash, timeout: 120000 })
    │ if receipt.status !== 'success' → throw
    ▼
trading.js:197-203 — 自动预授权
    │ 并行 approve 代币给:
    │   - getSellApproveTarget() (内盘: tokenManagerV2, 外盘: FREEDOM_ROUTER)
    │   - FREEDOM_ROUTER
    ▼
batch.js:89-115 — Promise.allSettled 收集结果
    │ 统计 success/failed
    │ showStatus() / showToast()
    │ loadBalances() + detectToken()
    ▼
完成
```

### 9.2 SOL 买入完整调用链

```
用户操作: 点击 "🚀 买入" 按钮 (SOL 链)
    │
    ▼
batch.js:40-49 — doBuy(id, tokenAddr, amountStr)
    │ isSol() === true
    │ return solBuy(id, tokenAddr, parseFloat(normalizedAmount), getSlippage(), {
    │   priorityFee: getPriorityFee(),
    │   jitoTip: getJitoTip(),
    │ })
    ▼
sol-trading.js:27-50 — solBuy(walletId, mintAddr, solAmount, slippage, opts)
    │ publicKey = state.solAddresses.get(walletId)
    │ return buy(walletId, publicKey, mintAddr, solAmount, slippage, {
    │   priorityFeeLamports, computeUnits: 200000,
    │   tipBps, jitoTipLamports,
    │   detectResult: getCachedDetectResult()
    │ })
    ▼
sol/trading.js:110-178 — buy(walletId, publicKey, mintAddress, solAmount, slippagePct, opts)
    │ ① mint = new PublicKey(mintAddress)
    │ ② lamports = floor(solAmount * 1e9)
    │ ③ blockhashPromise = getBlockhashFast()  // 可能 0ms (缓存命中)
    │ ④ 并行加载: getTokenProgram(mint), getBondingCurve(mint)
    │
    │ if !bc.complete (内盘 Bonding Curve):
    │   ⑤ feeConfig = getBcFeeConfig()
    │   ⑥ { minTokensOut, tokensOut } = calcBcBuyQuoteSync(lamports, bc, feeConfig, slippage)
    │   ⑦ instructions = [
    │       ComputeBudgetProgram.setComputeUnitLimit(200000),
    │       ComputeBudgetProgram.setComputeUnitPrice(microLamports),
    │       buildCreateBaseATAIx(user, mint, tokenProgram),  // 创建代币 ATA
    │       buildBondingCurveBuyIx(mint, user, bc, tokenProgram, lamports, minTokensOut),
    │       buildTipIx(user, solAmount, tipBps),   // 开发者小费
    │       buildMarkerIx(user),                    // 交易标记 (1 lamport)
    │       buildJitoTipIx(user, jitoTip),          // Jito 小费
    │     ]
    │
    │ if bc.complete (外盘 PumpSwap AMM):
    │   ⑤ pool = findPoolForMint(mint)
    │   ⑥ { baseReserve, quoteReserve } = getPoolReserves(pool)
    │   ⑦ { totalFeeBps } = getAmmDynamicFeesSync(baseReserve, quoteReserve)
    │   ⑧ { baseOut, maxQuoteIn } = calcAmmBuyQuote(lamports, baseReserve, quoteReserve, totalFeeBps, slippage)
    │   ⑨ instructions = [
    │       ComputeBudgetProgram.setComputeUnitLimit/Price,
    │       buildWrapSolInstructions(user, maxQuoteIn, wsolAta),  // 创建 WSOL ATA + 转 SOL + syncNative
    │       buildCreateBaseATAIx(user, pool.baseMint, tokenProgram),
    │       buildPumpSwapBuyIx(user, pool, tokenProgram, globalConfig, baseOut, maxQuoteIn),
    │       buildCloseWsolIx(user, wsolAta),  // 关闭 WSOL ATA，回收 SOL
    │       buildTipIx, buildMarkerIx, buildJitoTipIx,
    │     ]
    ▼
sol/trading.js:354-401 — buildAndSend(conn, walletId, user, instructions, t0, opts)
    │ ① tx = new Transaction().add(...instructions)
    │ ② tx.recentBlockhash = blockhash
    │ ③ tx.feePayer = user
    │ ④ txBase64 = serialize(unsigned) → base64
    │ ⑤ solSignAndSend(walletId, { txBase64, rpcUrl, jitoTipLamports })
    ▼
crypto.js:91-93 — solSignAndSend(walletId, params)
    │ chrome.runtime.sendMessage({action: 'solSignAndSend', ...})
    ▼
background.js:286-314 — handlers.solSignAndSend()
    │ ① checkExpiry()
    │ ② kp = solKeypairs.get(walletId)
    │ ③ tx = Transaction.from(txBuf)
    │ ④ tx.sign(kp)
    │ ⑤ raw = tx.serialize()
    │ ⑥ 并行双发:
    │     sends[0] = conn.sendRawTransaction(raw, { skipPreflight: true })
    │     sends[1] = sendToJito(rawBase64)
    │       → POST 到 6 个 Jito Block Engine
    │ ⑦ [sig] = await Promise.all(sends)
    │ ⑧ return { signature: sig }
    ▼
sol/trading.js:388-400 — 确认
    │ slot = confirmWithFallback(conn, sig)
    │   → WebSocket + 轮询竞争确认
    │ getBlockTime(conn, sig) — 异步获取链上时间（日志用）
    ▼
返回 { signature, buildMs, sendMs, confirmMs, elapsed, slot }
    ▼
batch.js — 收集所有钱包结果，更新 UI
```

### 9.3 时间性能指标

交易过程记录三个关键时间点:

```
t0 ─── buildMs ──→ tBuild ─── sendMs ──→ tSent ─── confirmMs ──→ tDone
       构建交易          签名+广播           等待确认

BSC 典型值:
  buildMs: ~50-200ms (RPC 读取)
  sendMs:  ~100-500ms (签名 + 发送)
  confirmMs: ~3-15s (等待出块)

SOL 典型值:
  buildMs: ~100-500ms (RPC 读取 + 指令构建)
  sendMs:  ~200-1000ms (签名 + RPC + Jito 广播)
  confirmMs: ~400ms-10s (WebSocket/轮询确认)
```

---

## 附录

### A. 关键合约地址

| 名称 | 地址 | 链 |
|------|------|-----|
| FreedomRouter Proxy | `0xCd4D70bb991289b5A8522adB93Cd3C4b93B4Dceb` | BSC |
| TokenManager V2 | `0x5c952063c7fc8610FFDB798152D69F0B9550762b` | BSC |
| Helper3 | `0xF251F83e40a78868FcfA3FA4599Dad6494E46034` | BSC |
| Flap Portal | `0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0` | BSC |
| PancakeSwap Router | `0x10ED43C718714eb63d5aA57B78B54704E256024E` | BSC |
| PancakeSwap Factory | `0xcA143Ce32Fe78f1f7019d7d551a6402fC5350c73` | BSC |
| WBNB | `0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c` | BSC |
| DEV (小费接收) | `0x2De78dd769679119b4B3a158235678df92E98319` | BSC |
| Pump.fun Program | `6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P` | SOL |
| PumpSwap AMM | `pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA` | SOL |
| Pump Fee Program | `pfeeUxB6jkeY1Hxd7CsFCAjcbHA9rWtchMGdZ6VojVZ` | SOL |
| SOL Tip Recipient | `D6kPpTmJQA3eCLAZVJj8c3JKsrmHzm9q9sTQu6BvzPxP` | SOL |

### B. 文件清单

```
FreedomRouter/
  contracts/FreedomRouter.sol    — 主合约（815行）
  contracts/VanityDeployer.sol   — CREATE2 部署器（22行）
  scripts/deploy.js             — 部署脚本
  scripts/upgrade.js            — UUPS 升级脚本
  scripts/test.js               — 测试脚本
  scripts/deploy-vanity.js      — Vanity 地址部署
  scripts/prepare-vanity.js     — Vanity 地址搜索
  scripts/tip-stats.js          — 小费统计

hooklimit/
  src/FreedomHook.sol           — PancakeSwap V4 Hook（489行）
  test/FreedomHook.t.sol        — Hook 测试
  script/DeployFreedomHook.s.sol — Hook 部署脚本

trader-extension/
  manifest.json                 — Chrome 扩展清单 (MV3)
  src/background.js             — Service Worker（416行）
  src/trader.js                 — 交易页入口（243行）
  src/trading.js                — BSC 交易逻辑（261行）
  src/batch.js                  — 批量交易（181行）
  src/crypto.js                 — 加密代理（93行）
  src/constants.js              — 常量定义（103行）
  src/state.js                  — 全局状态（32行）
  src/ui.js                     — UI 逻辑（455行）
  src/token.js                  — 代币检测路由（11行）
  src/token-bsc.js              — BSC 代币检测（183行）
  src/token-sol.js              — SOL 代币检测（207行）
  src/wallet.js                 — 钱包管理路由（72行）
  src/wallet-bsc.js             — BSC 钱包（105行）
  src/wallet-sol.js             — SOL 钱包（82行）
  src/sol-trading.js            — SOL 交易桥接（76行）
  src/lock.js                   — 锁屏逻辑（60行）
  src/settings.js               — 设置页面（633行）
  src/utils.js                  — 工具函数（83行）
  src/theme.js                  — 主题切换（21行）
  src/sol/trading.js            — SOL 交易核心（401行）
  src/sol/bonding-curve.js      — Bonding Curve 计算（163行）
  src/sol/pump-swap.js          — PumpSwap AMM（252行）
  src/sol/accounts.js           — 链上账户读取（343行）
  src/sol/connection.js         — RPC 连接管理（103行）
  src/sol/constants.js          — SOL 常量（71行）
  src/sol/pda.js                — PDA 派生（109行）
  contracts/FourMemeRouter.sol   — 早期路由合约（322行）
  scripts/build.js              — esbuild 构建（113行）
  scripts/test-regressions.mjs  — 回归测试
```

### C. 指令 Discriminator

| 操作 | Discriminator (hex) | 程序 |
|------|---------------------|------|
| BC Buy (ExactSolIn) | `38fc74089edfcd5f` | Pump.fun |
| BC Sell | `33e685a4017f83ad` | Pump.fun |
| AMM Buy | `66063d1201daebea` | PumpSwap |
| AMM Sell | `33e685a4017f83ad` | PumpSwap |

注: BC Sell 和 AMM Sell 共享同一 discriminator。
