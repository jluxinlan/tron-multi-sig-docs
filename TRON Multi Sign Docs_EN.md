# TRON Multi-Signature Wallet Integration Documentation

### I. Constructing Multi-Signature Transactions in the Wallet and Submitting to Multi-Signature Service

#### (1) Selecting the Control Address
Users can either input the target control address directly or call the `/openapi/multi/auth` endpoint to query a list of controllable addresses, and then select the control address for the intended operation. This endpoint allows users to obtain address information they have permission to control, ensuring accurate selection of the source address for the transaction. ​

#### (2) Constructing the Transaction
Users should choose the transaction type according to specific business needs (e.g., transfer, contract call) and construct the `transaction` object in compliance with FullNode standard transaction formats. During construction, it is essential to ensure the completeness and accuracy of the transaction information, including but not limited to the transaction amount, target address, and operation type. ​

#### (3) Submitting the Transaction
By calling the `/openapi/multi/transaction` endpoint, users can submit the constructed FullNode standard `transaction` to the multi-signature service. This endpoint accepts the multi-signature transaction information submitted by the user, providing foundational data for subsequent signing and processing.

### II. Query Pending Transactions for Signing and Submit Signed Transactions to Multi-Signature Service

#### (1) Querying Pending Transactions
Use the WebSocket endpoint `/openapi/multi/socket` to establish a real-time connection and query pending transactions for the current address. This endpoint supports real-time pushing of pending transaction information, ensuring users can promptly receive transactions requiring action and perform subsequent signing operations. ​

#### (2) Signing and Submitting Transactions
Users sign the pending transactions digitally with the corresponding private key. After signing, the updated FullNode standard `transaction` object is submitted again via `/openapi/multi/transaction`. The multi-signature service automatically verifies the signature weight, and when the threshold is met, the system will automatically broadcast the transaction to the blockchain network. After broadcasting, users can query the transaction status and details on the blockchain using the transaction hash for real-time tracking.

# API List

### **All endpoints require authentication. Authentication logic refers to [[API Authentication Specification]](#api-authentication-specification)**

## 1. Query Associated Addresses and Permissions for Current Address

### Endpoint Name
Address Permission Query

### Endpoint Path
GET /openapi/multi/auth (optional, users can input their controlled address directly)

### Request Parameters
------------------------------------------------------------------------------------------------------
| **Parameter** | **Type** | **Required** | **Description** | **Example** |
|:--------------|:---------|:------------|:----------------|:------------|
| address       | string  | Yes         | Current address (to query addresses it controls) | TMf7fBmKPDGVP8b6UrEu1t6oDBRnNgwTt7 |
------------------------------------------------------------------------------------------------------

### Response Parameters
-------------------------------------------------------------------------
| **Field**             | **Type** | **Description** |
|:---------------------|:--------|:----------------|
| code                  | int     | Status code (0 = success, non-0 = failure) |
| message               | string  | Status message |
| data                  | array   | List of permissions |
| ├─ owner_address      | string  | Associated address |
| ├─ owner_permission   | object  | Owner permission for the address |
| │ ├─ operations       | string  | Allowed operation codes (empty = full permission) |
| │ ├─ threshold        | int     | Permission threshold (minimum weight for signing) |
| │ ├─ weight           | int     | Current address weight |
| └─ active_permissions | array   | Active permissions for the address |
-------------------------------------------------------------------------

### Response Example
```
{
    "code": 0,
    "message": "OK",
    "data": [
        {
            "owner_address": "TFDP1vFeSYPT6FUznL7zUjhg5X7p2AA8vw",
            "owner_permission": {
                "operations": "",
                "threshold": 3,
                "weight": 1
            },
            "active_permissions": [
                {
                    "operations": "1620008000000000000000000000000000000000000000000000000000000000",
                    "threshold": 8,
                    "weight": 1
                },
                {
                    "operations": "1018000000000000000000000000000000000000000000000000000000000000",
                    "threshold": 5,
                    "weight": 2
                }
            ]
        }
    ]
}
```

## 2. Construct and Submit Multi-Signature Transactions

### Endpoint Name
Multi-Signature Transaction Submission

### Endpoint Path
POST /openapi/multi/transaction

### Request Body Example
```
{
    "address": "TE4CeJSjLmBsXQva3F1HXvAbdAP71Q2Ucw",
    "function_selector":"transfer(address,uint256)",
    "transaction": {
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
        "signature": ["659143f51bea6f0b16ce1e5f98a662cf086eb033ce9a17fb204cdbdfa34ba75448af68e3ba746eddd53b552e70e5dbd4273b6bd649ae493361dddb28ad72b53800"]
    }
}
```

### Key Field Description
---------------------------------------------------------------------------
| **Field** | **Type** | **Description** |
|:----------|:---------|:----------------|
| address | string | Control address initiating the transaction |
| function_selector | string | Smart contract method (required when triggering a contract) |
| transaction.raw_data | object | Raw transaction data compliant with blockchain protocol |
| transaction.signature | array | Array of signature data (added sequentially) |
---------------------------------------------------------------------------

## 3. Pending Transaction Listener (WebSocket)

### Endpoint Name
Real-Time Pending Transaction Listener

### Endpoint Path
GET /openapi/multi/socket

### Protocol
WebSocket

### Connection Process
1. Authentication: The client includes valid authentication headers (format defined by server) in the HTTP request.
2. Connection Establishment: After server validation, the client sends the current operation address to subscribe to relevant transactions.
3. Data Interaction: The server pushes pending transaction details. The client performs signing operations based on business logic. The server also pushes updates to the status of related multi-signature transactions, allowing the frontend to determine whether a transaction requires signing.

### Client Request Example
```
{
    "address": "TW6omSrQ1ZK37SwSvTQD5Cnp2QbEX2zDVZ",
    "version":"v1"
}
```

### Server Response Example
```
[
    {
        "hash": "18213ab5b1d277b4090f647b925952efe972facd19462101f3d94a58b8354c23",
        "contract_type": "TransferContract",
        "originator_address": "TQUsaH7DzTAPQEVsUvQsVyzvwqwT2p7WEm",
        "expire_time": 0,
        "threshold": 3,
        "current_weight": 2,
        "is_sign": 1,
        "signature_progress": [
            {"address": "TW6omSrQ1ZK37SwSvTQD5Cnp2QbEX2zDVZ","weight": 1,"is_sign": 0,"sign_time": 0},
            {"address": "TQUsaH7DzTAPQEVsUvQsVyzvwqwT2p7WEm","weight": 1,"is_sign": 1,"sign_time": 1741858044},
            {"address": "TFdACej5gjKqSmwNNESzAbfTmBBCx55G4G","weight": 1,"is_sign": 1,"sign_time": 1741858044}
        ],
        "contract_data": {"amount": 1000000,"to_address": "TMf7fBmKPDGVP8b6UrEu1t6oDBRnNgwTt7","owner_address": "TQUsaH7DzTAPQEVsUvQsVyzvwqwT2p7WEm"},
        "current_transaction": {"raw_data": {"ref_block_bytes": "3e96","ref_block_num": null,"ref_block_hash": "6c2afde05160d139","expiration": 1741944318000,"auths": null,"data": "","contract": [{"type": "TransferContract","parameter": {"value": "0a15419f2e05d49b5fe66dce55598984aace7b3dc45fb012154180358ff232c17134b914a71b346a647dad006dfe18c0843d","type_url": "type.googleapis.com/protocol/TransferContract"},"provider": null,"ContractName": null,"Permission_id": 3}],"scripts": "","timestamp": 1741857918000,"fee_limit": null},"signature": ["3a53f8f5e4ed22a49a32e797d8ec9ed9dee4cd2dba8f00ee882a51bfd6691d94113a3f886f992803c191e38f59973f2b6521a7bea5235ee57eb86e3b757b4d9c1B","0e586c656a95450de017c67da4b78e8639e2537873b8b8ed6a3b39bce875724f548ac0e0f3e5fe4bd6bc395b10672c51f12598bc0835ac8f673c54c8b3e4ad0f1B"],"raw_data_hex": "0a023e9622086c2afde05160d13940b0d0e39fd9325a69080112630a2d747970652e676f6f676c65617069732e636f6d2f70726f746f636f6c2e5472616e73666572436f6e747261637412320a15419f2e05d49b5fe66dce55598984aace7b3dc45fb012154180358ff232c17134b914a71b346a647dad006dfe18c0843d280370b098caf6d832"},
        "state": 1,
        "function_selector": "transfer(address,uint256)"
    }
]
```

## 4. Transaction List Query

### Endpoint Name
Multi-Signature Transaction History Query

### Endpoint Path
GET /openapi/multi/list

### Request Parameters
---------------------------------------------------------------------------------------------------------------
| **Parameter** | **Type** | **Required** | **Description** |
|:-------------|:--------|:------------|:-----------------------------------------|
| address      | string | Yes        | Current address |
| start        | int    | Yes        | Pagination start index (if limit=10, page 2 start=10) |
| limit        | int    | Yes        | Pagination limit (max 100) |
| is_sign      | boolean| No         | Filter by own signature (true = signed; false = unsigned, default false) |
| state        | int    | Yes        | Filter by transaction status (0 = processing; 1 = success; 2 = failure; 255 = all) |
---------------------------------------------------------------------------------------------------------------

### Response Parameters
------------------------------------------------------------------------------
| **Field** | **Type** | **Description** |
|:---------|:---------|:----------------|
| code     | int      | Status code |
| message  | string   | Status message |
| data.total | int    | Total matching records |
| data.range_total | int | Total in current page range |
| data.data | array  | Array of transaction details (structure same as WebSocket push) |
------------------------------------------------------------------------------

### Response Example
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
                    {"address": "TW6omSrQ1ZK37SwSvTQD5Cnp2QbEX2zDVZ","weight": 1,"is_sign": 0,"sign_time": 0},
                    {"address": "TQUsaH7DzTAPQEVsUvQsVyzvwqwT2p7WEm","weight": 1,"is_sign": 1,"sign_time": 1741858044},
                    {"address": "TFdACej5gjKqSmwNNESzAbfTmBBCx55G4G","weight": 1,"is_sign": 1,"sign_time": 1741858044}
                ],
                "contract_data": {"amount": 1000000,"to_address": "TMf7fBmKPDGVP8b6UrEu1t6oDBRnNgwTt7","owner_address": "TQUsaH7DzTAPQEVsUvQsVyzvwqwT2p7WEm"},
                "current_transaction": {"raw_data": {"ref_block_bytes": "3e96","ref_block_num": null,"ref_block_hash": "6c2afde05160d139","expiration": 1741944318000,"auths": null,"data": "","contract": [{"type": "TransferContract","parameter": {"value": "0a15419f2e05d49b5fe66dce55598984aace7b3dc45fb012154180358ff232c17134b914a71b346a647dad006dfe18c0843d","type_url": "type.googleapis.com/protocol/TransferContract"},"provider": null,"ContractName": null,"Permission_id": 3}],"scripts": "","timestamp": 1741857918000,"fee_limit": null},"signature": ["3a53f8f5e4ed22a49a32e797d8ec9ed9dee4cd2dba8f00ee882a51bfd6691d94113a3f886f992803c191e38f59973f2b6521a7bea5235ee57eb86e3b757b4d9c1B","0e586c656a95450de017c67da4b78e8639e2537873b8b8ed6a3b39bce875724f548ac0e0f3e5fe4bd6bc395b10672c51f12598bc0835ac8f673c54c8b3e4ad0f1B"],"raw_data_hex": "0a023e9622086c2afde05160d13940b0d0e39fd9325a69080112630a2d747970652e676f6f676c65617069732e636f6d2f70726f746f636f6c2e5472616e73666572436f6e747261637412320a15419f2e05d49b5fe66dce55598984aace7b3dc45fb012154180358ff232c17134b914a71b346a647dad006dfe18c0843d280370b098caf6d832"},
                "state": 1,
                "function_selector": "transfer(address,uint256)"
            }
        ]
    }
}
```

# **API Authentication Specification**

## 1. Common Request Headers

[All API requests must include the following common header fields for authentication, version identification, and request tracking:]{.mark}

---------------------------------------------------------------------------
| **Name**        | **Type**        | **Default Value**                                 |
|:--------------|:--------------|:----------------------------------------|
| sign_version  | string        | v1 (current v1 API version)                             |
| ts            | long          | Current time in milliseconds                            |
| address       | string        | 58-character Tron address representing the requesting account |
| channel       | string        | Name of the requesting project, defined by the project (e.g., tronlink) |
| uuid          | string        | Unique ID for this request, randomly generated |
| secret_id     | string        | Project's unique identifier as agreed with Tronlink |
| sign          | string        | API signature to validate the request with Tronlink |
---------------------------------------------------------------------------

## 2. Signature (sign) Generation Rules

### **2.1 Sorting Signature Parameters**

Sort all common header parameters (excluding `sign`) in ascending ASCII order by field name. Concatenate them as key=value pairs, separated by `&`.

Example:

```
address=TMf7fBmKPDGVP8b6UrEu1t6oDBRnNgwTt7&channel=tronlink&secret_id=3d717E259617EA528F8&sign_version=v1&ts=174592188000&uuid=a6e4563f-1ce4-4a8f-ba37-de1cc121b4f8
```

### **2.2 Construct the Signature String**

Format: HTTP Method + Request Path + ? + Concatenated Parameters

Example (GET request, WebSocket uses GET):

```
GET/api/wallet/v2/auth?address=TMf7fBmKPDGVP8b6UrEu1t6oDBRnNgwTt7&channel=tronlink&secret_id=3d717E259617EA528F8&&sign_version=v1&ts=174592188000&uuid=a6e4563f-1ce4-4a8f-ba37-de1cc121b4f8
```

### **2.3 Generate Signature Value**

1. Use the HmacSHA256 algorithm with the project's `secretKey` as the key to encrypt the signature string.
2. Encode the result in Base64 to obtain the final `sign` parameter value.

## 3. Secret (secretId/secretKey) Application Process

### **3.1 Application Method**

Tronlink official staff provide a [Google Doc link](https://docs.google.com/forms/d/e/1FAIpQLSc5EB1X8JN7LA4SAVAG99VziXEY6Kv6JxmlBry9rUBlwI-GaQ/viewform?pli=1) where the project fills in project name, project details, and contact email.

### **3.2 Response**

Once approved, the project receives an email containing:

```
secretID: SSSSSSSSSSS (Project unique identifier)

secretKey: CCCCCCCCCCCCCCC (Signing key, keep secure)
```

## 4. Security Considerations

1. **Key Confidentiality:** `secretKey` is sensitive and must be strictly controlled to prevent leakage.

2. **Timestamp Verification:** The server will check the `ts` timestamp. It is recommended to sync client time with an NTP server, with a deviation within 5 minutes.

3. **UUID Uniqueness:** Each request must generate a unique `uuid` to avoid duplicate request issues.

4. **Signature Integrity:** Ensure the signature algorithm implementation strictly follows this specification; otherwise, authentication will fail.

For technical support or key reset, contact the official team.
