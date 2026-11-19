# Iprahim Said - Azure Disaster Recovery and Cloud Security

ابدأ بفحص الاشتراك والـregion والـSKU من B6 §1. ابعت حالة التوفر والتكلفة المتوقعة لمحمود قبل إعداد الموارد. نسّق معاه وقت اختبار الاتصال بين الطرفين.

## Mission

Build the Azure side independently, document every cloud constraint, and deliver a cloud endpoint that Mahmoud and Youssef can connect to without receiving secrets in chat.

## Primary work

1. B6 §1 pre-flight: confirm Azure for Students subscription, allowed region, `Standard_B1s` availability, quota, public IP availability, and expected monthly credit impact.
2. Once Gate 3 is accepted and the pre-flight values are confirmed, create the resource group, `vnet-dr 172.16.0.0/16`, subnet `172.16.10.0/24`, NSG, public IP, and `cloud-dr`.
3. Configure Ubuntu Server 24.04, strongSwan cloud side, MariaDB replica prerequisites, restic destination, and Blob storage with versioning/immutability.
4. Use B6 §6 for Azure Monitor and cost controls. Deallocate the VM outside scheduled DR or backup windows. Keep a record of remaining charges for disks, public IP and storage.
5. Coordinate the cloud side of Gate 4 with Mahmoud (pfSense) and Youssef (MariaDB). In Gate 5, coordinate cloud-dr agent telemetry with Shaker over the approved tunnel path.

## Required outputs

- `HANDOFF-Iprahim-Said.md`.
- `azure-preflight.md` with region, SKU, quota, subscription type, and blockers.
- `azure-resources.txt` or redacted CLI output naming resources, subnets, NSG rules, public IP, and effective region.
- Blob immutability/versioning evidence and retention policy output.
- strongSwan cloud-side configuration with `<SET-IN-VAULT>` for PSK and no private secrets.
- Azure Monitor alert evidence and deallocation record.

## Acceptance checks

- The effective VNet and subnet match A exactly.
- Only the required inbound paths are open; SSH is restricted to the documented management path.
- Blob immutability is enabled at account creation and its retention is recorded.
- The public IP is sent to Mahmoud as a non-secret value; the PSK is never sent in chat.
- Gate 4 reports `INSTALLED` only after both sides provide evidence and the replication wire test succeeds.

## Stop conditions

Stop before creating resources if the subscription does not allow the planned region/SKU and report the exact error. Do not silently switch region, SKU, or retention. A changed cloud design requires a recorded approval.

## Where your work goes

Use [the handoff template](08-HANDOFF-TEMPLATE.md) for pre-flight, VM/network, tunnel, replica, Blob, Monitor, and the final exercise. Give Mahmoud the non-secret VPN endpoint details and tunnel status. Give Youssef the replica endpoint and verification output. Give Shaker the approved network values and evidence for Chapters 6 and 8, including actual retention, region, SKU, and whether deallocation has been demonstrated.
