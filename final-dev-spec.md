# QuickTrader 开发规格文档

> 版本 1.0 | 2026-03-20 | 基于 FreedomRouter v6 + arch-review 决策

---

## 目录

1. [项目概述与架构图](#1-项目概述与架构图)
2. [QuickRouter 合约设计](#2-quickrouter-合约设计)
3. [服务器端交易引擎](#3-服务器端交易引擎)
4. [Chrome 扩展设计](#4-chrome-扩展设计)
5. [速度优化方案](#5-速度优化方案)
6. [安全设计](#6-安全设计)
7. [开发阶段规划](#7-开发阶段规划)
8. [文件清单](#8-文件清单)

---

## 1. 项目概述与架构图

### 1.1 项目定位

QuickTrader 是一个 BSC 链上交易系统，核心场景：用户在 GMGN.ai / DexScreener 浏览代币时，通过 Chrome 扩展捕获买卖信号，WSS 发送到服务器，服务器签名并通过防夹 RPC 广播交易。支持一个信号同时驱动 N 个钱包执行（跟单）。

### 1.2 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     Chrome 扩展 (Manifest V3)                    │
│                                                                 │
│  Content Script                        Background SW            │
│  ┌──────────────────────┐              ┌──────────────────┐     │
│  │ GMGN.ai 信号捕获      │──message───→│ WSS 客户端        │     │
│  │ DexScreener 信号捕获   │             │ - 连接管理        │     │
│  │ - 按钮点击拦截         │             │ - 重连策略        │     │
│  │ - 快捷键捕获           │             │ - 消息路由        │     │
│  │ - 合约地址提取         │             │ - 认证管理        │     │
│  └──────────────────────┘              └────────┬─────────┘     │
└──────────────────────────────────────────────────┼───────────────┘
                                                   │ WSS (TLS)
                                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                 交易服务器 (43.154.84.61:8083)                    │
│                                                                 │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────────────────┐ │
│  │ WSS Server │→│ 信号解析器    │→│ 交易执行引擎              │ │
│  │ - 认证     │  │ - token解析  │  │ ┌────────┐ ┌──────────┐ │ │
│  │ - 心跳     │  │ - 金额/方向  │  │ │路由决策 │→│TX 构建   │ │ │
│  │ - 限流     │  │ - 协议识别   │  │ └────────┘ └──────┬───┘ │ │
│  └────────────┘  └─────────────┘  │                   │      │ │
│                                    │ ┌────────────────┘      │ │
│  ┌─────────────┐  ┌────────────┐  │ │ ┌──────────────────┐  │ │
│  │ 钱包管理器   │  │ 路由缓存    │  │ │ │ 签名 + 多RPC广播  │  │ │
│  │ - 加密存储  │  │ - tokenInfo │  │ │ │ ┌──────┐┌──────┐ │  │ │
│  │ - 内存解密  │  │ - 定期刷新  │  │ │ │ │48club││BRazor│ │  │ │
│  └─────────────┘  └────────────┘  │ │ │ └──────┘└──────┘ │  │ │
│                                    │ │ └──────────────────┘  │ │
│  ┌─────────────┐  ┌────────────┐  │ │                        │ │
│  │ Nonce 管理器 │  │ 跟单引擎   │  │ └── 1信号 → N钱包并行    │ │
│  │ - 本地计数  │  │ - 信号分发  │  └──────────────────────────┘ │
│  │ - 并行nonce │  │ - 结果聚合  │                               │
│  └─────────────┘  └────────────┘                               │
└──────────────────────────────────────────────────┬──────────────┘
                                                   │
                                                   ▼ (多路并发 sendRawTransaction)
┌─────────────────────────────────────────────────────────────────┐
│                          BSC 链上                                │
│                                                                 │
│  QuickRouter 合约 (无 Proxy，直接部署)                            │
│  ├── trade(token, isBuy, amountIn, amountOutMin, routeHint)     │
│  ├── getTokenInfo(token, user) → TokenInfo                      │
│  └── quote(token, isBuy, amountIn) → uint256                   │
│                                                                 │
│  底层协议：                                                      │
│  ├── Four.meme (TokenManager + Helper3)                         │
│  ├── Flap (Portal)                                              │
│  └── PancakeSwap V2 (Router + Factory)                          │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 数据流图

#### 买入流程

```
用户点击 GMGN "Buy" 按钮
  → [Content Script] 捕获点击事件，提取 {token, chain}
  → [Content Script → Background] chrome.runtime.sendMessage
  → [Background] 组装 signal: {action:"buy", token, amount, chain:"bsc"}
  → [Background → Server] WSS send(JSON)
  → [Server/信号解析] 解析 token + amount + direction
  → [Server/路由决策] 查缓存 tokenInfo → 确定 routeSource + routeHint
  → [Server/跟单引擎] 对每个关联钱包分发任务
  → [Server/TX构建] 为每个钱包:
      encodeFunctionData("trade", [token, true, 0, amountOutMin, routeHint])
      设置 value = amountIn (BNB)
      设置精确 gasLimit (根据路由: 内盘80k/PCS 200k/Flap 150k)
  → [Server/签名] wallet.signTransaction (secp256k1, <1ms)
  → [Server/广播] 同时 sendRawTransaction 到 48.club + BlockRazor
  → [Server → Background] WSS send({type:"tx_sent", txHash, walletId})
  → [Server] 异步 waitForReceipt
  → [Server → Background] WSS send({type:"tx_confirmed", txHash, status})
  → [Background → Content Script] 更新 UI
```

#### 卖出流程

与买入类似，但需要先检查:
1. 代币余额是否足够
2. approve 是否已授权（缓存 + 链上查询）
3. 如需 approve，先发 approve tx 等确认，再发 sell tx

### 1.4 模块划分

| 模块 | 位置 | 职责 |
|------|------|------|
| QuickRouter 合约 | `contracts/` | 链上聚合路由 — 路由检测 + 执行 + 滑点保护 |
| 交易服务器 | `server/` | WSS 服务 + 交易引擎 + 钱包管理 + Nonce 管理 |
| Chrome 扩展 | `extension/` | 信号捕获 + WSS 通信 + 状态展示 |

---

## 2. QuickRouter 合约设计

### 2.1 设计原则

- **无 Proxy**：合约不持有用户资金，无需保持地址不变。升级时部署新合约，服务器更新地址即可
- **无 Tip**：手续费在服务器端处理，不在合约里扣
- **routeHint 优化**：服务器预判路由，跳过链上检测，节省 gas + 时间
- **Tax token 兼容**：`SupportingFeeOnTransferTokens` fallback
- **单一入口 trade()**：买卖统一，减少合约体积

### 2.2 完整接口定义

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title QuickRouter — BSC 聚合路由（无 Proxy 版本）
/// @notice 支持 Four.meme 内外盘 + Flap 内外盘 + PancakeSwap
/// @dev 目标 200-400 行，不含 tip 逻辑，不含 UUPS
contract QuickRouter is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ==================== 常量 ====================

    address public constant WBNB  = 0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c;
    address public constant USDT  = 0x55d398326f99059fF775485246999027B3197955;
    address public constant USD1  = 0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d;
    address public constant USDC  = 0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d;
    address public constant BUSD  = 0xe9e7CEA3DedcA5984780Bafc599bD69ADd087D56;
    address public constant FDUSD = 0xc5f0f7b66764F6ec8C8Dff7BA683102295E16409;

    address public constant PANCAKE_FACTORY = 0xcA143Ce32Fe78f1f7019d7d551a6402fC5350c73;
    address public constant PANCAKE_ROUTER  = 0x10ED43C718714eb63d5aA57B78B54704E256024E;

    // ==================== 可配置存储 ====================

    address public tokenManagerV2;  // Four.meme TokenManager V2
    address public tmHelper3;       // Four.meme Helper3 (买入/卖出/报价)
    address public flapPortal;      // Flap Portal (swapExactInput)

    // ==================== 枚举 ====================

    /// @notice 路由来源，数值与 FreedomRouter 保持一致便于迁移
    enum RouteSource {
        NONE,                   // 0 - 无可用路由
        FOUR_INTERNAL_BNB,      // 1 - Four.meme 内盘，BNB 报价
        FOUR_INTERNAL_ERC20,    // 2 - Four.meme 内盘，ERC20 报价 (如 USDT)
        FOUR_EXTERNAL,          // 3 - Four.meme 已毕业，走 PancakeSwap
        FLAP_BONDING,           // 4 - Flap bonding curve，双向可交易
        FLAP_BONDING_SELL,      // 5 - Flap bonding curve，仅卖（ERC20 quote + nativeSwap 关闭）
        FLAP_DEX,               // 6 - Flap 已毕业，走 PancakeSwap
        PANCAKE_ONLY            // 7 - 纯 PancakeSwap 代币
    }

    // ==================== 结构体 ====================

    /// @notice getTokenInfo 返回的统一查询结构
    struct TokenInfo {
        // 基础 ERC20
        string symbol;
        uint8 decimals;
        uint256 totalSupply;
        uint256 userBalance;
        // 路由
        RouteSource routeSource;
        address approveTarget;      // 卖出前 approve 给谁
        // Four.meme 状态
        bool isInternal;            // true = 内盘 (mode != 0 && !liquidityAdded)
        uint256 tmVersion;
        address tmAddress;
        address tmQuote;            // address(0) = BNB quote
        uint256 tmFunds;            // 内盘 BNB 储备
        uint256 tmMaxFunds;
        uint256 tmOffers;           // 内盘 token 储备
        uint256 tmMaxOffers;
        uint256 tmLastPrice;
        uint256 tmLaunchTime;
        uint256 tmTradingFeeRate;
        bool tmLiquidityAdded;
        bool tradingHalt;
        // TaxToken
        bool isTaxToken;
        uint256 taxFeeRate;
        // Flap 状态
        uint8 flapStatus;           // 0=无效, 1=Tradable, 4=DEX
        uint256 flapReserve;
        uint256 flapCirculatingSupply;
        uint256 flapPrice;
        address flapQuoteToken;
        bool flapNativeSwapEnabled;
        uint256 flapTaxRate;
        address flapPool;
        uint256 flapProgress;       // bonding curve 进度 (0-10000)
        // PancakeSwap
        address pair;
        address quoteToken;         // 最优交易对的 quote (WBNB/USDT/...)
        uint256 pairReserve0;
        uint256 pairReserve1;
        bool hasLiquidity;
    }

    // ==================== 事件 ====================

    event Trade(
        address indexed user,
        address indexed token,
        bool indexed isBuy,
        uint256 amountIn,
        uint256 amountOut,
        RouteSource route
    );
    event ConfigUpdated(string key, address oldVal, address newVal);
    event TokensRescued(address indexed token, uint256 amount);

    // ==================== 构造函数 ====================

    constructor(
        address _owner,
        address _tmV2,
        address _helper3,
        address _flapPortal
    ) Ownable(_owner) {
        tokenManagerV2 = _tmV2;
        tmHelper3 = _helper3;
        flapPortal = _flapPortal;
    }

    // ==================== 核心入口：trade() ====================

    /// @notice 统一交易入口
    /// @param token 目标代币地址
    /// @param isBuy true=买入(BNB→Token), false=卖出(Token→BNB)
    /// @param amountIn 买入时为 0 (使用 msg.value)；卖出时为 token 数量
    /// @param amountOutMin 最小输出量（滑点保护）
    /// @param routeHint 路由提示：NONE(0)=自动检测，其他值=跳过检测直接使用
    function trade(
        address token,
        bool isBuy,
        uint256 amountIn,
        uint256 amountOutMin,
        RouteSource routeHint
    ) external payable nonReentrant returns (uint256 amountOut) {
        // 1. 路由决策：routeHint != NONE 则跳过链上检测
        RouteSource route = routeHint != RouteSource.NONE
            ? routeHint
            : _detectRoute(token);
        require(route != RouteSource.NONE, "No route");

        // 2. 执行交易
        if (isBuy) {
            require(msg.value > 0, "No BNB");
            amountOut = _executeBuy(token, msg.value, route);
        } else {
            require(amountIn > 0, "No amount");
            amountOut = _executeSell(token, amountIn, route);
        }

        // 3. 滑点检查
        require(amountOut >= amountOutMin, "Slippage");

        emit Trade(msg.sender, token, isBuy, isBuy ? msg.value : amountIn, amountOut, route);
    }

    // ==================== 路由检测（借鉴 FreedomRouter._detectRoute）====================

    /// @notice 四级瀑布路由检测：Four.meme → Flap → PancakeSwap → NONE
    /// @dev 每级用 try-catch 防止级联 revert
    function _detectRoute(address token) internal view returns (RouteSource) {
        // ---- 1) Four.meme ----
        if (tmHelper3 != address(0)) {
            try IHelper3(tmHelper3).getTokenInfo(token) returns (
                uint256 ver, address _tm, address _quote,
                uint256, uint256, uint256, uint256,
                uint256, uint256, uint256, uint256,
                bool _liquidityAdded
            ) {
                if (ver > 0 && _tm != address(0)) {
                    if (_liquidityAdded) {
                        return RouteSource.FOUR_EXTERNAL;
                    }
                    // 检查 _mode 确认内盘状态
                    try IFourToken(token)._mode() returns (uint256 mode) {
                        if (mode != 0) {
                            return _quote == address(0)
                                ? RouteSource.FOUR_INTERNAL_BNB
                                : RouteSource.FOUR_INTERNAL_ERC20;
                        }
                    } catch {}
                }
            } catch {}
        }

        // ---- 2) Flap ----
        if (flapPortal != address(0)) {
            try IFlapPortal(flapPortal).getTokenV7(token) returns (
                IFlapPortal.TokenStateV7 memory st
            ) {
                if (st.status == 1) {
                    // Bonding curve — 检查 nativeSwap 是否开启
                    if (st.quoteTokenAddress == address(0) || st.nativeToQuoteSwapEnabled) {
                        return RouteSource.FLAP_BONDING;
                    }
                    // ERC20 quote + nativeSwap 关闭：只能通过 Portal 卖，买走 PCS
                    return RouteSource.FLAP_BONDING_SELL;
                } else if (st.status == 4) {
                    return RouteSource.FLAP_DEX;
                }
            } catch {}
        }

        // ---- 3) PancakeSwap 兜底 ----
        (address bestQuote,) = _findBestQuote(token);
        if (bestQuote != address(0)) {
            return RouteSource.PANCAKE_ONLY;
        }

        return RouteSource.NONE;
    }

    // ==================== 买入实现 ====================

    function _executeBuy(address token, uint256 value, RouteSource route) internal returns (uint256) {
        if (route == RouteSource.FOUR_INTERNAL_BNB || route == RouteSource.FOUR_INTERNAL_ERC20) {
            return _buyFourInternal(token, value);
        }
        if (route == RouteSource.FLAP_BONDING) {
            return _buyFlap(token, value);
        }
        // FOUR_EXTERNAL, PANCAKE_ONLY, FLAP_DEX, FLAP_BONDING_SELL → PancakeSwap
        return _buyPancake(token, value);
    }

    /// @dev Four.meme 内盘买入
    /// BNB quote: 直接调 TokenManager.buyTokenAMAP
    /// ERC20 quote: 通过 Helper3.buyWithEth (先 swap BNB→ERC20，再内盘买入)
    function _buyFourInternal(address token, uint256 value) internal returns (uint256) {
        (uint256 ver, address tm, address quote) = _getTokenTM(token);
        require(ver > 0 && tm != address(0), "No TM");

        uint256 balBefore = IERC20(token).balanceOf(msg.sender);

        if (quote == address(0)) {
            // BNB 报价 — 直接买，token 发给 msg.sender
            ITMV2(tm).buyTokenAMAP{value: value}(token, msg.sender, value, 0);
        } else {
            // ERC20 报价 — 通过 Helper3 先 BNB→ERC20 再买入
            uint256 quoteBal = IERC20(quote).balanceOf(address(this));
            uint256 ethBal = address(this).balance - value;
            IHelper3(tmHelper3).buyWithEth{value: value}(0, token, msg.sender, value, 0);
            // 退还多余的 quote token 和 ETH
            _refundExcess(quote, quoteBal, ethBal, msg.sender);
        }

        return IERC20(token).balanceOf(msg.sender) - balBefore;
    }

    /// @dev Flap bonding curve 买入
    /// Portal.swapExactInput: inputToken=address(0) 即 BNB 输入
    /// 注意：Flap 可能把 token 发给 router 而非 msg.sender，需要手动转发
    function _buyFlap(address token, uint256 value) internal returns (uint256) {
        require(flapPortal != address(0), "No Flap");

        uint256 userBefore = IERC20(token).balanceOf(msg.sender);
        uint256 routerBefore = IERC20(token).balanceOf(address(this));
        uint256 ethBefore = address(this).balance - value;

        IFlapPortal(flapPortal).swapExactInput{value: value}(
            IFlapPortal.ExactInputParams({
                inputToken: address(0),
                outputToken: token,
                inputAmount: value,
                minOutputAmount: 0,
                permitData: ""
            })
        );

        // Flap 可能把 token 发到 router 地址，需要转给用户
        uint256 routerGain = IERC20(token).balanceOf(address(this)) - routerBefore;
        if (routerGain > 0) {
            IERC20(token).safeTransfer(msg.sender, routerGain);
        }
        // 退还多余 BNB
        uint256 ethNow = address(this).balance;
        if (ethNow > ethBefore) {
            _sendBNB(msg.sender, ethNow - ethBefore);
        }

        return IERC20(token).balanceOf(msg.sender) - userBefore;
    }

    /// @dev PancakeSwap 买入 (适用于 Four 外盘 / Flap DEX / 纯 PCS 代币)
    /// 先 try SupportingFeeOnTransferTokens，失败 fallback 到标准 swap
    /// 自动选择最优交易对 (WBNB/USDT/USD1/USDC/BUSD/FDUSD)
    function _buyPancake(address token, uint256 value) internal returns (uint256) {
        (address bestQuote,) = _findBestQuote(token);

        address[] memory path;
        if (bestQuote == WBNB || bestQuote == address(0)) {
            path = new address[](2);
            path[0] = WBNB;
            path[1] = token;
        } else {
            // 非 WBNB 对：WBNB → bestQuote → token
            path = new address[](3);
            path[0] = WBNB;
            path[1] = bestQuote;
            path[2] = token;
        }

        uint256 balBefore = IERC20(token).balanceOf(msg.sender);
        uint256 deadline = block.timestamp;

        // Tax token 兼容：先 try fee-on-transfer 版本
        try IPancakeRouter(PANCAKE_ROUTER)
            .swapExactETHForTokensSupportingFeeOnTransferTokens{value: value}(
                0, path, msg.sender, deadline
            )
        {} catch {
            IPancakeRouter(PANCAKE_ROUTER)
                .swapExactETHForTokens{value: value}(0, path, msg.sender, deadline);
        }

        return IERC20(token).balanceOf(msg.sender) - balBefore;
    }

    // ==================== 卖出实现 ====================

    function _executeSell(address token, uint256 amountIn, RouteSource route) internal returns (uint256) {
        if (route == RouteSource.FOUR_INTERNAL_BNB || route == RouteSource.FOUR_INTERNAL_ERC20) {
            return _sellFourInternal(token, amountIn);
        }
        if (route == RouteSource.FLAP_BONDING || route == RouteSource.FLAP_BONDING_SELL) {
            return _sellFlap(token, amountIn);
        }
        // FOUR_EXTERNAL, PANCAKE_ONLY, FLAP_DEX → PancakeSwap
        return _sellPancake(token, amountIn);
    }

    /// @dev Four.meme 内盘卖出
    /// BNB quote: TokenManager.sellToken 直接把 BNB 发给用户
    /// ERC20 quote: Helper3.sellForEth 先卖得 ERC20 再 swap 为 BNB
    /// 注意：用户需提前 approve token 给 TokenManager (内盘 approve target)
    function _sellFourInternal(address token, uint256 amountIn) internal returns (uint256) {
        (uint256 ver, address tm, address quote) = _getTokenTM(token);
        require(ver > 0 && tm != address(0), "No TM");

        uint256 bnbBefore = msg.sender.balance;

        if (quote == address(0)) {
            // tipRate=0, feeRecipient=address(0) — 服务器端自行处理手续费
            ITMV2(tm).sellToken(0, token, msg.sender, amountIn, 0, 0, address(0));
        } else {
            uint256 quoteBal = IERC20(quote).balanceOf(address(this));
            uint256 ethBal = address(this).balance;
            IHelper3(tmHelper3).sellForEth(0, token, msg.sender, amountIn, 0, 0, address(0));
            _refundExcess(quote, quoteBal, ethBal, msg.sender);
        }

        uint256 bnbAfter = msg.sender.balance;
        return bnbAfter > bnbBefore ? bnbAfter - bnbBefore : 0;
    }

    /// @dev Flap Portal 卖出
    /// 先 transferFrom token 到 router → approve 给 Portal → swapExactInput
    /// 使用 balanceOf 差值计算实际到账（兼容 tax token）
    function _sellFlap(address token, uint256 amountIn) internal returns (uint256) {
        require(flapPortal != address(0), "No Flap");

        // 先把 token 转到 router（兼容 tax token 的实际到账差值）
        uint256 balBefore = IERC20(token).balanceOf(address(this));
        IERC20(token).safeTransferFrom(msg.sender, address(this), amountIn);
        uint256 actualIn = IERC20(token).balanceOf(address(this)) - balBefore;

        IERC20(token).forceApprove(flapPortal, actualIn);

        uint256 bnbBefore = address(this).balance;

        IFlapPortal(flapPortal).swapExactInput(
            IFlapPortal.ExactInputParams({
                inputToken: token,
                outputToken: address(0), // 输出 BNB
                inputAmount: actualIn,
                minOutputAmount: 0,
                permitData: ""
            })
        );

        uint256 bnbOut = address(this).balance - bnbBefore;
        if (bnbOut > 0) _sendBNB(msg.sender, bnbOut);
        return bnbOut;
    }

    /// @dev PancakeSwap 卖出 (Four 外盘 / Flap DEX / 纯 PCS)
    /// 先 transferFrom → approve → swap，用 balanceOf 差值兼容 tax token
    function _sellPancake(address token, uint256 amountIn) internal returns (uint256) {
        // 转入 + 计算实际到账
        uint256 balBefore = IERC20(token).balanceOf(address(this));
        IERC20(token).safeTransferFrom(msg.sender, address(this), amountIn);
        uint256 actualIn = IERC20(token).balanceOf(address(this)) - balBefore;

        IERC20(token).forceApprove(PANCAKE_ROUTER, actualIn);

        (address bestQuote,) = _findBestQuote(token);
        address[] memory path;
        if (bestQuote == WBNB || bestQuote == address(0)) {
            path = new address[](2);
            path[0] = token;
            path[1] = WBNB;
        } else {
            path = new address[](3);
            path[0] = token;
            path[1] = bestQuote;
            path[2] = WBNB;
        }

        uint256 bnbBefore = address(this).balance;
        uint256 deadline = block.timestamp;

        // Tax token 兼容
        try IPancakeRouter(PANCAKE_ROUTER)
            .swapExactTokensForETHSupportingFeeOnTransferTokens(
                actualIn, 0, path, address(this), deadline
            )
        {} catch {
            IPancakeRouter(PANCAKE_ROUTER)
                .swapExactTokensForETH(actualIn, 0, path, address(this), deadline);
        }

        uint256 bnbOut = address(this).balance - bnbBefore;
        if (bnbOut > 0) _sendBNB(msg.sender, bnbOut);
        return bnbOut;
    }

    // ==================== 查询接口 ====================

    /// @notice 统一报价（仅 eth_call，链上不可调用）
    function quote(address token, bool isBuy, uint256 amountIn) external returns (uint256) {
        require(msg.sender == address(0) || tx.origin == address(0), "eth_call only");

        RouteSource route = _detectRoute(token);
        require(route != RouteSource.NONE, "No route");

        if (isBuy) {
            return _quoteBuy(token, amountIn, route);
        } else {
            return _quoteSell(token, amountIn, route);
        }
    }

    function _quoteBuy(address token, uint256 amountIn, RouteSource route) internal returns (uint256) {
        if (route == RouteSource.FOUR_INTERNAL_BNB || route == RouteSource.FOUR_INTERNAL_ERC20) {
            (,, uint256 estimated,,,,,) = IHelper3(tmHelper3).tryBuy(token, 0, amountIn);
            return estimated;
        }
        if (route == RouteSource.FLAP_BONDING) {
            return IFlapPortal(flapPortal).quoteExactInput(
                IFlapPortal.QuoteExactInputParams({
                    inputToken: address(0),
                    outputToken: token,
                    inputAmount: amountIn
                })
            );
        }
        // PCS 报价
        return _quotePancake(token, amountIn, true);
    }

    function _quoteSell(address token, uint256 amountIn, RouteSource route) internal returns (uint256) {
        if (route == RouteSource.FOUR_INTERNAL_BNB || route == RouteSource.FOUR_INTERNAL_ERC20) {
            (,, uint256 funds, uint256 fee) = IHelper3(tmHelper3).trySell(token, amountIn);
            uint256 net = funds > fee ? funds - fee : 0;
            // ERC20 quote 需要额外查 PCS 价格转换为 BNB
            (,, address _quote) = _getTokenTM(token);
            if (net > 0 && _quote != address(0)) {
                address[] memory p = new address[](2);
                p[0] = _quote;
                p[1] = WBNB;
                try IPancakeRouter(PANCAKE_ROUTER).getAmountsOut(net, p) returns (uint256[] memory a) {
                    return a[1];
                } catch { return 0; }
            }
            return net;
        }
        if (route == RouteSource.FLAP_BONDING || route == RouteSource.FLAP_BONDING_SELL) {
            return IFlapPortal(flapPortal).quoteExactInput(
                IFlapPortal.QuoteExactInputParams({
                    inputToken: token,
                    outputToken: address(0),
                    inputAmount: amountIn
                })
            );
        }
        return _quotePancake(token, amountIn, false);
    }

    function _quotePancake(address token, uint256 amountIn, bool isBuy) internal view returns (uint256) {
        (address bestQuote,) = _findBestQuote(token);
        address[] memory path;
        if (isBuy) {
            if (bestQuote == WBNB || bestQuote == address(0)) {
                path = new address[](2);
                path[0] = WBNB; path[1] = token;
            } else {
                path = new address[](3);
                path[0] = WBNB; path[1] = bestQuote; path[2] = token;
            }
        } else {
            if (bestQuote == WBNB || bestQuote == address(0)) {
                path = new address[](2);
                path[0] = token; path[1] = WBNB;
            } else {
                path = new address[](3);
                path[0] = token; path[1] = bestQuote; path[2] = WBNB;
            }
        }
        uint256[] memory amounts = IPancakeRouter(PANCAKE_ROUTER).getAmountsOut(amountIn, path);
        return amounts[amounts.length - 1];
    }

    /// @notice 统一 token 信息查询（一次 eth_call 拿到所有状态）
    function getTokenInfo(address token, address user) external view returns (TokenInfo memory info) {
        // ---- 基础 ERC20 ----
        try IERC20Metadata(token).symbol() returns (string memory s) { info.symbol = s; } catch {}
        try IERC20Metadata(token).decimals() returns (uint8 d) { info.decimals = d; } catch { info.decimals = 18; }
        try IERC20(token).totalSupply() returns (uint256 ts) { info.totalSupply = ts; } catch {}
        if (user != address(0)) {
            try IERC20(token).balanceOf(user) returns (uint256 b) { info.userBalance = b; } catch {}
        }

        // ---- Four.meme ----
        try IFourToken(token)._tradingHalt() returns (bool h) { info.tradingHalt = h; } catch {}
        if (tmHelper3 != address(0)) {
            try IHelper3(tmHelper3).getTokenInfo(token) returns (
                uint256 ver, address _tm, address _quote,
                uint256 lastPrice, uint256 tradingFeeRate, uint256,
                uint256 launchTime, uint256 offers, uint256 maxOffers,
                uint256 funds, uint256 maxFunds, bool liqAdded
            ) {
                info.tmVersion = ver;
                info.tmAddress = _tm;
                info.tmQuote = _quote;
                info.tmLastPrice = lastPrice;
                info.tmTradingFeeRate = tradingFeeRate;
                info.tmLaunchTime = launchTime;
                info.tmOffers = offers;
                info.tmMaxOffers = maxOffers;
                info.tmFunds = funds;
                info.tmMaxFunds = maxFunds;
                info.tmLiquidityAdded = liqAdded;
                // 内盘判定：mode != 0 且未添加流动性
                try IFourToken(token)._mode() returns (uint256 mode) {
                    info.isInternal = mode != 0 && ver > 0 && !liqAdded;
                } catch {}
            } catch {}
        }

        // ---- TaxToken (Four v2) ----
        if (info.tmVersion == 2 && tokenManagerV2 != address(0)) {
            try ITMQuery(tokenManagerV2)._tokenInfos(token) returns (ITMQuery.TMInfo memory tmInfo) {
                uint256 creatorType = (tmInfo.template >> 10) & 0x3F;
                info.isTaxToken = (creatorType == 5);
            } catch {}
            if (info.isTaxToken) {
                try ITaxToken(token).feeRate() returns (uint256 fr) { info.taxFeeRate = fr; } catch {}
            }
        }

        // ---- Flap ----
        if (flapPortal != address(0) && info.tmVersion == 0) {
            try IFlapPortal(flapPortal).getTokenV7(token) returns (IFlapPortal.TokenStateV7 memory st) {
                info.flapStatus = uint8(st.status);
                info.flapReserve = st.reserve;
                info.flapCirculatingSupply = st.circulatingSupply;
                info.flapPrice = st.price;
                info.flapQuoteToken = st.quoteTokenAddress;
                info.flapNativeSwapEnabled = st.nativeToQuoteSwapEnabled;
                info.flapTaxRate = st.taxRate;
                info.flapPool = st.pool;
                info.flapProgress = st.progress;
            } catch {}
        }

        // ---- PancakeSwap ----
        (address bestQuote, address bestPair) = _findBestQuote(token);
        info.quoteToken = bestQuote;
        info.pair = bestPair;
        if (bestPair != address(0)) {
            try IPancakePair(bestPair).getReserves() returns (uint112 r0, uint112 r1, uint32) {
                info.pairReserve0 = r0;
                info.pairReserve1 = r1;
                info.hasLiquidity = (r0 > 0 && r1 > 0);
            } catch {}
        }

        // ---- 路由 + approveTarget ----
        info.routeSource = _detectRoute(token);
        info.approveTarget = _getApproveTarget(info.routeSource);
    }

    // ==================== 内部工具函数 ====================

    /// @dev 最优 PancakeSwap 交易对选择
    /// 遍历 WBNB/USDT/USD1/USDC/BUSD/FDUSD，按 reserve0*reserve1 选流动性最深的
    function _findBestQuote(address token) internal view returns (address bestQuote, address bestPair) {
        address[6] memory quotes = [WBNB, USDT, USD1, USDC, BUSD, FDUSD];
        uint256 bestLiquidity;
        for (uint256 i = 0; i < 6; i++) {
            try IPancakeFactory(PANCAKE_FACTORY).getPair(token, quotes[i]) returns (address p) {
                if (p != address(0)) {
                    try IPancakePair(p).getReserves() returns (uint112 r0, uint112 r1, uint32) {
                        uint256 liq = uint256(r0) * uint256(r1);
                        if (liq > bestLiquidity) {
                            bestLiquidity = liq;
                            bestQuote = quotes[i];
                            bestPair = p;
                        }
                    } catch {}
                }
            } catch {}
        }
    }

    function _getTokenTM(address token) internal view returns (uint256 ver, address tm, address quote) {
        if (tmHelper3 == address(0)) return (0, address(0), address(0));
        try IHelper3(tmHelper3).getTokenInfo(token) returns (
            uint256 v, address _tm, address _quote,
            uint256, uint256, uint256, uint256,
            uint256, uint256, uint256, uint256, bool
        ) {
            return (v, _tm, _quote);
        } catch {
            return (0, address(0), address(0));
        }
    }

    /// @dev 根据路由决定卖出时 approve 给谁
    /// 内盘：approve 给 TokenManager（Four.meme 直接从 user 扣 token）
    /// 其他：approve 给 QuickRouter 本身（router 先 transferFrom 再操作）
    function _getApproveTarget(RouteSource route) internal view returns (address) {
        if (route == RouteSource.FOUR_INTERNAL_BNB || route == RouteSource.FOUR_INTERNAL_ERC20) {
            return tokenManagerV2;
        }
        return address(this);
    }

    function _refundExcess(address quoteToken, uint256 quoteBefore, uint256 ethBefore, address to) internal {
        uint256 quoteNow = IERC20(quoteToken).balanceOf(address(this));
        if (quoteNow > quoteBefore) {
            IERC20(quoteToken).safeTransfer(to, quoteNow - quoteBefore);
        }
        uint256 ethNow = address(this).balance;
        if (ethNow > ethBefore) {
            _sendBNB(to, ethNow - ethBefore);
        }
    }

    function _sendBNB(address to, uint256 amount) internal {
        (bool ok, ) = payable(to).call{value: amount}("");
        require(ok, "BNB transfer failed");
    }

    // ==================== 管理函数 ====================

    function setTokenManagerV2(address a) external onlyOwner {
        emit ConfigUpdated("tmV2", tokenManagerV2, a);
        tokenManagerV2 = a;
    }

    function setHelper3(address a) external onlyOwner {
        emit ConfigUpdated("helper3", tmHelper3, a);
        tmHelper3 = a;
    }

    function setFlapPortal(address a) external onlyOwner {
        emit ConfigUpdated("flapPortal", flapPortal, a);
        flapPortal = a;
    }

    function rescueTokens(address token, uint256 amount) external onlyOwner {
        if (token == address(0)) {
            uint256 bal = address(this).balance;
            _sendBNB(owner(), amount > bal ? bal : amount);
        } else {
            IERC20(token).safeTransfer(owner(), amount);
        }
        emit TokensRescued(token, amount);
    }

    receive() external payable {}
}

// ==================== 外部接口定义 ====================
// (与 FreedomRouter 中接口完全一致，直接复用)

interface IERC20Metadata {
    function symbol() external view returns (string memory);
    function decimals() external view returns (uint8);
}

interface IFourToken {
    function _mode() external view returns (uint256);
    function _tradingHalt() external view returns (bool);
}

interface ITaxToken {
    function feeRate() external view returns (uint256);
}

interface ITMV2 {
    function buyTokenAMAP(address token, address to, uint256 funds, uint256 minAmount) external payable;
    function sellToken(uint256 origin, address token, address from, uint256 amount,
                       uint256 minFunds, uint256 feeRate, address feeRecipient) external;
}

interface IHelper3 {
    function getTokenInfo(address token) external view returns (
        uint256 version, address tokenManager, address quote,
        uint256 lastPrice, uint256 tradingFeeRate, uint256 minTradingFee,
        uint256 launchTime, uint256 offers, uint256 maxOffers,
        uint256 funds, uint256 maxFunds, bool liquidityAdded
    );
    function buyWithEth(uint256 origin, address token, address to, uint256 funds, uint256 minAmount) external payable;
    function sellForEth(uint256 origin, address token, address from, uint256 amount,
                        uint256 minFunds, uint256 feeRate, address feeRecipient) external;
    function tryBuy(address token, uint256 amount, uint256 funds) external view returns (
        address tokenManager, address quote, uint256 estimatedAmount, uint256 estimatedCost,
        uint256 estimatedFee, uint256 amountMsgValue, uint256 amountApproval, uint256 amountFunds
    );
    function trySell(address token, uint256 amount) external view returns (
        address tokenManager, address quote, uint256 funds, uint256 fee
    );
}

interface ITMQuery {
    struct TMInfo {
        address base; address quote; uint256 template; uint256 totalSupply;
        uint256 maxOffers; uint256 maxRaising; uint256 launchTime;
        uint256 offers; uint256 funds; uint256 lastPrice; uint256 K; uint256 T; uint256 status;
    }
    function _tokenInfos(address token) external view returns (TMInfo memory);
}

interface IFlapPortal {
    struct TokenStateV7 {
        uint8 status; uint256 reserve; uint256 circulatingSupply; uint256 price;
        uint8 tokenVersion; uint256 r; uint256 h; uint256 k; uint256 dexSupplyThresh;
        address quoteTokenAddress; bool nativeToQuoteSwapEnabled; bytes32 extensionID;
        uint256 taxRate; address pool; uint256 progress; uint8 lpFeeProfile; uint8 dexId;
    }
    struct QuoteExactInputParams { address inputToken; address outputToken; uint256 inputAmount; }
    struct ExactInputParams {
        address inputToken; address outputToken; uint256 inputAmount;
        uint256 minOutputAmount; bytes permitData;
    }
    function getTokenV7(address token) external view returns (TokenStateV7 memory);
    function quoteExactInput(QuoteExactInputParams calldata params) external returns (uint256);
    function swapExactInput(ExactInputParams calldata params) external payable returns (uint256);
}

interface IPancakeFactory { function getPair(address, address) external view returns (address); }
interface IPancakePair { function getReserves() external view returns (uint112, uint112, uint32); }

interface IPancakeRouter {
    function swapExactETHForTokens(uint256, address[] calldata, address, uint256) external payable returns (uint256[] memory);
    function swapExactTokensForETH(uint256, uint256, address[] calldata, address, uint256) external returns (uint256[] memory);
    function swapExactETHForTokensSupportingFeeOnTransferTokens(uint256, address[] calldata, address, uint256) external payable;
    function swapExactTokensForETHSupportingFeeOnTransferTokens(uint256, uint256, address[] calldata, address, uint256) external;
    function getAmountsOut(uint256, address[] calldata) external view returns (uint256[] memory);
}
```

### 2.3 关键设计说明

#### routeHint 参数

这是相对 FreedomRouter 最重要的优化。服务器端通过 `getTokenInfo()` 已经知道了 token 的路由，交易时传入 `routeHint` 跳过链上路由检测：

```
无 routeHint（自动检测）:
  _detectRoute → 最多 3 次外部 call → 额外 15,000-30,000 gas + 时间

有 routeHint:
  直接跳到对应执行分支 → 省下所有检测 gas
```

**服务器端使用方式：**
```typescript
// 首次查询 token（或缓存过期时）
const info = await publicClient.readContract({
  address: QUICK_ROUTER, abi: ROUTER_ABI,
  functionName: 'getTokenInfo', args: [tokenAddr, walletAddr]
});
// 缓存 routeSource
cache.set(tokenAddr, { route: info.routeSource, approveTarget: info.approveTarget });

// 交易时直接传 routeHint
const { request } = await publicClient.simulateContract({
  address: QUICK_ROUTER, abi: ROUTER_ABI,
  functionName: 'trade',
  args: [tokenAddr, true, 0n, amountOutMin, cache.get(tokenAddr).route],
  value: amountIn
});
```

#### Four.meme 内外盘区分

```
内盘判定条件 (同 FreedomRouter):
  mode != 0 AND tmVersion > 0 AND !liquidityAdded

BNB 报价 vs ERC20 报价:
  tmQuote == address(0) → BNB 报价 → 直接调 TokenManager.buyTokenAMAP
  tmQuote != address(0) → ERC20 报价 → 通过 Helper3.buyWithEth (自动 BNB→ERC20→买入)

外盘 (liquidityAdded == true):
  统一走 PancakeSwap，自动选最优交易对
```

#### Flap 特殊情况

```
status=1 (Bonding curve):
  nativeSwapEnabled=true 或 quoteToken=address(0):
    → FLAP_BONDING: 买卖都通过 Portal
  nativeSwapEnabled=false 且 quoteToken!=address(0):
    → FLAP_BONDING_SELL: 卖通过 Portal，买走 PancakeSwap
    （因为 Portal 不支持 BNB→ERC20 quote 的买入方向）

status=4 (已毕业):
  → FLAP_DEX: 全部走 PancakeSwap
```

#### Gas 优化技巧

1. **`block.timestamp` 代替 deadline 参数**：省去 calldata 成本 + 不需要 require 检查
2. **amountOutMin=0 传入协议**：滑点检查在 router 层统一做，不在底层重复检查
3. **forceApprove 代替 approve**：SafeERC20 的 forceApprove 先清零再设值，兼容 USDT 等非标 token
4. **最优交易对选择**：避免 WBNB→USDT→Token 三跳，直接选流动性最深的对

#### 安全考量

- **ReentrancyGuard**：保留。Flap 和 Four.meme 的回调可能触发重入
- **onlyOwner 管理函数**：仅 owner 可更新协议地址和紧急提取
- **eth_call only 报价**：`require(msg.sender == address(0) || tx.origin == address(0))` 防止链上滥用
- **无 approve 常驻**：router 只在 swap 时临时 approve，不持有长期授权

### 2.4 部署流程

```bash
# Foundry 项目初始化
forge init quick-router --no-git
cd quick-router
forge install OpenZeppelin/openzeppelin-contracts

# 编译
forge build

# 部署脚本 script/Deploy.s.sol
forge script script/Deploy.s.sol:DeployQuickRouter \
  --rpc-url https://bsc-dataseed.bnbchain.org \
  --broadcast \
  --verify \
  --etherscan-api-key $BSCSCAN_KEY
```

**部署脚本：**

```solidity
// script/Deploy.s.sol
pragma solidity ^0.8.28;

import "forge-std/Script.sol";
import "../src/QuickRouter.sol";

contract DeployQuickRouter is Script {
    // BSC 主网地址
    address constant TM_V2    = 0x5c952063c7fc8610FFDB798152D69F0B9550762b;
    address constant HELPER3  = 0x267E15f8c94C1c53bD520C200352edD8046F3b43;
    address constant FLAP     = 0xFf62a00C818053bfc30d498c8fC345b4d6d3B0C9;

    function run() external {
        uint256 deployerKey = vm.envUint("DEPLOYER_KEY");
        address owner = vm.addr(deployerKey);

        vm.startBroadcast(deployerKey);
        QuickRouter router = new QuickRouter(owner, TM_V2, HELPER3, FLAP);
        vm.stopBroadcast();

        console.log("QuickRouter deployed at:", address(router));
    }
}
```

**Foundry 配置：**

```toml
# foundry.toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc = "0.8.28"
optimizer = true
optimizer_runs = 10000  # 高 runs 因为会被频繁调用
evm_version = "cancun"

[profile.default.fmt]
line_length = 120

[etherscan]
bsc = { key = "${BSCSCAN_KEY}", url = "https://api.bscscan.com/api" }

[rpc_endpoints]
bsc = "https://bsc-dataseed.bnbchain.org"
```

---

## 3. 服务器端交易引擎

### 3.1 项目结构

```
server/
├── package.json
├── tsconfig.json
├── ecosystem.config.cjs          # PM2 配置
├── .env.example
├── src/
│   ├── index.ts                  # 入口：启动 WSS 服务器
│   ├── config.ts                 # 环境变量 + 常量
│   ├── wss/
│   │   ├── server.ts             # WSS 服务器（认证、心跳、消息分发）
│   │   ├── auth.ts               # Token 认证逻辑
│   │   └── protocol.ts           # 消息类型定义 + 校验
│   ├── wallet/
│   │   ├── manager.ts            # 钱包管理器（加密存储、内存解密）
│   │   ├── crypto.ts             # AES-256-GCM 加密/解密
│   │   └── types.ts              # 钱包类型定义
│   ├── trading/
│   │   ├── engine.ts             # 交易执行引擎（核心）
│   │   ├── signal-parser.ts      # 信号解析器
│   │   ├── route-cache.ts        # 路由缓存
│   │   ├── tx-builder.ts         # 交易构建器
│   │   ├── broadcaster.ts        # 多 RPC 广播器
│   │   └── copy-trade.ts         # 跟单引擎
│   ├── nonce/
│   │   ├── manager.ts            # Nonce 管理器
│   │   └── recovery.ts           # Nonce 卡死修复
│   ├── rpc/
│   │   ├── pool.ts               # RPC 连接池
│   │   └── endpoints.ts          # RPC 节点列表
│   ├── storage/
│   │   ├── db.ts                 # SQLite 封装
│   │   ├── tx-log.ts             # 交易记录
│   │   └── migrations.ts         # 数据库迁移
│   ├── abi/
│   │   └── quick-router.ts       # QuickRouter ABI
│   └── utils/
│       ├── logger.ts             # 日志（不泄露敏感信息）
│       └── retry.ts              # 重试策略
├── data/                         # 运行时数据目录
│   ├── wallets.enc               # 加密的钱包文件
│   └── quicktrader.db            # SQLite 数据库
└── test/
    ├── engine.test.ts
    └── nonce.test.ts
```

### 3.2 WSS 服务器设计

```typescript
// src/wss/server.ts
import { WebSocketServer, WebSocket } from 'ws';
import https from 'https';
import fs from 'fs';
import { verifyToken } from './auth';
import { MessageType, parseMessage } from './protocol';
import { config } from '../config';

interface ClientSession {
  ws: WebSocket;
  userId: string;
  authenticated: boolean;
  lastPing: number;
}

export class WssServer {
  private wss: WebSocketServer;
  private sessions = new Map<string, ClientSession>();
  private heartbeatTimer: NodeJS.Timeout;

  constructor(private onSignal: (userId: string, msg: any) => void) {
    // TLS — 使用 Let's Encrypt 证书
    const server = https.createServer({
      cert: fs.readFileSync(config.TLS_CERT),
      key: fs.readFileSync(config.TLS_KEY),
    });

    this.wss = new WebSocketServer({ server });
    server.listen(config.WSS_PORT, () => {
      console.log(`WSS listening on :${config.WSS_PORT}`);
    });

    this.wss.on('connection', (ws, req) => this.handleConnection(ws, req));
    this.heartbeatTimer = setInterval(() => this.checkHeartbeats(), 30_000);
  }

  private handleConnection(ws: WebSocket, req: any) {
    const sessionId = crypto.randomUUID();
    const session: ClientSession = {
      ws, userId: '', authenticated: false, lastPing: Date.now()
    };
    this.sessions.set(sessionId, session);

    ws.on('message', (data) => {
      try {
        const msg = parseMessage(data.toString());
        session.lastPing = Date.now();

        if (msg.type === 'auth') {
          this.handleAuth(sessionId, session, msg);
        } else if (msg.type === 'ping') {
          ws.send(JSON.stringify({ type: 'pong', ts: Date.now() }));
        } else if (session.authenticated) {
          this.onSignal(session.userId, msg);
        } else {
          ws.send(JSON.stringify({ type: 'error', message: '未认证' }));
        }
      } catch (e) {
        ws.send(JSON.stringify({ type: 'error', message: '消息格式错误' }));
      }
    });

    ws.on('close', () => this.sessions.delete(sessionId));
    ws.on('error', () => this.sessions.delete(sessionId));

    // 10 秒内不认证则断开
    setTimeout(() => {
      if (!session.authenticated) {
        ws.close(4001, '认证超时');
        this.sessions.delete(sessionId);
      }
    }, 10_000);
  }

  private handleAuth(sessionId: string, session: ClientSession, msg: any) {
    const userId = verifyToken(msg.token);
    if (userId) {
      session.userId = userId;
      session.authenticated = true;
      session.ws.send(JSON.stringify({ type: 'auth_ok', userId }));
    } else {
      session.ws.send(JSON.stringify({ type: 'auth_fail' }));
      session.ws.close(4003, '认证失败');
      this.sessions.delete(sessionId);
    }
  }

  private checkHeartbeats() {
    const now = Date.now();
    for (const [id, session] of this.sessions) {
      if (now - session.lastPing > 60_000) {
        session.ws.close(4000, '心跳超时');
        this.sessions.delete(id);
      }
    }
  }

  /** 向指定用户发送消息 */
  send(userId: string, msg: object) {
    for (const session of this.sessions.values()) {
      if (session.userId === userId && session.authenticated) {
        session.ws.send(JSON.stringify(msg));
      }
    }
  }

  /** 向所有已认证客户端广播 */
  broadcast(msg: object) {
    const data = JSON.stringify(msg);
    for (const session of this.sessions.values()) {
      if (session.authenticated) session.ws.send(data);
    }
  }
}
```

#### 认证方案

```typescript
// src/wss/auth.ts
import crypto from 'crypto';
import { config } from '../config';

/**
 * 简单 HMAC token 认证
 * Token 格式: userId:timestamp:hmac
 * HMAC = SHA256(userId + ":" + timestamp, SECRET)
 * timestamp 5 分钟内有效
 */
export function generateToken(userId: string): string {
  const ts = Math.floor(Date.now() / 1000).toString();
  const hmac = crypto.createHmac('sha256', config.AUTH_SECRET)
    .update(`${userId}:${ts}`)
    .digest('hex');
  return `${userId}:${ts}:${hmac}`;
}

export function verifyToken(token: string): string | null {
  const parts = token.split(':');
  if (parts.length !== 3) return null;

  const [userId, ts, hmac] = parts;
  const age = Math.floor(Date.now() / 1000) - parseInt(ts);
  if (age > 300 || age < -30) return null; // 5 分钟有效期

  const expected = crypto.createHmac('sha256', config.AUTH_SECRET)
    .update(`${userId}:${ts}`)
    .digest('hex');

  return crypto.timingSafeEqual(Buffer.from(hmac), Buffer.from(expected))
    ? userId
    : null;
}
```

### 3.3 消息协议定义

所有消息为 JSON 格式，通过 WSS 传输。

#### 上行消息（扩展 → 服务器）

```typescript
// src/wss/protocol.ts

/** 认证 */
interface AuthMessage {
  type: 'auth';
  token: string;          // HMAC token
}

/** 心跳 */
interface PingMessage {
  type: 'ping';
  ts: number;             // 客户端时间戳
}

/** 交易信号 */
interface TradeSignalMessage {
  type: 'trade';
  id: string;             // 信号唯一 ID (UUID)
  token: string;          // 代币合约地址 (0x...)
  action: 'buy' | 'sell';
  amount: string;         // 金额字符串（买入: BNB 数量，卖出: token 数量或百分比如 "50%"）
  chain: 'bsc';
  source: string;         // 信号来源 ("gmgn" | "dexscreener" | "manual")
  slippage?: number;      // 滑点百分比 (默认 15)
  walletIds?: string[];   // 指定钱包（为空则使用全部活跃钱包）
}

/** 配置更新 */
interface ConfigMessage {
  type: 'config';
  action: 'get' | 'set';
  key?: string;
  value?: any;
}

/** 查询请求 */
interface QueryMessage {
  type: 'query';
  action: 'token_info' | 'balances' | 'tx_history' | 'wallet_list';
  params?: Record<string, any>;
}
```

#### 下行消息（服务器 → 扩展）

```typescript
/** 认证结果 */
interface AuthOkMessage {
  type: 'auth_ok';
  userId: string;
}

interface AuthFailMessage {
  type: 'auth_fail';
}

/** 心跳响应 */
interface PongMessage {
  type: 'pong';
  ts: number;             // 服务器时间戳
}

/** 交易已发送（txHash 可用） */
interface TxSentMessage {
  type: 'tx_sent';
  signalId: string;       // 对应 trade 信号的 id
  walletId: string;
  txHash: string;
  sendMs: number;         // 从收到信号到发送 tx 的耗时
}

/** 交易已确认 */
interface TxConfirmedMessage {
  type: 'tx_confirmed';
  signalId: string;
  walletId: string;
  txHash: string;
  status: 'success' | 'reverted';
  amountOut?: string;     // 实际输出量
  gasUsed?: string;
  totalMs: number;        // 从收到信号到确认的总耗时
}

/** 交易失败 */
interface TxErrorMessage {
  type: 'tx_error';
  signalId: string;
  walletId: string;
  error: string;
  stage: 'build' | 'sign' | 'send' | 'confirm';
}

/** 批量交易汇总 */
interface BatchResultMessage {
  type: 'batch_result';
  signalId: string;
  total: number;
  success: number;
  failed: number;
  results: Array<{
    walletId: string;
    txHash?: string;
    status: 'success' | 'reverted' | 'error';
    error?: string;
    amountOut?: string;
  }>;
  totalMs: number;
}

/** Token 信息响应 */
interface TokenInfoMessage {
  type: 'token_info';
  token: string;
  symbol: string;
  decimals: number;
  routeSource: number;
  approveTarget: string;
  isInternal: boolean;
  // ... 完整 TokenInfo 字段
}

/** 错误 */
interface ErrorMessage {
  type: 'error';
  message: string;
  code?: string;
}
```

### 3.4 钱包管理

```typescript
// src/wallet/manager.ts
import fs from 'fs';
import { privateKeyToAccount, Account } from 'viem/accounts';
import { encrypt, decrypt } from './crypto';
import { config } from '../config';

interface WalletEntry {
  id: string;
  label: string;
  address: string;
  // 私钥不在此结构中，运行时在内存的 accountMap 里
}

/**
 * 钱包管理器
 *
 * 存储方案（适合 1.7G 小内存服务器）：
 * - 私钥用 AES-256-GCM 加密存储在 data/wallets.enc
 * - 加密密钥来自环境变量 WALLET_MASTER_KEY（启动时输入或从文件读取）
 * - 运行时解密到内存的 Map 中，viem Account 对象常驻
 * - 单个钱包内存占用约 2KB，100 个钱包 = 200KB，可忽略
 */
export class WalletManager {
  private accounts = new Map<string, Account>();
  private wallets: WalletEntry[] = [];
  private filePath: string;

  constructor() {
    this.filePath = config.WALLET_FILE; // data/wallets.enc
  }

  async load(): Promise<void> {
    if (!fs.existsSync(this.filePath)) {
      this.wallets = [];
      return;
    }
    const encrypted = fs.readFileSync(this.filePath, 'utf-8');
    const json = decrypt(encrypted, config.WALLET_MASTER_KEY);
    const data: Array<{ id: string; label: string; privateKey: string }> = JSON.parse(json);

    this.wallets = [];
    this.accounts.clear();

    for (const w of data) {
      const key = w.privateKey.startsWith('0x') ? w.privateKey : `0x${w.privateKey}`;
      const account = privateKeyToAccount(key as `0x${string}`);
      this.accounts.set(w.id, account);
      this.wallets.push({ id: w.id, label: w.label, address: account.address });
    }
  }

  getAccount(walletId: string): Account | undefined {
    return this.accounts.get(walletId);
  }

  getAddress(walletId: string): string | undefined {
    return this.accounts.get(walletId)?.address;
  }

  getAllWallets(): WalletEntry[] {
    return this.wallets;
  }

  getActiveWalletIds(): string[] {
    return this.wallets.map(w => w.id);
  }

  /** 添加钱包并持久化 */
  async addWallet(id: string, label: string, privateKey: string): Promise<string> {
    const key = privateKey.startsWith('0x') ? privateKey : `0x${privateKey}`;
    const account = privateKeyToAccount(key as `0x${string}`);
    this.accounts.set(id, account);
    this.wallets.push({ id, label, address: account.address });
    await this.save();
    return account.address;
  }

  private async save(): Promise<void> {
    // 注意：这里需要从 accounts 反推私钥，实际实现中应保留原始私钥
    // 简化示意：实际应维护一个 privateKeys Map
    // ...
  }
}
```

#### 加密方案

```typescript
// src/wallet/crypto.ts
import crypto from 'crypto';

const ALGO = 'aes-256-gcm';
const KEY_LEN = 32;
const IV_LEN = 12;
const TAG_LEN = 16;

/**
 * 从 master password 派生加密密钥
 * PBKDF2, 100000 轮, SHA-256
 */
function deriveKey(password: string, salt: Buffer): Buffer {
  return crypto.pbkdf2Sync(password, salt, 100_000, KEY_LEN, 'sha256');
}

export function encrypt(plaintext: string, masterKey: string): string {
  const salt = crypto.randomBytes(16);
  const key = deriveKey(masterKey, salt);
  const iv = crypto.randomBytes(IV_LEN);
  const cipher = crypto.createCipheriv(ALGO, key, iv);
  const encrypted = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  const tag = cipher.getAuthTag();
  // 格式: salt(16) + iv(12) + tag(16) + ciphertext
  return Buffer.concat([salt, iv, tag, encrypted]).toString('base64');
}

export function decrypt(data: string, masterKey: string): string {
  const buf = Buffer.from(data, 'base64');
  const salt = buf.subarray(0, 16);
  const iv = buf.subarray(16, 28);
  const tag = buf.subarray(28, 44);
  const encrypted = buf.subarray(44);
  const key = deriveKey(masterKey, salt);
  const decipher = crypto.createDecipheriv(ALGO, key, iv);
  decipher.setAuthTag(tag);
  return decipher.update(encrypted) + decipher.final('utf8');
}
```

### 3.5 交易执行引擎

```typescript
// src/trading/engine.ts
import { createPublicClient, http, encodeFunctionData, parseEther, type Hex } from 'viem';
import { bsc } from 'viem/chains';
import { WalletManager } from '../wallet/manager';
import { NonceManager } from '../nonce/manager';
import { RouteCache } from './route-cache';
import { Broadcaster } from './broadcaster';
import { TxLogger } from '../storage/tx-log';
import { QUICK_ROUTER_ABI, QUICK_ROUTER_ADDRESS } from '../abi/quick-router';
import { config } from '../config';

interface TradeParams {
  token: string;
  isBuy: boolean;
  amount: string;        // BNB (买) 或 token 数量 (卖) 或百分比 "50%"
  slippage: number;       // 百分比，如 15
  routeHint?: number;     // RouteSource 枚举值
  walletId: string;
}

interface TradeResult {
  txHash: string;
  sendMs: number;
  confirmMs: number;
  totalMs: number;
  amountOut?: bigint;
  status: 'success' | 'reverted';
}

export class TradingEngine {
  private publicClient;
  private walletManager: WalletManager;
  private nonceManager: NonceManager;
  private routeCache: RouteCache;
  private broadcaster: Broadcaster;
  private txLogger: TxLogger;

  constructor(deps: {
    walletManager: WalletManager;
    nonceManager: NonceManager;
    routeCache: RouteCache;
    broadcaster: Broadcaster;
    txLogger: TxLogger;
  }) {
    this.publicClient = createPublicClient({
      chain: bsc,
      transport: http(config.RPC_URL),
    });
    this.walletManager = deps.walletManager;
    this.nonceManager = deps.nonceManager;
    this.routeCache = deps.routeCache;
    this.broadcaster = deps.broadcaster;
    this.txLogger = deps.txLogger;
  }

  async executeTrade(params: TradeParams): Promise<TradeResult> {
    const t0 = performance.now();

    const account = this.walletManager.getAccount(params.walletId);
    if (!account) throw new Error(`钱包未找到: ${params.walletId}`);

    // 1. 路由决策
    const tokenInfo = await this.routeCache.getTokenInfo(params.token);
    const routeHint = params.routeHint ?? tokenInfo.routeSource;

    // 2. 计算金额和滑点
    let amountIn: bigint;
    let value: bigint;
    let amountOutMin: bigint;

    if (params.isBuy) {
      amountIn = parseEther(params.amount);
      value = amountIn;
      // 报价 + 滑点
      const estimated = await this.getQuote(params.token, true, amountIn);
      amountOutMin = estimated * BigInt(Math.floor((100 - params.slippage) * 100)) / 10000n;
    } else {
      amountIn = this.parseSellAmount(params.amount, params.token, params.walletId);
      value = 0n;
      const estimated = await this.getQuote(params.token, false, amountIn);
      amountOutMin = estimated * BigInt(Math.floor((100 - params.slippage) * 100)) / 10000n;

      // 确保已 approve
      await this.ensureApproval(params.walletId, params.token, tokenInfo.approveTarget, amountIn);
    }

    // 3. 构建交易
    const data = encodeFunctionData({
      abi: QUICK_ROUTER_ABI,
      functionName: 'trade',
      args: [
        params.token as Hex,
        params.isBuy,
        params.isBuy ? 0n : amountIn,
        amountOutMin,
        routeHint,
      ],
    });

    // 4. Gas 估算（按路由类型设置精确值）
    const gasLimit = this.estimateGasLimit(routeHint, params.isBuy);
    const gasPrice = await this.getGasPrice();

    // 5. Nonce
    const nonce = await this.nonceManager.getNext(account.address);

    // 6. 签名
    const signedTx = await account.signTransaction({
      to: QUICK_ROUTER_ADDRESS,
      data,
      value,
      gas: gasLimit,
      gasPrice,
      nonce,
      chainId: 56,
      type: 'legacy', // BSC 用 legacy 交易更快
    });

    // 7. 多 RPC 广播
    const tSend = performance.now();
    const txHash = await this.broadcaster.broadcast(signedTx);
    const sendMs = performance.now() - t0;

    // 8. 异步等待确认（不阻塞返回）
    const confirmPromise = this.publicClient.waitForTransactionReceipt({
      hash: txHash,
      timeout: 30_000,
    });

    // 先返回 txHash，异步等确认
    const receipt = await confirmPromise;
    const totalMs = performance.now() - t0;

    // 9. 记录日志
    await this.txLogger.log({
      txHash,
      walletId: params.walletId,
      token: params.token,
      action: params.isBuy ? 'buy' : 'sell',
      amountIn: amountIn.toString(),
      status: receipt.status,
      gasUsed: receipt.gasUsed.toString(),
      timestamp: Date.now(),
    });

    return {
      txHash,
      sendMs,
      confirmMs: totalMs - sendMs,
      totalMs,
      status: receipt.status === 'success' ? 'success' : 'reverted',
    };
  }

  private estimateGasLimit(route: number, isBuy: boolean): bigint {
    // 精确 gas 估算，避免过度预估
    const GAS_MAP: Record<number, bigint> = {
      1: 150_000n,   // FOUR_INTERNAL_BNB
      2: 250_000n,   // FOUR_INTERNAL_ERC20 (包含 swap)
      3: 250_000n,   // FOUR_EXTERNAL (PCS)
      4: 200_000n,   // FLAP_BONDING
      5: 250_000n,   // FLAP_BONDING_SELL
      6: 250_000n,   // FLAP_DEX (PCS)
      7: 250_000n,   // PANCAKE_ONLY
    };
    const base = GAS_MAP[route] ?? 300_000n;
    // 卖出需要额外的 transferFrom + approve gas
    return isBuy ? base : base + 50_000n;
  }

  private async getGasPrice(): Promise<bigint> {
    // BSC 固定 gas price 策略
    // 默认 3 gwei，可配置额外 priority fee
    return BigInt(config.GAS_PRICE_GWEI) * 1_000_000_000n;
  }

  private async getQuote(token: string, isBuy: boolean, amountIn: bigint): Promise<bigint> {
    return await this.publicClient.readContract({
      address: QUICK_ROUTER_ADDRESS,
      abi: QUICK_ROUTER_ABI,
      functionName: 'quote',
      args: [token, isBuy, amountIn],
    });
  }

  private parseSellAmount(amount: string, token: string, walletId: string): bigint {
    if (amount.endsWith('%')) {
      const pct = parseInt(amount);
      // 从缓存的余额计算
      const balance = this.routeCache.getBalance(token, walletId);
      return (balance * BigInt(pct)) / 100n;
    }
    // 直接解析为 token 数量
    const decimals = this.routeCache.getDecimals(token);
    return BigInt(Math.floor(parseFloat(amount) * 10 ** decimals));
  }

  private async ensureApproval(walletId: string, token: string, spender: string, minAmount: bigint) {
    // 检查链上 allowance
    const account = this.walletManager.getAccount(walletId)!;
    const allowance = await this.publicClient.readContract({
      address: token as Hex,
      abi: [{ name: 'allowance', type: 'function', stateMutability: 'view',
              inputs: [{ type: 'address' }, { type: 'address' }], outputs: [{ type: 'uint256' }] }],
      functionName: 'allowance',
      args: [account.address, spender],
    });

    if (allowance >= minAmount) return;

    // 发送 approve 交易
    const MAX_UINT256 = 2n ** 256n - 1n;
    const approveData = encodeFunctionData({
      abi: [{ name: 'approve', type: 'function', stateMutability: 'nonpayable',
              inputs: [{ type: 'address' }, { type: 'uint256' }], outputs: [{ type: 'bool' }] }],
      functionName: 'approve',
      args: [spender, MAX_UINT256],
    });

    const nonce = await this.nonceManager.getNext(account.address);
    const gasPrice = await this.getGasPrice();
    const signedTx = await account.signTransaction({
      to: token as Hex,
      data: approveData,
      gas: 100_000n,
      gasPrice,
      nonce,
      chainId: 56,
      type: 'legacy',
    });
    const txHash = await this.broadcaster.broadcast(signedTx);
    await this.publicClient.waitForTransactionReceipt({ hash: txHash, timeout: 30_000 });
  }
}
```

### 3.6 Nonce 管理器

```typescript
// src/nonce/manager.ts
import { PublicClient } from 'viem';

/**
 * Nonce 管理策略：
 * 1. 启动时从链上获取每个地址的 nonce
 * 2. 维护本地计数器，每次 getNext() 原子递增
 * 3. 支持并行 nonce（同时发送 nonce=N 和 nonce=N+1）
 * 4. Nonce 卡死自动修复（发一笔同 nonce 高 gas 的空交易覆盖）
 *
 * 关键设计：不要每次 eth_getTransactionCount，这在 BSC 0.45s 出块下太慢
 */
export class NonceManager {
  // address → 当前可用 nonce
  private nonces = new Map<string, number>();
  // address → 正在 pending 的 nonce 集合
  private pending = new Map<string, Set<number>>();
  private publicClient: PublicClient;

  constructor(publicClient: PublicClient) {
    this.publicClient = publicClient;
  }

  /** 初始化：从链上获取当前 nonce */
  async init(addresses: string[]): Promise<void> {
    await Promise.all(addresses.map(async (addr) => {
      const count = await this.publicClient.getTransactionCount({ address: addr as `0x${string}` });
      this.nonces.set(addr.toLowerCase(), count);
      this.pending.set(addr.toLowerCase(), new Set());
    }));
  }

  /** 原子获取下一个 nonce */
  getNext(address: string): number {
    const addr = address.toLowerCase();
    const current = this.nonces.get(addr);
    if (current === undefined) throw new Error(`Nonce 未初始化: ${addr}`);
    this.nonces.set(addr, current + 1);
    this.pending.get(addr)!.add(current);
    return current;
  }

  /** 交易确认后释放 nonce */
  confirm(address: string, nonce: number): void {
    this.pending.get(address.toLowerCase())?.delete(nonce);
  }

  /** 交易失败后回退 nonce（只有当该 nonce 是最高的才回退） */
  rollback(address: string, nonce: number): void {
    const addr = address.toLowerCase();
    const current = this.nonces.get(addr);
    if (current !== undefined && current === nonce + 1) {
      this.nonces.set(addr, nonce);
    }
    this.pending.get(addr)?.delete(nonce);
  }

  /** 从链上同步 nonce（修复卡死） */
  async resync(address: string): Promise<void> {
    const addr = address.toLowerCase();
    const count = await this.publicClient.getTransactionCount({ address: addr as `0x${string}` });
    this.nonces.set(addr, count);
    this.pending.get(addr)?.clear();
  }

  /** 检测卡死：如果某个 nonce pending 超过 30 秒，返回该 nonce */
  getStuckNonces(address: string): number[] {
    // 实际实现需要跟踪 pending 开始时间
    // 简化：返回所有 pending nonce
    return [...(this.pending.get(address.toLowerCase()) ?? [])];
  }
}
```

### 3.7 Nonce 卡死修复

```typescript
// src/nonce/recovery.ts
import { Account, encodeFunctionData } from 'viem';
import { NonceManager } from './manager';
import { Broadcaster } from '../trading/broadcaster';

/**
 * Nonce 卡死修复策略：
 * 发一笔 同 nonce + 更高 gasPrice 的空交易（发给自己 0 BNB）覆盖
 */
export async function fixStuckNonce(
  account: Account,
  stuckNonce: number,
  gasPrice: bigint,
  broadcaster: Broadcaster
): Promise<string> {
  // gasPrice 比原交易高 50% 确保覆盖
  const bumpedGasPrice = gasPrice * 150n / 100n;

  const signedTx = await account.signTransaction({
    to: account.address,
    value: 0n,
    gas: 21_000n,
    gasPrice: bumpedGasPrice,
    nonce: stuckNonce,
    chainId: 56,
    type: 'legacy',
  });

  return await broadcaster.broadcast(signedTx);
}
```

### 3.8 多 RPC 广播器

```typescript
// src/trading/broadcaster.ts
import { type Hex } from 'viem';
import { config } from '../config';

/**
 * 多 RPC 竞速广播器
 * 同一笔签名交易同时发给所有防夹 RPC，谁先返回用谁的结果
 * 使用 Promise.any — 只要有一个成功就返回
 */
export class Broadcaster {
  private endpoints: string[];

  constructor() {
    this.endpoints = [
      config.RPC_48CLUB,     // https://rpc-bsc.48.club
      config.RPC_BLOCKRAZOR, // BlockRazor 防夹 RPC
      config.RPC_URL,        // 备用公共 RPC
    ].filter(Boolean);
  }

  async broadcast(signedTx: Hex): Promise<Hex> {
    const sends = this.endpoints.map(url => this.sendRaw(url, signedTx));
    // Promise.any: 第一个成功的返回，全部失败才 reject
    return await Promise.any(sends);
  }

  private async sendRaw(rpcUrl: string, signedTx: Hex): Promise<Hex> {
    const res = await fetch(rpcUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        jsonrpc: '2.0',
        id: 1,
        method: 'eth_sendRawTransaction',
        params: [signedTx],
      }),
    });
    const json = await res.json();
    if (json.error) throw new Error(json.error.message);
    return json.result as Hex;
  }
}
```

### 3.9 跟单引擎

```typescript
// src/trading/copy-trade.ts
import { TradingEngine } from './engine';
import { WalletManager } from '../wallet/manager';
import { WssServer } from '../wss/server';

interface CopyTradeSignal {
  signalId: string;
  userId: string;
  token: string;
  action: 'buy' | 'sell';
  amount: string;
  slippage: number;
  walletIds: string[];
}

interface CopyTradeResult {
  signalId: string;
  total: number;
  success: number;
  failed: number;
  results: Array<{
    walletId: string;
    txHash?: string;
    status: 'success' | 'reverted' | 'error';
    error?: string;
    sendMs?: number;
    totalMs?: number;
  }>;
}

/**
 * 跟单引擎
 * 一个信号 → N 个钱包同时执行
 * 每个钱包独立签名、独立 nonce、并行广播
 */
export class CopyTradeEngine {
  constructor(
    private tradingEngine: TradingEngine,
    private walletManager: WalletManager,
    private wssServer: WssServer,
  ) {}

  async execute(signal: CopyTradeSignal): Promise<CopyTradeResult> {
    const walletIds = signal.walletIds.length > 0
      ? signal.walletIds
      : this.walletManager.getActiveWalletIds();

    // 并行执行所有钱包
    const promises = walletIds.map(async (walletId) => {
      try {
        const result = await this.tradingEngine.executeTrade({
          token: signal.token,
          isBuy: signal.action === 'buy',
          amount: signal.amount,
          slippage: signal.slippage,
          walletId,
        });

        // 实时通知：交易已发送
        this.wssServer.send(signal.userId, {
          type: 'tx_sent',
          signalId: signal.signalId,
          walletId,
          txHash: result.txHash,
          sendMs: result.sendMs,
        });

        return {
          walletId,
          txHash: result.txHash,
          status: result.status as 'success' | 'reverted',
          sendMs: result.sendMs,
          totalMs: result.totalMs,
        };
      } catch (e: any) {
        // 实时通知：交易失败
        this.wssServer.send(signal.userId, {
          type: 'tx_error',
          signalId: signal.signalId,
          walletId,
          error: e.message,
          stage: 'build',
        });

        return {
          walletId,
          status: 'error' as const,
          error: e.message,
        };
      }
    });

    const results = await Promise.allSettled(promises);
    const mapped = results.map(r => r.status === 'fulfilled' ? r.value : {
      walletId: 'unknown',
      status: 'error' as const,
      error: (r as PromiseRejectedResult).reason?.message,
    });

    const success = mapped.filter(r => r.status === 'success').length;
    const failed = mapped.length - success;

    const result: CopyTradeResult = {
      signalId: signal.signalId,
      total: mapped.length,
      success,
      failed,
      results: mapped,
    };

    // 发送批量结果汇总
    this.wssServer.send(signal.userId, {
      type: 'batch_result',
      ...result,
    });

    return result;
  }
}
```

### 3.10 路由缓存

```typescript
// src/trading/route-cache.ts
import { createPublicClient, http } from 'viem';
import { bsc } from 'viem/chains';
import { QUICK_ROUTER_ABI, QUICK_ROUTER_ADDRESS } from '../abi/quick-router';
import { config } from '../config';

interface CachedTokenInfo {
  routeSource: number;
  approveTarget: string;
  decimals: number;
  symbol: string;
  isInternal: boolean;
  quoteToken: string;
  fetchedAt: number;
}

/**
 * 路由缓存
 * - 首次查询从链上拉取 getTokenInfo()
 * - 缓存 30 秒（热门 token）
 * - 定时刷新活跃 token（每 10 秒）
 * - 交易时传 routeHint 跳过链上检测
 */
export class RouteCache {
  private cache = new Map<string, CachedTokenInfo>();
  private balances = new Map<string, bigint>(); // "token:wallet" → balance
  private publicClient;
  private refreshInterval: NodeJS.Timeout;

  private TTL = 30_000; // 30 秒缓存有效期

  constructor() {
    this.publicClient = createPublicClient({
      chain: bsc,
      transport: http(config.RPC_URL),
    });
  }

  /** 启动定期刷新 */
  startAutoRefresh(activeTokens: () => string[]) {
    this.refreshInterval = setInterval(async () => {
      const tokens = activeTokens();
      await Promise.allSettled(tokens.map(t => this.refresh(t)));
    }, 10_000);
  }

  async getTokenInfo(token: string): Promise<CachedTokenInfo> {
    const cached = this.cache.get(token.toLowerCase());
    if (cached && Date.now() - cached.fetchedAt < this.TTL) {
      return cached;
    }
    return this.refresh(token);
  }

  async refresh(token: string): Promise<CachedTokenInfo> {
    const info = await this.publicClient.readContract({
      address: QUICK_ROUTER_ADDRESS,
      abi: QUICK_ROUTER_ABI,
      functionName: 'getTokenInfo',
      args: [token, '0x0000000000000000000000000000000000000000'],
    });

    const cached: CachedTokenInfo = {
      routeSource: Number(info.routeSource),
      approveTarget: info.approveTarget,
      decimals: info.decimals,
      symbol: info.symbol,
      isInternal: info.isInternal,
      quoteToken: info.quoteToken,
      fetchedAt: Date.now(),
    };

    this.cache.set(token.toLowerCase(), cached);
    return cached;
  }

  getDecimals(token: string): number {
    return this.cache.get(token.toLowerCase())?.decimals ?? 18;
  }

  getBalance(token: string, walletId: string): bigint {
    return this.balances.get(`${token.toLowerCase()}:${walletId}`) ?? 0n;
  }

  setBalance(token: string, walletId: string, balance: bigint) {
    this.balances.set(`${token.toLowerCase()}:${walletId}`, balance);
  }

  stop() {
    clearInterval(this.refreshInterval);
  }
}
```

### 3.11 交易记录存储

**方案选择：SQLite**

原因：
- 单文件，适合 1.7G 小内存服务器
- 不需要额外进程
- 支持 SQL 查询，便于统计分析
- better-sqlite3 是同步 API，性能极好

```typescript
// src/storage/db.ts
import Database from 'better-sqlite3';
import path from 'path';
import { config } from '../config';

let db: Database.Database;

export function initDB(): Database.Database {
  db = new Database(path.join(config.DATA_DIR, 'quicktrader.db'));
  db.pragma('journal_mode = WAL');  // WAL 模式支持并发读写
  db.pragma('synchronous = NORMAL');
  migrate(db);
  return db;
}

export function getDB(): Database.Database {
  return db;
}

function migrate(db: Database.Database) {
  db.exec(`
    CREATE TABLE IF NOT EXISTS tx_log (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      tx_hash TEXT NOT NULL UNIQUE,
      wallet_id TEXT NOT NULL,
      wallet_address TEXT NOT NULL,
      token TEXT NOT NULL,
      action TEXT NOT NULL CHECK(action IN ('buy', 'sell', 'approve')),
      amount_in TEXT NOT NULL,
      amount_out TEXT,
      status TEXT NOT NULL CHECK(status IN ('pending', 'success', 'reverted', 'error')),
      route_source INTEGER,
      gas_used TEXT,
      gas_price TEXT,
      error_message TEXT,
      signal_id TEXT,
      created_at INTEGER NOT NULL DEFAULT (unixepoch()),
      confirmed_at INTEGER
    );

    CREATE INDEX IF NOT EXISTS idx_tx_log_wallet ON tx_log(wallet_id, created_at DESC);
    CREATE INDEX IF NOT EXISTS idx_tx_log_token ON tx_log(token, created_at DESC);
    CREATE INDEX IF NOT EXISTS idx_tx_log_signal ON tx_log(signal_id);

    CREATE TABLE IF NOT EXISTS token_cache (
      token TEXT PRIMARY KEY,
      symbol TEXT,
      decimals INTEGER,
      route_source INTEGER,
      approve_target TEXT,
      is_internal INTEGER,
      updated_at INTEGER NOT NULL DEFAULT (unixepoch())
    );
  `);
}
```

```typescript
// src/storage/tx-log.ts
import { getDB } from './db';

interface TxLogEntry {
  txHash: string;
  walletId: string;
  walletAddress?: string;
  token: string;
  action: 'buy' | 'sell' | 'approve';
  amountIn: string;
  amountOut?: string;
  status: string;
  routeSource?: number;
  gasUsed?: string;
  gasPrice?: string;
  errorMessage?: string;
  signalId?: string;
  timestamp: number;
}

export class TxLogger {
  log(entry: TxLogEntry) {
    const db = getDB();
    db.prepare(`
      INSERT OR REPLACE INTO tx_log
      (tx_hash, wallet_id, wallet_address, token, action, amount_in, amount_out,
       status, route_source, gas_used, gas_price, error_message, signal_id, created_at)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    `).run(
      entry.txHash, entry.walletId, entry.walletAddress ?? '',
      entry.token, entry.action, entry.amountIn, entry.amountOut ?? null,
      entry.status, entry.routeSource ?? null, entry.gasUsed ?? null,
      entry.gasPrice ?? null, entry.errorMessage ?? null,
      entry.signalId ?? null, entry.timestamp,
    );
  }

  updateStatus(txHash: string, status: string, amountOut?: string, gasUsed?: string) {
    const db = getDB();
    db.prepare(`
      UPDATE tx_log SET status = ?, amount_out = ?, gas_used = ?, confirmed_at = unixepoch()
      WHERE tx_hash = ?
    `).run(status, amountOut ?? null, gasUsed ?? null, txHash);
  }

  getRecent(limit = 50): any[] {
    return getDB().prepare('SELECT * FROM tx_log ORDER BY created_at DESC LIMIT ?').all(limit);
  }

  getByWallet(walletId: string, limit = 20): any[] {
    return getDB().prepare('SELECT * FROM tx_log WHERE wallet_id = ? ORDER BY created_at DESC LIMIT ?').all(walletId, limit);
  }
}
```

### 3.12 错误处理与重试策略

```typescript
// src/utils/retry.ts

interface RetryOptions {
  maxRetries: number;
  baseDelay: number;     // ms
  maxDelay: number;      // ms
  shouldRetry?: (error: any) => boolean;
}

const DEFAULT_OPTIONS: RetryOptions = {
  maxRetries: 3,
  baseDelay: 200,
  maxDelay: 2000,
};

/**
 * 重试策略：指数退避
 *
 * 重试场景（shouldRetry 判断）：
 * - RPC 超时/网络错误 → 重试
 * - nonce too low → resync nonce 后重试
 * - gas estimation failed → 重试（可能是 RPC 负载高）
 * - replacement transaction underpriced → 提高 gas 重试
 *
 * 不重试场景：
 * - 交易 revert（合约逻辑错误，重试无意义）
 * - insufficient balance → 不重试
 * - 用户取消 → 不重试
 */
export async function withRetry<T>(
  fn: () => Promise<T>,
  options: Partial<RetryOptions> = {},
): Promise<T> {
  const opts = { ...DEFAULT_OPTIONS, ...options };
  let lastError: any;

  for (let attempt = 0; attempt <= opts.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (e: any) {
      lastError = e;
      if (attempt === opts.maxRetries) break;
      if (opts.shouldRetry && !opts.shouldRetry(e)) break;

      const delay = Math.min(opts.baseDelay * 2 ** attempt, opts.maxDelay);
      await new Promise(r => setTimeout(r, delay));
    }
  }
  throw lastError;
}

/** RPC 错误是否可重试 */
export function isRetryableRpcError(error: any): boolean {
  const msg = error?.message?.toLowerCase() ?? '';
  return msg.includes('timeout') ||
    msg.includes('econnrefused') ||
    msg.includes('econnreset') ||
    msg.includes('rate limit') ||
    msg.includes('429') ||
    msg.includes('502') ||
    msg.includes('503');
}
```

### 3.13 PM2 部署配置

```javascript
// ecosystem.config.cjs
module.exports = {
  apps: [{
    name: 'quicktrader',
    script: 'dist/index.js',
    cwd: '/opt/quicktrader/server',
    instances: 1,               // 单实例（nonce 管理需要单进程）
    exec_mode: 'fork',
    max_memory_restart: '500M', // 1.7G 总内存，限制 500M
    env: {
      NODE_ENV: 'production',
      WSS_PORT: '8083',
      // 其他环境变量从 .env 加载
    },
    // 日志配置
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
    error_file: '/opt/quicktrader/logs/error.log',
    out_file: '/opt/quicktrader/logs/out.log',
    merge_logs: true,
    log_file: '/opt/quicktrader/logs/combined.log',
    // 自动重启
    watch: false,
    autorestart: true,
    max_restarts: 10,
    restart_delay: 3000,
  }],
};
```

### 3.14 配置文件

```typescript
// src/config.ts
import dotenv from 'dotenv';
import path from 'path';

dotenv.config();

export const config = {
  // WSS
  WSS_PORT: parseInt(process.env.WSS_PORT || '8083'),
  TLS_CERT: process.env.TLS_CERT || '/etc/letsencrypt/live/jbot.live/fullchain.pem',
  TLS_KEY: process.env.TLS_KEY || '/etc/letsencrypt/live/jbot.live/privkey.pem',

  // 认证
  AUTH_SECRET: process.env.AUTH_SECRET || '', // 必须设置

  // RPC
  RPC_URL: process.env.RPC_URL || 'https://bsc-dataseed.bnbchain.org',
  RPC_48CLUB: process.env.RPC_48CLUB || 'https://rpc-bsc.48.club',
  RPC_BLOCKRAZOR: process.env.RPC_BLOCKRAZOR || '', // BlockRazor 的防夹 RPC

  // Gas
  GAS_PRICE_GWEI: parseInt(process.env.GAS_PRICE_GWEI || '3'),

  // 钱包
  WALLET_FILE: process.env.WALLET_FILE || path.join(__dirname, '../data/wallets.enc'),
  WALLET_MASTER_KEY: process.env.WALLET_MASTER_KEY || '', // 必须设置

  // 存储
  DATA_DIR: process.env.DATA_DIR || path.join(__dirname, '../data'),

  // 安全
  MAX_TRADE_BNB: parseFloat(process.env.MAX_TRADE_BNB || '5'),       // 单笔最大 BNB
  MAX_DAILY_BNB: parseFloat(process.env.MAX_DAILY_BNB || '50'),      // 每日最大 BNB
  RATE_LIMIT_PER_MIN: parseInt(process.env.RATE_LIMIT_PER_MIN || '30'), // 每分钟最大交易数

  // 合约
  QUICK_ROUTER: process.env.QUICK_ROUTER || '', // 部署后填入
};
```

### 3.15 服务器入口

```typescript
// src/index.ts
import { WssServer } from './wss/server';
import { WalletManager } from './wallet/manager';
import { NonceManager } from './nonce/manager';
import { RouteCache } from './trading/route-cache';
import { Broadcaster } from './trading/broadcaster';
import { TradingEngine } from './trading/engine';
import { CopyTradeEngine } from './trading/copy-trade';
import { TxLogger } from './storage/tx-log';
import { initDB } from './storage/db';
import { parseSignal } from './trading/signal-parser';
import { createPublicClient, http } from 'viem';
import { bsc } from 'viem/chains';
import { config } from './config';

async function main() {
  console.log('QuickTrader Server starting...');

  // 1. 初始化数据库
  initDB();

  // 2. 初始化钱包
  const walletManager = new WalletManager();
  await walletManager.load();
  console.log(`Loaded ${walletManager.getAllWallets().length} wallets`);

  // 3. 初始化 Nonce 管理
  const publicClient = createPublicClient({ chain: bsc, transport: http(config.RPC_URL) });
  const nonceManager = new NonceManager(publicClient);
  await nonceManager.init(walletManager.getAllWallets().map(w => w.address));

  // 4. 初始化子系统
  const routeCache = new RouteCache();
  const broadcaster = new Broadcaster();
  const txLogger = new TxLogger();
  const tradingEngine = new TradingEngine({
    walletManager, nonceManager, routeCache, broadcaster, txLogger
  });

  // 5. 启动 WSS
  const wssServer = new WssServer((userId, msg) => {
    handleSignal(userId, msg);
  });

  // 6. 跟单引擎
  const copyTradeEngine = new CopyTradeEngine(tradingEngine, walletManager, wssServer);

  // 信号处理
  async function handleSignal(userId: string, msg: any) {
    if (msg.type === 'trade') {
      const signal = parseSignal(msg);
      if (!signal) {
        wssServer.send(userId, { type: 'error', message: '无效信号' });
        return;
      }

      // 安全检查：限额
      if (signal.action === 'buy') {
        const bnbAmount = parseFloat(signal.amount);
        if (bnbAmount > config.MAX_TRADE_BNB) {
          wssServer.send(userId, { type: 'error', message: `单笔超限: ${bnbAmount} > ${config.MAX_TRADE_BNB} BNB` });
          return;
        }
      }

      await copyTradeEngine.execute({
        signalId: signal.id,
        userId,
        token: signal.token,
        action: signal.action,
        amount: signal.amount,
        slippage: signal.slippage ?? 15,
        walletIds: signal.walletIds ?? [],
      });
    }

    if (msg.type === 'query') {
      if (msg.action === 'token_info') {
        const info = await routeCache.getTokenInfo(msg.params.token);
        wssServer.send(userId, { type: 'token_info', ...info });
      }
      if (msg.action === 'wallet_list') {
        const wallets = walletManager.getAllWallets();
        wssServer.send(userId, { type: 'wallet_list', wallets });
      }
    }
  }

  // 7. 路由缓存自动刷新
  const activeTokens: string[] = []; // 从最近交易记录动态更新
  routeCache.startAutoRefresh(() => activeTokens);

  console.log(`QuickTrader Server ready on wss://jbot.live:${config.WSS_PORT}`);
}

main().catch(console.error);
```

---

## 4. Chrome 扩展设计

### 4.1 项目结构

```
extension/
├── manifest.json
├── package.json
├── tsconfig.json
├── vite.config.ts              # Vite 构建配置
├── src/
│   ├── background/
│   │   ├── index.ts            # Background Service Worker 入口
│   │   ├── wss-client.ts       # WSS 客户端（连接、重连、消息路由）
│   │   └── auth.ts             # 认证 token 管理
│   ├── content/
│   │   ├── index.ts            # Content Script 入口
│   │   ├── gmgn.ts             # GMGN.ai 信号捕获
│   │   ├── dexscreener.ts      # DexScreener 信号捕获
│   │   ├── signal-capture.ts   # 通用信号捕获框架
│   │   └── address-extract.ts  # 合约地址提取
│   ├── popup/
│   │   ├── index.html
│   │   ├── popup.ts            # Popup 交互逻辑
│   │   └── popup.css
│   └── shared/
│       ├── types.ts            # 共享类型定义
│       ├── protocol.ts         # 消息协议（与服务器一致）
│       └── constants.ts        # 常量
├── public/
│   ├── icons/                  # 扩展图标
│   └── _locales/               # i18n
└── dist/                       # 构建输出
```

### 4.2 Manifest V3 配置

```json
{
  "manifest_version": 3,
  "name": "QuickTrader",
  "version": "1.0.0",
  "description": "BSC 交易信号捕获",
  "permissions": [
    "storage",
    "activeTab",
    "sidePanel"
  ],
  "host_permissions": [
    "https://gmgn.ai/*",
    "https://dexscreener.com/*",
    "wss://jbot.live:8083/*"
  ],
  "background": {
    "service_worker": "background/index.js",
    "type": "module"
  },
  "content_scripts": [
    {
      "matches": [
        "https://gmgn.ai/*",
        "https://dexscreener.com/*"
      ],
      "js": ["content/index.js"],
      "run_at": "document_idle"
    }
  ],
  "action": {
    "default_popup": "popup/index.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  },
  "side_panel": {
    "default_path": "popup/index.html"
  },
  "icons": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}
```

### 4.3 Content Script — 信号捕获

```typescript
// src/content/index.ts
import { initGmgnCapture } from './gmgn';
import { initDexScreenerCapture } from './dexscreener';
import { extractContractFromUrl } from './address-extract';

// 根据当前 URL 初始化对应的信号捕获
const url = window.location.href;
if (url.includes('gmgn.ai')) {
  initGmgnCapture();
} else if (url.includes('dexscreener.com')) {
  initDexScreenerCapture();
}

// 监听 SPA 路由变化
let lastUrl = url;
const observer = new MutationObserver(() => {
  const current = window.location.href;
  if (current !== lastUrl) {
    lastUrl = current;
    onRouteChange(current);
  }
});
observer.observe(document.body, { childList: true, subtree: true });

// 同时用 History API 拦截
const origPushState = history.pushState;
history.pushState = function (...args) {
  origPushState.apply(this, args);
  onRouteChange(window.location.href);
};
const origReplaceState = history.replaceState;
history.replaceState = function (...args) {
  origReplaceState.apply(this, args);
  onRouteChange(window.location.href);
};
window.addEventListener('popstate', () => onRouteChange(window.location.href));

function onRouteChange(newUrl: string) {
  const contract = extractContractFromUrl(newUrl);
  if (contract) {
    chrome.runtime.sendMessage({
      type: 'contract_detected',
      address: contract.address,
      chain: contract.chain,
    });
  }
}
```

#### GMGN 信号捕获

```typescript
// src/content/gmgn.ts
import { extractContractFromUrl } from './address-extract';

/**
 * GMGN.ai 信号捕获策略：
 *
 * 1. 买卖按钮点击拦截
 *    - GMGN 的买卖按钮有特定的 class/data 属性
 *    - 使用 MutationObserver 监听 DOM 变化（SPA 动态渲染）
 *    - 事件代理在 document 上，捕获阶段拦截
 *
 * 2. 快捷键捕获
 *    - GMGN 支持 Space+Q/W/E/R (买入不同金额) 和 Space+A/S/D/F (卖出不同比例)
 *    - 拦截 keydown 事件，识别组合键
 *
 * 3. 合约地址提取
 *    - 从 URL 提取: gmgn.ai/bsc/token/0x...
 *    - 从页面 DOM 提取（备用）
 */
export function initGmgnCapture() {
  // 1. 按钮点击捕获
  document.addEventListener('click', (e) => {
    const target = e.target as HTMLElement;
    const btn = target.closest('[data-testid*="buy"], [data-testid*="sell"], .trade-btn');
    if (!btn) return;

    const contract = extractContractFromUrl(window.location.href);
    if (!contract) return;

    const isBuy = btn.textContent?.toLowerCase().includes('buy') ||
                  btn.getAttribute('data-testid')?.includes('buy');
    const action = isBuy ? 'buy' : 'sell';

    // 尝试从页面获取金额
    const amountInput = document.querySelector('input[placeholder*="amount"], input[data-testid*="amount"]') as HTMLInputElement;
    const amount = amountInput?.value || '';

    sendSignal({
      action,
      token: contract.address,
      chain: contract.chain,
      amount,
      source: 'gmgn',
    });
  }, true); // 捕获阶段

  // 2. 快捷键捕获
  let spaceHeld = false;
  document.addEventListener('keydown', (e) => {
    if (e.code === 'Space') { spaceHeld = true; return; }
    if (!spaceHeld) return;

    const contract = extractContractFromUrl(window.location.href);
    if (!contract) return;

    // Space + Q/W/E/R → 买入快捷键
    const buyKeys: Record<string, string> = { KeyQ: '0.01', KeyW: '0.05', KeyE: '0.1', KeyR: '0.5' };
    // Space + A/S/D/F → 卖出快捷键
    const sellKeys: Record<string, string> = { KeyA: '25%', KeyS: '50%', KeyD: '75%', KeyF: '100%' };

    if (buyKeys[e.code]) {
      sendSignal({ action: 'buy', token: contract.address, chain: contract.chain, amount: buyKeys[e.code], source: 'gmgn' });
    } else if (sellKeys[e.code]) {
      sendSignal({ action: 'sell', token: contract.address, chain: contract.chain, amount: sellKeys[e.code], source: 'gmgn' });
    }
  });

  document.addEventListener('keyup', (e) => {
    if (e.code === 'Space') spaceHeld = false;
  });
}

function sendSignal(signal: {
  action: string;
  token: string;
  chain: string;
  amount: string;
  source: string;
}) {
  chrome.runtime.sendMessage({
    type: 'trade_signal',
    ...signal,
    id: crypto.randomUUID(),
    timestamp: Date.now(),
  });
}
```

#### DexScreener 信号捕获

```typescript
// src/content/dexscreener.ts
import { extractContractFromUrl } from './address-extract';

/**
 * DexScreener 信号捕获
 * DexScreener 没有内建的买卖按钮，主要功能：
 * 1. 合约地址自动检测（从 URL）
 * 2. 快捷键触发交易（与 GMGN 相同的快捷键方案）
 * 3. 监听页面上的 "Trade" 外链按钮点击
 */
export function initDexScreenerCapture() {
  // 合约地址检测
  const contract = extractContractFromUrl(window.location.href);
  if (contract) {
    chrome.runtime.sendMessage({
      type: 'contract_detected',
      address: contract.address,
      chain: contract.chain,
    });
  }

  // 快捷键（与 GMGN 统一）
  let spaceHeld = false;
  document.addEventListener('keydown', (e) => {
    if (e.code === 'Space') { spaceHeld = true; return; }
    if (!spaceHeld) return;

    const c = extractContractFromUrl(window.location.href);
    if (!c) return;

    const buyKeys: Record<string, string> = { KeyQ: '0.01', KeyW: '0.05', KeyE: '0.1', KeyR: '0.5' };
    const sellKeys: Record<string, string> = { KeyA: '25%', KeyS: '50%', KeyD: '75%', KeyF: '100%' };

    if (buyKeys[e.code]) {
      chrome.runtime.sendMessage({
        type: 'trade_signal', id: crypto.randomUUID(),
        action: 'buy', token: c.address, chain: c.chain,
        amount: buyKeys[e.code], source: 'dexscreener', timestamp: Date.now(),
      });
    } else if (sellKeys[e.code]) {
      chrome.runtime.sendMessage({
        type: 'trade_signal', id: crypto.randomUUID(),
        action: 'sell', token: c.address, chain: c.chain,
        amount: sellKeys[e.code], source: 'dexscreener', timestamp: Date.now(),
      });
    }
  });
  document.addEventListener('keyup', (e) => {
    if (e.code === 'Space') spaceHeld = false;
  });
}
```

#### 合约地址提取

```typescript
// src/content/address-extract.ts

interface DetectedContract {
  address: string;
  chain: 'bsc' | 'sol';
}

const BSC_PATTERNS = [
  /gmgn\.ai\/(?:bsc|defi)\/token\/.+?\/(0x[a-fA-F0-9]{40})/i,
  /dexscreener\.com\/bsc\/(0x[a-fA-F0-9]{40})/i,
  /dexscreener\.com\/[^/]*\/(0x[a-fA-F0-9]{40})/i,
];

export function extractContractFromUrl(url: string): DetectedContract | null {
  for (const pattern of BSC_PATTERNS) {
    const match = url.match(pattern);
    if (match) {
      return { address: match[1].toLowerCase(), chain: 'bsc' };
    }
  }

  // 通用 0x 地址兜底（仅在已知 BSC 相关页面）
  if (url.includes('bsc') || url.includes('bnb')) {
    const match = url.match(/(0x[a-fA-F0-9]{40})/i);
    if (match) return { address: match[1].toLowerCase(), chain: 'bsc' };
  }

  return null;
}
```

### 4.4 Background Service Worker — WSS 客户端

```typescript
// src/background/index.ts
import { WssClient } from './wss-client';
import { getAuthToken } from './auth';

const wssClient = new WssClient();

// 启动 WSS 连接
chrome.runtime.onInstalled.addListener(async () => {
  await wssClient.connect();
});

// SW 唤醒时也尝试连接
wssClient.connect();

// 来自 Content Script 的消息
chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  if (msg.type === 'trade_signal') {
    // 通过 WSS 发送到服务器
    wssClient.send({
      type: 'trade',
      id: msg.id,
      token: msg.token,
      action: msg.action,
      amount: msg.amount,
      chain: msg.chain || 'bsc',
      source: msg.source,
      slippage: msg.slippage,
    });
    sendResponse({ ok: true });
  }

  if (msg.type === 'contract_detected') {
    // 可选：自动查询 token info
    wssClient.send({
      type: 'query',
      action: 'token_info',
      params: { token: msg.address },
    });
  }

  if (msg.type === 'get_status') {
    sendResponse({ connected: wssClient.isConnected() });
  }

  return true;
});
```

```typescript
// src/background/wss-client.ts

const WSS_URL = 'wss://jbot.live:8083';
const RECONNECT_DELAYS = [1000, 2000, 5000, 10000, 30000]; // 递增退避

export class WssClient {
  private ws: WebSocket | null = null;
  private reconnectAttempt = 0;
  private authenticated = false;
  private messageQueue: string[] = []; // 断连期间的消息队列

  async connect() {
    if (this.ws?.readyState === WebSocket.OPEN) return;

    try {
      this.ws = new WebSocket(WSS_URL);

      this.ws.onopen = async () => {
        this.reconnectAttempt = 0;
        // 认证
        const token = await this.getAuthToken();
        this.ws!.send(JSON.stringify({ type: 'auth', token }));
      };

      this.ws.onmessage = (event) => {
        const msg = JSON.parse(event.data);
        this.handleMessage(msg);
      };

      this.ws.onclose = () => {
        this.authenticated = false;
        this.scheduleReconnect();
      };

      this.ws.onerror = () => {
        this.ws?.close();
      };
    } catch (e) {
      this.scheduleReconnect();
    }
  }

  send(msg: object) {
    const data = JSON.stringify(msg);
    if (this.ws?.readyState === WebSocket.OPEN && this.authenticated) {
      this.ws.send(data);
    } else {
      // 队列缓存，重连后发送
      this.messageQueue.push(data);
      if (this.messageQueue.length > 100) this.messageQueue.shift(); // 防止溢出
    }
  }

  isConnected(): boolean {
    return this.ws?.readyState === WebSocket.OPEN && this.authenticated;
  }

  private handleMessage(msg: any) {
    if (msg.type === 'auth_ok') {
      this.authenticated = true;
      // 发送队列中的消息
      while (this.messageQueue.length > 0) {
        this.ws!.send(this.messageQueue.shift()!);
      }
    }

    if (msg.type === 'pong') return;

    // 转发到 popup/side panel
    chrome.runtime.sendMessage(msg).catch(() => {});

    // 交易结果通知
    if (msg.type === 'tx_confirmed' || msg.type === 'batch_result') {
      this.showNotification(msg);
    }
  }

  private showNotification(msg: any) {
    if (msg.type === 'batch_result') {
      chrome.notifications?.create({
        type: 'basic',
        iconUrl: 'icons/icon48.png',
        title: `交易完成 ${msg.success}/${msg.total}`,
        message: msg.failed > 0 ? `${msg.failed} 笔失败` : '全部成功',
      });
    }
  }

  private scheduleReconnect() {
    const delay = RECONNECT_DELAYS[Math.min(this.reconnectAttempt, RECONNECT_DELAYS.length - 1)];
    this.reconnectAttempt++;
    setTimeout(() => this.connect(), delay);
  }

  private async getAuthToken(): Promise<string> {
    const stored = await chrome.storage.local.get(['authToken']);
    return stored.authToken || '';
  }
}
```

### 4.5 Popup/Side Panel

```html
<!-- src/popup/index.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <link rel="stylesheet" href="popup.css">
</head>
<body>
  <div id="app">
    <div class="header">
      <h2>QuickTrader</h2>
      <span id="status-dot" class="dot disconnected"></span>
    </div>

    <!-- 连接状态 -->
    <div class="section">
      <div class="label">服务器连接</div>
      <div id="connection-status">未连接</div>
    </div>

    <!-- 当前代币 -->
    <div class="section">
      <div class="label">当前代币</div>
      <div id="current-token">-</div>
    </div>

    <!-- 最近交易 -->
    <div class="section">
      <div class="label">最近交易</div>
      <div id="recent-trades"></div>
    </div>

    <!-- 设置 -->
    <div class="section">
      <div class="label">设置</div>
      <label>WSS Token: <input id="auth-token" type="password" /></label>
      <button id="save-btn">保存</button>
    </div>
  </div>
  <script src="popup.js" type="module"></script>
</body>
</html>
```

### 4.6 构建配置

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  build: {
    outDir: 'dist',
    rollupOptions: {
      input: {
        background: resolve(__dirname, 'src/background/index.ts'),
        content: resolve(__dirname, 'src/content/index.ts'),
        popup: resolve(__dirname, 'src/popup/index.html'),
      },
      output: {
        entryFileNames: '[name]/index.js',
        chunkFileNames: 'shared/[name].js',
      },
    },
    target: 'esnext',
    minify: false, // 调试阶段不压缩
  },
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
});
```

---

## 5. 速度优化方案

### 5.1 延迟预算分配（0.45s 出块）

```
总目标：从用户点击到 tx 进入 pending pool < 200ms
链上确认是不可控变量（300-3000ms），优化重点在前面的环节

延迟预算：
┌──────────────────────────────┬───────────┬──────────┐
│ 环节                         │ 预算 (ms) │ 实际范围  │
├──────────────────────────────┼───────────┼──────────┤
│ Content Script 捕获事件       │ 2         │ 1-5      │
│ chrome.runtime.sendMessage   │ 3         │ 1-5      │
│ WSS 发送 → 服务器收到         │ 30        │ 10-80    │
│ 服务器解析信号                │ 1         │ <1       │
│ 路由查询（缓存命中）          │ 0         │ 0        │
│ 报价查询 (eth_call)          │ 40        │ 20-100   │
│ 构建 TX + 编码 calldata      │ 1         │ <1       │
│ secp256k1 签名               │ 1         │ <1       │
│ 多 RPC sendRawTransaction    │ 50        │ 20-100   │
├──────────────────────────────┼───────────┼──────────┤
│ 总计（到 pending pool）      │ ~128      │ 53-292   │
│ 链上确认（不可控）            │ -         │ 450-3000 │
└──────────────────────────────┴───────────┴──────────┘
```

### 5.2 各环节优化手段

#### 报价查询优化（最大收益）

| 策略 | 说明 | 预期收益 |
|------|------|----------|
| **跳过报价** | 用缓存的 tokenInfo 估算，amountOutMin 设为 0（高风险但最快） | -40ms |
| **routeHint** | 传入缓存的路由，跳过链上路由检测 | -20ms gas 时间 |
| **预计算** | 热门 token 定期刷新报价缓存，交易时直接用缓存值 | -40ms |
| **本地计算** | PCS 代币可以用 reserve 本地算 amountOut，不需要 eth_call | -40ms |

**本地 PancakeSwap 报价计算：**

```typescript
function getAmountOut(amountIn: bigint, reserveIn: bigint, reserveOut: bigint): bigint {
  const amountInWithFee = amountIn * 9975n; // PCS 0.25% fee
  const numerator = amountInWithFee * reserveOut;
  const denominator = reserveIn * 10000n + amountInWithFee;
  return numerator / denominator;
}
```

#### WSS 延迟优化

| 策略 | 说明 | 预期收益 |
|------|------|----------|
| **长连接保持** | 不要频繁断连重连 | 省 TCP 握手 |
| **二进制协议** | 用 MessagePack 替代 JSON（可选，JSON 通常够快） | -2ms |
| **香港服务器** | 已在香港，到国内延迟 <30ms | 已优化 |

#### RPC 广播优化

| 策略 | 说明 | 预期收益 |
|------|------|----------|
| **多 RPC 竞速** | Promise.any 同时发 48.club + BlockRazor | -30ms (取最快) |
| **HTTP Keep-Alive** | RPC 连接保持长连接，省 TLS 握手 | -50ms |
| **fetch Agent 复用** | Node.js 的 undici Agent 连接池 | -20ms |

```typescript
// RPC 连接池配置
import { Agent } from 'undici';

const agent = new Agent({
  keepAliveTimeout: 60_000,
  keepAliveMaxTimeout: 300_000,
  connections: 10,
  pipelining: 1,
});

// 使用：
const res = await fetch(rpcUrl, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body,
  dispatcher: agent,
});
```

### 5.3 Nonce 预分配

```typescript
/**
 * Nonce 预分配策略：
 *
 * 场景：一个信号 → 5 个钱包同时买入
 * 普通方式：每个钱包串行 getTransactionCount → 签名 → 发送
 * 预分配方式：本地计数器一次分配 5 个 nonce，5 个钱包并行签名+发送
 *
 * 实现：
 * 1. 启动时 getTransactionCount 初始化
 * 2. 每次 getNext() 原子递增，不走 RPC
 * 3. 交易确认后标记 nonce 完成
 * 4. 30 秒未确认的 nonce → 从链上 resync
 */

// 在跟单引擎中的使用：
async function executeParallel(walletIds: string[], token: string, amount: string) {
  // 所有钱包的 nonce 一次性分配好
  const tasks = walletIds.map(id => {
    const account = walletManager.getAccount(id)!;
    const nonce = nonceManager.getNext(account.address); // 原子操作
    return { id, account, nonce };
  });

  // 并行签名 + 发送
  await Promise.allSettled(tasks.map(async (task) => {
    const signedTx = await task.account.signTransaction({ ...txParams, nonce: task.nonce });
    const txHash = await broadcaster.broadcast(signedTx);
    return txHash;
  }));
}
```

### 5.4 预计算和缓存策略

```
┌─────────────────────────────────────────────────────┐
│                    缓存层设计                        │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Layer 1: 内存缓存 (Map)                             │
│  ├── tokenInfo: token → {route, decimals, ...}      │
│  │   TTL: 30s, 最多 200 条                           │
│  ├── gasPrice: 单值缓存                              │
│  │   TTL: 3s                                        │
│  ├── reserves: pair → {r0, r1}                      │
│  │   TTL: 5s                                        │
│  └── approvals: "owner:token:spender" → boolean     │
│      TTL: 永久 (MAX_UINT256 approve 不会变)          │
│                                                     │
│  Layer 2: SQLite 持久化                              │
│  └── token_cache: 已知 token 的基础信息              │
│      用于启动时预热内存缓存                           │
│                                                     │
│  定期刷新策略：                                       │
│  ├── 热门 token（最近 1 小时交易过）: 每 10 秒        │
│  ├── 活跃 token（当前页面浏览）: 每 30 秒             │
│  └── gasPrice: 每 3 秒                               │
└─────────────────────────────────────────────────────┘
```

---

## 6. 安全设计

### 6.1 WSS 认证方案

```
┌─────────────┐        ┌──────────────┐
│ Chrome 扩展  │        │ 交易服务器    │
│             │        │              │
│ 1. 用户设置  │        │              │
│    authToken │        │              │
│ 2. WSS 连接  │───────→│              │
│ 3. 发送 auth │ {type: │ 4. HMAC 验证  │
│    消息      │ "auth",│    - 解析 token│
│             │ token} │    - 校验时间戳│
│             │        │    - 验证签名  │
│             │←───────│ 5. auth_ok    │
│ 6. 开始通信  │        │    或 断开     │
└─────────────┘        └──────────────┘

Token 生成（一次性设置）：
  HMAC-SHA256(userId + ":" + timestamp, AUTH_SECRET) → base64

安全措施：
  - TLS 加密（Let's Encrypt 证书）
  - 10 秒认证超时
  - 60 秒心跳超时
  - timing-safe 比较防止时序攻击
  - 单 IP 限流（可选）
```

### 6.2 私钥加密存储

```
存储格式：data/wallets.enc

加密算法：AES-256-GCM
密钥派生：PBKDF2(master_password, salt, 100000 rounds, SHA-256)

文件格式：Base64(salt[16] + iv[12] + tag[16] + ciphertext)

明文格式（JSON）：
[
  {"id": "w1", "label": "主钱包", "privateKey": "0x..."},
  {"id": "w2", "label": "跟单1", "privateKey": "0x..."}
]

运行时：
  1. 启动时从环境变量读 WALLET_MASTER_KEY
  2. 解密文件 → 得到私钥列表
  3. privateKeyToAccount → viem Account 对象
  4. Account 常驻内存（~2KB/钱包）
  5. 原始私钥字符串解密后立即丢弃（不保留引用）

注意：
  - WALLET_MASTER_KEY 不能写在 .env 文件中（虽然方便但不安全）
  - 推荐方案：PM2 启动时通过 --env 传入，或启动脚本交互式输入
  - 1.7G 服务器不适合跑 HashiCorp Vault 之类的 KMS
  - 实际安全级别：单机加密文件 ≈ 中等安全，适合个人使用
```

### 6.3 交易限额控制

```typescript
// 限额策略（配置在 config.ts）

interface TradeLimits {
  maxSingleBnb: number;     // 单笔最大 BNB（买入）: 5 BNB
  maxDailyBnb: number;      // 每日最大 BNB: 50 BNB
  maxTradesPerMin: number;   // 每分钟最大交易数: 30
  maxPendingTx: number;      // 最大 pending 交易数: 10
}

// 检查逻辑（在 engine.ts 的 executeTrade 前调用）：
function checkLimits(params: TradeParams): void {
  // 1. 单笔限额
  if (params.isBuy && parseFloat(params.amount) > config.MAX_TRADE_BNB) {
    throw new Error(`单笔超限: ${params.amount} > ${config.MAX_TRADE_BNB} BNB`);
  }

  // 2. 每日限额（从 SQLite 查今日总量）
  const today = getDB().prepare(
    `SELECT COALESCE(SUM(CAST(amount_in AS REAL)), 0) as total
     FROM tx_log WHERE action = 'buy' AND created_at > unixepoch() - 86400`
  ).get() as any;
  if (today.total + parseFloat(params.amount) > config.MAX_DAILY_BNB) {
    throw new Error(`每日超限`);
  }

  // 3. 频率限制
  const recentCount = getDB().prepare(
    `SELECT COUNT(*) as c FROM tx_log WHERE created_at > unixepoch() - 60`
  ).get() as any;
  if (recentCount.c >= config.RATE_LIMIT_PER_MIN) {
    throw new Error(`频率超限: ${recentCount.c}/${config.RATE_LIMIT_PER_MIN} per min`);
  }
}
```

### 6.4 日志审计

```typescript
// src/utils/logger.ts
import pino from 'pino';

/**
 * 日志安全策略：
 * - 永远不打印私钥
 * - 永远不打印完整的签名交易数据
 * - 地址只打印前 6 + 后 4 位
 * - 金额可以打印（不是敏感信息）
 */
export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: {
    target: 'pino-pretty',
    options: { colorize: true, translateTime: 'SYS:standard' }
  },
  // 自动脱敏
  redact: {
    paths: ['privateKey', 'encryptedKey', 'masterKey', 'password', 'signedTx'],
    censor: '[REDACTED]',
  },
});

export function maskAddress(addr: string): string {
  if (!addr || addr.length < 10) return addr;
  return `${addr.slice(0, 6)}...${addr.slice(-4)}`;
}

// 使用示例：
// logger.info({ wallet: maskAddress(addr), token, action: 'buy', amount: '0.1 BNB' }, 'Trade executed');
// logger.error({ wallet: maskAddress(addr), error: e.message }, 'Trade failed');
```

---

## 7. 开发阶段规划

### Phase 1: 最小可用（单钱包买卖）

**目标**：一个钱包通过扩展触发 GMGN 信号 → 服务器执行买卖 → 返回结果

**交付物**：
1. QuickRouter 合约（支持 Four.meme 内外盘 + PancakeSwap）
2. 服务器：WSS + 单钱包交易引擎 + Nonce 管理
3. 扩展：GMGN Content Script + Background WSS + 简单 Popup

**关键里程碑**：
- [ ] QuickRouter 合约编写 + Foundry 测试 + BSC 部署
- [ ] 服务器 WSS 框架 + 认证
- [ ] 钱包加密存储 + 加载
- [ ] TradingEngine 核心流程（信号→路由→构建→签名→广播）
- [ ] Nonce 管理器基础版
- [ ] 扩展 Content Script（GMGN 买卖按钮捕获）
- [ ] 扩展 Background（WSS 连接 + 消息转发）
- [ ] 端到端测试：GMGN 点击 → 链上成交

**预估工作量**：所有模块并行推进

### Phase 2: 跟单 + 多钱包

**目标**：一个信号同时驱动 N 个钱包执行，结果聚合返回

**交付物**：
1. 跟单引擎（CopyTradeEngine）
2. 多钱包 Nonce 并行管理
3. 批量结果聚合 + 通知
4. Flap 协议支持（合约 + 服务器）
5. DexScreener 信号捕获
6. 快捷键支持（Space + QWER/ASDF）

**关键里程碑**：
- [ ] CopyTradeEngine 实现（并行执行 + 结果聚合）
- [ ] Nonce 并行分配优化
- [ ] QuickRouter 添加 Flap 支持
- [ ] DexScreener Content Script
- [ ] 快捷键捕获逻辑
- [ ] 交易记录 SQLite 存储
- [ ] Popup 展示连接状态 + 最近交易

### Phase 3: 速度优化 + 监控

**目标**：信号到 pending pool < 200ms，完善监控和错误恢复

**交付物**：
1. 路由缓存 + routeHint 预判
2. 本地 PCS 报价计算
3. RPC 连接池 + HTTP Keep-Alive
4. Nonce 卡死自动修复
5. 交易限额控制
6. 日志审计系统

**关键里程碑**：
- [ ] RouteCache 实现 + 自动刷新
- [ ] 本地 PCS amountOut 计算
- [ ] undici Agent 连接池
- [ ] Nonce recovery 模块
- [ ] 限额检查中间件
- [ ] pino 日志 + 脱敏
- [ ] PM2 部署 + 监控配置

---

## 8. 文件清单

### 合约 (`contracts/`)

| 文件 | 职责 |
|------|------|
| `contracts/src/QuickRouter.sol` | 聚合路由合约主体（~350 行） |
| `contracts/script/Deploy.s.sol` | Foundry 部署脚本 |
| `contracts/test/QuickRouter.t.sol` | Foundry 单元测试 |
| `contracts/foundry.toml` | Foundry 配置 |

### 服务器 (`server/`)

| 文件 | 职责 |
|------|------|
| `server/src/index.ts` | 服务入口，初始化所有模块并启动 WSS |
| `server/src/config.ts` | 环境变量加载 + 常量定义 |
| `server/src/wss/server.ts` | WSS 服务器（连接管理、心跳、消息分发） |
| `server/src/wss/auth.ts` | HMAC token 认证 |
| `server/src/wss/protocol.ts` | 消息类型定义 + JSON schema 校验 |
| `server/src/wallet/manager.ts` | 钱包管理器（加密存储、Account 初始化） |
| `server/src/wallet/crypto.ts` | AES-256-GCM 加密/解密工具 |
| `server/src/wallet/types.ts` | 钱包相关类型定义 |
| `server/src/trading/engine.ts` | 交易执行引擎核心 |
| `server/src/trading/signal-parser.ts` | 信号解析（校验 token 格式、action、amount） |
| `server/src/trading/route-cache.ts` | 路由缓存（定期刷新 tokenInfo） |
| `server/src/trading/tx-builder.ts` | 交易构建器（encodeFunctionData + gas 估算） |
| `server/src/trading/broadcaster.ts` | 多 RPC 竞速广播器 |
| `server/src/trading/copy-trade.ts` | 跟单引擎（1 信号 → N 钱包并行） |
| `server/src/nonce/manager.ts` | Nonce 本地计数器 + 原子分配 |
| `server/src/nonce/recovery.ts` | Nonce 卡死修复（高 gas 空交易覆盖） |
| `server/src/rpc/pool.ts` | RPC 连接池（undici Agent） |
| `server/src/rpc/endpoints.ts` | RPC 节点列表 + 健康检查 |
| `server/src/storage/db.ts` | SQLite 初始化 + migration |
| `server/src/storage/tx-log.ts` | 交易记录 CRUD |
| `server/src/abi/quick-router.ts` | QuickRouter ABI 导出 |
| `server/src/utils/logger.ts` | pino 日志 + 自动脱敏 |
| `server/src/utils/retry.ts` | 重试策略（指数退避） |
| `server/ecosystem.config.cjs` | PM2 部署配置 |
| `server/package.json` | 依赖声明 |
| `server/tsconfig.json` | TypeScript 配置 |
| `server/.env.example` | 环境变量模板 |

### 扩展 (`extension/`)

| 文件 | 职责 |
|------|------|
| `extension/manifest.json` | Manifest V3 配置 |
| `extension/src/background/index.ts` | Background SW 入口（消息路由） |
| `extension/src/background/wss-client.ts` | WSS 客户端（连接、重连、消息队列） |
| `extension/src/background/auth.ts` | 认证 token 管理 |
| `extension/src/content/index.ts` | Content Script 入口（路由分发 + SPA 监听） |
| `extension/src/content/gmgn.ts` | GMGN.ai 信号捕获（按钮 + 快捷键） |
| `extension/src/content/dexscreener.ts` | DexScreener 信号捕获 |
| `extension/src/content/signal-capture.ts` | 通用信号捕获框架 |
| `extension/src/content/address-extract.ts` | URL 合约地址提取 |
| `extension/src/popup/index.html` | Popup 页面 |
| `extension/src/popup/popup.ts` | Popup 交互逻辑 |
| `extension/src/popup/popup.css` | Popup 样式 |
| `extension/src/shared/types.ts` | 共享类型定义 |
| `extension/src/shared/protocol.ts` | 消息协议（与服务器一致） |
| `extension/src/shared/constants.ts` | 常量（WSS URL 等） |
| `extension/vite.config.ts` | Vite 构建配置 |
| `extension/package.json` | 依赖声明 |
| `extension/tsconfig.json` | TypeScript 配置 |

### 依赖清单

**服务器 (server/package.json)**：
```json
{
  "dependencies": {
    "viem": "^2.x",
    "ws": "^8.x",
    "better-sqlite3": "^11.x",
    "dotenv": "^16.x",
    "pino": "^9.x",
    "pino-pretty": "^11.x",
    "undici": "^7.x"
  },
  "devDependencies": {
    "typescript": "^5.x",
    "@types/ws": "^8.x",
    "@types/better-sqlite3": "^7.x",
    "tsx": "^4.x"
  }
}
```

**扩展 (extension/package.json)**：
```json
{
  "devDependencies": {
    "typescript": "^5.x",
    "vite": "^6.x",
    "@anthropic-ai/claude-code-types": "latest"
  }
}
```

**合约 (contracts/)**：
```
forge install OpenZeppelin/openzeppelin-contracts
```

---

> 本文档覆盖了从合约到服务器到扩展的完整技术设计，每个模块的接口定义和实现指引足够详细，可直接交给 Codex 或 Claude Code 独立实现各模块。
