# -*- mode: ruby -*-
# vi: set ft=ruby :

require "digest"
require "fileutils"
require "tmpdir"
require "yaml"

ENV["VAGRANT_DEFAULT_PROVIDER"] ||= "qemu"

# Alpine publishes no Vagrant boxes, so the VM boots Alpine's official cloud
# image directly. To change release, take an aarch64 "uefi-cloudinit" qcow2
# and its .sha512 from https://dl-cdn.alpinelinux.org/alpine/<version>/releases/cloud/
ALPINE_IMAGE_URL = "https://dl-cdn.alpinelinux.org/alpine/v3.24/releases/cloud/generic_alpine-3.24.1-aarch64-uefi-cloudinit-r0.qcow2"
ALPINE_IMAGE_SHA512 = "c80b2693236ac0c1d92078e5e6eb69edd015feac363811a4952a4366c6815a53a644d17b1e4de8302c52e6c8a7e197dbf1fb74c3bb64bfea096334c7c8a3b5ca"

IMAGE_DIR = File.join(__dir__, ".vagrant", "images")
ALPINE_IMAGE = File.join(IMAGE_DIR, File.basename(ALPINE_IMAGE_URL))
SEED_ISO = File.join(IMAGE_DIR, "seed.iso")

# Downloads the image to ALPINE_IMAGE. Raises, which stops `vagrant up`, if
# the download fails or its SHA-512 differs from ALPINE_IMAGE_SHA512.
def download_alpine_image
  partial = "#{ALPINE_IMAGE}.part"
  system("curl", "--fail", "--location", "--progress-bar", "--output", partial, ALPINE_IMAGE_URL) \
    or raise "Downloading #{ALPINE_IMAGE_URL} failed"
  actual = Digest::SHA512.file(partial).hexdigest
  if actual != ALPINE_IMAGE_SHA512
    File.delete(partial)
    raise "#{ALPINE_IMAGE_URL} has SHA-512 #{actual}, expected #{ALPINE_IMAGE_SHA512}"
  end
  File.rename(partial, ALPINE_IMAGE)
end

# Writes SEED_ISO, a cloud-init NoCloud seed (volume label "cidata") that the
# image reads on first boot. The image has no vagrant user, so the seed creates
# one that accepts Vagrant's insecure public keys; Vagrant swaps in a key of
# its own on first boot.
def build_seed_iso(hostname)
  user = {
    "name" => "vagrant",
    "shell" => "/bin/sh",
    "doas" => ["permit nopass vagrant"],
    # "*" rules out password logins without locking the account, because
    # Alpine's sshd refuses key logins to locked accounts when PAM is off.
    "hashed_passwd" => "*",
    "lock_passwd" => false,
    "ssh_authorized_keys" => File.readlines(Vagrant.source_root.join("keys", "vagrant.pub"), chomp: true),
  }
  Dir.mktmpdir("seed", IMAGE_DIR) do |dir|
    src = File.join(dir, "cidata")
    Dir.mkdir(src)
    File.write(File.join(src, "meta-data"), { "instance-id" => hostname, "local-hostname" => hostname }.to_yaml)
    File.write(File.join(src, "user-data"), "#cloud-config\n" + { "users" => [user] }.to_yaml)
    iso = File.join(dir, "seed.iso")
    system("hdiutil", "makehybrid", "-quiet", "-iso", "-joliet", "-default-volume-name", "cidata", "-o", iso, src) \
      or raise "Building #{SEED_ISO} with hdiutil failed"
    File.rename(iso, SEED_ISO)
  end
end

Vagrant.configure("2") do |config|
  config.vm.hostname = "pg-dev"

  config.vm.provider "qemu" do |qe|
    qe.image_path = ALPINE_IMAGE
    # The image's disk is small; cloud-init grows the root filesystem to this
    # size on first boot, and the VM's disk file only grows as data is written
    qe.disk_resize = "128G"
    qe.extra_qemu_args = %W(-drive file=#{SEED_ISO},if=virtio,format=raw,readonly=on)
    qe.smp = 2
    qe.memory = 4096
  end

  # The image ships doas instead of sudo, and no bash
  config.ssh.shell = "sh"
  config.ssh.sudo_command = "doas %c"

  # The VM needs no files from the host
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Downloads the image once, and rebuilds the seed on every `vagrant up` so a
  # new VM always boots with the seed this file describes
  config.trigger.before :up do |t|
    t.name = "Preparing the Alpine cloud image..."
    t.ruby do |env, machine|
      FileUtils.mkdir_p(IMAGE_DIR)
      download_alpine_image unless File.exist?(ALPINE_IMAGE)
      build_seed_iso(machine.config.vm.hostname)
    end
  end

  # QEMU on Apple Silicon stops the guest clock while the Mac sleeps; see
  # "Guest Clock" in the README. chrony steps the clock whenever it is more than
  # a second off (makestep 1 -1) and checks its servers every 64 s (maxpoll 6),
  # so the guest should be right again within about a minute of a wake. The
  # image's rtcsync is left out: its 11-minute copy of the guest clock into the
  # hardware clock, which follows the Mac's, could write a stale time there.
  # The other lines match the image's default.
  # Named so `vagrant provision --provision-with clock` can apply it alone.
  config.vm.provision "clock", type: "shell", inline: <<-SHELL
    set -e
    rc-service chronyd stop
    cat > /etc/chrony/chrony.conf <<EOF
# Written by the Vagrantfile provisioner; re-provisioning overwrites it.
confdir /etc/chrony/conf.d
pool pool.ntp.org iburst maxpoll 6
sourcedir /etc/chrony/sources.d
driftfile /var/lib/chrony/chrony.drift
ntsdumpdir /var/lib/chrony
logdir /var/log/chrony
maxupdateskew 100.0
makestep 1 -1
cmdport 0
EOF
    # chronyd starts at the rate saved in its drift file, which may have been
    # learned under another configuration
    rm -f /var/lib/chrony/chrony.drift
    # A per-minute `hwclock --hctosys` cron job, which existing VMs may have,
    # sets the clock back to the hardware clock's whole second and fights chrony
    if crontab -l 2>/dev/null | grep -q 'hwclock --hctosys'; then
      crontab -l | grep -v 'hwclock --hctosys' | crontab -
    fi
    rc-update add chronyd default > /dev/null
    rc-service chronyd start
  SHELL

  # Provision PostgreSQL
  config.vm.provision "shell", inline: <<-SHELL
    set -e

    apk update
    # postgresql18 by name: each major version keeps its cluster in
    # /var/lib/postgresql/<major>/data, so another major would start empty.
    # sudo: the image ships only doas, and the psql calls below use sudo -u.
    apk add postgresql18 postgresql18-client openssl sudo

    # Start the service (auto-initializes the cluster)
    rc-update add postgresql default > /dev/null
    rc-service postgresql start 2>/dev/null

    # Alpine reads config from /etc/postgresql, not the data directory
    PG_CONF="/etc/postgresql/postgresql.conf"
    PG_DATA=$(sudo -u postgres psql -tAc "SHOW data_directory;")
    PG_HBA="$PG_DATA/pg_hba.conf"

    # Configure PostgreSQL to accept remote connections
    sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" "$PG_CONF"

    # Generate self-signed SSL cert in the data directory (first provision only)
    if [ ! -f "$PG_DATA/server.crt" ]; then
      openssl req -new -x509 -days 3650 -nodes \
        -subj "/CN=pg-dev" \
        -keyout "$PG_DATA/server.key" \
        -out "$PG_DATA/server.crt" 2>/dev/null
      chown postgres:postgres "$PG_DATA/server.key" "$PG_DATA/server.crt"
      chmod 600 "$PG_DATA/server.key"
      chmod 644 "$PG_DATA/server.crt"
    fi

    # Enable SSL (first provision only)
    grep -q "^ssl = on" "$PG_CONF" || cat >> "$PG_CONF" <<EOF
ssl = on
ssl_cert_file = '$PG_DATA/server.crt'
ssl_key_file = '$PG_DATA/server.key'
EOF

    # Every TCP connection needs the password, including the SSH tunnel's,
    # which arrive from localhost; connections from other hosts also need SSL
    cat > "$PG_HBA" <<EOF
# Written by the Vagrantfile provisioner; re-provisioning overwrites it.
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             all                                     trust
local   replication     all                                     trust
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ::1/128                 scram-sha-256
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
hostssl all             all             0.0.0.0/0               scram-sha-256
EOF
    rc-service postgresql restart 2>/dev/null

    # Create user, database
    sudo -u postgres psql -tc "SELECT 1 FROM pg_roles WHERE rolname='staging'" | grep -q 1 \
      || sudo -u postgres psql -c "CREATE USER staging WITH PASSWORD '12312312';"
    # Grant CREATEDB (idempotent, so it also applies on re-provisioning of an existing VM)
    sudo -u postgres psql -c "ALTER ROLE staging CREATEDB;"
    sudo -u postgres psql -tc "SELECT 1 FROM pg_database WHERE datname='defaultdb'" | grep -q 1 \
      || sudo -u postgres psql -c "CREATE DATABASE defaultdb OWNER staging;"
    sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE defaultdb TO staging;"

    echo "PostgreSQL is ready (SSL enabled)."
    echo "  Host:     localhost:5432"
    echo "  User:     staging"
    echo "  Password: 12312312"
    echo "  Database: defaultdb"
    echo "  SSL:      on (self-signed, use sslmode=require)"
  SHELL

  # Alpine's sshd_config disables TCP forwarding, which the tunnel below needs
  config.vm.provision "shell", inline: <<-SHELL
    set -e
    echo "AllowTcpForwarding yes" > /etc/ssh/sshd_config.d/tunnel.conf
    rc-service sshd reload
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
end
