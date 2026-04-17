# Monero GUI — Fast Crypto Edition

**Author:** Roland Kohlhuber  
**Based on:** [monero-project/monero-gui](https://github.com/monero-project/monero-gui)  
**Optimized Node:** [tex8com/cuprate](https://github.com/tex8com/cuprate/tree/fast-rpc) — use both together for maximum sync speed

---

## What changed

### 1. Fast Crypto (1.65x faster scanning)

The wallet's innermost cryptographic function `generate_key_derivation()` — responsible for scanning every transaction during sync — has been replaced with a Rust/[curve25519-dalek](https://github.com/dalek-cryptography/curve25519-dalek) implementation via C FFI. The original Monero wallet uses the `ref10` C library from 2014. This fork replaces only the scalar multiplication with a modern, optimized implementation. **Zero changes** to wallet logic, GUI, Ledger support, or protocol code.

### 2. gzip RPC Compression (2-3x less network transfer)

The wallet sends `Accept-Encoding: gzip` in HTTP requests and decompresses gzip responses from the node. When connected to a gzip-capable node (like [tex8com/cuprate](https://github.com/tex8com/cuprate/tree/fast-rpc)), block data is compressed before transfer — reducing bandwidth usage by 2-3x. **Fully backwards compatible:** against standard nodes (monerod) that don't support gzip, responses arrive uncompressed as usual.

### 3. Increased Block Batch Limits (50 MB per batch)

Block batch limits have been increased (`MAX_BLOCK_COUNT` 1000→10000, `MAX_RPC_CONTENT_LENGTH` 1MB→50MB). This allows downloading more blocks per request when connected to a node with matching limits (like [tex8com/cuprate](https://github.com/tex8com/cuprate/tree/fast-rpc)). Against standard nodes, the node's lower limits apply — no negative effect.

### 4. macOS 26+ Ledger Crash Fix

Ledger HID calls are dispatched to the main thread on macOS (PAC pointer authentication enforcement).

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
| `src/cryptonote_config.h` | Increased block batch limits: `MAX_BLOCK_COUNT` 1000→10000, `MAX_RPC_CONTENT_LENGTH` 1MB→50MB |
| `src/device/device_io_hid.cpp` | Ledger fix: dispatch HID calls to main thread on macOS (PAC) |
| `external/monero-fast-crypto/` | New: Rust crate wrapping curve25519-dalek, exports C API |
| `share/Info.plist` | Added `NSCameraUsageDescription` for macOS QR scanner |
| `contrib/epee/include/net/http_client.h` | Sends `Accept-Encoding: gzip`, accepts gzip Content-Encoding |
| `contrib/epee/include/storages/http_abstract_invoke.h` | zlib gzip decompression in `invoke_http_bin()` before deserialization |
| `contrib/epee/src/CMakeLists.txt` | Links zlib (`z`) |

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

## Best performance: use with Cuprate node

For maximum sync speed, connect this wallet to [tex8com/cuprate (fast-rpc branch)](https://github.com/tex8com/cuprate/tree/fast-rpc):

| Setup | Initial Sync (45K blocks) | Block Scan |
|-------|---------------------------|------------|
| This wallet + standard monerod | 19s | network-limited |
| This wallet + **Cuprate (fast-rpc)** | **4s** | **gzip compressed, 50MB batches** |
| Standard wallet + Cuprate | 12s | no gzip (still fast) |

The combined optimizations:
- **gzip compression** — 2-3x less data over the network (only when both sides support it)
- **50MB batch limits** — fewer HTTP round-trips
- **On-the-fly TX pruning** — node strips RCT prunable data before sending
- **Batch output-index lookups** — node computes indices 46x faster
- **1.65x faster crypto** — wallet scans transactions faster

## License

Same as upstream Monero: BSD-3-Clause. The Rust library (`monero-fast-crypto`) is MIT.

## Credits

- [The Monero Project](https://github.com/monero-project) for the original wallet
- [dalek-cryptography](https://github.com/dalek-cryptography/curve25519-dalek) for the fast Ed25519 implementation
