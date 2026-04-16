# MQTT Deploy CODEX

This repository contains deployment materials for the cloud side of the project.

Current scope:

- `emqx/`: single-node EMQX deployment for Tencent Cloud CVM

## Quick Start

Clone the repo, then enter the EMQX directory:

```bash
git clone https://github.com/Qing-King/MQTT-Deploy-CODEX.git
cd MQTT-Deploy-CODEX/emqx
cp .env.example .env
```

Then follow:

- `emqx/README.md`

## Notes

- `.env` should stay local and must not be committed
- the first version keeps Dashboard and MQTT listeners private until access control is configured
