# duplicity-backup

Containerized [duplicity](https://duplicity.gitlab.io/) backups to Backblaze B2, with a daily cron job and a Home Assistant `last-backup` timestamp ping. Multi-arch image (`amd64`, `arm64`, `arm/v7`) published to GitHub Container Registry as `ghcr.io/michalghomelab/duplicity-backup`.

## Quick start

```yaml
# compose.yaml
services:
  backup:
    image: ghcr.io/michalghomelab/duplicity-backup:latest
    restart: unless-stopped
    volumes:
      - /opt/stacks:/source/stacks
      - /opt/dockge:/source/dockge
      - ./cache:/root/.cache
    env_file:
      - .env
```

```ini
# .env
B2_ACCOUNT_ID=...
B2_APPLICATION_KEY=...
B2_BUCKET_NAME=your-bucket-name
PASSPHRASE=long-random-passphrase
SRC=/source
HA_TOKEN=your-home-assistant-long-lived-token
HA_URL=http://homeassistant.local:8123
```

A cron inside the container runs `backup.sh` every day at 04:00 UTC. It performs a weekly full backup (`--full-if-older-than 7D`), an incremental on the rest of the days, then removes anything older than 8 days. Finally it POSTs a timestamp to the `sensor.backup_sensor` entity in Home Assistant.

## Configuration

| Variable           | Required | Description                                                                 |
|--------------------|----------|-----------------------------------------------------------------------------|
| `B2_ACCOUNT_ID`    | yes      | B2 application key ID                                                       |
| `B2_APPLICATION_KEY` | yes    | B2 application key                                                          |
| `B2_BUCKET_NAME`   | yes      | Target bucket                                                               |
| `PASSPHRASE`       | yes      | GPG passphrase used by duplicity to encrypt the archive                     |
| `SRC`              | yes      | Path inside the container to back up (mount your data here)                 |
| `HA_TOKEN`         | yes      | Home Assistant long-lived access token                                      |
| `HA_URL`           | yes      | Home Assistant base URL                                                     |
| `BACKUP_INCLUDES`  | no       | Whitespace/newline-separated list of patterns to include before excludes. See [Excludes](#excludes). |
| `BACKUP_EXCLUDES`  | no       | Whitespace/newline-separated list of patterns to exclude. See [Excludes](#excludes). |

### Excludes

By default Home Assistant `.storage` directories are included before excludes are evaluated:

```
/source/**/.storage
/source/**/.storage/**
```

By default the following are skipped:

```
/source/**/.*/**
/source/stacks/jellyfin/config/data/metadata
/source/stacks/adwireguard/adguard/opt-adguard-work/data/querylog.json*
```

The hidden-directory exclude skips contents of directories such as `.cache`, but does not exclude standalone hidden files such as `.env`. Everything else under `/source` is included by duplicity's default file selection behavior.

Override by setting `BACKUP_INCLUDES` and/or `BACKUP_EXCLUDES`. YAML block scalars are the cleanest way:

```yaml
environment:
  BACKUP_INCLUDES: |
    /source/**/.storage
    /source/**/.storage/**
  BACKUP_EXCLUDES: |
    /source/**/.*/**
    /source/stacks/jellyfin/config/data/metadata
    /source/some/huge/cache
```

Patterns are passed to `duplicity --include`/`--exclude` in include, then exclude order and follow its glob rules.

## Restore

Restore runs on the host, not in the container — you need duplicity + `b2sdk` and the same `PASSPHRASE` you used when backing up.

```sh
sudo apt install duplicity python3-b2sdk

export PASSPHRASE='your-passphrase'
sudo -E duplicity restore --force \
  b2://<B2_ACCOUNT_ID>:<B2_APPLICATION_KEY>@<B2_BUCKET_NAME> \
  /opt
```

Notes:

- `--force` lets duplicity write into a non-empty target directory; drop it for a dry-run into an empty dir first if you're unsure.
- Restore the full archive into a scratch path (e.g. `/tmp/restore`) if you only need a subset, then copy out what you want.
- Without `PASSPHRASE` exported, duplicity will prompt for it interactively.

To restore a single file or directory, add `--file-to-restore <path-inside-backup>`:

```sh
sudo -E duplicity restore --force \
  --file-to-restore stacks/jellyfin/config \
  b2://<B2_ACCOUNT_ID>:<B2_APPLICATION_KEY>@<B2_BUCKET_NAME> \
  /tmp/restore/jellyfin-config
```

## Building locally

```sh
make release   # multi-arch buildx push to ghcr.io/michalghomelab/duplicity-backup:latest
```

CI publishes tagged releases automatically on tag push (`X.Y.Z`).
