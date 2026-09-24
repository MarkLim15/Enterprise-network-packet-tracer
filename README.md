# Enterprise Network Infrastructure Lab

This is a small enterprise network I designed and built in **Cisco Packet Tracer** while studying networking and preparing for the CCNA.

Instead of creating a separate lab for every topic, I built one network that I could continue expanding and troubleshooting. The current version includes VLAN segmentation, redundant Layer 2 links, a four-router OSPF Area 0 topology, a branch LAN, centralized network services, access control, NAT/PAT, and a simulated ISP/Internet connection.

## Network Topology

![Enterprise Network Topology](TOPOLOGY.png)

The main office LAN is connected to **R1**, which performs Router-on-a-Stick inter-VLAN routing. R1, R2, R3, and R4 form the internal **OSPF Area 0** routed topology. **R2** acts as the WAN/Internet edge router and connects the internal network to the simulated ISP.

## What I Implemented

- VLAN segmentation and 802.1Q trunking
- Router-on-a-Stick (ROAS) and inter-VLAN routing
- Four-router single-area OSPF (Area 0)
- OSPF route advertisement for the main-office VLANs
- OSPF default-route propagation from the edge router
- Static default route from R2 toward the ISP
- DHCP and DHCP relay using `ip helper-address`
- DNS and internal HTTP services
- NAT/PAT on R2 for simulated Internet access
- Extended ACL for Guest VLAN isolation
- SSH remote management
- Spanning Tree Protocol (STP)
- LACP EtherChannel
- Redundant Layer 2 switch path
- Branch network connectivity
- Simulated ISP/Internet connectivity

## VLAN Design

| VLAN | Purpose | Network | Gateway |
|---|---|---|---|
| 10 | Users | 192.168.10.0/24 | 192.168.10.1 |
| 20 | Servers | 192.168.20.0/24 | 192.168.20.1 |
| 30 | Management | 192.168.30.0/24 | 192.168.30.1 |
| 40 | Guest | 192.168.40.0/24 | 192.168.40.1 |
| 50 | IoT | 192.168.50.0/24 | 192.168.50.1 |

## Routed Networks

| Connection | Network |
|---|---|
| R1 - R2 | 10.0.0.0/30 |
| R2 - ISP | 10.0.0.4/30 |
| R2 - R3 | 10.0.0.8/30 |
| R3 - R4 | 10.0.0.12/30 |
| R4 - R1 | 10.0.0.16/30 |
| Branch LAN | 192.168.60.0/24 |
| Simulated Internet | 203.0.113.0/24 |

## Routing Design

**R1** is the main-office inter-VLAN router. Its 802.1Q subinterfaces provide the default gateways for VLANs 10, 20, 30, 40, and 50. These VLAN networks are advertised into OSPF Area 0.

**R1, R2, R3, and R4** form a routed OSPF topology. The additional R3 and R4 paths give the lab a topology where I can practice OSPF neighbor relationships, route learning, path selection, and future failover/convergence scenarios.

**R2** is the Internet edge router. It has a static default route toward the ISP at `10.0.0.6` and uses `default-information originate` to advertise a default route to the internal OSPF domain.

## Switching Design

**SW1** is the central switch and is configured as the STP root bridge.

Two LACP EtherChannels connect SW1 to the access switches:

- Po1: SW1 - SW2
- Po2: SW1 - SW3

A separate SW2-SW3 trunk provides an additional Layer 2 path. STP prevents a switching loop by placing the redundant path into an alternate/blocking state when appropriate.

![EtherChannel Verification](ETHERNETCHANNEL.png)

![STP Verification](STP.png)

## Network Services

**Server1** uses the static address `192.168.20.10` and provides DHCP, DNS, and HTTP services.

R1 uses DHCP relay on the required client VLAN subinterfaces so DHCP broadcasts can reach the centralized server.

The internal DNS server resolves:

`www.company.local` → `192.168.20.10`

![DHCP Test](DHCP_TEST.png)

![DNS Test](DNS_TEST.png)

## Security

An extended ACL on R1 prevents the Guest VLAN (VLAN 40) from accessing the internal Server VLAN (VLAN 20) while still allowing the Guest network to reach the simulated Internet.

SSH is also configured on the network devices for remote CLI management.

![ACL Test](ACL_TEST.png)

## NAT/PAT and Simulated Internet

R2 performs PAT for traffic leaving the internal network toward the simulated ISP. Internal source addresses are translated to R2's ISP-facing address, `10.0.0.5`.

The simulated Internet server is `203.0.113.10`.

End-to-end testing confirmed that devices from the main office, branch network, and internal routed topology can reach the simulated Internet after routing and PAT are applied.

![NAT/PAT Verification](NAT-TEST.png)

## Verification

I verify each part of the network with Cisco IOS show commands and end-to-end tests rather than assuming a configuration works.

Examples include:

```text
show ip ospf neighbor
show ip route ospf
show ip route 0.0.0.0
show ip nat translations
show ip nat statistics
show etherchannel summary
show spanning-tree
show interfaces trunk
```

During OSPF verification, R2 formed FULL adjacencies with its internal OSPF neighbors and dynamically learned the VLAN networks behind R1.

## Troubleshooting Notes

A major goal of this project is to practice troubleshooting, not only configuration. I document the symptom, what I checked, the root cause, the fix, and how I verified recovery.

### Incident 1 - Remote Routers Could Not Reach the Main-Office VLANs

**Symptom:** R2, R3, and R4 could participate in OSPF, but networks behind R1 were not reachable from the other routers.

**What I checked:** I used `show ip route ospf` on R2 and noticed that the R1 VLAN networks were missing from the OSPF routing table.

**Root cause:** The ROAS subinterfaces on R1 were not participating in OSPF, so the connected VLAN networks were not being advertised to the other OSPF routers.

**Resolution:** I enabled OSPF Area 0 on the appropriate R1 subinterfaces using `ip ospf 1 area 0`.

**Verification:** R2 then learned `192.168.10.0/24`, `192.168.20.0/24`, `192.168.30.0/24`, `192.168.40.0/24`, and `192.168.50.0/24` as OSPF routes. R2, R3, and R4 could then reach the networks behind R1.

### Incident 2 - R2 Could Reach the Internet Server but Internal Devices Could Not

**Symptom:** R2 could successfully ping `203.0.113.10`, but R1, R3, R4, and internal hosts initially could not.

**Troubleshooting approach:** I first finished troubleshooting OSPF so internal routing was known to be working. I then verified that R2 could reach the ISP and the simulated Internet server. This narrowed the remaining problem to the edge/return path rather than the internal OSPF network.

**Root cause:** The NAT/PAT configuration on R2 was missing after R2's configuration had previously been lost while the lab was being modified.

**Resolution:** I defined R2's internal-facing interfaces as NAT inside, the ISP-facing G0/2 interface as NAT outside, and configured PAT using the G0/2 address.

**Verification:** `show ip nat translations` showed PC1's inside-local address `192.168.10.101` translated to the inside-global address `10.0.0.5`. PC1 and the other tested internal devices were then able to ping `203.0.113.10`.

### Incident 3 - Configuration Loss During Hardware Expansion

While adding a serial module to expand the routed topology, R2 lost its running configuration because no startup configuration had been saved before the device was powered down.

I rebuilt R2, restored routing in stages, and verified connectivity before moving to the next feature. After recovery, I saved the working configuration on all network devices using `copy running-config startup-config`.

This incident reinforced the importance of saving a known-good configuration before hardware or topology changes.

## Packet Tracer File

The working network is included in this repository as a Cisco Packet Tracer Activity:

`enterprise network by mark lim.pka`

The activity is intended for viewing and testing the network. Selected topology and configuration options are restricted while CLI access remains available so configurations and show commands can be inspected.

My original `.pkt` development file is kept separately as the master copy.

## What I Learned

This project helped me connect topics that felt separate when I first studied them. VLANs, trunking, ROAS, DHCP relay, OSPF, ACLs, NAT/PAT, STP, and EtherChannel all affect the same packet flow once they are placed in one network.

The troubleshooting process was especially useful. Instead of immediately changing configurations, I practiced narrowing the scope: checking whether routes existed, testing the next hop, separating internal routing problems from Internet-edge problems, and using show commands to verify the result.

## Next Steps

I plan to continue using this topology as a troubleshooting lab. Possible next exercises include intentionally breaking one feature at a time, testing OSPF convergence when a routed link fails, comparing alternate OSPF paths, and documenting additional failure/recovery scenarios.

## Author

**Mark Anthony Lim**

Computer Engineering graduate building hands-on experience in networking, IT infrastructure, and technical support.
