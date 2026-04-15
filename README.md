# Monero GUI — Fast Crypto Edition

**Author:** Roland Kohlhuber  
**Based on:** [monero-project/monero-gui](https://github.com/monero-project/monero-gui)

---

## What changed

The wallet's innermost cryptographic function `generate_key_derivation()` — responsible for scanning every transaction during sync — has been replaced with a Rust/[curve25519-dalek](https://github.com/dalek-cryptography/curve25519-dalek) implementation via C FFI. The original Monero wallet uses the `ref10` C library from 2014. This fork replaces only the scalar multiplication with a modern, optimized implementation. **Zero changes** to wallet logic, GUI, Ledger support, networking, or protocol code.

Additionally, a macOS 26+ Ledger crash fix has been applied (PAC pointer authentication enforcement).

### Measured performance (Apple M4, 50.000 iterations, 3 runs averaged)

| | Ops/s (1 core) | us/Op | Full Restore (1 core) | Full Restore (4 P-cores) |
|---|---|---|---|---|
| **ref10** (original Monero) | 28.200 | 35.5 | 10.0 min | 2.6 min |
| **curve25519-dalek** (this fork) | **46.460** | **21.5** | **6.1 min** | **1.6 min** |
| **Speedup** | **1.65x** | | **39% faster** | **38% faster** |

Correctness verified: **50.000/50.000 identical results** between ref10 and dalek output.

### Files modified (in the `monero` submodule)

| File | Change |
|------|--------|
| `src/crypto/crypto.cpp` | `generate_key_derivation()` calls `fast_generate_key_derivation()` via C FFI — scalarmult + cofactor x8, entirely in Rust |
| `src/crypto/CMakeLists.txt` | Links `libmonero_fast_crypto.a` |
| `src/cryptonote_config.h` | Increased block batch limits: `MAX_BLOCK_COUNT` 1000→10000, `MAX_RPC_CONTENT_LENGTH` 1MB→20MB. Only effective when connected to a node with matching limits (e.g. [tex8com/cuprate](https://github.com/tex8com/cuprate/tree/fast-rpc)). Against standard nodes, the node's lower limits apply — no negative effect. |
| `src/device/device_io_hid.cpp` | Ledger fix: dispatch HID calls to main thread on macOS (PAC) |
| `external/monero-fast-crypto/` | New: Rust crate wrapping curve25519-dalek, exports C API |
| `share/Info.plist` | Added `NSCameraUsageDescription` for macOS QR scanner |

---

## Building

### Prerequisites

```bash
# macOS
brew install boost cmake libsodium miniupnpc zeromq hidapi pkg-config qt@5
# Rust (for building the fast crypto library)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### Build

```bash
# 1. Clone with submodules
git clone --recursive -b fast-crypto https://github.com/tex8com/monero-gui.git
cd monero-gui

# 2. Build the Rust crypto library
cd monero/external/monero-fast-crypto
cargo build --release
cd ../../..

# 3. Build the GUI
mkdir -p build/release && cd build/release
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=$(brew --prefix qt@5) ../..
make -j$(sysctl -n hw.ncpu)

# 4. Run
open bin/monero-wallet-gui.app
```

### Ledger Support

Requires `hidapi` and `libusb` (installed via brew above). Connect your Ledger, open the Monero app, then select "Hardware Wallet" in the GUI.

**macOS 26+ (Tahoe):** The Ledger PAC crash fix is included — no additional steps needed.

## Note on real-world speedup

The 1.65x improvement applies to the **cryptographic scanning** (the `ge_scalarmult` operation). When connected to a **remote node**, network latency dominates — the wallet spends ~90% of its time waiting for blocks, not computing. The speedup is most noticeable during:

- Full wallet restore with a **local node**
- Scanning large numbers of blocks after being offline

## License

Same as upstream Monero: BSD-3-Clause. The Rust library (`monero-fast-crypto`) is MIT.

## Credits

- [The Monero Project](https://github.com/monero-project) for the original wallet
- [dalek-cryptography](https://github.com/dalek-cryptography/curve25519-dalek) for the fast Ed25519 implementation
