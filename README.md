# PostgreSQL Dev VM

Alpine Linux VM running PostgreSQL 18 via Vagrant + QEMU (Apple Silicon compatible).

## Prerequisites

- [Vagrant](https://www.vagrantup.com/)
- [QEMU](https://www.qemu.org/) (`brew install qemu`)
- vagrant-qemu plugin (`vagrant plugin install vagrant-qemu`)

## VM Specs

- 2 CPU cores, 4 GB RAM
- Alpine Linux (ARM64) from Alpine's official cloud image, see [Base Image](#base-image)
- PostgreSQL 18 (SSL enabled, self-signed cert)

## Usage

```bash
vagrant up        # Start VM, provision PostgreSQL, open SSH tunnel
vagrant halt      # Stop VM
vagrant destroy   # Delete VM
vagrant ssh       # SSH into the VM
```

Inside the VM, use `doas` where you would use `sudo`, for example `doas -u postgres psql`. The image has no bash, and the `vagrant` user has `doas` rights only.

## Connecting to PostgreSQL

After `vagrant up`, an SSH tunnel forwards port 5432 to your localhost.

```
Host:     localhost
Port:     5432
User:     staging
Password: 12312312
Database: defaultdb
```

Every connection through the tunnel needs the password; SSL is optional.

Connection string:

```
postgresql://staging:12312312@localhost:5432/defaultdb?sslmode=require
```

CLI:

```bash
psql -h localhost -U staging -d defaultdb
```

If the tunnel isn't running (check with `lsof -i :5432`), start it manually:

```bash
vagrant ssh -- -f -N -L 5432:localhost:5432
```

## Base Image

Alpine publishes no Vagrant boxes, so the VM boots [Alpine's official cloud image](https://alpinelinux.org/cloud/) instead of a box. On `vagrant up`, the Vagrantfile downloads the image at `ALPINE_IMAGE_URL` into `.vagrant/images/` once, and stops if its SHA-512 differs from `ALPINE_IMAGE_SHA512`. The image has no `vagrant` user, so each `vagrant up` also builds a cloud-init seed with macOS's `hdiutil`, which creates that user on the VM's first boot.

To move to a newer Alpine release, point `ALPINE_IMAGE_URL` at an aarch64 `uefi-cloudinit` qcow2 from `https://dl-cdn.alpinelinux.org/alpine/<version>/releases/cloud/`, set `ALPINE_IMAGE_SHA512` to the contents of the `.sha512` file next to it, and recreate the VM.

The PostgreSQL major version comes from the `postgresql18` package names in the provisioning script. Each major version keeps its cluster in its own directory, `/var/lib/postgresql/<major>/data`, and nothing migrates data between them.

## Maintenance Notes

These changes were considered and deferred, to avoid tooling changes for now:

- Upgrading vagrant-qemu. This setup was tested with version 0.3.12. With it, `vagrant up` straight after `vagrant destroy` can fail because the exiting QEMU process still holds port 50022; waiting a moment avoids that. From 0.4.1, `vagrant halt` kills QEMU when a clean shutdown doesn't finish; whether that also prevents the port collision is untested. The plugin is installed globally, so an upgrade affects every Vagrant project on this machine.
- Replacing `build_seed_iso` in the Vagrantfile with Vagrant's built-in `config.vm.cloud_init`, which vagrant-qemu supports and which builds the seed with the same `hdiutil` command. This is untested: vagrant-qemu attaches that seed as a CD-ROM drive, which hasn't been tried with QEMU's ARM `virt` machine.
- Moving from Vagrant to Lima. It would mainly replace VM plumbing, so the plan was dropped; it is in git history (commit 4174031).
