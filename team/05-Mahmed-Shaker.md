# Mahmed Shaker - siem01, Wazuh, Evidence, and Report Assembly

عليك VM واحدة هي `siem01` بذاكرة **4 GB**. ابدأ بتجهيزها واختبر Wazuh عليها لوحدها قبل اختبار الربط مع الأجهزة على لاب محمود. جهّز Appendix C وجدول Chapter 9 من دلوقتي. حط حالة `NOT RUN` للأدلة اللي لسه مستنيها واسم صاحب كل اختبار، وبعد ما يستلمه راجع الملف واربطه بالفصل.

## Mission

Make the project reviewable. You own siem01, Wazuh and the evidence register, diagrams, test-result structure, and final report assembly, while Mahmoud owns the technical merge and final sign-off.

## siem01 and Wazuh

1. Build the Ubuntu Server 24.04 base VM using A §4–5 and B5 §1. Allocate `2 vCPU`, `4 GB RAM` and a `24 GB` disk. The final lab uses `10.10.60.21` on VLAN 60 / `VMnet16`.
2. Stage the manager, indexer and dashboard on siem01 by itself using B7 §4. Save the version, build steps and local service status. Only label results from this setup `STAGED`. If the host cannot spare 4 GB, coordinate a standalone slot on Mahmoud's laptop.
3. Deliver the clean base VM or rebuild notes to Mahmoud before Gate 3. Give Youssef the access and VM details needed to set up Ansible on siem01. Save the Wazuh steps separately so the integrated Wazuh rollout runs in Gate 5.
4. In Gate 5, lead Wazuh configuration, agent enrollment and event validation using B7 §4 and B5 §7.4/§8. Coordinate Linux agents with Youssef, Windows and pfSense/Suricata with Mahmoud, and cloud-dr with Iprahim. Preserve A's approved event and enrollment paths. Record agent status by batch and demonstrate the expected VIP audit alert (rule 100100).
5. Document the Wazuh configuration backup under B8 §3. Youssef owns the Ansible configuration in the same backup scope. Mahmoud reviews the final configuration and signs off Gate 5 on his laptop.

## Evidence and report work

1. Build the draw.io diagrams from A: Figure 3.1 network, Figure 6.1 cloud, and an unnumbered defence-in-depth diagram for Chapter 7. Keep Figure 7.1 for the firewall rules evidence.
2. Maintain Appendix C evidence register. Every screenshot/log/CSV must have a path, date, owner, scenario/gate, and description.
3. Prepare Chapter 9 tables from C-test-plan.md with empty measured cells before execution.
4. Act as scenario scribe: record T0/T1 timestamps, commands, VM power phase, target status, and exact errors. Do not operate a test without the gate owner.
5. Assemble the final report in the official template: remove guidance boxes, update TOC, table/figure numbering, references, appendices, and task allocation.
6. Run the final cross-file QA against A, B, C, D, E and the Change Register.

## Evidence naming

Use `fig-<chapter>.<number>-<short-description>.png` for report figures and descriptive `.txt`, `.csv`, `.json`, or `.log` names for supporting evidence. Store all evidence under `E:\NileCare\evidence\`. A filename is not evidence until the file exists and its content is reviewed.

## Required outputs

- `HANDOFF-Mahmed-Shaker.md` with separate rows for base VM, Wazuh staging, Gate 5 integration, Wazuh config backup, and report/evidence.
- Redacted siem01 build notes, version and agent list by batch, `fig-7.2-wazuh-dashboard.png`, and evidence for the VIP audit alert when measured.
- `fig-3.1-network-diagram.png`, `fig-6.1-cloud-network-diagram.png`, and `defence-in-depth.drawio` when their inputs are approved.
- Updated Appendix C screenshot register.
- Chapter 9 result tables and scenario evidence map.
- `report-qa.md` listing every mismatch, missing file, placeholder, and final disposition.
- The final assembled report source and a release checklist for Mahmoud.

## Acceptance checks

- Gate 5 acceptance requires the intended agents and expected alert evidence on Mahmoud's lab, with his sign-off.
- No screenshot is labelled PASS without the corresponding command/result being visible.
- Evidence from a laptop is labelled staged and is not used as final gate evidence until reproduced on Mahmoud's laptop.
- Scenario phases and powered-on VMs match C-test-plan.md and remain at or below `10.5 GB`.
- Every missing result remains `<PENDING>` in the report.
- The final report says five team members, not three, and Appendix D matches this plan.

## Where your work goes

Keep the editable diagrams and caption list beside the report. For each evidence row record expected vs observed result, host, date, config revision and reviewer. Record baseline captures before the applicable hardening stage. Use [Appendix D](06-Appendix-D-task-allocation.md) and [the integration playbook](07-INTEGRATION-PLAYBOOK.md) to assemble the template. In [your handoff](08-HANDOFF-TEMPLATE.md), link the report source, figures, technical review items and release checklist. Each service owner supplies their own results and Mahmoud signs the technical release.
