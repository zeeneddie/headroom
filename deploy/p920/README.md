# headroom op P920 — AOS-M.1 deploy (from-source + offline Kompress)

Test/acceptatie-deploy van de `headroom`-context-compressie-proxy op P920 (`marqed003`,
Tailscale `100.84.112.24`), **from-source** (P22) en **egress-invariant** (offline Kompress-model,
`HF_HUB_OFFLINE=1` op runtime; de enige fetch is de eenmalige `model-seed`-init).

Hoort bij werkpakket **AOS-M.1** ([PLAN](https://github.com/marqed-ai/mq-platform/blob/main/docs/streams/stream-N-harness-engineering/projects/AOS-headroom-context-compression/PLAN.md)).

## Wat het is
- **headroom-proxy** — OpenAI/Anthropic-compatible drop-in proxy die context comprimeert vóór egress
  en doorstuurt naar de **MarQed LiteLLM-gateway** (`OPENAI_TARGET_API_URL`). Geen `/compress`-endpoint:
  je routeert de LLM-call *door* de proxy en leest de ratio uit `/stats`.
- **model-seed** — eenmalige init die `chopratejas/kompress-v2-base` (de Kompress-ONNX, ~262MB) in een
  Docker-volume zet, zodat de proxy daarna **zonder netwerk** comprimeert (egress-invariant).
- **qdrant + neo4j** — optioneel (`--profile semantic`), alléén voor de semantische/multi-hop-memory-laag
  bóvenop token-compressie. **Niet nodig** voor de core (Kompress draait standalone).

## Eenmalige pre-condities (Eddie)
1. Self-hosted GitHub Actions runner op P920 met label **`p920`** voor `zeeneddie/headroom`.
2. Repo-secret **`HEADROOM_UPSTREAM_URL`** = LiteLLM-gateway-URL (OpenAI-compatible base).
3. (optioneel) repo-vars `HEADROOM_BIND_IP` / `HEADROOM_PORT` (default `100.84.112.24` / `8790`).

## Deploy
Automatisch via **`.github/workflows/deploy-p920.yml`** bij merge naar `main` (paths `deploy/p920/**`,
`Dockerfile`, `headroom/**`) of handmatig (`workflow_dispatch`). De workflow: build from-source →
seed-model → `up -d --force-recreate` (Tailscale boot-race-safe) → health-smoke.

### Handmatig op P920 (alleen voor debug; normaal via CI/CD — P8)
```bash
cd deploy/p920
echo "HEADROOM_UPSTREAM_URL=<litellm-gateway>" > .env
docker compose --env-file .env build                 # from-source (P22)
docker compose --env-file .env run --rm model-seed   # eenmalig: offline-model in volume
docker compose --env-file .env up -d --force-recreate headroom-proxy
# semantische laag erbij (optioneel): docker compose --env-file .env --profile semantic up -d
```

## Verifiëren (compressie werkt offline)
```bash
# bind-IP/poort:
curl -s http://100.84.112.24:8790/health        # healthy
# drive 1 request door de proxy (OpenAI-compatible) en lees de compressie:
curl -s http://100.84.112.24:8790/stats | python3 -c \
  "import json,sys;c=json.load(sys.stdin)['summary']['compression'];print(c['avg_compression_pct'],'% ·',c['total_tokens_removed'],'tokens removed')"
```
Lokaal bewezen 2026-06-13: **2.216 → 1.444 tokens (−34,8%)** met `HF_HUB_OFFLINE=1` (geen download).

## Boot-race (Tailscale)
Na een P920-reboot draait de container "healthy" maar kan de poort dood zijn (docker start vóór
`tailscale0`-IP er is). **Fix: `docker compose up -d --force-recreate`** (niet `docker restart`).

## Config-knoppen (compose `environment`)
| Env | Default | Wat |
|---|---|---|
| `HEADROOM_COMPRESS_USER_MESSAGES` | `1` | comprimeer user-messages (default-uit in upstream) |
| `HEADROOM_FORCE_KOMPRESS` | `1` | forceer de Kompress-ONNX-compressor |
| `HEADROOM_KOMPRESS_BACKEND` | `onnx` | ONNX-runtime (geen torch nodig) |
| `HEADROOM_MIN_TOKENS` | `200` | minimale prompt-grootte vóór compressie |
| `HEADROOM_SAVINGS_TARGET` | `0.50` | doel-compressieratio |
| `HF_HUB_OFFLINE` | `1` | **egress-invariant**: nooit HF-fetch op runtime |
