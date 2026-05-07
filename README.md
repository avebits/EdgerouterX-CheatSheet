# EdgerouterX-CheatSheet

Cheatcheet for cli setup of the infamous Ubiquiti Edgerouter X

## ToDo
- Clean up this document
- Add OpenVPN/WireGuard server/klient
- Segment on vyos.io ?
- ???

# Topics
- Basics
- Advanced
- Full VLAN+DHCP setup

WIP document notes:
- P0   == Admin
- P1-3 == LAN
- P4   == WAN

→ 192.168.1.1/24 on Port 0. 

→ ssh username@192.168.1.1

→ shared-network-name = DHCP scope container → DHCP configuration grouping (service layer)

→ vif = VLAN interface

→ "Traffic tagged with VLAN 10 arrives → goes to this interface"

vif 10              → VLAN ID
192.168.10.0/24     → subnet
LAN10               → DHCP name

## Basics
### Getting info before entering configure mode:
```
show interfaces
```

### Enter configure mode: 
```
configure
```

### Show interfaces
```
show interfaces
show interfaces ethernet eth4
```

### Quick setup:
```
set system host-name <hostname>
set system domain-name <name.domain>
set system name-server 1.1.1.1
set system time-zone Europe/Oslo

set interfaces ethernet eth4 address dhcp
set interfaces ethernet eth4 description "WAN"
set interfaces ethernet eth3 address 10.112.1.1/24
set interfaces ethernet eth3 description "LAN"

set service nat rule 5000 outbound-interface eth4
set service nat rule 5000 type masquerade
```

### Enable/Disable ports:
```
set interfaces ethernet eth2 enable
set interfaces ethernet eth3 disable
```

### LAN (eth1-eth3) as a switch (sw0), with gateway IP 192.168.10.1/24:

This will be the layout:
```
eth1 ─┐
eth2 ─┼── switch0 (192.168.10.1/24) → router |switch0 = gateway interface|
eth3 ─┘
```

```
set interfaces switch switch0
set interfaces switch switch0 description 'LAN'
set interfaces switch switch0 address 192.168.10.1/24
set interfaces switch switch0 mtu 1500
set interfaces switch switch0 switch-port interface eth1
set interfaces switch switch0 switch-port interface eth2
set interfaces switch switch0 switch-port interface eth3
```

### WAN (eth4): DHCP client:
```
set interfaces ethernet eth4 description 'WAN'
set interfaces ethernet eth4 duplex auto
set interfaces ethernet eth4 speed auto
set interfaces ethernet eth4 address dhcp
```

### verify IP was acquired from DHCP:
```
show dhcp client leases
```

### Example LAN (eth1): static gateway IP:
```
set interfaces ethernet eth1 description 'LAN10'
set interfaces ethernet eth1 address 192.168.10.1/24
```

### Commit, saving and exit:
```
commit
save
exit
```

As it is based on Debian Linux it supports shortcuts like:
```
commit ;save
```
```
commit && save
```
```
command | command
```

## Advanced

### MAC - IP binding
```
set service dhcp-server shared-network-name LAN10 subnet 192.168.10.0/24 static-mapping printer mac-address aa:bb:cc:dd:ee:ff
set service dhcp-server shared-network-name LAN10 subnet 192.168.10.0/24 static-mapping printer ip-address 192.168.10.10
```

### DHCP server on LAN (ex.):
```
set service dhcp-server disabled false
set service dhcp-server shared-network-name LAN authoritative enable
set service dhcp-server shared-network-name LAN subnet 192.168.10.0/24 default-router 192.168.2.2
set service dhcp-server shared-network-name LAN subnet 192.168.10.0/24 dns-server 192.168.2.3
set service dhcp-server shared-network-name LAN subnet 192.168.10.0/24 lease 86400
set service dhcp-server shared-network-name LAN subnet 192.168.10.0/24 range 0 start 192.168.10.10
set service dhcp-server shared-network-name LAN subnet 192.168.10.0/24 range 0 stop 192.168.10.100

set service dhcp-server shared-network-name LAN subnet 192.168.10.0/24 range 1 start 192.168.10.105
set service dhcp-server shared-network-name LAN subnet 192.168.10.0/24 range 1 stop 192.168.10.150

set service dhcp-server shared-network-name LAN subnet 192.168.10.0/24 range 2 start 192.168.10.200
set service dhcp-server shared-network-name LAN subnet 192.168.10.0/24 range 2 stop 192.168.10.240
etc.
```

### Network segmenting - Adding VLAN(s)
## Example vlan
```
set interfaces ethernet eth2 address 192.168.10.1/24
set service dhcp-server shared-network-name vlan10 subnet 192.168.10.1/24 default-router 192.168.10.1
set service dhcp-server shared-network-name vlan10 subnet 192.168.10.1/24 dns-server 192.168.10.1
set service dhcp-server shared-network-name vlan10 subnet 192.168.10.1/24 start 192.168.10.10 stop 192.168.10.100
set service dns forwarding listen-on eth2

set interfaces switch switch0 vlan-aware enable
```

### Removing DCHP service
```
delete service dhcp-server shared-network-name LAN10
delete interfaces ethernet eth3 address dhcp
```

### DHCP relay (use instead of DHCP server when an external DHCP: server is present)
### Remove/disable DHCP server for that LAN if previously enabled
### Relay requests arriving on LAN to upstream DHCP server
```
set service dhcp-relay interface eth1
set service dhcp-relay server 192.168.20.2
set service dhcp-relay relay-options hop-count 10
set service dhcp-relay relay-options max-size 576
```

### DNS forwarding (router as caching DNS for clients)
### Enable DNS forwarding (dnsmasq) and listen on LAN interface:
```set service dns forwarding cache-size 150```
```set service dns forwarding listen-on eth1```
### Forward to upstream resolvers (examples: Cloudflare and ....):
```set service dns forwarding name-server 1.1.1.1```
```set service dns forwarding name-server 8.8.8.8```
### Forward local hostnames:
```set service dns forwarding system```
### Setting dnsmasq:
```set service dhcp-server use-dnsmasq enable```
```set service dhcp-server shared-network-name LAN10 subnet 192.168.10.0/24 domain-name ubnt.local```

### Setting a masquerade NAT Rule to translate all internal traffic thru eth4:
```
set service nat rule 5000 description "WAN masquerade"
set service nat rule 5000 log disable
set service nat rule 5000 protocol all
set service nat rule 5000 outbound-interface eth4
set service nat rule 5000 type masquerade
```

### Minimal WAN firewall policy:
```
set firewall name WAN_IN default-action drop

set firewall name WAN_IN rule 10 action accept
set firewall name WAN_IN rule 10 state established enable
set firewall name WAN_IN rule 10 state related enable

set interfaces ethernet eth0 firewall in name WAN_IN
```

### Explicitly drop ICMP echo requests in to Eth4
```
set firewall name WAN_IN rule 20 action drop
set firewall name WAN_IN rule 20 protocol icmp
set firewall name WAN_IN rule 20 icmp type echo-request
```

### Allowing ping from a specific IP
```
set firewall name WAN_IN rule 15 action accept
set firewall name WAN_IN rule 15 protocol icmp
set firewall name WAN_IN rule 15 icmp type echo-request
set firewall name WAN_IN rule 15 source address <ip>
```

### Hair-pin NAT, allowing devices on the same internal network to access other internal devices using the network's external IP address, useful when internal clients need to access services hosted on internal servers via their public domain names or external IPs:
```set port-forward hairpin-nat enable```

### Keeping the last 10 configuration commit revisions (by default none is kept):
```set system config-management commit-revisions 10```

### Rollback to specficic revision:
```rollback ? # lists commits```
```rollback {NUM}```

### Secure admin:
```set system login user admin authentication plaintext-password 'STRONG_PASSWORD'```
```delete system login user ubnt```

### Restrict admin management (SSH and WebUI)
```
set service ssh listen-address 192.168.10.1/24
set service gui listen-address 192.168.10.1/24
```

### Enable hardware offload for performance gain:
```set system offload hwnat enable```
```set system offload ipsec enable```
https://help.uisp.com/hc/en-us/articles/22591077433879-EdgeRouter-Hardware-Offloading

### Back up the config after a commit via SSH/SCP
```set system config-management commit-archive location scp://user:pass@hostname/Some/Path/To/Backups```

### Disable SIP ALG: 
```set system conntrack modules sip disable```

### Restart crashed/hanging web gui
1. Kill lighttpd process manually first, otherwise the delete delete service command hangs
```pkill -9 -f lighttpd```
2. Delete and re-add the GUI service
```
configure
delete service gui
commit
set service gui
commit
```

## Update the firmware:

### show version information
```
show version
```
### show storage information
```
show system storage
show system image storage
```
### show installed firmware images
```
show system image
```
### remove old system image (free up some space) 
```
delete system image
```
### download new firmware image
```
add system image https://dl.ubnt.com/firmwares/edgemax/v1.10.x/ER-exxx.xxxxxx.tar
```
### set default boot image, if required
```
set system image default-boot
```


## Example full VLAN+DHCP setup

### Setting switch and vlan interface:
```
set interfaces switch switch0
set interfaces switch switch0 switch-port interface eth1
set interfaces switch switch0 switch-port interface eth2
set interfaces switch switch0 switch-port interface eth3
```
```
set interfaces switch switch0 vif 10 address 192.168.10.1/24
set interfaces switch switch0 vif 20 address 192.168.20.1/24
set interfaces switch switch0 vif 30 address 192.168.30.1/24
```

### Or, set vlan per access port:

This will be the layout now with VLANs:
```
--> switch0
 ├── vif10 VLAN10 → 192.168.10.0/24 (LAN)
 ├── vif20 VLAN20 → 192.168.20.0/24 (IoT)
 └── vif30 VLAN30 → 192.168.30.0/24 (Guest)
```

switch0.10 (vif 10) → 192.168.10.1 (gateway)
switch0.20 (vif 20) → 192.168.20.1 (gateway)
switch0.30 (vif 30) → 192.168.30.1 (gateway)

Key takeaway here:
Without VLANs → switch0 has the IP
With VLANs → each vif has its own IP

```
set interfaces switch switch0 switch-port interface eth1 vlan pvid 10
set interfaces switch switch0 switch-port interface eth1 vlan vid 10
```
VLAN20 - IOT:
```
set interfaces switch switch0 switch-port interface eth2 vlan pvid 20
set interfaces switch switch0 switch-port interface eth2 vlan vid 20
```
VLAN30 - Guests:
```
set interfaces switch switch0 switch-port interface eth3 vlan pvid 30
set interfaces switch switch0 switch-port interface eth3 vlan vid 30
```
(Device → untagged traffic → router tags it with PVID → enters correct VLAN)

Then trunking it:
```
set interfaces switch switch0 switch-port interface eth4 vlan vid 10
set interfaces switch switch0 switch-port interface eth4 vlan vid 20
set interfaces switch switch0 switch-port interface eth4 vlan vid 30
```
(native VLAN)
```
set interfaces switch switch0 switch-port interface eth4 vlan pvid 10
```

## Remember
*1 If switch0 exists with a single gateway address - ADD VLAN first! 
*2 Then move ports into VLAN 10:
```
set interfaces switch switch0 vlan-aware enable
set interfaces switch switch0 switch-port interface eth1 vlan pvid 10
set interfaces switch switch0 switch-port interface eth1 vlan vid 10
```
(Do this for the port you’re connected on first, otherwise you’ll lock yourself out)
*3 Then remove switch0's gateway. 
```
delete interfaces switch switch0 address 192.168.10.1/24
```
*4 Then commit with a timer, 5 minutes
```
commit-confirm 5
```

### --> NAT
set service nat rule 5000 outbound-interface eth0
set service nat rule 5000 type masquerade

### Main VLAN10 dhcp
```
set service dhcp-server shared-network-name LAN10 subnet 192.168.10.0/24 default-router 192.168.10.1
set service dhcp-server shared-nework-name LAN10 subnet 192.168.10.0/24 dns-server 192.168.10.1

set service dhcp-server shared-network-name LAN10 subnet 192.168.10.0/24 range 0 start 192.168.10.50
set service dhcp-server shared-network-name LAN10 subnet 192.168.10.0/24 range 0 stop 192.168.10.200
```
### IOT VLAN20 dhcp
```
set service dhcp-server shared-network-name IOT subnet 192.168.20.0/24 default-router 192.168.20.1
set service dhcp-server shared-network-name IOT subnet 192.168.20.0/24 dns-server 192.168.20.1

set service dhcp-server shared-network-name IOT subnet 192.168.20.0/24 range 0 start 192.168.20.50
set service dhcp-server shared-network-name IOT subnet 192.168.20.0/24 range 0 stop 192.168.20.150
```
### GUEST VLAN30 dhcp
```
set service dhcp-server shared-network-name GUEST subnet 192.168.30.0/24 default-router 192.168.30.1
set service dhcp-server shared-network-name GUEST subnet 192.168.30.0/24 dns-server 192.168.30.1

set service dhcp-server shared-network-name GUEST subnet 192.168.30.0/24 range 0 start 192.168.30.100
set service dhcp-server shared-network-name GUEST subnet 192.168.30.0/24 range 0 stop 192.168.30.200
```

### VLAN Firewall example:
LAN → allow everything
IoT → limited access
Guest → internet only

### Trusted main lan:
```
set firewall name LAN_IN default-action accept
set interfaces switch switch0 vif 10 firewall in name LAN_IN
```
### IOT:
```
set firewall name IOT_IN default-action drop
```
- allow established/related
```
set firewall name IOT_IN rule 10 action accept
set firewall name IOT_IN rule 10 state established enable
set firewall name IOT_IN rule 10 state related enable
```
- allow DNS + DHCP
```
set firewall name IOT_IN rule 20 action accept
set firewall name IOT_IN rule 20 protocol udp
set firewall name IOT_IN rule 20 destination port 53,67,68
```
- allow internet
```
set firewall name IOT_IN rule 30 action accept
set firewall name IOT_IN rule 30 destination address 0.0.0.0/0
```
- block access to LAN
```
set firewall name IOT_IN rule 40 action drop
set firewall name IOT_IN rule 40 destination address 192.168.10.0/24
```
- Last line for IOT:
```
set interfaces switch switch0 vif 20 firewall in name IOT_IN
```

And for guests:
```
set firewall name GUEST_IN default-action drop
```
- allow established/related
```
set firewall name GUEST_IN rule 10 action accept
set firewall name GUEST_IN rule 10 state established enable
set firewall name GUEST_IN rule 10 state related enable
```
- allow DNS + DHCP
```
set firewall name GUEST_IN rule 20 action accept
set firewall name GUEST_IN rule 20 protocol udp
set firewall name GUEST_IN rule 20 destination port 53,67,68
```
- allow internet only
```
set firewall name GUEST_IN rule 30 action accept
set firewall name GUEST_IN rule 30 destination address 0.0.0.0/0
```
- Last line for Guest:
```
set interfaces switch switch0 vif 30 firewall in name GUEST_IN
```


## Wireguard / OpenVPN setup 

## VyOS
