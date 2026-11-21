# Security Lab: project plan

Status: design draft, nothing implemented yet.
Last updated: 2026-11-21.

## 1. Goal

Build a test lab to learn and experiment with the most complete open source
security platform the available hardware can run.
The lab is not production. It must stay fully separate from the environment
managed by `ansible-prod-srv`, and must not require any change to it.

## 2. Decisions

| Topic | Decision |
|---|---|
| Switch | Managed, with VLANs and a mirror port available |
| Windows and Active Directory | Yes, tuned for minimal RAM. Slowness and alternating between the Windows and Linux scenarios are acceptable |
| Internet | Security systems go out through an allowlist. Target and attack VMs never go out |
| Firewall | OPNsense dedicated to the lab, connected directly to the ISP router, alongside the production OPNsense |
| Routing | Outbound NAT on the lab OPNsense. Static routes on the router only so the production notebook can reach the lab without a VPN |
| Mirroring | Open vSwitch mirrors on `nw2` and on the N100, each with a direct cable to `nw1` |
| `nw1` network card | 4x 1 GbE instead of the 1x 2.5 GbE |
| Additional SIEM | Graylog Open, alongside Security Onion |
| Virtualization | Proxmox VE cluster on three nodes |
| Storage | ext4 with LVM-thin, the default Proxmox install. No ZFS, no Ceph, no replication |
| Switch speed | 1 GbE on every port |
| Automation | New repository following the same conventions as the production repositories |

## 3. Hardware and node roles

| Node | Hardware | Role |
|---|---|---|
| `nw1` | NodeWeaver, Xeon 4c/8t, 32 GB, 2 TB SSD, 2x 1 GbE onboard, 4x 1 GbE card | Security Onion standalone |
| `nw2` | NodeWeaver, Xeon 4c/8t, 24 GB, 2 TB SSD, 2x 1 GbE, 1x 2.5 GbE | Targets, Kali, Graylog |
| `n100` | N100 mini PC 4c/4t, 32 GB, 1 TB SSD, 4x 2.5 GbE | Lab OPNsense, Wazuh, Caldera, light SOC services, quorum |

The 32 GB NodeWeaver must be `nw1`. Security Onion standalone needs at least
24 GB and does not fit on the 24 GB node.

Per-node reserves:

| Item | RAM |
|---|---|
| Proxmox system | 2 GB |
| Usable RAM on `nw1` and `n100` | about 30 GB |
| Usable RAM on `nw2` | about 22 GB |

### 3.0 Storage

Standard Proxmox install on ext4: the system lives on an ext4 volume, the VMs
on an LVM-thin pool on the same disk.

LVM-thin uses no RAM for caching and keeps the two features the lab needs:
- VM **snapshots**, to go back to a "clean" state after each attack;
- **linked clones** from templates, which save disk space.

Each node has its own local storage. With no replication, a VM lives only on
its own node, and if the disk fails the VM can only be recovered from backup.
ISOs and backups live on a directory storage, also on ext4.
A backup on the same disk as the VM does not protect against a disk failure.
Backups of the SOC VMs therefore go to another node, shared over NFS, or to an
external disk.

KSM, the Proxmox memory deduplication, must be enabled on `nw2`.
Several identical Windows VMs share many pages, and the saving is real.

### 3.1 Lab OPNsense: virtual on the N100

Recommendation: OPNsense as a VM on the N100, with the WAN port assigned to
the VM through PCI passthrough.

Reasons:
- the N100 stays the third cluster node, needed for quorum;
- its four ports are enough for WAN, LAN, cluster and the mirror to Security Onion;
- with passthrough, the Proxmox host never has an address on the router network.

Fallback if passthrough does not work: a dedicated bridge on the WAN port,
with no IP address on the host.

Accepted consequence: if the N100 goes down, the whole lab loses its network.

Rejected alternative: OPNsense installed directly on the N100. It would need
another device for quorum and would take 32 GB of RAM away from the cluster.

## 4. Network

### 4.1 Connection to the router

```
ISP router
 ├── Production OPNsense  (unchanged)
 └── Lab OPNsense, WAN    (new, on the N100)
        └── Lab VLANs
```

Static routes on the router:

| Destination | Gateway |
|---|---|
| 10.66.0.0/16 | WAN address of the lab OPNsense |

Agreed behaviour:
- **Outbound to the Internet:** the lab OPNsense applies outbound NAT on its WAN. The router only sees the lab's WAN address.
- **Inbound from the production notebook:** the router routes 10.66.0.0/16 to the lab WAN. The notebook reaches LAB-MGMT without a VPN.
- **Fixed WAN address** for the lab OPNsense, through a DHCP reservation or a static address.

Traffic path from the notebook:

```
notebook  ->  prod OPNsense  ->  router  ->  lab OPNsense  ->  LAB-MGMT
notebook  <-  prod OPNsense  <----------------  lab OPNsense
```

Replies do not go back through the router: the lab OPNsense answers the
production OPNsense WAN directly, since both sit on the same network. The path
is asymmetric. Both OPNsense instances handle that fine, but some ISP routers
drop TCP connections when they only see one direction.
This must be tested in phase 1. If the router drops them, the only fix is a
static route to 10.66.0.0/16 on the production OPNsense, which is a small
change to production.

### 4.2 Segments

VLAN IDs and subnets are placeholders. They must be checked against the
production address plan to avoid overlaps.

| VLAN | Name | Subnet | Contents |
|---|---|---|---|
| 600 | LAB-MGMT | 10.66.0.0/24 | Proxmox interfaces, tool web UIs |
| 602 | LAB-CLUSTER | 10.66.2.0/24 | Proxmox cluster corosync traffic, no gateway |
| 610 | LAB-SOC | 10.66.10.0/24 | Security Onion, Graylog, Wazuh, Velociraptor, MISP, IRIS, Shuffle |
| 620 | LAB-TGT-WIN | 10.66.20.0/24 | AD domain and Windows clients |
| 621 | LAB-TGT-LNX | 10.66.21.0/24 | Vulnerable Linux servers |
| 630 | LAB-ATTACK | 10.66.30.0/24 | Kali, Caldera |
| 640 | LAB-BUILD | 10.66.40.0/24 | Template preparation with allowlisted Internet access |

Mirrored traffic does not use a VLAN: it travels on a direct cable, see 4.5.

Main fixed addresses:

| Host | Address |
|---|---|
| Lab OPNsense, gateway of every VLAN | .1 |
| `nw1`, `nw2`, `n100` on LAB-MGMT | 10.66.0.21, .22, .23 |
| `nw1`, `nw2`, `n100` on LAB-CLUSTER | 10.66.2.21, .22, .23 |
| Reserved for the IPMI interfaces of `nw1`, `nw2`, `n100` | .11, .12, .13 |

Addresses .11 to .13 stay reserved in both subnets, so every node follows the
same scheme: .1x for IPMI, .2x for the system. The IPMI interfaces live on
LAB-MGMT.

To check in phase 0: whether the NodeWeaver IPMI has a dedicated port or
shares a network port. A dedicated port needs two more access ports on VLAN
600 of the switch.

### 4.3 Traffic rules

The lab OPNsense is the only filter between the lab and the rest of the world.
The production OPNsense still protects production, because it sees the lab
like any other host on its WAN side.

| From | To | Allowed |
|---|---|---|
| Admin access, see 4.4 | LAB-MGMT | HTTPS and SSH |
| LAB-MGMT | Any lab VLAN | Management |
| LAB-SOC | Internet | Allowlist only: package repositories, ET rules, MISP feeds, GitHub |
| LAB-BUILD | Internet | Allowlist only: Windows Update, Debian, Ubuntu and Kali mirrors |
| LAB-TGT-WIN and LAB-TGT-LNX | LAB-SOC | Agent ports only: Wazuh, Velociraptor, Elastic Agent, Graylog Sidecar |
| LAB-ATTACK | LAB-TGT-WIN and LAB-TGT-LNX | Everything |
| LAB-ATTACK | LAB-SOC | Nothing, except for specific exercises |
| Any lab VLAN | Router network and private addresses outside the lab | Never, except the router as gateway |
| Targets and attack | Internet | Never |

The router network rule matters. The lab OPNsense shares that network with the
production OPNsense WAN, and no lab host may be able to explore it.

Targets use the domain controller or the lab OPNsense for DNS, with no
forwarding to the outside.

### 4.4 Admin access

Normal access: from the `v-admin` notebook, through the static routes, without
a VPN.

The production OPNsense applies NAT towards the router. The lab therefore sees
every production host with the same address, the production OPNsense WAN, and
cannot tell the notebook from the server. Countermeasures:
- **Rule on the lab OPNsense:** LAB-MGMT accepts HTTPS and SSH only from the production OPNsense WAN address. The rest of the router network is blocked.
- **SSH with a YubiKey**, as already done towards the production server.
- **Second factor** on the Proxmox and OPNsense web interfaces, with WebAuthn on the YubiKey or TOTP.

If the notebook's outbound traffic through the production OPNsense is
allowlisted, a rule allowing 10.66.0.0/16 is needed. If outbound traffic is
open, nothing is needed.

Emergency access: a switch port configured as an access port on LAB-MGMT.
Plug the notebook in with a cable when the path through the router does not
work. The notebook and the Proxmox hosts are then on the same subnet, so
access works even with the lab OPNsense down.

### 4.5 Physical ports

On `nw1`, the 1x 2.5 GbE PCIe card is replaced with a 4x 1 GbE card.
`nw1` then has six 1 GbE ports, two onboard and four on the card.

```
                              Internet
                                 |
                    +------------+------------+
                    |       ISP router        |
                    |  route 10.66.0.0/16     |
                    |  via lab OPNsense WAN   |
                    +----+---------------+----+
                         |               |
              +----------+-------+       |
              | prod OPNsense    |       |
              | unchanged        |       |
              +----------+-------+       |
                         |               |
                 production network      |
                 v-admin notebook        |
                                         | p1, WAN in passthrough
                         +---------------+---------------+
                         |             n100              |
                         |  lab OPNsense VM, Wazuh, SOC  |
                         |      p2         p3        p4  |
                         +-------+----------+---------+--+
                                 |          |         |
                            trunk, all    VLAN 602    | N100 mirror:
                            lab VLANs     corosync    | mgmt, SOC, build
                                 |          |         |
        +------------------------+----------+------+  |
        |       Managed switch, 1 GbE only         |  |  VLAN 600 port:
        |                                          +--+--- emergency,
        +--+--------+--------+-------+--------+----+  |   notebook cable
           |        |        |       |        |       |
        trunk  VLAN 602    SPAN   trunk  VLAN 602     |
           |        |        |       |        |       |
    +------+--------+--------+--+ +--+--------+-----+ |
    |  nw1   32 GB              | |  nw2   24 GB    | |
    |  onb. 1   trunk           | |  1G 1    trunk  | |
    |  onb. 2   corosync        | |  1G 2    coros. | |
    |  card 3   switch SPAN     | |                 | |
    |  card 1   sniff      +----+-+--+ 2.5G mirror  | |
    |  card 2   sniff      +----+-+  | OVS output   | |
    |  card 4   spare      |    | |  +--------------+ |
    |  Security Onion      +----+-+-------------------+
    +-------------------------+
         direct cables: mirrored traffic only,
         no IP, not through the switch
```

| Node | Port | Connection | Effective speed | Use |
|---|---|---|---|---|
| `n100` | 2.5 GbE #1 | Router | Depends on the router | Lab OPNsense WAN, in passthrough |
| `n100` | 2.5 GbE #2 | Switch, trunk | 1 GbE | All lab VLANs |
| `n100` | 2.5 GbE #3 | Switch, VLAN 602 | 1 GbE | Corosync |
| `n100` | 2.5 GbE #4 | Direct cable to `nw1`, card #2 | 1 GbE | N100 mirror output |
| `nw1` | Onboard 1 GbE #1 | Switch, trunk | 1 GbE | Management and SOC VLAN |
| `nw1` | Onboard 1 GbE #2 | Switch, VLAN 602 | 1 GbE | Corosync |
| `nw1` | Card 1 GbE #1 | Direct cable from `nw2` | 1 GbE | Sniffing: targets and attack |
| `nw1` | Card 1 GbE #2 | Direct cable from the N100 | 1 GbE | Sniffing: management, SOC, build |
| `nw1` | Card 1 GbE #3 | Switch, SPAN destination | 1 GbE | Sniffing: experiments on physical traffic |
| `nw1` | Card 1 GbE #4 | Free | | Spare |
| `nw2` | 1 GbE #1 | Switch, trunk | 1 GbE | VM traffic |
| `nw2` | 1 GbE #2 | Switch, VLAN 602 | 1 GbE | Corosync |
| `nw2` | 2.5 GbE | Direct cable to `nw1` | 1 GbE | `nw2` mirror output |
| Switch | One port | Access, VLAN 600 | 1 GbE | Emergency access |

The 2.5 GbE ports of `nw2` and the N100 run at 1 GbE, because the `nw1` card is
1 GbE. That is enough for the lab: scans and attacks generate little traffic.
The limit only shows when copying large files between VMs on the same node.

Corosync uses two rings: the first on the dedicated VLAN 602, the second on
the management network as a backup.

The 1 GbE trunk is enough for the lab. It only becomes the bottleneck for heavy
transfers between nodes, such as backups or moving a VM.

Card compatibility with the PCIe slot and the chassis: verified.

### 4.6 Mirroring to Security Onion

Traffic between VMs on the same node never goes through the switch, so
internal mirrors are needed. Each mirror has a direct cable to `nw1`, with no
transport VLAN and without going through the switch.

| Source | Mirrored VLANs | Receiving port on `nw1` |
|---|---|---|
| OVS mirror on `nw2` | 620, 621, 630: targets and attack | Card #1 |
| OVS mirror on the N100 | 600, 610, 640: management, SOC, build | Card #2 |
| Switch SPAN | As needed, for experiments | Card #3 |

The three receiving ports are attached to the Security Onion VM as sniffing
interfaces, with no IP. Security Onion bonds them into one logical interface.

The N100 mirror copies only the VLANs that `nw2` does not already copy. That
way, traffic between Kali and the targets, which also crosses the OPNsense, is
not seen twice. Some flows will still show up twice, for example target agents
talking to Wazuh. That is acceptable for a lab.

The OPNsense WAN is in passthrough and does not go through Open vSwitch, so no
mirror sees it. Suricata built into OPNsense covers it, running on the WAN,
with alerts sent to Graylog and Security Onion.

If a copy exceeds 1 GbE, Security Onion drops packets but the lab keeps
working.

## 5. VM sizing

### 5.1 `nw1`

| VM | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| Security Onion standalone | 8 | 26 GB | 1 TB | Official minimum 24 GB, 32 recommended. Management on LAB-SOC, three sniffing interfaces on the direct cables and the SPAN |

Without the ZFS cache, Security Onion can have 26 GB instead of 24.

Fallback if RAM is not enough: Eval mode, which needs 8 GB but gives up
Logstash, Redis and full packet capture.

### 5.2 `n100`

| VM or container | vCPU | RAM | Disk | VLAN |
|---|---|---|---|---|
| Lab OPNsense | 2 | 2 GB | 20 GB | WAN and all VLANs, as gateway |
| Wazuh all-in-one | 4 | 6 GB | 100 GB | LAB-SOC |
| Velociraptor server | 1 | 1 GB | 50 GB | LAB-SOC |
| MISP | 2 | 4 GB | 50 GB | LAB-SOC |
| DFIR-IRIS | 2 | 2 GB | 20 GB | LAB-SOC |
| Shuffle | 2 | 4 GB | 30 GB | LAB-SOC |
| Caldera | 1 | 2 GB | 20 GB | LAB-ATTACK |
| Total | | 21 GB | 290 GB | |

Caldera moved here from `nw2` to free RAM on the 24 GB node.

Wazuh drops to 6 GB by reducing the indexer heap. The services in this table
do not all need to run at once from the start: they are turned on phase by
phase.

### 5.3 `nw2`, fixed services

| VM | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| Graylog | 4 | 8 GB | 200 GB | Graylog 7.2, MongoDB and Data Node in the same VM. VLAN LAB-SOC |
| Kali | 2 | 3 GB | 60 GB | Light XFCE desktop |
| Total | | 11 GB | 260 GB | |

Indicative Graylog RAM split: 2 GB for the Graylog server, 4 GB for the Data
Node, 1 GB for MongoDB, 1 GB for the system. Without the ZFS cache Graylog gets
its 8 GB back. If RAM is needed, it can drop to 6 GB with 1 GB for the server
and 3 GB for the Data Node.

Graylog sits on `nw2` because it is the only node with free RAM. It is on the
SOC VLAN even though it shares the hypervisor with the targets. For a lab, the
risk of a VM escape is negligible.

### 5.4 `nw2`, Windows scenario

| VM | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| DC01, Windows Server Core | 2 | 2 GB | 40 GB | Domain controller and DNS, no GUI |
| SRV01, Windows Server Core | 2 | 2 GB | 40 GB | File server and vulnerable services, optional |
| WS01, Windows 11 Enterprise | 2 | 4 GB | 60 GB | Victim user's workstation |
| Total | | 8 GB | 140 GB | |

### 5.5 `nw2`, Linux scenario

| VM | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| LNX01, Ubuntu Server | 2 | 2 GB | 30 GB | Juice Shop and DVWA in containers |
| LNX02, Metasploitable | 1 | 1 GB | 20 GB | Intentionally vulnerable services |
| LNX03, Debian | 1 | 1 GB | 20 GB | "Clean" server with agents, for false positives |
| Total | | 4 GB | 70 GB | |

### 5.6 `nw2` budget

| Configuration | RAM |
|---|---|
| Usable RAM | 22 GB |
| Fixed services and Windows scenario | 19 GB |
| Fixed services and Linux scenario | 15 GB |
| Fixed services and both scenarios | 23 GB, does not fit |
| Both scenarios with Graylog at 6 GB | 21 GB, fits with 1 GB of headroom |

With 24 GB, alternating between the Windows and Linux scenarios remains the
normal way of working. Running both at once is only possible by shrinking
Graylog, and the headroom is too small for continuous use.
The two scenarios should be grouped into Proxmox pools, so each can be started
and stopped with a single command. KSM and ballooning give a few GB of extra
headroom, but they should not be counted in the budget.

### 5.7 Keeping Windows RAM low

- **Server Core** instead of the desktop experience, managed with RSAT from the workstation.
- **VirtIO drivers and ballooning** on every Windows VM.
- **KSM** enabled on `nw2`.
- **Rotating agents.** Sysmon is always on. Wazuh, Elastic Agent, Graylog Sidecar and Velociraptor are enabled depending on the phase, not all at once on the same VM.
- **Unneeded services disabled** in the templates, for example search and telemetry.

## 6. Templates and VM lifecycle

1. Every template is built in the LAB-BUILD VLAN, the only one with updates from outside.
2. The template is updated, cleaned up, generalized with Sysprep on Windows, and converted into a Proxmox template.
3. Target VMs are linked clones of the template, in the isolated VLANs. Linked clones save disk space.
4. Before each exercise, take a "clean" snapshot. After the exercise, roll back to it.
5. A target VM never goes back to the LAB-BUILD VLAN. To update it, rebuild the template.

Licensing: Windows Server evaluation ISOs last 180 days, Windows 11 Enterprise
90 days. Periodic rebuilds must be automated.

Vulnerable domain: built from scratch to understand it well, then populated
with tools such as BadBlood or vulnerable-AD to get realistic weak users,
groups and configurations. GOAD-Mini remains a ready-made alternative.

## 7. Security platform

| Area | Tool | Where |
|---|---|---|
| Network and SIEM | Security Onion: Suricata, Zeek, Strelka, Elastic | `nw1` |
| Log management and SIEM | Graylog Open 7.2 | `nw2` |
| Endpoint and XDR | Wazuh | `n100` |
| Windows telemetry | Sysmon with a reference configuration | Windows targets |
| Live forensics | Velociraptor | `n100` |
| Case management | DFIR-IRIS | `n100` |
| Automation | Shuffle | `n100` |
| Threat intelligence | MISP | `n100` |
| Adversary emulation | Caldera on `n100`, Atomic Red Team on the targets | `n100`, `nw2` |
| Rules | Sigma, converted for Security Onion, Graylog and Wazuh | Repository |

### 7.1 Graylog's role

Security Onion stays the main network tool. Graylog becomes the central log
platform, with these sources:
- **Lab OPNsense** over syslog. It is Graylog's classic use case and works from day one.
- **Windows and Sysmon events** through Graylog Sidecar.
- **Wazuh alerts** forwarded by Wazuh.
- **Linux logs** from the targets through Sidecar or syslog.

What Graylog Open offers, according to the product page:
- search, dashboards, streams and processing pipelines;
- threshold and event-based alerts;
- import of Sigma rules from private Git repositories;
- SSPL license, free, with no volume limits for internal use.

Paid editions keep: anomaly detection, investigations, ready-made detection
content, risk scoring. Sigma support in the Open edition must be checked in
practice in phase 5.

Seeing the same attack in Security Onion and in Graylog gives a direct
comparison between the Elastic stack and the Graylog stack.

## 8. Phases and completion criteria

### Phase 0: inventory
- Disk health and CPU model of the NodeWeavers.
- Address plan checked against production.
- Identify the 32 GB NodeWeaver, which becomes `nw1`.
- Check the notebook's outbound rules on the production OPNsense, see 4.4.
- NodeWeaver IPMI port: dedicated or shared, see 4.2.
- **Done when:** this document has no placeholders left.

### Phase 1: infrastructure
- Proxmox on three nodes, cluster formed with two corosync rings, ext4 storage with LVM-thin, KSM on `nw2`.
- Lab OPNsense on the N100 with WAN in passthrough and outbound NAT, static routes on the router.
- VLANs on the switch, rules from section 4.3, second factor on the web interfaces, emergency port.
- Test of the asymmetric path from the notebook, see 4.1.
- Backups of the SOC VMs.
- **Done when:** a test VM on LAB-TGT-WIN reaches neither the Internet nor the router network. LAB-SOC reaches only the allowlist. The production notebook opens a stable SSH session to `nw1` and the Proxmox web interface.

### Phase 2: templates and targets
- Templates for Windows Server Core, Windows 11, Ubuntu, Debian, Kali.
- `lab.local` domain with DC01 and WS01 joined.
- **Done when:** the Windows scenario starts and stops from its pool and returns to a clean snapshot in a few minutes.

### Phase 3: endpoint visibility
- Sysmon and the Wazuh agent on the targets.
- **Done when:** an Atomic Red Team run, for example a credential dump, raises a Wazuh alert with the correct ATT&CK technique.

### Phase 4: network visibility
- 4x 1 GbE card installed in `nw1`.
- Security Onion installed, OVS mirrors on `nw2` and on the N100, direct cables in place.
- Suricata on the lab OPNsense WAN.
- **Done when:** an Nmap scan from Kali shows up in Zeek and raises Suricata alerts. Traffic between two VMs on the same node is visible. A package update from LAB-SOC shows up in Zeek, and the same flow between Kali and a target does not appear twice.

### Phase 5: log management with Graylog
- Graylog with Data Node and MongoDB.
- Ingest the lab OPNsense logs first, then Sidecar on the targets, then Wazuh alerts.
- Pipelines to normalize fields, dashboards for the firewall and Windows events.
- Test the Sigma rule import.
- **Done when:** the phase 3 alert is visible in Graylog together with the firewall logs from the same moment.

### Phase 6: attacks
- Full chain from Kali: reconnaissance, initial access, lateral movement, domain compromise.
- The same techniques replayed repeatably with Caldera.
- **Done when:** there is a table with every technique executed and where it was detected or missed, in Wazuh, Security Onion and Graylog.

### Phase 7: detection engineering
- Purple team loop: attack, check, Sigma rule, attack again.
- The same Sigma rules converted for Security Onion and Graylog, with the results compared.
- **Done when:** the techniques missed in phase 6 have a tested rule in the repository, on at least one of the two SIEMs.

### Phase 8: incident response
- Velociraptor for forensic collection, IRIS for cases.
- Shuffle: alert, enrichment with MISP, case opened in IRIS.
- **Done when:** a real lab alert opens an enriched case on its own.

### Phase 9: extensions
- Chosen based on what came out of the earlier phases.
- Candidates: OpenCanary for internal deception, OpenCTI, NetFlow from the OPNsense to Graylog.

## 9. Automation

New repository, for example `ansible-lab-security`, following the conventions
of the production repositories:
- a single playbook with tags per area;
- inventory and group variables for Proxmox nodes, SOC services and targets;
- encrypted vault with a pre-commit hook;
- documentation and a verification file.

VM creation: OpenTofu with the `bpg/proxmox` provider, or the Ansible Proxmox
modules. To be decided in phase 1.
Windows target configuration: Ansible over WinRM or SSH.
Lab OPNsense configuration: to be decided between the Ansible collection for
OPNsense and a versioned export of the configuration.

## 10. Risks

| Risk | Mitigation |
|---|---|
| Malicious traffic leaking to the Internet | No outbound access for targets and attack, tested in phase 1 |
| Lab host exploring the router network or the production WAN | Explicit rule on the lab OPNsense, see 4.3 |
| Router dropping the asymmetric traffic from the notebook | Static route on the production OPNsense, or the emergency port |
| Lab reachable from any production host, because of NAT | Rule allowing only the production WAN, YubiKey and second factor |
| N100 down: lab without network or quorum | Accepted for a lab |
| Compromised agents attacking the SOC | Only agent ports open towards LAB-SOC |
| Not enough RAM on `nw1` | Fall back to Security Onion Eval |
| Tight RAM on `nw2`, the 24 GB node | Alternate the scenarios, Caldera on `n100` |
| Disk failure with no replication | Backups of the important VMs on another node's disk or an external disk |
| N100 too slow | Services turned on per phase. `nw2` has no headroom left to take them |
| Windows licenses expiring | Templates rebuilt automatically |
| Management interface exposed on the router network | LAB-MGMT accepts only the production WAN and the emergency port |
