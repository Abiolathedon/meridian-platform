# Runbook — Ubuntu AD Onboarding (manual)

**Purpose:** onboard a fresh clone of the Ubuntu golden into the fleet by hand — re-stamp identity, set a static IP, disable cloud-init, and join Active Directory with the per-environment access model. Used until the foundation roles are cross-distro (see ADR-0014).
**First run:** 2026-11-21 — `exodus-observability-02` (192.168.0.165), MER-015.
**Related:** ADR-0014 (manual-first decision) · `ubuntu-golden-hardening.md` (the golden this clones from).

> Mirrors what `clone_init` + the foundation roles do on RHEL, using Ubuntu-native tools. All commands run on the clone as `exodus-admin` (bootstrap key) unless noted. Keep a VMware console open as a lifeline for the network + reboot steps.

## RHEL <-> Ubuntu tool map
| Concern | RHEL (roles) | Ubuntu (this runbook) |
|---|---|---|
| Static IP | NetworkManager / nmcli | netplan |
| DNS | NetworkManager | netplan / systemd-resolved |
| First-boot config | kickstart | cloud-init (disabled here) |
| AD client pkgs | sssd (bundled), samba-common-tools | sssd, libnss-sss, libpam-sss, samba-common-bin |
| Home dirs | authselect + oddjobd | pam-auth-update --enable mkhomedir |
| Admin group | wheel | sudo |
| SSH service | sshd | ssh |

## Prerequisites
- Clone created from the `ubuntu-golden-hardened` snapshot, resized to spec, powered on.
- The golden template is POWERED OFF (avoid identity clash).
- Bootstrap key `id_ed25519_exodus` available; AD DC (.157) reachable.
- Target identity known: hostname + static IP (per node).

## 1. Re-stamp identity
    sudo hostnamectl set-hostname <hostname>
    # fix /etc/hosts: change the 127.0.1.1 line to <hostname>
    sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup       # dbus symlink follows
    sudo rm -f /etc/ssh/ssh_host_* && sudo ssh-keygen -A              # fresh host keys

## 2. Static IP (netplan)
Create `/etc/netplan/01-exodus-static.yaml` (chmod 600); interface name from `50-cloud-init.yaml`:
    network:
      version: 2
      ethernets:
        ens33:
          dhcp4: false
          addresses: [<ip>/24]
          routes:
            - to: default
              via: 192.168.0.1
          nameservers:
            addresses: [192.168.0.157]
            search: [exodus.local]
Then: `sudo rm /etc/netplan/50-cloud-init.yaml` and `sudo netplan generate`.

## 3. Disable cloud-init
    sudo touch /etc/cloud/cloud-init.disabled
(stops cloud-init reverting hostname/network on boot — it re-runs when the machine-id changes)

## 4. Apply + prove
    sudo reboot
Reconnect on the new IP; confirm `hostname -I`, `hostnamectl`, and `resolvectl query exodus.local` -> .157.

## 5. Install the AD client stack
    sudo apt update
    sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss \
      adcli samba-common-bin oddjob oddjob-mkhomedir packagekit

## 6. Join AD
    realm discover exodus.local                          # pre-flight: configured: no
    sudo realm join --user=Administrator exodus.local    # prompts for AD password
    realm list                                           # configured: kerberos-member
    id abiola.osota@exodus.local                         # sssd resolves AD users

## 7. Restrict access to the environment group
    sudo realm permit -g 'sg-<env>-admins@exodus.local'  # e.g. sg-mgmt-admins
    sudo pam-auth-update --enable mkhomedir              # home dirs on first login

## 8. SSH drop-in  (/etc/ssh/sshd_config.d/10-exodus-ad-access.conf)
    AllowGroups sudo sg-<env>-admins@exodus.local
    Match Group sg-<env>-admins@exodus.local
        PasswordAuthentication yes
Then: `sudo sshd -t` and `sudo systemctl reload ssh`.

## 9. Group sudo  (/etc/sudoers.d/80-ad-access-sudo — edit with visudo, NO .conf extension)
    %sg-<env>-admins@exodus.local ALL=(ALL) ALL

## Verification checklist
- [ ] hostname + static IP persist across a reboot
- [ ] cloud-init disabled (no reversion)
- [ ] `realm list` -> kerberos-member; `id <aduser>` resolves
- [ ] exodus-admin key login still works (no lockout)
- [ ] AD user logs in by password; home dir auto-created
- [ ] AD user `sudo` works (password-required) -> root
