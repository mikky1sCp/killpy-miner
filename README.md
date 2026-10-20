# killpy-miner

High-performance external miner for **Quantus Network**. Fork of the official
[`quantus-miner`](https://github.com/Quantus-Network/quantus-miner) with
rebranded tooling and a Windows-first release.

**Wire-compatible with Quantus nodes** — same QUIC protocol, same ALPN
(`quantus-miner/2`), same auth-token / TLS-pin file conventions. Drop-in
replacement for `quantus-miner serve`; no node-side changes needed.

## Downloads

Latest release: **[v0.1.0](https://github.com/mikky1sCp/killpy-miner/releases/tag/v0.1.0)**

| Platform | Binary |
|---|---|
| Windows 10/11 x64 | `killpy-miner-windows-x86_64.exe` |
| Linux x86_64 | _not in this release — build from source_ |
| macOS arm64 | _not in this release — build from source_ |

## Requirements

### All platforms
- A running **Quantus node** with the external-miner endpoint enabled
  (`--miner-listen-port 9833` or default). The node is what talks to the
  network; the miner only talks to the node.
- Two files from the node's chain config directory
  (`<base-path>/chains/<chain>/`):
  - `miner-auth-token` — shared secret
  - `miner-tls-cert-sha256` — SHA-256 fingerprint of the node's miner TLS cert

### GPU mining
- **NVIDIA / AMD / Intel**: any GPU with a working Vulkan or DX12 driver.
  The default `wgpu` path needs no extra software.
- **Native CUDA** (`--cuda-gpu`, NVIDIA only): requires `libnvrtc` from the
  CUDA Toolkit. NVRTC is **not** shipped with the display driver.
  - Windows: install the [CUDA Toolkit](https://developer.nvidia.com/cuda-downloads)
    (12.x) or drop `nvrtc64_*.dll` + `nvrtc-builtins64_*.dll` next to the
    executable.
  - Linux: `apt install nvidia-cuda-toolkit`, or set `LD_LIBRARY_PATH` to
    the directory containing `libnvrtc.so.12`.

## Quick start

### CPU mining

```cmd
killpy-miner-windows-x86_64.exe serve ^
  --node-addr 127.0.0.1:9833 ^
  --auth-token-file C:\path\to\miner-auth-token ^
  --tls-cert-sha256-file C:\path\to\miner-tls-cert-sha256 ^
  --cpu-workers 4
```

### GPU mining (NVIDIA, native CUDA — fastest)

```cmd
killpy-miner-windows-x86_64.exe serve ^
  --node-addr 127.0.0.1:9833 ^
  --auth-token-file C:\path\to\miner-auth-token ^
  --tls-cert-sha256-file C:\path\to\miner-tls-cert-sha256 ^
  --cuda-gpu ^
  --gpu-devices 1 ^
  --cpu-workers 0
```

### GPU mining (AMD / Intel / NVIDIA without CUDA Toolkit)

Omit `--cuda-gpu` — the default `wgpu` path uses Vulkan (Linux/Windows) or
Metal (macOS):

```cmd
killpy-miner-windows-x86_64.exe serve ^
  --node-addr 127.0.0.1:9833 ^
  --auth-token-file C:\path\to\miner-auth-token ^
  --tls-cert-sha256-file C:\path\to\miner-tls-cert-sha256 ^
  --gpu-devices 1 ^
  --cpu-workers 0
```

### Hybrid CPU + GPU

```cmd
killpy-miner-windows-x86_64.exe serve ^
  --node-addr 127.0.0.1:9833 ^
  --auth-token-file C:\path\to\miner-auth-token ^
  --tls-cert-sha256-file C:\path\to\miner-tls-cert-sha256 ^
  --cuda-gpu ^
  --gpu-devices 1 ^
  --cpu-workers 4
```

### Environment variables

Every flag has an env-var equivalent (prefix `KILLPY_`), handy for service
managers and Docker:

```cmd
set KILLPY_NODE_ADDR=127.0.0.1:9833
set KILLPY_AUTH_TOKEN_FILE=C:\path\to\miner-auth-token
set KILLPY_TLS_CERT_SHA256_FILE=C:\path\to\miner-tls-cert-sha256
set KILLPY_CUDA_GPU=1
set KILLPY_CPU_WORKERS=4

killpy-miner-windows-x86_64.exe serve
```

### Full options

```cmd
killpy-miner-windows-x86_64.exe serve --help
```

| Flag | Env var | Default | Description |
|---|---|---|---|
| `--node-addr` | `KILLPY_NODE_ADDR` | `127.0.0.1:9833` | Node address |
| `--auth-token-file` | `KILLPY_AUTH_TOKEN_FILE` | — | Path to `miner-auth-token` |
| `--tls-cert-sha256-file` | `KILLPY_TLS_CERT_SHA256_FILE` | — | Path to `miner-tls-cert-sha256` |
| `--cpu-workers` | `KILLPY_CPU_WORKERS` | half of logical cores | CPU threads |
| `--gpu-devices` | `KILLPY_GPU_DEVICES` | auto-detect | GPU count |
| `--cuda-gpu` | `KILLPY_CUDA_GPU` | off | Native CUDA (NVIDIA) |
| `--gpu-batch-size` | `KILLPY_GPU_BATCH_SIZE` | 1M (32M with CUDA) | Nonces per dispatch |
| `--cpu-batch-size` | `KILLPY_CPU_BATCH_SIZE` | 10000 | Hashes between cancel checks |
| `--gpu-throttle-ms` | `KILLPY_GPU_THROTTLE_MS` | 0 | Sleep between GPU batches |
| `--metrics-port` | `KILLPY_METRICS_PORT` | 9900 | Prometheus HTTP port |
| `--allow-integrated` | `KILLPY_ALLOW_INTEGRATED` | off | Use iGPU with dGPU present |
| `-v, --verbose` | `KILLPY_VERBOSE` | off | Debug logging |

## Algorithm

`killpy-miner` mines **QPoW** (Quantum Proof-of-Work), the consensus algorithm
of Quantus Network. Unlike Bitcoin's SHA-256, QPoW is built on **Poseidon2**,
an algebraic hash that is efficient inside ZK circuits.

| Property | Value |
|---|---|
| Hash function | Poseidon2 over the Goldilocks field (p = 2⁶⁴ − 2³² + 1) |
| State width | 12 field elements (sponge) |
| Full hash | 512 bits (two 256-bit squeezes) |
| Permutations per nonce | 3 full (2 cached in midstate) |
| Round structure | 4 initial external + 22 internal + 4 terminal external |
| Block time (network) | ~12 seconds |
| Total supply | 21,000,000 QTC |

The miner absorbs the 32-byte block header and the **high 256 bits** of the
nonce once (midstate), then iterates the **low 256 bits**. This is why batches
are clamped at `2⁶⁴` (CUDA) / `2²⁵⁶` (wgpu) — a carry into the cached part
would invalidate the midstate.

### Mining paths

| Path | What it does | Best for |
|---|---|---|
| `cpu` | Reference Poseidon2 in Rust, midstate cache, early-exit on first 256 bits | Small rigs, background mining |
| `gpu-wgpu` | WGSL shaders via `wgpu`: Vulkan / DX12 / Metal | AMD, Intel, Apple, NVIDIA without CUDA Toolkit |
| `gpu-cuda` | Native CUDA kernel via NVRTC, `__constant__` round tables, `reduce128` | NVIDIA (fastest) |

## Hashrate

Measured on rented hosts (Clore.ai, Vast.ai) and local hardware. Same-host
comparisons only — do not compare across different machines.

### CPU (reference Poseidon2)

| CPU | Workers | Per-worker | Total |
|---|---:|---:|---:|
| Windows desktop (tested) | 3 | ~12.4 MH/s | ~37.1 MH/s |

CPU mining is dominated by Poseidon2 scalar math. On a modern desktop expect
**~10–20 MH/s per physical core** depending on the microarchitecture. No SIMD
path is enabled in this release (`simd-poseidon2` feature is a stub upstream).

### NVIDIA — native CUDA (`--cuda-gpu`)

| GPU | Hashrate | Notes |
|---|---:|---|
| RTX 3060 Ti | 68 MH/s | Clore, driver 550.144, 200 W |
| RTX 3070 | 81 MH/s | Clore, driver 550.144 |
| RTX 4060 Ti | 79 MH/s | Clore, 100 W cap |
| RTX 5060 Ti | 105 MH/s | Clore, driver 610.43 |
| RTX 3080 Ti | **380 MH/s** | Vast, driver 580.173, 330 W |
| RTX 4090 | **818 MH/s** | Vast, driver 580.159, 350 W |

These numbers are from the fork's native CUDA kernel after the
`reduce128` / `__constant__` optimisation pass (PR #100 upstream). The
baseline `v4.1.1` release was ~30% slower on the same cards.

### NVIDIA — wgpu (`wgpu` default path)

| GPU | Hashrate | Notes |
|---|---:|---|
| RTX 3080 Ti | 106 MH/s | Vast, Vulkan, official `v4.0.2` binary |
| RTX 5080 | 21 MH/s | Clore, 30 s sample, older kernel |
| RTX 5060 Ti | 10 MH/s | Clore, older kernel |

CUDA is **~3–4× faster than wgpu on the same NVIDIA card**. If you have an
NVIDIA GPU and can install the CUDA Toolkit, use `--cuda-gpu`.

### AMD / Intel / Apple

No same-host numbers are recorded in this release. The `wgpu` path works on
Vulkan (AMD/Intel Linux/Windows) and Metal (Apple Silicon), but performance
varies widely by driver. If you benchmark a card, open an issue with the
result — we collect these into `docs/benchmarks/`.

### GTX 1660 Super

Community reports ~110–128 MH/s on the native CUDA path (same ballpark as
RTX 3060 Ti). This is **not** a same-host measurement from our benchmarks —
verify with `benchmark --cuda-gpu` on your card before trusting it for
planning.

## Benchmarking

No node needed — the benchmark uses a random header and maximum difficulty:

```cmd
:: CPU
killpy-miner-windows-x86_64.exe benchmark --cpu-workers 8 --duration 30

:: wgpu (default GPU path)
killpy-miner-windows-x86_64.exe benchmark --gpu-devices 1 --duration 30

:: CUDA (fastest on NVIDIA)
killpy-miner-windows-x86_64.exe benchmark --cuda-gpu --gpu-devices 1 --cpu-workers 0 --duration 30

:: Hybrid
killpy-miner-windows-x86_64.exe benchmark --cuda-gpu --gpu-devices 1 --cpu-workers 4 --duration 30
```

The benchmark prints `Average rate: X.XXM H/s`. Run for at least 30 seconds
on GPU to skip warmup effects.

## Metrics

Prometheus endpoint on `:9900/metrics`. All metric names are prefixed
`killpy_`:

| Metric | Type | Description |
|---|---|---|
| `killpy_hash_rate` | gauge | Total hash rate (CPU + GPU) |
| `killpy_cpu_hash_rate` | gauge | CPU-only hash rate |
| `killpy_gpu_hash_rate` | gauge | GPU-only hash rate |
| `killpy_hashes_total` | counter | Total hashes computed |
| `killpy_active_jobs` | gauge | Currently running jobs (0 or 1) |
| `killpy_workers` | gauge | Total worker count |
| `killpy_cpu_workers` | gauge | CPU worker count |
| `killpy_gpu_devices` | gauge | GPU device count |
| `killpy_effective_cpus` | gauge | Detected logical cores |

Quick check:

```powershell
curl http://127.0.0.1:9900/metrics | Select-String killpy_
```

## Building from source

```powershell
# Windows
rustup default stable-x86_64-pc-windows-msvc
cargo build -p miner-cli --release
# → target\release\killpy-miner.exe
```

```bash
# Linux / macOS
cargo build -p miner-cli --release
# → target/release/killpy-miner
```

The CUDA kernel is compiled **at runtime** by NVRTC — no CUDA SDK is needed
at build time. The `wgpu` shaders are embedded in the binary.

## Known limitations

- **Windows-only release.** Linux and macOS binaries are not part of
  `v0.1.0`; build from source if you need them.
- **CUDA `reduce128` skips two carry corrections** per field multiply
  (~1 in 3M nonces hashes wrong on GPU). Every candidate is re-verified on
  CPU, so no invalid blocks are ever produced — but a rare valid nonce may
  be missed. This is the same trade-off the upstream miner makes.
- **wgpu `hash_count` is approximate** when multiple threads find candidates
  in one batch. Affects metrics only, not correctness.
- **`simd-poseidon2` is a stub.** The CPU path is scalar; enabling the
  feature does nothing in this release.

## License

MIT OR Apache-2.0, inherited from upstream `quantus-miner`.

## Credits

Fork of [Quantus-Network/quantus-miner](https://github.com/Quantus-Network/quantus-miner).
QPoW, Poseidon2 parameters, and the wire protocol are upstream work.
