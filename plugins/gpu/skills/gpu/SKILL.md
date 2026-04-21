---
description: gpu skill for RobboHome automation.
---

# Skill: GPU Management (RTX 3060 Ti)

## Specs
- GPU: NVIDIA GeForce RTX 3060 Ti
- VRAM: 8GB
- Driver: 580.126.09
- Power range: 100W–220W, default 200W
- Current limit: 150W (persistent via systemd)

## Check GPU status
```bash
ssh robbohome-server 'nvidia-smi'
```

## Check power limit
```bash
ssh robbohome-server 'nvidia-smi --query-gpu=power.limit,power.draw --format=csv,noheader'
```

## Change power limit (temporary)
```bash
ssh robbohome-server 'sudo nvidia-smi -pl WATTS'
```

## Change power limit (persistent)
Edit /etc/systemd/system/nvidia-power-limit.service on svr002, then:
```bash
ssh robbohome-server 'sudo systemctl daemon-reload && sudo systemctl restart nvidia-power-limit.service'
```

## TPS vs power limit findings (qwen3:8b)
| Power | TPS  | vs default |
|-------|------|------------|
| 200W  | 72.8 | baseline   |
| 150W  | 69.1 | -5%        |
| 125W  | 61.7 | -15%       |
| 100W  | 37.2 | -49%       |

**Sweet spot: 150W** — 25% power saving, only 5% TPS drop.

## Verify GPU available in Docker
```bash
ssh robbohome-server 'docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu24.04 nvidia-smi'
```

## LACT (undervolting) — TODO
LACT is the recommended tool for headless Linux undervolting. Not yet installed.
