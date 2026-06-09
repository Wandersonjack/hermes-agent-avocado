# Deploy this fork as the Avocado multi-tenant Hermes (handoff)

This fork powers Avocado's per-customer agents: **one deployment, one Hermes profile per customer** (each = its own Telegram bot, pointed at that customer's Avocado account via the Avocado MCP). No source changes to Hermes are required — only sync, deploy, and a provisioning script.

## ✅ STEP 0 — Sync to upstream (DONE, 2026-06-09)
This fork was v0.14.0, which predated the per-profile gateway supervision we depend on. It has been **fast-forwarded to v0.16.0** (the fork had zero divergent commits — a clean fast-forward, no conflicts). The s6 supervision the multi-profile model needs is now present and verified:

- `docker/s6-rc.d/` ✓  (Dockerfile entrypoint is now s6 `/init`, not tini)
- `docker/cont-init.d/02-reconcile-profiles` ✓  (auto-restarts profiles whose last state was `running`)
- `hermes_cli/service_manager.py` ✓  (`gateway start` registers an s6-supervised slot when `detect_service_manager() == "s6"`)

How it was done (reproduce only if re-syncing later):
```sh
git remote add upstream https://github.com/NousResearch/hermes-agent.git
git fetch upstream
git merge --ff-only upstream/main      # clean fast-forward; no conflicts
ls docker/s6-rc.d/ && ls docker/cont-init.d/ | grep profile && ls hermes_cli/service_manager.py
```
Rollback if ever needed: `git reset --hard cae753735`. **Not yet pushed to `origin`** — push when ready to deploy.

## STEP 1 — Build & smoke-test the image locally (optional but recommended)
```sh
docker build -t hermes-avocado .
# boot it with a throwaway volume; confirm s6 /init comes up and `gateway run` starts
docker run --rm -e HERMES_HOME=/opt/data -v hermes_test:/opt/data hermes-avocado gateway run
```
Watch for s6 init + gateway start, no crash. Ctrl-C, `docker volume rm hermes_test`.

## STEP 2 — Deploy (pick one)

### Railway (easiest; Wanderson has tested Railway before)
- New Project → Deploy from this GitHub repo. `railway.json` sets the Dockerfile build + `gateway run` start command.
- **Attach a Volume at `/opt/data`** (critical — persists all customer profiles/state across redeploys).
- Variables: `HERMES_HOME=/opt/data`, `AUTO_UPDATE=false`.
- Polling mode = outbound only, so **no public domain/port needed**.

### Hetzner (cheapest; ~€5–7/mo CAX ARM)
- Provision a CAX/CX box, install Docker.
- `docker compose up -d` works out of the box (`docker-compose.yml` is present). Note: the compose binds `~/.hermes:/opt/data` and uses host networking — fine on a dedicated VPS. Set `AUTO_UPDATE=false`.
- The `dashboard` service is localhost-only; reach it via SSH tunnel if needed.

## STEP 3 — Provision customer #1 (concierge)
Inside the running container's shell (Railway Shell, or `docker exec -it hermes sh`):
```sh
SLUG=pilot-1 \
TELEGRAM_BOT_TOKEN=<botfather token> \
TELEGRAM_USER_ID=<their numeric telegram id> \
AVOCADO_MCP_KEY=<their avocado mcp key> \
OPENROUTER_API_KEY=<key with $10 hard cap set in OpenRouter> \
sh provision-in-container.sh
```
`provision-in-container.sh` creates the profile, writes its config (Avocado MCP scoped to their key, **safe toolset — no shell/code tools**, manual approvals), and starts the bot. It **refuses to run unless `$RAILWAY_ENVIRONMENT` is set** (or run it via `docker exec` on Hetzner — adjust the guard there if needed).

## STEP 4 — Validate
1. Bot replies on Telegram. 2. Image generates **to the customer's** Avocado account (their credits drop). 3. A different Telegram account is **ignored** (allowlist). 4. Survives a service restart (volume + s6 reconcile). 5. Exhaust a test-capped OpenRouter key → agent hard-stops.

## Notes
- The full pilot toolkit + rationale lives in the Avocado app repo at `ops/hermes-pilot/` (README, DEPLOY-RAILWAY.md, two-account isolation test, the HTTP sidecar for later self-serve).
- Keep `AUTO_UPDATE=false` so a paying customer's agent never breaks from an unattended upstream pull. Re-sync deliberately, test, then redeploy.
- Don't expose the dashboard/API server publicly per customer — customers use Telegram only.
