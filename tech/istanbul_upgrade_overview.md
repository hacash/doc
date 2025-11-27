Istanbul Upgrade Overview
===

Since the release of its white paper in 2018 and the mining of its first block in 2019, Hacash has been running steadily for six years. The richness of its PoW economic model and the forward-thinking nature of its Crypto Sound Money theory have been proven over the long term.

This upgrade (named Istanbul, borrowing the city name) integrates a variety of major technologies proposed and developed over the years, covering aspects such as asset issuance, contract construction, and programmability, including HIP-20, TDEX, AST (HIP-16), and HVM (developed by Hacash.com). From this point on, Hacash has achieved a revolutionary breakthrough in public chain technology: transforming from a pure currency issuance and payment blockchain into a foundational ecosystem that supports the continuous innovation and secure operation of global currencies, assets, and financial systems.

This document will briefly introduce this series of major technologies and outline a new world with encrypted financial centered on sound money.


### HIP-20

The reason this protocol is called HIP-20 (Hacash Improvement Proposal) rather than HRC-20 (Hacash Request for Comment) is that assets issued on Hacash are not second-class citizens residing on the HVM layer, but first-class assets treated exactly the same as HAC, HACD, and BTC.

This means that all technical infrastructure layers of Hacash, including readable contracts, conditional contracts, and channel chain of Layer-2 payments, will be fully open to HIP-20 assets, without requiring additional programming at the VM level to simulate them. For example, an atomic swap transaction between a HIP-20 asset and HAC can be natively supported, just like the current atomic swaps between HACD and HAC on the chain. Similarly, within the channel chain network, through currency exchanges provided by payment service providers, a customer can instantly pay with HAC while the merchant receives USDT in real time.

This new paradigm of chain-native asset issuance, distinct from smart contract programming, can enhance both security and user-friendliness. The public chain will evolve from a purely technical smart contract programming platform into an ecosystem of native assets.

Of course, it is also fully possible to issue and trade traditional contract-based assets through programming within the HVM.


### TDEX

This is another on-chain automated trading protocol based on Hacash readable contracts, officially called the Transaction Decentralized Exchange (TDEX).

Unlike the already implemented on-chain atomic swap (which require explicitly specifying the addresses of both trading parties and having both sign the entire transaction), this technology does not rely on the counterparty being online to sign in real time, nor does it require specifying exactly who to trade with. The trading party only needs to sign "half" of the transaction: under what conditions, receiving how much HAC, and paying how much USDT. Then, it is broadcasted to third-party liquidators, who match multiple trading requests and combine them into a single transaction. The profit for the liquidators comes from the reconciliation difference. Of course, liquidators can also act as counterparties to provide liquidity, thereby serving as market maker.

This is an on-chain order book clearing system that relies on Hacash's original single-transaction clearing technology: maintaining a multi-token clearing pool within a single transaction, where different orders use this pool as their counterparty, receiving and spending different tokens. The system checks the token transfers when the transaction goes on-chain and ensures that all tokens have reached their respective accounts and that the token pool balances are zeroed out. Only then is the transaction successfully cleared and confirmed; otherwise, it is rejected from the blockchain.

This trading method does not rely on any unique transaction ID for deduplication, which means the same order can be put on-chain multiple times until the user's one-sided token balance is exhausted. For example, signing an order to buy 100 HACD for 1000 USDT, with an account containing 5000 USDT, means this order can be executed on-chain five times, consuming 5000 USDT and receiving a total of 500 HACD.

Of course, by specifying conditions such as an expiry block height or token balance checks within the order, you can set the transaction's validity period, limit the number of executions or the amount, or make the order single-use only.

Compared with the inherent drawbacks of AMM methods, such as slippage, MEV, and liquidity token locking, this approach provides significant user-friendliness, trading flexibility, and capital efficiency. It also largely avoids the security vulnerabilities of contract programming and could put an end to most of the hacking and token theft incidents that still frequently cause huge losses in the industry today.

It is foreseeable that the creation of TDEX will lay the foundation for the large-scale adoption of next-generation on-chain asset clearing systems.


### AST

This technology originates from the readable contract abstract syntax tree section in the [HIP-16](https://github.com/hacash/doc-chinese/blob/main/HIP/protocol/account_and_syntax_tree_abstraction.md) proposal. This technology can be referred to as a "Conditional Contract" or a "Readable Contract Abstract Syntax Tree."

Specifically, by adding structural Actions such as If and Select that execute transactions conditionally, under the readable contract architecture, it is possible to achieve conditional branches, pattern matching, and other logical semantics found in traditional programming languages, realizing basic and clear programmability.

Compared to Turing-complete programming at the HVM level, the flexibility of the AST approach is limited. However, this brings significant advantages:

1. Security and auditability. Since programming is not implemented using a bytecode virtual machine but instead uses a data structure similar to an abstract syntax tree to define program logic, with strictly limited logic boundaries (e.g., loops are prohibited), it greatly facilitates manual intuitive audits or formal verification of contract security, almost entirely avoiding contract vulnerabilities associated with traditional VM programming.

2. User-friendliness. Without relying on professional programmers, users without technical backgrounds can now create the financial contracts they want by clicking, dragging, and filling out forms, without potential vulnerabilities.

3. AI adaptability. Because the programmable logic of conditional contracts is implemented through very explicit data structures, it becomes possible to send natural language instructions to AI (e.g., "Before this Christmas, request Bob to transfer me 20 HACD, if successful I will pay him 300 USDT, if not, do nothing"), allowing for the generation of completely accurate and unambiguous financial contracts, potentially becoming the main way for ordinary people to use on-chain finance in the future.

4. Cross-chain swap primitives. Since the logic executed on Chain A and Chain B can be defined within the same transaction, with precise control over conditions such as timelocks and token amounts, Layer 3 cross-chain assets will be based on this technology.

The programmability of AST conditional contracts lies between one-dimensional, combinable, transactional readable contracts and HVM smart contracts, but it greatly enhances the safety of on-chain financial operations. This clears a significant barrier for traditional large-scale capital to actively participate in DeFi.

### HVM

Hacash Virtual Machine(HVM) is tailored for secure financial applications with powerful account abstraction and state optimization. Ideal for DeFi, BTCFi, PayFi and stablecoin.

It surpasses traditional blockchain VMs in both smart contract security and architectural clarity, and several classic test cases have already been created:

1. [AMM for swapping Bitcoin SAT and HAC](https://github.com/hacash/fullnode/blob/main/vm/tests/amm.rs)

2. [HACD Fragmented Token Protocol](https://github.com/hacash/fullnode/blob/main/vm/tests/hacds.rs)

3. [HRC20 Smart Contract Token](https://github.com/hacash/fullnode/blob/main/vm/tests/hrc20.rs)

4. [On-chain Contract Inheritance Implementation](https://github.com/hacash/fullnode/blob/main/vm/tests/inherit.rs)

The HVM source code has now been merged into [Fullnode](https://github.com/hacashcom/fullnode). For more information, please visit [https://hacash.com/HVM](https://hacash.com/HVM).


---

If you want to run tests, enable the fullnode source configuration in Cargo.toml at compile time with `default = ["db-sled", "hip20", "tex", "ast", "hvm"]` features. This will activate these technical upgrades when running a local test fullnode.

Finally, we believe this is not just an big upgrade of Hacash, but also represents the forefront of public blockchain technology development in the entire crypto industry. Hacash is entering a brand new phase, and an era of a thriving ecosystem is about to begin.