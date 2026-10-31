# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Docker Compose media-automation stack (Sonarr, Radarr, NZBGet, Seerr, Recyclarr, cloudflared) running on Docker Desktop on Windows. Plex runs natively on the Windows host, outside Compose. There is no application code, build, or test suite. The tracked files are `docker-compose.yml`, `.env.example`, `config-templates/`, and `README.md`. `README.md` is the full operator guide and holds the intended wiring between the apps.

## Commands

Run from `F:\ArrStack`:

```powershell
docker compose config            # validate compose syntax/interpolation after editing (prints .env values; don't paste output)
docker compose up -d             # apply compose changes (recreates only changed services)
docker compose ps                # health status (all services except recyclarr have healthchecks)
docker compose logs --tail 100 <service>
docker compose run --rm recyclarr sync   # apply Recyclarr profile/custom-format changes now
docker compose up -d --force-recreate cloudflared   # required after changing .env; a restart won't reload env
```

Never run `docker compose down -v`. It deletes the `seerr-data` named volume, which holds Seerr's users and request history.

## Architecture points that span files

- **Two copies of the Recyclarr config.** The live config is `config/recyclarr/configs/recyclarr.yml`. It is gitignored because it contains API keys. `config-templates/recyclarr.yml` is the sanitized copy that git tracks, with `RADARR_API_KEY`/`SONARR_API_KEY` placeholders. A profile or custom-format change must go into both files. Commit only the template. Recent commits (language CF penalty, `delete_old_custom_formats`) follow this pattern.
- **README mirrors config values.** The README hardcodes the image tags, NZBGet settings, Recyclarr scores and profile names, port bindings, and healthcheck behavior. When you change any of these in compose or the templates, update the matching README section in the same commit.
- **One shared `/data` mount.** NZBGet, Sonarr, and Radarr all mount `F:/Media Server` as `/data`, so downloads (`/data/Downloads`) and libraries (`/data/Movies`, `/data/TV Shows`) are on one mount point and imports are true renames. Do not reintroduce separate per-folder mounts: Docker Desktop makes each bind mount its own mount point, so a cross-mount "move" silently becomes a full copy (measured 10 s vs 4 ms per GB). When changing root folders, Radarr's Collections carry their own root-folder path that the Movie Editor does not update (fix via Collections → Edit Collections). NZBGet's category names (`Movies`, `Series`) differ from their output folders (`completed/movies`, `completed/tv`). The *arr download-client settings must use the category names.
- **Networking:** apps reach each other over the Compose default network by service name (`http://radarr:7878`, `nzbget:6789`, `http://seerr:5055` for the tunnel). Published ports are bound to `127.0.0.1` on purpose. Do not widen them without securing the app first.
- **Image tags are pinned.** `docker compose pull` does nothing until a tag in `docker-compose.yml` is edited. Shared env and logging come from the `x-lsio-env` / `x-logging` YAML anchors.
- `config/` (all app state, SQLite DBs, API keys) and `.env` (the `CLOUDFLARE_TUNNEL_TOKEN`) are gitignored. Never commit them, and don't copy their contents into tracked files.
- `config-templates/nzbget-restore-settings.txt` is a reference you apply through the NZBGet UI. It is not a `nzbget.conf` and must never overwrite the generated config.

## Environment quirks

- No Python on the host (`python` opens the Microsoft Store). Use the shell or the Edit tool.
- Bash commands that grep passwords or API keys out of `config/*` (e.g. `config.xml`, `nzbget.conf`) get blocked. Verify app state with HTTP checks instead, such as `curl -fsS http://127.0.0.1:8989/ping` or `docker compose ps`.
- In Git Bash, set `MSYS_NO_PATHCONV=1` for `docker exec` commands that pass container paths like `/app/...`.
