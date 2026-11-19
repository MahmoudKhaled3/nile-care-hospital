# Youssef Al-Sayed - Linux, Ansible, and Linux Backup

لابك 8 GB. ابدأ بـ`emr01`، وبعد إيقافها جرّب `pacs01` ثم `portal01` حسب المساحة المتاحة. جهّز `lin-client` كـUbuntu Server minimal من B5. محمود يعيد الخطوات المعتمدة على لابه.

## Mission

Build and document the Linux service and automation side so it can be reproduced on Mahmoud's laptop. Use your laptop for independent staging. Mahmoud runs the integrated build locally with your guidance.

## Laptop rule

The laptop has `8 GB RAM` according to the team. CPU, free disk, and virtualization support are `<PENDING>`. Do not run the full estate. Build one VM or one service phase at a time. Keep all staged results labelled `STAGED` until reproduced on Mahmoud's laptop.

## Primary work

1. Stage Ubuntu Server `24.04.3` for your four VMs: `emr01`, `pacs01`, `portal01` and `lin-client` using A's exact IPs and roles.
2. Implement B5: OpenEMR/nginx/PHP-FPM/MariaDB, Orthanc/DICOMweb, portal01, UFW/AppArmor/Fail2ban, SSSD integration after dc01 exists, Linux-side rsyslog forwarding in coordination with Shaker, and Ansible.
3. In Gate 3, configure and run Ansible on the base siem01 VM handed over by Shaker. Coordinate Linux-side agent installation with Shaker in Gate 5 and keep the Wazuh agent playbook aligned with his settings. Shaker owns manager and alert validation.
4. Implement B8 §1/§3 Linux backup pieces: MariaDB dump, GPG/restic flow, Ansible configuration backup, and the Linux service configuration backups.
5. Own the replication source and EMR validation. Iprahim owns the cloud replica and Mahmoud owns the firewall path. The replica initiates TCP 3306 to emr01 over IPsec.

## Ansible placement

The project files place the Ansible control node on `siem01`. WSL may be used on a laptop for shellcheck and syntax checks, but WSL is not the project control-node location. Do not rewrite this as an admin-host control node without an approved design change.

## Required outputs

- `HANDOFF-Youssef-Al-Sayed.md`.
- B5 playbooks, inventory templates, host variables, and syntax-check output.
- `linux-build-notes.md` with exact commands, errors, fixes, package versions, and staged/reproduced status.
- Linux-side agent deployment notes and Ansible changes handed to Shaker for Gate 5.
- MariaDB replication and backup commands with secrets left as `<SET-IN-VAULT>`.
- A list of differences between the laptop staging environment and Mahmoud's laptop.

## Dependencies

- Do not perform the AD join until Mahmoud reports dc01 healthy at Gate 2.
- Do not claim Gate 3 until `ansible all -m ping`, syntax checks, and the real playbook run are evidenced.
- Do not claim Gate 4 until the IPsec path and `SHOW REPLICA STATUS` are evidenced.
- Hand over your Linux hosts and approved agent settings to Shaker before his Gate 5 validation.

## Acceptance checks

- Every service listens on the IP and ports in A.
- Ubuntu uses UFW, AppArmor, and Fail2ban; no discarded Linux stack is introduced.
- Wazuh manager is `10.10.60.21`. Events use TCP `1514`, enrollment uses TCP `1515`, and Linux syslog uses TCP `514` for the sources listed in A.
- No real password, PSK, private key, or storage key appears in files or screenshots.
- Each error is recorded with its fix; a clean-looking handoff without command output is incomplete.

## Where your work goes

Send Mahmoud the editable inventory and playbooks, the source and version of each dependency, and the order needed to rebuild. Use [the handoff template](08-HANDOFF-TEMPLATE.md) for one row per service and per test. Give Shaker the service evidence for Chapter 5, the Linux hardening evidence for Chapter 7, and backup/replica notes for Chapter 8. Get the approved synthetic data manifest from Salh before the import and hand the emr01 source details to Iprahim before the replica setup.
