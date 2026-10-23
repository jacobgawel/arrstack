# ArrStack — Media Automation Stack

Docker Compose stack for automated movies and TV: request media via Seerr, manage with Sonarr/Radarr, download from Usenet via NZBGet, and serve through an external Plex install.

## Services

| Service | Port | Purpose | Image |
|---|---|---|---|
| Sonarr | http://localhost:8989 | TV series management | `lscr.io/linuxserver/sonarr:latest` |
| Radarr | http://localhost:7878 | Movie management | `lscr.io/linuxserver/radarr:latest` |
| NZBGet | http://localhost:6789 | Usenet download client | `nzbgetcom/nzbget:latest` |
| Seerr | http://localhost:5055 | Media requests & discovery | `ghcr.io/seerr-team/seerr:latest` |
| Recyclarr | — | Syncs TRaSH Guide profiles/custom formats to Sonarr/Radarr | `ghcr.io/recyclarr/recyclarr:8` |

All services use `TZ=Europe/London` and restart automatically (`unless-stopped`).

## Volume Layout

### Host folders
- `F:\Media Server\Movies` — final movie library (scanned by Plex)
- `F:\Media Server\TV Shows` — final TV library (scanned by Plex)
- `F:\Media Server\Downloads` — NZBGet download area (shared with Sonarr/Radarr)
- `F:\ArrStack\config\{sonarr,radarr,nzbget}` — app config/data
- `seerr-data` (Docker named volume) — Seerr config and database

### Container mount points
| Host path | Container path | Used by |
|---|---|---|
| `F:/Media Server/TV Shows` | `/tv` | Sonarr |
| `F:/Media Server/Movies` | `/movies` | Radarr |
| `F:/Media Server/Downloads` | `/downloads` | Sonarr, Radarr, NZBGet |
| `./config/<app>` | `/config` or `/app/config` | Sonarr, Radarr, NZBGet, Seerr (named volume) |

Because Sonarr, Radarr, and NZBGet all mount `F:\Media Server\Downloads` at the **same** container path (`/downloads`), no remote path mappings are needed anywhere.

> **Note:** Seerr's config lives in a Docker named volume (`seerr-data`) rather than a host folder. On Windows, bind-mounting Seerr's SQLite database over SMB can corrupt it. Deleting the volume (`docker compose down -v` or `docker volume rm seerr-data`) loses Seerr's settings, users, and request history. Backup location: `\\wsl$\docker-desktop-data\version-pack-data\community\docker\volumes\seerr-data\_data`.

## How the Flow Works

1. A user requests a movie/series in **Seerr** (http://localhost:5055)
2. Seerr forwards it to **Radarr** (movies) or **Sonarr** (TV)
3. Radarr/Sonarr search configured **indexers** and send the NZB to **NZBGet**
4. NZBGet downloads to `F:\Media Server\Downloads\completed\<category>`
5. Radarr/Sonarr rename and import the file into `/movies` or `/tv`
6. Seerr marks the request as available once **Plex** has scanned the new media into its library (Plex → Scan Library Files, then Seerr → Settings → Plex → Scan Plex Libraries)

## Configuration Summary

### NZBGet
- Provider: Eweka — host `news.eweka.nl`, port `563`, encryption on, connections `50` (plan max)
- `Level`/`Group` left at defaults (single provider)
- `ArticleCache` 600, `WriteBuffer` 2048 (KiB), `ArticleReadChunkSize` 64 (KiB) — tuned for ~900 Mbps line / NVMe disk
- Paths: `MainDir=/downloads`, `DestDir=${MainDir}/completed`, `AppendCategoryDir=no`
- Categories: `Movies` → `${MainDir}/completed/movies`, `Series` → `${MainDir}/completed/tv`
- Web UI login: `nzbget` / `tegbzn6789` (change under Settings → Security)

### Indexers
- NZBGeek (https://nzbgeek.info/) — usenet indexer, added to both Radarr and Sonarr (Settings → Indexers) with its API key
- Retention: set to ~3500 days (Eweka has ~5000; leave headroom)

### Radarr
- Root folder: `/movies`
- Download client: NZBGet, host `nzbget`, port `6789`, category `Movies`

### Sonarr
- Root folder: `/tv`
- Download client: NZBGet, host `nzbget`, port `6789`, category `tv` (matches NZBGet's `Series` category)
- Rename episodes enabled (Settings → Media Management)

### Seerr
- Media server: Plex (external install — detected via setup wizard)
- Radarr/Sonarr connections: add via API keys from Radarr/Sonarr Settings → General → API Key
- Root folder dropdowns are populated from Radarr/Sonarr's root folders

### Recyclarr
- Config: `config/recyclarr/configs/recyclarr.yml` (gitignored — contains API keys)
- Syncs the TRaSH Guides **Remux + WEB 1080p** quality profile to both Radarr and Sonarr
- **Special Edition** custom format boosted to score 200 in Radarr, so searches prefer extended/director's cut/unrated/special editions of movies
- Runs a scheduled sync automatically (daily); manual sync: `docker compose run --rm recyclarr sync`
- After first sync, bulk-switch existing movies/series to the "Remux + WEB 1080p" profile (Movies/Series → table view → select all → Edit → Quality Profile); for new requests, set the profile default in Seerr's Radarr/Sonarr settings

### Container networking
All services share the Compose default network and can reach each other by container name (`nzbget`, `radarr`, `sonarr`). Do not use `localhost` when configuring inter-container connections.

## Common Commands

```powershell
# Start the stack
docker compose up -d

# Apply changes and remove old containers (e.g. after service swaps)
docker compose up -d --remove-orphans

# Update all images and recreate
docker compose pull
docker compose up -d

# View logs
docker logs -f sonarr
docker logs -f nzbget

# Stop / stop and wipe Seerr data
docker compose down
docker compose down -v   # WARNING: deletes seerr-data volume
```

## Troubleshooting

- **"Directory does not appear to exist inside the container"** — a category DestDir points outside the shared `/downloads` mount, or the folder hasn't been created yet. Keep category paths inside `${MainDir}/completed/...` and make sure names match exactly between Radarr/Sonarr and NZBGet.
- **Downloads stuck in intermediate** — check NZBGet → Settings → Categories DestDir values; a category DestDir equal to MainDir can create nested folder trees inside the downloads folder.
- **Seerr can't reach Radarr/Sonarr** — use container names (`radarr`, `sonarr`) as host, not `localhost`.
- **Slow NZBGet downloads** — verify Eweka connections is 20, ArticleCache/WriteBuffer enabled, and ArticleReadChunkSize is 64.
