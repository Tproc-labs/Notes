*Moving from CentOS/Rocky to Debian 13 Stable “Trixie” (VirtualBox Simulation)*

| Type           | vCores | RAM | Storage (GB) | Role Summary                      |
| -------------- | ------ | --- | ------------ | --------------------------------- |
| Vproc-Core-R1  | 2      | 1   | 20           | Core Router                       |
| Vproc-Edge-R1  | 1      | 1   | 20           | edge router serving Customer 1    |
| Vproc-Edge-R2  | 1      | 1   | 20           | edge router serving Customer 2    |
| Vproc-Services | 2      | 2   | 25           | Services DNS/DHCP/Monitoring etc. |
| Vproc-Cust1    | 1      | 1   | 10           | Customer 1                        |
| Vproc-Cust2    | 1      | 1   | 10           | Customer 2                        |

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

###Note:
	I always add biosdevnames=0 and net.ifnames=0 in the boot parameters to turn enp-blah-blah-blah (Thanks 2015) to eth0, eth1 etc. whoever made that choice really needs a kick in the head another SystemD-runk failure.
### Basic Setup

```bash
apt update && sudo apt upgrade -y  
apt install frr nftables iproute2 tcpdump net-tools -y
```

### Some early security and access
`/etc/ssh/sshd_config`
```
Change:
#PermitRootLogin [Default]
to
PermitRootLogin no

End of file: 
AllowUsers [non-root-user]
```
With your Adapter 1 NAT you can port forward SSH access to the Router.

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
This is the proper way to do it, but lately with Trixie it IGNORES `sysctl.conf` a fix I found to enable forwarding and obey these rules was to put them into `sysctl.d/`
One-Liner:
```bash
echo -e "net.ipv4.ip_forward=1\nnet.ipv6.conf.all.forwarding=1" > /etc/sysctl.d/99-vproc-router.conf
```
OR just put the prior rules into `/etc/sysctl.d/99-[network]-router.conf`

Apply the changes:
```sudo sysctl -p```
I like to reboot at this point. `sync; reboot -h now`
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
```bash
systemctl enable nftables
systemctl start nftables
```

This allows internal hosts on the core-net to reach the Internet through to Vproc-Core-R1.

### Configure `FRRouting` (Basic Example)

Edit `/etc/frr/daemons` and enable the required routing protocols
(set `zebra=yes`and `ospfd=yes`, others no for now).

Restart FRR:
```bash
systemctl enable frr
systemctl restart frr
```
Enter the FRR shell:

`vtysh`

Example configuration:
```
conf t
hostname Vproc-Core-R1
interface eth0
 description Uplink to Internet
interface eth1
 description Core network
 ip address 10.0.0.1/24
router ospf
 network 10.0.0.0/24 area 0
exit
exit
write
exit // leave after writing
```
### Verification

Check routing table: `ip r`

Verify IP forwarding: 
```cat /proc/sys/net/ipv4/ip_forward```
1=Forwarding is working
0=Forwarding is NOT working.
 
# Vproc-Services Setup 
the Vproc-Services machine will provide DHCP, DNS, and NTP services for the internal `core-net`.

---

## 1. VirtualBox Configuration

- **Name:** `Vproc-Services`
- **Type:** Linux Debian (64-bit)
- **vCPU:** 2  
- **Memory:** 2 GB  
- **Storage:** 20 GB (dynamically allocated) - VDI
- **Network Adapter:**
  - Adapter 1 → *Internal Network*: `core-net` 
- **Adapter Type:** Paravirtualized Network (Virtio-net)
- Enable `I/O APIC` and `PAE/NX`

---

## 2. Debian Installation

1. Boot from the Debian 13 “Trixie” ISO  
2. Select **Guided – use entire disk**  
3. Do **not** install a desktop environment  
4. Select:
   - `SSH Server`
   - `Standard System Utilities`
5. Reboot after install

Update the system:
```bash
apt update &&  apt upgrade -y
```

## 3. Network Config
edit `/etc/network/interfaces`
```
auto eth0
iface eth0 inet static
  address 10.0.0.2
  netmask 255.255.255.0
  gateway 10.0.0.1
  dns-nameservers 8.8.8.8
```

Restart networking:
`sudo systemctl restart networking`

Verify and ping router:
```
ip a
ping 10.0.0.1
```

## 4. Install Essential Services
Now that we have a network connection from the router. Let's install essential services.
```
sudo apt install isc-dhcp-server bind9 chrony -y
```

## 5. Configure DHCP
Edit: `/etc/dhcp/dhcpd.conf`

```
option domain-name "[Network.net]";
option domain-name-servers 10.0.0.2, 8.8.8.8;
default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet 10.0.0.0 netmask 255.255.255.0 {
  range 10.0.0.100 10.0.0.200;
  option routers 10.0.0.1;
  option broadcast-address 10.0.0.255;
}

```
Edit: `/etc/default/isc-dhcp-server`
Change: `INTERFACESv4=""` to `INTERFACESv4="eth0"`
This will enable IPv4 while disabling IPv6 otherwise the isc-dhcp will faill to start.

Enable and Start services:
```bash
systemctl enable isc-dhcp-server
systemctl start isc-dhcp-server
systemctl status isc-dhcp-server
```
You should see "active (running)".

### 6. DNS (Bind9/Named) Configuration
Edit `/etc/bind/named.conf.options`:
```
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders { 8.8.8.8; 1.1.1.1; };
    dnssec-validation auto;
    listen-on { any; };
};
```
Restart Bing9:
`sudo systemctl restart named`

## 7. NTP (Chrony) Configuration

Edit `/etc/chrony/chrony.conf`:

```
allow 10.0.0.0/24
```

Enable and restart:
```
sudo systemctl enable chrony
sudo systemctl restart chrony
```

## 8. Verification Steps

#### Router connectivity
`ping 10.0.0.1`
#### DHCP service check:
`journalctl -u isc-dhcp-server | tail -n 10`
#### DNS test:
`dig google.com @10.0.0.2`
#### NTP test:
`chronyc sources`

## 9. Test VM
I created another small Debian Install with a desktop environment when connected to the core-net as the only adapter (NO NAT!) I was giving the IP address 10.0.0.100 as we specified in the DHCP settings and had DNS and IP connectivity to the outside world!
This machine will be my access point from here on out using SSH to access the now headlessly running Router and Services machines.

### 10. Optional DNS
I wanted to make it so my Services machine also has HTTP/web and define my own DNS records.
Example: example.net
Edit: `/etc/bind/named.conf.local`
```
zone "example.net" {
    type master;
    file "/etc/bind/db.example.net";
};

```

create the zone:
`cp /etc/bind/db.local /etc/bind/db.example.net`
edit `/etc/bind/db.vproc.net`
```
$TTL    604800
@       IN      SOA     ns1.example.net. admin.example.net. (
                        3         ; Serial
                        604800    ; Refresh
                        86400     ; Retry
                        2419200   ; Expire
                        604800 )  ; Negative Cache TTL

; Name servers
@       IN      NS      ns1.example.net.

; A Records
ns1     IN      A       10.0.0.2
example.net. IN   A       10.0.0.2
www     IN      A       10.0.0.2
```

#### Verify Configuration Syntax

Run:
```
named-checkconf
named-checkzone vproc.net /etc/bind/db.vproc.net
```

Expected output:
```
zone vproc.net/IN: loaded serial 3
OK
```

#### Restart Bind9
```
systemctl restart bind9
systemctl status bind9
```
#### Test DNS Resolution from Test VM

On the TestVM machine (which gets DNS from 10.0.0.2 via DHCP):
```
dig example.net
dig www.example.net
```

Expected result:
```
;; ANSWER SECTION:
vproc.net.       604800 IN A 10.0.0.2
```

Now we have custom DNS records for our local mock-net
