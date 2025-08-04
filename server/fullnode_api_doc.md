Hacash Rust Fullnode RPC API document
===

This document contains the api interface specification and examples for block scanning, transaction transfer query, account balance query, block diamond information query, creation of new accounts, creation of transfer transactions, etc. It is part of the development of the Hacash cryptocurrency nd facilitates the blockchain explorer browser and exchange functionality.

This document will provide a sample test interface (currently available, but deprecated), which can help you view the content returned by the interface immediately, or temporarily use it for testing and debugging. In a production environment, please configure your own server and set up a full node to ensure the stable availability and security of the interface. See [hacash.org](https://hacash.org/) for the tutorial of building a full node.

If you need to compile and build a full node, or learn about every feature of the configuration file, visit the following two documents:

- [Build_compilation.md](https://github.com/hacash/doc/blob/main/build/build_compilation.md)
- [Configuration description](https://github.com/hacash/doc/blob/main/build/config_description.md)

After downloading the latest version of the full node program and starting the program to synchronize all blocks, you can enable the RPC API service by adding the following configuration in the configuration file (hacash.config.ini):

```ini

[server]
enable = true
listen = 8081

```

The above configuration `enable = true` means to enable the RPC interface service, `listen = 8081` means that the listening http service port is 8081.

Visit `http://127.0.0.1:8081/` to verify operation, if all is good you will see:

```test
Hacash console
Latest height 575258 time 2024-08-13 13:31:22
Block span times: day: 288s, week: 320s, month: 304s, quarter: 303s, year: 297s, all: 302s
[TxPool] tx count: normal(0), diamond mint(1)
Miner worker notice connected: 0
```

### Quick start

Before documenting the detailed specification we can show a few examples, with the full node running, visit [http://127.0.0.1:8081/query/balance?address=1AVRuFXNFi3rdMrPH4hdqSgFrEBnWisWaS](http://127.0.0.1:8081/query/balance?address=1AVRuFXNFi3rdMrPH4hdqSgFrEBnWisWaS), this is an api query for the balance of an account address. For example upon success, it will return:

```json
{
    "list": [
        {
            "diamond": 1016,
            "hacash": "1474845:244",
            "satoshi": 0
        }
    ],
    "ret": 0
}
```

The Hacash RPC API service adopts a standard http interface, and you can simply use it in a browser, with curl or in an application.

The following document will use the test interface provided by hacash.org as an example, visit [http://nodeapi.hacash.org/query/balance?address=1AVRuFXNFi3rdMrPH4hdqSgFrEBnWisWaS](http://nodeapi.hacash.org/query/balance?address=1AVRuFXNFi3rdMrPH4hdqSgFrEBnWisWaS) should return the same interface data content as above.

Example Two: [http://nodeapi.hacash.org/query/balance?address=1AVRuFXNFi3rdMrPH4hdqSgFrEBnWisWaS,1MzNY1oA3kfgYi75zquj3SRUPYztzXHzK9&unit=mei](http://nodeapi.hacash.org/query/balance?address=1AVRuFXNFi3rdMrPH4hdqSgFrEBnWisWaS,1MzNY1oA3kfgYi75zquj3SRUPYztzXHzK9&unit=mei) represents Query the balances of two addresses at the same time, and the returned balance unit is "mei".

### Interface format, general parameters

The standard HTTP interface request and response method is adopted, including 4 paths:

   1. /create generates data, GET method, such as creating an account, generating transfer transactions, etc.
   2. /submit submit data, POST method, such as submitting the transaction to the pool to be confirmed, etc.
   3. /query query data, GET method, such as query account balance, etc.
   4. /operate modify data, GET/POST method, such as modify system operating parameters, etc.

The parameters common to multiple interfaces are introduced as follows:

| Parameter name | Type | Default value | Example value | Function introduction |
| ---- | ---- | ---- | ---- | ---- |
| unit | string  | - | mei,zhu,shuo,ai,miao,248,244,240,... | return the HAC amount use unit: mei, zhu, shuo, zi, miao |
| coinkind | menu |-| h, s, d, hs, hd, hsd, all | Filter the returned account and transaction information type. h: hacash, s: satoshi, d: diamond. Purpose: For example, when scanning a block, you only need to return the HAC transfer content and ignore the other two, just pass `coinkind=h`. |
| hexbody | bool | false | true, false | `/submit` Whether to use the hex string form of Http Body when submitting data. The default format is native bytes. |
| base64body | bool | false | true, false | Whether all binary data is returned with base64 encoding. |

All of the above parameters are optional.

### Return value, public field

Respond to all requests in standard JSON format. The public fields are as follows:

```js

{
  "ret": 0, // indicates the return type, 0 is correct, >= 1 indicates an error occurred or the query does not exist
  "list": [...], // Some interfaces that return list data will be used
  "err": "...." // This field is returned if there is an error
}

```

The following section will introduce the parameter passing and return value details of each specific interface.

---

## 1. /create create

#### 1.1 Create an account `GET: /create/accounts`

Create accounts in batches at random, and return a list of account information including private keys, public keys, and addresses. Passable parameters:

- quantity [int] indicates the number of accounts created in batches, the default is 1, the maximum is 200

Example interface: [http://nodeapi.hacash.org/create/account?quantity=5](http://nodeapi.hacash.org/create/account?quantity=5)

return value:

```js
{
    list: [
        {
            address: "1NLEYVmmUkhAH18WfCUDc5CHnbr7Bv5TaS", // account address
            prikey: "2e50243243abc2e41f3b2ae90029640e235d884a88cfb5ea3e4d0e9efbae6710", // private key
            pubkey: "03e22fc27a0d7ae325fa024875febd58266b8b6adbfb966116c9ba958ff5bad7e6" // public key
        },
        ...
    ],
    ret: 0
}
```

Special note: The system uses a random generation algorithm to create an account. The system does not retain or record the account private key created by this interface. Please make sure to back up and store the created private key, and take security protection.

#### 1.2 Create a transfer transaction `GET: /create/coin/transfer`

This interface is used to create HAC, one-way transfer of Bitcoin and block diamond transfer transactions. The basic parameters are as follows:

- main_prikey [hex string] The private key hex string of the main address/handling fee address
- from_prikey [hex string] The private key hex string of the from address (When not give in, it is the same as main_prikey)
- fee [string] The fee value to be given for the transaction, such as "0.0001" or "1:244"
- timestamp [int] The timestamp of the transaction; it is optional to pass; if it is not passed, it is automatically set to the current timestamp
- to_address [string] Counter (receiving) account address

[Note 1]: When only passing the same `timestamp` timestamp parameter and keeping other parameters always the same, each transaction created will have the same hash value and be regarded as the same transaction.

[Note two]: Since the Hacash system supports repeated signature fee bidding for the same transaction, only changing the `fee` field will not change the transaction's `hash` value, but only its `hash_with_fee` value.

[Note three]: The fee field of `fee` cannot be set too small, otherwise it will not be accepted by the entire system. The current minimum fee is 0.0001 (ie ㄜ 1:244). Please do not set the handling fee lower than this value.

##### 1.2.1 Create HAC ordinary transfer transaction

Pass the parameter `hacash=1:248`, and add the following parameters:

 - hacash [string] The transfer amount; the unit can be mei or float, "0.1" or "1:247".

Call Interface Example: [http://nodeapi.hacash.org/create/coin/transfer?main_prikey=8D969EEF6ECAD3C29A3A629280E686CF0C3F5D5A86AFF3CA12020C923ADC6C92&fee=0.0001&timestamp=1603284999&hacash=12.45&to_address=1NLEYVmmUkhAH18WfCUDc5CHnbr7Bv5TaS](http://nodeapi.hacash.org/create/coin/transfer?main_prikey=8D969EEF6ECAD3C29A3A629280E686CF0C3F5D5A86AFF3CA12020C923ADC6C92&fee=0.0001&timestamp=1603284999&hacash=12.45&to_address=1NLEYVmmUkhAH18WfCUDc5CHnbr7Bv5TaS)


The return value is as follows:

```js
{
    // public parameters
    ret: 0,
    // The hash value of the transaction
    hash: "6066cef4fe51669aec5b5596375dba11dafaf2560c4bd8c0432ac4ea98ff3ad1",
    // The transaction contains the hash value of the fee
    hash_with_fee: "9cbc4821d0d921b429dbe4d6b67d6412aa5cae4b724f1e7dab9f870646cb1bb6",
    // The hex value of transaction body and content
    body: "02005f90300700e63c33a796b3032ce6b856f68fccf06608d9ed18f401010001000100e9fdd992667de1734f0ef758bafcd517179e6f1bf60204dd00010231745adae24044ff09c3541537160abb8d5d720275bbaeed0b3d035b1e8b263cb73b724218f13c09c16e7065212128cf0c037ebb9e588754eb073886486d950607d59bef462d2731e15b667c6ff1f0badd6259c6f58d5ca7a5f75856b8cae8e80000 ",
    // timestamp used in the transaction
    timestamp: 1603284999
}
```

In a production environment, please save the above return value in the database for reconciliation, or resubmit unconfirmed transactions when the blockchain network is delayed. The above content does not reveal your private key, but only the transaction data after the signature, please make sure to store it.

Create BTC transfer and block diamond transfer, the return value of calling the interface is the same as the above.

##### 1.2.2 Create transferred bitcoin ordinary transfer transaction

Pass the parameter `satoshi=?`, and add the following parameters:

 - satoshi [int] The amount of bitcoins to be paid, in units of "satoshi" and "satoshi" (0.00000001 bitcoins); for example, transfer 10 bitcoins to transfer "1000000000", transfer 0.01 bitcoins to transfer "1000000"; system Bitcoin units below 1 satoshi are not supported.

For example, to transfer a bit of an address token, the example interfaces such as:

[http://nodeapi.hacash.org/create/coin/transfer?main_prikey=8D969EEF6ECAD3C29A3A629280E686CF0C3F5D5A86AFF3CA12020C923ADC6C92&fee=0.0001&timestamp=1603284999&satoshi=50000000&to_address=1NLEYVmmUkhAH18WfCUDc5CHnbr7Bv5TaS](http://nodeapi.hacash.org/create/coin/transfer?main_prikey=8D969EEF6ECAD3C29A3A629280E686CF0C3F5D5A86AFF3CA12020C923ADC6C92&fee=0.0001&timestamp=1603284999&satoshi=50000000&to_address=1NLEYVmmUkhAH18WfCUDc5CHnbr7Bv5TaS)

##### 1.2.3 Create block diamond transfer transaction

Pass the parameter `diamonds=?`, and add the following parameters:

- diamonds [string] The literal value of diamonds separated by commas, such as "EVUNXZ,BVVTSI", one or more can be transferred, and up to 200 diamonds can be transferred in one batch

Interface call Example:

[http://nodeapi.hacash.org/create/coin/transfer?main_prikey=8D969EEF6ECAD3C29A3A629280E686CF0C3F5D5A86AFF3CA12020C923ADC6C92&fee=0.0003&timestamp=1603284999&diamonds=EVUNXZ,BVVTSI&from_prikey=EF797C8118F02DFB649607DD5D3F8C7623048C9C063D532CC95C5ED7A898A64F&to_address=1NLEYVmmUkhAH18WfCUDc5CHnbr7Bv5TaS](http://nodeapi.hacash.org/create/coin/transfer?main_prikey=8D969EEF6ECAD3C29A3A629280E686CF0C3F5D5A86AFF3CA12020C923ADC6C92&fee=0.0003&timestamp=1603284999&diamonds=EVUNXZ,BVVTSI&from_prikey=EF797C8118F02DFB649607DD5D3F8C7623048C9C063D532CC95C5ED7A898A64F&to_address=1NLEYVmmUkhAH18WfCUDc5CHnbr7Bv5TaS)


---

## 2. /submit Submit

#### 2.1 Submit a transaction to the transaction pool `POST: /submit/transaction`

Submit a transaction to the memory pool of the entire network.

The url parameters are as follows:

- hexbody [bool] Pass the post body value in the form of hex string; otherwise pass it in the form of native bytes

The return value after the call is as follows

```js
{
    ret: 0 // ret = 0 means return success
}
```

Or return an error

```js
{
    ret: 1 // ret = 1 means submitting a transaction error
    // For example, if the balance is insufficient, the error message is as follows:
    err: "address 1MzNY1oA3kfgYi75zquj3SRUPYztzXHzK9 balance ㄜ0:0 not enough, need ㄜ1,245:246."
}
```
Example of curl command line test call:

```shell script
curl "http://nodeapi.hacash.org/submit/transaction?hexbody=true" -X POST -d "02005f90300700e63c33a796b3032ce6b856f68fccf06608d9ed18f401010001000100e9fdd992667de1734f0ef758bafcd517179e6f1bf60204dd00010231745adae24044ff09c3541537160abb8d5d720275bbaeed0b3d035b1e8b263cb73b724218f13c09c16e7065212128cf0c037ebb9e588754eb073886486d950607d59bef462d2731e15b667c6ff1f0badd6259c6f58d5ca7a5f75856b8cae8e80000"
```

Handling fee bidding description 1: After the transaction is submitted, when you need to pay a higher handling fee as a diamond bid, or increase the handling fee to speed up the transaction confirmation, you can call the `/create` interface to create the same transfer transaction again, and only modify the amount The higher `fee` field (all other fields including the timestamp must be consistent, otherwise it will be two different transactions), and then resubmit the transaction, the entire network will discard the lower fee and pack this A transaction with a higher fee bid. All different fee bids are regarded as different states of a transaction, no matter how many bids are made, there will be only one package in the end.

Handling fee bidding explanation 2: When the transaction is still in the trading pool, you can call the `/operate` interface to modify the handling fee bidding more conveniently without resubmitting the entire transaction content. For interface description, see the content at the bottom of the document.

#### 2.2 Submit a new block to the mainnet `POST: /submit/block`

Submit a block to the blockchain of the entire network.

The url parameters are as follows:

- hexbody [bool] Pass the post body value in the form of hex string; otherwise pass it in the form of native bytes

This interface is mainly used by developers of third-party mining pools.

The return value after the call is as follows

```js
{
    ret: 0 // ret = 0 means return success
}
```

---

## 3. /query Query

#### 3.0 Summary

| Query | Arguments | Mandatory | Notes |
| ---- | ---- | ---- | ---- |
| balance | address, unit, kind,  |address| up to 200 comma, delmited addresses , kind : hsd, unit : boolean |
| diamond | name or number | name or number | singleton returns diamond information |
| latest |  | | block height & diamond number |
| supply |  | | coin total supply |
| hashrate |  | | PoW mining hashrates |
| hashrate/logs |  | | PoW mining hashrates historical statistics |
| block/intro | height or hash | height or hash | returns block headers |
| coin/transfer | height, txhash, txposi, kind, unit| height or txhash| returns transaction for block height index txposi (default=0) |
| transaction | hash, unit| hash| return transaction info and actions |


#### 3.1 Query account balance `GET: /query/balance`

Check the HAC, BTC, and diamond balances of one or more accounts at once.

Pass parameters:

- address [string] : A comma-separated list of account addresses; up to 200 batch queries
- diamonds [bool] : Whether to return a list of all diamond names owned by this account
- coinkind [menu] : Query the type of return; `kind=h` only returns HAC balance, `kind=hs` returns HAC, BTC balance, if you do not pass or pass `hsd`, all types of balances are returned

 Example Interface:

 [http://nodeapi.hacash.org/query/balance?unit=mei&address=18Yt6UbnDKaXaBaMPnBdEHomRYVKwcGgyH,1DYY4ZRsWnhjcwwnE3dWgtiqe2mctDS2HF](http://nodeapi.hacash.org/query/balance?unit=mei&address=18Yt6UbnDKaXaBaMPnBdEHomRYVKwcGgyH,1DYY4ZRsWnhjcwwnE3dWgtiqe2mctDS2HF)
 
 Or get all HACD name list by add param `diamonds=true`:

 [http://nodeapi.hacash.org/query/balance?unit=mei&address=18Yt6UbnDKaXaBaMPnBdEHomRYVKwcGgyH,1DYY4ZRsWnhjcwwnE3dWgtiqe2mctDS2HF&diamonds=true](http://nodeapi.hacash.org/query/balance?unit=mei&address=18Yt6UbnDKaXaBaMPnBdEHomRYVKwcGgyH,1DYY4ZRsWnhjcwwnE3dWgtiqe2mctDS2HF&diamonds=true)
 

 Example return value:

 ```js
{
    ret: 0, // general return value
    list: [
        {
            diamond: 350, // the number of diamonds
            hacash: "3101.0826", // HAC balance
            satoshi: 0, // BTC balance, unit "satoshi": satoshi
            diamonds: "ITHWMSVBYYYTWKIKNZKNTUYWAAABBBWWWTTT"
        },
        {
            diamond: 0,
            hacash: "3842.24693214",
            satoshi: 0
        }
    ]
}
```
Note 1: The returned balance list corresponds to the position of the passed account list.

Note 2: If you need to return the `diamonds` field, please enable `diamond_form = true` in the`[server]` group in the full node configuration file, and the full node will enable the record of the account HACD list.

#### 3.2 Query block diamond information `GET: /query/diamond`

Query detailed information such as the hash of a diamond, the height of the generated block, and the current address, etc.; you can query by diamond literal (name) or label (number).

Pass parameters:

- name [string] Diamond literal value, for example: "ZAKXMI"
- number [int] Diamond label, for example: "20001"

Example query interface 1: [http://nodeapi.hacash.org/query/diamond?number=20001](http://nodeapi.hacash.org/query/diamond?number=20001)

Example query interface 2: [http://nodeapi.hacash.org/query/diamond?name=ZAKXMI](http://nodeapi.hacash.org/query/diamond?name=ZAKXMI)

The above two interfaces return the same content:

```js
{
    ret: 0, // general return value
    number: 20001, // Diamond number
    name: "ZAKXMI", // Diamond name / face value
    bid_fee: "382:246", // The bidding fee for this diamond
    mint_height: 179610, // Height of excavated block
    belong: "1KcXiRhMgGcvgxZGLBkLvogKLNKNXfKjEr", // Diamond's current belong address
    visual_gene: "e5e5b79409af7423bca1",
    life_gene:"aa41cc92bbff792266fca82217ddd0aabb2477f519badf071472037b7cea21e5",
    inscripts:["...."],
}

```

We can see that the above two interface queries are essentially the same diamond.

#### 3.3 Query the latest valid block information `GET: /query/latest`

Query the latest effective block information of the current blockchain.

Request example: [http://nodeapi.hacash.org/query/latest](http://nodeapi.hacash.org/query/latest)

return data： 

```js
{
    ret: 0,
    height: 575611, // Currently, the block height that has been written into the effective blockchain
    diamond: 96254, // The number of the diamond that has been minted
}
```

#### 3.4 Query the specified block information `GET: /query/block/intro`

params:

- height [int] Optional, query block information by specifying height
- hash [string] Optional, query block information by block hash value
 
Example interface 1:  [http://nodeapi.hacash.org/query/block/intro?height=177255&unit=mei](http://nodeapi.hacash.org/query/block/intro?height=177255&unit=mei)

Example interface 2: [http://nodeapi.hacash.org/query/block/intro?hash=000000001e76b27f70f9a5e5b0f9b9824b416c210548ffad298577ed5378cdcd](http://nodeapi.hacash.org/query/block/intro?hash=000000001e76b27f70f9a5e5b0f9b9824b416c210548ffad298577ed5378cdcd)
 
return data：

```js
{
    ret: 0,
    height: 177255, // block height
    hash: "000000001e76b27f70f9a5e5b0f9b9824b416c210548ffad298577ed5378cdcd", // block hash
    mrklroot: "d18df37a52c377115f053aca7822c65e5d154dac4ab6e062326dc770b5d1e5ad", // Merkel Root Hash
    prevhash: "000000001868742351b8ab196444d22c0db2951c522ebc5b9ef057ad70a6556c", // Previous block hash
    timestamp: 1601931702, // Block creation timestamp
    transaction: 5, // The block contains the number of transactions
}
```

#### 3.5 Scan each transaction to obtain a brief transfer operation `GET: /query/coin/transfer`

The `scan_value_transfer` interface returns richer and more structured content. If you only need to obtain transfer actions for HAC, HACD, and BTC for simple needs such as exchange recharge, you can use `scan_coin_transfer` interface makes the content more concise.

Pass parameters:

- height [int] Block height to scan
- txposi [int] The transaction to be scanned is at the array index position in the block, starting from 0
- unit [string] Optional, whether to return floating-point string in units of "Mei"
- coinkind [menu] Optional, Query the type of return; `kind=h` returns only HAC transfer, `kind=hs` returns HAC and BTC transfer, and returns all types of transfer without transferring or passing `kind=hsd`
- from [string] Optional, The from address to filter will only display the incoming address after passing, ignoring other from addresses
- to [string] Optional, The to address to filter will only display the incoming address after passing, ignoring other to addresses

Example API 1：[http://nodeapi.hacash.org/query/coin/transfer?unit=mei&height=180115&txposi=1&coinkind=hsd](http://nodeapi.hacash.org/query/coin/transfer?unit=mei&height=180115&txposi=1&coinkind=hsd)

Example Return：

```js
{
    ret: 0,
    tx_hash: "43e98177bc426a2f15c15fe3e8968ece1b2e0829eab7cacb8715c89082f48aef",
    tx_timestamp: 1698278005,
    block_hash: "0000000006010e2588ccd1a2b3c391348e204e119a9be6c05f05e57fba8c7ddd",
    block_timestamp: 1602764680,
    transfers: [
        {
            // By default, both from and to addresses will be displayed, making it easier to query
            from: "13RnDii79ypWayV8XkrBFFci29cHtzmq3Z",
            to: "1EcrtFAUmVeLnGeaEcMDoPZH7ZPysks1H2",
            hacash: "0.9370996"
        },
        {
            from: "13RnDii79ypWayV8XkrBFFci29cHtzmq3Z",
            to: "1EcrtFAUmVeLnGeaEcMDoPZH7ZPysks1H2",
            sotoshi: "5000000"
        },
        {
            from: "13RnDii79ypWayV8XkrBFFci29cHtzmq3Z",
            to: "1EcrtFAUmVeLnGeaEcMDoPZH7ZPysks1H2",
            diamond: 1,
            diamonds: "WUZXYM"
        },
        {
            from: "13RnDii79ypWayV8XkrBFFci29cHtzmq3Z",
            to: "1EcrtFAUmVeLnGeaEcMDoPZH7ZPysks1H2",
            diamond: 3,
            diamonds: "WUZXYMIZHTEWIIUMWH" // Batch transfer of diamonds
        }
    ],
    type: 2
}

```


#### 3.6 Scan each diamond inscription or clear `GET: /query/diamond/engrave`

Pass parameters:

- height [int] Block height to scan
- txposi [int] Optional, The transaction to be scanned is at the array index position in the block, starting from 0
- tx_hash [bool] Optional, Whether to return each engraved transaction hash

If don't pass the `txposi` field, the inscription for the entire block is scanned at once.

Example API 1：[http://nodeapi.hacash.org/query/diamond/engrave?height=519242&txposi=0](http://nodeapi.hacash.org/query/diamond/engrave?height=519242&txposi=0)

Example API 2：[http://nodeapi.hacash.org/query/diamond/engrave?height=531045](http://nodeapi.hacash.org/query/diamond/engrave?height=531045)

Example API 3：[http://nodeapi.hacash.org/query/diamond/engrave?height=540771&tx_hash=true](http://nodeapi.hacash.org/query/diamond/engrave?height=540771&tx_hash=true)

Example Return：

```js
{
    ret: 0,
    list: [
    {   /* do single engrave */
        inscription: "HIP-15 1st HACD",
        diamonds:"BMABAT"
    },
    {   /* multiple diamonds are engraved in batches */
        inscription: "Hacds",
        diamonds:"SHHHKSUKIUEIZKBYEVHKYWBYAUTZHY",
        tx_hash: '9beeb7c528d5e9f9edfbd08f16a5d117164d466ae4c8edc038609d9117119cc2' /* if pass `txposi` field  */
    },
    {   /* clear all inscription */
        clear: true, // clear mark
        diamonds:"WETUAX" // or "SHHHKSUKIUEIZKBYEVHKYWBYAUTZHY"
    }
    ]
}
```



#### 3.7 get transaction info `GET: /query/transaction`

Pass parameters:

- hash [int] Tx hash
- action [bool] Optional, whether to return actions info
- unit [string] Optional, whether to return floating-point string in units of "mei" or "248"
- body [bool] Optional, whether to return transaction body data of hex
- signature [bool] Optional, whether to return transaction signature check object
- description [bool] Optional, whether to return transaction actions description

Example API 1：[http://nodeapi.hacash.org/query/transaction?hash=b563b525d5a840f14c2f1d7da52a1eb3f7ee7357e784af8b68d0b0f74b95cbb4&action=true&unit=mei](http://nodeapi.hacash.org/query/transaction?hash=b563b525d5a840f14c2f1d7da52a1eb3f7ee7357e784af8b68d0b0f74b95cbb4&action=true&unit=mei)

Example Return：

```js
{
    ret: 0,
    block: {
        height: 583881,
        timestamp: 1726169459
    },
    confirm: 4,
    hash_with_fee: "ee5c254b1f1628eacff17ea39836a17569d5d5bbd8d9e97b8a0383dac95740ac",
    hash: "b563b525d5a840f14c2f1d7da52a1eb3f7ee7357e784af8b68d0b0f74b95cbb4"
    fee: "0.0001",
    fee_got: "0.0001",
    type: 2,
    timestamp: 1726169260,
    main_address: "12pbJdDmvZcRs3BhJnJ1RKuiHE48mVJS71",
    action: 1,
    actions:[{
        kind: 1,
        hacash: "15.9975414",
        from: "12pbJdDmvZcRs3BhJnJ1RKuiHE48mVJS71",
        to: "1NzdWYfp42gpRDf9Hsv5CA6D5gQvMt3599"
    }],
}

```

Note: the fields ' hacash / satoshi / diamond / diamonds / from / to ' in the `actions` array are the same as the `transfers` fields in the other api `GET: /query/coin/transfer`


#### 3.8 Query total supply `GET: /query/supply`

 examples link：[http://nodeapi.hacash.org/query/supply](http://nodeapi.hacash.org/query/supply)
 
 examples return data：
 
```js
{
    ret: 0,
    minted_diamond: 27538, // The number of diamonds that have been minted successfully
    channel_interest: 151.6902713463587, // Channel interest HAC accumulation
    burning_fee: 890984.44654182, // Tx fee by burning
    current_circulation: 240364.69027134636, // Current circulation supply
    block_reward: 1804911.0, // Accumulate the block rewards that have been released
}
```

#### 3.9 Query average fee `GET: /query/fee/average`

Get real-time average fees for the current blocks of the blockchain (recommendation fees).

Pass parameters:

- consumption [int] Tx byte size (must full size including all signature datas), The handling fee of ordinary Hacash transactions is calculated in terms of size, and when the transaction size is large, more fees need to be paid. This parameter will return the recommended tx fee when passed
- unit [string] Optional, whether to return floating-point string in units of "mei" or "248"

examples link：[http://nodeapi.hacash.org/query/fee/average?consumption=120](http://nodeapi.hacash.org/query/fee/average?consumption=120)
 
examples return data：
 
```js
{
    ret: 0,
    feasible: "6024:240" // fee settings for tx with size 120 bytes
}
```


---

## 4. /operate Modification, operation

#### 4.1 Increase transaction fee for confirmation `GET: /operate/fee/raise`

When a transaction exists in the transaction pool and has not been confirmed, it can be re-signed to increase (but not reduce) the transaction fee, so as to speed up transaction confirmation or diamond bidding.

Pass parameters:

- tx_hash [string] The hash value of the transaction to increase the fee
- fee [string] The target fee to be modified
- fee_prikey [string] The private key of the fee address
- unit [string] Optional, whether to use the unit "mei" as the unit 


Note: when the transaction does not exist in the transaction pool, modify the fee by using POST the submitted HTTP BODY, and broadcast the transaction to the whole network again

Example Access Interface: 

[http://nodeapi.hacash.org/operate/fee/raise?fee_prikey=8D969EEF6ECAD3C29A3A629280E686CF0C3F5D5A86AFF3CA12020C923ADC6C92&fee=32:247&tx_hash=ad26a35116664176426f3c08adad147577b9a85999cb89d465becf6a27002c04](http://nodeapi.hacash.org/operate?action=raise_tx_fee&fee_prikey=8D969EEF6ECAD3C29A3A629280E686CF0C3F5D5A86AFF3CA12020C923ADC6C92&fee=ㄜ32:247&txhash=ad26a35116664176426f3c08adad147577b9a85999cb89d465becf6a27002c04)

Example of interface return:

```js
{
    ret: 0,
    hash: "",
    hash_with_fee: "",
    fee: "2:244",
    tx_body: "...",
}
```

Example of failed return:

```js
{
    ret: 1,
    err: "Tx fee address password error: need 18Yt6UbnDKaXaBaMPnBdEHomRYVKwcGgyH but got 1MzNY1oA3kfgYi75zquj3SRUPYztzXHzK9" // Reason for failure: Inconsistent fee address
}

```   


---


## Interfaces related to mining or mining pools

The new Rust version of Full Node does not integrate mining functions internally, but is exposed to external programs through an HTTP interface. The PoWorker mining program released with the full node is directly connected to the full node and automatically mined through the following interface.


#### 5.1 Read stuff about the block being mined `GET: /query/miner/pending`

Example Access Interface: 

[http://nodeapi.hacash.org/query/miner/pending](http://nodeapi.hacash.org/query/miner/pending)

Pass parameters:

- detail [bool] Optional, Whether to return more detailed block fields
- transaction [bool] Optional, Whether to return all transaction body data
- base64 [bool] Optional, Whether all binary data is returned with base64 encoding


Example of interface return:

```js
{
    ret: 0,
    height: 575618,
    target_hash: "00000000001172658fffffffffffffffffffffffffff",
    coinbase_nonce: "0000000000000000000000000000000000000000000000000000000000000001",
    block_intro: "01000008c8820066bc9bce000000000008e3c7eda5a7abd046e09ac2b5902aca50d4cd88ae8963e13925522566fb0e6c3e95ed652ee5e23a7d0b10db6e88ac5c27c1ba2b1cbe429a68cb540000000700000000d48b932c0000"
}
```

If pass the url parameter `detail=true`, increment the following fields to return:

```js
{
    version: 1,
    prevhash: "...",
    timestamp: 23485762,
    transaction_count: 6,
    reward_address: "1AVRuFXNFi3rdMrPH4hdqSgFrEBnWisWaS",
}
```

If pass the url parameter `transaction=true`, increment the following fields to return:

```js
{
    transaction_body_list: [
        "....",
        "...."
        // ...
    ]
}
```

The above return value, when the same block, will return a different `coinbase_nonce` for each req, so as to automatically get a `uint32` hash attempt space called the size of `block_nonce`.

`target_hash` is the target difficulty hash under the current computing difficulty, that is, the number of leading zeros of the calculated block hash cannot be less than this. The `coinbase_nonce` needs to be recorded by the mining side so that it can be returned through the interface when the block is successfully mined. 'block_intro' is the block header data, which is the raw material for mining.

Mining is done by constantly changing the nonce value of a block in an attempt to calculate a hash of the block that satisfies the `target_hash` difficulty goal.

```rust
    let mut block_intro = hex::decode("01000008c8820066bc9bce000000000008e3c7eda5a7abd046e09ac2b5902aca50d4cd88ae8963e13925522566fb0e6c3e95ed652ee5e23a7d0b10db6e88ac5c27c1ba2b1cbe429a68cb540000000700000000d48b932c0000").unwrap();
    for nonce in 0..u32::MAX {
        block_intro[79..83].copy_from_slice(&nonce.to_be_bytes());
        let reshx = x16rs::block_hash(hei, &intro).to_vec();
        // check reshx
    }
```

The implementation of the mining algorithm can be found here:

[https://github.com/hacash/rust/blob/95eb6670a3f26b69e4b4a7f006942553fec003af/src/run/poworker.rs#L488](https://github.com/hacash/rust/blob/95eb6670a3f26b69e4b4a7f006942553fec003af/src/run/poworker.rs#L488)



#### 5.2 Submit a successfully mined block `GET: /submit/miner/success`

Pass parameters:

- height [int] 
- block_nonce [uint32] Block nonce your find successfully
- coinbase_nonce [string] The value get by the interface `/query/miner/pending`, returned it unchanged

Example Access Interface: 

[http://nodeapi.hacash.org/submit/miner/success?height=575618&block_nonce=2236475323&coinbase_nonce=0000000000000000000000000000000000000000000000000000000000000001](http://nodeapi.hacash.org/submit/miner/success?height=575618&block_nonce=2236475323&coinbase_nonce=0000000000000000000000000000000000000000000000000000000000000001)

Example of interface return:

```js
{
    ret: 0,
    height: 575618,
    mining: "success",
}
```

Note that you must first access the `/query/miner/pending` interface to create and get the block to be mined, and then submit the block through this interface later for it to take effect.




#### 5.3 Subscribe to the latest block height changes `GET: /query/miner/notice`

Pass parameters:

- height [int] The height of the block to be monitored
- rqid [string] A unique arbitrary string is randomly generated for each request to prevent problems such as HTTP caching
- wait [int] The number of seconds that a long HTTP link waits

Example Access Interface: 

[http://127.0.0.1:8081/query/miner/notice?height=367454&rqid=50d4cd88ae8963e13925522566fb0e6c3e95ed652ee5e23a](http://127.0.0.1:8081/query/miner/notice?height=367454&rqid=50d4cd88ae8963e13925522566fb0e6c3e95ed652ee5e23a)

Example of interface return:

```js
{
    ret: 0,
    height: 575622,
}
```

This API is an interface that polls the entire network for block height changes through HTTP long links. When the incoming 'height' has been mined, this interface will return immediately, and when the asked block height has not been mined, this interface will remain connected and wait, and the time of each wait is specified by the `wait` parameter, up to a maximum of 300 seconds.

The use of HTTP round robin instead of TCP or WebSocket is mainly for the simplicity of mining or mining pool development, as well as the richness of multiple devices or network environments, which can support more technical systems to access mining.

Here's a 500-line Rust implementation of a multi-threaded mining client that displays hashrate statistics in real-time, which is a very clear example if you're a developer of Hacash mining or mining pools:

[https://github.com/hacash/rust/blob/main/src/run/poworker.rs](https://github.com/hacash/rust/blob/main/src/run/poworker.rs)



