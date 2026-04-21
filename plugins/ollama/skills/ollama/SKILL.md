---
description: ollama skill for RobboHome automation.
---

# Skill: Ollama & Open WebUI Management

## Access
- Ollama API: http://svr002:11434 (localhost only, not exposed externally)
- Open WebUI: http://svr002:3001 / https://ai.robbohome.com (Google auth)
- Open WebUI credentials in ~/data/config/.secrets

## Installed models
| Model        | Size  | Capabilities              | TPS @ 150W |
|--------------|-------|---------------------------|------------|
| gemma3:4b    | 3.3GB | completion, vision        | ~111       |
| qwen3:8b     | 5.2GB | completion, tools, thinking| ~69       |
| gemma4:e4b   | 9.6GB | completion, vision, audio, tools, thinking | ~31 (RAM-bound) |

Default model: qwen3:8b

## Pull a new model
```bash
ssh robbohome-server 'docker exec ollama ollama pull MODEL_NAME'
```

## List models
```bash
ssh robbohome-server 'docker exec ollama ollama list'
```

## TPS benchmark
```bash
ssh robbohome-server 'python3 << EOF
import urllib.request, json
models = ["gemma3:4b", "qwen3:8b", "gemma4:e4b"]
prompt = "Explain what a large language model is in 3 sentences."
for model in models:
    payload = json.dumps({"model": model, "prompt": prompt, "stream": False}).encode()
    req = urllib.request.Request("http://localhost:11434/api/generate", data=payload, headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=120) as resp:
        r = json.loads(resp.read())
    tps = r["eval_count"] / (r["eval_duration"] / 1e9)
    print(f"{model:<14} | {tps:>6.1f} TPS")
EOF'
```

## Check model capabilities (vision, tools, thinking)
```bash
ssh robbohome-server 'curl -s http://localhost:11434/api/show \
  -d "{\"model\":\"MODEL\"}" | python3 -c "import sys,json; r=json.load(sys.stdin); print(r[\"capabilities\"])"'
```

## Docker compose location
~/data/ollama/docker-compose.yml on svr002
~/data/infrastructure/ollama/docker-compose.yml on Mac (source of truth)

## Config
- OLLAMA_MAX_LOADED_MODELS=1 (one model in VRAM at a time, 8GB VRAM)
- OLLAMA_NUM_PARALLEL=1
- Ollama bound to 127.0.0.1:11434 (internal only, accessed via open-webui container network)
