# Evidence index and review boundary

This repository records what evidence is required and what was observed. Raw screenshots, logs, VM exports, credentials, and patient data remain in the controlled local evidence location and are never committed here.

## Evidence rule

Every work PR includes:

1. The evidence filename and the gate/scenario it supports.
2. A short sanitised observation that contains no secret, token, private key, real patient data, or unnecessary host detail.
3. The measured value, or `<PENDING>` until the test actually runs.
4. The controlled location or handoff reference needed by the reviewer. A path such as `E:\NileCare\evidence\...` is host-local and is not presented as a publicly downloadable link.

## Current index

| Area | Required evidence | Current observation | State |
|---|---|---|---|
| Gate 1 / VMnet10 | Network Editor screenshot | Verified on the earlier reference host; final-laptop reproduction remains `<PENDING>` | `STAGED` |
| Gate 1 / VMnet16 | Network Editor screenshot | Verified on the earlier reference host; final-laptop reproduction remains `<PENDING>` | `STAGED` |
| Gate 1 / VMnet12, 13, 14, 15, 17 | Screenshot or read-only VMware configuration check | `<PENDING>` | `NOT RUN` |
| fw01 | VM settings, NIC order, installer/version result | `<PENDING>`; previous reference VM had 512 MB and installer failure | `NOT RUN` |
| Windows / AD | dcdiag, DNS/DHCP, GPO outputs | `<PENDING>` | `NOT RUN` |
| Linux / Ansible | service versions, syntax/run output, handoff report | `<PENDING>` | `NOT RUN` |
| Azure / IPsec | region/SKU/quota, cost, tunnel, immutability, deallocation | `<PENDING>` | `NOT RUN` |
| Wazuh / Suricata | agent/alert dashboard and sanitised event summary | `<PENDING>` | `NOT RUN` |
| Backup / restore | restore timestamps, integrity result, snapshot/object identifiers | `<PENDING>` | `NOT RUN` |

An evidence row becomes `MEASURED` only when the owner supplies the actual observation and the independent reviewer can inspect the controlled evidence through the agreed handoff. A staged screenshot or unverified claim never becomes a gate pass by itself.
