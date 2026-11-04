# Command cheat sheet

[← Back to the homelab overview](README.md)

A quick reference for commands I used in Labs 01–03, grouped by machine. The IP addresses and interface names are from my setup.

## Windows / Active Directory

| Command | Why I used it |
| --- | --- |
| `hostname` | Current computer name |
| `ipconfig /all` | Full interface configuration, DHCP and DNS |
| `ncpa.cpl` | Open adapter configuration |
| `sysdm.cpl` | Computer name/domain settings |
| `dsa.msc` | Active Directory Users and Computers |
| `gpmc.msc` | Group Policy Management |
| `whoami` | Confirm which local/domain user is signed in |
| `gpupdate /force` | Refresh Group Policy |
| `gpresult /r /scope user` | See which user GPOs actually applied |
| `control.exe` | Try opening Control Panel when testing the GPO |

```powershell
# DNS lookups and domain controller diagnostics
Resolve-DnsName dc01.lab.home.arpa -Server 192.168.10.10
Get-DnsServerZone
dcdiag /test:DNS

# Domain join (use an elevated PowerShell window)
Add-Computer -DomainName 'lab.home.arpa' -Credential 'LAB\Administrator' -Restart

# Rename a domain member (when necessary)
Rename-Computer -NewName 'CLIENT01' -DomainCredential 'LAB\Administrator' -Restart

# Confirm membership
(Get-CimInstance Win32_ComputerSystem).Domain
```

**Note:** I needed an elevated PowerShell window for the domain join. Supplying domain administrator credentials to `Add-Computer` wasn't enough on its own.

## Proxmox host

```sh
# Network bridges and VMs
cat /etc/network/interfaces
qm config 102                   # OPNsense VM config
pct config 104                  # IT LXC config
bridge link show master vmbr0   # Interfaces attached to vmbr0
bridge vlan show dev tap102i1   # VLANs admitted to OPNsense net1
bridge vlan show dev nic0 | head -n 20  # Avoid printing 4,000 VLAN IDs

# Snapshots
qm listsnapshot 101
# Only if a snapshot task has stopped and a stale lock is confirmed:
qm unlock 101

# Capture Ethernet on different sides of the bridge
tcpdump -eni nic0 'vlan 20'
tcpdump -eni tap102i1 'vlan 20'
```

`vmbr0` is the physical-backed VLAN-aware bridge in the **final** Lab 03 configuration. `vmbr1` is the Windows domain LAN; `vmbr2` remains isolated OPNsense management.

For the OPNsense VM's second adapter, Proxmox needed tagged trunk VLANs **10 and 20**, not just the default untagged VLAN 1 (`trunks=10;20` in the VM NIC config).

## Aruba 2530 / ArubaOS-Switch

```text
show version                  ← Firmware and boot image
show interfaces brief         ← Link status and speed
show vlans                    ← VLAN inventory
show vlans 10                 ← Ports tagged/untagged in VLAN 10
show vlans 20
show mac-address 4            ← MAC addresses learned on port 4

configure terminal
vlan 10
 name IT
 exit
vlan 20
 name HR
 exit
vlan 10 untagged 3          ← Access-style IT port
vlan 20 untagged 4          ← Access-style HR port
vlan 10 tagged 1            ← Carry VLAN 10 over switch uplink
vlan 20 tagged 1            ← Carry VLAN 20 over same uplink
write memory                ← Save running configuration
```

For the same-subnet physical VLAN test, port 5 was moved between VLANs:

```text
vlan 10 untagged 5          ← Separate Legion from Pi
vlan 20 untagged 5          ← Put Legion back with Pi
```

**Untagged:** no 802.1Q tag on the wire toward a normal endpoint.  
**Tagged:** 802.1Q header identifies the VLAN over a shared link.  
**PVID:** VLAN assigned to incoming untagged frames.

## OPNsense / FreeBSD

```sh
ifconfig -l                       # Interface names
ifconfig vtnet1                   # Check UP / link flags
ifconfig vtnet1 up                # Temporarily enable an interface
ifconfig vlan01                   # VLAN details
netstat -rn -f inet               # IPv4 routing table
tcpdump -eni vtnet0 'vlan and icmp' # Initial virtual VLAN trunk capture
tcpdump -eni vtnet1 'vlan and icmp' # Physical-side trunk capture
```

In FreeBSD's route output:

| Flag / term | Meaning |
| --- | --- |
| `U` | Route up |
| `H` | Single host route |
| `S` | Static route |
| `G` | Next-hop gateway |
| `lo0` | Local loopback interface |
| `link#N` | Kernel interface index, not a hop count or VLAN ID |

**Routing versus filtering:** A route says where traffic could go. A firewall pass rule says whether the packet is allowed to go there. In this lab, IT-originated ICMP to HR was allowed; unsolicited HR-originated ICMP to IT was not.

## Linux, Raspberry Pi and LXC

```sh
ip -br addr                             # Short view of all interfaces
ip -4 addr show eth0                   # IPv4 addresses
ip route                               # Routes
ip route get 192.168.20.10            # Selected path and source address
ip neigh show 192.168.30.1            # ARP/neighbor entry
ping -c 4 192.168.30.1                 # Four pings
ssh rahyan@192.168.30.20               # Pi over IPv4
exit                                   # Leave SSH session

# Temporary only — lost on reconfiguration/reboot
sudo ip addr add 192.168.50.20/24 dev eth0
sudo ip addr del 192.168.50.20/24 dev eth0

# Inspect NetworkManager
nmcli connection show --active
nmcli device status
journalctl -u NetworkManager -b --no-pager -n 50

# Raspberry Pi power and clock checks
vcgencmd get_throttled
vcgencmd measure_clock arm
```

Persistent Pi address and the route back to IT:

```sh
sudo nmcli con mod 'Wired connection 1' \
  ipv4.method manual \
  ipv4.addresses '192.168.30.20/24' \
  ipv4.gateway '' \
  ipv4.routes '192.168.20.0/24 192.168.30.1'
sudo nmcli con up 'Wired connection 1'
```

**Note:** `ip addr add` only updates the running configuration. I used `nmcli` for the Pi's saved static address and route.

## tcpdump and Wireshark

```sh
# -e: Ethernet headers (MACs, tagging)
# -n: numeric addresses, no reverse lookup
# -i: capture interface
# 'vlan and icmp': capture tagged ICMP packets

tcpdump -eni vtnet0 'vlan and icmp'
tcpdump -eni vtnet1 'vlan 20'
tcpdump -eni vtnet0 -w lab03.pcap 'vlan and icmp'
```

Useful **Wireshark display filters** (different syntax from tcpdump capture filters):

```text
arp
vlan.id == 20
eth.addr == dc:a6:32:a9:23:fd
icmpv6.type == 135 || icmpv6.type == 136
```

The last filter shows IPv6 Neighbor Solicitation (135) and Neighbor Advertisement (136). IPv6 uses Neighbor Discovery rather than ARP.

## Quick troubleshooting order

1. **Physical / virtual link:** cable, negotiated link, NIC administratively UP.
2. **VLAN membership:** access port vs trunk, Proxmox bridge port VLANs, guest VLAN parent.
3. **Addressing:** correct subnet, DHCP/static configuration, default gateway.
4. **Neighbor resolution:** ARP for IPv4, NDP for IPv6.
5. **Routes:** connected routes and return paths.
6. **Firewall:** rules on the ingress interface, state table and live logs.
7. **Capture:** watch at the exact point where you think traffic is being lost.

I also managed to forget **Apply** once while moving a VLAN interface in OPNsense. Worth checking before troubleshooting anything else.
