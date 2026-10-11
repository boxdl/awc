# AWC-Chain 完整 API 文档

## 1. 项目概述

AWC-Chain 是一个基于 DPoS 共识的区块链项目，包含：

- AWC 原生币转账
- 账户系统
- Bandwidth / Energy 资源模型
- 冻结 / 解冻
- 超级代表 SR 注册与投票
- 资源委托 / 收回委托
- 自定义代币发行与转账
- 区块和交易查询
- P2P 节点同步
- TronLink / TRON RPC 兼容接口
- 基础 TVM 合约系统

项目同时提供两套 HTTP API：

| API 类型 | 默认端口 | 说明 |
|---|---:|---|
| 原生 REST API | `port`，如 `8800` | 项目自己的 `/api/...` 接口 |
| TRON 兼容 RPC API | `port + 100`，如 `8900` | `/wallet/...` 接口，钱包和前端优先使用 |
| P2P 端口 | `public_addr` 配置，如 `9800` | 节点同步，不是 HTTP API |

当前部署端口（端口 = `config.json` 的 `port`，RPC = `port + 100`）：

| 节点 | 原生 API | TRON RPC | P2P | 说明 |
|---|---:|---:|---:|---|
| 主节点 | 8800 | 8900 | 9800 | 本机 + 公网（tpai.wanys.xyz） |
| sr1 | 8200 | 8300 | 9200 | 本机，出块节点之一 |

推荐开发和钱包接入优先使用 **TRON RPC API**，例如：

```bash
http://127.0.0.1:8900
```

### 配套收款测试页

仓库根目录 `payment-test.html` 是一个零依赖单文件网页，直接用浏览器打开即可：

- 填入 RPC 地址（默认 `http://127.0.0.1:8900`），3 秒轮询余额、带宽、节点高度
- 展示收款地址 + 二维码 + 一键复制，实时监听新到账交易（到账横幅 + 提示音 + 会话累计）
- 期望金额提醒：填入目标金额，累计达标后大字提示
- 内置发送测试：填私钥签名广播一笔，闭环验证"发出 → 出块 → 监听到账"
- 金额换算：页面按 `1 AWC = 100 内部单位` 显示，接口原始 `balance` 为内部单位

### 配套区块浏览器

仓库根目录 `explorer.html` 同为零依赖单文件，浏览器直接打开：

- 首页：链状态（高度 / 最新块龄 / 平均出块间隔 / 近40块交易**火花线**与活跃地址 / 出块者分布）+ 最新区块表（含间隔列）+ 最近交易表（类型分布 chips），每 4 秒自动刷新（只增量拉新块）
- 历史区块 `#/blocks/<高度>`：20 块/页翻页浏览（更新 ↔ 更早 ↔ 最新），区间显示
- 区块详情 `#/block/<高度|hash>`：区块头（含见证签名）+ 区块内交易表（类型 / 金额 / From → To / TxID）+ **前/后块翻页**（← 更早 / 更新 →，按当前高度封顶）+ 原始响应 JSON
- 交易详情 `#/tx/<hash>`：确认数（≥19 最终确认）、类型、金额、过期时间、执行结果、原始响应 JSON；txID 查不到时自动回退尝试按区块 Hash 打开
- 地址页 `#/addr/<T...>`：余额 / 激活状态 / 带宽 / nonce + **SR 徽章**（若为见证人，`/wallet/listwitnesses` 匹配）+ **交易统计卡**（收到/付出总额、类型分布）+ **冻结与资源代理明细**（冻结带宽/能量、到期时间、代出/收到代理）+ **投票明细**（候选人票数条、投票力/已用/可用）+ 最近 100 笔交易（收/付方向按 base58 载荷比较，兼容非标准校验和地址）
- 见证人页 `#/witness`（`/wallet/listwitnesses`）：活跃数 / epoch / 总票数 / 注册费统计 + SR 列表（排名、票数占比条、**近30块实时出块数**、累计出块/漏块、在岗状态）
- 网络页 `#/network`：本节点信息（`/wallet/getnodeinfo`：高度、连接数、监听端口、P2P版本）+ P2P 对等节点表（REST `/api/p2p/status` + `/api/p2p/peers`：节点ID、入站/出站、对端高度与本节点差异、最近活跃）+ **链参数表**（`/wallet/getchainparameters`，27 项，中文标签+原始 key）
- 搜索框自动识别：纯数字 → 区块高度；64 位 hex → 交易（失败转区块）；`T` 开头 → 地址
- RPC 默认连接公网 `http://mc.wanys.xyz:8900`，可切换本地节点并记忆（localStorage 键 `awc_exp_rpc_v2`），与 `payment-test.html` 一致支持 `file://` 直接打开（节点 CORS `*`）
- 资产页 `#/assets`（独立页，网络页不再塞这些）：
  - **资产总览**：发行代币 / 本地钱包 / 部署合约 / 近50块活动 计数
  - **代币资产**：`/wallet/gettokenlist` 表（名称/符号/TokenID/总量/精度/发行人）+ **近50块代币活动**扫描（TxType9 创建、10 转账）；空态给出 `/wallet/createassetissue`、`/wallet/transferasset` 创建指引
  - **本地钱包**（REST `/api/wallet/list`）：地址 / 余额（实时 `/wallet/getaccount`）/ 交易次数 / **代币持仓**（`/wallet/getassetissuebyaccount`）；仅地址不含私钥
  - **智能合约**（REST `/api/contract/list`）：合约地址 / 创建者 / 余额 / Gas 用量上限 / 代码与存储键数 + **VM 开关徽章**（合约创建、自适应能量，取自 `getnodeinfo`）+ **近50块合约活动**扫描（TxType4 部署、5 调用）+ 部署接口指引
  - REST 不可达时整块降级提示（代币与活动区不依赖 REST，照常显示）
- 依赖 REST 的卡片（P2P / 本地钱包 / 智能合约）要求节点 ≥ **b27**（REST `/api/*` 已挂 CORS；REST 端口 = RPC 端口 − 100）；页脚可互跳 `payment-test.html` 收款测试页

---

## 2. 通用规则

### 2.1 请求格式

除特别说明外，请求均使用 JSON：

```http
Content-Type: application/json
```

### 2.2 地址格式

对外 API 使用 Base58Check 地址，通常以 `T` 开头，例如：

```text
TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk
```

内部状态使用 hex 地址，例如：

```text
3c71d60db57ec051eed2248eaa9816d79f5b8c65
```

### 2.3 金额单位

当前链内金额使用整数最小单位：

```text
100 = 1 AWC
```

示例：

| 显示金额 | API amount |
|---:|---:|
| 1 AWC | 100 |
| 10 AWC | 1000 |
| 100 AWC | 10000 |

代币数量目前也是整数，前端如需小数显示可按 `decimals = 6` 自行换算。

### 2.4 签名规则

所有会改变链状态的接口 `private_key` 为**可选**：

- 提供 `private_key`：后端自动签名、加入 pending pool、广播到 P2P
- 不提供 `private_key`：返回未签名交易 + `raw_data_hex`，供前端本地签名后通过 `broadcasttransaction` 广播

⚠️ 私钥是敏感信息，永远不要在文档、日志或前端代码中暴露。生产环境推荐前端本地签名模式。

### 2.5 出块确认

区块间隔约 10 秒。提交交易后建议等待：

```text
8 - 12 秒
```

再查询余额或状态。

---

# 3. TRON 兼容 RPC API `/wallet/...`

基础地址示例：

```text
http://127.0.0.1:8900
```

---

## 3.1 创建账户（已禁用）

### 接口

```http
POST /wallet/createaccount
```

### 说明

**当前已禁用，恒返回 `403`：**

```json
{"error":"Account creation is disabled. Accounts are activated automatically by incoming transfers."}
```

账户只能通过**首笔转账自动激活**（附带 1 AWC 激活费），公网匿名建户会灌状态、绕过激活费，因此关闭。
下方参数表仅作历史参考，实际调用一律 403。

### 请求参数（历史参考）

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| owner_address | string | 是 | 账户所有者地址 |
| account_address | string | 是 | 要创建的新地址 |
| private_key | string | 否 | 私钥；提供则自动签名并广播 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/createaccount \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "account_address":"TGQgfK497YXmjdgvun9Bg5Zu3xE15v17cu",
    "private_key":"<private_key_hex>"
  }'
```

---

## 3.2 查询最新区块

### 接口

```http
POST /wallet/getnowblock
```

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/getnowblock \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 响应核心字段

```json
{
  "blockID": "...",
  "block_header": {
    "raw_data": {
      "number": 1290,
      "parentHash": "...",
      "timestamp": 1785130000000
    }
  }
}
```

---

## 3.3 按高度查询区块

### 接口

```http
POST /wallet/getblockbynum
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| num | int64 | 是 | 区块高度 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/getblockbynum \
  -H "Content-Type: application/json" \
  -d '{"num": 100}'
```

### 响应说明

`transactions[]` 中每笔交易同时包含两种形态：

- TRON 嵌套形态：`raw_data.contract[0].parameter.value`（兼容钱包/TronLink 习惯）
- 顶层扁平字段（b26 起新增，便于浏览器/前端直接读取）：

| 字段 | 类型 | 说明 |
|---|---|---|
| type | int | 交易类型码（0 转账 / 1 冻结 / 2 解冻 / 3 投票 / 4 部署 / 5 调用 / 6 注册SR / 7 委托 / 8 收回委托 / 9 创建代币 / 10 代币转账） |
| type_name | string | 类型名（`transfer` `delegate` `undelegate` 等，与 `/wallet/gettransactionsbyaddress` 命名一致） |
| from / to | string | Base58 地址（to 对投票/注册类为空字符串） |
| amount | int64 | 内部单位金额 |
| fee | int64 | 手续费 |
| nonce | int64 | 账户交易序号 |
| timestamp | int64 | 毫秒时间戳 |

> 注意：节点重启加载新二进制后该组字段才出现；旧响应只有嵌套形态，前端应做兼容（`explorer.html` 即按"扁平优先、嵌套回退"实现）。

---

## 3.4 按区块 ID 查询区块

### 接口

```http
POST /wallet/getblockbyid
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| value | string | 是 | 区块 hash / blockID |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/getblockbyid \
  -H "Content-Type: application/json" \
  -d '{"value":"<block_id>"}'
```

---

## 3.5 按 txID 查询交易

### 接口

```http
POST /wallet/gettransactionbyid
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| value | string | 是 | 交易 ID |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/gettransactionbyid \
  -H "Content-Type: application/json" \
  -d '{"value":"<txID>"}'
```

### 响应核心字段

| 字段 | 类型 | 说明 |
|---|---|---|
| txID | string | 交易 ID |
| blockNumber | int64 | 所在区块高度；未上链为 `0`（走 pending 池兜底响应） |
| confirmations | int | `当前高度 - 所在块高度 + 1`；未上链为 `0` |
| confirmed | bool | `confirmations >= 19` 即视为最终确认 |
| ret[0].contractRet | string | 执行结果，如 `SUCCESS` |
| raw_data.contract[0] | object | TRON 嵌套交易内容（type / parameter.value） |
| type / type_name / from / to / amount / fee / nonce / timestamp | — | 顶层扁平字段，同 §3.3 说明（b26 起新增） |

未上链且不在 pending 池时返回 `404 {"Error":"Transaction not found"}`。

---

## 3.6 创建 AWC 转账交易

### 接口

```http
POST /wallet/createtransaction
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| owner_address | string | 是 | 发送者地址 |
| to_address | string | 是 | 接收者地址 |
| amount | int64 | 是 | 转账金额，最小单位 |
| private_key | string | 否 | 私钥；提供则自动签名并广播，不提供则返回未签名交易 |

### 示例（服务端签名）

```bash
curl -X POST http://127.0.0.1:8900/wallet/createtransaction \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "to_address":"TGQgfK497YXmjdgvun9Bg5Zu3xE15v17cu",
    "amount":1000,
    "private_key":"<private_key_hex>"
  }'
```

### 示例（前端本地签名）

不传 `private_key`：

```bash
curl -X POST http://127.0.0.1:8900/wallet/createtransaction \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "to_address":"TGQgfK497YXmjdgvun9Bg5Zu3xE15v17cu",
    "amount":1000
  }'
```

返回未签名交易：

```json
{
  "txID": "",
  "owner_address": "TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
  "to_address": "TGQgfK497YXmjdgvun9Bg5Zu3xE15v17cu",
  "amount": 1000,
  "fee": 1,
  "nonce": 42,
  "timestamp": 1695000000,
  "raw_data_hex": "00000000...",
  "signature": null,
  "public_key": null,
  "signed": false,
  "expiration": 1695000060,
  "contract_type": "TransferContract"
}
```

前端用 `raw_data_hex` 本地签名后，调用 `broadcasttransaction` 广播。

---

## 3.7 广播交易

### 接口

```http
POST /wallet/broadcasttransaction
```

### 说明

广播已签名交易。支持两种模式：

1. **传统模式**：传 `txID` 查找已签名交易（兼容旧客户端）
2. **新模式**：传完整签名交易（前端本地签名场景）

### 请求参数（新模式）

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| raw_data_hex | string | 是 | 待签原始数据（hex） |
| signature | string | 是 | 签名（hex） |
| public_key | string | 是 | 公钥（hex） |
| owner_address | string | 是 | 发送者地址 |
| to_address | string | 是 | 接收者地址 |
| amount | int64 | 是 | 转账金额 |
| fee | int64 | 是 | 手续费 |
| nonce | int64 | 是 | 交易序号 |
| timestamp | int64 | 是 | 交易时间戳 |

### 示例（前端签名后广播）

```bash
curl -X POST http://127.0.0.1:8900/wallet/broadcasttransaction \
  -H "Content-Type: application/json" \
  -d '{
    "raw_data_hex":"00000000...",
    "signature":"<签名hex>",
    "public_key":"<公钥hex>",
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "to_address":"TGQgfK497YXmjdgvun9Bg5Zu3xE15v17cu",
    "amount":1000,
    "fee":1,
    "nonce":42,
    "timestamp":1695000000
  }'
```

---

## 3.8 查询账户

### 接口

```http
POST /wallet/getaccount
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| address | string | 是 | 查询地址 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/getaccount \
  -H "Content-Type: application/json" \
  -d '{"address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk"}'
```

### 返回字段说明

```json
{
  "address": "TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
  "balance": 100000,
  "bandwidth": 5000,
  "energy": 10000,
  "frozen_balance_for_bandwidth": 50000,
  "frozen_balance_for_energy": 30000,
  "votes": {
    "TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk": 500
  },
  "nonce": 12,
  "delegated_frozen_for_bandwidth": 0,
  "delegated_frozen_for_energy": 0
}
```

字段说明：

- `address` - 账户地址
- `balance` - 可用余额（最小单位）
- `bandwidth` - 可用带宽
- `energy` - 可用能量
- `frozen_balance_for_bandwidth` - 为带宽冻结的金额
- `frozen_balance_for_energy` - 为能量冻结的金额
- `votes` - 投票记录
- `nonce` - 交易序号（防重放）
- `delegated_frozen_for_bandwidth` - 委托出去的带宽冻结金额
- `delegated_frozen_for_energy` - 委托出去的能量冻结金额

---

## 3.9 地址转换 / 获取地址

### 接口

```http
POST /wallet/getaddress
```

### 说明

用于地址格式转换或根据输入获取地址信息。

---

## 3.10 查询地址交易历史

### 接口

```http
POST /wallet/gettransactionsbyaddress
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| address | string | 是 | 地址 |
| limit | int | 否 | 返回数量 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/gettransactionsbyaddress \
  -H "Content-Type: application/json" \
  -d '{"address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk","limit":50}'
```

---

# 4. 资源模型 API

## 4.1 冻结余额

### 接口

```http
POST /wallet/freezebalance
```

### 功能

冻结 AWC 获取 Bandwidth 或 Energy。

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| owner_address | string | 是 | 账户地址 |
| frozen_balance | int64 | 是 | 冻结金额，最小单位 |
| resource | string | 否 | `BANDWIDTH` 或 `ENERGY`，默认 `BANDWIDTH` |
| frozen_days | int64 | 否 | 冻结天数，默认 3；测试可传 0 |
| private_key | string | 否 | 私钥；提供则自动签名并广播 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/freezebalance \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "frozen_balance":10000,
    "resource":"BANDWIDTH",
    "frozen_days":3,
    "private_key":"<private_key_hex>"
  }'
```

---

## 4.2 解冻余额

### 接口

```http
POST /wallet/unfreezebalance
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| owner_address | string | 是 | 账户地址 |
| resource | string | 否 | `BANDWIDTH` 或 `ENERGY` |
| private_key | string | 否 | 私钥；提供则自动签名并广播 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/unfreezebalance \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "resource":"BANDWIDTH",
    "private_key":"<private_key_hex>"
  }'
```

---

## 4.3 委托资源

### 接口

```http
POST /wallet/delegateresource
```

### 功能

把自己冻结得到的资源委托给其他地址。

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| owner_address | string | 是 | 委托人地址 |
| to_address | string | 是 | 接收资源地址 |
| amount | int64 | 是 | 委托金额 |
| resource | string | 否 | `BANDWIDTH` 或 `ENERGY` |
| private_key | string | 否 | 私钥；提供则自动签名并广播 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/delegateresource \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "to_address":"TGQgfK497YXmjdgvun9Bg5Zu3xE15v17cu",
    "amount":10000,
    "resource":"BANDWIDTH",
    "private_key":"<private_key_hex>"
  }'
```

---

## 4.4 收回委托资源

### 接口

```http
POST /wallet/undelegateresource
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| owner_address | string | 是 | 委托人地址 |
| to_address | string | 是 | 被委托人地址 |
| resource | string | 否 | `BANDWIDTH` 或 `ENERGY` |
| private_key | string | 否 | 私钥；提供则自动签名并广播 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/undelegateresource \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "to_address":"TGQgfK497YXmjdgvun9Bg5Zu3xE15v17cu",
    "resource":"BANDWIDTH",
    "private_key":"<private_key_hex>"
  }'
```

---

# 5. DPoS / 超级代表 API

## 5.1 查询超级代表列表

### 接口

```http
POST /wallet/listwitnesses
```

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/listwitnesses \
  -H "Content-Type: application/json" \
  -d '{}'
```

---

## 5.2 注册超级代表

### 接口

```http
POST /wallet/registerwitness
```

### 说明

注册成为超级代表候选人。需要 5000 AWC 押金。

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| owner_address | string | 是 | 注册地址 |
| url | string | 否 | 节点 URL 或备注 |
| private_key | string | 否 | 私钥；提供则自动签名并广播 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/registerwitness \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "url":"sr-node-1",
    "private_key":"<private_key_hex>"
  }'
```

---

## 5.3 投票超级代表

### 接口

```http
POST /wallet/votewitness
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| owner_address | string | 是 | 投票人地址 |
| votes | map | 是 | 投票映射：`{"SR地址": 票数}` |
| private_key | string | 否 | 私钥；提供则自动签名并广播 |

### votes 格式

```json
{
  "TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk": 100
}
```

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/votewitness \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TGQgfK497YXmjdgvun9Bg5Zu3xE15v17cu",
    "votes":{
      "TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk": 100
    },
    "private_key":"<private_key_hex>"
  }'
```

### 撤票

把票数传 `0` 表示撤销该候选人的投票。

---

# 6. 代币 API

## 6.1 发行代币

### 接口

```http
POST /wallet/createassetissue
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| owner_address | string | 是 | 发行者地址 |
| name | string | 是 | 代币名称 |
| symbol | string | 是 | 代币符号 |
| total_supply | int64 | 是 | 总供应量 |
| private_key | string | 否 | 私钥；提供则自动签名并广播 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/createassetissue \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "name":"SyncToken",
    "symbol":"SYN",
    "total_supply":123456,
    "private_key":"<private_key_hex>"
  }'
```

---

## 6.2 查询代币列表

### 接口

```http
GET /wallet/gettokenlist
POST /wallet/gettokenlist
```

### 示例

```bash
curl http://127.0.0.1:8900/wallet/gettokenlist
```

---

## 6.3 转账代币

### 接口

```http
POST /wallet/transferasset
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| owner_address | string | 是 | 发送者地址 |
| to_address | string | 是 | 接收者地址 |
| token_id | string | 是 | 代币 ID |
| amount | int64 | 是 | 转账数量 |
| private_key | string | 否 | 私钥；提供则自动签名并广播 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/transferasset \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "to_address":"TGQgfK497YXmjdgvun9Bg5Zu3xE15v17cu",
    "token_id":"token_3c71d60db57ec051eed2248eaa9816d79f5b8c65_1785131194",
    "amount":1234,
    "private_key":"<private_key_hex>"
  }'
```

---

## 6.4 查询账户代币余额

### 接口

```http
POST /wallet/getassetissuebyaccount
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| address | string | 是 | 查询地址 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/getassetissuebyaccount \
  -H "Content-Type: application/json" \
  -d '{"address":"TGQgfK497YXmjdgvun9Bg5Zu3xE15v17cu"}'
```

### 响应示例

```json
{
  "asset": [
    {
      "token_id": "token_3c71d60db57ec051eed2248eaa9816d79f5b8c65_1785131194",
      "balance": 1234
    }
  ]
}
```

---

# 7. 节点信息 API

## 7.1 查询节点信息

### 接口

```http
POST /wallet/getnodeinfo
```

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/getnodeinfo \
  -H "Content-Type: application/json" \
  -d '{}'
```

---

## 7.2 查询链参数

### 接口

```http
POST /wallet/getchainparameters
```

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/getchainparameters \
  -H "Content-Type: application/json" \
  -d '{}'
```

---

## 7.3 部署智能合约

### 接口

```http
POST /wallet/deploycontract
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| owner_address | string | 是 | 部署者地址 |
| code | string | 是 | 合约代码 |
| gas_limit | int64 | 否 | Gas 限额 |
| private_key | string | 否 | 私钥；提供则自动签名并广播 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/deploycontract \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "code":"PUSH 42 PUSH 10 ADD RETURN",
    "gas_limit":1000,
    "private_key":"<private_key_hex>"
  }'
```

---

## 7.4 调用智能合约

### 接口

```http
POST /wallet/triggersmartcontract
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| owner_address | string | 是 | 调用者地址 |
| contract_address | string | 是 | 合约地址 |
| input | string | 是 | 调用参数 |
| call_value | int64 | 否 | 随合约转账金额 |
| gas_limit | int64 | 否 | Gas 限额 |
| private_key | string | 否 | 私钥；提供则自动签名并广播 |

### 示例

```bash
curl -X POST http://127.0.0.1:8900/wallet/triggersmartcontract \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "contract_address":"<contract_address>",
    "input":"42",
    "private_key":"<private_key_hex>"
  }'
```

---

# 8. Solidity 兼容接口

这些接口和 `/wallet/...` 功能一致，用于兼容钱包：

```http
POST /walletsolidity/getaccount
POST /walletsolidity/getnowblock
```

示例：

```bash
curl -X POST http://127.0.0.1:8900/walletsolidity/getnowblock \
  -H "Content-Type: application/json" \
  -d '{}'
```

---

# 9. 原生 REST API `/api/...`

基础地址示例：

```text
http://127.0.0.1:8800
```

> 注意：当前钱包和前端建议优先使用 `/wallet/...` 接口；`/api/...` 主要用于调试和内部工具。

---

## 9.1 区块链接口

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/blockchain` | 查询链信息和区块列表 |
| GET | `/api/blockchain/height` | 查询当前高度 |
| POST | `/api/blockchain/mine` | 手动挖块（已禁用） |
| GET | `/api/blockchain/block/{number}` | 按高度查询区块 |
| GET | `/api/blockchain/block/hash/{hash}` | 按 hash 查询区块 |
| GET | `/api/blockchain/transaction/{hash}` | 查询交易 |

---

## 9.2 账户接口

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/account/create` | 创建账户（已禁用，恒返回 403） |
| POST | `/api/account/transfer` | AWC 转账（不带私钥只返回未签名交易，不消耗带宽） |
| POST | `/api/account/freeze` | 冻结余额 |
| POST | `/api/account/unfreeze` | 解冻余额 |
| GET | `/api/account/{address}` | 查询账户详情 |
| GET | `/api/account/{address}/balance` | 查询余额 |
| GET | `/api/account/{address}/transactions` | 查询交易历史 |

---

## 9.3 钱包接口

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/wallet/create` | 创建钱包（已禁用） |
| GET | `/api/wallet/list` | 查询钱包列表 |
| GET | `/api/wallet/balance/{address}` | 查询钱包余额 |

## 9.4 合约接口

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/contract/deploy` | 部署 TVM 合约 |
| POST | `/api/contract/execute` | 执行 TVM 合约（已禁用） |
| GET | `/api/contract/list` | 查询合约列表 |
| GET | `/api/contract/{address}` | 查询合约详情 |
| GET | `/api/contract/balance/{address}` | 查询合约余额 |
| GET | `/api/contract/storage/{address}/{key}` | 查询合约存储 |
| POST | `/api/contract/storage/{address}/{key}` | 设置合约存储（已禁用） |

### 部署合约示例

```bash
curl -X POST http://127.0.0.1:8800/api/contract/deploy \
  -H "Content-Type: application/json" \
  -d '{
    "owner":"<hex_address>",
    "code":"PUSH 42 PUSH 10 ADD RETURN",
    "gas_limit":1000
  }'
```

### 执行合约示例

```bash
curl -X POST http://127.0.0.1:8800/api/contract/execute \
  -H "Content-Type: application/json" \
  -d '{
    "address":"<contract_address>",
    "input":42,
    "amount":0
  }'
```

---

## 9.5 DPoS 接口

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/dpos/super-representatives` | 查询 SR 列表 |
| POST | `/api/dpos/register` | 注册 SR（已禁用） |
| POST | `/api/dpos/vote` | 投票（已禁用） |
| GET | `/api/dpos/epoch` | 查询 epoch |
| GET | `/api/dpos/total-votes` | 查询总票数 |

---

## 9.6 P2P 接口

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/p2p/peers` | 查询已连接节点 |
| POST | `/api/p2p/connect` | 主动连接节点 |
| GET | `/api/p2p/status` | 查询 P2P 状态 |

### 连接节点示例

```bash
curl -X POST http://127.0.0.1:8800/api/p2p/connect \
  -H "Content-Type: application/json" \
  -d '{"address":"127.0.0.1","port":9400}'
```

---

# 10. 完整使用流程

## 10.1 启动节点

```bash
cd /mnt/sdc22/opencode/tron-chain/localnet/sr2
./awc-chain serve --config config.json
```

或后台 screen 启动：

```bash
screen -dmS sr2 ./awc-chain serve --config config.json
```

## 10.2 查询最新区块

```bash
curl -X POST http://127.0.0.1:8900/wallet/getnowblock \
  -H "Content-Type: application/json" \
  -d '{}'
```

## 10.3 查询账户余额

```bash
curl -X POST http://127.0.0.1:8900/wallet/getaccount \
  -H "Content-Type: application/json" \
  -d '{"address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk"}'
```

## 10.4 AWC 转账

```bash
curl -X POST http://127.0.0.1:8900/wallet/createtransaction \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "to_address":"TGQgfK497YXmjdgvun9Bg5Zu3xE15v17cu",
    "amount":1000,
    "private_key":"<private_key_hex>"
  }'
```

## 10.5 发行代币

```bash
curl -X POST http://127.0.0.1:8900/wallet/createassetissue \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "name":"DemoToken",
    "symbol":"DEMO",
    "total_supply":1000000,
    "private_key":"<private_key_hex>"
  }'
```

## 10.6 查询代币 ID

```bash
curl http://127.0.0.1:8900/wallet/gettokenlist
```

## 10.7 转账代币

```bash
curl -X POST http://127.0.0.1:8900/wallet/transferasset \
  -H "Content-Type: application/json" \
  -d '{
    "owner_address":"TFUov3SmNRkQqNpavfDFD7tC7AqCF8XKrk",
    "to_address":"TGQgfK497YXmjdgvun9Bg5Zu3xE15v17cu",
    "token_id":"<token_id>",
    "amount":10000,
    "private_key":"<private_key_hex>"
  }'
```

---

# 11. 错误说明

常见错误：

| 错误 | 原因 | 处理 |
|---|---|---|
| `Invalid address` | 地址格式错误 | 检查是否为 T 开头 Base58 地址 |
| `Account not found` | 账户不存在 | 先转入 AWC 激活账户 |
| `Insufficient funds` | AWC 不足 | 增加余额或减少金额 |
| `Insufficient token balance` | 代币余额不足 | 检查代币余额 |
| `Token not found` | token_id 错误 | 用 `/wallet/gettokenlist` 查询 |
| `Invalid private key` | 私钥格式错误 | 使用 hex 私钥 |
| `Failed to sign transaction` | 签名失败 | 检查私钥和地址是否匹配 |
| `invalid nonce` | nonce 不匹配 | 等待 pending 交易确认后重试 |

---

# 12. 交易类型

| Type | 名称 | 说明 |
|---:|---|---|
| 0 | Transfer | AWC 转账 |
| 1 | FreezeBalance | 冻结余额 |
| 2 | UnfreezeBalance | 解冻余额 |
| 3 | VoteWitness | 投票 SR |
| 4 | DeployContract | 部署合约 |
| 5 | TriggerContract | 触发合约 |
| 6 | RegisterWitness | 注册 SR |
| 7 | Delegate | 委托资源 |
| 8 | Undelegate | 收回委托资源 |
| 9 | CreateToken | 发行代币 |
| 10 | TransferToken | 转账代币 |

