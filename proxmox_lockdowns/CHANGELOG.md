# Changelog

## 2.0.0 - 2026-08-11

- Breaking rename: role identity and directory changed to `proxmox_lockdowns`.
- Breaking rename: all public variables now use the `proxmox_lockdowns_*` namespace.
- Renamed role-managed files and examples consistently; preserved authoritative upstream guide names and URLs.


## 1.0.0 - 2026-08-11

- Initial production-oriented role based on *Proxmox VE 9.x Hardening Guide* 0.9.2.
- Added strict Debian 13/PVE 9 checks, cluster/Ceph discovery, safe defaults, check-mode-aware tasks, and verification.
- Added opt-in firewall, SSH, repositories, kernel lockdown, filesystem, certificates, KSM, PBS backup, and Ceph controls.
- Added a complete source-guide traceability matrix and manual-control report.
