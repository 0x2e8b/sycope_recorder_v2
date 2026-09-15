# Sycope Traffic Recorder v2

Alert-triggered PCAP extraction service. Sycope calls `POST /extract`; the
service builds a BPF filter from the alert, runs `npcapextract` against the
n2disk rolling capture, and returns an HTTPS download URL. Files are served by
Caddy; the API runs FastAPI under gunicorn.

See `docs/superpowers/specs/2026-07-01-sycope-recorder-v2-design.md` for the
design and `SPEC.md` for the preserved legacy behavioral contract.

## Quickstart (Docker)

```bash
# 1. Generate a bcrypt hash for the shared Basic-auth credential:
docker run --rm caddy:2 caddy hash-password --plaintext 'yourpassword'

# 2. Set required environment and launch:
export SR_PUBLIC_HOST=recorder.example.com
export BASIC_AUTH_USER=sycope
export BASIC_AUTH_HASH='<paste the hash>'
# optional: restrict downloads (and optionally /extract) by source IP
export ALLOWED_IPS="10.0.0.0/8 192.168.0.0/16"
docker compose up -d --build
```

The API listens only on a unix socket; Caddy publishes 443 (and 80→redirect)
with a self-signed cert from its internal CA. Point Sycope's webhook at
`https://$SR_PUBLIC_HOST/extract` with the Basic-auth credential; clients must
trust Caddy's internal CA or skip verification.

## Configuration

All app settings use the `SR_` env prefix — see `src/sycope_recorder/config.py`
for the full list and defaults (host, dirs, filter/before/after defaults,
`max_concurrent_extractions`, `extract_timeout_seconds`, retention knobs,
logging). Auth and the download IP allowlist are configured in Caddy, not the
app.

The `/extract` request body is capped at 2 MiB. This is enforced
authoritatively at the edge by Caddy (`request_body { max_size 2MiB }` in
`caddy/Caddyfile`), which rejects oversized bodies regardless of the
`Content-Length` header; the app also checks `Content-Length` as a cheap
early-out before reading the body.

## The recorder (n2disk)

n2disk is not yet containerized. Templates for its config and systemd unit are
in `docs/reference/`. When integrating it as a compose service, attach it to the
`rolling` volume with capture privileges (see the commented `recorder` block in
`compose.yaml`), and decide where the `npcapextract` binary lives (bundled in
the API image or invoked from the recorder container).

## Development

```bash
uv sync
uv run pytest -q     # unit + integration; uses a stub npcapextract, no n2disk
uv run ruff check .
```
