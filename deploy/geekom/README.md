# garmin_mcp on the Geekom (private OMP tooling)

This is a fork branch of `Taxuspt/garmin_mcp`, based on upstream commit `655efb8`, with one
functional change: **`garminconnect==0.3.19`** instead of `0.3.2`.

Version 0.3.2 can no longer authenticate. Garmin rejects its tokens at
`/userprofile-service/socialProfile` with "Failed to retrieve social profile"; see
python-garminconnect #369, garmin_mcp #366 and #345. The newer versions also discard rejected
cached tokens and persist refreshed ones.

## What changed on this branch

| File | Change | Why |
|---|---|---|
| `pyproject.toml` | `garminconnect==0.3.2` → `==0.3.19` | The fix. |
| `pyproject.toml` | `requires-python >=3.10` → `>=3.12` | 0.3.19 only supports Python 3.12 or newer, and `uv lock` refuses the wider range. |
| `uv.lock` | `uv lock --upgrade-package garminconnect` | The only version change is garminconnect. Python <3.12 backports were dropped, and `tzdata` was added only for `sys_platform == 'win32'`. |
| `deploy/geekom/Dockerfile` | The upstream Dockerfile with the base image (`python:3.12.14-slim`), uv (`0.12.16`) and every runtime dependency pinned | Reproducible rebuild. |
| `deploy/geekom/constraints.txt` | Exact runtime dependency set | Taken with `pip freeze` from the production image. |

The upstream `Dockerfile` and source code are unchanged.

## Production identity (deployed 2026-10-22)

- **Image:** `garmin-mcp:655efb8f5639-gc0.3.19`, image ID
  `sha256:e22bba01ee4a748e3203af2cbc4e82db237bb13fe4a12110f716caa91407c683`.
  - This is the exact image that passed the isolated authentication test. It is also tagged
    `garmin-mcp-test:gc0.3.19`.
  - It was built as one extra layer on the previous production image `garmin-mcp:655efb8f5639`
    (`sha256:97bc6925381c…`), using
    `uv pip install --system garminconnect==0.3.19 pytest==9.0.3 pytest-asyncio==1.3.0`.
  - The pytest packages are test-only and unused at runtime.
- **Rebuild check:** a build from `deploy/geekom/Dockerfile` produces a byte-identical runtime
  dependency set (`pip freeze`, ignoring the pytest packages), with Python 3.12.14 and uv 0.12.16.
  Its image ID differs.
- **Compose file:** `/DATA/AppData/garmin-mcp/docker-compose.yml` on the host. It is not version
  controlled; a backup sits next to it in `backups/`.
  - Service `garmin-mcp`, container name `garmin-mcp`.
  - Network: the external `omp-tools` only. There are no published ports; OMP reaches
    `http://garmin-mcp:8000/mcp`.
  - Volume: the external `garmin-tokens`, mounted at `/root/.garminconnect`
    (`GARMINTOKENS=/root/.garminconnect`).
  - `restart: unless-stopped`, plus a 25-tool read-only allowlist in `GARMIN_ENABLED_TOOLS`.

## Build and deploy

The host has no compose plugin on its default `DOCKER_CONFIG`; use OMP's.

```sh
git clone -b geekom/655efb8-gc0.3.19 https://github.com/al1sse/garmin_mcp.git && cd garmin_mcp
docker build -f deploy/geekom/Dockerfile -t garmin-mcp:655efb8f5639-gc0.3.19-r .
# Set image: in /DATA/AppData/garmin-mcp/docker-compose.yml, back up the file first, then:
cd /DATA/AppData/garmin-mcp
DOCKER_CONFIG=/DATA/dev/omp/config/docker docker compose -p garmin-mcp -f docker-compose.yml config --quiet
DOCKER_CONFIG=/DATA/dev/omp/config/docker docker compose -p garmin-mcp -f docker-compose.yml up -d garmin-mcp
docker logs garmin-mcp 2>&1 | grep 'client initialized successfully'
```

Verify with real data, not just HTTP 200. Through the MCP endpoint, `get_devices` must return a
non-empty JSON list and `get_user_summary` a JSON object, with no "Garmin login failed" text.

## Token persistence

- Production tokens: the single file `garmin_tokens.json` (`root:root`, `0600`) in the volume
  `garmin-tokens` (directory `0700`). It holds `di_token`, `di_refresh_token` and `di_client_id`.
- 0.3.19 rewrites this file atomically when it refreshes tokens, so the volume must stay writable.
- Tokens are never in the image or in Git. The Dockerfile only creates the empty mount point.
- `garmin-mcp-auth` and the server use the same path (`$GARMINTOKENS`).

## Authentication troubleshooting

1. Check `docker logs garmin-mcp`. "Garmin Connect client initialized successfully" means token
   login worked.
2. If tokens are rejected, re-authenticate once, interactively, in your own terminal:
   `ssh -t geekom 'docker exec -it garmin-mcp garmin-mcp-auth --force-reauth'`, then run
   `docker restart garmin-mcp`.
3. On 429 or an account challenge, stop and wait at least 1 hour. Repeated attempts extend
   Garmin's block.
4. The "Failed to retrieve social profile" message from plain `garmin-mcp-auth` (without
   `--force-reauth`) is only the existing-token check. The credential prompt follows it.

## Rollback

```sh
# 1. Image: in docker-compose.yml set image: garmin-mcp:655efb8f5639 (kept locally), then
#    `compose up -d garmin-mcp`.
#    Note: 0.3.2 cannot authenticate against Garmin today, so this restores the old, non-working state.
# 2. Tokens: restore with the container stopped:
#    /DATA/AppData/garmin-mcp/backups/garmin-tokens-pre-gc0319-<UTC>.tar is a tar of the volume
#    root (root-only, 0600).
docker run --rm --network none -v garmin-tokens:/dst \
  -v /DATA/AppData/garmin-mcp/backups:/bk:ro alpine \
  sh -c 'tar -xpf /bk/garmin-tokens-pre-gc0319-<UTC>.tar -C /dst'
```

A recovery copy of the working tokens from 2026-10-22 is kept in the volume
`garmin-tokens-test-0319`. Once production refreshes and rotates its refresh token, that copy may
stop working.
