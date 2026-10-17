# PostgreSQL Dev VM

Alpine Linux VM running PostgreSQL 18 via Vagrant + QEMU (Apple Silicon compatible).

## Prerequisites

- [Vagrant](https://www.vagrantup.com/)
- [QEMU](https://www.qemu.org/) (`brew install qemu`)
- vagrant-qemu plugin 0.6.4 (`vagrant plugin install vagrant-qemu --plugin-version 0.6.4`), see [Maintenance Notes](#maintenance-notes)

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

To move to a newer Alpine release, point `ALPINE_IMAGE_URL` at an aarch64 `uefi-cloudinit` qcow2 from `https://dl-cdn.alpinelinux.org/alpine/<version>/releases/cloud/`, set `ALPINE_IMAGE_SHA512` to the contents of the `.sha512` file next to it, and recreate the VM. The `clock` provisioner overwrites the image's `/etc/chrony/chrony.conf` with a copy of the image's default plus the changes described in [Guest Clock](#guest-clock), so check the new release's default for changes to carry over.

The PostgreSQL major version comes from the `postgresql18` package names in the provisioning script. Each major version keeps its cluster in its own directory, `/var/lib/postgresql/<major>/data`, and nothing migrates data between them.

## Guest Clock

QEMU on Apple Silicon stops the guest's clock while the Mac sleeps ([QEMU issue #864](https://gitlab.com/qemu-project/qemu/-/issues/864)), so after every wake the guest is behind by the length of the sleep. The hardware clock QEMU gives the guest (`/dev/rtc0`) keeps following the Mac's clock as long as nothing in the guest writes to it.

The `clock` provisioner sets chrony, which the image runs against `pool.ntp.org`, to step the clock whenever it is more than a second off (`makestep 1 -1`) and to check its servers every 64 seconds (`maxpoll 6`), so the guest should be right again within about a minute of a wake; this hasn't been measured across a real sleep. With the image's own settings, chrony steps the clock only in its first three updates after starting and otherwise speeds it up by at most 1/12, so catching up would take 12 times as long as the sleep. The provisioner also leaves out the image's `rtcsync`, which copies the guest's clock into the hardware clock every 11 minutes and so could write a stale time there right after a wake.

chrony can correct the clock only while it reaches its servers over UDP port 123. While the Mac is offline, or on a network that blocks that port, each sleep leaves the guest behind by its length until chrony reaches the servers again.

To apply the clock setup to an existing VM without rerunning the PostgreSQL provisioner, which restarts PostgreSQL and rewrites `pg_hba.conf`:

```bash
vagrant provision --provision-with clock
```

To check the clock, run the commands below. The first prints the guest's clock, the hardware clock and the Mac's clock, in that order; the Mac's is read after the connection closes, so it can be a second ahead. `chronyc` needs `doas` because chrony's network command port is disabled (`cmdport 0`).

```bash
vagrant ssh -c 'date -u +%s; cat /sys/class/rtc/rtc0/since_epoch'; date -u +%s
vagrant ssh -c 'doas chronyc tracking'
```

Rejected alternatives:

- A cron job that copies the hardware clock into the guest's clock every minute (`hwclock --hctosys`). The hardware clock counts whole seconds, so each copy set the clock back by up to a second, which chrony then sped the clock up to undo. When a stale time got into the hardware clock, most likely through `rtcsync` right after a wake, the copy held the guest to it: on 2026-10-09 the guest ran 5 minutes behind.
- chrony with the hardware clock as its only source (`refclock RTC`), which would keep working offline. It is untested, and chrony may read this guest's `rtc-efi` clock only to within half a second.

## Maintenance Notes

This setup was tested with Vagrant 2.4.9 and vagrant-qemu 0.6.4 on macOS 27.0.1. On macOS 27, vagrant-qemu releases before 0.6.4 can't start the VM. Before `vagrant up` starts QEMU, Vagrant checks that the SSH port, 50022, is free by trying to connect to it with Ruby's `Socket.tcp` and a `connect_timeout`. On macOS 27, that call in the Ruby 3.3 bundled with Vagrant 2.4.9 reports a refused connection as a successful one ([Ruby bug #22223](https://bugs.ruby-lang.org/issues/22223)), so every port looks taken and `vagrant up` stops with "The forwarded port to 50022 is already in use on the host machine". vagrant-qemu 0.6.4 replaces that check with its own. Vagrant tracks the bug in [issue #13845](https://github.com/hashicorp/vagrant/issues/13845).

`vagrant halt` asks the guest to power off and waits up to `graceful_timeout`, a vagrant-qemu provider setting this Vagrantfile leaves at its default, for QEMU to exit. If QEMU is still running then, it tells QEMU to quit, which stops the guest the way a power cut would, and after another `graceful_timeout` it kills QEMU. `vagrant destroy` stops the VM the same way before deleting it, so the SSH port is free again when either command returns.

The plugin is installed globally, so changing its version affects every Vagrant project on this machine.

These changes were considered and deferred, to avoid tooling changes for now:

- Replacing `build_seed_iso` in the Vagrantfile with Vagrant's built-in `config.vm.cloud_init`, which vagrant-qemu supports and which builds the seed with the same `hdiutil` command. This is untested: vagrant-qemu attaches that seed as a CD-ROM drive, which hasn't been tried with QEMU's ARM `virt` machine.
- Moving from Vagrant to Lima. It would mainly replace VM plumbing, so the plan was dropped; it is in git history (commit 4174031).
