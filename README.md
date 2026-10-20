# Killpy Miner — External Miner for Quantus Network

High-performance external mining service for Quantus Network with CPU, GPU, and
hybrid CPU+GPU mining. Killpy is a fork of the official `quantus-miner` that
stays wire-compatible with Quantus nodes: no Python, no Docker, no subprocess
wrappers — just a Rust binary plus whatever CUDA/Vulkan driver your GPU needs.

## Building

```bash
# CPU-only build (default)
cargo build -p miner-cli --release

# With GPU support (recommended)
cargo build -p miner-cli --release
```

The binary is emitted at `target/release/killpy-miner`.

## Running

The node requires a shared auth token and TLS cert pin. Both live under the
node's chain config dir (`<base-path>/chains/<chain>/`):

- `miner-auth-token` — shared secret (read this file)
- `miner-tls-cert-sha256` — SHA-256 of the miner TLS cert (also printed in
  node logs)

```bash
# Preferred: read the node's chain config files
./target/release/killpy-miner serve \
  --node-addr 127.0.0.1:9833 \
  --auth-token-file /path/to/miner-auth-token \
  --tls-cert-sha256-file /path/to/miner-tls-cert-sha256 \
  --cpu-workers 4
```

## Configuration

| Argument | Environment Variable | Description | Default |
|----------|---------------------|-------------|---------|
| `--node-addr <ADDR>` | `KILLPY_NODE_ADDR` | Node address to connect to | `127.0.0.1:9833` |
| `--auth-token <TOKEN>` | `KILLPY_AUTH_TOKEN` | Shared secret (not logged) | required |
| `--auth-token-file <PATH>` | `KILLPY_AUTH_TOKEN_FILE` | Read the shared secret from a file (preferred) | — |
| `--tls-cert-sha256 <HEX>` | `KILLPY_TLS_CERT_SHA256` | SHA-256 of the node's miner TLS cert | required |
| `--tls-cert-sha256-file <PATH>` | `KILLPY_TLS_CERT_SHA256_FILE` | Read the fingerprint from a file | — |
| `--cpu-workers <N>` | `KILLPY_CPU_WORKERS` | Number of CPU worker threads | Auto-detect |
| `--gpu-devices <N>` | `KILLPY_GPU_DEVICES` | Number of GPU devices | Auto-detect |
| `--cuda-gpu` | `KILLPY_CUDA_GPU` | Use native CUDA instead of wgpu/Vulkan (NVIDIA) | off |
| `--gpu-batch-size <N>` | `KILLPY_GPU_BATCH_SIZE` | GPU batch size in nonces | 1000000 (32000000 with `--cuda-gpu`) |
| `--cpu-batch-size <N>` | `KILLPY_CPU_BATCH_SIZE` | CPU batch size in hashes | 10000 |
| `--gpu-throttle-ms <MS>` | `KILLPY_GPU_THROTTLE_MS` | Sleep (ms) between GPU batches | 0 |
| `--metrics-port <PORT>` | `KILLPY_METRICS_PORT` | Prometheus metrics port | 9900 |
| `--allow-integrated` | `KILLPY_ALLOW_INTEGRATED` | Use iGPUs even when a discrete GPU is present | off |
| `-v, --verbose` | `KILLPY_VERBOSE` | Verbose logging | off |

## GPU Mining

Two GPU paths:

- **wgpu** (default): Metal on macOS, Vulkan/DX12 elsewhere.
- **CUDA** (`--cuda-gpu`): native NVIDIA compute via NVRTC. No Vulkan ICD needed,
  which makes it the right choice on headless CUDA-only boxes (Clore.ai, Vast.ai).

Same-host measured on an RTX 3080 Ti: wgpu/WGSL ~106 MH/s, native CUDA ~380 MH/s
after the `reduce128`/constant-memory pass. On an RTX 4090, CUDA reaches ~820 MH/s.

```bash
./target/release/killpy-miner benchmark --cuda-gpu --gpu-devices 1 --cpu-workers 0 --duration 10
```

### Platform requirements

- **macOS**: works out of the box (Metal)
- **Linux**: `nvidia-driver` + `libcuda`, `libnvrtc` from the CUDA toolkit for
  `--cuda-gpu`; `mesa-vulkan-drivers` for the wgpu path on AMD/Intel
- **Windows**: current graphics drivers; DX12 for wgpu, CUDA toolkit for `--cuda-gpu`

### Performance monitoring

- **macOS**: `sudo powermetrics --samplers gpu_power -i 1000`
- **Linux**: `nvidia-smi` (NVIDIA) or `radeontop` (AMD)
- **Windows**: Task Manager → GPU tab

## Metrics

Prometheus endpoint on `:9900/metrics`, all metrics prefixed with `killpy_`:

- `killpy_hash_rate`, `killpy_cpu_hash_rate`, `killpy_gpu_hash_rate`
- `killpy_hashes_total`, `killpy_active_jobs`
- `killpy_workers`, `killpy_cpu_workers`, `killpy_gpu_devices`, `killpy_effective_cpus`

## Protocol

Killpy speaks the same wire protocol as the official miner:

- **Transport**: QUIC with TLS 1.3 (self-signed certificate, pinned by SHA-256)
- **Auth**: `Ready { token }` must match the node's `miner-auth-token`
- **ALPN**: `quantus-miner/2`
- **Port**: 9833 (default)
- **Messages**: `Ready` (miner→node), `NewJob` (node→miner), `JobResult` (miner→node)

For the full protocol specification, see the node's `MINING.md`.

## Benchmarks

Recorded same-host runs live under `docs/benchmarks/`. Comparison method and
rejected CUDA variants are documented in `docs/clore-gpu-benches.md` and
`docs/vast-gpu-benches.md`.

## Benchmarking

```bash
# Benchmark CPU performance
./target/release/killpy-miner benchmark --cpu-workers 8 --duration 30

# Benchmark GPU performance
./target/release/killpy-miner benchmark --gpu-devices 1 --duration 30

# Benchmark hybrid performance
./target/release/killpy-miner benchmark --cpu-workers 4 --gpu-devices 1 --duration 30
```

## License

MIT OR Apache-2.0 (inherited from the upstream `quantus-miner` workspace).
```

---