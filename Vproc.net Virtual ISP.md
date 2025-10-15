*Moving from CentOS/Rocky to Debian 13 Stable “Trixie” (VirtualBox Simulation)*

| Type           | Specs (Cores, RAM, Storage) | Role Summary |     |                                   |
| -------------- | --------------------------- | ------------ | --- | --------------------------------- |
| Vproc-Core-R1  | 2                           | 1            | 20  | Core Router                       |
| Vproc-Edge-R1  | 1                           | 1            | 20  | edge router serving Customer 1    |
| Vproc-Edge-R2  | 1                           | 1            | 20  | edge router serving Customer 2    |
| Vproc-Services | 2                           | 2            | 25  | Services DNS/DHCP/Monitoring etc. |
| Vproc-Cust1    | 1                           | 1            | 10  | Customer 1                        |
| Vproc-Cust2    | 1                           | 1            | 10  | Customer 2                        |

---

## Vproc-Core-R1 Router Setup Guide

### VirtualBox Configuration
- Create a new Virtual Machine named `Vproc-Core-R1`.
- Type: Linux, Version: Debian (64-bit).
- CPUs: 2 vCPUs.
- Memory: 1 GB RAM.
- Storage: 20 GB dynamically allocated VDI.
- Network adapters:
  - Adapter 1: Attached to NAT (for simulated Internet access).
  - Adapter 2: Attached to Internal Network named `core-net`.
- System → Motherboard: enable I/O APIC.
- System → Processor: enable PAE/NX.
- Network → Advanced: set Adapter Type to **Paravirtualized Network (Virtio-net).**

### Installation
- Boot from the Debian 13 “Trixie” ISO.
- Use Graphical Install or Install.
- Partitioning: choose Guided – use entire disk.
- Unselect “Debian Desktop Environment.”
- Select SSH server and Standard system utilities only.
- After installation, remove the ISO and reboot.

### Basic Setup

```
sudo apt update && sudo apt upgrade -y  
sudo apt install frr nftables iproute2 tcpdump net-tools -y
```

Confirm both network interfaces exist with IP address.  
Set static IPs in /etc/network/interfaces (or /etc/systemd/network/ if using systemd-networkd).

### Example Config:
```
auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
  address 10.0.0.1
  netmask 255.255.255.0
```
### Enable IP Forwarding
Edit `/etc/sysctl.conf`:
```
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

Apply the changes:
```sudo sysctl -p```

### Configure NAT with `nftables`

Create `/etc/nftables.conf`:
```
table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100;
        oifname "eth0" masquerade
    }
}
```

Enable and start `nftables`:
```
sudo systemctl enable nftables
sudo systemctl start nftables
```

This allows internal hosts on the core-net to reach the Internet through to Vproc-Core-R1.

### Configure `FRRouting` (Basic Example)

Edit `/etc/frr/daemons` and enable the required routing protocols
(set `zebra=yes`, others no for now).

Restart FRR:
```
sudo systemctl enable frr
sudo systemctl restart frr
```
Enter the FRR shell:

`sudo vtysh`

Example configuration:
```
conf t
hostname Vproc-Core-R1
interface eth0
 ip address dhcp
interface eth1
 ip address 10.0.0.1/24
router ospf
 network 10.0.0.0/24 area 0
exit
write
```
### Verification

Check routing table: `ip r`

Verify IP forwarding: 
```cat /proc/sys/net/ipv4/ip_forward```

Test NAT:

From another VM on core-net, ping an Internet address (e.g. 8.8.8.8).

Confirm FRR service status: `sudo systemctl status frr`