# Azure Pre-Flight Check — Iprahim Said (`cloud-dr`)

Date: 2026-11-27
Runbook reference: `report/B-runbooks/B6-cloud.md` (§1)
Member brief: `team/03-Iprahim-Said.md`
Subscription status: Active — **no Azure resources created yet** (`$0.00` spent; stopped per Stop Condition).

---

## 1. Subscription and Resource Providers

| Item | Observed Value |
|---|---|
| Offer / Subscription Name | `Azure for Students` |
| Subscription State | `Enabled` |
| Subscription ID | `<REDACTED-SUBSCRIPTION-ID>` |
| Tenant ID | `<REDACTED-TENANT-ID>` |
| Available Credits | `$100.00` (365 days, expires `2027-11-27`) |
| `Microsoft.Compute` | `Registered` |
| `Microsoft.Network` | `Registered` |
| `Microsoft.Storage` | `Registered` |

---

## 2. Primary Target Region (`uaenorth`) — Blocker / Stop Condition

### 2.1 Compute & Network Quotas (`uaenorth`)

| Quota | Current | Limit |
|---|---:|---:|
| Total Regional vCPUs | `0` | `6` |
| Standard BS Family vCPUs | `0` | `4` |
| Standard Basv2 Family vCPUs | `0` | `10` |
| Standard Bsv2 Family vCPUs | `0` | `10` |
| Virtual Networks | `0` | `1000` |
| Network Security Groups | `0` | `5000` |
| Public IP Addresses (Total) | `0` | `3` |
| Public IPv4 Addresses - Standard | `0` | `3` |
| Public IPv4 Addresses - Basic | `0` | `0` |

### 2.2 SKU Availability Blocker (`uaenorth`)

Querying the Compute Resource SKUs API for `location eq 'uaenorth'` showed that **all `Standard_B*` VM SKUs** — including `Standard_B1s`, `Standard_B2s`, and `Standard_B2ats_v2` — carry the restriction:

```text
Restrictions: NotAvailableForSubscription
```

Only large memory/compute/GPU families (`Standard_FX*`, `Standard_M*`, `Standard_HB*`, `Standard_EC*`, `Standard_NC*`, `Standard_NV*`) are unrestricted in `uaenorth`, which exceed both the `Total Regional vCPUs` (`6`) / family quotas and the student credit budget.

Per `team/03-Iprahim-Said.md` (**Stop conditions**), resource creation was stopped before creating any VM or network resources.

---

## 3. Alternate Region & SKU Scan

Candidate regions checked against the live subscription via Compute Resource SKUs API:

| Region | Unrestricted `Standard_B*` SKUs | Notes |
|---|---|---|
| `uaenorth` | None | All `Standard_B*` = `NotAvailableForSubscription` |
| `qatarcentral` | None | All `Standard_B*` restricted |
| `italynorth` | None | All `Standard_B*` restricted |
| `francecentral` | None | All `Standard_B*` restricted |
| `germanywestcentral` | None | All `Standard_B*` restricted |
| `westeurope` | None | All `Standard_B*` restricted |
| `northeurope` | None | All `Standard_B*` restricted |
| `uksouth` | None | 0 unrestricted VM SKUs |
| `polandcentral` | None | All `Standard_B*` restricted |
| `switzerlandnorth` | None | All `Standard_B*` restricted |
| `norwayeast` | None | All `Standard_B*` restricted |
| `spaincentral` | None | All `Standard_B*` restricted |
| `eastus` / `eastus2` / `westus3` | None | All `Standard_B*` restricted |
| `westus2` / `centralus` / `canadacentral` | ARM64 B-series v2 only (`B2pts_v2`, `B2pls_v2`, `B2ps_v2`, ...) | Non-x86_64 and outside EMEA |
| **`swedencentral`** | **31 unrestricted B-series v2 SKUs** (`Standard_B2ats_v2`, `Standard_B2ts_v2`, `Standard_B2ls_v2`, `Standard_B2s_v2`, ...) | **Closest EMEA region with unrestricted x86_64 B-series v2** |

---

## 4. Recommended Fallback Region & SKU (Pending Approval)

- **Recommended Region:** `swedencentral`
- **Recommended VM SKU:** `Standard_B2ats_v2` (2 vCPUs, `1024 MB` / 1 GB RAM, x86_64 AMD, **no restrictions**)
- **Secondary SKU option:** `Standard_B2ts_v2` (2 vCPUs, `1024 MB` / 1 GB RAM, x86_64 Intel, **no restrictions**)

### Verified Quotas in `swedencentral`

| Quota | Current | Limit |
|---|---:|---:|
| Total Regional vCPUs | `0` | `6` |
| Standard Basv2 Family vCPUs (`Standard_B2ats_v2`) | `0` | `10` |
| Standard Bsv2 Family vCPUs (`Standard_B2ts_v2`) | `0` | `10` |
| Virtual Networks | `0` | `1000` |
| Network Security Groups | `0` | `5000` |
| Public IP Addresses (Total) | `0` | `3` |
| Public IPv4 Addresses - Standard | `0` | `3` |
| Public IPv4 Addresses - Basic | `0` | `0` |

Note: `Basic` Public IP quota is `0`, which aligns with `B6-cloud.md` §2 (`--public-ip-sku Standard`).

---

## 5. Expected Monthly Credit Impact (`swedencentral`)

Verified via Azure Retail Prices API (`armRegionName eq 'swedencentral'`, `priceType eq 'Consumption'`, Linux):

| Item | Rate | 24/7 Estimate (730 h/mo) | Lab Schedule Estimate (~120 h/mo, deallocated when idle) |
|---|---|---|---|
| `Standard_B2ats_v2` VM (`Basv2 Series`) | `$0.00972 / hr` | `~$7.10 / mo` | `~$1.17 / mo` |
| `Standard_B2ts_v2` VM (alt., `Bsv2 Series`) | `$0.0108 / hr` | `~$7.88 / mo` | `~$1.30 / mo` |
| 30 GB OS Disk + Standard Public IP + Blob LRS | Standard student/PAYG rates | To be tracked under `$10 / mo` Portal Budget | Remaining charges (`disk + public IP + storage`) monitored via Portal Budget |

---

## 6. Commands Run (Sanitised)

```powershell
az login
az account show -o table
az provider register --namespace Microsoft.Compute --wait
az provider register --namespace Microsoft.Network --wait
az provider register --namespace Microsoft.Storage --wait
az vm list-usage --location uaenorth -o table
az network list-usages --location uaenorth -o table
az rest --method get --uri "https://management.azure.com/subscriptions/<REDACTED-SUBSCRIPTION-ID>/providers/Microsoft.Compute/skus?api-version=2021-07-01&`$filter=location eq 'uaenorth'" ...
az vm list-usage --location swedencentral -o table
az network list-usages --location swedencentral -o table
az vm list-sizes --location swedencentral ...
```

---

## 7. Required Decisions & Next Dependency

1. **Mahmoud (Design & Integration Sign-off):** Approve switching the B6 target region from `uaenorth` to **`swedencentral`** and VM SKU from `Standard_B1s` to **`Standard_B2ats_v2`** (and reflect the EU region in `report/B-runbooks/B6-cloud.md` §1/§5/§6.6 & Deliverable D — all stored data remains synthetic).
2. **Mahmed Shaker (Report / Ch. 6 & 8):** Note `swedencentral` + `Standard_B2ats_v2` once approved for Chapter 6 / data-residency wording.
3. **Gate 3 Prerequisite:** Wait for Gate 3 acceptance before provisioning `nilecare-dr`, `vnet-dr`, and `cloud-dr` so credit is not consumed prematurely.
