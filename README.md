# Valheim Dedicated Server (mbround18/valheim-docker)

Uses the [mbround18/valheim-docker](https://github.com/mbround18/valheim-docker) image
(Odin manager) with auto-update, auto-backup, and Discord webhook notifications.

## Directory layout

```
docker/
├── docker-compose.yml
├── .env                  # your config (copy from .env.example)
└── data/
    ├── saves/            # -> /home/steam/.config/unity3d/IronGate/Valheim  (your world lives here)
    ├── server/           # -> /home/steam/valheim                          (game install, managed by Odin)
    └── backups/          # -> /home/steam/backups                          (auto backups land here)
```

Create the data dirs before first start:

```sh
mkdir -p data/saves data/server data/backups
```

## 1. Copying your local world in

Your world is two files: `<WorldName>.db` and `<WorldName>.fwl`.

**Where they live locally (Mac):**

| Install type | Path |
|---|---|
| Steam (no Steam Cloud) | `~/Library/Application Support/Steam/userdata/<steamid>/892970/remote/worlds_local/` |
| Standalone/non-Steam-Cloud | `~/Library/Application Support/IronGate/Valheim/worlds_local/` |
| Windows | `%USERPROFILE%\AppData\LocalLow\IronGate\Valheim\worlds_local\` |

Copy them into the saves volume **under `worlds_local/`**:

```sh
mkdir -p data/saves/worlds_local
cp "<path>/MyWorld.db" "<path>/MyWorld.fwl" data/saves/worlds_local/
```

Then in `.env` set `WORLD=MyWorld` — this must match the filename **exactly**
(case-sensitive on Linux; match the original casing to be safe).

> If you've never launched the world on a dedicated server before and only have
> the old-style `worlds/` folder, use that folder name instead — the game
> migrates it automatically. `worlds_local` is the current standard.

## 2. Running locally on macOS

Prereq: Docker Desktop for Mac.

```sh
cd docker
cp .env.example .env    # edit values
mkdir -p data/saves data/server data/backups
# copy your world in (see above)
docker compose up -d
docker compose logs -f  # wait for "DungeonDB" / server started lines
```

**Mac-specific notes:**

- **UID/GID**: Docker Desktop runs a VM with ownership mapping, so
  `PUID=1000/PGID=1000` works even though your macOS UID is likely 501. Leave the
  defaults; no `chown` needed.
- **Networking**: clients on your LAN connect to `<your-mac-ip>:2456`. Internet
  play requires forwarding UDP 2456–2458 on your router to your Mac, and
  allowing it through macOS Firewall for Docker.
- **Sleep**: the server dies when the Mac sleeps — fine for testing, bad for
  hosting. Use the VPS path for anything serious.
- **Crossplay**: if you need Xbox/MS Store crossplay, add `ENABLE_CROSSPLAY=1`
  to the environment section (it's not in the compose file by default).

## 3. Running on a Linux VPS

```sh
git clone <this repo> && cd docker   # or scp the docker/ folder over
cp .env.example .env                 # edit values
id -u; id -g                         # set PUID/PGID in .env to these
mkdir -p data/saves data/server data/backups
sudo chown -R 1000:1000 data         # match PUID:PGID (rootless image!)

# Upload your world files first:
#   scp MyWorld.db MyWorld.fwl user@vps:~/docker/data/saves/worlds_local/

docker compose up -d
```

**VPS-specific notes:**

- **PUID/PGID matter**: unlike Docker Desktop, bind mounts on Linux use real
  host ownership. The image is rootless — if `data/` is root-owned the
  container will crash on permissions. `chown` to match `.env`.
- **Firewall**: open UDP 2456–2458:
  ```sh
  sudo ufw allow 2456:2458/udp
  ```
  Also open them in your provider's security group / network firewall
  (AWS SG, GCP firewall rules, Hetzner firewall, etc.).
- **Swap**: Valheim + SteamCMD update wants ~4GB RAM. On small instances add a
  2GB swapfile to survive auto-updates.

## 4. PaaS Docker hosts (Railway, Fly.io, ECS, etc.)

- Most container PaaSes only forward **TCP** — Valheim needs **UDP 2456–2458**.
  Fly.io supports UDP; Railway/Render/Heroku generally do not. Check before
  committing.
- Replace the three bind mounts with **named volumes** or the platform's
  persistent volume feature; upload your world via the platform's file copy
  (`fly ssh sftp shell`, etc.) into `<vol>/.config/unity3d/IronGate/Valheim/worlds_local/`.
- Set `PUID/PGID` per the platform's guidance; some platforms force their own
  user and ignore the `user:` directive.
- `restart: unless-stopped` and `stop_signal: SIGINT` still apply — keep them.
- Health checks: the image bundles **Huginn**, an HTTP sidecar (default port
  3000) with `/health`, `/readiness`, `/liveness`, `/status`. Expose env
  `HTTP_PORT` and point the platform's health check there.

## 5. How the QoL features integrate with the host

Everything below runs **inside the container** via Odin's scheduler — the host
only needs Docker and the compose file; **no host cron required** for core
features.

| Feature | Where it runs | Host-level touchpoints |
|---|---|---|
| `AUTO_UPDATE` | Container checks Steam on cron, shuts down, updates via SteamCMD, restarts | Host needs enough RAM/disk headroom during updates; `restart: unless-stopped` covers crash recovery only |
| `UPDATE_ON_STARTUP` | Runs SteamCMD validate on every container start | After `docker compose pull`, `up -d` gets you the latest server build |
| `AUTO_BACKUP` | Zips the world into `data/backups/` on cron | **Host-level offsite copy recommended** (below) |
| `AUTO_BACKUP_ON_UPDATE/SHUTDOWN` | Pre-update and graceful-stop backups | `stop_grace_period: 2m` gives it time; don't `docker kill` |
| `SCHEDULED_RESTART` | Daily restart keeps memory sane | Optional |
| `WEBHOOK_URL` | Discord pings on start/stop/update/backup (+ player join/leave) | — |

### Host-level offsite backups (recommended)

In-container backups protect against world corruption but **not** against host
loss. Add a host cron / systemd timer to ship `data/backups/` offbox:

```cron
# /etc/cron.d/valheim-backup (Linux VPS)
15 * * * * root rsync -az --delete /srv/valheim/docker/data/backups/ user@offsite:/backups/valheim/
```

or use [restic](https://restic.net/) to B2/S3 for versioned, encrypted backups:

```cron
30 * * * * root restic -r s3:s3.amazonaws.com/my-bucket backup /srv/valheim/docker/data/backups --exclude-caches
```

On macOS locally, a launchd job or just `rsync` to an external drive does the
same job.

### Restoring a backup

```sh
cd docker
docker compose down
# backups are zips named by date; extract the .db/.fwl into:
unzip data/backups/<file>.zip -d data/saves/worlds_local/
docker compose up -d
```

## 6. Useful commands

```sh
docker compose logs -f valheim        # follow logs
docker compose exec valheim odin status   # server status
docker compose down                   # graceful stop (triggers shutdown backup)
docker compose pull && docker compose up -d  # update the image itself
```

## Gotchas

- Password: min 5 chars, must **not** contain the server name.
- `WORLD` must match your save filename exactly.
- The server listens on `PORT`, `PORT+1`, `PORT+2` (2456–2458) — forward all three, UDP only.
- Don't edit anything under `data/server/` — Odin/SteamCMD owns it and updates will clobber it.
