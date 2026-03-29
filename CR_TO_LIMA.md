# Migration: Vagrant+QEMU to Lima

## Why Migrate

Lima uses Apple's native Virtualization.framework instead of QEMU emulation — faster boot, better performance, and virtiofs for instant file sharing (no rsync). Simpler setup: `brew install lima` replaces Vagrant + QEMU + vagrant-qemu plugin.

## What Changes

| Aspect | Current (Vagrant) | New (Lima) |
|--------|-------------------|------------|
| Config file | `Vagrantfile` (Ruby) | `lima.yaml` (YAML) |
| Provider | QEMU via vagrant-qemu plugin | Apple Virtualization.framework |
| File sharing | rsync (one-way, on-demand) | virtiofs (instant, bidirectional) |
| Port forwarding | Manual SSH tunnel (trigger hack) | Automatic — Lima auto-forwards guest ports to localhost |
| DB backup trigger | Vagrant triggers (`before :halt`) | Manual script or cron (see below) |
| Prerequisites | Vagrant + QEMU + vagrant-qemu | `brew install lima` |
| Commands | `vagrant up/halt/destroy/ssh` | `limactl start/stop/delete/shell` |

## Step 1 — Create `lima.yaml` (replaces `Vagrantfile`)

```yaml
base:
  - template:alpine

vmType: "vz"
cpus: 2
memory: "4GiB"

mountType: "virtiofs"
mounts:
  - location: "."
    mountPoint: "/vagrant"
    writable: true

containerd:
  system: false
  user: false

provision:
  - mode: system
    script: |
      #!/bin/sh
      set -e

      apk update
      apk add postgresql postgresql-client openssl

      rc-update add postgresql default > /dev/null
      rc-service postgresql start 2>/dev/null

      PG_CONF="/etc/postgresql/postgresql.conf"
      PG_DATA=$(sudo -u postgres psql -tAc "SHOW data_directory;")
      PG_HBA="$PG_DATA/pg_hba.conf"

      sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" "$PG_CONF"

      openssl req -new -x509 -days 3650 -nodes \
        -subj "/CN=pg-dev" \
        -keyout "$PG_DATA/server.key" \
        -out "$PG_DATA/server.crt" 2>/dev/null
      chown postgres:postgres "$PG_DATA/server.key" "$PG_DATA/server.crt"
      chmod 600 "$PG_DATA/server.key"
      chmod 644 "$PG_DATA/server.crt"

      cat >> "$PG_CONF" <<EOF
      ssl = on
      ssl_cert_file = '$PG_DATA/server.crt'
      ssl_key_file = '$PG_DATA/server.key'
      EOF

      echo "hostssl all all 0.0.0.0/0 md5" >> "$PG_HBA"
      rc-service postgresql restart 2>/dev/null

      sudo -u postgres psql -tc "SELECT 1 FROM pg_roles WHERE rolname='staging'" | grep -q 1 \
        || sudo -u postgres psql -c "CREATE USER staging WITH PASSWORD '12312312';"
      sudo -u postgres psql -tc "SELECT 1 FROM pg_database WHERE datname='defaultdb'" | grep -q 1 \
        || sudo -u postgres psql -c "CREATE DATABASE defaultdb OWNER staging;"
      sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE defaultdb TO staging;"

      if [ -f /vagrant/db/backup.sql ]; then
        echo "Restoring database from backup..."
        sudo -u postgres psql defaultdb < /vagrant/db/backup.sql
        echo "Restore complete."
      fi

      echo "PostgreSQL is ready (SSL enabled)."
      echo "  Host:     localhost:5432"
      echo "  User:     staging"
      echo "  Password: 12312312"
      echo "  Database: defaultdb"
      echo "  SSL:      on (self-signed, use sslmode=require)"
```

## Step 2 — Port Forwarding

No SSH tunnel hack needed. Lima automatically forwards any port the guest listens on to localhost. PostgreSQL on guest `:5432` will be available at `localhost:5432` on the host automatically.

Remove the entire `config.trigger.after :up` SSH tunnel block from the current setup.

## Step 3 — DB Backup on Stop

Lima doesn't have lifecycle triggers like Vagrant's `before :halt`. Options:

**Option A (simple) — Manual backup script:**
```bash
#!/bin/sh
# backup.sh
limactl shell pg-dev -- sudo -u postgres pg_dump defaultdb > db/backup.sql
```

**Option B (automatic) — Cron inside VM:**
Since `./db` is shared via virtiofs, a cron job inside the VM can dump periodically:
```bash
# Inside VM crontab:
*/5 * * * * sudo -u postgres pg_dump defaultdb > /vagrant/db/backup.sql
```

**Option C (wrapper script) — Dump then stop:**
```bash
#!/bin/sh
# pg-stop.sh
limactl shell pg-dev -- sudo -u postgres pg_dump defaultdb > db/backup.sql
limactl stop pg-dev
```

## Step 4 — Update README.md Commands

| Old | New |
|-----|-----|
| `vagrant up` | `limactl start --name=pg-dev ./lima.yaml` |
| `vagrant halt` | `limactl stop pg-dev` (run backup first) |
| `vagrant destroy` | `limactl delete pg-dev` (run backup first) |
| `vagrant ssh` | `limactl shell pg-dev` |

## Step 5 — Update Prerequisites

Remove:
- Vagrant
- QEMU (`brew install qemu`)
- vagrant-qemu plugin

Add:
- Lima (`brew install lima`)

## Step 6 — Delete `Vagrantfile`, Add `lima.yaml`

## Key Benefit of virtiofs

The `./db` directory is now instantly bidirectional. No more rsync — `backup.sql` written inside the VM appears on the host immediately. Restore on startup reads directly from the host. No sync delays or missed files.
