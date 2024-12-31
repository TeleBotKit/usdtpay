# usdtpay
TRC20 USDT/USDC payment gateway, 数字货币 TRC20 USDT/TRX 充值支付回调网关    
USDT,收款,支付,TRC20,trx,充值,提现,API支付接口,监控,回调,USDT TRC20    
开通服务联系 https://t.me/secp256k0


# 签名算法
sign = 大写HEX( hmac.Sha1( POST包体, `${{APISecretToken}}` ) )  
示例：C27FBE1830EC3A13E5D308950DF28E80D0274B09  

# 添加监听地址

## 接口地址
`http://[网关地址]/api/v1/listeners/setaddress`

---

## 请求方式
POST

---

## 请求头
| 参数名        | 类型   | 是否必填 | 说明                                                          |
| ------------- | ------ | -------- | ------------------------------------------------------------- |
| Content-Type  | string | 是       | `application/json`                                            |
| X-Api-ID      | int64  | 是       | 接口鉴权Id`${{APIId}}`                                        |
| X-Api-Token   | string | 是       | 接口鉴权密钥`${{APIKey}}`                                     |
| X-Api-Sign    | string | 是       | 见签名算法                                                    |
| X-Function-ID | int64  | 是       | 可填任意int64数值，与监听地址绑定，通知回调时在协议头原样返回 |
| X-Robot-ID    | int64  | 是       | 可填任意int64数值，与监听地址绑定，通知回调时在协议头原样返回 |
| X-User-ID     | int64  | 是       | 可填任意int64数值，与监听地址绑定，通知回调时在协议头原样返回 |

---

## 请求参数

### JSON 请求包
```json
{
  "address_base58": "TAVDdkxfqVRtYhwqnkwFKFJRf3gM5PrG1T", // 要监控的地址
  "send_type": 1, // 固定1
  "send_to": "http://127.0.0.1:8445/webhook/event/wallet/trans?address=TCM1mEMRm2nWwBwnP2hCt61qwA7c5RfRCD&t=1724238752", // 通知地址
  "trx_in": -1, // 是否开启trx转入通知, 1=开，0=关，-1=无变化
  "trx_out": -1, // 是否开启trx转出通知, 1=开，0=关，-1=无变化
  "usdt_in": -1, // 是否开启usdt转入通知, 1=开，0=关，-1=无变化
  "usdt_out": -1, // 是否开启usdt转出通知, 1=开，0=关，-1=无变化
  "energy": -1, // 通知时附带能量/带宽信息, 1=开，0=关，-1=无变化
  "balance": -1, // 通知时附带余额信息, 1=开，0=关，-1=无变化
  "resource_contract": -1 // 是否开启资源代理/回收通知, 1=开，0=关，-1=无变化
}
```

### 成功响应
 HTTP 200

# 移除监听地址

## 接口地址
`http://[网关地址]/api/v1/listeners/removeaddress`

---

## 请求方式
POST

---

## 请求头
| 参数名        | 类型   | 是否必填 | 说明                                             |
| ------------- | ------ | -------- | ------------------------------------------------ |
| Content-Type  | string | 是       | `application/json`                               |
| X-Api-ID      | int64  | 是       | 接口鉴权Id`${{APIId}}`                           |
| X-Api-Token   | string | 是       | 接口鉴权密钥`${{APIKey}}`                        |
| X-Api-Sign    | string | 是       | 见签名算法                                       |
| X-Function-ID | int64  | 是       | 设置该地址时填了什么值，删除地址时也要填一样的值 |
| X-Robot-ID    | int64  | 是       | 设置该地址时填了什么值，删除地址时也要填一样的值 |
| X-User-ID     | int64  | 是       | 设置该地址时填了什么值，删除地址时也要填一样的值 |

---

## 请求参数

### JSON 请求包
```json
{
  "address_base58": "TAVDdkxfqVRtYhwqnkwFKFJRf3gM5PrG1T" // 要移除的地址
}
```

### 成功响应
 HTTP 200

# 回调事件定义如下: 
```json
    {
    // 事件类型 
    // TransferContract = 本链交易(大多为TRX转账)
    // TriggerSmartContract = 智能合约(USDT转账属于智能合约)
    // UnDelegateResourceContract = 资源回收(能量带宽回收)
    // DelegateResourceContract = 资源代理
    // AccountCreateContract = 激活账户(链上有新账户激活)
    "event": "TransferContract", 
    "nativeTransfer": 1, // 是否TRX转账
    "trc20Transfer": 0, // 是否TRC20交易
    "trc20Token": "_", // TRC20交易的合约地址，如果是USDT转账则为 TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t
    "methodId": "", // TRC20交易执行的方法Id，如果是USDT转账则为 a9059cbb
    "methodName": "", // TRC20交易执行的方法名称，如果是USDT转账则为 transfer
    "resourceContract": 0, // 是否为资源代理/回收事件
    "resourceType": "", // 资源类型
    "direction": 1, // 交易方向，1=接收，2=发起
    "from": "TSUUVjysXV8YqHytSNjfkNXnnB49QDvZpx", // 来源地址
    "to": "TQooBX9o8iSSprLWW96YShBogx7Uwisuim", // 目标地址
    "fromHex": "41a2c2426d23bb43809e6eba1311afddde8d45f5d8", // hex格式来源地址
    "toHex": "4105ad01ce0c880eebf776ce3e9c0bb190e38b869a", //hex格式目标地址
    "txid": "79d34459036512807440210271d40135145c4f9f38454a24db0b2129368de9de", // 交易Id
    "value": 3000000, // 交易金额，uint64，单位为SUN
    "timeStamp": 1733921449650, // 交易时间
    "net": -1,
    "energy": -1,
    "trx": 911272158, // 被监控地址的TRX余额
    "usdt": 1524650000, // 被监控地址的USDT余额
    "netUsage": -1, 
    "energyUsage": -1, 
    "totalNet": -1, 
    "totalEnergy": -1, 
    "trxBalanceForBandwidth": -1,
    "trxBalanceForEnergy": -1,
    "trxBalanceForAcquiredEnergy": -1,
    "trxBalanceForAcquiredBandwidth": -1,
    "block_id": "000000000409e3c92d1913533f722f814cab3c00adab798166af78b7192dab32", // 交易所在区块Id
    "block_num": 67757001, // 交易所在区块高度
    "block_ts": 1733921457000, // 区块产生时间戳
    "cntTotal": 189, // 当前区块总交易数
    "cntTransfer": 150 // 当前区块总转账数
}
```
