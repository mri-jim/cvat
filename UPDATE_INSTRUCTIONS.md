# CVAT Update Instructions

## Current deployment

- CVAT 2.82.0 is built from source in `/home/mi-cvat/Downloads/cvat-v2.82.0`.
- The original checkout at `/home/mi-cvat/Downloads/cvat` remains on its
  original branch; the release checkout is a separate, detached Git worktree.
- The release checkout's `.env` sets `CVAT_HOST=10.0.10.103` and
  `CVAT_VERSION=v2.82.0-local`. Its `docker-compose.override.yml` preserves
  the existing data volume and read-only SMB share. Preserve both files.
- Access the instance at `http://10.0.10.103:8080/`. HTTPS requires a
  separately configured TLS proxy.

## Starting and checking the deployment

Run commands from the release checkout:

```bash
cd /home/mi-cvat/Downloads/cvat-v2.82.0
docker compose -f docker-compose.yml -f docker-compose.dev.yml -f docker-compose.override.yml -f components/serverless/docker-compose.serverless.yml up -d --no-build
docker compose -f docker-compose.yml -f docker-compose.dev.yml -f docker-compose.override.yml -f components/serverless/docker-compose.serverless.yml ps
docker exec cvat_server python /opt/cvat/manage.py health_check
docker exec cvat_server python /opt/cvat/manage.py shell -c 'from cvat import __version__; print(__version__)'
```

To rebuild the current checkout from source, use:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml -f docker-compose.override.yml -f components/serverless/docker-compose.serverless.yml build cvat_server cvat_ui
```

Building alone does not recreate running containers. Rebuilding the same image
tag also replaces the old tag, so preserve rollback images first.

## Before another upgrade

1. Schedule downtime and verify that `/mnt/smb_10T` is mounted to the intended
   SMB share. Never start the stack against an empty local mountpoint.
2. Stop all writers and back up the database, media, keys, logs, Redis, Kvrocks,
   ClickHouse, Nuclio storage, and deployment configuration. Verify the backup
   and ensure the read-only SMB share is backed up separately.
3. Prepare the complete target release source and review its upgrade notes.
   Preserve the data volume name and SMB bindings in the release override.
   Build both images with `docker-compose.dev.yml` under a distinct image tag.
4. Start with all four Compose files and monitor `docker logs cvat_server`.
   Database migrations run at startup; do not interrupt them.
5. Check health, version, workers, data access, and any Nuclio functions.

The verified pre-upgrade archives for the 2026-11-12 update are in
`/home/mi-cvat/cvat-backup-20261112`. They contain sensitive data and are
restricted to the local user. They do not include the broader read-only SMB
share. Archive checksums are in `SHA256SUMS`; these checks do not replace a
restore test or an independent backup.

Do not use `docker compose down -v`. Returning to the previous CVAT image
after a database migration requires restoring the corresponding backup, not
merely switching the image tag.

See the [upstream upgrade guide](https://docs.cvat.ai/docs/administration/community/advanced/upgrade_guide/)
and [backup guide](https://docs.cvat.ai/docs/administration/community/advanced/backup_guide/).
