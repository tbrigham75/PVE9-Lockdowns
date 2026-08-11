# proxmox_lockdowns

Production-oriented, idempotent lockdown controls for **Proxmox VE 9.x on Debian 13 (trixie), x86-64**. The role supports standalone nodes by default and detects clustered nodes. It deliberately does **not** claim PVE 8 or other Debian support.

The role is conservative: `proxmox_lockdowns_enable` defaults to `false`; SSH, firewall, repository, certificate, filesystem, KSM, host-backup, Corosync, and Ceph changes require separate opt-ins. It never formats disks, enrolls Secure Boot keys, changes network topology, reboots, restarts Corosync, removes repository files, or infers guest firewall rules. Always preserve console/out-of-band access and test in a lab.

## Authoritative source

Implementation source: HomeSecExplorer, **“Proxmox VE 9.x Hardening Guide,” Version 0.9.2 - February 09, 2026**:

- Rendered: <https://github.com/HomeSecExplorer/Proxmox-Hardening-Guide/blob/main/docs/pve9-hardening-guide.md>
- Raw: <https://raw.githubusercontent.com/HomeSecExplorer/Proxmox-Hardening-Guide/main/docs/pve9-hardening-guide.md>

The complete 1,948-line raw guide was reviewed. Controls 1.1.7 and 1.1.8 are present in the body but absent from its table of contents; both are mapped below. Extra sysctl values and safety assertions are labeled implementation safeguards, not guide requirements.

## Installation

```bash
ansible-galaxy collection install -r proxmox_lockdowns/requirements.yml
mkdir -p roles
cp -a proxmox_lockdowns roles/proxmox_lockdowns
```

Version 2.0.0 is a breaking rename: use role `proxmox_lockdowns` and variables `proxmox_lockdowns_*`; the former identity and namespace are not aliases.

Use ansible-core 2.16 or newer. The control host needs `ansible.posix` and `community.general`; the managed node must already be PVE 9 on Debian 13.

## Minimal safe example

```yaml
---
- name: Baseline a PVE 9 lab node
  hosts: proxmox
  become: true
  gather_facts: true
  serial: 1
  roles:
    - role: proxmox_lockdowns
      vars:
        proxmox_lockdowns_enable: true
        proxmox_lockdowns_apply_ssh: false
        proxmox_lockdowns_apply_firewall: false
        proxmox_lockdowns_apply_certificates: false
        proxmox_lockdowns_apply_ceph: false
        proxmox_lockdowns_manage_repositories: false
```

## Full opt-in example

This is illustrative, not copy/paste policy. Model every cluster/storage/migration/backup/service flow first.

```yaml
---
- name: Stage PVE lockdowns one node at a time
  hosts: proxmox
  become: true
  gather_facts: true
  serial: 1
  roles:
    - role: proxmox_lockdowns
      vars:
        proxmox_lockdowns_enable: true
        proxmox_lockdowns_cluster_mode: auto
        proxmox_lockdowns_reboot_policy: notify

        proxmox_lockdowns_apply_ssh: true
        proxmox_lockdowns_ssh_port: 22
        proxmox_lockdowns_ssh_permit_root_login: prohibit-password
        proxmox_lockdowns_ssh_password_authentication: false
        proxmox_lockdowns_ssh_disable_forwarding: true
        proxmox_lockdowns_root_match_management_networks: [10.10.0.0/24]
        proxmox_lockdowns_cluster_ssh_networks: [10.20.0.0/24, 10.21.0.0/24]
        proxmox_lockdowns_manage_cluster_ssh_client: true
        proxmox_lockdowns_cluster_ssh_host_patterns: ['pve*', '*.cluster.example']

        proxmox_lockdowns_apply_firewall: true
        proxmox_lockdowns_firewall_takeover: true
        proxmox_lockdowns_firewall_enable: true
        proxmox_lockdowns_trusted_management_networks: [10.10.0.0/24]
        proxmox_lockdowns_firewall_default_input_policy: DROP
        proxmox_lockdowns_firewall_default_output_policy: ACCEPT
        proxmox_lockdowns_firewall_default_forward_policy: ACCEPT
        proxmox_lockdowns_firewall_additional_rules:
          - 'IN ACCEPT -source 10.20.0.0/24 -p udp -dport 5405:5412 -log nolog'
          - 'IN ACCEPT -source 10.30.0.0/24 -p tcp -dport 3128 -log nolog'

        proxmox_lockdowns_apply_auditd: true
        proxmox_lockdowns_apply_sysctl: true
        proxmox_lockdowns_apply_fail2ban: true
        proxmox_lockdowns_apply_central_logging: true
        proxmox_lockdowns_rsyslog_target: logs.example.net

        proxmox_lockdowns_apply_proxmox: true
        proxmox_lockdowns_ksm_policy: disable
        proxmox_lockdowns_disable_ksm_for_existing_vms: false

        proxmox_lockdowns_apply_certificates: true
        proxmox_lockdowns_certificate_mode: acme
        proxmox_lockdowns_acme_account: default
        proxmox_lockdowns_acme_email: pve-ops@example.net
        proxmox_lockdowns_acme_domain: pve01.example.net

        proxmox_lockdowns_apply_ceph: true
        proxmox_lockdowns_ceph_enabled: true
        proxmox_lockdowns_ceph_expected_public_network: 10.30.0.0/24
        proxmox_lockdowns_ceph_expected_cluster_network: 10.31.0.0/24
        proxmox_lockdowns_ceph_manage_messenger_encryption: true
        proxmox_lockdowns_ceph_restart_allowed: false
```

## Variable reference

Blank strings mean “leave unchanged” or “required only when opted in.” Secrets must be supplied through Ansible Vault or an external secret provider.

| Variable | Default | Type | Effect | Risk / impact |
|---|---:|---|---|---|
| `proxmox_lockdowns_enable` | `false` | bool | Master mutation switch | Must be deliberately enabled |
| `proxmox_lockdowns_backup` | `true` | bool | Back up changed text configs | May consume small disk space |
| `proxmox_lockdowns_reboot_policy` | `never` | enum | Reporting policy only | Role never reboots |
| `proxmox_lockdowns_apply_packages` | `true` | bool | Permit baseline package tasks | Package changes |
| `proxmox_lockdowns_apply_unattended_upgrades` | `true` | bool | Install/configure security updates | Updates can affect workloads; no auto reboot |
| `proxmox_lockdowns_unattended_origins` | Debian origins | list | Allowed unattended origins | Bad patterns may omit or broaden updates |
| `proxmox_lockdowns_unattended_mail` | `''` | string | Optional update mail recipient | Requires working mail transport |
| `proxmox_lockdowns_install_microcode` | `true` | bool | Install detected Intel/AMD microcode | Reboot required to activate |
| `proxmox_lockdowns_manage_repositories` | `false` | bool | Write dedicated deb822 files | Can affect package source selection |
| `proxmox_lockdowns_repository_subscription` | `none` | enum | Select enterprise/no-subscription/none in managed file | Does not remove legacy duplicates |
| `proxmox_lockdowns_enable_non_free_firmware` | `false` | bool | Add `non-free-firmware` component | Repository policy/legal review |
| `proxmox_lockdowns_debian_mirror` / `_security_mirror` | Debian HTTPS URLs | string | Debian mirror endpoints | Trust/availability dependency |
| `proxmox_lockdowns_apply_kernel_lockdown` | `false` | bool | Add `lockdown=integrity` and update GRUB | Can break unsigned modules; reboot needed |
| `proxmox_lockdowns_confirm_secure_boot` | `false` | bool | Operator confirmation gate | Must match `mokutil` detection |
| `proxmox_lockdowns_kernel_lockdown_mode` | `integrity` | enum | Lockdown mode | Integrity is the only supported role value |
| `proxmox_lockdowns_manage_vz_mount` | `false` | bool | Add fstab entry only | Data migration/mount remains manual |
| `proxmox_lockdowns_vz_mount_source` | `''` | string | Pre-created block device | Wrong device can cause boot/mount failure |
| `proxmox_lockdowns_vz_mount_fstype` | `xfs` | string | fstab filesystem type | Must match existing filesystem |
| `proxmox_lockdowns_vz_mount_options` | `defaults,discard,nodev,nosuid` | string | Mount flags | `noexec` is rejected |
| `proxmox_lockdowns_apply_accounts` | `false` | bool | Manage declared PVE users/ACLs | RBAC mistakes can grant/deny access |
| `proxmox_lockdowns_pve_users` | `[]` | list(dict) | Create personalized users | Does not manage passwords/enrollment |
| `proxmox_lockdowns_pve_acls` | `[]` | list(dict) | Add declared ACLs | Review path/role/propagation carefully |
| `proxmox_lockdowns_manage_realm_tfa` | `false` | bool | Apply realm TFA type | Can lock out unenrolled users |
| `proxmox_lockdowns_tfa_realms` | `[]` | list(dict) | Realm/type declarations | Human enrollment remains manual |
| `proxmox_lockdowns_apply_ssh` | `false` | bool | Install validated SSH drop-in | Remote lockout risk |
| `proxmox_lockdowns_ssh_port` | `22` | int | SSH listen/firewall port | Must match routing and firewall |
| `proxmox_lockdowns_ssh_permit_root_login` | `''` | enum/string | Leave unchanged or set root policy | Cluster workflows depend on root SSH |
| `proxmox_lockdowns_ssh_password_authentication` | `''` | bool/string | Leave unchanged or set password auth | Disable only after key/break-glass test |
| `proxmox_lockdowns_ssh_pubkey_authentication` | `true` | bool | Keep public-key auth enabled | Required before disabling passwords |
| `proxmox_lockdowns_ssh_disable_forwarding` | `false` | bool | Disable forwarding globally | Breaks live migration unless Match exception exists |
| `proxmox_lockdowns_cluster_ssh_networks` | `[]` | list(CIDR) | Root forwarding Match exception | Must cover cluster peers |
| `proxmox_lockdowns_root_match_management_networks` | `[]` | list(CIDR) | Root-login Match exception | Scope tightly; preserve cluster needs |
| `proxmox_lockdowns_ssh_manage_crypto_profile` | `false` | bool | Emit explicit KEX/cipher/MAC lines | Compatibility risk; guide delegates exact profile |
| `proxmox_lockdowns_ssh_kex_algorithms` / `_ciphers` / `_macs` | `''` | string | Explicit OpenSSH lists | Test clients and PVE workflows |
| `proxmox_lockdowns_manage_cluster_ssh_client` | `false` | bool | Prefer RSA-SHA2 for peers | Required patterns must be precise |
| `proxmox_lockdowns_cluster_ssh_host_patterns` | `[]` | list | Peer `Host` patterns | Broad patterns weaken external preference |
| `proxmox_lockdowns_apply_networking` | `false` | bool | Enable network-related tasks | Forwarding/topology impact |
| `proxmox_lockdowns_apply_sdn_sysctl` | `false` | bool | Apply guide 1.3.6 sysctls | Can break routing/SDN workloads |
| `proxmox_lockdowns_load_br_netfilter` | `false` | bool | Load bridge netfilter now/at boot | Performance and packet-path impact |
| `proxmox_lockdowns_apply_firewall` | `false` | bool | Manage PVE cluster firewall | High lockout/cluster-wide risk |
| `proxmox_lockdowns_firewall_takeover` | `false` | bool | Confirm full `cluster.fw` ownership | Existing cluster rules are replaced/backed up |
| `proxmox_lockdowns_firewall_enable` | `false` | bool | Enable PVE firewall in policy | Requires OOB and management allows |
| `proxmox_lockdowns_firewall_default_input_policy` | `ACCEPT` | enum | Host input default | `DROP` is disruptive |
| `proxmox_lockdowns_firewall_default_output_policy` | `ACCEPT` | enum | Host output default | Restriction can break updates/NTP/logging |
| `proxmox_lockdowns_firewall_default_forward_policy` | `ACCEPT` | enum | Forward default | `DROP` requires nftables and complete guest flows |
| `proxmox_lockdowns_firewall_enable_nftables` | `false` | bool | Enable the node nftables backend through the PVE API | Tech-preview backend; stage in a lab |
| `proxmox_lockdowns_trusted_management_networks` | `[]` | list(CIDR) | Pre-allow SSH/GUI | Non-empty assertion when firewall applied |
| `proxmox_lockdowns_pve_gui_port` | `8006` | int | GUI allow port | Must match PVE proxy exposure |
| `proxmox_lockdowns_firewall_additional_rules` | `[]` | list(string) | Complete PVE rule lines | Must model cluster/storage/migration/backups/services |
| `proxmox_lockdowns_apply_sysctl` | `true` | bool | Install baseline sysctl drop-in | Extra implementation safeguards; tune for workloads |
| `proxmox_lockdowns_sysctl_values` | documented map | dict | Host lockdown values | Overrides are supported |
| `proxmox_lockdowns_sdn_sysctl_values` | guide 1.3.6 map | dict | SDN bridge/forward settings | Opt-in only |
| `proxmox_lockdowns_apply_auditd` | `true` | bool | Install auditd and `/etc/pve` watch | Audit volume/performance |
| `proxmox_lockdowns_apply_central_logging` | `false` | bool | Forward host/PVE logs | Collector trust/network dependency |
| `proxmox_lockdowns_rsyslog_target` | `''` | string | Collector host | Required when enabled |
| `proxmox_lockdowns_rsyslog_port` / `_protocol` | `6514` / `tcp` | int/enum | Transport endpoint | Template does not configure TLS credentials |
| `proxmox_lockdowns_apply_rkhunter` | `false` | bool | Install daily rkhunter job | Guide marks unvalidated; false positives |
| `proxmox_lockdowns_apply_fail2ban` | `false` | bool | Protect SSH/PVE GUI | Can ban administrators/NAT gateways |
| `proxmox_lockdowns_fail2ban_maxretry` / `_findtime` / `_bantime` | `3` / `2h` / `1h` | int/string | Jail thresholds | Tune for shared source IPs |
| `proxmox_lockdowns_apply_proxmox` | `false` | bool | PVE-specific KSM/Corosync tasks | Separate sub-switches still apply |
| `proxmox_lockdowns_ksm_policy` | `unchanged` | enum | Optionally stop/disable KSM | Memory consumption may rise |
| `proxmox_lockdowns_disable_ksm_for_existing_vms` | `false` | bool | Set `allow-ksm=0` on QEMU guests | Guest-wide configuration mutation |
| `proxmox_lockdowns_cluster_mode` | `auto` | enum | Auto/deliberate cluster state | Standalone conflict is rejected |
| `proxmox_lockdowns_manage_corosync_secauth` | `false` | bool | Audit gate only | Role never edits/restarts Corosync |
| `proxmox_lockdowns_apply_certificates` | `false` | bool | Manage supplied or ACME certificate | GUI outage/trust risk |
| `proxmox_lockdowns_certificate_mode` | `none` | enum | `none`, `files`, or `acme` | Required inputs asserted |
| `proxmox_lockdowns_certificate_fullchain_src` / `_private_key_src` | `''` | path | Controller-side PEM inputs | Protect private key |
| `proxmox_lockdowns_acme_account` / `_email` / `_directory` / `_domain` | `default` / blank | string | ACME account/node identity | DNS/CA ownership required |
| `proxmox_lockdowns_acme_plugin` | `''` | string | Reference a pre-existing DNS challenge plugin | Plugin secret lifecycle remains a manual PVE workflow |
| `proxmox_lockdowns_apply_ceph` | `false` | bool | Enter Ceph task set | Also requires `ceph_enabled` and detection |
| `proxmox_lockdowns_ceph_enabled` | `false` | bool | Explicit Ceph acknowledgement | Prevents accidental changes |
| `proxmox_lockdowns_ceph_manage_messenger_encryption` | `false` | bool | Configure cephx/Messenger secure modes | Cluster compatibility/performance impact |
| `proxmox_lockdowns_ceph_restart_allowed` | `false` | bool | Permit notified `ceph.target` restart | Never enable without staged maintenance |
| `proxmox_lockdowns_ceph_expected_public_network` / `_cluster_network` | `''` | CIDR string | Explicit assumptions/gate | Required for Messenger changes |
| `proxmox_lockdowns_ceph_pool_policy_audit` | `true` | bool | Assert replica policy | Audit fails but does not mutate live pools |
| `proxmox_lockdowns_ceph_size` / `_min_size` | `3` / `2` | int | Minimum audit thresholds | Failure domain still manual |
| `proxmox_lockdowns_manage_host_backup` | `false` | bool | Install PBS client/service/timer | Bandwidth, secret, restore-design impact |
| `proxmox_lockdowns_pbs_repository` / `_password` / `_fingerprint` | blank | string | PBS credential environment | Use Vault; file is root 0600 |
| `proxmox_lockdowns_pbs_backup_calendar` | `*-*-* 06:00:00` | string | Daily systemd schedule | Load/window impact |
| `proxmox_lockdowns_pbs_encryption_enabled` | `false` | bool | Client-side host-backup encryption | Lost key makes backups unrecoverable |
| `proxmox_lockdowns_pbs_encryption_key_file` | `/root/.pbs-enc-key.json` | path | Key location | Escrow offline |
| `proxmox_lockdowns_pbs_encryption_password` | `''` | secret | Encryption key secret | Use Vault, never Git |
| `proxmox_lockdowns_pbs_manage_encryption_key` | `false` | bool | Create key if absent | Offline escrow required immediately |
| `proxmox_lockdowns_run_verification` | `true` | bool | Run post-state checks | Read-only except handler flush timing |
| `proxmox_lockdowns_emit_manual_review` | `true` | bool | Print non-automated obligations | Informational |

## Tags

`preflight` (always), `packages`, `repositories`, `kernel`, `filesystem`, `accounts`, `ssh`, `networking`, `firewall`, `sysctl`, `audit`, `logging`, `services`, `certificates`, `pve`, `ceph`, `verify`, and `manual`.

## Firewall lockout prevention

The role refuses firewall application unless takeover and enablement are explicit and trusted management CIDRs are non-empty. It writes SSH and GUI allows before operator-supplied rules. Because `/etc/pve/firewall/cluster.fw` is cluster-wide, `firewall_takeover=true` means full ownership; the prior file is backed up when changed. Inventory every Corosync, Ceph, migration, storage, backup, DNS, NTP, logging, monitoring, guest, and application flow. Keep default policies `ACCEPT` until rules are validated. Guest NIC/scope enablement and guest-specific rules are manual because the role cannot infer legitimate workloads.

## Cluster and Ceph cautions

Cluster detection uses `/etc/pve/corosync.conf`. The role blocks unsafe clustered SSH combinations, audits `secauth:on`, and never edits Corosync or restarts it. Redundant ring changes must be performed one node at a time with quorum/OOB checks.

Ceph is untouched unless `apply_ceph` and `ceph_enabled` are both true and `/etc/ceph/ceph.conf` exists. Replica policy is audit-only. OSD encryption is creation-time/manual. Messenger secure mode is opt-in; expected networks are mandatory, and `ceph.target` is not restarted unless separately allowed. Stage secure mode across a lab/compatible cluster and measure CPU/latency before production.

## Reboots and kernel updates

The role never reboots. Microcode and kernel lockdown notify that a reboot is required. Secure Boot enrollment is manual. Stage kernel/microcode updates on a non-critical node, verify signed modules/ZFS/Ceph/PBS, reboot one node at a time, and confirm cluster health before continuing.

## Validation and rollback

1. Capture `/etc`, `/etc/pve`, network configuration, current repository policy, SSH effective config, firewall rules, and PVE/Ceph health.
2. Run `ansible-playbook --check --diff` first. PVE CLI queries still run read-only in check mode.
3. Apply to a non-production standalone node, then run a second time and require zero changes.
4. Test a new SSH connection and PVE GUI session before closing the original session.
5. Roll back from Ansible backup files or configuration snapshots. For cluster files, use supported PVE workflows and one-node-at-a-time procedures.
6. Validate with `sshd -t`, `auditctl -l`, `pve-firewall status`, `pvecm status`, `corosync-cfgtool -s`, `ceph health detail`, and a scheduled restore test as applicable.

## Manual actions and operational controls

`tasks/manual_review.yml` emits the complete operational list. Major boundaries: CIS Debian 13 profiles and ssh-audit algorithm selection are external baselines; firmware/Secure Boot/LUKS/partitioning need pre-boot or installation work; network/SDN/EVPN and Corosync need topology-aware maintenance; human 2FA, break-glass custody, API-token governance, subscription procurement, certificate authority/DNS ownership, guest policy, monitoring response, documentation, exceptions, backups, and recovery exercises need organizational decisions. These are reported, never falsely marked remediated.

## Limitations and assumptions

- Only PVE 9 on Debian 13 trixie x86-64, with root/become and systemd.
- The role does not reproduce the licensed CIS benchmark or external ssh-audit guides. It preserves the guide’s PVE deviations and exposes compatible integration variables.
- Repository management adds a dedicated file and does not delete/disable unknown sources.
- Firewall takeover is all-or-nothing because PVE has no safe include fragment for cluster policy.
- ACME DNS plugin credential lifecycle, guest backup jobs, monitoring backends, and notification targets vary by environment and remain manual or externally managed.
- Check mode predicts file/package/module changes where supported; commands that mutate PVE are skipped or guarded. A real lab run is still required.

## Complete traceability matrix

State is **enabled** only after the master switch; **conditional** means bounded/detected; **disabled** means explicit opt-in; **manual** means report/audit only.

| Guide section/title | Recommendation summary | Role task/file(s) | Default | Variables/preconditions | Validation | Notes/risk |
|---|---|---|---|---|---|---|
| Overview / Safe Remediation Workflow | Inventory, impact review, lab, pilot, monitor, phased rollout | README; `manual_review.yml` | manual | Change process | Evidence/runbook review | Operational gate |
| Overview / Installation Note; Pre-deployment Checklist | Debian-first layout; firmware/BMC updates | `manual_review.yml` | manual | Installation/physical access | Firmware/inventory review | Pre-install/physical |
| Overview / Inventory, Levels, Design Principles | Inventory each node; select level; dedicated hypervisor; separated planes; default deny; UI/API and break-glass root | README; `manual_review.yml` | manual | Architecture/governance | Document review | Organizational |
| 1.1.1 Apply Debian 13 CIS Level 1 | Apply CIS L1; preserve root cluster SSH, fuse/Ceph, and PVE firewall deviation | `preflight.yml`, `ssh.yml`, `firewall.yml`, manual report | manual/conditional | Licensed CIS content; selected firewall | CIS scan plus PVE tests | Full benchmark not reproduced |
| 1.1.2 Apply Debian 13 CIS Level 2 | Apply after L1; preserve cluster forwarding; lab Ceph/ZFS/PBS | SSH Match template; manual report | manual/disabled | SSH opt-in and cluster CIDRs | `sshd -t`, migration test | External benchmark |
| 1.1.3 Configure Automatic Security Updates | Install packages, Debian security origins, no auto reboot, optional mail | `packages.yml`; APT templates | enabled | Master enable | service enabled; logs | Package changes |
| 1.1.4 Apply ssh-audit Hardening Profile | Harden server/client; RSA-SHA2 preference for cluster peers | SSH variables/template; `.ssh/config` block; manual report | disabled | Explicit compatible algorithms/patterns | `sshd -t`; ssh-audit/manual peer test | External guides define exact lists |
| 1.1.5 Enable Full-Disk Encryption | LUKS2/ZFS encryption; escrow keys | `manual_review.yml` | manual | Pre-install design | Boot/recovery drill | Cannot retrofit safely |
| 1.1.6 Dedicated Filesystem for /var/lib/vz | Separate mount, nodev/nosuid, never noexec | `filesystem.yml` | disabled | Existing device/fstype/options | fstab review; maintenance mount test | No format/mount/data move |
| 1.1.7 Enable Debian non-free-firmware repositories | Add component and install required firmware | `apt_repositories.yml`, `debian.sources.j2` | disabled | Repo management + component flag | `apt-cache policy` | Body-only control, missing from guide TOC |
| 1.1.8 Install CPU microcode | Install vendor package; reboot | `packages.yml` | conditional | Detected Intel/AMD | package state; post-reboot kernel log | Body-only control, missing from guide TOC |
| 1.2.1.1 Enable UEFI Secure Boot | Follow PVE setup; stage kernels | manual report; lockdown precheck | manual | Firmware access | `mokutil --sb-state` | Unvalidated by guide |
| 1.2.1.2 Kernel Lockdown (Integrity Mode) | Confirm Secure Boot; add lockdown; update GRUB; reboot/verify | `kernel_and_boot.yml` | disabled | Secure Boot explicit + detected | GRUB; post-reboot `/sys/.../lockdown` | Unsigned module risk |
| 1.2.2 Network Separation | Separate IPMI/management/cluster/storage/guest; no host IP on tenant bridge; MTU | manual report | manual | Network design | Diagram/config/traffic tests | Cannot infer topology |
| 1.2.3 Maintain Valid Proxmox Subscription | Procure/upload; enterprise repo; disable no-subscription | repository template; manual report | manual/disabled | Subscription and repo flag | `apt-cache policy`; UI status | Procurement is not hardening alone |
| 1.2.4 Enable PVE Firewall | Allow trusted SSH/GUI/needed ports before INPUT DROP | `preflight.yml`, `firewall.yml`, `cluster.fw.j2` | disabled | takeover, enable, trusted CIDRs, OOB | `pve-firewall status`; new sessions | Cluster-wide lockout risk |
| 1.2.5 PVE Firewall FORWARD DROP | nftables, node/guest firewall, complete guest rules | cluster template plus manual report | disabled/manual | nftables acknowledgement; modeled flows | lab traffic tests | Tech preview; guest inference unsafe |
| 1.2.6 Review/Configure KSM | Disable/unmerge KSM; set QEMU Allow KSM false | `proxmox.yml` | disabled | KSM policy; optional guest mutation | sysfs/service/qm config | Increased RAM use |
| 1.2.7 Avoid LXC/OCI | Run container platforms inside VMs | manual report | manual | Workload architecture | Inventory review | Migration/workload specific |
| 1.2.8 Unprivileged Containers by Default | Prefer unprivileged CTs; safe mounts/devices | manual report | manual | CT design | `pct config` review | Existing conversion risky |
| 1.3.1 SDN Baseline | Isolated zones/VNets, unique CIDRs, firewall, no host management bridge | manual report | manual | SDN design | PVE/diagram audit | Guide says unvalidated |
| 1.3.2 SDN Firewall Defaults | nftables; VNet INPUT/FORWARD DROP; reusable allows | manual report/firewall guard | manual | Complete VNet flows | lab connectivity | Legacy backend ignores forward |
| 1.3.3 DHCP/RA Guard | Permit trusted DHCP/RA/DNS only | manual report | manual | Server/router identities | maintenance `tcpdump` | VNet-specific |
| 1.3.4 VXLAN Hardening | Dedicated transport; UDP 4789 node-only; MTU/monitoring | manual report | manual | Network ACL design | overlay/MTU monitoring | Upstream controls |
| 1.3.5 EVPN/BGP Hardening | Restrict/auth peers; prefix policy; TTL; logs | manual report | manual | FRR/controller design/secrets | BGP route/session audit | Platform/support variability |
| 1.3.6 Bridge & Kernel Settings | br_netfilter and specified sysctls; validate isolation/performance | `networking.yml`, `sysctl.yml`, SDN template | disabled | all three network opt-ins | module/sysctl/traffic test | Routing/performance impact |
| 2.1.1 Use Personalized Accounts | Unique PVE users | `accounts_and_auth.yml` | disabled | declared users | `pveum user list` | Password/enrollment external |
| 2.1.2 Grant Least Privilege | Minimal RBAC paths/roles; quarterly review | ACL task; manual report | disabled/manual | declared ACLs | `pveum acl list` | Propagation/path risk |
| 2.1.3 Enable 2FA | Realm requirement and human enrollment; root exception | realm task; manual report | disabled/manual | realm/type; enrolled users | login test/UI audit | Lockout risk |
| 2.1.4 Break-glass Access | Offline strong root credential; log/rotate use; stated composition | manual report | manual | Custody process | sealed-record drill | Secret/human process |
| 2.1.5 Privileged Access Model | Tier root/shell/RBAC; sudo/logging; tokens; root key-only | accounts/SSH/logging/audit tasks; manual report | conditional/manual | Organization policy | access/log review | Shell remains exceptional |
| 2.2.1 Use Scoped API Tokens | Service users, path scope, expiry, quarterly review | manual report | manual | Automation owner | token/ACL inventory | Secret returned once |
| 2.2.2 Least Privilege Tokens | Minimal token role | manual report | manual | Role design | quarterly review | Application-specific |
| 2.2.3 Store Tokens Securely | Vault, never VCS | manual report; `no_log` on PBS secrets | manual | Secret manager | repository/CI audit | Governance |
| 2.2.4 Rotate Tokens Regularly | Expire and rotate within 365 days | manual report | manual | Lifecycle owner | age report | External consumers |
| 2.3.1 Install Trusted Certificates | Install full chain/key or use ACME | `certificates.yml` | disabled | file inputs or ACME | browser/TLS chain | pveproxy access risk |
| 2.3.2 Automate Certificate Renewal | Cluster ACME account/plugin; node domain/order | `certificates.yml`; manual plugin governance | disabled | email/domain/DNS or HTTP reachability | per-node cert/renewal test | Cluster account, node cert |
| 2.3.3 Protect GUI with Fail2Ban | SSH/PVE jails and PVE auth filter | `cron_and_services.yml`; fail2ban templates | disabled | explicit opt-in | `fail2ban-client status` | NAT/admin banning risk |
| 3.1 Redundant Corosync Links | Two independent rings; one-at-a-time restart/failure test | manual report | manual | NIC/switch/VLAN/quorum | `corosync-cfgtool -s`; `pvecm status` | Cluster outage risk |
| 3.2 Secure Inter-Node Communication | Ensure `secauth:on` | `proxmox.yml` audit gate | disabled/audit | clustered; audit flag | grep/config review | No automatic mutation/restart |
| 3.3 Ceph Pool Sizing and Failure Domains | size 3/min_size 2; CRUSH failure domains; health/failure test | `ceph.yml` audit; manual report | conditional/manual | Ceph explicit/detected | pool JSON, tree/rules/map, drills | Live mutation withheld |
| 3.4 Ceph OSD Encryption | Encrypt OSDs at creation | manual report | manual | OSD provisioning | OSD/LUKS inventory | Existing OSD migration disruptive |
| 3.5 Ceph Messenger Encryption | cephx, msgr2, all secure modes; restart | `ceph.yml`; handler | disabled | Ceph flags/networks; optional restart | config dump; Ceph health/traffic | Stage compatibility/performance |
| 4.1 Enforce 3-2-1 | Three copies, two media, one off-site/offline | manual report | manual | Backup architecture | inventory/restore evidence | Organizational |
| 4.2.1 Backup Host Configuration | PBS client, 0600 token file, daily systemd backup, quarterly restore | `cron_and_services.yml`; PBS templates | disabled | PBS token/repo/fingerprint | timer/run/restore test | Guide example path/name inconsistency is not hidden |
| 4.2.2 Encrypt Host Configuration Backups | Scrypt key, 0600, encrypt mode, offline escrow | PBS tasks/templates | disabled | encryption flags/secret/key | PBS shows encrypted; restore test | Lost key is unrecoverable |
| 4.3 Automate Backups | Daily VM/CT backups to PBS | manual report | manual | Guest/storage policy | PVE job history | Role handles host config only |
| 4.4 Encrypt Backups | PBS client encryption and offline key | manual report | manual | Storage/key governance | PBS/restore audit | Guest-job specific |
| 4.5 Notify on Failed Backup | On-failure/always notifications | manual report | manual | Notification target | forced failure test | PVE job specific |
| 4.6 Document and Test Restores | Quarterly isolated restore; record RTO/RPO | manual report | manual | Isolated VLAN/runbook | drill evidence | Operational |
| 5.1.1 Centralized Logging | Forward `/var/log/pve*` in addition to CIS logging | `audit_logging.yml`; rsyslog template | disabled | collector host/port/protocol | collector receipt | TLS credentials external |
| 5.1.2 Auditd for `/etc/pve` | Persistent wa watch, load and verify | audit task/template/verification | enabled | master enable | `auditctl -l`, `ausearch` | pmxcfs audit volume |
| 5.2.1 Centralized Metrics | PVE exporter/Prometheus/Grafana or equivalent | manual report | manual | Monitoring platform | dashboard/data review | External system |
| 5.2.2 Alerting | CPU/disk/RAM >80%; Ceph not OK | manual report | manual | Alert platform/on-call | synthetic alert tests | Operational response |
| 5.3.1 System Audits | Quarterly and post-upgrade OpenSCAP/equivalent | manual report | manual | Scanner/profile | dated reports | Benchmark integration external |
| 5.3.2 Rootkit Detection | Install daily rkhunter | cron/services task | disabled | explicit opt-in | cron/log/manual run | Guide marks unvalidated |
| 5.4 Documentation | Update diagrams, inventory, change logs after every significant change | manual report | manual | Ownership/process | document review | Human process |
| Exception Handling | Record deviation, justification, acceptance, signatures | manual report | manual | Governance | approved exception record | Human approval |
| Appendix A CIS Benchmark | References map to Debian 13 CIS v1.0.0 (2025-12-16) | README/manual report | manual | Licensed benchmark | Profile/version evidence | Version pinned by guide |
| Appendix B Ansible Snippets | External sshaudit/autoupdate role examples | Native package/SSH tasks; manual external-profile note | conditional | Optional external comparison | role/test review | No hidden dependency |
| Appendix C Recovery Drill Checklist | Annual cold, quarterly file, Ceph OSD, Corosync ring tests; document results | manual report | manual | Lab/backups/OOB | drill records | Destructive simulation |
| Appendix D.1 Host Checklist | Firmware, segmentation, Debian partitioning, guide, docs | manual report | manual | Install/change process | signed checklist | Mixed physical/manual |
| Appendix D.2 Guest Checklist | Machine model, VirtIO/agent, baseline, minimal services, guest firewall | manual report | manual | Per-guest owner | guest audit | Outside host role |
