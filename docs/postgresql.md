# PostgreSQL

From <https://www.postgresql.org/>:

> PostgreSQL is a powerful, open source object-relational database system with over 35 years of active development that has earned it a strong reputation for reliability, feature robustness, and performance.

Security Onion uses PostgreSQL as its central data platform, running on manager nodes. It serves as the primary relational database for the Security Onion platform.

## Storage

PostgreSQL data is stored at `/nsm/postgres/` on the manager node. Configuration files are located at `/opt/so/conf/postgres/` and logs at `/opt/so/log/postgres/`.

!!! warning

    Do not manually modify files in `/nsm/postgres/` as this could corrupt the database.

## Authentication

PostgreSQL uses a two-tier authentication model:

- **`postgres` superuser** — used for administrative operations only. The password is auto-generated during installation and stored in the Salt secrets pillar.
- **`so_postgres` application user** — used by the Security Onion platform for day-to-day database access. The password is auto-generated and stored in the Salt auth pillar.

Both sets of credentials are automatically managed and should not need manual intervention.

## Encryption

All connections to PostgreSQL are encrypted with TLS using certificates signed by the Security Onion CA. The certificate and key are located at `/etc/pki/postgres.crt` and `/etc/pki/postgres.key`.

## Configuration

You can configure PostgreSQL by going to [Administration](administration.md) --> Configuration --> postgres.

Key settings include:

| Setting | Default | Description |
|---------|---------|-------------|
| `listen_addresses` | `*` | Addresses to listen on (firewall controls access) |
| `port` | `5432` | PostgreSQL listen port |
| `max_connections` | `100` | Maximum concurrent connections |
| `shared_buffers` | `256MB` | Memory for shared buffer cache |
| `ssl` | `on` | Enable TLS encryption |

## Firewall

PostgreSQL is only accessible from the manager node itself. Access is controlled by the Security Onion [firewall](firewall.md) via the `DOCKER-USER` iptables chain. Non-manager nodes cannot reach port 5432.

## Management Commands

Security Onion provides several command-line tools for managing PostgreSQL. These tools run `psql` inside the [Docker](docker.md) container so that PostgreSQL client tools do not need to be installed on the host.

### Interactive Shell

Open an interactive `psql` session:

```bash
sudo so-postgres-manage shell
```

### Execute SQL

Run a SQL command directly:

```bash
sudo so-postgres-manage sql "SELECT version();"
```

### Execute SQL File

Run a SQL file:

```bash
sudo so-postgres-manage sqlfile /path/to/script.sql
```

### List Databases

```bash
sudo so-postgres-manage dblist
```

### List Database Roles

```bash
sudo so-postgres-manage userlist
```

### Container Lifecycle

```bash
sudo so-postgres-start
sudo so-postgres-stop
sudo so-postgres-restart
```

## Extensions

The following PostgreSQL extensions are installed and available:

| Extension | Description |
|-----------|-------------|
| **pgvector** | Vector similarity search for AI/ML embeddings |
| **pg_cron** | Job scheduling within PostgreSQL |
| **pg_partman** | Automated table partition management |
| **pgcrypto** | Cryptographic functions for column-level encryption |

## Backup

PostgreSQL is automatically backed up daily at 00:05. The backup uses `pg_dumpall` to capture all databases and roles, compressed with gzip. Backup files are stored at `/nsm/backup/so-postgres-backup-YYYY_MM_DD.sql.gz` with 7-day retention.

For more information about backups, see the [Backup](backup.md) page.

To manually create a backup:

```bash
docker exec so-postgres pg_dumpall -U postgres | gzip > /nsm/backup/so-postgres-manual-backup.sql.gz
```

To restore from a backup:

```bash
zcat /nsm/backup/so-postgres-backup-2025_03_17.sql.gz | docker exec -i so-postgres psql -U postgres
```

## Diagnostic Logging

PostgreSQL logs can be found at `/opt/so/log/postgres/`. Depending on what you're looking for, you may also need to look at the [Docker](docker.md) logs for the container:

```bash
sudo docker logs so-postgres
```

## More Information

!!! note

    For more information about PostgreSQL, please see <https://www.postgresql.org/docs/>.
