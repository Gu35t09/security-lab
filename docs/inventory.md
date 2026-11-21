# Phase 0 inventory

Hardware facts collected in phase 0. Serial numbers and MAC addresses are kept
out of this public repository; MAC addresses go to the private Ansible
inventory later.

## nw1

Collected on 2026-11-21 from a Debian 13 test install.

| Item | Value |
|---|---|
| Board | Supermicro, based on the onboard NIC and BMC vendor |
| CPU | Intel Xeon E3-1230 v3, 4 cores, 8 threads, VT-x |
| VT-d | Enabled: the kernel exposes IOMMU groups |
| RAM | 32 GB, confirmed. This node is `nw1` |
| Disk | WD Red SA500 2 TB SATA SSD |
| Disk health | SMART passed, no reallocated sectors, no errors logged, 2525 power-on hours. Extended self-test completed without error |
| Disk temperature | 44 °C now, 76 °C recorded maximum |

### Network interfaces

| Interface | Controller | PCI address | IOMMU group | Role in the plan |
|---|---|---|---|---|
| `eno1` | Intel I210, onboard | 03:00.0 | 11 | Onboard #1: trunk |
| `eno2` | Intel I210, onboard | 04:00.0 | 12 | Onboard #2: corosync |
| `enp5s0f0` | Intel I350, 4-port card | 05:00.0 | 13 | Card #1: sniffing, mirror from `nw2` |
| `enp5s0f1` | Intel I350, 4-port card | 05:00.1 | 14 | Card #2: sniffing, mirror from the N100 |
| `enp5s0f2` | Intel I350, 4-port card | 05:00.2 | 15 | Card #3: sniffing, switch SPAN |
| `enp5s0f3` | Intel I350, 4-port card | 05:00.3 | 16 | Card #4: spare |

The 4x 1 GbE card is already installed. Every port has its own IOMMU group,
so any of them could be passed through to a VM if needed.

### IPMI

| Item | Value |
|---|---|
| BMC | Supermicro, with its own MAC address, separate from `eno1` and `eno2` |
| LAN mode | Dedicated, set on 2026-11-21. It was failover (`02`), which falls back to `eno1` without a link on the IPMI port |
| Current address | Static, 10.0.0.211/24, gateway 10.0.0.1, left over from the previous setup |
| VLAN tagging | Disabled |
| Enabled authentication types | MD2, MD5, PASSWORD |
| SNMP community | `public` |

Open points:
- **LAN mode.** Done: dedicated, so the BMC never appears on the trunk port.
- **Hardening.** A Haswell-era BMC has known weaknesses. It must live only on LAB-MGMT, with a new admin password and the latest firmware available.
- **Placement, decided 2026-11-21.** The BMC stays on the router network, 10.0.0.0/24, for now. That keeps out-of-band access even when the lab nodes or the lab OPNsense are down. The move to LAB-MGMT, at 10.66.0.11, is postponed.
- **While it stays on the router network,** every device on that network can reach it. A new admin password, the latest firmware and SNMP turned off reduce the exposure. Admin password changed and BMC firmware updated on 2026-11-21. SNMP still to check.

## nw2

Collected on 2026-11-21 from a Debian 13 test install.

| Item | Value |
|---|---|
| Board | Supermicro, same model as `nw1` |
| CPU | Intel Xeon E3-1230 v3, 4 cores, 8 threads, VT-x |
| VT-d | Enabled: the kernel exposes IOMMU groups |
| RAM | 24 GB, confirmed. This node is `nw2` |
| Disk | WD Red SA500 2 TB SATA SSD |
| Disk health | SMART passed, no reallocated sectors, no errors logged, 2521 power-on hours. Extended self-test completed without error |
| Disk temperature | 43 °C now, 76 °C recorded maximum, the same as `nw1` |

Both disks reached 76 °C in the past, above the usual 70 °C operating limit
of SATA SSDs. Probably the previous workload, since the two histories match.
Worth watching once Proxmox runs, with smartd alerts.

### Network interfaces

| Interface | Controller | PCI address | IOMMU group | Role in the plan |
|---|---|---|---|---|
| `eno1` | Intel I210, onboard | 03:00.0 | 11 | 1 GbE #1: trunk |
| `eno2` | Intel I210, onboard | 04:00.0 | 12 | 1 GbE #2: corosync |
| `enp5s0` | Intel I226-V, 2.5 GbE card | 05:00.0 | 13 | 2.5 GbE: mirror output to `nw1` |

### IPMI

| Item | Value |
|---|---|
| BMC | Supermicro, with its own MAC address, separate from `eno1` and `eno2` |
| LAN mode | Dedicated, set on 2026-11-21. It was failover (`02`) |
| Current address | Static, 10.0.0.212/24, gateway 10.0.0.1, on the router network |
| VLAN tagging | Disabled |
| Enabled authentication types | MD2, MD5, PASSWORD |
| SNMP community | `public` |

Open points:
- **LAN mode.** Done: dedicated.
- **Hardening.** Admin password changed, IPMI users checked and BMC firmware updated on 2026-11-21. SNMP still to check. It stays on the router network for now, like `nw1`.

## n100

Not collected yet.
