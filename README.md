# UniFi OS Server on Proxmox LXC

> Tested with UniFi OS Server 5.0.6 on Debian 13 (Trixie) LXC, Proxmox VE 8.x.

## Overview

The UniFi OS Server (UOS) installer has several issues when running inside an LXC container:

1. **sysctl errors** — The installer aborts when it can't set kernel parameters that are read-only in LXC namespaces
2. **`su` fails with `cannot set groups`** — LXC restricts the `setgroups()` syscall used by `su`, which Podman needs to switch users
3. **Podman can't mount `/proc`** — Nested containers need the LXC `nesting` feature enabled
4. **`/dev/net/tun` missing** — Podman needs TUN device access for VPN features
5. **Disk space** — The installer requires at least ~20 GB free (15 GB in `/home/` alone)

All of these are solvable with the workarounds documented below.

---

## Step 1 — Create the LXC Container

Create a **privileged** container in Proxmox with:

| Setting | Value | Notes |
|---------|-------|-------|
| Template | Debian 13 or Ubuntu 24.04+ | |
| Privileged | **Yes** | Unprivileged will not work |
| Disk | **32 GB+** | Installer needs ~20 GB free |
| RAM | **4 GB+** | UOS is heavier than legacy controller |
| CPU | **2+ cores** | |
| Swap | 512 MB | |
| Network | DHCP or static IP | |

### Via Proxmox CLI

```bash
pct create <CTID> <template> \
  --hostname unifi-os \
  --cores 2 \
  --memory 4096 \
  --swap 512 \
  --rootfs <storage>:<size_in_GB> \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 0
```

## Step 2 — Configure the LXC (on the Proxmox host)

Edit `/etc/pve/lxc/<CTID>.conf` and ensure these lines are present:

```ini
# Enable nesting (required for Podman to run nested containers)
features: nesting=1

# Allow TUN device access (required for Podman networking / VPN)
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

> **Important:** Do NOT set `lxc.apparmor.profile: unconfined` — this overrides the
> nesting feature and will cause Podman to fail with `mount proc: Operation not permitted`.

## Step 3 — Start the Container and Install Dependencies

```bash
pct start <CTID>
pct enter <CTID>
```

Inside the container:

```bash
apt update && apt dist-upgrade -y
apt install -y podman uidmap slirp4netns iptables curl wget ca-certificates
```

## Step 4 — Create the sysctl Wrapper

The UOS installer calls `sysctl --system` which fails on kernel parameters that are
read-only inside an LXC namespace. These parameters are not required for operation —
the installer just doesn't handle the errors gracefully.

**Back up and replace `/sbin/sysctl`:**

```bash
# Back up the original
cp /sbin/sysctl /bin/sysctl.orig

# Create the wrapper
cat > /sbin/sysctl << 'EOF'
#!/bin/sh
/bin/sysctl.orig "$@" 2>/dev/null || true
exit 0
EOF

chmod +x /sbin/sysctl
```

Verify it works (should produce no errors):

```bash
sysctl --system
```

## Step 5 — Create the `su` → `runuser` Wrapper

Inside LXC, `su` fails with `cannot set groups: Operation not permitted` because the
kernel restricts the `setgroups()` syscall. The `runuser` command achieves the same
result without calling `setgroups()`.

The UOS installer uses `su` internally to run Podman commands as the `uosserver` user,
so we need to transparently redirect `su` calls to `runuser`.

```bash
# Back up the original
cp /bin/su /bin/su.orig

# Create the wrapper
cat > /bin/su << 'WRAPPER'
#!/bin/bash
# Wrapper: redirect su to runuser for LXC compatibility
USER=""
CMD=""
SHELL=""
while [[ $# -gt 0 ]]; do
    case "$1" in
        -s) SHELL="$2"; shift 2;;
        -c) CMD="$2"; shift 2;;
        -l|-) shift;;
        --) shift; USER="$1"; shift;;
        *) USER="$1"; shift;;
    esac
done

if [ -n "$CMD" ]; then
    if [ -n "$SHELL" ]; then
        exec /usr/sbin/runuser -u "${USER:-root}" -- "$SHELL" -c "$CMD"
    else
        exec /usr/sbin/runuser -u "${USER:-root}" -- /bin/sh -c "$CMD"
    fi
else
    exec /usr/sbin/runuser -u "${USER:-root}" -- "${SHELL:-/bin/sh}"
fi
WRAPPER

chmod +x /bin/su
```

Verify it works:

```bash
# Create a test user
useradd -m testuser
su -c "whoami" testuser
# Should print: testuser
userdel -r testuser
```

## Step 6 — Download and Run the UOS Installer

Get the latest installer from the [Ubiquiti downloads page](https://www.ui.com/download/software/unifi-os-server).

```bash
mkdir -p /usr/local/src/unifiserver
cd /usr/local/src/unifiserver

# Download (replace URL with the latest version)
wget -O unifi-os-server \
  "https://fw-download.ubnt.com/data/unifi-os-server/1856-linux-x64-5.0.6-33f4990f-6c68-4e72-9d9c-477496c22450.6-x64"

chmod +x unifi-os-server
./unifi-os-server
```

Confirm with `y` when prompted. The installer will:

1. Create the `uosserver` user and group
2. Set up Podman configuration
3. Load the container image (~800 MB)
4. Start the UOS container
5. Wait for systemd readiness

A successful install ends with:

```
!!! INSTALLATION COMPLETE !!!
UOS Server is running at: https://<IP>:11443/
```

## Step 7 — Enable the Service and Verify

```bash
systemctl enable uosserver
systemctl status uosserver
```

Check ports are listening:

```bash
ss -tlnp | grep -E '(11443|8080|8443)'
```

## Step 8 — Access the Dashboard

Open in your browser: **`https://<container-ip>:11443/`**

Accept the self-signed certificate warning and complete the initial setup wizard.

---

## Key Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 11443 | TCP | UOS Web UI / Dashboard |
| 8080 | TCP | Device inform (adoption) |
| 8443 | TCP | UniFi Network Application UI |
| 3478 | UDP | STUN |
| 5514 | UDP | Syslog |
| 10003 | UDP | Device discovery |
| 6789 | TCP | Speed test |

## Uninstall / Reset

To completely remove UOS and start fresh:

```bash
uosserver-purge
```

This removes all containers, images, volumes, user accounts, and systemd services.
You can then re-run the installer from Step 6.

---

## Troubleshooting

### Installer fails with sysctl errors

Ensure the sysctl wrapper is in place (Step 4) and that it's at `/sbin/sysctl`
(not `/usr/local/sbin/sysctl` — LXC may not include that in `$PATH`).

### `su: cannot set groups: Operation not permitted`

Ensure the `su` → `runuser` wrapper is in place (Step 5) at `/bin/su`.

### `crun: mount proc to proc: Operation not permitted`

Your LXC config is missing `features: nesting=1`, or it's being overridden by
`lxc.apparmor.profile: unconfined`. Remove the apparmor override — nesting
provides the correct profile automatically.

### `loginctl enable-linger` timeout

`systemd-logind` isn't running. Add `features: nesting=1` to the LXC config
and reboot the container.

### Port already in use (e.g., 11002)

A previous failed install left processes running. Kill them:

```bash
pkill -9 discovery
```

Then clean up the leftover user/group and retry:

```bash
sed -i '/^uosserver/d' /etc/passwd /etc/shadow /etc/group /etc/gshadow /etc/subuid /etc/subgid
rm -rf /home/uosserver /var/lib/uosserver /usr/local/bin/uosserver /usr/local/bin/uosserver-purge
rm -f /etc/systemd/system/uosserver* /etc/systemd/system/multi-user.target.wants/uosserver*
systemctl daemon-reload
```

---

## References

- [kam821/Unifi-OS-Server-on-LXC-Proxmox](https://github.com/kam821/Unifi-OS-Server-on-LXC-Proxmox) — Original sysctl + TUN workaround
- [Ubiquiti Self-Hosting Guide](https://help.ui.com/hc/en-us/articles/34210126298775-Self-Hosting-UniFi)
- [Crosstalk Solutions — UOS Server Best Practices](https://www.crosstalksolutions.com/complete-unifi-os-server-installation-on-linux-best-practices/)
- [Install UOS 5.0.6 on Debian 13](https://randomcontributions.blogspot.com/2026/03/install-unifi-os-server-506-on-debian.html)
