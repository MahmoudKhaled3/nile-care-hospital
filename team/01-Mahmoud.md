# Mahmoud - Lead Integration, VMware, Windows, and Final Lab

هتشتغل مباشرة على لابك. اللاب i7 جيل تاسع و16 GB RAM حسب مواصفاتك. سجّل المساحة الفاضية وحالة virtualization ومسار المشروع عليه، وبعدها امسك الـgates بالترتيب الموجود في [دليل الدمج](07-INTEGRATION-PLAYBOOK.md).

## Mission

Own the final working environment on your laptop and connect every member's output into one reproducible project. You are the technical sign-off owner, not the only person who writes every file.

## Primary work

1. Gate 1: verify the remaining VMnets read-only, correct the fw01 VM to the approved target, install pfSense through B0 §3 or the documented fallback, assign interfaces in NIC order, and record exact pfSense version.
2. Gate 2: build dc01 on `10.10.70.11`, DNS, DHCP scopes/relay, OUs, users, groups, GPOs, and the early/domain AppLocker evidence.
3. Windows backup work: dc02 integration, System State path, FSRM coordination, and the Windows side of B8.
4. Gate 4: configure the pfSense side of IPsec and the approved firewall direction `cloud-dr -> emr01 TCP 3306`.
5. Final Gate 6: coordinate restore and rehearse the defense on the same laptop.
6. Merge all reviewed handoffs, resolve conflicts against A, and sign the final technical checklist.

Your four VMs are `fw01`, `dc01`, `dc02` and `win-client`. Shaker owns `siem01` and Wazuh. In Gate 5, connect his setup on your laptop, coordinate Windows agent checks and pfSense/Suricata forwarding, and sign off with the actual evidence. For the defense, switch between the groups listed in the [demo schedule](07-INTEGRATION-PLAYBOOK.md#5-demo-power-schedule).

## Exact source sections

- `B0-prep-pfsense.md` §§1-8.
- `B4-windows.md` §§1-9.
- `B6-cloud.md` pfSense/IPsec side.
- `B8-backup-dr.md` §§2-5.
- `C-test-plan.md` scenarios 1, 2, 7, and the final orchestration of 3, 6, 8, and 9.

## Required outputs

- `HANDOFF-Mahmoud.md`.
- fw01 `.vmx` and settings evidence, with the final approved folder and RAM.
- `fw01-interface-assignments.png`, `fw-default-deny-log.png`, and gate log entries.
- dc01/dc02 command outputs, GPO/AppLocker evidence, and Windows backup evidence.
- pfSense version, interface map, firewall export, and IPsec status evidence.
- Final `INTEGRATION-STATUS.md` listing every member's status and unresolved issue.

## Acceptance checks

- fw01 has `1 GB`, `16 GB` disk, BIOS, and eight NICs: VMnet8, VMnet10, VMnet12, VMnet13, VMnet14, VMnet15, VMnet16, VMnet17.
- Gate 1 is not marked PASS until its B0 §8 checks are actually evidenced.
- dc01 `dcdiag`, DNS, DHCP relay, and GPO output are captured.
- No Windows result is called passed without a screenshot or command output.
- Every technical merge is checked against A before Shaker updates the report.

## Stop conditions

Stop and report if a member proposes a different IP/port/VM size, if a VMnet cannot be reproduced, if a secret appears in a handoff, or if a test result has no evidence.

## Handoff format

Use [the handoff template](08-HANDOFF-TEMPLATE.md). Add a row for Gate 1, dc01/Windows validation, dc02 and shares, pfSense IPsec, and Windows/pfSense backup. Include the final laptop capacity, VM settings, evidence and the outstanding checks for each gate. Shaker reviews the evidence paths. You own the final lab and its release.

## Where your work goes

Ch. 3 receives the approved network and firewall implementation. Ch. 4 receives the Windows build and its validation. Ch. 6 receives your VPN-side values, and Ch. 8 receives your backup and recovery evidence. Give Shaker the approved facts plus filenames. Use your integration status file to track which revision of each of the other four handoffs is on your laptop and which checks were repeated.
