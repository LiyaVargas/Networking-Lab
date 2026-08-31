# Day 7 — Review Lab

## What I Learned
- DHCP requests go out as broadcasts. A broadcast never crosses a router unless you either put the DHCP server on the same subnet as the client, or configure something like `ip helper-address` to relay it. Learned this the hard way while figuring out where to place the DHCP server in this topology.
- OSPF doesn't care how many "layers" are between two networks — as long as both routers advertise their connected networks and form a neighbor adjacency, routing just works. Same wildcard mask logic as Day 4.
- A single VLAN on a switch is still worth configuring explicitly instead of leaving everything on VLAN 1 — keeps traffic organized even when there's only one group of devices.
- DNS just needs an A record pointing a name to an IP. Once the client has the DNS server's address (handed out via DHCP), name resolution works automatically.
- Every layer in this topology depends on the one below it. If VLANs are wrong, nothing above it works. If OSPF isn't up, DHCP and DNS traffic has nowhere to go even if configured perfectly.

## Lab Goal
Build a review topology tying together VLANs, OSPF, DHCP, and DNS: PC0 connects through a switch to two OSPF-routed routers, with a server on the far end providing DNS. By the end, PC0 should get an IP automatically, resolve the server's name, and reach it.

## Topology

![Topology](images/Day7%20images/day07-topology.png)

| Device | Interface | IP | Subnet |
|--------|-----------|-----|--------|
| PC0 | NIC | DHCP (got 192.168.1.2) | /24 |
| Switch0 | Fa0/1, Fa0/2 | VLAN 10 (Users) | — |
| R0 | G0/0 | 192.168.1.1 | /24 |
| R0 | G0/1 | 10.0.0.1 | /30 |
| R1 | G0/0 | 10.0.0.2 | /30 |
| R1 | G0/1 | 192.168.2.1 | /24 |
| Server0 | FastEthernet0 | 192.168.2.10 | /24 |

## What I Configured

VLAN on Switch0:
```
vlan 10
name Users
exit
interface fastEthernet0/1
switchport mode access
switchport access vlan 10
exit
interface fastEthernet0/2
switchport mode access
switchport access vlan 10
exit
```

Router interfaces:

R0:
```
interface gigabitEthernet0/0
ip address 192.168.1.1 255.255.255.0
no shutdown
exit
interface gigabitEthernet0/1
ip address 10.0.0.1 255.255.255.252
no shutdown
exit
```

R1:
```
interface gigabitEthernet0/0
ip address 10.0.0.2 255.255.255.252
no shutdown
exit
interface gigabitEthernet0/1
ip address 192.168.2.1 255.255.255.0
no shutdown
exit
```

OSPF on both routers:

R0:
```
router ospf 1
network 192.168.1.0 0.0.0.255 area 0
network 10.0.0.0 0.0.0.3 area 0
```

R1:
```
router ospf 1
network 10.0.0.0 0.0.0.3 area 0
network 192.168.2.0 0.0.0.255 area 0
```

DHCP pool on R0:
```
ip dhcp excluded-address 192.168.1.1
ip dhcp pool PC_NETWORK
network 192.168.1.0 255.255.255.0
default-router 192.168.1.1
dns-server 192.168.2.10
```

Server0 static IP: `192.168.2.10 /24`, gateway `192.168.2.1`, DNS `192.168.2.10`.

DNS service on Server0: A record `server.local` → `192.168.2.10`.

## What I Verified

### VLAN on Switch0
![VLAN Brief](images/Day7%20images/day07-show-vlan-brief.png)

VLAN 10 active with Fa0/1 and Fa0/2 both assigned.

### OSPF Neighbor
![OSPF Neighbor](images/Day7%20images/day07-ospf-neighbor.png)

R0 and R1 show FULL adjacency over the 10.0.0.0/30 link.

### PC0 DHCP Lease
![PC0 DHCP](images/Day7%20images/day07-pc0-dhcp.png)

PC0 received 192.168.1.2, gateway 192.168.1.1, and DNS 192.168.2.10 — all handed out automatically.

### Ping to Server
![Ping Server](images/Day7%20images/day07-ping-server.png)

PC0 reached the server at 192.168.2.10 with 0% loss, confirming OSPF routing across both routers works end to end.

### DNS Resolution
![DNS Resolution](images/Day7%20images/day07-dns-resolution.png)

PC0 resolved `server.local` to 192.168.2.10 and reached it by name.

## Practice Checklist
- [x] Create a VLAN and assign access ports
- [x] Configure router interfaces and bring them up
- [x] Configure OSPF and confirm FULL neighbor adjacency
- [x] Configure a DHCP pool for the PC's subnet
- [x] Configure DNS service with an A record
- [x] Confirm PC0 receives an IP automatically via DHCP
- [x] Resolve the server by name and reach it by ping