Hacash SDK Development Doc
===
When centralized exchanges, third-party wallets, etc. are connected to the Hacash blockchain in a normal way, the [API interface](https://github.com/hacash/doc-chinese/blob/main/server/fullnode_api_doc.md) of the full node already includes functions such as account and transfer creation, transaction query, and submission, which can meet the basic development needs.

When querying the latest transactions and blocks and submitting new transactions, full nodes need to maintain network operation to synchronize to the latest blocks, which will always be exposed to risks for businesses with large amounts and high security requirements. You can run a full-node software that is not networked, no blocks separately and isolate it in a intranet environment to create only create accounts, build transfer transactions, and sign transactions, and then broadcast transactions through another full node that remains online. Delete the `boots` line in the full node configuration file and run it in a networkless environment, which avoids the potential risk of private keys touching the network.

However, while the way to run two full nodes is very simple, and its HTTP API interface can be used in server environments developed in any language, it is difficult or even not feasible for special environments such as embedded systems, small specialized devices, and front-end browsers. In this case, integrating Hacash through SDK development is the best technical solution to achieve high security and rich customization, especially providing an SDK that can run in the browser, which will greatly improve the technical foundation of the Hacash ecosystem.

### Go & Rust SDK

As of now, Hacash provides SDKs in three languages: Go, Rust, and WASM. The early version of Hacash's full node software was developed in Golang, and later it was wholly switched to Rust. Both languages can be compiled to WASM, and the Hacash WASM SDK has been implemented for both languages. 

The Golang version of the full node has currently ceased iteration and development, but its already implemented basic functionalities, such as account creation, transaction construction, and transaction signing, remain secure and usable. Systems developed in Go can be integrated directly by importing code packages, and there are two source codes available for development reference:

1. [Golang Fullnode API Codes](https://github.com/hacash/service/blob/master/rpc), Example: [Build transfer tx](https://github.com/hacash/service/blob/master/rpc/createTransferTx.go)
2. [Golang WASM SDK Codes](https://github.com/hacash/jssdk/blob/main/wasmsdk/hac_transfer.go), Example:[Build HAC transfer tx](https://github.com/hacash/jssdk/blob/main/wasmsdk/hac_transfer.go)

Rust has also implemented both of these and can be integrated through direct reference to the source code:

1. [Rust Fullnode API Codes](https://github.com/hacash/fullnode/tree/main/server/src/api), Example:[Build transfer tx](https://github.com/hacash/fullnode/blob/main/server/src/api/create_transfer.rs)
2. [Rust WASM SDK Codes](https://github.com/hacash/fullnode/tree/main/sdk/src), Example:[Build transfer tx](https://github.com/hacash/fullnode/blob/main/sdk/src/coin.rs)

The SDKs for Go and Rust are not provided as separate downloadable code packages, but are open-sourced and released together with the full node source code. 

### WASM SDK 

Currently, Rust is used to compile and build the WASM SDK for developing Hacash-related business on Node.js servers, web browsers, or any other JavaScript language environments, allowing for functions such as account creation, transfer construction, or transaction signing. 

With the following crate, documentation, and test cases, you can compile the WASM SDK yourself and develop based on the SDK, provided that you have a Rust development environment installed:

1. [Rust WASM SDK Crate](https://github.com/hacash/fullnode/tree/main/sdk)
2. [WASM SDK Build Doc](https://github.com/hacash/fullnode/tree/main/sdk/README.md)
3. [WASM SDK Example](https://github.com/hacash/fullnode/tree/main/sdk/tests/test.html)

In addition, Hacash will release a compiled WASM SDK package whenever a full node is released, which can be downloaded and used directly.

- [Rust Fullnode Releases](https://github.com/hacash/fullnode/releases)

### Load WASM SDK

It is available in Node.js, Web, and a single JS file version for different development environments.Taking a single-page application as an example, you can include the compiled and packaged JS file that contains the WASM code in your HTML:

```html
<script src="./hacashsdk_bg.js"></script>
```

After successfully loading the file, you can initialize the Hacash SDK by using the global function `hacash_sdk()` with async/await:

```js
async function run() {
    let sdk = await hacash_sdk()
    console.log(sdk)
    do_test(sdk)
};
run()
```

或者：

```js
function do_test(sdk) {
    console.log(sdk)
}
hacash_sdk().then(do_test)
```

In the Node.js environment, the WASM SDK will be automatically initialized when loading the JS module, while in the Web approach, it uses the standard wasm-bindgen method to load. Please refer to the [wasm-bindgen documentation](https://wasm.rust-lang.net.cn/docs/wasm-bindgen/) for more information.

### SDK Doc

#### 1. Create Account `create_account`

```js
// password
let pass_or_prikey_hex = "123456"
// or private key
// let pass_or_prikey_hex = "8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92"
let account = sdk.create_account(
    pass_or_prikey_hex
)
console.log(account)
```

`account` return value example:

```js
{

    prikey: "8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92",
    pubkey: "0231745adae24044ff09c3541537160abb8d5d720275bbaeed0b3d035b1e8b263c",
    address_hex: "00e63c33a796b3032ce6b856f68fccf06608d9ed18",
    address: "1MzNY1oA3kfgYi75zquj3SRUPYztzXHzK9",
}
```

#### 2. Verify Address `verify_address`

```js
let addr = "1MzNY1oA3kfgYi75zquj3SRUPYztzXHzK9"
let result = sdk.verify_address(addr)
```

`result` return value example:

```js
{
    ok: true, // 地址是否有效
    error: "...some error tip...", // 如果无效的错误信息
}
```

#### 3. Create Coin Transfer Tx `create_coin_transfer`

```js

let trsobj = new sdk.CoinTransferParam()
// print(trsobj.timestamp.toString())
trsobj.timestamp = BigInt(1755223764)
trsobj.main_prikey = "123456"
trsobj.to_address = "1MzNY1oA3kfgYi75zquj3SRUPYztzXHzK9"
trsobj.fee = "1:244"
trsobj.hacash = "13.5" // HAC
// trsobj.satoshi = BigInt(12000000) // BTC
// trsobj.diamonds = "AAABBB,WWTTSS" // HACD name list
// prints(trsobj, trsobj.timestamp.toString())
let txres = sdk.create_coin_transfer(trsobj)
console.log(txres.body)

```

`txres` return value example:

```js
{
    hash: "5a86aff3ca12020c929280e686cf0c23a3f58d969eef6ecad3c29a3a6ddc6c92", // tx hash
    hash_with_fee: "...",
    body: "541537160abb8d5d720275bbaeed0b3d......", // tx body
    timestamp: 1755223764n, // tx timestamp
}
```

`body` is the value that contains the signed transaction body, which can be directly submitted to the full node.


#### 4. Do Sign for One Tx `sign_transaction`

```js

let stps = new sdk.SignTxParam()
stps.prikey = "abc123"
stps.body = "0200689e96d400e63c33a796b3032ce6b856f68fccf06608d9ed18f401010002000100e63c33a796b3032ce6b856f68fccf06608d9ed18f8010c000a00e63c33a796b3032ce6b856f68fccf06608d9ed180000000000b71b0000010231745adae24044ff09c3541537160abb8d5d720275bbaeed0b3d035b1e8b263c9b607f2bd9e1031536c13741facb78585755c116aa7d10628ebc2adbb4be96493bc1bb8ac6c3e78dee6717b9c4a27280b698efc91097d5900418a59c9d8e7ac30000" // tx body
try {
    let resg = sdk.sign_transaction(stps)
    console.log("sign res:", resg)
    console.log("tx body:", resg.body)
}catch(e) {
    // if something error
    console.log(e)
}
```

`resg` return value example:

```js
{

    hash:          "...", // tx hash
    hash_with_fee: "...",
    body:          "...", 
    signature:     signature.signature.hex(),
    timestamp:     trs.timestamp().uint(),
}
```

---

If you need more API or would like to participate in development, please [Submit issue](https://github.com/hacash/fullnode/issues) or push new codes.


