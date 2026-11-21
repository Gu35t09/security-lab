# Phase 1 runbook: infrastructure

Step-by-step order for phase 1 of [PROJECT-PLAN.md](../PROJECT-PLAN.md):
switch, Proxmox on three nodes, the lab OPNsense, the cluster, hardening and
the tests that close the phase.

The order matters. The Proxmox nodes use the lab OPNsense as their gateway,
and the lab OPNsense is a VM on the N100. So the N100 comes first, reached from
the emergency port, then the OPNsense, then the two NodeWeavers.

## Decisions taken for this phase

| Topic | Decision | Reason |
|---|---|---|
| VLANs on the trunks | All lab VLANs tagged, no native VLAN in use | No ambiguity between untagged and VLAN 600 on the Proxmox bridges |
| Host management address | On `vmbr0.600`, a VLAN interface on a VLAN-aware bridge | Standard Proxmox pattern for a tagged management network |
| Bridges | Linux VLAN-aware bridges on all nodes | Open vSwitch is needed only for the mirrors, so it arrives in phase 4 |
| DNS domain of the hosts | `lab.internal` | `.internal` is reserved for private use. `.local` clashes with mDNS |
| Proxmox repositories | No-subscription | Lab use, no subscription |
| Internet for LAB-MGMT | Allowlist: Proxmox and Debian repositories | Missing from section 4.3 of the plan: the hosts need updates |

## Before starting

Values to fill in before step 3. They live on the router network, so only
you know them.

| Value | Used for | Value chosen |
|---|---|---|
| Lab OPNsense WAN address on 10.0.0.0/24, outside the router DHCP range | OPNsense WAN, router static route | |
| Production OPNsense WAN address on 10.0.0.0/24 | Lab rule that lets the notebook in | |

Downloads, on the notebook:
- Proxmox VE 9.2 ISO from the official download page. Check its SHA256 against the one published on that page.
- OPNsense, the latest `dvd` image for amd64, from an official mirror. Check its SHA256 against the published checksum file.

Write the Proxmox ISO to a USB stick. The OPNsense image does not need a USB
stick: it is uploaded to the N100 through the Proxmox web interface.

Fixed addresses used in this runbook:

| Host | LAB-MGMT, VLAN 600 | LAB-CLUSTER, VLAN 602 |
|---|---|---|
| Lab OPNsense | 10.66.0.1 | none |
| `nw1` | 10.66.0.21 | 10.66.2.21 |
| `nw2` | 10.66.0.22 | 10.66.2.22 |
| `n100` | 10.66.0.23 | 10.66.2.23 |
| Notebook on the emergency port | 10.66.0.250, static, no gateway | none |

## Step 1: switch

Create VLANs 600, 602, 610, 620, 621, 630 and 640, then configure the ports.
Port numbers depend on the switch; record them in the last column.

| Device and port | Switch port mode | VLANs | Switch port |
|---|---|---|---|
| `n100` LAN2, `enp2s0` | Trunk | 600, 610, 620, 621, 630, 640 tagged | |
| `n100` LAN3, `enp3s0` | Access | 602 | |
| `nw1` `eno1` | Trunk | 600, 610, 620, 621, 630, 640 tagged | |
| `nw1` `eno2` | Access | 602 | |
| `nw2` `eno1` | Trunk | 600, 610, 620, 621, 630, 640 tagged | |
| `nw2` `eno2` | Access | 602 | |
| Emergency port, notebook | Access | 600 | |

On the trunk ports, set the native VLAN to an unused VLAN, or remove the
untagged membership, depending on what the switch allows.

`n100` LAN1 goes to the router, not to the switch. The direct mirror cables
and the SPAN port belong to phase 4. The IPMI ports stay as they are.

## Step 2: Proxmox on the N100

Cabling for this step: LAN2 to its trunk port, LAN3 to its corosync port,
nothing on LAN1 yet.

Installer settings:

| Setting | Value |
|---|---|
| Target disk | `sda`, filesystem ext4 |
| Disk options | Defaults: LVM with a root volume and the `data` thin pool |
| Country and time zone | Italy, Europe/Rome |
| Management interface | `enp2s0` |
| Hostname | `n100.lab.internal` |
| IP address | 10.66.0.23/24 |
| Gateway | 10.66.0.1 |
| DNS server | 10.66.0.1 |

The installer puts the address straight on the bridge, untagged. The switch
port only carries tagged VLANs, so the node is not reachable yet. That is
expected.

After the first boot, on the N100 console, replace the network part of
`/etc/network/interfaces` with:

```
auto lo
iface lo inet loopback

# LAN1: passed through to the OPNsense VM, never used by the host.
iface enp1s0 inet manual

# LAN2: trunk with all the lab VLANs, tagged.
iface enp2s0 inet manual

# LAN3: corosync, VLAN 602 untagged on the switch.
auto enp3s0
iface enp3s0 inet static
    address 10.66.2.23/24

# LAN4: mirror output to nw1, used from phase 4.
iface enp4s0 inet manual

auto vmbr0
iface vmbr0 inet manual
    bridge-ports enp2s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# Management address, VLAN 600 tagged.
auto vmbr0.600
iface vmbr0.600 inet static
    address 10.66.0.23/24
    gateway 10.66.0.1
```

Apply it with `ifreload -a`. Then check the LAN1 mapping, as noted in the
inventory: `ip -br link` must show `enp1s0` as the port without a link, and
`enp2s0` up.

From the notebook on the emergency port, with 10.66.0.250/24 and no gateway,
open `https://10.66.0.23:8006`. There is no Internet yet, so updates wait for
step 5.

IOMMU check, needed for the WAN passthrough:

```
dmesg | grep -i -e DMAR -e IOMMU | head
```

The output must mention that the IOMMU is enabled. Recent Proxmox kernels
enable it by default on Intel; if not, add `intel_iommu=on` to the kernel
command line.

## Step 3: lab OPNsense VM

Upload the OPNsense image to the `local` storage of the N100 from the web
interface, then create the VM:

| Setting | Value |
|---|---|
| VM ID and name | 100, `opnsense-lab` |
| Machine | q35, BIOS OVMF or SeaBIOS |
| CPU | 2 cores, type `host` |
| RAM | 2 GB, ballooning off |
| Disk | 20 GB on `local-lvm`, VirtIO SCSI |
| Network | `net0` VirtIO on `vmbr0`, no VLAN tag: the VM receives every tagged VLAN |
| PCI device | `0000:01:00.0`, LAN1, all functions off, PCI-Express on |
| Start at boot | Yes, order 1 |

Plug LAN1 into the router, start the VM and install OPNsense. Then assign
the interfaces:

| OPNsense interface | Device | Address |
|---|---|---|
| WAN | `igc0`, the passed-through LAN1 | Lab OPNsense WAN address chosen above, gateway 10.0.0.1 |
| LAB_MGMT | VLAN 600 on `vtnet0` | 10.66.0.1/24 |
| LAB_SOC | VLAN 610 on `vtnet0` | 10.66.10.1/24 |
| LAB_TGT_WIN | VLAN 620 on `vtnet0` | 10.66.20.1/24 |
| LAB_TGT_LNX | VLAN 621 on `vtnet0` | 10.66.21.1/24 |
| LAB_ATTACK | VLAN 630 on `vtnet0` | 10.66.30.1/24 |
| LAB_BUILD | VLAN 640 on `vtnet0` | 10.66.40.1/24 |

`vtnet0` itself stays unassigned. VLAN 602 has no gateway and does not reach
the OPNsense.

Settings that are easy to miss:
- **WAN, "Block private networks" off.** The router network is private, and the notebook arrives from it. With the default on, the static route is useless.
- **Outbound NAT automatic** on WAN, as decided in the plan.
- **Unbound** listening on LAB_MGMT and LAB_SOC only for now. Targets and attack get no DNS from the OPNsense until phase 2.
- **NTP** served on LAB_MGMT, for the Proxmox hosts.
- **Web interface** on LAB_MGMT only, with the default anti-lockout rule.

Aliases:

| Alias | Type | Content |
|---|---|---|
| `PROD_OPNSENSE_WAN` | Host | Production OPNsense WAN address chosen above |
| `PVE_REPOS` | Host | `download.proxmox.com`, `deb.debian.org`, `security.debian.org` |
| `SOC_ALLOW` | Host | `github.com`, `deb.debian.org`, `security.debian.org`, extended in later phases |
| `LAB_NETS` | Network | 10.66.0.0/16 |
| `ROUTER_NET` | Network | 10.0.0.0/24 |

Rules for this phase. OPNsense blocks anything without a matching rule.

| Interface | Action | Source | Destination | Ports | Purpose |
|---|---|---|---|---|---|
| WAN | Pass | `PROD_OPNSENSE_WAN` | LAB_MGMT net | 22, 443, 8006 | Notebook access through the static route |
| LAB_MGMT | Pass | LAB_MGMT net | `LAB_NETS` | any | Management of the lab |
| LAB_MGMT | Pass | LAB_MGMT net | `PVE_REPOS` | 80, 443 | Proxmox and Debian updates |
| LAB_SOC | Pass | LAB_SOC net | `SOC_ALLOW` | 80, 443 | Allowlisted Internet |
| Every lab interface | Block | any | `ROUTER_NET` | any | Keep the lab out of the router network |

The block on `ROUTER_NET` goes above any pass rule that could match the
router network. The other target and attack rules arrive with phases 2 and 6.

Export the configuration from System, Configuration, Backups and keep it on
the notebook.

## Step 4: router static route and notebook access

On the router:
- static route 10.66.0.0/16 via the lab OPNsense WAN address;
- if the WAN address is outside the DHCP range, nothing else is needed.

Then, from the notebook on its normal production network:

```
ssh root@10.66.0.23
```

and open `https://10.66.0.23:8006`. This also tests the asymmetric path from
section 4.1 of the plan: keep the SSH session open for a few minutes and
check that it does not drop. If the router drops it, the fallback is in the
plan.

## Step 5: updates on the N100

In the web interface, Node, Updates, Repositories:
- disable the `enterprise` repositories for Proxmox VE and Ceph;
- add the `No-Subscription` repository.

Then on the N100:

```
apt update
apt full-upgrade
```

Point the time sync to the lab OPNsense: in `/etc/chrony/chrony.conf`
replace the default pool with `server 10.66.0.1 iburst`, then
`systemctl restart chrony`.

## Step 6: Proxmox on nw1 and nw2

Same installer settings as the N100, with these differences:

| Setting | `nw1` | `nw2` |
|---|---|---|
| Management interface | `eno1` | `eno1` |
| Hostname | `nw1.lab.internal` | `nw2.lab.internal` |
| IP address | 10.66.0.21/24 | 10.66.0.22/24 |
| Corosync interface and address | `eno2`, 10.66.2.21/24 | `eno2`, 10.66.2.22/24 |

The console after the first boot is the IPMI remote console. The network
file follows the same pattern as the N100: `eno1` in `vmbr0`, `eno2` with the
corosync address, the management address on `vmbr0.600`. The other ports
stay `manual` until phase 4.

Then repeat step 5 on both nodes.

## Step 7: cluster

On the N100:

```
pvecm create lab --link0 10.66.2.23 --link1 10.66.0.23
```

On `nw1`:

```
pvecm add 10.66.0.23 --link0 10.66.2.21 --link1 10.66.0.21
```

On `nw2`:

```
pvecm add 10.66.0.23 --link0 10.66.2.22 --link1 10.66.0.22
```

Check from any node:

```
pvecm status
corosync-cfgtool -s
```

The first must show three nodes and quorum. The second must show both links
connected towards the other two nodes.

## Step 8: hardening

- **Second factor** for `root@pam` and for a personal admin user, from Datacenter, Permissions, Two Factor. TOTP is the simplest option. WebAuthn with the YubiKey needs a stable origin name, so it can come later.
- **Personal admin user** `v@pve` with the Administrator role on `/`, used for daily work instead of `root@pam`.
- **SSH keys**: add the notebook's YubiKey public key to `/root/.ssh/authorized_keys` on each node, check that key login works, then set `PasswordAuthentication no` in an sshd drop-in and restart sshd. The cluster's own node-to-node SSH keys are not affected.

## Step 9: backups

- **OPNsense configuration**: export after every change, kept on the notebook.
- **VM backups**: on `nw1`, create a directory for backups, export it over NFS to 10.66.0.22 and 10.66.0.23 only, and add it to the cluster as an NFS storage with content `VZDump backup file`. Then a weekly backup job for VM 100, the lab OPNsense.

The OPNsense lives on the N100's entry-level disk, so its backup is the most
important one in this phase.

## Step 10: tests that close phase 1

| Test | Expected result |
|---|---|
| Test VM on LAB-TGT-WIN, `ping 1.1.1.1` and `curl https://example.com` | Both fail |
| Same test VM, `ping 10.0.0.1` and any other router network address | Fails |
| Test VM on LAB-SOC, `curl https://github.com` | Works |
| Same test VM, `curl https://example.com` | Fails |
| Notebook on the production network, SSH to `nw1` | Stable session, no drops |
| Notebook, `https://10.66.0.21:8006` | Proxmox login with second factor |
| `pvecm status` | Three nodes, quorate |
| Unplug the corosync cable of one node for a minute | Cluster stays quorate on link1 |

A small Debian cloud image or the Debian netinst ISO is enough for the test
VMs. There is no DHCP on the lab VLANs yet, so give each test VM a static
address, for example 10.66.20.50/24 with gateway 10.66.20.1, and 10.66.10.50/24
with gateway 10.66.10.1 and DNS 10.66.10.1. Delete the test VMs after the
tests.

When every test passes, phase 1 is done.
