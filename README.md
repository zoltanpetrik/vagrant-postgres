# PostgreSQL Dev VM

Alpine Linux 3.18 VM running PostgreSQL 15 via Vagrant + QEMU (Apple Silicon compatible).

## Prerequisites

- [Vagrant](https://www.vagrantup.com/)
- [QEMU](https://www.qemu.org/) (`brew install qemu`)
- vagrant-qemu plugin (`vagrant plugin install vagrant-qemu`)

## VM Specs

- 2 CPU cores, 4 GB RAM
- Alpine Linux 3.18 (ARM64)
- PostgreSQL 15

## Usage

```bash
vagrant up        # Start VM, provision PostgreSQL, open SSH tunnel
vagrant halt      # Stop VM (dumps database to ./db/backup.sql first)
vagrant destroy   # Delete VM (dumps database first)
vagrant ssh       # SSH into the VM
```

## Connecting to PostgreSQL

After `vagrant up`, an SSH tunnel forwards port 5432 to your localhost.

```
Host:     localhost
Port:     5432
User:     staging
Password: 12345678
Database: defaultdb
```

Connection string:

```
postgresql://staging:12345678@localhost:5432/defaultdb
```

CLI:

```bash
psql -h localhost -U staging -d defaultdb
```

If the tunnel isn't running (check with `lsof -i :5432`), start it manually:

```bash
vagrant ssh -- -f -N -L 5432:localhost:5432
```

## Data Persistence

The `./db/` directory is used for database backup persistence across VM rebuilds.

- On `vagrant halt` or `vagrant destroy`, the database is automatically dumped to `./db/backup.sql`
- On `vagrant up`, if `./db/backup.sql` exists, it is restored into the database
