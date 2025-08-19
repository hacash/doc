# Paper
Whitepaper & Documents & Hacash improvement proposals & theoretical articles


- [Whitepaper](https://github.com/hacash/doc/blob/main/whitepaper.pdf)
- [HIP Table (Hacash improvement proposals list)](https://github.com/hacash/doc/blob/main/HIP/HIP-table.md)

---

### Software download & release log

- [Fullnode & PoWorker](https://github.com/hacash/rust/releases)
- [Configuration description](https://github.com/hacash/doc/blob/main/build/config_description.md)

### Compilation build instructions 

---

Compilation build instructions: 

- [Build_compilation.md](https://github.com/hacash/doc/blob/main/build/build_compilation.md)

---

### SDK & RPC & API Doc 

- [SDK_doc.md](https://github.com/hacash/doc/tree/main/server/sdk_doc.md) 
- [Rust_fullnode_api_doc.md](https://github.com/hacash/doc/blob/main/server/fullnode_api_doc.md) 
- [Memtxpool_operation_important_note.md](https://github.com/hacash/doc/blob/main/server/memtxpool_operation_important_note.md) 
- [HACD_explain_for_exchange.md](https://github.com/hacash/doc/blob/main/server/hacd_explain_for_exchange.md) 

---

### X16RS algorithm design explanation

- [X16RS_algorithm_description.md](https://github.com/hacash/doc/blob/main/tech/x16rs_algorithm_description.md)

---

### HAC & HACD mining fairness description

- [HAC_HACD_mining_fairness_description.md](https://github.com/hacash/doc/blob/main/tech/HAC_HACD_mining_fairness_description.md)

---

### Project code engineering architecture

The architecture of Hacash full node code is divided into 7 levels from bottom to top:

X16RS -> Core -> Chain -> Mint -> Node -> Server -> Miner

Each layer of the architecture has independent functions and responsibilities for the upper layer to call, and the implementation of the lower layer to the upper layer is unknown. The responsibilities of each layer are roughly as follows:

1. [X16RS] Basic algorithm - including HAC mining, block diamond mining, GPU version algorithm, etc.
2. [Core] Core - block structure definition, interface definition, data serialization and deserialization, storage object, field format, genesis block definition, etc.
3. [Chain] Chain - underlying database, block and transaction storage, blockchain state storage, logs, etc.
4. [Mint] Mint - block mining difficulty adjustment algorithm, coinbase definition, block construction, transaction execution and status update, etc.
5. [Node] Node - P2P underlying module, Backend blockchain synchronization terminal, point-to-point network message definition and processing, etc.
6. [Server] Server - RPC API interface service, block and transaction and account data query, other services, etc.
7. [Miner] Miner - block construction and mining, diamond mining, transaction memory pool, mining pool server, mining pool worker, etc.

