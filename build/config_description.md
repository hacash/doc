# Description of the configuration file

Hacash's full nodes, mining terminals, or other executables all use simple configuration files in the format of '.ini' to set parameters and enable or disable certain functions. Each of these profiles will be described in detail in this document.

In general, a basic configuration file will be attached to the compiled and downloaded package, and some basic configurations will be filled in. For example, a full node, you can run it directly from the command line without specifying the path of the configuration file, and the program will automatically load the configuration file in the same directory, such as 'hacash.config.ini' (Fullnode) or 'poworker.config.ini' (Mining worker).

You can also switch between these configurations without having to change the configuration each time:

```bash

# Loads and uses the configuration with the default name of current directory hacash.config.ini
./hacash_fullnode

# Specify the profile
./hacash_fullnode hacash.config.ini
./hacash_fullnode ../../hacash_mainnet.config.ini
./hacash_fullnode ./configs/hacash2.config.ini

```

## Fullnode configuration

```ini

;;; Directory of all block data and account state data
data_dir = ./hacash_mainnet_data

;;; A group of configurations, P2P network connectivity and synchronization aspects
[node]

;;; The name of the node that appears in the network connection
name = rust_node ; hn_usa or any

;;; The port that the node listens on, and uses this port when other nodes connect to you
listen = 3337

;;; The entry nodes at the start of the full node, through which they can join the entire decentralized network
;;; These nodes are only used to boot, not to connect all the time. The network automatically explores to continuously connect and disconnect different full nodes, maintaining the decentralized structure of the network
boots = 54.193.49.59:3337, 182.92.163.225:3337, 54.219.80.127:3337

;;; true or false, which indicates that the block data is synchronized as quickly as possible, avoiding some security checks and data proofreading
;;; Note that this is only recommended if you are synchronizing all blocks in history from scratch, and should not be enabled if you are running normally or if the block may face forks or other inconsistencies
fast_sync = false

;;; Do not search for nodes on the whole network, only connect to boot nodes, and it is generally not recommended to enable them
;;; This is often used for local testing, or for quickly synchronizing block data between two local nodes
not_find_nodes = false


;;; HTTP API Server
[server]

;;; true or false indicates that the api service for the full node is enabled
enable = true

;;; The port on which the HTTP service listens, such as accessing the API via http://127.0.0.1:8081
listen = 8081


;;; Mining configurations
[miner]

;;; true or false indicates that mining support is enabled on full nodes
;;; This is required for both HAC and HACD mining to enable full node transaction pooling or to pack new blocks
enable = true

;;; If the HAC mining is successful, the address of the block reward
reward = 1AVRuFXNFi3rdMrPH4hdqSgFrEBnWisWaS

;;; The announcer of the block, or the name of the mining pool
message = HacPool ; my_pool_node



```

## PoWorker HAC Mining configuration


The miner maintains a connection to the whole network through the API interface of the full node, obtains new blocks to be mined, listens for the next block, or submits successfully mined blocks.

Each full node can mount many mining terminals to form a mining cluster. You can connect via both local and public networks.

Please note that in the past, the mining side had to be connected to a separate TCP protocol mining pool port, while the Rust version of the mining side adopts a full node interface in the ordinary HTTP way, which is more versatile and robust, can support more abundant programming languages and devices, develop mining pool programs and more versions of the mining side, and is more convenient to integrate into the third-party mining ecosystem and tool system.


```ini

;;; The full node API service IP address and port of the connection
connect = 127.0.0.1:8081

;;; Currently, only CPU device mining is supported
;;; Multi-threaded mining, using CPU multi-core parallel mining.
supervene = 2

```





