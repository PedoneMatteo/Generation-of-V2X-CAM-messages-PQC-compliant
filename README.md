# CamGen_PQC — Post-Quantum Cryptography for V2X Communications

**CamGen_PQC** is a research project that integrates **Post-Quantum Cryptography (PQC)** into **V2X (Vehicle-to-Everything)** communication security. It extends the standard ETSI ITS message generation and certificate infrastructure to support **CRYSTALS-Dilithium** digital signatures alongside classical ECDSA, providing a migration path toward quantum-resistant vehicular networks.

Developed as a Master's thesis at *Politecnico di Torino* in collaboration with *Nardò Technical Center — Porsche Engineering*.

---

## Table of Contents
- [Architecture](#architecture)
- [Results](#results)
- [Usage](#usage)
  - [Docker Setup](#docker-setup)
  - [Build Order](#build-order-inside-container)
  - [Generate Certificates](#generate-certificates)
  - [Run the Message Generator](#run-the-message-generator)
- [Repository Structure](#repository-structure)
- [Limitations & Future Work](#limitations--future-work)

---

## Architecture

The project chains together **5 tools** into a single pipeline:

```
asn1c → liboqs → itscertgen → fsmsggen
  │        │          │           │
  │        │          │           └── V2X message generator (CAM/DENM/PKI)
  │        │          └── Certificate generator (RCA/AA/AT)
  │        └── Open Quantum Safe library (Dilithium, Falcon, SPHINCS+)
  └── ASN.1 compiler (generates C structures from .asn specs)
```

| Component | Role | Key Files |
|-----------|------|-----------|
| **fsmsggen** | Generates and injects secured V2X messages (CAM, DENM, PKI) onto Ethernet via PCAP. Toggles ECDSA/Dilithium via `flag_PQC`. | `fsmsggen.c`, `extensions.c/h`, `msggen_cam.c`, `load_data.c` |
| **itscertgen** | Generates ETSI TS 103 097 certificates (OER format) with both ECDSA and Dilithium keypairs. | `certgen.c`, `keygen.c`, `asncodec/DilithiumKey.c/h`, `asncodec/DilithiumSignature.c/h` |
| **benchmark** | Standalone ECDSA vs Dilithium/Falcon/SPHINCS+ benchmark (2000 rounds each). | `benchmark.c` |
| **liboqs** | Open Quantum Safe library providing PQC algorithm implementations. Pre-built for Dilithium-2. | `build/lib/liboqs.a`, `build/include/oqs/` |
| **asn1c** | FilLabs fork of the ASN.1-to-C compiler. Generates codec for `DilithiumKey`, `DilithiumSignature`, and ITS message types. | Pre-built in-tree |
| **TS.ITS** | ETSI ITS test suite — provides certificate profiles (`.xer` templates). | `data/profiles/` |

### PQC Integration Points

- **ASN.1 types**: `DilithiumKey` (1312-byte OCTET STRING) and `DilithiumSignature` (2420-byte OCTET STRING) added as choices in `PublicVerificationKey` and `Signature`.
- **Certificate generation**: `fill_Dilithium_keyPair()` in `certgen.c` generates Dilithium-2 keypairs via `OQS_SIG_dilithium_2_keypair()`. The `Signature_oer_encoder()` signs with `OQS_SIG_dilithium_2_sign()`.
- **Message signing**: When `flag_PQC == 1`, `MsgGenApp_Send()` in `fsmsggen.c` bypasses standard ECDSA and manually constructs the security envelope with Dilithium signatures and certificates.
- **Chain verification**: `load_data.c` contains `Emulated_InstallCertificate()` which parses OER certificates and verifies the RCA→AA→AT chain using `OQS_SIG_dilithium_2_verify()`.
- **Benchmark**: `compare_sigTime()` in `extensions.c` and standalone `benchmark.c` compare ECDSA, Dilithium, Falcon, and SPHINCS+.

### Message Size Impact

| Mode | Message Type | Size |
|------|-------------|------|
| ECDSA | CAM with certificate | 373 bytes |
| ECDSA | CAM with digest | 192 bytes |
| Dilithium | CAM with certificate | **6,368 bytes** (17× larger) |
| Dilithium | CAM with digest | **2,547 bytes** (13× larger) |

This requires MTU extension: `sudo ip link set dev eth0 mtu 7000`.

### Certificate Pools

Three pools exist in `fsmsggen/`:

| Pool | Algorithm | Contents |
|------|-----------|----------|
| `POOL_CAM/` | ECDSA NIST P-256 | RCA + AA + AT certificates (`.oer`, `.vkey`, `.vkey_pub`, `.ekey`, `.ekey_pub`) |
| `POOL_CAM_PQC/` | Dilithium-2 | Same hierarchy with Dilithium verification keys and signatures |
| `POOL_CAM_PQC_new/` | Dilithium-2 | Regenerated version of `POOL_CAM_PQC` with fresh key material |

### Build System

Build order (from inside the container at `/root/Project_FsMsGen/`):

```
1. liboqs     — cd liboqs && mkdir -p build && cd build && cmake -GNinja -DOQS_USE_OPENSSL=ON .. && ninja
2. itscertgen — cd itscertgen/certgen && make
3. fsmsggen   — cd fsmsggen && make
4. benchmark  — cd benchmark && make
```

All three component Makefiles hardcode the path `/root/Project_FsMsGen/liboqs/build/` for includes and libraries. Both `fsmsggen` and `itscertgen` share the `cshared/common.mk` build framework.

---

## Results

Performance measured on **Intel Core i7-8565U**, 2000 rounds each:

### Signing Time

| Algorithm | Avg Time | vs ECDSA |
|-----------|----------|----------|
| ECDSA NIST P-256 | 0.110 ms | 1× |
| **Dilithium-2** | **0.302 ms** | **~2.7× slower** |
| Falcon-512 | 0.584 ms | ~5.3× slower |
| SPHINCS+ 128s | 599.0 ms | ~5446× slower |

### Verification Time

| Algorithm | Avg Time | vs ECDSA |
|-----------|----------|----------|
| ECDSA NIST P-256 | 0.281 ms | 1× |
| Falcon-512 | 0.089 ms | ~3.2× faster |
| **Dilithium-2** | **0.096 ms** | **~2.9× faster** |
| SPHINCS+ 128s | 0.716 ms | ~2.5× slower |

### Key Takeaways

- Dilithium signing is sub-millisecond (0.302 ms), well within the 100 ms CAM generation interval.
- Dilithium verification is **~3× faster** than ECDSA — critical since every received message must be verified.
- SPHINCS+ is completely impractical for V2X (599 ms signature time).
- The **main obstacle is message size**, not latency: 13-17× larger messages threaten channel congestion in dense V2X environments. This exceeds the default Ethernet MTU of 1500 bytes.
- Falcon-512 offers smaller signatures (666 bytes) than Dilithium (2420 bytes) with similar verification speed, but its floating-point dependency makes integration harder.
- The integration was functionally successful: Dilithium-signed CAMs were generated, injected over Ethernet, and captured with Wireshark.

---

## Usage

### Docker Setup

The project is designed to run inside the pre-built Docker image:

```bash
# Pull the image
docker pull matteopedone/cybersec_camgen:5.0

# Run with X11 forwarding (for Wireshark, etc.)
docker run -it \
  -e SDL_VIDEODRIVER=x11 \
  -e DISPLAY=$DISPLAY \
  --ipc host \
  --privileged \
  --network host \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  matteopedone/cybersec_camgen:5.0

# Working directory inside container: /root/Project_FsMsGen/
```

**Note**: If you need to mount the project from the host (instead of using the pre-built image):

```bash
docker run -it \
  -v ./Project_FsMsgGen:/root/Project_FsMsGen \
  --network host \
  --privileged \
  matteopedone/cybersec_camgen:5.0
```

### Build Order (inside container)

```bash
# 0. Install prerequisites (if needed)
apt update
apt install -y cmake ninja-build libssl-dev git wget

# 1. liboqs (pre-built in the image — skip unless rebuilding)
cd /root/Project_FsMsGen/liboqs
mkdir -p build && cd build
cmake -GNinja -DOQS_USE_OPENSSL=ON .. && ninja

# 2. Certificate generator
cd /root/Project_FsMsGen/itscertgen/certgen && make

# 3. Message generator
cd /root/Project_FsMsGen/fsmsggen && make

# 4. Benchmark (optional)
cd /root/Project_FsMsGen/benchmark && make
```

### Generate Certificates

```bash
cd /root/Project_FsMsGen/itscertgen/build/x86_64-linux-gnu-d
mkdir -p outputcertificates

# ECDSA certificates
./certgen -o outputcertificates CERT_IUT_A_RCA.xer

# Dilithium (PQC) certificates
./certgen -o outputcertificates CERT_IUT_A_RCA_Dilithium.xer
```

Generated files in `outputcertificates/`:
- `*.oer` — certificates in OER format
- `*.vkey` — private keys
- `*.vkey_pub` — public keys

### Run the Message Generator

```bash
# Network setup (run once)
sudo setcap cap_net_raw,cap_net_admin=eip /root/Project_FsMsGen/fsmsggen/build/x86_64-linux-gnu-d/fsmsggen
sudo ip link set dev eth0 mtu 7000   # required for large Dilithium messages

cd /root/Project_FsMsGen/fsmsggen/build/x86_64-linux-gnu-d

# ECDSA mode
./fsmsggen -i eth0 -1 ../../POOL_CAM/

# Dilithium + ECDSA dual mode
./fsmsggen -i eth0 -1 ../../POOL_CAM_PQC -1 ../../POOL_CAM/
```

**Network interface commands**:
```bash
ip a                          # list available interfaces
ping 192.168.1.12             # test connectivity to host
```

Monitor traffic with **Wireshark** on the host interface.

### Troubleshooting

| Problem | Solution |
|---------|----------|
| `cmake` or `ninja` not found | `apt install cmake ninja-build` |
| `certgen` not found | Build first: `cd itscertgen/certgen && make` |
| `certgen` fails reading `.xer` | Ensure `.xer` files are in the same directory as the `certgen` binary |
| Permission denied on liboqs headers | `chown -R ubuntu:ubuntu liboqs/` |
| `./fsmsggen` can't open raw socket | `sudo setcap cap_net_raw,cap_net_admin=eip ./fsmsggen` |
| Packet truncated / MTU errors | `sudo ip link set dev eth0 mtu 7000` |
| `asn1c` version incompatible | Build from source (see `asn1c/` directory); the apt version is too old |
| Build fails in `asncodec` | First build `asn1c`, then run `make` inside `itscertgen/certgen/asncodec/` |
| Windows `Zone.Identifier` files | `rm -f *Zone.Identifier` |

---

## Repository Structure

```
CamGen_PQC/
├── Project_FsMsgGen/              # Main project codebase
│   ├── fsmsggen/                  # V2X message generator
│   │   ├── POOL_CAM/              # ECDSA certificate pool
│   │   ├── POOL_CAM_PQC/          # Dilithium certificate pool
│   │   ├── POOL_CAM_PQC_new/      # Regenerated Dilithium pool
│   │   ├── extensions.c/h         # PQC support layer
│   │   ├── load_data.c            # Certificate loader (ECDSA + PQC)
│   │   └── build/x86_64-linux-gnu-d/  # Build output + fsmsggen binary
│   ├── itscertgen/                # Certificate generator
│   │   ├── certgen/               # Source + Makefile
│   │   │   └── asncodec/          # ASN.1 codec (DilithiumKey, etc.)
│   │   └── build/x86_64-linux-gnu-d/  # Build output + certgen binary
│   ├── benchmark/                 # PQC vs ECDSA benchmark
│   ├── liboqs/                    # Open Quantum Safe library (pre-built)
│   ├── asn1c/                     # ASN.1 compiler (pre-built)
│   ├── TS.ITS/                    # ETSI ITS test suite (profiles)
│   └── AGENTS.md                  # Project documentation for AI agents
├── thesis/                        # Master's thesis (PDF + summary)
├── docker_log/                    # Docker command history
├── commands_inside_docker/        # Shell history from inside container
```

---

## Limitations & Future Work

- Only a minimal PKI setup (1 RCA, 1 AA, 1 AT) was tested; not representative of real-world multi-CA deployments.
- Message **reception and verification** at the receiver side were not implemented — only generation/injection.
- Encryption keys remain **ECIES-based** (hybrid certificates — only verification keys use PQC).
- No real-world multi-vehicle V2X scenario simulation or network bandwidth/latency analysis.
- Message size explosion (13-17×) requires mitigation: fragmentation, caching, or compression.
- Only CAM messages were fully implemented; DENM and other message types are not covered.
- Falcon and SPHINCS+ were only benchmarked standalone, not integrated into the message pipeline.

---

## Author

**Matteo Pedone** — Master's Thesis in Computer Engineering, Politecnico di Torino  
Supervisors: Prof. Fulvio Valenza, Eng. Leonardo De Candia  
In collaboration with *Nardò Technical Center — Porsche Engineering*

---

*Generated by 4 AI agents analyzing the codebase, thesis, Docker history, and internal workflow.*
