# Compilation instructions (linux-like platforms)

Hacash in written in Rust with a crypto hash library X16RS written in C for performance and stability.
There are also some web interface and explorer functionality written in javascript, however that is
only required for the web site functionality or if you want to implement your own block explorer.
This guide is currently only for linux-like platforms.

The Rust source is portable across a wide range of platforms and has been tested
on linux ubuntu, centos, macosx (with homebrew) and armhf (raspberry pi. ).

In principle Hacash should work on any platform which supports Rust, and can compile the
X16RS library.  The author has used gcc >= 6.3 and cmake >= 3.7.2 and Rust >= 1.80 however your mileage may vary.

## Outline to compile Hacash

1. Install c compiler & cmake (gcc >=6.3 is known to be good)
2. Install Rust (Rustup is recommended)
3. Download the source from Github

Install depend compiler:

```bash
sudo apt install g++
sudo apt install cmake     
```

Clone the codes:

```bash
mkdir -p ~/hacash
cd ~/hacash
git clone https://github.com/hacash/rust.git
done
```

You can move the source to any directory you want to place, You can also update the git source at any time.

## Compile the Fullnode

The whole project follows the usual way of compiling a Rust project, and it's very simple:

```bash
cd ~/hacash/rust
cargo build --release
```

If you want to ignore those warnings, get a clean compilation interface:

```bash
RUSTFLAGS="$RUSTFLAGS -Awarnings" cargo build --release
```

If you get a fatal error here, then the most likely thing is that the C compiler or the CMake development package is not installed.

## Compile the PoWorker

In the Rust version of the full node, there is no HAC or HACD mining functionality, they are moved out to a separate executable file that communicates with the full node via an HTTP interface.

To compile a separate HAC miner, you need to modify `src/main.rs` file like this:

```rust
use crate::run::*;

fn main() {

    // poworker(); // PoW Miner Worker
    fullnode(); // Hacash Full Node

}
```

Turn on the comments in front of powoker and cancel the fullnode function:

```rust
use crate::run::*;

fn main() {

    poworker(); // PoW Miner Worker
    // fullnode(); // Hacash Full Node

}
```

In this way, the compilation result can be obtained as small as possible, which is convenient for a large number of mining machines to deploy.

---

## Run the Fullnode or PoW worker

If you want to run a fullnode or miner worker, check out the [Hacash Configuration Description](https://github.com/hacash/doc/blob/main/build/config_description.md).





