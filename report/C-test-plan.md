# Deliverable C - Test Plan (sanitised publication copy)

**Secure Hybrid Enterprise Infrastructure - Nile Care Hospital** | Version 1.0 - 2026-11-19

This publication copy records the test objectives, VM phases, acceptance criteria, and evidence register. Detailed command sequences for security exercises remain in the controlled local working copy and are not published here.

## 0. Standing rules

1. Run a scenario only after the required B-README gates have passed.
2. Every screenshot is saved under `E:\NileCare\evidence\<name>.png`; command output is saved beside it. Use ISO-8601 timestamps and the append-only `dr-timeline.log` for disaster-recovery exercises.
3. Record Actual and Pass/Fail exactly as observed. Never pre-fill RTO, RPO, scan counts, or scores.
4. Keep powered-on VMs at or below the 10.5 GB concurrent cap. Anything not listed for a phase stays off.
5. Use synthetic Synthea records and public anonymised DICOM samples only.

**Current Gate 1 network acceptance:** `PARTIAL: VMnet10 and VMnet16 verified by screenshot; VMnet12/13/14/15/17 UNVERIFIED`. Before Gate 1 sign-off, capture screenshots of the remaining networks or inspect VMware's configuration read-only. Do not change network settings during verification.

## 0.1 Test matrix

| # | Scenario | Powered-on phase | Acceptance criterion | Evidence |
|---|---|---|---|---|
| 1 | Domain login and RBAC | fw01, dc01, emr01, win-client | Authorised Windows and Linux logins succeed; department access is enforced | fig-9.1 |
| 2 | GPO application | fw01, dc01, win-client | Required GPOs are present and the lock-screen and removable-media controls behave as designed | fig-9.2 |
| 3 | Test-share file-screening and restore | 3A/3B/3C phases below | FSRM blocks the approved test extensions; SIEM records the event; restore is hash-identical | fig-9.3 |
| 4 | Guest segmentation | fw01, lin-client on VMnet15 | Internal connections are filtered and logged; the permitted external control works | fig-9.4 |
| 5 | SSH protection | fw01, emr01, siem01, lin-client on VMnet16 | Fail2ban records a ban at the configured threshold and Wazuh receives the event | fig-9.5 |
| 6 | VIP record access | 6A/6B phases below | VIP access creates the expected audit/SIEM event; negative and RBAC controls behave correctly | fig-9.6 |
| 7 | EMR cloud failover | fw01, dc01, emr01, win-client plus cloud-dr/Blob | Service returns through cloud-dr within the target RTO; login and record lookup work | fig-9.7 |
| 8 | Database restore | fw01, dc02, emr01, siem01 plus Blob | Immutable dump restores the database and row counts match | fig-9.8 |
| 9 | Vulnerability comparison | matched host batches | High/Critical findings decrease after hardening, with the same scan policy | fig-9.9 |

## 3. Scenario 3 - Test-share screening and restore

**Phases:** 3A uses fw01, dc01, dc02, and win-client (9.5 GB); 3B replaces win-client with siem01 while fw01, dc01, and dc02 remain on (9.5 GB); 3C uses fw01, dc01, and dc02 (5.5 GB). Record FSRM event IDs, SIEM receipt, snapshot ID, restore start/end, file count, and hashes. The pass condition is 100% blocking of the approved test cases, at least one matching SIEM event, and a byte-identical restore. Detailed simulation commands stay local.

## 4. Scenario 4 - Guest segmentation

**VMs:** fw01 and lin-client on VMnet15 (10.10.50.60). Verify the approved internal targets are filtered, pfSense logs the denies, and the IDS/SIEM evidence is correlated. The primary pass condition is zero successful internal connections and logged denies. If no IDS signature fires, record a coverage note; do not invent an alert.

## 5. Scenario 5 - SSH protection

**VMs:** fw01, emr01, siem01, and lin-client on VMnet16 (10.10.60.150). Use only the approved non-destructive lab validation with disposable test identities; never use real credentials or contact out-of-scope systems. Verify the ban, the Wazuh event, and the clean unban after evidence capture. Pass requires the configured ban and matching SIEM alert.

## 6. Scenario 6 - VIP record access

**6A:** fw01, dc01, emr01, win-client. **6B:** after win-client is off, fw01, dc01, emr01, and siem01. Verify the VIP audit event, the rule-100100 alert, the non-VIP negative control, and the billing-role denial. Label host-origin evidence explicitly.

## 7-9. DR and hardening scenarios

Scenario 7 measures T0/T1 for cloud failover and records the clinician validation. Scenario 8 measures dump restore and row-count integrity. Scenario 9 uses the same Nessus policy before and after hardening, with credentialed checks where supported. All measured values remain `<PENDING>` until evidence exists.

## Evidence register

| File | Content |
|---|---|
| fig-9.1 through fig-9.9 | Composite evidence for each scenario |
| `dr-timeline.log` | Append-only timestamps for scenarios 3, 7, and 8 |
| `scan-before.csv` / `scan-after.csv` | Nessus exports using the same policy |
| `ansible-<name>.log` | Playbook output, when the relevant gate is complete |

## Honest-measurement reminder

No result is a pass until its evidence file exists and the actual value is recorded. The Gate-1 network status remains partial until VMnet12/13/14/15/17 are verified.
