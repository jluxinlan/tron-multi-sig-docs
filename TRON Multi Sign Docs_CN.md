# TRON多签服务钱包接入文档
### 一、钱包构造多签交易并提交到多签服务

（一）选择控制地址

用户可输入目标控制地址，或通过调用 /openapi/multi/auth
接口查询可控制地址列表，并从中选中拟操作的控制地址。该接口支持用户获取其有权限控制的地址信息，以便准确选定交易操作的源地址。​

（二）构造交易

用户根据具体业务需求，选择对应的交易类型（如转账、合约调用等），并按照FullNode标准交易的规范格式构造transaction对象。在构造过程中，需确保交易信息的完整性和准确性，包括但不限于交易金额、目标地址、操作类型等关键参数。​

（三）提交交易

通过调用 /openapi/multi/transaction
接口，将构造好的FullNode标准transaction
提交至多签服务。此接口用于接收用户构造的多签交易信息，为后续的签名和处理流程提供基础数据。

### 二、 待签名地址查询待签名交易，签名后提交到多签服务

（一）查询待签名交易

利用 WebSocket 接口 /openapi/multi/socket
建立实时连接，实时查询当前地址待签名的交易。该接口支持实时推送待签名交易信息，确保用户能够及时获取需要处理的交易任务，以便进行后续的签名操作。​

（二）签名并提交交易

用户使用对应的私钥对待签名交易进行数字签名。完成签名后，再次通过
/openapi/multi/transaction 接口提交更新后的FullNode标准transaction
对象。多签服务将自动验证签名权重，当签名权重达到预设的阈值时，系统会自动将交易广播至区块链网络。交易广播后，用户可通过交易
hash 在区块链上查询交易状态及详情，以便实时跟踪交易进度。

# 接口列表

### **所有接口都需要鉴权，鉴权逻辑参照 [[接口鉴权规范]](#接口鉴权规范)**

## 1、查询当前地址控制的关联地址和权限

### 接口名称

地址权限查询

### 接口路径

GET /openapi/multi/auth (非必要的，用户可以自己输入自己控制的地址)

### 请求参数

  ------------------------------------------------------------------------------------------------------
| **参数名**  | **类型**   | **是否必传** | **描述**         | **示例值**                            |
|:---------|:---------|:---------|:---------------|:-----------------------------------|
| address  | string   | 是        | 当前地址（查询其控制的地址） | TMf7fBmKPDGVP8b6UrEu1t6oDBRnNgwTt7 |

  ------------------------------------------------------------------------------------------------------

### 返回参数

  -------------------------------------------------------------------------
| **字段名**                  | **类型**        | **描述**               |
|:-------------------------|:--------------|:---------------------|
| code                     | int           | 状态码（0 = 成功，非 0 = 失败） |
| message                  | string        | 状态描述                 |
| data                     | array         | 权限列表                 |
| ├─ owner_address         | string        | 关联地址                 |
| ├─ owner_permission      | object        | 关联地址的owner权限         |
| │ ├─ operations          | string        | 允许的操作编码（空值表示全权限）     |
| │ ├─ threshold           | int           | 权限阈值（签名所需最小权重）       |      
| │ ├─ weight              | int           | 当前地址权重               |
| └─ active_permissions    | array         | 关联地址的active权限        |
  -------------------------------------------------------------------------

### 返回示例
```
{
    "code": 0,
    "message": "OK",
    "data": [
        {
            "owner_address": "TFDP1vFeSYPT6FUznL7zUjhg5X7p2AA8vw",
            "owner_permission": {
                "operations": "",
                "threshold": 3, //阈值
                "weight": 1     //当前权重
            },
            "active_permissions": [
                {
                    "operations": "1620008000000000000000000000000000000000000000000000000000000000",
                    "threshold": 8, //阈值
                    "weight": 1     //当前权重
                },
                {
                    "operations": "1018000000000000000000000000000000000000000000000000000000000000",
                    "threshold": 5, //阈值
                    "weight": 2     //当前权重
                }
            ]
        }
    ]
}

```

## 2、构造并提交多签交易

### 接口名称

多签交易提交

### 接口路径

POST /openapi/multi/transaction

### 请求体示例

```
{
    "address": "TE4CeJSjLmBsXQva3F1HXvAbdAP71Q2Ucw",
    "function_selector":"transfer(address,uint256)";  //交易是触发智能合约时，这里是真实的智能合约的方法。方便解析parameter的数据
    "transaction": {    // 构造的要上链的真实交易体。
        "raw_data": {
            "ref_block_bytes": "ded4",
            "ref_block_num": null,
            "ref_block_hash": "1bb8282d1cf51fb2",
            "expiration": 1766034581308,
            "auths": null,
            "data": "",
            "contract": [
                {
                    "type": "TransferContract",
                    "parameter": {
                        "value": {
                            "amount": 12000000,
                            "owner_address": "412a60357d1648251fca11576bdfea19a62ce1b45e",
                            "to_address": "417e9696f656dc848478782a429c5ad421d93dde88"
                        },
                        "type_url": "type.googleapis.com/protocol.TransferContract"
                    },
                    "provider": null,
                    "ContractName": null,
                    "Permission_id": 8
                }
            ],
            "scripts": "",
            "timestamp": 1765948176000,
            "fee_limit": null
        },
        "signature": ["659143f51bea6f0b16ce1e5f98a662cf086eb033ce9a17fb204cdbdfa34ba75448af68e3ba746eddd53b552e70e5dbd4273b6bd649ae493361dddb28ad72b53800"
        ]
    }
}

```

### 关键字段说明

  ---------------------------------------------------------------------------
| **字段名称**              | **数据类型** | **描述说明**             |
|:----------------------|:---------|:---------------------|
| address               | string   | 发起交易的控制地址            |
| function_selector     | string   | 智能合约方法（触发智能合约时必填）    |
| transaction.raw_data  | object   | 符合区块链底层协议的原始交易数据     |
| transaction.signature | array    | 签名信息数组（签名信息依次追加）     |
---------------------------------------------------------------------------

### 

## 3、待签名交易监听（WebSocket）

### 接口名称

待签名交易实时监听接口

### 接口路径

GET /openapi/multi/socket

### 协议类型

WebSocket

### 链接流程：

1.  鉴权校验：客户端在 HTTP 请求 Header
    中携带有效鉴权参数（具体格式由服务端统一定义，参考接口鉴权部分）​

2.  建立连接：服务端校验通过后，客户端发送当前操作地址信息以订阅相关交易​

3.  数据交互：服务端主动推送待签名交易详情，客户端根据业务逻辑进行签名处理；也会推送自己相关多签交易的状态变动，前端根据数据状态判断当前交易是不是需要签名。

### 客户端请求示例

```
{
    "address": "TW6omSrQ1ZK37SwSvTQD5Cnp2QbEX2zDVZ"，// 订阅待签名交易； 订阅交易状态更新 
    "version":"v1"
}
```

## 服务端响应示例

```
[
    {
        "hash": "18213ab5b1d277b4090f647b925952efe972facd19462101f3d94a58b8354c23",
        "contract_type": "TransferContract",
        "originator_address": "TQUsaH7DzTAPQEVsUvQsVyzvwqwT2p7WEm",
        "expire_time": 0,
        "threshold": 3,  // 阈值
        "current_weight": 2, // 当前权重
        "is_sign": 1,  // 当前地址是否已签名
        "signature_progress": [  // 都有哪些地址来签名
            {
                "address": "TW6omSrQ1ZK37SwSvTQD5Cnp2QbEX2zDVZ",
                "weight": 1,  // 权重
                "is_sign": 0,  // 是否签名
                "sign_time": 0 // 签名时间
            },
            {
                "address": "TQUsaH7DzTAPQEVsUvQsVyzvwqwT2p7WEm",
                "weight": 1,
                "is_sign": 1,
                "sign_time": 1741858044
            },
            {
                "address": "TFdACej5gjKqSmwNNESzAbfTmBBCx55G4G",
                "weight": 1,
                "is_sign": 1,
                "sign_time": 1741858044
            }
        ],
        "contract_data": { // 解析出来的parameter参数
            "amount": 1000000,
            "to_address": "TMf7fBmKPDGVP8b6UrEu1t6oDBRnNgwTt7",
            "owner_address": "TQUsaH7DzTAPQEVsUvQsVyzvwqwT2p7WEm"
        },
        "current_transaction": {
            "raw_data": {
                "ref_block_bytes": "3e96",
                "ref_block_num": null,
                "ref_block_hash": "6c2afde05160d139",
                "expiration": 1741944318000,
                "auths": null,
                "data": "",
                "contract": [
                    {
                        "type": "TransferContract",
                        "parameter": {
                            "value": "0a15419f2e05d49b5fe66dce55598984aace7b3dc45fb012154180358ff232c17134b914a71b346a647dad006dfe18c0843d",
                            "type_url": "type.googleapis.com/protocol.TransferContract"
                        },
                        "provider": null,
                        "ContractName": null,
                        "Permission_id": 3
                    }
                ],
                "scripts": "",
                "timestamp": 1741857918000,
                "fee_limit": null
            },
            "signature": ["3a53f8f5e4ed22a49a32e797d8ec9ed9dee4cd2dba8f00ee882a51bfd6691d94113a3f886f992803c191e38f59973f2b6521a7bea5235ee57eb86e3b757b4d9c1B","0e586c656a95450de017c67da4b78e8639e2537873b8b8ed6a3b39bce875724f548ac0e0f3e5fe4bd6bc395b10672c51f12598bc0835ac8f673c54c8b3e4ad0f1B"],
            "raw_data_hex": "0a023e9622086c2afde05160d13940b0d0e39fd9325a69080112630a2d747970652e676f6f676c65617069732e636f6d2f70726f746f636f6c2e5472616e73666572436f6e747261637412320a15419f2e05d49b5fe66dce55598984aace7b3dc45fb012154180358ff232c17134b914a71b346a647dad006dfe18c0843d280370b098caf6d832"
        },
        "state": 1,
        "function_selector": "transfer(address,uint256)"
    }
]

```

## 4、交易列表查询

### 接口名称

多签交易历史记录查询

### 接口路径

GET /openapi/multi/list

### 请求参数

  ---------------------------------------------------------------------------------------------------------------
| **参数名称**     | **数据类型**  | **是否必填** | **描述说明**                                 |
|:-------------|:----------|:---------|:-----------------------------------------|
| address      | string    | 是        | 当前地址                                     |
| start        | int       | 是        | 分页查询，起始索引(如果limit=10,那第二页start=10)       |
| limit        | int       | 是        | 分页数据量（最大值100）                            |
| is_sign      | boolean   | 否        | 是否只查看自己的签名数据（true-已签名；false-未签名，默认false） |
| state        | int       | 是        | 交易状态过滤（0-处理中；1-成功；2-失败；255-全部）           |
---------------------------------------------------------------------------------------------------------------

### 返回参数

  ------------------------------------------------------------------------------
|**字段名称**       | **数据类型** | **描述说明**  |
|:--------------- |:---------|:----------|
| code            | int      | 状态码       |
| message         | string   | 状态描述信息    |
| data.total      | int      | 符合条件的记录总数 |
| data.range_total   | int   | 当前分页范围内的总数 |
| data.data          | array | 交易详情数组（结构同WebSocket推送数据）|
------------------------------------------------------------------------------

## 返回示例

```
{
    "code": 0,
    "message": "OK",
    "data": {
        "total": 1,
        "range_total": 14,
        "data": [
            {
                "hash": "18213ab5b1d277b4090f647b925952efe972facd19462101f3d94a58b8354c23",
                "contract_type": "TransferContract",
                "originator_address": "TQUsaH7DzTAPQEVsUvQsVyzvwqwT2p7WEm",
                "expire_time": 0,
                "threshold": 2,  
                "current_weight": 2, 
                "is_sign": 1,  
                "signature_progress": [  
                    {
                        "address": "TW6omSrQ1ZK37SwSvTQD5Cnp2QbEX2zDVZ",
                        "weight": 1,  
                        "is_sign": 0,  
                        "sign_time": 0 
                    },
                    {
                        "address": "TQUsaH7DzTAPQEVsUvQsVyzvwqwT2p7WEm",
                        "weight": 1,
                        "is_sign": 1,
                        "sign_time": 1741858044
                    },
                    {
                        "address": "TFdACej5gjKqSmwNNESzAbfTmBBCx55G4G",
                        "weight": 1,
                        "is_sign": 1,
                        "sign_time": 1741858044
                    }
                ],
                "contract_data": { // 解析出来的parameter参数
                    "amount": 1000000,
                    "to_address": "TMf7fBmKPDGVP8b6UrEu1t6oDBRnNgwTt7",
                    "owner_address": "TQUsaH7DzTAPQEVsUvQsVyzvwqwT2p7WEm"
                },
                "current_transaction": {
                    "raw_data": {
                        "ref_block_bytes": "3e96",
                        "ref_block_num": null,
                        "ref_block_hash": "6c2afde05160d139",
                        "expiration": 1741944318000,
                        "auths": null,
                        "data": "",
                        "contract": [
                            {
                                "type": "TransferContract",
                                "parameter": {
                                    "value": "0a15419f2e05d49b5fe66dce55598984aace7b3dc45fb012154180358ff232c17134b914a71b346a647dad006dfe18c0843d",
                                    "type_url": "type.googleapis.com/protocol.TransferContract"
                                },
                                "provider": null,
                                "ContractName": null,
                                "Permission_id": 3
                            }
                        ],
                        "scripts": "",
                        "timestamp": 1741857918000,
                        "fee_limit": null
                    },
                    "signature": ["3a53f8f5e4ed22a49a32e797d8ec9ed9dee4cd2dba8f00ee882a51bfd6691d94113a3f886f992803c191e38f59973f2b6521a7bea5235ee57eb86e3b757b4d9c1B","0e586c656a95450de017c67da4b78e8639e2537873b8b8ed6a3b39bce875724f548ac0e0f3e5fe4bd6bc395b10672c51f12598bc0835ac8f673c54c8b3e4ad0f1B"],
                    "raw_data_hex": "0a023e9622086c2afde05160d13940b0d0e39fd9325a69080112630a2d747970652e676f6f676c65617069732e636f6d2f70726f746f636f6c2e5472616e73666572436f6e747261637412320a15419f2e05d49b5fe66dce55598984aace7b3dc45fb012154180358ff232c17134b914a71b346a647dad006dfe18c0843d280370b098caf6d832"
                },
                "state": 1,
                "function_selector": "transfer(address,uint256)"
            }
        ]
    }
}

```

# **接口鉴权规范**

## 一、公共请求参数

[所有接口请求需包含以下公共请求头字段，用于身份验证、版本识别及请求追踪：]{.mark}

  ---------------------------------------------------------------------------
| **名称**        | **类型**        | **默认值**                                 |
|:--------------|:--------------|:----------------------------------------|
| sign_version  | string        | v1 目前v1版本接口                             |
| ts            | long          | 以毫秒为单位表示当前时间                            |
| address       | string        | tron链的58地址，表示当前接口请求的当前账户地址              |
| channel       | string        | 表示请求项目方名称, 申请时项目方自己的定的项目名称（eg:tronlink） |
| uuid          | string        | 表示当前请求唯一id,请求时随机一个                      |
| secret_id     | string        | 跟tronlink约定的项目唯一标识                      |
| sign          | string        | 接口签名，tronlink校验请求是否合法                   |
---------------------------------------------------------------------------

## 二、签名（sign）生成规则

### **2.1 签名参数排序**

将公共请求头参数（除 sign 外）按字段名 ASCII
码升序排列，拼接为key=value格式的字符串，参数间以&连接。

示例：

```
address=TMf7fBmKPDGVP8b6UrEu1t6oDBRnNgwTt7&channel=tronlink&secret_id=3d717E259617EA528F8&sign_version=v1&ts=174592188000&uuid=a6e4563f-1ce4-4a8f-ba37-de1cc121b4f8
```

### **2.2 构造签名原文字符串**

格式为：请求方法 + 请求路径 + ? + 参数拼接字符串

示例（GET 请求，WebSocket是GET请求）：

```
GET/api/wallet/v2/auth?address=TMf7fBmKPDGVP8b6UrEu1t6oDBRnNgwTt7&channel=tronlink&secret_id=3d717E259617EA528F8&&sign_version=v1&ts=174592188000&uuid=a6e4563f-1ce4-4a8f-ba37-de1cc121b4f8
```

### **2.3 生成签名值**

1.  使用 HmacSHA256
    算法，以项目方申请的secretKey作为密钥，对签名原文字符串进行加密。

2.  将加密结果进行 Base64 编码，得到最终的sign参数值。

## 三、密钥（secretId/secretKey）申请流程

### **3.1 申请方式**

Tronlink的官方运营人员会提供一个申请[google
doc链接](https://docs.google.com/forms/d/e/1FAIpQLSc5EB1X8JN7LA4SAVAG99VziXEY6Kv6JxmlBry9rUBlwI-GaQ/viewform?pli=1)，项目在google
doc上留下项目名称，项目信息以及联系用的邮箱地址。

### **3.2 响应内容**

申请通过后，将收到包含以下信息的回复邮件：

```
secretID: SSSSSSSSSSS （项目唯一标识）

secretKey: CCCCCCCCCCCCCCC （签名密钥，需妥善保管）
```

## 四、安全注意事项

1.  密钥保密：secretKey属于敏感信息，需严格控制访问权限，避免泄露。

2.  时间戳校验：服务端将校验请求时间戳ts，建议客户端与 NTP
    服务器同步时间，误差需控制在 5 分钟内。

3.  UUID 唯一性：每个请求需生成唯一的uuid，避免重复请求导致的业务异常。

4.  签名不可伪造：请确保签名算法实现与本文档完全一致，否则将导致鉴权失败。

如需技术支持或密钥重置，请联系官方团队。
