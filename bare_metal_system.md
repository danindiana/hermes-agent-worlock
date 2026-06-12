# Bare-Metal System — `worlock`

The physical host this hermes-agent deployment runs on. Hardware/OS facts captured 2026-06-12;
networking and services facts from the machine's maintained `CLAUDE.md` context file.

## Identity

| | |
|---|---|
| Hostname | `worlock` |
| User | `jeb` (has `NOPASSWD: ALL` sudo) |
| OS | Ubuntu 22.04.5 LTS |
| Kernel | Linux 6.8.12 |
| Shell | zsh (`~/.zshrc`, wrapper at `~/.bash_env_wrapper`) |
| Primary working dir | `~/Documents/claude_creations/` (hermes lives at `~/hermes-agent`) |

## CPU & memory

| | |
|---|---|
| CPU | AMD Ryzen 9 5950X — 16 cores / 32 threads, 1 socket |
| RAM | 125 GiB total (~110 GiB available at capture) |

## GPUs

Dual NVIDIA, driver `580.159.03`:

| GPU | VRAM |
|-----|------|
| NVIDIA GeForce RTX 5080 | ~16 GB (16303 MiB) |
| NVIDIA GeForce RTX 3080 | ~10 GB (10240 MiB) |

Combined ~26.5 GB VRAM. Tools: `nvidia-smi`, `nvtop`. Note from host history: the RTX 5080's PCIe
clearance displaced an X540-T2 NIC (removed 2026-05-01) and has caused repeated PCIe renumbering
(interface renames enp7s0→enp8s0→enp9s0).

## Networking

| Interface | Role | Address |
|-----------|------|---------|
| `enp9s0` | LAN (active Ethernet, DHCP) | 192.168.1.85/24 |
| `wlp7s0` | WiFi (SSID `SuperNus`) | 192.168.1.134/24 |
| `wgpia0` | PIA WireGuard VPN (`piavpn.service`, auto-start) | varies |
| `tun0` | Legacy PIA OpenVPN (not normally up) | — |

LAN IPs drift via DHCP; the Ethernet interface name has changed across GPU PCIe renumbering.

## Key services

| Service | Port | Notes |
|---------|------|-------|
| SSH | 22222 | pubkey-only; UFW LAN + VPN only |
| OpenWebUI | 3000 | Docker host-net |
| Ollama (native) | 11434 | systemd, localhost-only |
| Ollama (Docker GPU) | 11435 | not always running |
| Netdata | 19999 | UFW LAN-only |
| NTP | 123 | ntpsec |

## Local AI tooling alongside hermes

- **Ollama** (systemd) with both GPUs (`CUDA_VISIBLE_DEVICES=0,1`), used as a fast local
  reasoning/planning layer. See the Ollama delegation toolkit at
  `~/Documents/claude_creations/.../ollama-delegate` (repo: `github.com/danindiana/ollama-delegate`).
- **Docker** stacks: `open-webui` and `self-hosted-ai-starter-kit` (GPU profile).

For exact config values (paths, env vars, UFW rules, Python versions), see
[`bare_metal_system_config.md`](bare_metal_system_config.md).
