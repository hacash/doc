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
;;; If want to mine HAC or HACD, must open the server
enable = true

;;; The port on which the HTTP service listens, such as accessing the API via http://127.0.0.1:8081
listen = 8081


;;; HAC Mining configurations
[miner]

;;; true or false indicates that mining support is enabled on full nodes
;;; This is required for both HAC and HACD mining to enable full node tx pool or to pack new blocks
enable = true

;;; If the HAC mining is successful, the address of the block reward
reward = 1AVRuFXNFi3rdMrPH4hdqSgFrEBnWisWaS

;;; The announcer of the block, or the name of the mining pool
message = HacPool ; my_pool_node


;;; HACD Mining configurations
[diamondminer]

;;; true or false indicates that mining support is enabled on full nodes
;;; This is required for both HAC and HACD mining to enable full node tx pool or to pack new blocks
enable = true

;;; If the HACD mining is successful, the address of the reward
reward = 1AVRuFXNFi3rdMrPH4hdqSgFrEBnWisWaS

;;; The private key or password of the bidding account used for HACD mining
bid_password = pass123456 ; or private key of hex

;;; The minimum bid for the HACD auction
bid_min = 0.0001 ; or 1:244

;;; The maximum bid for the HACD auction
bid_max = 25:248 ; or 25.0

;;; The magnitude of the increase per quote
bid_step = 1:248 ; or 1.0


```

## `PoWorker` HAC Mining configuration


The miner maintains a connection to the whole network through the API interface of the full node, obtains new blocks to be mined, listens for the next block, or submits successfully mined blocks.

Each full node can mount many mining terminals to form a mining cluster. You can connect via both local and public networks.

Please note that in the past, the mining side had to be connected to a separate TCP protocol mining pool port, while the Rust version of the mining side adopts a full node interface in the ordinary HTTP way, which is more versatile and robust, can support more abundant programming languages and devices, develop mining pool programs and more versions of the mining side, and is more convenient to integrate into the third-party mining ecosystem and tool system.


```ini

;;; The full node API service IP address and port of the connection
connect = 127.0.0.1:8081

;;; Multi-threaded mining, using CPU multi-core parallel mining.
supervene = 2


;;; OpenCL GPU configurations
[gpu]

;;; Enable or disable GPU mining via OpenCL
# true  = use the OpenCL GPU miner
# false = fallback to CPU mining
use_opencl = false

;;; Number of work-groups to launch
;;; Increasing this increases total parallelism, but also GPU load
;;; total_threads = work_groups × local_size
work_groups = 1024

;;; Number of work-items per work-group
;;; Must match hardware constraints. Common values: 128, 256
;;; Higher values generally improve occupancy but use more local memory
local_size = 256

;;; Number of nonces processed per work-item.
;;; Larger values increase parallel hashing within each thread, improving performance
;;; Typical values: 32–512 depending on GPU memory and architecture
unit_size = 128

;;; Path to the directory containing the OpenCL kernel (.cl) files
;;; IMPORTANT: the path must end with a slash or backslash (e.g. "opencl/")
;;; Can be relative or absolute
opencl_dir = opencl/

;;; ID of the OpenCL platform to use
;;; Try setting this value to 1 if you encounter compilation errors
platform_id = 0

;;; Comma-separated list of GPU device IDs to use (e.g. "0,1,2")
;;; Leave empty to automatically use all available GPU devices on the selected platform
device_ids =

```



## `DiaWorker` HACD Mining configuration


Miners stay connected to the entire network through the full node's API interface, get new HACDs to be mined, or listen to switch to the next HACD.

Multiple machines can be connected to a full node at the same time to mine, through the public network or local area network connection. The hashrate will be stacked.


```ini

;;; The full node API service IP address and port of the connection
connect = 127.0.0.1:8081

;;; Currently, only CPU device mining is supported
;;; Multi-threaded mining, using CPU multi-core parallel mining.
supervene = 2

```








