# usdtpay

TRON 链上地址实时监听与回调服务，让您的系统在几秒内感知每一笔 **TRX 原生转账** 和 **USDT(TRC20) 转账**。

只需一个 HTTP 接口提交要监听的地址，交易发生时我们立即把结构化事件 **Webhook 推送到您自己的服务器**——无需自建节点、无需维护同步器、无需盯着链上扫块。

USDT,收款,支付,TRC20,trx,充值,提现,API支付接口,监控,回调,USDT TRC20

> 开通服务 / 获取接入凭证，请联系：[https://t.me/secp256k0](https://t.me/secp256k0)

---

## 目录

- [为什么选择 usdtpay](#为什么选择-usdtpay)
- [应用场景](#应用场景)
- [工作原理](#工作原理)
- [快速开始](#快速开始)
- [鉴权](#鉴权)
- [监听事件类型](#监听事件类型)
- [API 参考（v2）](#api-参考v2)
- [Webhook 回调](#webhook-回调)
- [兼容旧版 SDK（v1）](#兼容旧版-sdkv1)
- [错误码与排障](#错误码与排障)
- [联系我们](#联系我们)

---

## 为什么选择 usdtpay

| 优势 | 说明 |
| ---- | ---- |
| **实时** | 服务端持续追块，约 3 秒一轮扫块，交易确认后数秒内回调送达 |
| **免运维** | 部署在 Cloudflare Workers 全球网络，无需自建 TRON 节点或同步器 |
| **接入简单** | 一个 POST 接口提交地址，业务数据通过 Webhook 主动推送给您 |
| **多租户隔离** | 同一地址可被多个业务线独立监听，各配各的回调地址与事件类型 |
| **防伪可验证** | 回调携带 HMAC-SHA256 签名（`X-Api-Sign`），可确认请求确实来自 usdtpay |
| **批量能力** | 支持一次提交/移除海量地址，也支持批量更新回调信息 |
| **成本低** | 按调用与绑定配额计费，无需为闲置地址或空闲机器买单（额度详情联系开通时确认） |

## 应用场景

- **充值/入金监听**：交易所、钱包、OTC 平台实时感知用户 USDT/TRX 充值到账。
- **支付网关**：电商、游戏、内容平台接入 TRC20 收款，回调驱动订单状态流转。
- **提现/归集监控**：监听热钱包转出，自动化记账、对账与风控。
- **资金安全告警**：监控大额进出、异常转账，第一时间触发风控策略。

## 工作原理

```text
您的服务器                     usdtpay 服务                        TRON 链
    │                              │                                │
    │  1. POST /setaddress ──────> │  提交要监听的地址 + 回调地址      │
    │                              │                                │
    │                              │  2. 持续扫块（约 3 秒一轮）──────>│
    │                              │<───────────────── 区块数据       │
    │                              │  3. 命中订阅事件                 │
    │  <──── 4. Webhook POST ──────│                                │
    │  5. 验签、处理业务            │                                │
```

## 快速开始

1. **开通账号**：联系 [Telegram](https://t.me/secp256k0) 开通服务，我们会为您分配三个凭证：
   - `apiId` —— 接入方 ID（整数）
   - `apiKey` —— 调用密钥（放在请求头 `X-Api-Key`）
   - `apiSecretToken` —— 签名密钥（只用于本地计算签名，**永远不要放进任何网络请求**）
2. **提交监听地址**：调用 `POST /api/v2/listeners/setaddress`，告诉我们要监听哪个地址、关心哪些事件、回调推送到哪里。
3. **实现回调端点**：在您的服务器上提供一个公网可访问的 POST 接口，接收并处理我们推送的事件。
4. **验证签名（推荐）**：用 `apiSecretToken` 校验回调头 `X-Api-Sign`，确认请求来自 usdtpay（示例见下文）。

下面是一个完整的最小示例（Node.js）：

```js
const crypto = require("node:crypto");

const API_ID = "<您的 apiId>";
const API_KEY = "<您的 apiKey>";
const API_SECRET_TOKEN = "<您的 apiSecretToken>";
const BASE_URL = "<开通服务后由我们提供的接口地址>";

// 1. 构造请求体
const body = JSON.stringify({
  addressBase58: "TYourTronAddressBase58...",
  sendTo: "https://your-server.com/webhook/tron",
  eventTypes: ["TRX_IN", "USDT_TRANSFER_IN"],
});

// 2. 计算签名：HMAC-SHA256(apiSecretToken, 请求体原始字节) -> 小写 hex
const sign = crypto.createHmac("sha256", API_SECRET_TOKEN).update(body).digest("hex");

// 3. 提交监听地址
const res = await fetch(`${BASE_URL}/api/v2/listeners/setaddress`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-Api-ID": API_ID,
    "X-Api-Key": API_KEY,
    "X-Api-Sign": sign,
  },
  body,
});
console.log(await res.json());
// => { "status": "ok", "id": 123 }
```

## 鉴权

### Base URL

**开通服务后由我们告知**。为保障服务安全，接口地址不在公开文档中展示；您完成开通后，我们会随
`apiId` / `apiKey` / `apiSecretToken` 一起提供正式的调用地址，本文档中所有接口均基于该地址。

### 请求头

| 请求头 | 必填 | 说明 |
| ------ | ---- | ---- |
| `Content-Type` | 是 | `application/json` |
| `X-Api-ID` | 是 | 我们分配给您的接入方 ID（整数），用于识别调用方 |
| `X-Api-Key` | 是 | 我们分配给您的调用密钥，缺失或错误一律返回 HTTP 401 |
| `X-Api-Sign` | 按配置 | 消息签名，见下方"签名算法"。默认账号不强制校验，可传空或不传；如已为您开启"真实签名校验"，则必须填写 |

其余业务参数（地址、回调地址、事件类型等）一律放在请求体 JSON 里。

### 签名算法

```text
sign = 小写 hex( HMAC-SHA256( apiSecretToken, 请求体原始字节 ) )
```

- **POST 请求**：对**完整请求体原始字节**签名——签名用的字节必须与实际发送的请求体字节完全一致，不要"为了签名序列化一次、实际发送又序列化一次"，字节稍有出入签名就会对不上。
- **无请求体的请求**（如 `GET /info`）：对**空字节**签名。
- 默认情况下（`requireSign=false`）服务端不校验 `X-Api-Sign`，鉴权依靠 `X-Api-ID` + `X-Api-Key`；为了安全，建议申请开通时要求开启真实签名校验。

Node.js：

```js
const crypto = require("node:crypto");

function computeSignature(apiSecretToken, bodyBytes) {
  return crypto.createHmac("sha256", apiSecretToken).update(bodyBytes).digest("hex");
}

const bodyBytes = Buffer.from(JSON.stringify(payload), "utf8");
const sign = computeSignature(API_SECRET_TOKEN, bodyBytes);
```

Python：

```python
import hashlib
import hmac

sign = hmac.new(
    API_SECRET_TOKEN.encode("utf-8"),
    body_bytes,  # 实际发送的请求体原始字节
    hashlib.sha256,
).hexdigest()
```

cURL + openssl：

```bash
# 先设置环境变量：API_ID / API_KEY / API_SECRET_TOKEN / BASE_URL
BODY='{"addressBase58":"TYourTronAddressBase58...","sendTo":"https://your-server.com/webhook/tron","eventTypes":["TRX_IN"]}'
SIGN=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET_TOKEN" -hex | awk '{print $2}')

curl -X POST "$BASE_URL/api/v2/listeners/setaddress" \
  -H "Content-Type: application/json" \
  -H "X-Api-ID: $API_ID" \
  -H "X-Api-Key: $API_KEY" \
  -H "X-Api-Sign: $SIGN" \
  -d "$BODY"
```

## 监听事件类型

提交订阅时用 `eventTypes` 字段（字符串数组）指定关心哪些事件：

| 标签 | 含义 | 状态 |
| ---- | ---- | ---- |
| `TRX_IN` | 收到 TRX 原生转账 | ✅ 已支持 |
| `TRX_OUT` | 转出 TRX 原生转账 | ✅ 已支持 |
| `USDT_TRANSFER_IN` | 收到 USDT(TRC20) 转账 | ✅ 已支持 |
| `USDT_TRANSFER_OUT` | 转出 USDT(TRC20) 转账 | ✅ 已支持 |
| `USDT_BLACK_LIST` | USDT 冻结/黑名单事件 | 预留，暂未生效 |
| `RESOURCE_DELEGATE` | 资源代理事件 | 预留，暂未生效 |
| `RESOURCE_UNDELEGATE` | 资源回收事件 | 预留，暂未生效 |

> **注意**：`eventTypes` 每次提交都是**完整覆盖**——本次传什么，这条订阅最终监听的事件就是什么，不是"追加"。想保留原有事件，请把完整的目标列表都传上。

## API 参考（v2）

### 通用响应约定

- 成功：HTTP 200，响应体至少包含 `"status": "ok"`。
- 参数错误：HTTP 400，响应体 `{ "error": "错误描述" }`。
- 服务端异常：HTTP 500，响应体 `{ "error": "错误描述" }`。

### 多租户订阅标识

一条订阅由以下五元组唯一定位：`apiId`（来自请求头）+ `addressBase58` + `robotId` + `userId` + `functionId`。

`robotId` / `userId` / `functionId` 是您自定义的业务参数（整数），回调推送时会在请求头**原样带回**，方便您一个回调端点按业务线分发。不使用就传 `0` 或省略。

同一个地址、以完全相同的五元组再次提交，视为**刷新这条订阅**（更新回调地址、事件类型等），不会产生重复记录。

---

### POST /api/v2/listeners/setaddress

新增或刷新一条地址订阅。提交前会检查您名下活跃订阅数是否已达绑定上限，达到上限会被拒绝。

**请求体：**

```json
{
  "addressBase58": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "robotId": 0,
  "userId": 0,
  "functionId": 0,
  "sendType": 1,
  "sendTo": "https://your-server.com/webhook/tron",
  "eventTypes": ["TRX_IN", "TRX_OUT", "USDT_TRANSFER_IN", "USDT_TRANSFER_OUT"],
  "energy": 0,
  "balance": 0,
  "remark": "交易所充值监控"
}
```

| 字段 | 类型 | 必填 | 默认值 | 说明 |
| ---- | ---- | ---- | ------ | ---- |
| `addressBase58` | string | 是 | - | 要监听的 TRON 地址（T 开头），格式或校验和不正确会报错 |
| `robotId` | number | 否 | 0 | 自定义参数，回调时原样带回 |
| `userId` | number | 否 | 0 | 自定义参数，回调时原样带回 |
| `functionId` | number | 否 | 0 | 自定义参数，回调时原样带回 |
| `sendType` | number | 否 | 1 | 推送方式：`1` = HTTP 回调（目前唯一开放） |
| `sendTo` | string | 是 | - | 推送目标地址（HTTP URL），不能为空 |
| `eventTypes` | string[] | 否 | [] | 关心的事件类型，见上文 |
| `energy` | number | 否 | 0 | 0/1，回调是否附带能量信息（预留字段） |
| `balance` | number | 否 | 0 | 0/1，回调是否附带余额信息（预留字段） |
| `remark` | string | 否 | "" | 备注，仅供您自己识别 |

**成功响应：**

```json
{ "status": "ok", "id": 123 }
```

`id` 是这条订阅在服务端数据库里的行 ID。

---

### POST /api/v2/listeners/setmultiaddress

批量提交，一次提交多个地址，共用同一个 `X-Api-ID`。

**请求体：**

```json
{
  "data": [
    { "addressBase58": "T...", "eventTypes": ["TRX_IN"] },
    { "addressBase58": "T...", "eventTypes": ["USDT_TRANSFER_IN", "USDT_TRANSFER_OUT"] }
  ]
}
```

`data` 数组中每一项的字段规则与 `setaddress` 完全一致。

**成功响应：**

```json
{
  "status": "ok",
  "results": [
    { "addressBase58": "T...", "ok": true },
    { "addressBase58": "T...", "ok": false, "error": "invalid checksum for TRON address: T..." }
  ]
}
```

批量提交中每条地址**独立处理**，某一条格式错误只影响这一条，不影响同批其它地址。

---

### POST /api/v2/listeners/removeaddress

移除一条订阅（软删除）。

**请求体：**

```json
{ "addressBase58": "T...", "robotId": 0, "userId": 0, "functionId": 0 }
```

`robotId` / `userId` / `functionId` 需要与提交订阅时使用的值完全一致才能准确定位（不传按 0 处理）。

**成功响应：**

```json
{ "status": "ok", "removed": true }
```

`removed` 为 `false` 表示没有找到匹配的订阅。

---

### POST /api/v2/listeners/removealladdress

按 `X-Api-ID`（可选再按 `functionId`）批量移除。

**请求体：**

```json
{ "functionId": 0 }
```

`functionId` 省略或传 `0` 表示移除您名下**全部**订阅；传具体值只移除该 `functionId` 下的订阅。

**成功响应：**

```json
{ "status": "ok", "removed": 37 }
```

`removed` 为实际移除的订阅条数。

---

### POST /api/v2/listeners/updatealladdresscbinfo

批量修改您名下（或某个 `functionId` 下）所有订阅的推送回调信息，不影响已配置的事件类型等其它设置。

**请求体：**

```json
{ "functionId": 0, "robotId": 0, "sendType": 1, "sendTo": "https://new-callback.example.com" }
```

| 字段 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `functionId` | number | 否 | 省略或 0 表示全部订阅生效，传具体值只更新该 `functionId` 下的 |
| `robotId` | number | 否 | 省略或 0 表示**不修改**原有 robotId（三态语义，不是"改成 0"） |
| `sendType` | number | 否 | 推送方式，省略默认按 1（HTTP）写入 |
| `sendTo` | string | 是 | 新的推送目标地址 |

**成功响应：**

```json
{ "status": "ok", "updated": 12 }
```

`updated` 为实际更新的订阅条数。

---

### GET /api/v2/listeners/info

查询您名下当前的订阅统计与配额。无请求体，签名按"空字节"计算（如开启了真实签名校验）。

**成功响应：**

```json
{ "apiId": 732390, "addressBindCount": 20000, "addressBindLimit": 1000000, "expireDate": null }
```

| 字段 | 说明 |
| ---- | ---- |
| `addressBindCount` | 您名下当前生效的订阅数量（真实值） |
| `addressBindLimit` | 我们为您分配的绑定上限，达到后 `setaddress` 会被拒绝，提额请联系我们 |
| `expireDate` | 接口到期时间（毫秒时间戳），`null` 表示长期有效 |

---

## Webhook 回调

您监听的地址上发生匹配的事件后，我们会向您提交的 `sendTo` 发起**一次** `POST` 请求，请求体为 `application/json`。

> **只发一次，失败不重试**：请确保您的回调端点稳定可用、正确返回 HTTP 200。网络异常或非 200 响应只记录服务端日志，不会自动重发。

### 请求头

| 请求头 | 说明 |
| ------ | ---- |
| `Content-Type` | `application/json` |
| `X-Api-ID` | 命中订阅所属的接入方 ID |
| `X-Robot-ID` | 您提交订阅时填写的 `robotId`，原样带回 |
| `X-User-ID` | 您提交订阅时填写的 `userId`，原样带回 |
| `X-Function-ID` | 您提交订阅时填写的 `functionId`，原样带回 |
| `X-Api-Sign` | 对**回调 body 原始字节**的 HMAC-SHA256 小写 hex——始终是真实签名，建议您在接收端验证，确认请求确实来自 usdtpay |

### 请求体字段

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| `event` | string | `TransferContract`（TRX 原生转账）/ `TriggerSmartContract`（合约调用，目前仅 USDT） |
| `nativeTransfer` | number | `1` 表示这笔是 TRX 原生转账，否则 `0` |
| `trc20Transfer` | number | `1` 表示这笔是 USDT(TRC20) 转账，否则 `0` |
| `trc20Token` | string | TRX 固定 `"_"`；USDT 固定 `"TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t"`（USDT 合约地址） |
| `methodId` | string | TRX 固定 `""`；USDT 固定 `"a9059cbb"` |
| `methodName` | string | TRX 固定 `""`；USDT 固定 `"transfer"` |
| `resourceContract` | number | 固定 `0`（资源代理/回收推送尚未开放） |
| `resourceType` | string | 固定 `""`（同上） |
| `direction` | number | `1` = 监听地址是收款方（收到转账）；`2` = 监听地址是付款方（发起转账） |
| `from` / `to` | string | 转出方 / 转入方地址（Base58） |
| `fromHex` / `toHex` | string | 转出方 / 转入方地址（Hex，`41` 前缀） |
| `txid` | string | 交易哈希 |
| `value` | number | **裸数字（不带引号）**，链上原始最小单位整数：TRX 单位是 SUN（1 TRX = 1,000,000 SUN），USDT 是 6 位小数整数。需要您按代币精度自行换算 |
| `timeStamp` | number | 交易时间戳（毫秒） |
| `net` ~ `trxBalanceForAcquiredBandwidth` | number | 资源/余额相关字段，**目前固定为 `-1`**（尚未开放） |
| `block_id` | string | 交易所在区块哈希 |
| `block_num` | number | 交易所在区块高度 |
| `block_ts` | number | 区块产生时间戳（毫秒），与 `timeStamp` 数值相同 |
| `cntTotal` / `cntTransfer` | number | 该区块总交易数 / 该区块内转账类交易数，仅供参考 |

### 完整示例（TRX 转账）

```json
{
  "event": "TransferContract", "nativeTransfer": 1, "trc20Transfer": 0,
  "trc20Token": "_", "methodId": "", "methodName": "",
  "resourceContract": 0, "resourceType": "",
  "direction": 1,
  "from": "TDrbCtdTj5KMo67mtQw2XXu5eTqwwVYoKz", "to": "TNNGE51GrA8ryvAXRT2nxXyDJEBwofoLjZ",
  "fromHex": "412aa02059a547f746c7e488a226620f098df40d4c", "toHex": "4187fdc49f6d1f051be970fefe00a8d80b1954dd96",
  "txid": "7968fe012653ea6b3c3a52f5825475c85320cb1af96fd517d8568aa075858339",
  "value": 11700000, "timeStamp": 1784911773000,
  "net": -1, "energy": -1, "trx": -1, "usdt": -1, "netUsage": -1, "energyUsage": -1,
  "totalNet": -1, "totalEnergy": -1, "trxBalanceForBandwidth": -1, "trxBalanceForEnergy": -1,
  "trxBalanceForAcquiredEnergy": -1, "trxBalanceForAcquiredBandwidth": -1,
  "block_id": "00000000050d23468b4a0f49b0613876f5f6320f7fe8e1debfda0c99a6fb091f",
  "block_num": 84747078, "block_ts": 1784911773000, "cntTotal": 371, "cntTransfer": 242
}
```

上例 `value` 为 11,700,000 SUN，即 **11.7 TRX**。USDT 转账时 `event` = `"TriggerSmartContract"`、`nativeTransfer` = `0`、`trc20Transfer` = `1`，其余字段含义一致。

### 回调验签示例（Node.js）

```js
const crypto = require("node:crypto");

// 在您的回调处理函数里，rawBody 必须是收到的原始请求体字节（不要格式化）
function verifyWebhook(rawBody, signHeader, apiSecretToken) {
  const expected = crypto.createHmac("sha256", apiSecretToken).update(rawBody).digest("hex");
  const a = Buffer.from(expected, "utf8");
  const b = Buffer.from(signHeader.toLowerCase(), "utf8");
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}

// 伪代码示意
const ok = verifyWebhook(req.rawBody, req.headers["x-api-sign"], API_SECRET_TOKEN);
if (!ok) return res.status(401).end("signature mismatch");
// 签名通过后再处理业务，例如：按 txid 做幂等、更新订单状态
```

### 安全建议

- **验签**：所有回调都建议先验证 `X-Api-Sign` 再处理业务。
- **幂等**：以 `txid` + `direction`（+ 您自己的 `robotId`/`functionId`）作为唯一键做去重，防止重复处理。
- **密钥保管**：`apiKey` / `apiSecretToken` 只放在您的服务端，不要出现在前端代码、日志或任何外部请求里；`apiSecretToken` 永远不需要随请求发送。

---

## 兼容旧版 SDK（v1）

为了兼容既有 `TronListenerSDK` 客户端，服务端保留了 `/api/v1/listeners/*` 路径，存量系统可零改造接入：

`POST /api/v1/listeners/setaddress` · `POST /api/v1/listeners/removeaddress` · `POST /api/v1/listeners/setmultiaddress` · `POST /api/v1/listeners/removealladdress` · `POST /api/v1/listeners/updatealladdresscbinfo` · `GET /api/v1/listeners/info`

与 v2 的主要区别：

| 差异点 | v2（推荐） | v1（兼容层） |
| ------ | ---------- | ------------ |
| 业务参数位置 | 请求体，camelCase | `robotId`/`userId`/`functionId` 走请求头，body 是 snake_case |
| 事件类型 | `eventTypes` 字符串数组，全量覆盖 | `trx_in`/`trx_out`/`usdt_in`/`usdt_out`/`resource_contract` 三态整数（`1`=开，`0`=关，`-1`=不变），增量合并 |

**新接入方一律使用 v2**。

---

## 错误码与排障

### HTTP 状态码

| 状态码 | 含义 |
| ------ | ---- |
| 200 | 成功，按各接口返回响应体 |
| 400 | 请求参数错误（缺少必填字段、地址格式不对等），响应体 `{ "error": "..." }` |
| 401 | 鉴权失败——`apiId` 不存在 / `X-Api-Key` 错误 / 接口已到期 / 签名校验失败，统一返回，不区分具体原因 |
| 404 | 接口路径不存在 |
| 500 | 服务端内部异常，响应体 `{ "error": "..." }` |

### 常见问题（FAQ）

**多久能收到回调？**
服务端约 3 秒一轮扫块，加上出块与网络时间，通常交易确认后数秒内送达。

**收不到回调怎么办？**
按顺序排查：① 回调地址 `sendTo` 公网可达且 POST 返回 200；② 订阅的事件类型是否包含该笔交易的方向（注意 `eventTypes` 是覆盖不是追加）；③ 地址是否为 Base58 格式；④ 注意服务**只发一次、不重试**，请保证端点稳定。

**`value` 字段怎么换算成金额？**
`value` 是链上最小单位的整数：TRX 为 SUN（1 TRX = 1,000,000 SUN），USDT(TRC20) 为 6 位小数整数（例如 `3000000` = 3 USDT）。展示时请按币种精度自行换算，**不要**按字符串或浮点直接除以精度后再四舍五入，建议用十进制字符串运算。

**怎么确认回调真的来自 usdtpay？**
用您的 `apiSecretToken` 对回调**原始 body 字节**计算 HMAC-SHA256，与请求头 `X-Api-Sign` 做恒定时间比较。验签通过即可确认来源。

**支持 USDC 或其它 TRC20 代币吗？**
目前内置支持 TRX 原生转账与 USDT(TRC20)。其它 TRC20 代币（如 USDC）可按需扩展，请联系我们评估接入。

**一个地址可以同时被多个业务监听吗？**
可以。`(apiId, robotId, userId, functionId)` 任一组合不同，就是一条独立订阅，各自推送、互不影响。

**`energy` / `balance` 字段为什么是 -1？**
能量/余额附加信息功能尚未开放，回调中相关字段固定为 `-1`（表示"未查询"），订阅时无需关心。

**`expireDate` 为 null 是什么含义？**
表示您的接口长期有效，无到期时间；非 null 时为毫秒时间戳，到期后请求会返回 401。

---

## 联系我们

开通服务、获取接入凭证、提额、定制代币或功能，请联系：

**Telegram：[https://t.me/secp256k0](https://t.me/secp256k0)**
