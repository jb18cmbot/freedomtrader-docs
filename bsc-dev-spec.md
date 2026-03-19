# BSC 交易 Chrome 插件 — 开发规格文档

> 基于 Freedom Trader 源码深度分析，为全新 BSC 交易插件设计的完整开发文档
> 版本: 1.0 | 日期: 2026-03-20

---

## 目录

- [Part 1: BSC 交易架构深度分析](#part-1-bsc-交易架构深度分析)
- [Part 2: 架构决策深度讨论](#part-2-架构决策深度讨论)
- [Part 3: 完整开发规格文档](#part-3-完整开发规格文档)

---

# Part 1: BSC 交易架构深度分析

## 1.1 Freedom Trader 核心架构

Freedom Trader 是一个 Chrome Manifest V3 侧边栏扩展，BSC 部分采用 **三层分离架构**：

```
┌─────────────────────────────────────────────┐
│            Side Panel UI Layer              │
│  trader.html / settings.html               │
│  ┌─────────────────────────────────────┐   │
│  │  前端 ES Modules                     │   │
│  │  ui.js ← state.js ← utils.js       │   │
│  │  batch.js → trading.js              │   │
│  │  token-bsc.js / wallet-bsc.js       │   │
│  │  crypto.js (消息代理，非加密模块)     │   │
│  └──────────────┬──────────────────────┘   │
│                  │ chrome.runtime.sendMessage│
│  ┌──────────────┴──────────────────────┐   │
│  │  Background Service Worker          │   │
│  │  - 私钥解密 & 内存持有              │   │
│  │  - viem WalletClient 签名           │   │
│  │  - PBKDF2+AES-256-GCM 加密体系     │   │
│  └──────────────┬──────────────────────┘   │
└─────────────────┼──────────────────────────┘
                  │
         ┌────────┴────────┐
         ▼                 ▼
    BSC RPC Node    FreedomRouter 合约
                    (UUPS Proxy v6)
```

### 安全边界

这是整个架构中最关键的设计——**前端永远无法接触到私钥明文**：

1. **crypto.js 是消息代理，不是加密库**：它只做 `chrome.runtime.sendMessage()` 转发
2. **BigInt 序列化协议**：前端 `{__bigint: "123"}` ↔ 后台 `BigInt("123")`
3. **前端只能拿到**：钱包地址（string）和交易哈希（string）
4. **Session Storage 隔离**：密码缓存 `ft_pw` 设为 `TRUSTED_CONTEXTS`，只有 Service Worker 可读

## 1.2 交易流程详解

### 买入流程

```
用户点击买入
  ↓
batch.js#executeBatchTrade()
  ↓ Promise.allSettled (多钱包并行)
trading.js#buy(walletId, tokenAddr, amountStr, gasPrice)
  ↓
① normalizeAmount() → parseUnits() 转 BigInt
② getTipRate() → 0-500 bps (默认 0)
③ _getQuoteBuy() → FreedomRouter.quote(quoteAddr, amt, tokenAddr)
④ amountOutMin = estimated × (100 - slippage)%
⑤ ensureApproved() → 非 BNB 时检查/执行 approve
⑥ bscWriteContract() → background.js 签名发送
   调用: FreedomRouter.trade(quoteAddr, tokenAddr, 0n, amountOutMin, tipRate, deadline)
   value: amt (BNB时)
⑦ waitForTransactionReceipt() 等待确认
⑧ 买入后自动 approve 卖出目标（预授权优化）
```

### 卖出流程

```
trading.js#sell(walletId, tokenAddr, amountStr, gasPrice)
  ↓
① 数量处理（Four 内盘 ERC20 精度对齐到 Gwei）
② _getQuoteSell() → FreedomRouter.quote(tokenAddr, amt, sellQuoteAddr)
③ 检查 balanceOf >= amt
④ ensureApproved() → approve 给 getSellApproveTarget()
   - 内盘: TokenManagerV2
   - 外盘/Flap: FreedomRouter
⑤ bscWriteContract() 签名发送
⑥ 等待确认，返回 timing 数据
```

### Approve 缓存机制

这是一个值得借鉴的优化设计：

```
ensureApproved(walletId, owner, tokenAddr, spender, gasPrice, minAllowance):
  ① 内存缓存检查: state.approvedTokens (Set<"owner:token:spender">)
  ② 并发防重入: approvalInFlight (Map) 防止多钱包同时 approve 同一代币
  ③ 链上读取 allowance → 够用则跳过
  ④ 不够则 approve(MAX_UINT256)，确认后持久化到 chrome.storage.local
```

**买入后自动预授权卖出目标**（trading.js:197-202）：买入成功后立即并行 approve 给 `getSellApproveTarget()` 和 `FREEDOM_ROUTER`，这样卖出时可以省掉一笔 approve 交易。

## 1.3 FreedomRouter 合约逻辑

### 路由检测 (`_detectRoute`)

合约的核心价值在于**统一路由检测**——一次 `getTokenInfo()` 调用返回代币在哪个协议交易：

```
检测优先级: Four.meme → Flap → PancakeSwap

Four.meme 检测:
  IHelper3.getTokenInfo(token) →
    ver > 0 && tm != 0:
      liquidityAdded = true  → FOUR_EXTERNAL (走 PCS)
      liquidityAdded = false → IFourToken._mode():
        mode != 0:
          quote == 0    → FOUR_INTERNAL_BNB
          quote != 0    → FOUR_INTERNAL_ERC20

Flap 检测:
  IFlapPortal.getTokenV7(token) →
    status == 1 (Tradable):
      nativeSwapEnabled → FLAP_BONDING
      否则             → FLAP_BONDING_SELL
    status == 4 (DEX):  → FLAP_DEX

兜底:
  findBestQuote(token) → 遍历 [WBNB, USDT, USD1, USDC, BUSD, FDUSD]
    找到最大流动性配对 → PANCAKE_ONLY
    都没有           → NONE
```

### 交易执行路径

```
buy(): 只接受 BNB (msg.value > 0)
  FOUR_INTERNAL → _buyFourInternal → ITMV2.buyTokenAMAP / IHelper3.buyWithEth
  FOUR_EXTERNAL / PANCAKE / FLAP_DEX / FLAP_BONDING_SELL → _buyPancake → PancakeRouter
  FLAP_BONDING → _buyFlap → IFlapPortal.swapExactInput

sell(): 接受 ERC20 代币
  FOUR_INTERNAL → _sellFourInternal → ITMV2.sellToken / IHelper3.sellForEth
  FOUR_EXTERNAL / PANCAKE / FLAP_DEX → _sellPancakeCompat → PancakeRouter (tax-token 兼容)
  FLAP_BONDING / FLAP_BONDING_SELL → _sellFlap → IFlapPortal.swapExactInput
```

### Tax Token 兼容设计

`_sellPancakeCompat` 使用 `balanceOf` 差值而非 `amountIn` 来获取实际到账量：

```solidity
uint256 balBefore = IERC20(token).balanceOf(address(this));
IERC20(token).safeTransferFrom(msg.sender, address(this), amountIn);
uint256 actualIn = IERC20(token).balanceOf(address(this)) - balBefore;
// actualIn 可能 < amountIn（税币扣税了）
```

买入也类似：先尝试 `swapExactETHForTokensSupportingFeeOnTransferTokens`，失败再降级到普通版本。

### 统一报价 (quote)

合约提供了 `quoteBuy` / `quoteSell` 函数（限 `eth_call` only），前端通过 `readContract` 调用获取预估数量。前端实际使用的是统一的 `quote(tokenIn, amountIn, tokenOut)` 函数（ROUTER_ABI 中定义），一个函数覆盖买卖两个方向。

### 小费机制

```
MAX_TIP = 500 (5%)
tip = amount × tipRate / 10000
DEV 地址: 0x2De78dd769679119b4B3a158235678df92E98319
默认 tipRate = 0（纯自愿）
```

## 1.4 状态管理

`state.js` 是一个全局单例对象，包含：

| 字段 | 类型 | 用途 |
|------|------|------|
| `publicClient` | viem PublicClient | 读取链上数据 |
| `walletClients` | Map<id, {address}> | 前端只持有地址 |
| `walletBalances` | Map<id, bigint> | BNB 余额 |
| `tokenBalances` | Map<id, bigint> | 当前代币余额 |
| `approvedTokens` | Set<string> | approve 缓存 |
| `tokenInfo` | {decimals, symbol, balance, address} | 当前代币信息 |
| `lpInfo` | {hasLP, routeSource, approveTarget, isInternal, ...} | 路由信息 |
| `quoteToken` | {symbol, address, decimals} | 报价代币(BNB/USDT等) |
| `config` | {} | 用户配置 |

## 1.5 关键设计模式总结

| 模式 | 实现 | 评价 |
|------|------|------|
| 私钥隔离 | Service Worker 持有，前端只能通过消息代理 | 优秀，必须保留 |
| 多钱包并行 | Promise.allSettled，互不影响 | 实用 |
| Approve 缓存 | 内存 + chrome.storage.local 双层缓存 | 省 gas，值得借鉴 |
| 预授权卖出 | 买入后自动 approve 卖出目标 | 省一笔交易 |
| 合约聚合查询 | getTokenInfo 一次返回全部路由信息 | 减少 RPC 调用 |
| URL 自动识别 | background.js 监听 tab URL，正则匹配合约地址 | 好的 UX |
| 金额规范化 | sanitizeAmountInput + normalizeAmount | 防精度问题 |
| gas/slippage 可配置 | 快捷按钮 + 自定义值 | 灵活 |

---

# Part 2: 架构决策深度讨论

## 2.1 是否需要写聚合路由合约？

### FreedomRouter 的核心价值

FreedomRouter v6 做了三件事：

1. **统一路由检测** — `_detectRoute()` + `getTokenInfo()` 一次调用返回代币在 Four.meme / Flap / PancakeSwap 的状态
2. **统一交易入口** — `buy()` / `sell()` 根据路由自动分发到不同协议
3. **Tax Token 兼容** — `balanceOf` 差值计算 + `SupportingFeeOnTransferTokens` 降级

### 有合约 vs 无合约对比

| 维度 | 有合约（如 FreedomRouter） | 无合约（纯前端路由） |
|------|---------------------------|---------------------|
| **gas** | 额外 ~5-15K gas（代理转发开销） | 更低，直接调用目标协议 |
| **灵活性** | 需要升级合约才能支持新协议 | 前端修改即时生效 |
| **安全性** | 合约需审计，owner 权限是风险点 | 无合约风险，但前端逻辑更复杂 |
| **路由检测** | 链上一次 call 返回全部信息 | 需要前端多次 RPC 调用 |
| **维护成本** | 需要 Solidity 开发+部署+审计 | 纯前端维护 |
| **Approve 目标** | 统一 approve 给路由合约 | 每个协议需要单独 approve |
| **Tax Token** | 合约内中转可精确测量实际到账量 | 前端难以精确处理 |
| **速度** | 路由检测更快（一次 call） | 多次 RPC 调用有延迟 |

### 深度分析

**合约的真正痛点是升级和审计成本**。FreedomRouter 已经从 v1 升级到 v6，每次 Four.meme 或 Flap 改接口都需要升级合约。UUPS 升级模式虽然允许热升级，但：

- 升级后所有用户立即受影响（无灰度）
- 每次改动都应该审计
- owner 私钥是单点故障

**无合约方案的难点是路由检测效率**。FreedomRouter.getTokenInfo() 一次 RPC 调用返回代币的完整状态（Four.meme 信息 + Flap 信息 + PancakeSwap 配对 + 路由源 + approve 目标）。如果前端自己做，需要至少 3-5 次 RPC 调用：

```
不用合约的路由检测：
① IHelper3.getTokenInfo(token)     — 检查 Four.meme
② IFlapPortal.getTokenV7(token)    — 检查 Flap
③ IPancakeFactory.getPair(token, WBNB)  — 检查 PCS/WBNB
④ IPancakeFactory.getPair(token, USDT)  — 检查 PCS/USDT
⑤ ... 更多 quote token 配对检查
```

但这些调用可以用 `Promise.all` 并行，总延迟约等于单次 RPC 调用的延迟（~50-200ms），与合约方案相当。

**Tax Token 处理**是合约方案的独特优势。合约可以：
1. `transferFrom` 代币到自己
2. 测量 `balanceOf` 差值获取实际到账量
3. 用实际到账量调用 DEX

前端无合约方案处理 tax token 需要：
1. 先 `readContract` 模拟获取税率
2. 或者用 `estimateGas` / `eth_call` 模拟交易
3. 直接调用 PancakeRouter 的 `SupportingFeeOnTransferTokens` 版本

### 最终建议：**不写合约，采用纯前端路由**

理由：

1. **维护成本远低于收益** — 对于一个 Chrome 插件项目，合约升级+审计的成本过高
2. **前端路由足够用** — BSC 的主要 DEX 就是 PancakeSwap，Four.meme 和 Flap 的接口已经稳定
3. **PancakeSwap 原生支持 Tax Token** — `SupportingFeeOnTransferTokens` 函数就是为此设计的
4. **减少 gas** — 不经过中间合约，直接调用目标协议
5. **灵活性** — 新增协议只需要前端更新，不需要链上部署
6. **去中心化** — 没有 owner 权限风险，没有合约升级风险

**实现方案**：

```javascript
// 前端路由检测（并行调用）
async function detectRoute(tokenAddr) {
  const [fourInfo, flapInfo, pcsQuotes] = await Promise.all([
    detectFourMeme(tokenAddr),      // IHelper3.getTokenInfo
    detectFlap(tokenAddr),           // IFlapPortal.getTokenV7
    detectPancakeSwap(tokenAddr),    // 并行检查多个 quote token
  ]);

  if (fourInfo.found) return fourInfo.isInternal ? 'four-internal' : 'four-external';
  if (flapInfo.found) return flapInfo.status === 1 ? 'flap-bonding' : 'flap-dex';
  if (pcsQuotes.bestPair) return 'pancake';
  return 'none';
}

// 前端直接调用目标协议
async function buyFourInternal(token, amount) {
  return bscWriteContract(walletId, {
    address: TOKEN_MANAGER_V2,  // 直接调用 Four.meme
    abi: TM_ABI,
    functionName: 'buyTokenAMAP',
    args: [token, userAddress, amount, minOut],
    value: amount,
  });
}

async function buyPancake(token, amount, minOut) {
  return bscWriteContract(walletId, {
    address: PANCAKE_ROUTER,  // 直接调用 PancakeSwap
    abi: PANCAKE_ABI,
    functionName: 'swapExactETHForTokensSupportingFeeOnTransferTokens',
    args: [minOut, path, userAddress, deadline],
    value: amount,
  });
}
```

## 2.2 如何提升交易速度？

### 2.2.1 RPC 节点选择策略

#### 公共 vs 付费 vs 自建

| 方案 | 延迟 | 可靠性 | 防夹 | 成本 | 推荐场景 |
|------|------|--------|------|------|----------|
| 公共 RPC (bnbchain.org) | 100-500ms | 低（限流） | 无 | 免费 | 开发测试 |
| 付费 RPC (QuickNode/GetBlock) | 30-100ms | 高 | 无 | $50-200/月 | 普通用户 |
| 防夹 RPC (48.club/BlockRazor) | 50-150ms | 高 | 有 | $100-300/月 | 交易用户（推荐） |
| 自建节点 | 最低 | 最高 | 可自定义 | $500+/月 | 专业用户 |

#### 推荐策略

```
默认: 公共 RPC (https://bsc-dataseed.bnbchain.org)
推荐: 引导用户配置防夹 RPC
  - 48.club: https://rpc-bsc.48.club (免费防夹)
  - BlockRazor: https://bsc-rpc.blockrazor.xyz (付费防夹)

读取 vs 写入分离:
  - publicClient (读取): 可以用公共 RPC，延迟不太敏感
  - walletClient (写入/签名): 必须用防夹节点
```

#### 防夹原理

BSC 的 MEV（三明治攻击）非常猖獗。防夹节点的原理是：
1. 交易不进入公共 mempool
2. 直接发送给验证者/矿工的私有通道
3. 攻击者无法看到你的交易并在前面插入

**48.club** 是 BSC 最大的验证者之一，它的 RPC 直接将交易送入自己的出块流程。
**BlockRazor** 类似，通过私有通道分发交易。

### 2.2.2 Gas 策略

```javascript
// 动态 gas price 策略
async function getOptimalGasPrice() {
  const block = await publicClient.getBlock();
  const baseFee = block.baseFeePerGas; // BSC EIP-1559 后支持

  // BSC 当前最低 1 Gwei，但高峰期可能需要 3-5 Gwei
  // 策略: 取链上近几个区块的中位 gas price 的 110%
  const gasPrice = await publicClient.getGasPrice();
  return gasPrice * 110n / 100n; // +10% 确保优先被打包
}

// 用户可调的 gas preset
const GAS_PRESETS = {
  normal: 3,    // 3 Gwei
  fast: 5,      // 5 Gwei
  turbo: 10,    // 10 Gwei（高峰期）
};
```

### 2.2.3 交易构建优化

#### 并行构建

```javascript
// Freedom Trader 现有方案：每个钱包串行 approve → trade
// 优化方案：并行构建所有钱包的交易

async function batchBuy(walletIds, tokenAddr, amount) {
  // 1. 并行检查所有钱包的 approve 状态
  const approveStates = await Promise.all(
    walletIds.map(id => checkApproval(id, tokenAddr))
  );

  // 2. 需要 approve 的钱包并行发送 approve
  const needApprove = walletIds.filter((id, i) => !approveStates[i]);
  await Promise.all(needApprove.map(id => sendApprove(id, tokenAddr)));

  // 3. 所有钱包并行发送交易
  return Promise.allSettled(
    walletIds.map(id => sendBuy(id, tokenAddr, amount))
  );
}
```

#### Nonce 管理

多钱包并行时，同一钱包不会有 nonce 冲突（每个钱包独立地址）。但如果同一钱包快速连续发交易（如 approve + buy），需要手动管理 nonce：

```javascript
async function sendWithManagedNonce(walletId, txParams) {
  const nonce = await publicClient.getTransactionCount({
    address: walletAddress,
    blockTag: 'pending'  // 包含 pending 交易的 nonce
  });
  return bscWriteContract(walletId, { ...txParams, nonce });
}
```

#### 预签名（不适用当前架构）

由于 BSC 的 nonce 机制，无法真正"预签名"。但可以：
- 提前获取 nonce
- 提前构建交易参数
- 在用户点击时只做签名+发送（最后一步）

### 2.2.4 WebSocket vs 轮询

```
交易确认:
  - 当前方案: waitForTransactionReceipt (轮询)
  - 优化方案: WebSocket 订阅 newHeads + 轮询双重确认

BSC 出块时间: ~3 秒
  - 轮询间隔建议: 1 秒
  - WebSocket: 连接 wss://bsc-ws-node.nariox.org 监听新块

价格监控:
  - 实时价格: WebSocket 订阅 PancakeSwap 配对事件 (Swap/Sync)
  - 定期刷新: 每 5 秒轮询一次 getReserves
```

### 2.2.5 延迟优化数据

```
典型交易耗时分解 (BSC):
┌────────────────────────────┬──────────┐
│ 阶段                        │ 耗时      │
├────────────────────────────┼──────────┤
│ RPC 读取 (quote/balance)   │ 50-200ms │
│ 签名 (viem)                │ 1-5ms    │
│ RPC 发送 (eth_sendRawTx)   │ 50-200ms │
│ 等待确认 (1-2 个区块)      │ 3-6s     │
│ Approve (如需要)           │ +3-6s    │
├────────────────────────────┼──────────┤
│ 总计 (无 approve)          │ 3-7s     │
│ 总计 (有 approve)          │ 6-13s    │
└────────────────────────────┴──────────┘

优化后目标:
- 预授权 → 消除 approve 等待
- 防夹节点 → 发送延迟 < 50ms
- 预取 quote → RPC 读取并行化
- 总计: 3-5s (受限于区块确认时间)
```

### 2.2.6 多 RPC 竞速发送

```javascript
// 将签名后的交易同时发送到多个 RPC
async function raceSubmit(signedTx) {
  const rpcs = [
    'https://rpc-bsc.48.club',          // 防夹
    'https://bsc-dataseed.bnbchain.org', // 公共
    'https://bsc.publicnode.com',        // 公共
  ];

  // 竞速：第一个成功的 RPC 返回 txHash
  const result = await Promise.any(
    rpcs.map(rpc => sendRawTransaction(rpc, signedTx))
  );
  return result.txHash;
}
```

**注意**: 多 RPC 竞速发送需要使用 `sendRawTransaction` 而非 `writeContract`，因为后者包含签名步骤。架构上需要：
1. 在 background.js 中签名得到 raw tx
2. 返回 raw tx 给前端（或后台自己竞速发送）
3. 并行发送到多个 RPC

当前 Freedom Trader 的架构是 `writeContract` 一步完成签名+发送，要支持竞速需要拆分。

## 2.3 Chrome 插件形态设计

### 2.3.1 侧边栏 vs Popup vs 独立标签页

| 形态 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **侧边栏 (Side Panel)** | 常驻显示，不会关闭；与网页并排 | 宽度受限 (~350px)；需要 Chrome 114+ | 配合 GMGN/DexScreener 使用 |
| **Popup** | 最简单；点击图标即弹出 | 点击外部就关闭；交易中不能切换 | 简单查询 |
| **独立标签页** | 无尺寸限制；完整 UI | 需要切换标签；与行情页不在同一视野 | 复杂 Dashboard |
| **混合方案** | 侧边栏做交易，标签页做设置/历史 | 开发量稍大 | 推荐方案 |

### 最终建议：**侧边栏 + 独立标签页混合**

```
侧边栏 (主交易界面):
  - 代币地址输入 + 自动检测
  - 快捷买入/卖出按钮
  - 余额显示
  - 实时价格
  - 交易状态

独立标签页 (设置 & 历史):
  - 钱包管理
  - RPC 配置
  - 交易历史
  - 高级设置
```

### 2.3.2 与 GMGN/DexScreener 集成

```javascript
// URL 监听 + 自动填充代币地址（复用 Freedom Trader 方案）
chrome.tabs.onUpdated.addListener((tabId, changeInfo, tab) => {
  if (changeInfo.status === 'complete') {
    const contract = extractContractFromUrl(tab.url);
    if (contract) {
      // 通知侧边栏自动检测代币
      chrome.runtime.sendMessage({
        type: 'CONTRACT_DETECTED',
        address: contract.address,
        chain: 'bsc',
        source: tab.url,
      });
    }
  }
});

// 支持的平台
const BSC_URL_PATTERNS = [
  /gmgn\.ai\/token\/[^/]*\/(0x[a-fA-F0-9]{40})/i,
  /dexscreener\.com\/[^/]*\/(0x[a-fA-F0-9]{40})/i,
  /debot\.ai\/(address|token)\/[^/]*\/(?:\d+_)?(0x[a-fA-F0-9]{40})/i,
  /bscscan\.com\/(token|address)\/(0x[a-fA-F0-9]{40})/i,
  /pancakeswap\.finance.*outputCurrency=(0x[a-fA-F0-9]{40})/i,
  /ave\.ai\/token\/(0x[a-fA-F0-9]{40})/i,
  /dextools\.io.*\/(0x[a-fA-F0-9]{40})/i,
  /birdeye\.so\/token\/(0x[a-fA-F0-9]{40})/i,
];
```

### 2.3.3 UI 框架选择

| 框架 | 包大小 | 性能 | 生态 | 学习曲线 | Chrome 扩展适配 |
|------|--------|------|------|----------|----------------|
| **纯原生 JS** | 0 KB | 最快 | 无 | 高 | 最佳 |
| **Svelte** | ~5 KB | 极快 (编译时) | 中 | 低 | 优秀 |
| **Preact** | ~3 KB | 快 | React 兼容 | 低 | 优秀 |
| **React** | ~40 KB | 良好 | 最大 | 中 | 良好 |
| **SolidJS** | ~7 KB | 极快 | 小 | 中 | 良好 |

### 最终建议：**Svelte 5**

理由：
1. **编译时框架** — 没有运行时开销，最终产物是纯 JS
2. **包体积最小** — Chrome 扩展对加载速度敏感
3. **响应式原语** — `$state` / `$derived` 非常适合金融数据的实时更新
4. **无虚拟 DOM** — 直接操作 DOM，适合侧边栏这种轻量场景
5. **组件化** — 比纯原生 JS 更易维护，比 React 更轻量

Freedom Trader 用的是纯原生 JS + 手动 DOM 操作，虽然性能最好，但代码维护性差（大量 `$('id').innerHTML = ...`）。Svelte 在保持接近原生性能的同时大幅提升开发体验。

## 2.4 私钥与配置存储方案

### 方案对比

#### 方案 A: 纯本地存储（Freedom Trader 方案）

```
┌──────────────────────────────────┐
│ chrome.storage.local             │
│ ┌────────────────────────────┐   │
│ │ ft_salt: base64            │   │  ← PBKDF2 盐值
│ │ ft_pw_hash: base64         │   │  ← 密码哈希
│ │ wallets: [{                │   │
│ │   id, name, address,       │   │
│ │   encryptedKey: "enc:..."  │   │  ← AES-256-GCM 加密的私钥
│ │ }]                         │   │
│ └────────────────────────────┘   │
│                                  │
│ chrome.storage.session           │
│ ┌────────────────────────────┐   │
│ │ ft_pw: string              │   │  ← 密码明文 (TRUSTED_CONTEXTS)
│ │ ft_ut: timestamp           │   │  ← 最后活动时间
│ └────────────────────────────┘   │
└──────────────────────────────────┘
```

**优点**：
- 安全性最高——私钥永远不离开本机
- 无服务器依赖——离线可用
- 无网络攻击面——不存在"服务器被黑"的风险
- 用户完全控制数据

**缺点**：
- 跨设备不同步——换电脑需要重新导入私钥
- 备份靠用户自己——用户忘记备份私钥就丢了
- Chrome 数据清除会丢失所有数据

#### 方案 B: 服务器存储

```
┌──────────────────┐      WSS       ┌─────────────────────┐
│ Chrome 扩展       │ ←───────────→ │ 后端服务器            │
│                  │               │                     │
│ 本地只存:         │               │ 存储:                │
│ - 会话 token     │               │ - 加密私钥密文       │
│ - 缓存配置       │               │ - 配置               │
│                  │               │ - 监控列表           │
└──────────────────┘               └─────────────────────┘
```

**优点**：
- 跨设备同步
- 可备份恢复
- 可做云端监控/推送

**缺点**：
- 致命安全风险——服务器存储私钥（即使是密文），一旦被攻击+密码泄露=全军覆没
- 需要维护服务器——成本、可用性、运维
- 用户信任门槛高——"你要存我的私钥？"
- 合规风险——可能被视为托管服务

**如果选方案 B 的加密设计**：

```
端对端加密方案:
① 用户设置主密码 (master password)
② 客户端: PBKDF2(password, salt, 600000) → masterKey
③ 客户端: AES-256-GCM(masterKey, privateKey) → ciphertext
④ 上传 ciphertext 到服务器 (服务器只存密文)
⑤ 服务器永远不知道 masterKey / privateKey

↓ 问题：
- 密码强度完全取决于用户
- 弱密码 + 密文泄露 = 可暴力破解
- 需要 KDF 迭代次数非常高 (>600000)
- 仍然比本地存储多了一个攻击面（服务器被入侵获取密文）
```

#### 方案 C: 混合方案

```
┌──────────────────┐      WSS       ┌─────────────────────┐
│ Chrome 扩展       │ ←───────────→ │ 后端服务器            │
│                  │               │                     │
│ 本地存储:         │               │ 存储:                │
│ - 加密私钥 ★     │               │ - 监控列表           │
│ - 密码 hash      │               │ - 用户配置           │
│ - salt           │               │ - 实时价格推送       │
│                  │               │ - 交易通知           │
└──────────────────┘               └─────────────────────┘
```

**私钥永远不上服务器，配置和监控走云端。**

### WSS 的具体用途

WSS 连接适合以下场景：

1. **实时价格推送** — 服务器聚合多个 DEX 价格，推送给客户端，减少客户端 RPC 调用
2. **交易状态通知** — 交易上链后推送确认通知
3. **监控列表同步** — 用户在 A 设备加了监控代币，B 设备实时同步
4. **配置同步** — RPC 设置、gas 策略、快捷按钮配置等
5. **新代币发现** — 服务器监听链上新部署的 Meme 代币，推送给用户

**不适合 WSS 的场景**：
- 私钥传输（绝对不行）
- 交易签名（必须本地完成）

### 最终建议：**Phase 1 纯本地 → Phase 2 混合方案**

#### Phase 1（MVP）

- 完全复用 Freedom Trader 的加密方案：PBKDF2 + AES-256-GCM
- 私钥加密存储在 chrome.storage.local
- 密码缓存在 chrome.storage.session (TRUSTED_CONTEXTS)
- 不需要服务器

#### Phase 2（增强版）

- 新增 WSS 连接（可选，不影响核心交易功能）
- WSS 用途：价格推送 + 配置同步 + 监控列表
- 私钥仍然纯本地
- 用户可选择是否启用云同步

#### Phase 2 WSS 方案设计

```
WSS 协议:
  连接: wss://api.xxx.com/ws?token=jwt
  认证: 用户名+密码注册，JWT token 鉴权

消息格式 (JSON):
  // 价格订阅
  → {"type": "subscribe", "channel": "price", "tokens": ["0x..."]}
  ← {"type": "price", "token": "0x...", "price": "0.00123", "change24h": "-5.2%"}

  // 配置同步
  → {"type": "sync", "data": {"rpcUrl": "...", "gas": 3, "slippage": 15}}
  ← {"type": "sync_ack"}

  // 监控推送
  ← {"type": "alert", "token": "0x...", "event": "price_drop", "details": {...}}

加密层:
  - WSS 本身是 TLS 加密
  - 配置数据可选额外加密 (用本地 masterKey)
  - 私钥相关数据永远不上传
```

---

# Part 3: 完整开发规格文档

## 3.1 技术栈选型

| 层级 | 技术 | 理由 |
|------|------|------|
| **前端框架** | Svelte 5 | 编译时框架，极小包体积，响应式原语适合金融数据 |
| **构建工具** | Vite + @crxjs/vite-plugin | 原生 HMR，Chrome 扩展专用插件 |
| **链上库** | viem | 类型安全，tree-shakeable，Freedom Trader 已验证 |
| **UI 样式** | Tailwind CSS | 原子化 CSS，配合 Svelte 使用体积极小 |
| **状态管理** | Svelte $state + 简单 store | 不需要 Redux 等重量级方案 |
| **存储** | chrome.storage.local/session | 与 Freedom Trader 一致 |
| **TypeScript** | 全量 TS | 类型安全，减少运行时错误 |
| **测试** | Vitest | 与 Vite 同生态，快速 |

### 关键依赖版本

```json
{
  "devDependencies": {
    "@crxjs/vite-plugin": "^2.0.0",
    "svelte": "^5.0.0",
    "vite": "^6.0.0",
    "typescript": "^5.5.0",
    "tailwindcss": "^4.0.0",
    "vitest": "^3.0.0"
  },
  "dependencies": {
    "viem": "^2.20.0"
  }
}
```

## 3.2 项目结构

```
bsc-trader/
├── manifest.json                 # Chrome MV3 manifest
├── vite.config.ts                # Vite + CRXJS 配置
├── svelte.config.js
├── tailwind.config.ts
├── tsconfig.json
├── package.json
│
├── src/
│   ├── background/               # Service Worker (隔离域)
│   │   ├── index.ts              # SW 入口，消息路由
│   │   ├── crypto.ts             # PBKDF2 + AES-256-GCM
│   │   ├── wallet-manager.ts     # 私钥解密、WalletClient 管理
│   │   └── url-detector.ts       # URL 合约地址识别
│   │
│   ├── lib/                      # 共享库（前端 + background）
│   │   ├── types.ts              # 全局类型定义
│   │   ├── constants.ts          # 合约地址、ABI、RPC 列表
│   │   ├── abi/                  # ABI 文件
│   │   │   ├── pancake-router.ts
│   │   │   ├── pancake-factory.ts
│   │   │   ├── erc20.ts
│   │   │   ├── four-helper3.ts
│   │   │   ├── four-token-manager.ts
│   │   │   ├── flap-portal.ts
│   │   │   └── four-token.ts
│   │   └── utils.ts              # 通用工具函数
│   │
│   ├── core/                     # 核心业务逻辑
│   │   ├── messaging.ts          # chrome.runtime.sendMessage 代理
│   │   ├── rpc-client.ts         # viem PublicClient 创建 + fallback
│   │   ├── route-detector.ts     # 路由检测（前端版本）
│   │   ├── trading.ts            # 买入/卖出核心逻辑
│   │   ├── batch.ts              # 多钱包批量交易
│   │   ├── approve-cache.ts      # Approve 缓存管理
│   │   ├── gas-strategy.ts       # Gas 动态策略
│   │   ├── price-quoter.ts       # 报价查询
│   │   └── balance-loader.ts     # 余额加载
│   │
│   ├── stores/                   # Svelte stores
│   │   ├── wallet.svelte.ts      # 钱包状态
│   │   ├── token.svelte.ts       # 当前代币状态
│   │   ├── trade.svelte.ts       # 交易模式/参数
│   │   └── config.svelte.ts      # 用户配置
│   │
│   ├── panel/                    # 侧边栏 UI
│   │   ├── Panel.svelte          # 主面板
│   │   ├── components/
│   │   │   ├── TokenInput.svelte     # 代币地址输入
│   │   │   ├── TradePanel.svelte     # 买入/卖出面板
│   │   │   ├── QuickButtons.svelte   # 快捷按钮
│   │   │   ├── WalletSelector.svelte # 钱包选择器
│   │   │   ├── BalanceDisplay.svelte # 余额显示
│   │   │   ├── PriceInfo.svelte      # 价格/报价
│   │   │   ├── LPInfo.svelte         # 流动性信息
│   │   │   ├── StatusBar.svelte      # 状态栏
│   │   │   ├── GasControl.svelte     # Gas 控制
│   │   │   └── SlippageControl.svelte # 滑点控制
│   │   ├── panel.html
│   │   └── panel.ts              # 侧边栏入口
│   │
│   ├── settings/                 # 设置页面 (独立标签页)
│   │   ├── Settings.svelte
│   │   ├── pages/
│   │   │   ├── WalletManager.svelte  # 钱包管理
│   │   │   ├── RpcConfig.svelte      # RPC 配置
│   │   │   ├── SecuritySettings.svelte # 密码/锁定
│   │   │   └── TradeSettings.svelte  # 交易参数
│   │   ├── settings.html
│   │   └── settings.ts
│   │
│   └── styles/
│       └── global.css            # Tailwind + 自定义变量
│
├── tests/
│   ├── unit/
│   │   ├── route-detector.test.ts
│   │   ├── approve-cache.test.ts
│   │   ├── gas-strategy.test.ts
│   │   └── utils.test.ts
│   └── integration/
│       └── trading-flow.test.ts
│
└── scripts/
    └── build.ts                  # 构建脚本
```

## 3.3 模块拆分与接口定义

### 3.3.1 Background Service Worker (`src/background/`)

**职责**: 私钥管理、交易签名、URL 检测

```typescript
// src/background/index.ts — 消息处理器注册

type MessageHandlers = {
  // 密码管理
  setPassword: (data: { password: string }) => Promise<{ ok: boolean }>;
  unlock: (data: { password: string }) => Promise<{ ok: boolean }>;
  lock: () => Promise<{ ok: boolean }>;
  isUnlocked: () => Promise<{ unlocked: boolean }>;
  hasPassword: () => Promise<{ has: boolean }>;

  // 钱包管理
  initWallets: (data: { rpcUrl: string }) => Promise<{ addresses: Record<string, string> }>;
  getWalletAddress: (data: { walletId: string }) => Promise<{ address: string | null }>;

  // 交易签名
  signAndSendTx: (data: {
    walletId: string;
    address: string;       // 合约地址
    abi: any[];
    functionName: string;
    args: SerializedArg[];
    value?: string;        // BigInt as string
    gas: string;
    gasPrice: string;
  }) => Promise<{ txHash: string }>;

  // 原始交易签名 (用于多 RPC 竞速)
  signRawTx: (data: {
    walletId: string;
    to: string;
    data: string;          // calldata hex
    value?: string;
    gas: string;
    gasPrice: string;
    nonce: number;
  }) => Promise<{ rawTx: string }>; // 返回签名后的 raw tx

  // 加密
  encrypt: (data: { plaintext: string; password?: string }) => Promise<{ result: string }>;

  // 设置
  getLockDuration: () => Promise<{ duration: number }>;
  setLockDuration: (data: { minutes: number }) => Promise<{ ok: boolean }>;
  resetAll: () => Promise<{ ok: boolean }>;
};
```

```typescript
// src/background/crypto.ts

export async function deriveKey(password: string): Promise<CryptoKey>;
export async function encrypt(key: CryptoKey, plaintext: string): Promise<string>;
export async function decrypt(key: CryptoKey, ciphertext: string): Promise<string | null>;
export async function hashPassword(password: string): Promise<string>;
```

```typescript
// src/background/wallet-manager.ts

export class WalletManager {
  private clients: Map<string, { client: WalletClient; account: Account }>;
  private publicClient: PublicClient;

  async buildClients(wallets: EncryptedWallet[], rpcUrl: string): Promise<Record<string, string>>;
  async writeContract(walletId: string, params: WriteContractParams): Promise<string>; // txHash
  async signTransaction(walletId: string, params: RawTxParams): Promise<string>; // rawTx hex
  getAddress(walletId: string): string | null;
  clear(): void;
}
```

### 3.3.2 Core 业务逻辑 (`src/core/`)

```typescript
// src/core/messaging.ts — Background 消息代理

export async function sendMessage<T extends keyof MessageHandlers>(
  action: T,
  data?: Parameters<MessageHandlers[T]>[0]
): Promise<ReturnType<MessageHandlers[T]>>;

// 便捷方法
export async function bscWriteContract(walletId: string, params: TxParams): Promise<{ txHash: string }>;
export async function bscSignRawTx(walletId: string, params: RawTxParams): Promise<{ rawTx: string }>;
export async function encryptPrivateKey(key: string, password?: string): Promise<string>;
```

```typescript
// src/core/route-detector.ts — 前端路由检测

export enum RouteSource {
  NONE = 0,
  FOUR_INTERNAL_BNB = 1,
  FOUR_INTERNAL_ERC20 = 2,
  FOUR_EXTERNAL = 3,
  FLAP_BONDING = 4,
  FLAP_BONDING_SELL = 5,
  FLAP_DEX = 6,
  PANCAKE_ONLY = 7,
}

export interface DetectedRoute {
  source: RouteSource;
  approveTarget: Address;      // 卖出时 approve 给谁
  quoteToken: Address;          // 报价代币地址
  isInternal: boolean;          // 是否内盘
  label: string;                // UI 显示标签
  protocol: 'four' | 'flap' | 'pancake' | 'none';

  // Four.meme 特有
  fourInfo?: {
    tokenManager: Address;
    quote: Address;
    funds: bigint;
    maxFunds: bigint;
    offers: bigint;
    liquidityAdded: boolean;
  };

  // Flap 特有
  flapInfo?: {
    status: number;
    reserve: bigint;
    price: bigint;
    quoteToken: Address;
    nativeSwapEnabled: boolean;
    taxRate: bigint;
  };

  // PancakeSwap 特有
  pancakeInfo?: {
    pair: Address;
    quoteToken: Address;
    reserve0: bigint;
    reserve1: bigint;
  };
}

export async function detectRoute(
  publicClient: PublicClient,
  tokenAddr: Address
): Promise<DetectedRoute>;

// 内部函数（并行调用各协议）
async function detectFourMeme(client: PublicClient, token: Address): Promise<FourMemeResult | null>;
async function detectFlap(client: PublicClient, token: Address): Promise<FlapResult | null>;
async function detectPancakeSwap(client: PublicClient, token: Address): Promise<PancakeResult | null>;
async function findBestQuote(client: PublicClient, token: Address): Promise<{ quote: Address; pair: Address; liquidity: bigint } | null>;
```

```typescript
// src/core/trading.ts — 交易核心

export interface TradeResult {
  txHash: string;
  sendMs: number;
  confirmMs: number;
  totalMs: number;
}

export async function buy(
  walletId: string,
  tokenAddr: Address,
  amountBNB: string,      // e.g. "0.1"
  gasPrice: number,       // Gwei
  slippage: number,       // percentage, e.g. 15
  route: DetectedRoute
): Promise<TradeResult>;

export async function sell(
  walletId: string,
  tokenAddr: Address,
  amountToken: string,    // token 数量
  gasPrice: number,
  slippage: number,
  route: DetectedRoute
): Promise<TradeResult>;

// 内部函数 — 按协议分发
async function buyFourInternal(walletId: string, token: Address, amount: bigint, minOut: bigint, gasPrice: bigint, fourInfo: FourMemeInfo): Promise<string>;
async function buyFourExternal(walletId: string, token: Address, amount: bigint, minOut: bigint, gasPrice: bigint, deadline: bigint, quoteToken: Address): Promise<string>;
async function buyFlap(walletId: string, token: Address, amount: bigint, minOut: bigint, gasPrice: bigint): Promise<string>;
async function buyPancake(walletId: string, token: Address, amount: bigint, minOut: bigint, gasPrice: bigint, deadline: bigint, quoteToken: Address): Promise<string>;

async function sellFourInternal(walletId: string, token: Address, amount: bigint, minOut: bigint, gasPrice: bigint, fourInfo: FourMemeInfo): Promise<string>;
async function sellPancake(walletId: string, token: Address, amount: bigint, minOut: bigint, gasPrice: bigint, deadline: bigint, quoteToken: Address): Promise<string>;
async function sellFlap(walletId: string, token: Address, amount: bigint, minOut: bigint, gasPrice: bigint): Promise<string>;
```

```typescript
// src/core/batch.ts — 批量交易

export async function executeBatchTrade(
  walletIds: string[],
  tokenAddr: Address,
  amount: string,
  mode: 'buy' | 'sell',
  gasPrice: number,
  slippage: number,
  route: DetectedRoute
): Promise<BatchResult>;

export async function fastBuy(
  walletIds: string[],
  tokenAddr: Address,
  amount: string,
  gasPrice: number,
  slippage: number,
  route: DetectedRoute
): Promise<BatchResult>;

export async function fastSell(
  walletIds: string[],
  tokenAddr: Address,
  percentage: number,      // 25, 50, 75, 100
  tokenBalances: Map<string, bigint>,
  tokenDecimals: number,
  gasPrice: number,
  slippage: number,
  route: DetectedRoute
): Promise<BatchResult>;

interface BatchResult {
  success: number;
  failed: number;
  results: Array<{ walletId: string; status: 'fulfilled' | 'rejected'; value?: TradeResult; reason?: string }>;
  totalMs: number;
}
```

```typescript
// src/core/approve-cache.ts

export class ApproveCache {
  private approved: Set<string>;           // "owner:token:spender"
  private inFlight: Map<string, Promise<void>>;

  async load(): Promise<void>;             // 从 chrome.storage.local 加载
  async ensureApproved(
    walletId: string,
    ownerAddr: Address,
    tokenAddr: Address,
    spender: Address,
    gasPrice: bigint,
    minAllowance: bigint
  ): Promise<void>;

  isApproved(owner: Address, token: Address, spender: Address): boolean;
}
```

```typescript
// src/core/price-quoter.ts — 报价查询

export async function getQuoteBuy(
  publicClient: PublicClient,
  route: DetectedRoute,
  tokenAddr: Address,
  amountIn: bigint
): Promise<bigint>;  // 预估获得的代币数量

export async function getQuoteSell(
  publicClient: PublicClient,
  route: DetectedRoute,
  tokenAddr: Address,
  amountIn: bigint
): Promise<bigint>;  // 预估获得的 BNB/quote 数量

export async function getTokenPrice(
  publicClient: PublicClient,
  route: DetectedRoute,
  tokenAddr: Address,
  tokenDecimals: number
): Promise<number>;  // 代币价格 (以 quote token 计)
```

```typescript
// src/core/gas-strategy.ts

export interface GasPreset {
  name: string;
  gwei: number;
}

export const GAS_PRESETS: GasPreset[] = [
  { name: '标准', gwei: 1 },
  { name: '快速', gwei: 3 },
  { name: '极速', gwei: 5 },
  { name: '闪电', gwei: 10 },
];

export async function getRecommendedGasPrice(publicClient: PublicClient): Promise<number>;
```

### 3.3.3 Svelte Stores (`src/stores/`)

```typescript
// src/stores/wallet.svelte.ts

export const walletStore = {
  // 状态
  wallets: $state<WalletInfo[]>([]),
  activeIds: $state<string[]>([]),
  balances: $state<Map<string, bigint>>(new Map()),
  quoteBalances: $state<Map<string, bigint>>(new Map()),

  // 计算
  get activeWallets() { ... },
  get totalBalance() { ... },

  // 操作
  async init(rpcUrl: string): Promise<void>,
  async loadBalances(): Promise<void>,
  toggleActive(id: string): void,
};
```

```typescript
// src/stores/token.svelte.ts

export const tokenStore = {
  address: $state(''),
  info: $state<TokenInfo>({ decimals: 18, symbol: '', balance: 0n }),
  route: $state<DetectedRoute | null>(null),
  tokenBalances: $state<Map<string, bigint>>(new Map()),
  detecting: $state(false),

  async detect(addr: string): Promise<void>,
  clear(): void,
};
```

```typescript
// src/stores/trade.svelte.ts

export const tradeStore = {
  mode: $state<'buy' | 'sell'>('buy'),
  amount: $state(''),
  gasPrice: $state(3),
  slippage: $state(15),
  trading: $state(false),
  lastResult: $state<BatchResult | null>(null),

  async executeTrade(): Promise<void>,
  async fastBuy(amount: string): Promise<void>,
  async fastSell(pct: number): Promise<void>,
};
```

## 3.4 前端架构

### 3.4.1 侧边栏布局

```
┌─────────────────────────────┐ 350px
│ [BSC]              [⚙️设置] │ <- 顶部栏
├─────────────────────────────┤
│ [代币地址输入框]      [检测] │
│ 🔥 PEPE • Four 内盘 • 0.001│ <- 代币信息条
├─────────────────────────────┤
│ BNB 余额: 1.2345            │
│ [钱包1 ✓] [钱包2 ✓] [钱包3]│ <- 钱包选择器
├─────────────────────────────┤
│ [买入] [卖出]               │ <- 模式切换
│                             │
│ 数量 (BNB/钱包):            │
│ [0.01] [0.05] [0.1] [0.5]  │ <- 快捷买入
│ [___________________] [MAX] │ <- 金额输入
│                             │
│ 滑点: [5] [10] [15] [25]   │
│ Gas:  [1] [3] [5] [_____]  │
│                             │
│ ≈ 1,234,567 PEPE × 2       │ <- 预估
│ ≥ 1,049,382 PEPE            │ <- 最低获得
│                             │
│ [        🚀 买入        ]   │ <- 交易按钮
├─────────────────────────────┤
│ ⚡快捷交易                   │
│ [买0.01] [买0.05] [买0.1]   │
│ [卖25%] [卖50%] [卖75%][全] │
├─────────────────────────────┤
│ LP 信息                      │
│ 类型: 🔥 Four 内盘          │
│ BNB 储备: 12.34             │
│ Token 储备: 1,234,567       │
├─────────────────────────────┤
│ ✓ 全部成功 | 3.2s           │ <- 状态栏
└─────────────────────────────┘
```

### 3.4.2 组件层级

```
Panel.svelte
├── TokenInput.svelte           # 地址输入 + 检测状态 + 代币标签
├── WalletSelector.svelte       # 钱包选择 + 余额显示
├── TradePanel.svelte           # 买卖切换 + 金额输入 + 快捷按钮
│   ├── QuickButtons.svelte     # 自定义快捷金额按钮
│   ├── SlippageControl.svelte  # 滑点选择
│   └── GasControl.svelte       # Gas 价格控制
├── PriceInfo.svelte            # 预估价格 + 最低获得
├── QuickTrade.svelte           # 快捷买卖面板
├── LPInfo.svelte               # 流动性信息
├── StatusBar.svelte            # 交易状态
└── Toast.svelte                # 通知弹窗
```

### 3.4.3 数据流

```
URL 变化 → background/url-detector → CONTRACT_DETECTED 消息
                                       ↓
侧边栏接收 → tokenStore.detect(addr)
                ↓
route-detector.ts#detectRoute() ← publicClient (并行调用 Four/Flap/PCS)
                ↓
tokenStore.route 更新 → UI 响应式更新
                ↓
用户点击买入 → tradeStore.executeTrade()
                ↓
batch.ts#executeBatchTrade()
  ↓ 每个钱包并行:
trading.ts#buy() → 根据 route.source 分发
  ↓
messaging.ts#bscWriteContract() → chrome.runtime.sendMessage
  ↓
background/wallet-manager.ts → viem writeContract → RPC → txHash
  ↓
publicClient.waitForTransactionReceipt() → 确认
  ↓
UI 更新交易结果
```

## 3.5 安全设计

### 3.5.1 私钥保护

完全复用 Freedom Trader 的方案：

```
加密栈:
  PBKDF2(password, salt_16bytes, 100000, SHA-256) → AES-256-GCM key
  AES-256-GCM(key, iv_12bytes, private_key) → ciphertext
  存储格式: "enc:" + base64(iv + ciphertext)

存储位置:
  - 加密私钥: chrome.storage.local["wallets"][].encryptedKey
  - Salt: chrome.storage.local["ft_salt"]
  - 密码哈希: chrome.storage.local["ft_pw_hash"]
  - 会话密码: chrome.storage.session["ft_pw"] (TRUSTED_CONTEXTS only)
```

### 3.5.2 安全边界

```
前端 (Side Panel / Settings Page):
  ✗ 无法访问私钥明文
  ✗ 无法读取 session storage (TRUSTED_CONTEXTS)
  ✓ 只能通过 sendMessage 请求签名
  ✓ 只能拿到 address 和 txHash

Background (Service Worker):
  ✓ 持有解密后的私钥 (内存中)
  ✓ 验证 sender.id === chrome.runtime.id
  ✓ 超时自动清除密钥
  ✓ 锁定后立即清除所有密钥
```

### 3.5.3 XSS 防护

```typescript
// 1. CSP 配置 (manifest.json)
"content_security_policy": {
  "extension_pages": "script-src 'self'; object-src 'self'"
}

// 2. 所有用户输入 escapeHtml
function escapeHtml(str: string): string {
  const div = document.createElement('div');
  div.textContent = str;
  return div.innerHTML;
}

// 3. Svelte 默认转义（{variable} 自动 escape，{@html} 需要显式标记）
// 绝不在用户输入上使用 {@html}
```

### 3.5.4 输入验证

```typescript
// 地址验证
function isValidBscAddress(addr: string): boolean {
  return /^0x[a-fA-F0-9]{40}$/.test(addr);
}

// 金额验证（防精度问题）
function normalizeAmount(input: string, maxDecimals: number): string {
  // 复用 Freedom Trader 的 sanitizeAmountInput + normalizeAmount
}

// 私钥格式验证（只在导入时，不在运行时）
function isValidPrivateKey(key: string): boolean {
  const hex = key.startsWith('0x') ? key : '0x' + key;
  return /^0x[a-fA-F0-9]{64}$/.test(hex);
}
```

## 3.6 RPC 与速度优化方案

### 3.6.1 RPC 层设计

```typescript
// src/core/rpc-client.ts

import { createPublicClient, http, fallback } from 'viem';
import { bsc } from 'viem/chains';

const DEFAULT_RPC = 'https://bsc-dataseed.bnbchain.org';
const FALLBACK_RPCS = [
  'https://bsc-dataseed.bnbchain.org',
  'https://bsc-dataseed1.bnbchain.org',
  'https://bsc-dataseed2.bnbchain.org',
  'https://bsc.publicnode.com',
  'https://bsc-dataseed1.defibit.io',
];

// 读取客户端（支持 fallback）
export function createReadClient(customRpc?: string): PublicClient {
  const urls = customRpc ? [customRpc, ...FALLBACK_RPCS] : FALLBACK_RPCS;
  return createPublicClient({
    chain: bsc,
    transport: fallback(urls.map(u => http(u, { timeout: 10_000 }))),
  });
}

// 写入客户端（在 background 中创建，使用用户配置的防夹 RPC）
// 由 wallet-manager.ts 管理
```

### 3.6.2 速度优化清单

| 优化项 | 实现 | 预估收益 |
|--------|------|----------|
| Approve 缓存 | 内存 + 持久化双层缓存 | -3~6s (省掉 approve 交易) |
| 买入后预授权 | 买入成功后并行 approve 卖出目标 | -3~6s (下次卖出时) |
| 并行路由检测 | Promise.all 同时查 Four/Flap/PCS | 延迟不叠加 |
| 多 RPC 竞速 | signRawTx 后同时发 3 个 RPC | -50~100ms |
| 报价预取 | 300ms 防抖后自动查询 | 感知延迟更低 |
| 防夹 RPC | 推荐 48.club / BlockRazor | 防止被夹 |
| Deadline 优化 | 10 秒 deadline (非 300 秒) | 快速失败 |
| Gas 预设 | 快捷按钮 + 动态建议 | 减少手动输入 |

### 3.6.3 多 RPC 竞速实现

```typescript
// src/core/trading.ts 中的竞速发送

async function raceSubmitTx(
  walletId: string,
  txParams: TxParams,
  rpcUrls: string[]
): Promise<string> {
  // 1. 在 background 中签名得到 raw tx
  const { rawTx } = await bscSignRawTx(walletId, {
    to: txParams.address,
    data: encodeFunctionData(txParams),
    value: txParams.value,
    gas: txParams.gas,
    gasPrice: txParams.gasPrice,
    nonce: await getNonce(walletId),
  });

  // 2. 并行发送到多个 RPC
  const txHash = await Promise.any(
    rpcUrls.map(rpc => sendRawTx(rpc, rawTx))
  );

  return txHash;
}

async function sendRawTx(rpcUrl: string, rawTx: string): Promise<string> {
  const response = await fetch(rpcUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      id: 1,
      method: 'eth_sendRawTransaction',
      params: [rawTx],
    }),
  });
  const result = await response.json();
  if (result.error) throw new Error(result.error.message);
  return result.result;
}
```

## 3.7 开发阶段规划

### Phase 1: MVP — 核心交易功能 (2-3 周)

**目标**: 能跑通 PancakeSwap 买入/卖出 + 多钱包管理

| 任务 | 模块 | 优先级 |
|------|------|--------|
| 项目脚手架 (Vite + Svelte + MV3) | 基础 | P0 |
| Background Service Worker (密码 + 加密) | `background/` | P0 |
| 钱包管理页面 (导入/删除/选择) | `settings/` | P0 |
| RPC 客户端 + PancakeSwap 路由检测 | `core/rpc-client.ts` + `route-detector.ts` | P0 |
| PancakeSwap 买入/卖出 | `core/trading.ts` | P0 |
| 侧边栏主界面 | `panel/` | P0 |
| 代币检测 + 余额显示 | `stores/token.svelte.ts` | P0 |
| 多钱包批量交易 | `core/batch.ts` | P0 |
| Approve 缓存 | `core/approve-cache.ts` | P1 |
| URL 自动识别 | `background/url-detector.ts` | P1 |
| Gas 策略 | `core/gas-strategy.ts` | P1 |

**Phase 1 交付物**: 可以在 PancakeSwap 上用多个钱包并行买卖任意 BEP20 代币。

### Phase 2: 协议扩展 — Four.meme + Flap (1-2 周)

| 任务 | 模块 | 优先级 |
|------|------|--------|
| Four.meme 检测 (IHelper3) | `route-detector.ts` | P0 |
| Four.meme 内盘买入 (TokenManagerV2) | `trading.ts` | P0 |
| Four.meme 内盘卖出 | `trading.ts` | P0 |
| Four.meme ERC20 quote 支持 (Helper3) | `trading.ts` | P1 |
| Flap 检测 (IFlapPortal.getTokenV7) | `route-detector.ts` | P0 |
| Flap 内盘买卖 (Portal.swapExactInput) | `trading.ts` | P0 |
| Tax Token 兼容 (SupportingFeeOnTransfer) | `trading.ts` | P1 |
| 报价查询优化 (并行 + 缓存) | `price-quoter.ts` | P1 |
| LP 信息展示 (储备量/进度) | `panel/LPInfo.svelte` | P2 |

**Phase 2 交付物**: 完整支持 BSC 上 Four.meme / Flap / PancakeSwap 三大协议交易。

### Phase 3: 优化与增强 (1-2 周)

| 任务 | 模块 | 优先级 |
|------|------|--------|
| 多 RPC 竞速发送 | `trading.ts` | P1 |
| 快捷按钮自定义 | `panel/QuickButtons.svelte` | P1 |
| 交易历史记录 | `settings/TradeHistory.svelte` | P2 |
| 自定义快捷键 | 全局 | P2 |
| 深色/浅色主题 | 全局 | P2 |
| ERC20 Quote Token 支持 (USDT/USDC 计价) | `stores/` + `core/` | P2 |
| 导出/导入配置 (备份) | `settings/` | P2 |
| 性能优化 (减少重渲染) | 全局 | P3 |

## 3.8 各模块详细实现指引

### 3.8.1 Background Service Worker 实现指引

#### 文件: `src/background/index.ts`

```typescript
// 核心要点:
// 1. Service Worker 可能被 Chrome 杀死后重启，必须从 session storage 恢复状态
// 2. 所有 handler 在执行前必须 checkExpiry() 验证会话有效
// 3. sender 验证防止外部扩展调用

import { WalletManager } from './wallet-manager';
import { deriveKey, hashPassword, encrypt, decrypt } from './crypto';

// Session storage 限制为 TRUSTED_CONTEXTS
chrome.storage.session.setAccessLevel({ accessLevel: 'TRUSTED_CONTEXTS' });

// 侧边栏行为
chrome.sidePanel.setPanelBehavior({ openPanelOnActionClick: true });

let cryptoKey: CryptoKey | null = null;
let unlockTime = 0;
const walletManager = new WalletManager();

// 会话恢复（SW 重启后自动恢复）
async function restoreSession() {
  const s = await chrome.storage.session.get(['pw', 'ut']);
  if (!s.pw) return;
  cryptoKey = await deriveKey(s.pw);
  unlockTime = s.ut || Date.now();
}

// 活动超时检查（N 分钟不活动后锁定）
async function checkExpiry(): Promise<boolean> {
  if (!cryptoKey) await restoreSession();
  if (!cryptoKey) return false;
  const config = await chrome.storage.local.get(['lockDuration']);
  const dur = (config.lockDuration || 240) * 60 * 1000;
  if (Date.now() - unlockTime > dur) {
    await clearSession();
    return false;
  }
  unlockTime = Date.now();
  chrome.storage.session.set({ ut: unlockTime });
  return true;
}

// 消息路由
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.type === 'CONTRACT_DETECTED') return; // 转发，不处理
  if (sender.id !== chrome.runtime.id) return;       // 安全检查

  const handler = handlers[message.action];
  if (handler) {
    handler(message)
      .then(sendResponse)
      .catch(e => sendResponse({ error: e.message }));
    return true; // 异步响应
  }
});
```

#### 文件: `src/background/crypto.ts`

```typescript
// 完全复用 Freedom Trader 的加密方案
// 要点:
// 1. Salt 16 字节，持久化到 chrome.storage.local
// 2. PBKDF2 100000 iterations，SHA-256
// 3. AES-256-GCM，12 字节 IV
// 4. 存储格式: "enc:" + base64(IV + ciphertext)

const SALT_KEY = 'ft_salt';
const ITERATIONS = 100_000;

export async function getSalt(): Promise<Uint8Array> {
  const stored = await chrome.storage.local.get([SALT_KEY]);
  if (stored[SALT_KEY]) {
    return Uint8Array.from(atob(stored[SALT_KEY]), c => c.charCodeAt(0));
  }
  const salt = crypto.getRandomValues(new Uint8Array(16));
  await chrome.storage.local.set({ [SALT_KEY]: uint8ToBase64(salt) });
  return salt;
}

export async function deriveKey(password: string): Promise<CryptoKey> {
  const salt = await getSalt();
  const keyMaterial = await crypto.subtle.importKey(
    'raw', new TextEncoder().encode(password), 'PBKDF2', false, ['deriveKey']
  );
  return crypto.subtle.deriveKey(
    { name: 'PBKDF2', salt, iterations: ITERATIONS, hash: 'SHA-256' },
    keyMaterial,
    { name: 'AES-GCM', length: 256 },
    false,
    ['encrypt', 'decrypt']
  );
}

export async function encrypt(key: CryptoKey, plaintext: string): Promise<string> {
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const data = new TextEncoder().encode(plaintext);
  const encrypted = await crypto.subtle.encrypt({ name: 'AES-GCM', iv }, key, data);
  const combined = new Uint8Array(12 + encrypted.byteLength);
  combined.set(iv);
  combined.set(new Uint8Array(encrypted), 12);
  return 'enc:' + uint8ToBase64(combined);
}

export async function decrypt(key: CryptoKey, ciphertext: string): Promise<string | null> {
  const raw = ciphertext.startsWith('enc:') ? ciphertext.slice(4) : ciphertext;
  try {
    const combined = Uint8Array.from(atob(raw), c => c.charCodeAt(0));
    const iv = combined.slice(0, 12);
    const encrypted = combined.slice(12);
    const decrypted = await crypto.subtle.decrypt({ name: 'AES-GCM', iv }, key, encrypted);
    return new TextDecoder().decode(decrypted);
  } catch {
    return null;
  }
}
```

#### 文件: `src/background/wallet-manager.ts`

```typescript
// 要点:
// 1. 私钥解密后存在内存 Map 中，前端只能拿到地址
// 2. 支持 writeContract (签名+发送) 和 signTransaction (只签名)
// 3. 自动重试: 如果 walletClient 不存在，尝试重新初始化

import { createPublicClient, createWalletClient, http } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { bsc } from 'viem/chains';

interface WalletEntry {
  client: WalletClient;
  account: Account;
}

export class WalletManager {
  private clients = new Map<string, WalletEntry>();

  async buildClients(
    wallets: Array<{ id: string; encryptedKey: string }>,
    rpcUrl: string,
    decryptFn: (ct: string) => Promise<string | null>
  ): Promise<Record<string, string>> {
    this.clients.clear();
    const result: Record<string, string> = {};

    for (const w of wallets) {
      const plain = await decryptFn(w.encryptedKey);
      if (!plain) continue;
      const key = plain.startsWith('0x') ? plain : `0x${plain}`;
      const account = privateKeyToAccount(key as `0x${string}`);
      const client = createWalletClient({
        chain: bsc,
        transport: http(rpcUrl),
        account,
      });
      this.clients.set(w.id, { client, account });
      result[w.id] = account.address;
    }
    return result;
  }

  async writeContract(walletId: string, params: WriteContractParams): Promise<string> {
    const entry = this.clients.get(walletId);
    if (!entry) throw new Error('钱包未初始化: ' + walletId);
    return entry.client.writeContract(params);
  }

  // 用于多 RPC 竞速：只签名不发送
  async signTransaction(walletId: string, params: RawTxParams): Promise<string> {
    const entry = this.clients.get(walletId);
    if (!entry) throw new Error('钱包未初始化: ' + walletId);
    return entry.account.signTransaction(params);
  }

  getAddress(walletId: string): string | null {
    return this.clients.get(walletId)?.account.address ?? null;
  }

  clear(): void {
    this.clients.clear();
  }
}
```

### 3.8.2 路由检测实现指引

#### 文件: `src/core/route-detector.ts`

```typescript
// 核心设计: 并行调用 Four.meme / Flap / PancakeSwap，取最优路由
// 与 FreedomRouter 合约的 _detectRoute 逻辑一致，但在前端实现

import { FOUR_HELPER3, FLAP_PORTAL, PANCAKE_FACTORY, WBNB, USDT, USD1, USDC, BUSD, FDUSD } from '../lib/constants';
import { helper3Abi, flapPortalAbi, pancakeFactoryAbi, pancakePairAbi, fourTokenAbi } from '../lib/abi';

const QUOTE_TOKENS = [WBNB, USDT, USD1, USDC, BUSD, FDUSD] as const;

export async function detectRoute(
  client: PublicClient,
  tokenAddr: Address
): Promise<DetectedRoute> {
  // 并行调用三个协议
  const [fourResult, flapResult, pcsResult] = await Promise.allSettled([
    detectFourMeme(client, tokenAddr),
    detectFlap(client, tokenAddr),
    findBestPancakeQuote(client, tokenAddr),
  ]);

  const four = fourResult.status === 'fulfilled' ? fourResult.value : null;
  const flap = flapResult.status === 'fulfilled' ? flapResult.value : null;
  const pcs = pcsResult.status === 'fulfilled' ? pcsResult.value : null;

  // 优先级: Four.meme > Flap > PancakeSwap
  if (four) {
    if (!four.liquidityAdded && four.mode !== 0n) {
      // Four 内盘
      return {
        source: four.quote === ZERO_ADDR ? RouteSource.FOUR_INTERNAL_BNB : RouteSource.FOUR_INTERNAL_ERC20,
        approveTarget: four.tokenManager, // 内盘 approve 给 TokenManager
        quoteToken: four.quote || WBNB,
        isInternal: true,
        label: four.quote === ZERO_ADDR ? 'Four.meme 内盘 (BNB)' : 'Four.meme 内盘 (ERC20)',
        protocol: 'four',
        fourInfo: four,
      };
    }
    if (four.liquidityAdded) {
      // Four 外盘 → 走 PancakeSwap
      return {
        source: RouteSource.FOUR_EXTERNAL,
        approveTarget: PANCAKE_ROUTER, // 外盘 approve 给 PCS Router
        quoteToken: pcs?.quoteToken || WBNB,
        isInternal: false,
        label: 'Four.meme 外盘',
        protocol: 'four',
        fourInfo: four,
        pancakeInfo: pcs ?? undefined,
      };
    }
  }

  if (flap) {
    if (flap.status === 1) {
      // Flap 内盘
      if (flap.quoteToken === ZERO_ADDR || flap.nativeSwapEnabled) {
        return {
          source: RouteSource.FLAP_BONDING,
          approveTarget: FLAP_PORTAL, // Flap 内盘 approve 给 Portal
          isInternal: true,
          label: 'Flap 内盘',
          protocol: 'flap',
          flapInfo: flap,
          quoteToken: WBNB,
        };
      }
      return {
        source: RouteSource.FLAP_BONDING_SELL,
        approveTarget: FLAP_PORTAL,
        isInternal: true,
        label: 'Flap 内盘 (仅卖)',
        protocol: 'flap',
        flapInfo: flap,
        quoteToken: WBNB,
      };
    }
    if (flap.status === 4) {
      // Flap DEX → 走 PancakeSwap
      return {
        source: RouteSource.FLAP_DEX,
        approveTarget: PANCAKE_ROUTER,
        isInternal: false,
        label: 'Flap DEX',
        protocol: 'flap',
        flapInfo: flap,
        pancakeInfo: pcs ?? undefined,
        quoteToken: pcs?.quoteToken || WBNB,
      };
    }
  }

  if (pcs && pcs.pair !== ZERO_ADDR) {
    return {
      source: RouteSource.PANCAKE_ONLY,
      approveTarget: PANCAKE_ROUTER,
      isInternal: false,
      label: 'PancakeSwap',
      protocol: 'pancake',
      pancakeInfo: pcs,
      quoteToken: pcs.quoteToken,
    };
  }

  return {
    source: RouteSource.NONE,
    approveTarget: ZERO_ADDR,
    isInternal: false,
    label: '无路由',
    protocol: 'none',
    quoteToken: WBNB,
  };
}

// Four.meme 检测
async function detectFourMeme(client: PublicClient, token: Address) {
  try {
    const [ver, tm, quote, lastPrice, tradingFeeRate, , launchTime, offers, maxOffers, funds, maxFunds, liquidityAdded] =
      await client.readContract({
        address: FOUR_HELPER3,
        abi: helper3Abi,
        functionName: 'getTokenInfo',
        args: [token],
      });

    if (ver === 0n || tm === ZERO_ADDR) return null;

    let mode = 0n;
    try {
      mode = await client.readContract({ address: token, abi: fourTokenAbi, functionName: '_mode' });
    } catch {}

    return { version: ver, tokenManager: tm, quote, funds, maxFunds, offers, maxOffers, liquidityAdded, mode };
  } catch {
    return null;
  }
}

// Flap 检测
async function detectFlap(client: PublicClient, token: Address) {
  try {
    const st = await client.readContract({
      address: FLAP_PORTAL,
      abi: flapPortalAbi,
      functionName: 'getTokenV7',
      args: [token],
    });
    if (st.status === 0) return null;
    return {
      status: Number(st.status),
      reserve: st.reserve,
      price: st.price,
      quoteToken: st.quoteTokenAddress,
      nativeSwapEnabled: st.nativeToQuoteSwapEnabled,
      taxRate: st.taxRate,
      pool: st.pool,
      progress: st.progress,
    };
  } catch {
    return null;
  }
}

// PancakeSwap 最佳配对查找
async function findBestPancakeQuote(client: PublicClient, token: Address) {
  const results = await Promise.allSettled(
    QUOTE_TOKENS.map(async (qt) => {
      const pair = await client.readContract({
        address: PANCAKE_FACTORY,
        abi: pancakeFactoryAbi,
        functionName: 'getPair',
        args: [token, qt],
      });
      if (pair === ZERO_ADDR) return null;
      const [r0, r1] = await client.readContract({
        address: pair,
        abi: pancakePairAbi,
        functionName: 'getReserves',
      });
      return { quoteToken: qt, pair, reserve0: r0, reserve1: r1, liquidity: r0 * r1 };
    })
  );

  let best = null;
  for (const r of results) {
    if (r.status === 'fulfilled' && r.value) {
      if (!best || r.value.liquidity > best.liquidity) {
        best = r.value;
      }
    }
  }
  return best;
}
```

### 3.8.3 交易模块实现指引

#### 文件: `src/core/trading.ts`

```typescript
// 关键设计决策:
// 1. 无聚合合约 — 直接调用各协议的合约
// 2. 买入路径:
//    - Four 内盘 BNB → ITMV2.buyTokenAMAP
//    - Four 内盘 ERC20 → IHelper3.buyWithEth
//    - 外盘/PCS → PancakeRouter.swapExactETHForTokensSupportingFeeOnTransferTokens
//    - Flap 内盘 → IFlapPortal.swapExactInput
// 3. 卖出路径:
//    - Four 内盘 → ITMV2.sellToken / IHelper3.sellForEth
//    - 外盘/PCS → PancakeRouter.swapExactTokensForETHSupportingFeeOnTransferTokens
//    - Flap 内盘 → IFlapPortal.swapExactInput
// 4. Approve 目标:
//    - Four 内盘 → TokenManagerV2 (token 直接从用户转给 TM)
//    - 外盘/PCS → PancakeRouter
//    - Flap → 用户直接 approve 给 FlapPortal

import { parseUnits, formatUnits } from 'viem';
import { bscWriteContract } from './messaging';
import { ApproveCache } from './approve-cache';
import { getQuoteBuy, getQuoteSell } from './price-quoter';

const approveCache = new ApproveCache();
const DEADLINE_SECONDS = 10; // 短 deadline 快速失败

export async function buy(
  walletId: string,
  tokenAddr: Address,
  amountBNB: string,
  gasPrice: number,
  slippage: number,
  route: DetectedRoute,
  walletAddress: Address,
  publicClient: PublicClient
): Promise<TradeResult> {
  const amt = parseUnits(amountBNB, 18);
  if (amt <= 0n) throw new Error('数量太小');

  const slipBps = BigInt(Math.floor((100 - slippage) * 100));
  const estimated = await getQuoteBuy(publicClient, route, tokenAddr, amt);
  const amountOutMin = (estimated * slipBps) / 10000n;

  const gasPriceWei = parseUnits(gasPrice.toString(), 9);
  const deadline = BigInt(Math.floor(Date.now() / 1000) + DEADLINE_SECONDS);

  const t0 = performance.now();
  let txHash: string;

  switch (route.source) {
    case RouteSource.FOUR_INTERNAL_BNB:
      txHash = await buyFourInternalBNB(walletId, tokenAddr, amt, amountOutMin, gasPriceWei, route.fourInfo!);
      break;
    case RouteSource.FOUR_INTERNAL_ERC20:
      txHash = await buyFourInternalERC20(walletId, tokenAddr, amt, amountOutMin, gasPriceWei, route.fourInfo!);
      break;
    case RouteSource.FLAP_BONDING:
      txHash = await buyFlapBonding(walletId, tokenAddr, amt, amountOutMin, gasPriceWei);
      break;
    default:
      // FOUR_EXTERNAL, PANCAKE_ONLY, FLAP_DEX, FLAP_BONDING_SELL → PancakeSwap
      txHash = await buyPancakeSwap(walletId, tokenAddr, amt, amountOutMin, gasPriceWei, deadline, route.pancakeInfo?.quoteToken || WBNB);
      break;
  }

  const tSent = performance.now();
  const receipt = await publicClient.waitForTransactionReceipt({ hash: txHash as `0x${string}`, timeout: 120_000 });
  const tConfirm = performance.now();

  if (receipt.status !== 'success') throw new Error('交易失败: ' + txHash);

  // 买入后预授权卖出目标
  approveCache.ensureApproved(walletId, walletAddress, tokenAddr, route.approveTarget, gasPriceWei, MAX_HALF)
    .catch(e => console.warn('预授权失败:', e.message));

  return {
    txHash,
    sendMs: tSent - t0,
    confirmMs: tConfirm - tSent,
    totalMs: tConfirm - t0,
  };
}

async function buyPancakeSwap(
  walletId: string,
  token: Address,
  amount: bigint,
  minOut: bigint,
  gasPrice: bigint,
  deadline: bigint,
  quoteToken: Address
): Promise<string> {
  // 构建路径
  const path = quoteToken === WBNB
    ? [WBNB, token]
    : [WBNB, quoteToken, token];

  const res = await bscWriteContract(walletId, {
    address: PANCAKE_ROUTER,
    abi: pancakeRouterAbi,
    functionName: 'swapExactETHForTokensSupportingFeeOnTransferTokens',
    args: [minOut, path, '$WALLET_ADDRESS$', deadline], // $WALLET_ADDRESS$ 由 background 替换
    value: amount,
    gas: 500_000n,
    gasPrice,
  });
  return res.txHash;
}

async function buyFourInternalBNB(
  walletId: string,
  token: Address,
  amount: bigint,
  minOut: bigint,
  gasPrice: bigint,
  fourInfo: FourMemeInfo
): Promise<string> {
  const res = await bscWriteContract(walletId, {
    address: fourInfo.tokenManager,
    abi: tmV2Abi,
    functionName: 'buyTokenAMAP',
    args: [token, '$WALLET_ADDRESS$', amount, minOut],
    value: amount,
    gas: 500_000n,
    gasPrice,
  });
  return res.txHash;
}

// ... 类似实现其他买入/卖出函数
```

### 3.8.4 批量交易实现指引

#### 文件: `src/core/batch.ts`

```typescript
// 完全复用 Freedom Trader 的 Promise.allSettled 并行模式
// 但增加了类型安全和更好的错误处理

export async function executeBatchTrade(
  walletIds: string[],
  tokenAddr: Address,
  amount: string,
  mode: 'buy' | 'sell',
  gasPrice: number,
  slippage: number,
  route: DetectedRoute,
  walletAddresses: Map<string, Address>,
  publicClient: PublicClient,
  tokenDecimals: number,
  tokenBalances: Map<string, bigint>
): Promise<BatchResult> {
  if (walletIds.length === 0) throw new Error('请选择至少一个钱包');

  const t0 = performance.now();
  const promises = walletIds.map(id => {
    const addr = walletAddresses.get(id)!;
    if (mode === 'buy') {
      return buy(id, tokenAddr, amount, gasPrice, slippage, route, addr, publicClient);
    } else {
      return sell(id, tokenAddr, amount, gasPrice, slippage, route, addr, publicClient);
    }
  });

  const results = await Promise.allSettled(promises);
  const totalMs = performance.now() - t0;

  let success = 0, failed = 0;
  const items = results.map((r, i) => {
    if (r.status === 'fulfilled') {
      success++;
      return { walletId: walletIds[i], status: 'fulfilled' as const, value: r.value };
    } else {
      failed++;
      return { walletId: walletIds[i], status: 'rejected' as const, reason: r.reason?.message || '未知错误' };
    }
  });

  return { success, failed, results: items, totalMs };
}

export async function fastSell(
  walletIds: string[],
  tokenAddr: Address,
  percentage: number,
  tokenBalances: Map<string, bigint>,
  tokenDecimals: number,
  gasPrice: number,
  slippage: number,
  route: DetectedRoute,
  walletAddresses: Map<string, Address>,
  publicClient: PublicClient
): Promise<BatchResult> {
  const t0 = performance.now();

  const promises = walletIds.map(id => {
    const bal = tokenBalances.get(id) || 0n;
    if (bal <= 0n) return Promise.reject(new Error('余额为零'));
    const amt = (bal * BigInt(percentage)) / 100n;
    const amountStr = formatUnits(amt, tokenDecimals);
    const addr = walletAddresses.get(id)!;
    return sell(id, tokenAddr, amountStr, gasPrice, slippage, route, addr, publicClient);
  });

  const results = await Promise.allSettled(promises);
  // ... 同上统计
}
```

### 3.8.5 Svelte Store 实现指引

#### 文件: `src/stores/token.svelte.ts`

```typescript
// Svelte 5 runes 风格的响应式 store

import { detectRoute, type DetectedRoute } from '../core/route-detector';

class TokenStore {
  address = $state('');
  info = $state<TokenInfo>({ decimals: 18, symbol: '', balance: 0n, address: '' });
  route = $state<DetectedRoute | null>(null);
  balances = $state<Map<string, bigint>>(new Map());
  detecting = $state(false);
  price = $state<number | null>(null);

  get hasLP() { return this.route !== null && this.route.source !== RouteSource.NONE; }
  get isInternal() { return this.route?.isInternal ?? false; }
  get label() { return this.route?.label ?? ''; }

  async detect(addr: string, publicClient: PublicClient, walletEntries: WalletEntry[]) {
    if (!addr || !/^0x[a-fA-F0-9]{40}$/.test(addr)) {
      this.clear();
      return;
    }

    this.detecting = true;
    try {
      // 并行: 路由检测 + symbol + decimals + 各钱包余额
      const [route, symbol, decimals, ...bals] = await Promise.all([
        detectRoute(publicClient, addr as Address),
        publicClient.readContract({ address: addr, abi: erc20Abi, functionName: 'symbol' }).catch(() => '???'),
        publicClient.readContract({ address: addr, abi: erc20Abi, functionName: 'decimals' }).catch(() => 18),
        ...walletEntries.map(e =>
          publicClient.readContract({ address: addr, abi: erc20Abi, functionName: 'balanceOf', args: [e.address] }).catch(() => 0n)
        ),
      ]);

      this.route = route;
      let total = 0n;
      const balMap = new Map<string, bigint>();
      walletEntries.forEach((e, i) => {
        balMap.set(e.id, bals[i] as bigint);
        total += bals[i] as bigint;
      });
      this.balances = balMap;
      this.info = { decimals: Number(decimals), symbol: String(symbol), balance: total, address: addr };
    } catch (e) {
      console.error('检测失败:', e);
    } finally {
      this.detecting = false;
    }
  }

  clear() {
    this.info = { decimals: 18, symbol: '', balance: 0n, address: '' };
    this.route = null;
    this.balances = new Map();
    this.price = null;
  }
}

export const tokenStore = new TokenStore();
```

### 3.8.6 Manifest 配置指引

```json
{
  "manifest_version": 3,
  "name": "BSC Trader",
  "version": "1.0.0",
  "description": "BSC 快速交易工具",

  "permissions": [
    "storage",
    "sidePanel",
    "tabs",
    "activeTab"
  ],

  "host_permissions": [
    "https://*.bnbchain.org/*",
    "https://*.publicnode.com/*",
    "https://*.defibit.io/*",
    "https://*.ninicoin.io/*",
    "https://*.48.club/*",
    "https://*.blockrazor.xyz/*",
    "https://*.drpc.org/*",
    "https://*.quicknode.pro/*",
    "https://*.getblock.io/*"
  ],

  "background": {
    "service_worker": "src/background/index.ts",
    "type": "module"
  },

  "side_panel": {
    "default_path": "src/panel/panel.html"
  },

  "action": {
    "default_title": "BSC Trader"
  },

  "content_security_policy": {
    "extension_pages": "script-src 'self'; object-src 'self'"
  },

  "icons": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}
```

### 3.8.7 ABI 文件指引

```typescript
// src/lib/abi/pancake-router.ts
// 只包含用到的函数，减少包体积

export const pancakeRouterAbi = [
  {
    name: 'swapExactETHForTokensSupportingFeeOnTransferTokens',
    type: 'function',
    stateMutability: 'payable',
    inputs: [
      { name: 'amountOutMin', type: 'uint256' },
      { name: 'path', type: 'address[]' },
      { name: 'to', type: 'address' },
      { name: 'deadline', type: 'uint256' },
    ],
    outputs: [],
  },
  {
    name: 'swapExactETHForTokens',
    type: 'function',
    stateMutability: 'payable',
    inputs: [
      { name: 'amountOutMin', type: 'uint256' },
      { name: 'path', type: 'address[]' },
      { name: 'to', type: 'address' },
      { name: 'deadline', type: 'uint256' },
    ],
    outputs: [{ name: 'amounts', type: 'uint256[]' }],
  },
  {
    name: 'swapExactTokensForETHSupportingFeeOnTransferTokens',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'amountIn', type: 'uint256' },
      { name: 'amountOutMin', type: 'uint256' },
      { name: 'path', type: 'address[]' },
      { name: 'to', type: 'address' },
      { name: 'deadline', type: 'uint256' },
    ],
    outputs: [],
  },
  {
    name: 'swapExactTokensForETH',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'amountIn', type: 'uint256' },
      { name: 'amountOutMin', type: 'uint256' },
      { name: 'path', type: 'address[]' },
      { name: 'to', type: 'address' },
      { name: 'deadline', type: 'uint256' },
    ],
    outputs: [{ name: 'amounts', type: 'uint256[]' }],
  },
  {
    name: 'getAmountsOut',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'amountIn', type: 'uint256' },
      { name: 'path', type: 'address[]' },
    ],
    outputs: [{ name: 'amounts', type: 'uint256[]' }],
  },
] as const;
```

### 3.8.8 常量定义指引

```typescript
// src/lib/constants.ts

// === 合约地址 ===
export const PANCAKE_ROUTER = '0x10ED43C718714eb63d5aA57B78B54704E256024E' as const;
export const PANCAKE_FACTORY = '0xcA143Ce32Fe78f1f7019d7d551a6402fC5350c73' as const;
export const WBNB = '0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c' as const;
export const USDT = '0x55d398326f99059fF775485246999027B3197955' as const;
export const USD1 = '0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d' as const;
export const USDC = '0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d' as const;
export const BUSD = '0xe9e7CEA3DedcA5984780Bafc599bD69ADd087D56' as const;
export const FDUSD = '0xc5f0f7b66764F6ec8C8Dff7BA683102295E16409' as const;

export const ETH_SENTINEL = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE' as const;
export const ZERO_ADDR = '0x0000000000000000000000000000000000000000' as const;

// Four.meme
export const TOKEN_MANAGER_V2 = '0x5c952063c7fc8610FFDB798152D69F0B9550762b' as const;
export const FOUR_HELPER3 = '0xF251F83e40a78868FcfA3FA4599Dad6494E46034' as const;

// Flap
export const FLAP_PORTAL = '0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0' as const;

// === RPC 列表 ===
export const DEFAULT_RPC = 'https://bsc-dataseed.bnbchain.org';
export const BSC_RPCS = [
  'https://bsc-dataseed.bnbchain.org',
  'https://bsc-dataseed1.bnbchain.org',
  'https://bsc-dataseed2.bnbchain.org',
  'https://bsc-dataseed3.bnbchain.org',
  'https://bsc.publicnode.com',
  'https://bsc-dataseed1.defibit.io',
];

// === Quote Tokens ===
export const QUOTE_TOKENS = [
  { symbol: 'BNB', address: ETH_SENTINEL, decimals: 18 },
  { symbol: 'USDT', address: USDT, decimals: 18 },
  { symbol: 'USD1', address: USD1, decimals: 18 },
  { symbol: 'USDC', address: USDC, decimals: 18 },
] as const;

// === BigInt 常量 ===
export const MAX_UINT256 = BigInt('0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff');
export const MAX_HALF = BigInt('0x7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff');
```

### 3.8.9 关键注意事项

#### 给 Codex / Claude Code 团队的备注

1. **BigInt 序列化** — chrome.runtime.sendMessage 不能传 BigInt，必须序列化：
   ```typescript
   // 发送端
   function serializeArg(v: any): any {
     if (typeof v === 'bigint') return { __bigint: v.toString() };
     if (Array.isArray(v)) return v.map(serializeArg);
     return v;
   }
   // 接收端
   function deserializeArg(v: any): any {
     if (v && typeof v === 'object' && '__bigint' in v) return BigInt(v.__bigint);
     if (Array.isArray(v)) return v.map(deserializeArg);
     return v;
   }
   ```

2. **Service Worker 生命周期** — Chrome 会在 30 秒不活动后杀死 SW。密码必须存在 session storage 中，SW 重启时从 session storage 恢复。

3. **Wallet Address 传递** — 前端不持有私钥，但需要知道钱包地址才能查余额。`initWallets` 返回的 `{id: address}` 映射存在 store 中。交易时，`$WALLET_ADDRESS$` 这个占位符不能真的用——实际上 background 端已经知道每个 walletId 对应的地址，在 background 构建交易参数时直接填入。

4. **Approve 与交易不能同 nonce** — 如果需要先 approve 再 trade，必须等 approve 确认后再发 trade。不能用 `nonce` 和 `nonce+1` 同时发，因为 approve 可能失败。

5. **Tax Token 处理** — 直接调用 PancakeRouter 的 `SupportingFeeOnTransferTokens` 版本。如果这个版本 revert（不是 tax token），降级到普通版本。这个 try-catch 逻辑在 background 端不好做（writeContract 只发送一次），所以建议前端先检测是否是 tax token（读取 `feeRate()` 或看 Four.meme 的 `creatorType`），然后选择正确的函数。

6. **Deadline** — Freedom Trader 用 10 秒 deadline，这很激进但适合 meme 币交易。对于稳定币/主流币，可以放宽到 300 秒。

7. **错误处理** — 所有 bscWriteContract 调用可能返回 `{ error: '...' }`。前端 messaging.ts 的 `sendMessage` 应该将 error 转为 throw。

---

## 附录 A: 协议合约地址速查

| 协议 | 合约 | 地址 |
|------|------|------|
| PancakeSwap Router | IPancakeRouter | `0x10ED43C718714eb63d5aA57B78B54704E256024E` |
| PancakeSwap Factory | IPancakeFactory | `0xcA143Ce32Fe78f1f7019d7d551a6402fC5350c73` |
| WBNB | WBNB | `0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c` |
| Four.meme TokenManagerV2 | ITMV2 | `0x5c952063c7fc8610FFDB798152D69F0B9550762b` |
| Four.meme Helper3 | IHelper3 | `0xF251F83e40a78868FcfA3FA4599Dad6494E46034` |
| Flap Portal | IFlapPortal | `0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0` |

## 附录 B: 接口 ABI 来源

所有 ABI 从 FreedomRouter.sol 底部的 interface 定义中提取。关键函数签名：

```
IHelper3.getTokenInfo(address) → (uint256 ver, address tm, address quote, ...)
IHelper3.buyWithEth(uint256 origin, address token, address to, uint256 funds, uint256 minAmount)
IHelper3.sellForEth(uint256 origin, address token, address from, uint256 amount, uint256 minFunds, uint256 feeRate, address feeRecipient)
IHelper3.tryBuy(address token, uint256 amount, uint256 funds) → (..., uint256 estimated, ...)
IHelper3.trySell(address token, uint256 amount) → (..., uint256 funds, uint256 fee)

ITMV2.buyTokenAMAP(address token, address to, uint256 funds, uint256 minAmount) payable
ITMV2.sellToken(uint256 origin, address token, address from, uint256 amount, uint256 minFunds, uint256 feeRate, address feeRecipient)

IFlapPortal.getTokenV7(address token) → TokenStateV7
IFlapPortal.swapExactInput(ExactInputParams) payable → uint256
IFlapPortal.quoteExactInput(QuoteExactInputParams) → uint256

IPancakeRouter.swapExactETHForTokensSupportingFeeOnTransferTokens(uint256, address[], address, uint256) payable
IPancakeRouter.swapExactTokensForETHSupportingFeeOnTransferTokens(uint256, uint256, address[], address, uint256)
IPancakeRouter.getAmountsOut(uint256, address[]) view → uint256[]

IPancakeFactory.getPair(address, address) view → address
IPancakePair.getReserves() view → (uint112, uint112, uint32)

IFourToken._mode() view → uint256
IFourToken._tradingHalt() view → bool
IERC20.approve(address, uint256) → bool
IERC20.allowance(address, address) view → uint256
IERC20.balanceOf(address) view → uint256
IERC20.symbol() view → string
IERC20.decimals() view → uint8
```
