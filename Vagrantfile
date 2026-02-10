# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV["VAGRANT_DEFAULT_PROVIDER"] ||= "qemu"

Vagrant.configure("2") do |config|
  config.vm.box = "generic/alpine318"
  config.vm.hostname = "pg-dev"

  config.vm.provider "qemu" do |qe|
    qe.smp = 2
    qe.memory = 4096
  end

  # Shared folder for DB dump persistence (QEMU provider uses rsync)
  config.vm.synced_folder "./db", "/vagrant/db", create: true, type: "rsync"

  # Provision PostgreSQL
  config.vm.provision "shell", inline: <<-SHELL
    set -e

    apk update
    apk add postgresql postgresql-client openssl

    # Start the service (auto-initializes the cluster)
    rc-update add postgresql default > /dev/null
    rc-service postgresql start 2>/dev/null

    # Alpine reads config from /etc/postgresql, not the data directory
    PG_CONF="/etc/postgresql/postgresql.conf"
    PG_DATA=$(sudo -u postgres psql -tAc "SHOW data_directory;")
    PG_HBA="$PG_DATA/pg_hba.conf"

    # Configure PostgreSQL to accept remote connections
    sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" "$PG_CONF"

    # Generate self-signed SSL cert in the data directory
    openssl req -new -x509 -days 3650 -nodes \
      -subj "/CN=pg-dev" \
      -keyout "$PG_DATA/server.key" \
      -out "$PG_DATA/server.crt" 2>/dev/null
    chown postgres:postgres "$PG_DATA/server.key" "$PG_DATA/server.crt"
    chmod 600 "$PG_DATA/server.key"
    chmod 644 "$PG_DATA/server.crt"

    # Enable SSL
    cat >> "$PG_CONF" <<EOF
ssl = on
ssl_cert_file = '$PG_DATA/server.crt'
ssl_key_file = '$PG_DATA/server.key'
EOF

    # Require SSL for remote connections
    echo "hostssl all all 0.0.0.0/0 md5" >> "$PG_HBA"
    rc-service postgresql restart 2>/dev/null

    # Create user, database
    sudo -u postgres psql -tc "SELECT 1 FROM pg_roles WHERE rolname='staging'" | grep -q 1 \
      || sudo -u postgres psql -c "CREATE USER staging WITH PASSWORD '12345678';"
    sudo -u postgres psql -tc "SELECT 1 FROM pg_database WHERE datname='defaultdb'" | grep -q 1 \
      || sudo -u postgres psql -c "CREATE DATABASE defaultdb OWNER staging;"
    sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE defaultdb TO staging;"

    # Restore from dump if one exists
    if [ -f /vagrant/db/backup.sql ]; then
      echo "Restoring database from backup..."
      sudo -u postgres psql defaultdb < /vagrant/db/backup.sql
      echo "Restore complete."
    fi

    echo "PostgreSQL is ready (SSL enabled)."
    echo "  Host:     localhost:5432"
    echo "  User:     staging"
    echo "  Password: 12345678"
    echo "  Database: defaultdb"
    echo "  SSL:      on (self-signed, use sslmode=require)"
  SHELL

  # Open SSH tunnel for PostgreSQL port forwarding (runs in background)
  config.trigger.after :up do |t|
    t.name = "Setting up SSH tunnel for PostgreSQL on localhost:5432..."
    t.ruby do |env, machine|
      info = machine.ssh_info
      spawn("ssh -f -N -L 5432:localhost:5432 " \
            "-p #{info[:port]} " \
            "-i #{info[:private_key_path].first} " \
            "-o StrictHostKeyChecking=no " \
            "-o UserKnownHostsFile=/dev/null " \
            "-o LogLevel=QUIET " \
            "#{info[:username]}@#{info[:host]}")
    end
  end

  # Dump database to host before halt/destroy
  config.trigger.before [:halt, :destroy] do |t|
    t.name = "Dumping PostgreSQL database to host..."
    t.ruby do |env, machine|
      info = machine.ssh_info
      Dir.mkdir("db") unless Dir.exist?("db")
      system("ssh -p #{info[:port]} " \
             "-i #{info[:private_key_path].first} " \
             "-o StrictHostKeyChecking=no " \
             "-o UserKnownHostsFile=/dev/null " \
             "-o LogLevel=QUIET " \
             "#{info[:username]}@#{info[:host]} " \
             "\"sudo su postgres -c 'pg_dump defaultdb'\" > db/backup.sql 2>/dev/null")
    end
  end
end
