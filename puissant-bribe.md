# BSC 防夹贿赂机制：48 Club Puissant

> **重要补充**：Freedom Trader 源码中没有实现此机制，只用了普通 `eth_sendRawTransaction`。我们必须实现 `eth_sendPuissant` 才能真正享受防夹 + 优先打包。

---

## 核心概念

**Gas 是 gas，贿赂是贿赂。** 两者不是一回事。

- **Gas fee** = 交易执行的基本费用，谁都要付
- **贿赂（Bid）** = Puissant bundle 中第一笔交易的实际 gas fee，决定你的优先级

## Puissant API

端点：`https://puissant-bsc.48.club`

### 1. 查询最低 gasPrice 门槛

```json
{
  "jsonrpc": "2.0",
  "id": 73,
  "method": "eth_gasPrice"
}
// 返回 Puissant 接受的最低 gasPrice，低于此值直接拒绝
```

### 2. 发送单笔隐私交易

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "eth_sendPrivateRawTransaction",
  "params": ["0x签名交易hex"]
}
// 交易不进公共 mempool，仅 BNB48 及合作验证节点可见
// 适合简单的防夹需求，但没有优先排序
```

### 3. 发送 Puissant Bundle（核心）

```json
{
  "jsonrpc": "2.0",
  "id": 48,
  "method": "eth_sendPuissant",
  "params": [{
    "txs": ["0x签名交易1hex", "0x签名交易2hex"],
    "maxTimestamp": 1679900000,
    "acceptReverting": []
  }]
}
// 返回 UUID，用于查询状态
```

## 竞价机制

### Bid 计算

Puissant 的第一笔交易 = **竞价单（the bid）**

```
竞价得分 = 3 × min(10000, gas_fee / (21000 × 60gwei))

其中 gas_fee = 第一笔 tx 实际消耗的 gasPrice × gasUsed
```

- 得分越高 → 优先级越高
- 多个 Puissant 包含同一笔 tx → **bid 最高的赢（CASE BEATEN）**
- 同一 sender 的冲突交易 → 只保留 gasPrice 最高的

### 规则约束

| 规则 | 说明 |
|------|------|
| 第一笔 tx gasUsed ≥ 21000 | 否则整包作废（CASE INVALID） |
| gasPrice ≥ eth_gasPrice 返回值 | 低于门槛直接拒绝 |
| txs 排序：nonce 升序，gasPrice 降序 | 违反则作废 |
| maxTimestamp 必须在请求时间 2 分钟内 | 过期则作废（CASE EXPIRED） |
| 任一 tx revert（不在 acceptReverting 中） | 整包丢弃（CASE REVERTED） |

### 信誉机制（Score）

每个 IP 有独立信誉分，初始值 10：
- 成功打包 → 加分：`3 × min(10000, gas_fee / (21000 × 60gwei))`
- 任一 tx revert → **扣 1 分**
- 分数为负 → 下一小时内拒绝所有请求

**重要：不要提交会 revert 的交易！** 信誉分很珍贵。

## 合作验证节点

查询哪些验证节点支持 Puissant：
```
合约地址：0x5cc05fde1d231a840061c1a2d7e913cedc8eabaf
方法：getPuissants() → 返回所有合作验证节点的 coinbase 地址
```

## 实际操作策略

### 策略 A：单交易模式（简单）
```
直接用 eth_sendPrivateRawTransaction
gasPrice 设为 Puissant 最低要求的 1.5-2 倍
适合普通买卖，不需要优先级竞争
```

### 策略 B：Bundle 模式（竞争场景）
```
第一笔 tx = 贿赂交易（自转账，gasPrice 拉高，gasLimit 设 21000+）
第二笔 tx = 实际交易（买入/卖出）
用 eth_sendPuissant 提交 bundle
适合抢跑、狙击新币等需要优先级的场景
```

### 策略 C：单交易 Bundle（推荐）
```
只有一笔 tx 的 puissant
gasPrice 拉高即可同时充当 bid
避免额外的贿赂交易 gas 消耗
用 eth_sendPuissant 提交（享受原子性保护）
```

## 服务器端实现要点

```typescript
// 1. 查询最低 gasPrice
const minGasPrice = await puissantClient.getGasPrice();

// 2. 构造交易，gasPrice 高于门槛
const tx = await walletClient.signTransaction({
  to: routerAddress,
  data: tradeCalldata,
  gasPrice: minGasPrice * 150n / 100n,  // 1.5 倍门槛
  gas: 800000n,
});

// 3. 用 eth_sendPuissant 提交（而不是 eth_sendRawTransaction）
const result = await fetch('https://puissant-bsc.48.club', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    jsonrpc: '2.0',
    id: 48,
    method: 'eth_sendPuissant',
    params: [{
      txs: [tx],
      maxTimestamp: Math.floor(Date.now() / 1000) + 120,
      acceptReverting: []
    }]
  })
});

// 4. 返回 UUID，查询状态
const uuid = result.result;
// GET https://explorer.48.club/api/v1/puissant/{uuid}
```

## KOGE 持有者 Gas 折扣

48 Club 还有一个 `koge-rpc-bsc` 服务：

| KOGE 持有量 | 可享 1 gwei 的 gasLimit 上限 |
|------------|--------------------------|
| < 1 | 无折扣 |
| 1-10 | 240,000 |
| 100 | 480,000 |
| 1,000 | 960,000 |
| 10,000 | 1,920,000 |

持有 KOGE 代币可以用 1 gwei 发交易（远低于正常 gasPrice），但只在 48Club 出块时有效。

端点：`https://koge-rpc-bsc.48.club`

## 与 BlockRazor 的配合

建议同时向多个防夹节点提交：
1. `eth_sendPuissant` → `puissant-bsc.48.club`（Bundle 模式）
2. `eth_sendPrivateRawTransaction` → `debot.bsc.blockrazor.xyz`（单笔隐私）

同一笔签名交易广播到多个防夹节点，谁先打包算谁的。

---

> 数据来源：[48Club/docs-gitbook-cn](https://github.com/48Club/docs-gitbook-cn)、[48Club/docs-gitbook-en](https://github.com/48Club/docs-gitbook-en)、[48Club/enhanced_rpc](https://github.com/48Club/enhanced_rpc)
