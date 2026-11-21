# ADR-0014: Ubuntu foundation onboarding — manual-first, cross-distro roles deferred

- **Status:** Accepted — 2026-11-21
- **Ticket:** MER-015 (#35)
- **Relates to:** ADR-0011 (clone lifecycle), ADR-0007 (monitoring), ADR-0009 (AD-integrated Linux access)

## Context

The observability rebuild (Prometheus + Grafana + Loki + Alloy) runs on **Ubuntu Server 24.04**, adding a second OS family to the fleet. MER-014 built and hardened the Ubuntu golden and proved the `baseline` role is cross-distro (packages + time only).

Onboarding the first Ubuntu host (`exodus-observability-02`, .165) revealed that the remaining **foundation roles are Red Hat-specific** underneath:

- `clone_init` — sets the static IP with `community.general.nmcli` (NetworkManager); Ubuntu has no NetworkManager — it uses **netplan**.
- `dns_resolver` — sets DNS via NetworkManager; Ubuntu uses **netplan / systemd-resolved**.
- `ad_join` — installs **RHEL package names**; the Ubuntu AD client set differs (`libnss-sss`, `libpam-sss`, `samba-common-bin`, …).
- `parity_check` — asserts **SELinux + firewalld**; Ubuntu uses **AppArmor + UFW**, so the audit would report false drift.

Making all four cross-distro (Debian branches + Molecule on both distros) is a real multi-role project. Blocking the observability build on it would delay the project goal.

## Decision

**Onboard the first Ubuntu host by hand, the Ubuntu-native way**, and **defer cross-distro role support to a dedicated epic.**

The manual onboarding performs the same lifecycle as `clone_init` + the foundation roles, using Ubuntu tools:

1. Identity re-stamp — `hostnamectl`, `systemd-machine-id-setup`, `ssh-keygen -A`.
2. Static IP — a **netplan** file (`01-exodus-static.yaml`); cloud-init's DHCP file removed.
3. **cloud-init disabled** (`/etc/cloud/cloud-init.disabled`) so it stops reverting hand-managed config.
4. DNS → AD verified via `resolvectl`.
5. AD join — `realmd`/`sssd`/`adcli` with the Ubuntu package set.
6. Access — `realm permit -g sg-mgmt-admins`, `pam-auth-update --enable mkhomedir`, an sshd drop-in (`AllowGroups sudo <group>` + a `Match Group` password rule), and `sudoers.d/80-ad-access-sudo`.

The full procedure is recorded in `docs/runbooks/ubuntu-ad-onboarding.md`.

**Deferred to the cross-distro epic:**
- Debian branches for `clone_init` (netplan), `dns_resolver` (systemd-resolved), `ad_join` (Ubuntu packages), `parity_check` (AppArmor/UFW), Molecule-tested on both distros.
- The `svc-ansible-mgmt` automation account and registering `exodus-observability-02` in the management inventory (both belong with Ansible management of the node).

## Consequences

**Positive**
- The observability build is unblocked; the node is live, AD-joined, and access-controlled (tested: key login, AD password login, AD sudo → root).
- The cross-distro differences are learned and documented (runbook + the build-concept record).
- The automation debt is named explicitly rather than half-built.

**Negative / trade-offs**
- This node is **not yet managed by Ansible** — it is hand-built and documented.
- Future Ubuntu hosts cannot be onboarded with a single command until the epic lands.
- `parity_check` does not cover Ubuntu yet, so there is no automated golden-parity audit for this node.

## References

- MER-015 (#35) · runbook `docs/runbooks/ubuntu-ad-onboarding.md`
- Builds on MER-014 (Ubuntu golden; `baseline` proven cross-distro)
- Extends ADR-0011 (clone lifecycle) to Ubuntu via a manual path pending cross-distro roles
