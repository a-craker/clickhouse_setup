# ClickHouse Setup

**Author:** Tivan - updated by Aidan

**Date:** 2025-08-05

---

## Introduction

At the time of writing, ClickHouse describes itself as "the fastest and most resource efficient real-time data warehouse and open-source database."

## Installing ClickHouse with Docker

We use [Docker to install ClickHouse](https://clickhouse.com/docs/install/docker) because it is quick and relatively easy. We do so using a `docker-compose.yml` file and pre-defined configuration files to ensure the setup is ready to go with minimal manual tweaks.

### Directory Structure

The project directory is structured as follows:

```text
├── ch_backup
│   └── backup_disk.xml
├── ch_config
│   ├── default_database.xml
│   ├── disk_limits.xml
│   └── listenhost_config.xml
├── ch_data         # Database storage (See Data & Logs section)
├── ch_logs         # Application logs (See Data & Logs section)
├── ch_users
│   └── default-user.xml
├── docker-compose.yml
└── .env

```

---

## Data and Logs Folders

In this deployment, we map local directories to the container to ensure data persistence and easy log access.

### `ch_data`

This folder maps to `/var/lib/clickhouse/`. It contains the actual database engine data, including:

* **Store/Data:** The physical parts and columns of your tables.
* **Metadata:** Table schemas and structural definitions.
* **Access:** User and role definitions created via SQL.

### `ch_logs`

This folder maps to `/var/log/clickhouse-server/`. It contains `clickhouse-server.log` (general information) and `clickhouse-server.err.log` (error reporting). Monitoring these is essential for debugging performance or connection issues.

### Permissions Notice

ClickHouse runs inside the container as user `clickhouse` (typically UID `101`). For the container to write data and logs:

* **Linux/Mac:** You may need to change ownership of these folders:
```bash
sudo chown -R 101:101 ch_data ch_logs

```


* Alternatively, ensure the folders have broad write permissions (`chmod 777`) if you are in a restricted development environment.

---

## docker-compose.yml

The `docker-compose.yml` handles the container orchestration.

* **Image:** Uses the latest stable ClickHouse image.
* **Ports:**
* `9000`: Native TCP protocol (for clickhouse-client and high-performance drivers).
* `8123`: HTTP protocol (for web interfaces and SQLTools).
* `5432`: PostgreSQL protocol compatibility.


* **Environment:** Uses a `.env` file to manage `CLICKHOUSE_USER` and `CLICKHOUSE_PASSWORD`.

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:25.5.9.14
    container_name: clickhouse-server
    restart: unless-stopped
    ports:
      - "127.0.0.1:19000:9000"
      - "127.0.0.1:18123:8123"
      - "127.0.0.1:15432:5432"
    volumes:
      - ./ch_data:/var/lib/clickhouse/
      - ./ch_logs:/var/log/clickhouse-server/
      - ./ch_config:/etc/clickhouse-server/config.d/
      - ./ch_users:/etc/clickhouse-server/users.d/
      - ./ch_backup:/backup/
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    logging:      
      driver: "json-file"      
      options:        
        max-size: "100m"        
        max-file: "10"
    environment:
      CLICKHOUSE_DB: warehouse
      CLICKHOUSE_USER: ${CLICKHOUSE_USER}
      CLICKHOUSE_PASSWORD: ${CLICKHOUSE_PASSWORD}
      CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT: 1

```

---

## Config Files

ClickHouse merges files in `config.d/` and `users.d/` alphabetically into the main configuration.

### `default_database.xml`

Sets the default database at startup.

```xml
<clickhouse>
    <default_database>warehouse</default_database>
</clickhouse>

```

### `disk_limits.xml`

Sets the temporary path and the max size for temporary files (100GB).

```xml
<clickhouse>
    <tmp_path>/var/lib/clickhouse/tmp/</tmp_path>
    <max_temporary_files_size>100000000000</max_temporary_files_size>
</clickhouse>

```

### `listenhost_config.xml`

Allows connections from any host (0.0.0.0) to facilitate communication on internal networks.

### `default-user.xml`

This file deactivates the default ClickHouse user to enforce security. The primary administrative user is instead defined via environment variables in the `docker-compose.yml`.

### `backup_disk.xml`

Defines a local disk named `backups` pointing to the `/backup/` directory inside the container.

---

## Running the Container

Navigate to the directory and run:

```bash
docker compose up -d

```

To test the connection, use **SQLTools** in VSCode with the ClickHouse driver, connecting via `localhost:18123`.

---

## ClickHouse Terminology

### Parts

ClickHouse stores data in **parts**. Each part is a folder containing compressed binary files for every column. ClickHouse merges these parts in the background to optimize storage.

### Granules

A granule is the smallest indivisible dataset (default 8,192 rows) that ClickHouse reads. Granules are indexed by the **sorting key**.

### Sorting Key

Defined in the `ORDER BY` clause, this determines how data is physically sorted within parts, allowing ClickHouse to skip irrelevant granules during queries.

---

## Backup & Restore

### Backup

Run the following SQL to create a backup on the defined backup disk:

```sql
BACKUP DATABASE warehouse TO Disk('backups', 'warehouse_backup_2025');

```

### Restore

To restore to a new instance:

1. Place the backup files in the `ch_backup` folder.
2. Create the database: `CREATE DATABASE warehouse;`
3. Run the restore:

```sql
RESTORE DATABASE warehouse FROM Disk('backups', 'warehouse_backup_2025');

```

