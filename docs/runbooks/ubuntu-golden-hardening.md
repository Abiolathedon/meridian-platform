# Runbook — Ubuntu Golden Image Hardening

**Purpose:** build a hardened Ubuntu Server 24.04 LTS "golden" image for EXODUS, to clone Ubuntu hosts (e.g. the observability node) from. Mirrors the RHEL golden's hardening using Ubuntu's equivalent tools.
**Ticket:** MER-014 · **First built:** 2026-11-13 (`exodus-ubuntu-golden`, DHCP 192.168.0.82).
**Related:** Word notes 35–36; the RHEL golden parity baseline.

> This is a MANUAL procedure: the golden is hardened by hand, then cloned — the same model as the RHEL golden. Baseline *packages + time* are applied separately by the cross-distro `baseline` Ansible role (Molecule-tested on Ubuntu 24.04). The hardening is baked into the image, not (yet) role-applied — moving it to role code is a future "hardening-as-code" epic.

## RHEL ↔ Ubuntu tool map
| Concern | RHEL golden | Ubuntu golden |
|---|---|---|
| Package manager | dnf / yum | apt |
| Firewall | firewalld | UFW |
| Mandatory Access Control | SELinux (Enforcing) | AppArmor (enforce) |
| Time sync | chrony (`chronyd`) | chrony (`chrony`) |
| Auto security updates | dnf-automatic | unattended-upgrades |
| SSH service name | `sshd` | `ssh` |

## 1. Create VM + install Ubuntu Server 24.04 LTS
- VMware `exodus-ubuntu-golden`: 4 vCPU (2×2), 4 GB RAM, 40 GB thin disk, **Bridged** network, Guest OS = Ubuntu 64-bit.
- ISO: `ubuntu-24.04.x-live-server-amd64.iso` (Server, not Desktop).
- Install: **minimal**, **install OpenSSH server**, user `exodus-admin`, DHCP (static IP is stamped per-clone later).

## 2. Strip unneeded services
Remove multipath (useless on a single-disk VM). Disable + mask — do NOT `apt purge` (purging drags out the `ubuntu-server-minimal` meta-package):
    sudo systemctl disable --now multipathd.service
    sudo systemctl disable --now multipathd.socket
    sudo systemctl mask multipathd.service multipathd.socket

## 3. Patch
    sudo apt update
    sudo apt upgrade -y
    sudo reboot        # if a kernel/binaries were updated

## 4. SSH key access (BEFORE disabling passwords)
From the control node, install the bootstrap key and prove it works:
    ssh-copy-id -i ~/.ssh/id_ed25519_exodus.pub -o PubkeyAuthentication=no exodus-admin@<ip>
    ssh -i ~/.ssh/id_ed25519_exodus -o IdentitiesOnly=yes exodus-admin@<ip> whoami   # must return exodus-admin

## 5. SSH hardening
`/etc/ssh/sshd_config.d/00-exodus-ssh-hardening.conf` — named `00-` so it is read BEFORE cloud-init's `50-cloud-init.conf` (sshd uses the FIRST value, so ours wins):
    PermitRootLogin no
    PubkeyAuthentication yes
    PasswordAuthentication no
    KbdInteractiveAuthentication no
Apply + prove (keep your session open): `sudo sshd -t` then `sudo systemctl reload ssh`; from a new session confirm key works and password is refused.

## 6. sysctl hardening
`/etc/sysctl.d/99-exodus-hardening.conf` — byte-identical to the RHEL golden (Ubuntu does NOT disable redirects by default):
    net.ipv4.conf.all.accept_redirects = 0
    net.ipv4.conf.all.send_redirects = 0
Apply: `sudo sysctl --system`.

## 7. journald (persistent, capped logging)
`sudo mkdir -p /etc/systemd/journald.conf.d/`, then `/etc/systemd/journald.conf.d/20-exodus-logging.conf` (identical to RHEL golden):
    [Journal]
    Storage=persistent
    Compress=yes
    SystemMaxUse=200M
    SystemMaxFileSize=50M
    RateLimitIntervalSec=30s
    RateLimitBurst=5000
    ForwardToSyslog=no
Apply: `sudo systemctl restart systemd-journald`; confirm `/var/log/journal` exists.

## 8. Firewall (UFW — ssh only)
    sudo apt install -y ufw
    sudo ufw allow 22/tcp      # ALLOW SSH FIRST (no lockout)
    sudo ufw enable            # answer y
    sudo ufw status verbose    # active, default deny incoming, 22/tcp ALLOW

## 9. AppArmor (verify enforcing — on by default)
    sudo aa-status                  # "apparmor module is loaded"; real profiles in enforce mode
    systemctl is-enabled apparmor   # enabled

## 10. Time sync (chrony — fleet parity)
Install chrony; it auto-retires systemd-timesyncd:
    sudo apt install -y chrony
    chronyc sources                        # a "^*" source = synced
    systemctl is-active systemd-timesyncd  # inactive

## 11. Snapshot as the golden
VMware snapshot `ubuntu-golden-hardened` — the clone-from base.

## Verification checklist
- [ ] multipathd masked
- [ ] fully patched + rebooted
- [ ] SSH key-only, no root, no password (proven from a 2nd session)
- [ ] sysctl redirects = 0
- [ ] journald persistent (`/var/log/journal` exists)
- [ ] UFW active, ssh only
- [ ] AppArmor loaded + enabled at boot
- [ ] chrony synced, timesyncd inactive
- [ ] `baseline` role runs green + idempotent (Molecule passed on Ubuntu 24.04)
