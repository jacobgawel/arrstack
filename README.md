# ArrStack — Media Automation Stack

Docker Compose stack for media requests via Seerr, library automation via Sonarr/Radarr, and Usenet downloads via NZBGet. Plex runs separately on the Windows host.

## Configuration and verification

Reviewed on 29 October 2026 against `docker-compose.yml`, the saved Sonarr/Radarr XML settings, NZBGet configuration, and Recyclarr configuration. On 30 October 2026 the Compose file was hardened: image tags pinned, web ports bound to localhost, healthchecks added for every service except Recyclarr, container log rotation enabled, and Recyclarr made to wait for healthy Sonarr/Radarr. On 31 October 2026 NZBGet, Sonarr, and Radarr were given a single shared `/data` mount so imports rename instead of copy (see "Storage and networking"). This is a snapshot, not a continuous health check. An end-to-end request, download, import, and Plex playback was not tested.

Sonarr/Radarr download-client categories, indexers, quality-profile assignments, and Seerr connections must be checked in their UIs; they were not independently verified during this review. The setup instructions below describe the intended wiring.

## Services

| Service | Local URL | Purpose | Image |
|---|---|---|---|
| Sonarr | http://localhost:8989 | TV library automation | `lscr.io/linuxserver/sonarr:4.0.20.3014-ls1` |
| Radarr | http://localhost:7878 | Movie library automation | `lscr.io/linuxserver/radarr:6.4.4.10685-ls323` |
| NZBGet | http://localhost:6789 | Usenet downloads | `nzbgetcom/nzbget:v26.3` |
| Seerr | http://localhost:5055 | Requests and discovery | `ghcr.io/seerr-team/seerr:v3.5.1` |
| cloudflared | No published port | Cloudflare Tunnel connector for Seerr | `cloudflare/cloudflared:2026.10.4` |
| Recyclarr | No published port | Scheduled TRaSH quality-profile/custom-format sync | `ghcr.io/recyclarr/recyclarr:8.8.0` |
All services use `restart: unless-stopped` and set `TZ=Europe/London`. Every service except Recyclarr has a Compose healthcheck (`docker compose ps` shows `healthy`/`unhealthy`): Sonarr and Radarr poll `/ping`, NZBGet polls its web root from inside the container (localhost is exempt from its auth), cloudflared runs `cloudflared tunnel ready` against its metrics endpoint, and Seerr polls its public settings API. A healthy status proves the process answers HTTP, not that its integrations work. Restart policies do not automatically restart an unhealthy container or start Docker Desktop after a Windows reboot. Recyclarr waits for Sonarr and Radarr to be healthy before starting.

The four published web ports bind to `127.0.0.1` only, so the apps are reachable from this PC but not from other devices on the LAN. If LAN access to an app is needed, change its port mapping back to the `8989:8989` form deliberately, and secure that app first. Container logs rotate at 10 MB x 3 files per service.

## Storage and networking

| Host storage | Container path | Used by |
|---|---|---|
| `F:/Media Server` | `/data` | NZBGet, Sonarr, Radarr |
| `./config/sonarr` | `/config` | Sonarr |
| `./config/radarr` | `/config` | Radarr |
| `./config/nzbget` | `/config` | NZBGet |
| `./config/recyclarr` | `/config` | Recyclarr |
| Compose named volume `seerr-data` | `/app/config` | Seerr |

Relative paths resolve from this Compose project. Plex reads the Windows library folders directly. All containers share the Compose default network: use service names such as `radarr`, `sonarr`, and `nzbget` for connections between containers, not `localhost`. Host Plex connections require an address reachable from the container.

NZBGet, Sonarr, and Radarr all mount the single parent folder `F:/Media Server` as `/data`, so downloads (`/data/Downloads`) and the libraries (`/data/Movies`, `/data/TV Shows`) are on one mount and no remote path mappings are needed. This matters because Docker Desktop presents each bind mount as a separate mount point: a rename between two mounts fails and falls back to copy-then-delete, even when both host folders are on F:. Measured on 31 October 2026 inside the Sonarr container, moving a 1 GB file from `/downloads` to `/tv` took 10.5 s and produced a new inode (a copy), while the same move from `/data/Downloads` to `/data/TV Shows` took 4 ms and kept the inode (a true rename). With `/data`, imports are instant and do not need the file's size in free space twice.

The previous per-folder mounts (`/tv`, `/movies`, `/downloads`) were removed on 31 October 2026 after all series and movie paths had been repointed to `/data/...` with the Mass/Movie Editor ("don't move files"). Do not reintroduce them.

Seerr's database is in Docker-managed storage, separate from `./config`. The actual volume name is normally project-prefixed; discover it with `docker volume ls`. Do not rely on a hard-coded internal WSL filesystem path to back it up.

## Request flow

1. A user requests media in Seerr, locally or through the configured Cloudflare hostname.
2. Seerr submits an approved request to Radarr or Sonarr.
3. Radarr/Sonarr search their configured indexers and send an NZB to NZBGet.
4. NZBGet downloads and unpacks into `/data/Downloads/completed/movies` or `/data/Downloads/completed/tv`, according to category.
5. Radarr/Sonarr import and rename media into `/data/Movies` or `/data/TV Shows`.
6. Plex scans the Windows library folders. Seerr's subsequent Plex library sync updates availability.

If availability is stale, scan the library in Plex, then trigger Seerr's Plex library sync. Library scans and request approval depend on the relevant app settings.

## NZBGet

Saved configuration at review:

| Setting | Value |
|---|---|
| Provider | Eweka, `news.eweka.nl`, port `563`, encryption enabled |
| Connections | `40` |
| MainDir | `/data/Downloads` |
| DestDir | `${MainDir}/completed` |
| InterDir | `${MainDir}/intermediate` |
| AppendCategoryDir | `no` |
| Category `Movies` | `${MainDir}/completed/movies` |
| Category `Series` | `${MainDir}/completed/tv` |
| ArticleCache | `600` MiB |
| WriteBuffer | `2048` KiB |
| ArticleReadChunkSize | `64` KiB |
| UnrarCmd | `unrar` |
| DirectUnpack | `no` |
| ParRepair | `yes` |

These are the saved values, not a guarantee of optimal performance. The previous setup notes report connection-limit errors at 50 connections; keep 40 as the restore baseline.

NZBGet still had its default control password at review. Change it in Settings → Security and update the credentials in Sonarr/Radarr. Its web control interface has TLS disabled; keep access within trusted networks or use a protected proxy. Provider encryption is a separate setting and is enabled.

## Sonarr, Radarr, and indexers

Both apps have forms authentication enabled and authentication required in their saved XML configuration.

Intended connections:

| App | Root folder | Download client | Category to configure |
|---|---|---|---|
| Radarr | `/data/Movies` | NZBGet at `nzbget:6789` | `Movies` |
| Sonarr | `/data/TV Shows` | NZBGet at `nzbget:6789` | `Series` |

`Series` is the saved NZBGet category name; `tv` is the output folder name. Do not confuse them when configuring Sonarr. Check the existing client setting before changing anything.

Earlier setup notes list NZBGeek and AltHUB as indexers in both apps and a retention limit around 3500 days. Verify these in Settings → Indexers; provider retention and subscription limits should be checked against the current account. Configure episode/movie renaming and test download-client connections through the UIs.

## Seerr and Cloudflare Tunnel

Connect Seerr to the external Plex installation and to Radarr/Sonarr using their API keys. Choose the correct root folders and quality profiles for new requests; existing library profile assignments are managed separately in Radarr/Sonarr.

Store `CLOUDFLARE_TUNNEL_TOKEN` in the gitignored `.env` (see `.env.example`). It is the connector token from the Cloudflare tunnel setup, not the tunnel ID. Configure the public hostname in Cloudflare to route to `http://seerr:5055`. The live dashboard routing and access policies are not defined in this repository.

After changing the token, recreate the connector:

```powershell
docker compose up -d --force-recreate cloudflared
```

A plain restart does not load changed environment variables. Verify Seerr sign-in and permissions before sharing its public hostname. The intended public route is Seerr; there is no documented public route for the administration apps.

## Recyclarr

Live configuration: `config/recyclarr/configs/recyclarr.yml` (gitignored because it contains API keys). Sanitized restore template: `config-templates/recyclarr.yml`.

Both configs select **Remux + WEB 1080p** for Radarr and Sonarr. Radarr gives Special Edition a score of `200`; both apps give Language: Not English a score of `-100`. Actual eligibility depends on the resulting profile's minimum score and other custom formats; this penalty alone does not guarantee a non-English fallback.

The config enables deletion of old managed custom formats and resets unmatched profile scores. Review local customizations before syncing. The service uses the image's scheduled sync; inspect logs for successful runs rather than assuming the container being up proves the sync worked.

```powershell
docker compose run --rm recyclarr sync
```

After syncing, assign the profile to existing media in Radarr/Sonarr and select it in Seerr for new requests as appropriate.

## Backups and recovery

Restore templates are setup references, not backups of users, databases, request history, or library state. Sonarr and Radarr write their own scheduled backups to `config/sonarr/Backups/scheduled` and `config/radarr/Backups/scheduled` (first seen 30 October 2026). Those live on the same disk as the data they protect, so they only count as backups once copied elsewhere. Backups elsewhere and Windows backup jobs were not checked.

Back up the following to separate storage:

- `docker-compose.yml`, `.env`, `config-templates/`, and the entire `config/` tree. Protect backups containing API keys, provider credentials, and the tunnel token.
- The actual Seerr named volume, using a Docker volume backup procedure. Stop Seerr while making a filesystem copy of its SQLite database, and restore into the volume mounted at `/app/config`.
- The external Plex configuration/database, using a procedure appropriate to that Windows installation.
- Media libraries, if they need protection against disk loss. Keep enough free space for downloads, unpacking, and import copies.

For a consistent filesystem backup of app databases, stop their containers first and start them again after copying. Verify recovery on a separate test instance before depending on a backup. Never start a second instance against the same live database files.

To rebuild without full app backups:

1. Restore this project and supply a new or recovered `.env` tunnel token. Create the Windows media directories if needed.
2. Run `docker compose up -d` to initialize the apps.
3. Secure the apps and apply the NZBGet values from `config-templates/nzbget-restore-settings.txt` through its UI. This file is not a complete `nzbget.conf` and must not replace it.
4. Configure indexers, root folders, and download clients using the intended wiring above. Supply account credentials and API keys.
5. Copy the Recyclarr template to its live config path, replace its API-key placeholders, and run a sync.
6. Configure Seerr's Plex/Radarr/Sonarr connections, request permissions, and default profiles. Configure the Cloudflare hostname if needed.
7. Test one request through download, import, Plex playback, and Seerr availability.

With full backups, restore the app data and Seerr volume while the affected containers are stopped, preserving paths and checking image compatibility before starting. Templates alone will not recover previous users or request history.

## Operations

Run from `F:\ArrStack`:

```powershell
# Start / apply configuration changes
docker compose up -d

# Status and recent logs
docker compose ps
docker compose logs --tail 100 sonarr radarr nzbget seerr recyclarr cloudflared

# Update images: edit the pinned tag(s) in docker-compose.yml first,
# take a backup and review release notes, then pull and recreate
docker compose pull
docker compose up -d

# Stop and remove containers/networks; retain named volumes
docker compose down
```

All image tags are pinned, so `docker compose pull` changes nothing until a tag in `docker-compose.yml` is edited. Updating can change behavior or migrate databases; to roll back, restore the previous tag and recreate (an app database that has already migrated may not accept the older version). Check the images' release pages for new versions, or run a notifier such as Diun for alerts.

**Do not use `docker compose down -v` for a normal shutdown:** it removes the Seerr named volume and its settings, users, and request history.

## Troubleshooting

- **Missing download directory:** check category output paths remain inside `/data/Downloads`, exist, and match the mounts. `Series` should write to `/data/Downloads/completed/tv`.
- **"Missing root folder for movie collection":** Radarr collections store their own root folder, separate from the movies, and the Movie Editor does not update it. Movies → Collections → Edit Collections → Select All → Root Folder → `/data/Movies` → Apply.
- **Imports slow or disk fills during import:** the series/movie path or the download path is on a legacy mount (`/tv`, `/movies`, `/downloads`) instead of `/data`, so the import copies instead of renaming. Repoint the root folder and item paths to `/data/...`.
- **Wrong category or import stuck:** compare the download client's category with NZBGet's exact category name. Check completed-download handling and app logs.
- **Cannot connect between containers:** use service names and container ports. `localhost` refers to the calling container itself.
- **Slow downloads:** check provider errors, connection limits, disk space, and repair/unpack activity. The current baseline is 40 connections; performance depends on the workload and hardware.
- **Seerr unavailable externally:** check connector logs and Cloudflare hostname routing, then check Seerr locally.
- **Request downloaded but unavailable:** confirm the import succeeded, Plex scanned the intended Windows folder, and Seerr completed its Plex sync.
