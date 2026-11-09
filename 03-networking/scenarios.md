# Networking — Scenario-Based Questions

<!-- DEVOPS-SHACK:HEADER -->
<p align="center">
  <a href="https://www.youtube.com/@devopsshack"><img alt="YouTube DevOps Shack" src="https://img.shields.io/badge/YouTube-DevOps%20Shack-FF0000?style=flat-square&logo=youtube&logoColor=white"/></a>
  <a href="https://www.youtube.com/@shackverse"><img alt="YouTube DevOps Shack Hindi" src="https://img.shields.io/badge/Hindi-@shackverse-FF0000?style=flat-square&logo=youtube&logoColor=white"/></a>
  <a href="https://devopsshack.com"><img alt="devopsshack.com" src="https://img.shields.io/badge/Courses-devopsshack.com-C6FF4D?style=flat-square&labelColor=0B0F1A"/></a>
  <a href="https://www.instagram.com/devopsshack"><img alt="Instagram" src="https://img.shields.io/badge/Instagram-@devopsshack-E4405F?style=flat-square&logo=instagram&logoColor=white"/></a>
  <a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack"><img alt="Original repo" src="https://img.shields.io/badge/Original-DevOps%20Shack%20repo-4DE8FF?style=flat-square&logo=github&labelColor=0B0F1A"/></a>
</p>
<p align="center"><sub>Part of <b><a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack">DevOps &amp; Cloud Interview Preparation</a></b> — created by <b><a href="https://www.youtube.com/@devopsshack">DevOps Shack</a></b>. Free to learn from; please credit DevOps Shack when sharing.</sub></p>
<!-- /DEVOPS-SHACK:HEADER -->

> Real-world networking scenarios: what you would do, why, and how to say it in an interview · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Networking](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Addressing, CIDR & Routing](#addressing-cidr--routing) — Q1–Q25
- [VPC & Cloud Network Design](#vpc--cloud-network-design) — Q26–Q50
- [DNS & Service Discovery](#dns--service-discovery) — Q51–Q75
- [Load Balancing, Proxies & CDN](#load-balancing-proxies--cdn) — Q76–Q98
- [TLS, HTTP & Application Layer](#tls-http--application-layer) — Q99–Q120
- [Security: Firewalls, SGs, NACLs & VPN](#security-firewalls-sgs-nacls--vpn) — Q121–Q145
- [Kubernetes Networking](#kubernetes-networking) — Q146–Q169
- [Performance, MTU, Latency & Tools](#performance-mtu-latency--tools) — Q170–Q194
- [Network Operations & Change Management](#network-operations--change-management) — Q195–Q200

---

## Addressing, CIDR & Routing

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. You own 10.0.0.0/16 for a new VPC and need separate /24 subnets for dev, QA and prod across two AZs. How do you carve it up? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic subnetting question. /16 gives 256 possible /24s, so space is not the issue; the issue is leaving room to grow and keeping the plan readable.

**What I'd do →**
1. Give each environment a block of the third octet, for example dev `10.0.0.0/20`, QA `10.0.16.0/20`, prod `10.0.32.0/20`, and leave `10.0.48.0` onward unallocated
2. Inside each /20 use /24s per AZ and tier, for example `10.0.32.0/24` prod-public-a, `10.0.33.0/24` prod-public-b, `10.0.34.0/24` prod-private-a
3. Write the plan in a table in the repo and codify it in Terraform with `cidrsubnet("10.0.0.0/16", 4, 2)`
4. Check with `ipcalc 10.0.32.0/20` that the ranges do not overlap

**Why →** Aligned blocks summarize cleanly in route tables and security rules (one /20 rule per environment instead of 16 /24 rules), and unallocated space lets you add a region or tier later.

**Interview tip →** Say you never carve a VPC "as needed" because ad hoc allocation is what produces overlap when you later peer.

</details>

### Q2. A host is configured as 192.168.1.130/25. What network does it belong to and what is the broadcast address? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A /25 splits the last octet into two halves of 128 addresses: .0-.127 and .128-.255.

**What I'd do →**
1. Block size is 128, and 130 falls in the second block
2. Network is `192.168.1.128`, first usable `.129`, last usable `.254`, broadcast `192.168.1.255`
3. Verify with `ipcalc 192.168.1.130/25` or `python3 -c "import ipaddress as i; print(i.ip_network('192.168.1.130/25', strict=False))"`

**Why →** Network address is the host address ANDed with the mask; doing it by block size is faster than binary in an interview.

**Interview tip →** Mention that cloud providers reserve extra addresses, so usable hosts differ from the textbook 2^n - 2.

</details>

### Q3. How many usable IPs does an AWS subnet of /28 have? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** AWS reserves five addresses in every subnet, which surprises people who only know the textbook formula.

**What I'd do →**
1. A /28 has 16 addresses
2. AWS reserves the first four (network, VPC router, DNS, future use) and the last one (broadcast, unused)
3. Usable is 16 - 5 = 11
4. Confirm with `aws ec2 describe-subnets --subnet-ids subnet-0abc --query 'Subnets[0].AvailableIpAddressCount'`

**Why →** The smallest subnet AWS allows is /28, and small subnets get eaten fast by ENIs from load balancers, Lambda in VPC and EKS nodes.

**Interview tip →** Add that EKS with the VPC CNI can consume dozens of IPs per node, so /28 for anything but a tiny appliance is a trap.

</details>

### Q4. A route table has 10.0.0.0/16 -> local and 10.0.1.0/24 -> vpn-gw. Where does a packet to 10.0.1.5 go? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two routes match the destination and the router must pick one.

**What I'd do →**
1. Apply longest prefix match: /24 is more specific than /16
2. Packet goes to the VPN gateway, not local
3. AWS now accepts routes more specific than the local route (for example to steer a subnet range to a firewall ENI), so check `aws ec2 describe-route-tables` for what is really configured
4. On Linux test with `ip route get 10.0.1.5`

**Why →** Routers always prefer the most specific matching prefix before looking at metrics or administrative distance.

**Interview tip →** Say you read the effective route table rather than assuming, because more-specific-than-local routes are legal in AWS today.

</details>

### Q5. A newly built server can ping everything in its subnet but nothing outside. What is the first thing you check? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same-subnet traffic is L2 and needs no gateway. Anything off-subnet needs a default route and a working gateway.

**What I'd do →**
1. Run `ip route` and confirm there is a `default via <gw>` line
2. Run `ip route get 8.8.8.8` to see the chosen gateway and interface
3. Ping the gateway itself with `ping -c2 <gw>` and check `ip neigh` for a REACHABLE entry
4. In cloud, check the subnet route table has `0.0.0.0/0` to an IGW or NAT gateway

**Why →** Same-subnet works while off-subnet fails almost always means a missing or wrong default gateway or route table entry.

**Interview tip →** Walk the path outward: host, gateway, route table, NAT or IGW, then security controls.

</details>

### Q6. You are picking a private range for a new company network. Which RFC 1918 range do you choose and why? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Three ranges exist: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16. The wrong choice causes future conflicts.

**What I'd do →**
1. Choose from 10.0.0.0/8 because it gives the most room to subdivide by region and environment
2. Avoid 192.168.0.0/16 and 172.17.0.0/16 because home routers and Docker use them
3. Pick a non-obvious slice such as `10.84.0.0/14` rather than starting at 10.0.0.0
4. Register it in an IPAM tool so nobody else reuses it

**Why →** Remote engineers on home networks and Docker bridges commonly collide with 192.168.x.x and 172.17.x.x, breaking VPN access.

**Interview tip →** Say you plan addressing for merges and multi-cloud before the first VPC is created.

</details>

### Q7. A server has two NICs and traffic to 10.20.0.0/16 is leaving through the wrong one. How do you see and fix the route? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-homed Linux hosts use the kernel routing table and the lowest-metric default route wins.

**What I'd do →**
1. Run `ip route get 10.20.1.5` to see which interface and source IP are used
2. List all routes with `ip route show table all`
3. Add a specific route: `sudo ip route add 10.20.0.0/16 via 10.20.0.1 dev eth1`
4. Persist it in netplan or NetworkManager, not only at runtime

**Why →** Specific routes beat the default route, so a route for the destination prefix on the right interface fixes the problem without touching the default.

**Interview tip →** Mention that runtime `ip route add` is lost on reboot, so the real fix lives in config management.

</details>

### Q8. Why can't two VPCs with the same CIDR be peered? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A router cannot decide where to send 10.0.1.5 if both sides own that prefix.

**What I'd do →**
1. Show the conflict: both route tables would need `10.0.0.0/16` pointing at the peer and local at once
2. AWS rejects the peering request with an overlapping CIDR error
3. Real options are re-IP one VPC, add a non-overlapping secondary CIDR for the shared workloads, or use PrivateLink for specific services
4. Prevent it with a central IPAM, for example AWS VPC IPAM allocations

**Why →** Peering is pure routing with no NAT, so addresses must be globally unique inside the connected domain.

**Interview tip →** Offer PrivateLink as the answer when the requirement is only "expose one service" rather than full network reachability.

</details>

### Q9. Plan IP addressing for a platform that will run 3 environments in 3 regions with 3 AZs each and 2 years of growth. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is a design question where the interviewer wants hierarchy and summarization, not a random list of CIDRs.

**What I'd do →**
1. Assign each region a /12 or /14 from the corporate 10.0.0.0/8 plan, for example eu-west-1 `10.16.0.0/14`, ap-south-1 `10.20.0.0/14`
2. Inside each region give each environment one VPC /16
3. Split every VPC into tier subnets sized per AZ, public /24, private app /22, data /24, plus reserved space
4. Reserve a pod CIDR separately (for example 100.64.0.0/10) if running EKS
5. Document it in IPAM and enforce with Terraform validation

**Why →** Hierarchical allocation means one summary route per region on the TGW or backbone and small firewall rule sets.

**Interview tip →** Mention that you size private app subnets generously because they hold nodes, pods and load balancer ENIs.

</details>

### Q10. Your company acquired another one and both use 10.0.0.0/8 with overlapping subnets. How do you connect them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Full routed connectivity is impossible with overlap, so the answer is selective connectivity or renumbering.

**What I'd do →**
1. List which services really need cross-company access, often fewer than 10
2. Expose those via PrivateLink endpoints or an API gateway instead of routing the networks
3. Where a network path is unavoidable use a NAT appliance with a translated non-overlapping range on one side
4. Start a re-IP project for the smaller network, migrating subnet by subnet
5. Block the old ranges in the shared VPN to avoid half-working routes

**Why →** NAT between overlapping networks works but is fragile (logs show translated IPs, protocols with embedded addresses break), so it must be a bridge, not the destination.

**Interview tip →** Say the end state is renumbering, with NAT or PrivateLink only as the bridge.

</details>

### Q11. Four /24s 10.1.0.0/24 through 10.1.3.0/24 sit behind a VPN. How do you advertise them efficiently? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Route summarization reduces table size and hides internal changes.

**What I'd do →**
1. Check the prefixes are contiguous and aligned on a power of two: 0,1,2,3 in the third octet
2. Summarize to `10.1.0.0/22`
3. Advertise only the /22 over BGP or add it as one static route on the remote side
4. Confirm no other host uses 10.1.0.0 to 10.1.3.255 outside the intended subnets

**Why →** A summary is only valid if every address in it is reachable through that next hop, otherwise you attract traffic you cannot deliver.

**Interview tip →** Mention that over-summarizing creates black holes and that AWS TGW lets you advertise summaries with attachments.

</details>

### Q12. Users complain about random connection resets when a server has two uplinks and a stateful firewall in front. What is happening? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic asymmetric routing: the SYN goes out through firewall A and the reply returns through B, which has no state for the flow and drops it.

**What I'd do →**
1. Capture on both firewalls with `tcpdump -ni eth0 host 10.2.3.4 and port 443` and check that each sees only half of the flow
2. Review routing on the server with `ip route get <client>` to see the return path
3. Make routing symmetric by using source-based routing (`ip rule add from 10.2.3.4 table 100`) or by pinning flows with ECMP hashing
4. If symmetry is impossible use state sync between the firewalls

**Why →** Stateful devices track connections and only accept return traffic they have seen leave.

**Interview tip →** Name the symptom correctly: SYN seen, no SYN-ACK or an ACK that gets dropped as invalid.

</details>

### Q13. When do you pick static routes vs BGP between on-prem and AWS? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The decision depends on the number of prefixes, the need for failover and who owns changes.

**What I'd do →**
1. Use static routes for a single site, a handful of stable prefixes and a single tunnel
2. Use BGP when you have multiple tunnels or circuits, many prefixes or need automatic failover
3. With BGP on Site-to-Site VPN or Direct Connect, control path with local preference and AS-path prepending
4. Monitor tunnel state with the `TunnelState` CloudWatch metric and BGP session alarms

**Why →** BGP withdraws a dead path on its own, while static routes need a human or health tracking to fail over.

**Interview tip →** Say you always prefer BGP for anything production because failover without human action is the point.

</details>

### Q14. Your EKS cluster ran out of pod IPs in a /24 subnet. What are your options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** With the AWS VPC CNI each pod consumes a VPC IP, so small subnets exhaust quickly.

**What I'd do →**
1. Add a secondary CIDR to the VPC, for example `100.64.0.0/16`, and create new subnets: `aws ec2 associate-vpc-cidr-block --vpc-id vpc-0abc --cidr-block 100.64.0.0/16`
2. Enable custom networking so pods use ENIConfig subnets (`AWS_VPC_K8S_CNI_CUSTOM_CNI=true`)
3. Enable prefix delegation (`ENABLE_PREFIX_DELEGATION=true`) to assign /28 prefixes per ENI
4. Roll nodes so they pick up the new settings

**Why →** Pod IPs come from the VPC; separating them into a secondary range keeps the primary range for nodes and load balancers.

**Interview tip →** Mention IPv6 clusters as the long-term fix for IP exhaustion.

</details>

### Q15. Traffic must flow through a virtual firewall instance in the middle of a VPC. How do you route it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need to steer traffic by route table to an instance ENI and let it forward packets that are not addressed to it.

**What I'd do →**
1. Add a route in the app subnet route table: destination `10.0.2.0/24` target the firewall ENI
2. Disable the source/destination check: `aws ec2 modify-instance-attribute --instance-id i-0abc --no-source-dest-check`
3. Enable forwarding in the instance: `sysctl -w net.ipv4.ip_forward=1`
4. Add the matching return route so traffic is symmetric

**Why →** By default AWS drops packets whose source or destination is not the instance; routers and NAT boxes must opt out.

**Interview tip →** For production prefer Gateway Load Balancer, which scales and health-checks appliances, over a single ENI route.

</details>

### Q16. You have 12 VPCs that need to talk to each other. Peering or Transit Gateway? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Peering is non-transitive and needs a full mesh, so the link count grows quickly.

**What I'd do →**
1. Count: full mesh of 12 VPCs needs 66 peering connections and route table entries in each VPC
2. Create one Transit Gateway and attach each VPC
3. Use separate TGW route tables to control who can reach whom
4. Accept the per-attachment and per-GB cost versus peering, which has no hourly charge

**Why →** Hub-and-spoke scales linearly and gives central routing control, at a price.

**Interview tip →** Say peering is still right for two or three VPCs with heavy traffic because it is cheaper and has no throughput cap.

</details>

### Q17. A developer's Docker bridge network 172.17.0.0/16 collides with the corporate VPN range. How do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Docker's default bridge uses 172.17.0.0/16, and anything on the corporate side using that block becomes unreachable from the host.

**What I'd do →**
1. Change the default address pool in `/etc/docker/daemon.json`: `{"bip":"172.30.0.1/24","default-address-pools":[{"base":"172.31.0.0/16","size":24}]}`
2. Restart with `sudo systemctl restart docker`
3. Recreate compose networks with `docker compose down && docker compose up -d`
4. Check with `docker network inspect bridge` and `ip route`

**Why →** The kernel sees the Docker bridge as a directly connected route that wins over the VPN route.

**Interview tip →** Roll the daemon.json change through your laptop or runner image so it never happens twice.

</details>

### Q18. Traffic from one source IP should leave through ISP B while everything else uses ISP A. How do you do it on Linux? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is policy-based routing: a rule chooses a routing table based on the packet source.

**What I'd do →**
1. Add a table in `/etc/iproute2/rt_tables`: `100 ispb`
2. Add a default route in it: `ip route add default via 203.0.113.1 dev eth1 table ispb`
3. Add the rule: `ip rule add from 10.0.0.50/32 table ispb priority 1000`
4. Verify with `ip rule show` and `ip route get 8.8.8.8 from 10.0.0.50 iif eth0`

**Why →** The kernel evaluates rules in priority order and uses the table of the first match, so the main table is only the fallback.

**Interview tip →** Mention making it persistent via netplan routing-policy or systemd-networkd.

</details>

### Q19. Remote engineers should reach only 10.0.0.0/8 through the VPN and use their own internet for everything else. How do you configure that? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is a split-tunnel design and the details differ per VPN type.

**What I'd do →**
1. On OpenVPN push specific routes only: `push "route 10.0.0.0 255.0.0.0"` and do not push `redirect-gateway def1`
2. On WireGuard set `AllowedIPs = 10.0.0.0/8` in the client peer, not `0.0.0.0/0`
3. Push internal DNS only for internal domains (split DNS)
4. Test with `ip route get 8.8.8.8` and `ip route get 10.1.2.3` from the client

**Why →** AllowedIPs in WireGuard and pushed routes in OpenVPN decide which destinations enter the tunnel.

**Interview tip →** Mention the trade-off: split tunnel saves bandwidth but internet traffic is not inspected by corporate security.

</details>

### Q20. Your team wants to roll out IPv6 in an existing AWS VPC. What is your plan? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dual-stack is the safe migration path because you keep IPv4 while adding IPv6.

**What I'd do →**
1. Associate an Amazon-provided /56 with the VPC and a /64 with each subnet
2. Add `::/0` routes to the IGW (or egress-only IGW for private subnets)
3. Update security groups and NACLs with `::/0` or the correct IPv6 CIDRs
4. Add AAAA records and test with `curl -6 https://example.com`
5. Roll out per service and watch dual-stack client behaviour (Happy Eyeballs)

**Why →** IPv6 addresses are public by default, so private subnets use an egress-only gateway and rules instead of NAT.

**Interview tip →** Say security rules are the usual forgotten part: teams add AAAA records but forget the IPv6 rules.

</details>

### Q21. Design TGW routing for shared services, prod and dev VPCs where prod and dev must never reach each other. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Isolation inside a Transit Gateway comes from separate route tables and attachment associations, not from security groups.

**What I'd do →**
1. Create TGW route tables: `rt-shared`, `rt-prod`, `rt-dev`
2. Associate the prod attachment with `rt-prod` and dev with `rt-dev`
3. Propagate shared-services routes into both, and propagate prod and dev routes only into `rt-shared`
4. Disable default association and propagation on the TGW so new attachments do not land in an open table
5. Test with `aws ec2 search-transit-gateway-routes` and a ping across the boundary

**Why →** The association decides which table an attachment looks up; propagation decides where its prefixes are advertised. Isolation is the absence of a route.

**Interview tip →** Mention adding a blackhole route for the other environment as defense in depth.

</details>

### Q22. Direct Connect is primary and a Site-to-Site VPN is backup. How do you make sure traffic prefers Direct Connect and fails over cleanly? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** AWS prefers Direct Connect over VPN when prefixes are equal, but you must control it deliberately.

**What I'd do →**
1. Advertise the same prefixes over both paths
2. AWS prefers Direct Connect for equal-length prefixes, so keep both prefixes the same length
3. On-prem set higher local preference for the DX learned path and prepend the AS path on the VPN advertisement
4. Enable BFD on the DX virtual interface for sub-second failure detection
5. Run a failover test during a change window and watch the `ConnectionState` metric

**Why →** A more specific prefix on the backup path would steal traffic, so keep prefixes aligned and control the preference with attributes.

**Interview tip →** Say you test failover on a schedule, because untested backup paths fail on the day you need them.

</details>

### Q23. Two VPN tunnels use ECMP and some flows break after you put a NAT device in the path. What went wrong? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** ECMP hashing can send packets of one flow on different tunnels, and stateful NAT on one path never sees the other half.

**What I'd do →**
1. Confirm per-flow load balancing and which tunnel each direction uses with flow logs and `tcpdump`
2. Make the NAT instance a single egress point or give each tunnel its own NAT range
3. Prefer active/standby (higher local preference on one tunnel) when state cannot be shared
4. If you need ECMP, use Transit Gateway VPN attachments, which support ECMP, and keep NAT out of that path

**Why →** Stateful NAT and ECMP only combine safely if all packets of a flow take the same path.

**Interview tip →** Show you understand that per-flow hashing is deterministic but is not aware of NAT state.

</details>

### Q24. The route table has both a static route and a BGP-propagated route to the same prefix. Which wins in a VPC? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** AWS has a defined route priority order, and getting it wrong creates outages during failover.

**What I'd do →**
1. Remember the order: local route first, then longest prefix match, then for equal prefixes static routes beat propagated ones
2. Among propagated routes, Direct Connect beats VPN static, which beats VPN BGP
3. Inspect the effective routes: `aws ec2 describe-route-tables --route-table-ids rtb-0abc --query 'RouteTables[0].Routes'`
4. If a stale static route outlives a migration, delete it

**Why →** A forgotten static route pins traffic to a dead path because it outranks propagated BGP routes of equal length.

**Interview tip →** Say you audit static routes after every network migration.

</details>

### Q25. You need to renumber a live /16 VPC that is running out of space without downtime. How do you approach it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You cannot change a VPC primary CIDR, but you can add CIDRs and move workloads onto them.

**What I'd do →**
1. Associate a secondary CIDR with the VPC and create new subnets in it
2. Update route tables, security groups and NACLs to allow the new ranges
3. Launch new instances or node groups in the new subnets and shift traffic through the load balancer
4. Drain and terminate the old instances, then remove the old subnets
5. Keep DNS TTLs low and update any IP-based allow lists in partner systems first

**Why →** The VPC is only a container; subnets and instances are what you move, which allows a blue-green style migration.

**Interview tip →** Highlight external IP allow lists: partners hardcoding your IPs are what breaks a renumbering.

</details>

## VPC & Cloud Network Design

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. What makes a subnet public versus private in AWS? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Subnets have no public/private flag; the difference is entirely the route table and the IP assignment.

**What I'd do →**
1. Look at the subnet route table: a route `0.0.0.0/0 -> igw-xxxx` makes it public
2. Check that instances have a public or Elastic IP, otherwise the IGW route is useless for them
3. Private subnets send `0.0.0.0/0` to a NAT gateway or have no default route at all
4. Verify with `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=subnet-0abc`

**Why →** The internet gateway only does 1:1 NAT for instances that already have a public IPv4 address.

**Interview tip →** Say that load balancers and NAT gateways go in public subnets, while application and database tiers stay private.

</details>

### Q27. Explain internet gateway versus NAT gateway and when you use each. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** They solve opposite directions of internet access.

**What I'd do →**
1. Use an IGW for resources with public IPs that must be reachable from and reach the internet (ALB, bastions)
2. Use a NAT gateway so private instances can start outbound connections (package updates, API calls) without being reachable inbound
3. Place the NAT gateway in a public subnet with an Elastic IP
4. Point private route tables `0.0.0.0/0` at the NAT gateway

**Why →** NAT gateways translate the private source to the NAT's Elastic IP and only allow return traffic for connections initiated from inside.

**Interview tip →** Mention NAT gateway cost (hourly plus per GB) because it is a favourite follow-up.

</details>

### Q28. Private instances need to pull packages from the internet but must not be reachable from it. What do you set up? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Standard outbound-only requirement.

**What I'd do →**
1. Create a NAT gateway in a public subnet with an Elastic IP
2. Add `0.0.0.0/0 -> nat-0abc` to the private subnet route table
3. Allow outbound 80/443 in the security group (the default allows all outbound)
4. Test from the instance with `curl -sI https://example.com`

**Why →** Inbound traffic cannot start connections to the private instances because no route or public IP points to them.

**Interview tip →** Offer VPC endpoints or a package mirror as the cheaper, more secure alternative.

</details>

### Q29. Why shouldn't production run in the default VPC? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The default VPC is built for convenience, not for controlled production.

**What I'd do →**
1. List problems: every default subnet is public with auto-assign public IP
2. The CIDR is fixed at 172.31.0.0/16 and can clash with other networks
3. There is no tier separation or private subnets by default
4. Build a custom VPC with Terraform, with explicit public, private and data subnets
5. Optionally delete the default VPC in every region with `aws ec2 delete-vpc`

**Why →** Everything launched there is internet-adjacent by default, which is the opposite of least privilege.

**Interview tip →** Mention that you delete or lock down default VPCs in new accounts through SCPs or a baseline stack.

</details>

### Q30. Public IP, Elastic IP and private IP: how do they differ on EC2? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Interviewers check whether you know which survives a stop/start.

**What I'd do →**
1. Private IP stays with the instance for life in the VPC and is used for internal traffic
2. Auto-assigned public IPv4 is released on stop and changes on start
3. Elastic IP is a static public address you own until you release it and can remap to another instance or ENI
4. Check with `aws ec2 describe-addresses`

**Why →** Public addressing is a pool resource; only an Elastic IP is stable.

**Interview tip →** Mention that idle or unattached Elastic IPs and all public IPv4 addresses are billed now, so you clean them up.

</details>

### Q31. Why use SSM Session Manager instead of a bastion host? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bastions are a long-lived internet-facing attack surface; SSM removes the open port.

**What I'd do →**
1. Attach an instance profile with `AmazonSSMManagedInstanceCore`
2. Ensure the instance can reach SSM through NAT or the interface endpoints `ssm`, `ssmmessages`, `ec2messages`
3. Connect with `aws ssm start-session --target i-0abc`
4. Enable session logging to S3 or CloudWatch for audit
5. Port forward with `aws ssm start-session --document-name AWS-StartPortForwardingSession`

**Why →** No inbound port 22, no key distribution, and every session is authenticated by IAM and logged.

**Interview tip →** Say you still keep a break-glass path documented for when SSM itself is down.

</details>

### Q32. Your private subnet instances reach S3 through the NAT gateway and the bill is large. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** NAT data processing charges apply to every GB, including traffic that never needs the internet.

**What I'd do →**
1. Create an S3 gateway endpoint: `aws ec2 create-vpc-endpoint --vpc-id vpc-0abc --service-name com.amazonaws.eu-west-1.s3 --route-table-ids rtb-0abc`
2. The endpoint adds a prefix-list route so S3 traffic bypasses NAT
3. Do the same for DynamoDB (also a gateway endpoint)
4. Confirm with Cost Explorer or flow logs that NAT bytes drop

**Why →** Gateway endpoints are free and keep the traffic on the AWS network.

**Interview tip →** Mention restricting the endpoint with an endpoint policy and bucket policy `aws:SourceVpce`.

</details>

### Q33. How do you design a highly available 3-tier network in one region? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The interviewer wants AZ redundancy per tier and a clear traffic flow.

**What I'd do →**
1. Use a VPC with subnets in at least 3 AZs per tier: public (ALB, NAT), private app, private data
2. Put one NAT gateway per AZ and route each AZ's private subnet to its own NAT
3. Place the ALB across all public subnets and spread ASG instances across private app subnets
4. Isolate data subnets with no default route, reachable only from the app security group
5. Add flow logs and endpoints for S3, ECR and SSM

**Why →** A single NAT gateway is a hidden single point of failure and also adds cross-AZ data charges.

**Interview tip →** Draw the picture; interviewers reward seeing the flow ALB to app to DB.

</details>

### Q34. Gateway endpoint versus interface endpoint: when do you use which? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both keep traffic off the internet but work very differently.

**What I'd do →**
1. Gateway endpoints exist only for S3 and DynamoDB and work as route table entries (free)
2. Interface endpoints are ENIs with private IPs in your subnets powered by PrivateLink and cost hourly plus per GB
3. Interface endpoints are reachable from on-prem over VPN or Direct Connect, gateway endpoints are not
4. Enable private DNS so `ssm.eu-west-1.amazonaws.com` resolves to the ENI IPs
5. Secure with a security group allowing 443 from the VPC CIDR

**Why →** The gateway type is a routing construct, the interface type is a network interface, which is why only the latter extends to on-prem.

**Interview tip →** Mention S3 interface endpoints exist too, for on-prem access to S3 without public IPs.

</details>

### Q35. Traffic is being rejected somewhere in your VPC. How do you use VPC Flow Logs to find out where? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Flow logs show ACCEPT or REJECT decisions at the ENI level, which separates security group and NACL drops from other issues.

**What I'd do →**
1. Enable flow logs on the VPC or ENI to CloudWatch Logs or S3
2. Query with Athena or CloudWatch Insights for `action = "REJECT"` and the target IP and port
3. Read the line: `2 123456789012 eni-0abc 10.0.1.5 10.0.2.9 51234 5432 6 10 840 1700000000 1700000060 REJECT OK`
4. Check whether the rejection came on the inbound ENI (SG or NACL) and whether the return leg is also rejected (NACL, since SGs are stateful)

**Why →** Flow logs record per-ENI decisions, so you can pin the drop to a specific hop.

**Interview tip →** Say flow logs do not show payload and are delayed by minutes, so you pair them with tcpdump for deep dives.

</details>

### Q36. How do you expose a service in one VPC to a customer's VPC without peering? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is the PrivateLink use case: one-way service exposure without full network connectivity or CIDR coordination.

**What I'd do →**
1. Put the service behind a Network Load Balancer or Gateway LB
2. Create a VPC endpoint service pointing at the NLB and require acceptance
3. The consumer creates an interface endpoint to your service name in their VPC
4. Allow principals with `aws ec2 modify-vpc-endpoint-service-permissions`
5. Consumer sees a private IP in their subnet and overlapping CIDRs do not matter

**Why →** PrivateLink is unidirectional and NATs the connection, so neither side exposes its network.

**Interview tip →** Mention that this is how most SaaS vendors offer private access.

</details>

### Q37. How do you connect VPCs across two AWS accounts and two regions? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-account and cross-region are both supported by peering and TGW with some caveats.

**What I'd do →**
1. For a few VPCs use inter-region VPC peering: request from one account, accept in the other, add routes on both sides
2. For many VPCs use TGW in each region with a TGW peering attachment and static routes (peering does not propagate)
3. Share a TGW across accounts with AWS RAM
4. Remember inter-region traffic is encrypted on the AWS backbone and billed per GB

**Why →** TGW peering routes are static, so you must add both sides explicitly.

**Interview tip →** Say you lay out CIDRs globally first because peering across regions amplifies any overlap.

</details>

### Q38. Your NAT gateway bill jumped. How do you find what is driving it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Data processing charges scale with bytes, so one chatty workload can dominate.

**What I'd do →**
1. Check CloudWatch metrics `BytesOutToDestination` and `BytesInFromSource` for the NAT
2. Enable VPC flow logs on the NAT ENI and query top talkers with Athena grouped by source IP
3. Look for container image pulls, S3 or log shipping traffic
4. Add gateway endpoints for S3 and DynamoDB and interface endpoints for ECR and CloudWatch
5. Consider NAT per AZ to avoid cross-AZ charges

**Why →** Most NAT cost is traffic that never needed the internet: AWS service calls.

**Interview tip →** Mention that image pulls from ECR over NAT are a classic hidden cost.

</details>

### Q39. A shared-services team wants to host subnets that other accounts launch into. What do you suggest? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is VPC sharing with AWS Resource Access Manager.

**What I'd do →**
1. Share subnets with RAM to the target accounts or OU
2. Participants launch resources into the shared subnet but cannot modify the VPC, route tables or subnets
3. Control placement and rules with security groups owned by participants
4. Use it to cut the number of VPCs, TGW attachments and NAT gateways

**Why →** It centralizes network ownership without making every team build and pay for its own network stack.

**Interview tip →** Warn that blast radius is shared, so noisy or compromised workloads share IP space and NAT.

</details>

### Q40. Design egress from private subnets that is stable for partners who allow-list your IPs. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Partners need a small, predictable set of source IPs.

**What I'd do →**
1. Route private subnets through NAT gateways with Elastic IPs (one per AZ)
2. Give the partner the three EIPs and document them
3. Consider centralized egress in a dedicated VPC through a TGW so IPs are shared across accounts
4. Use `aws ec2 describe-nat-gateways` to print the current IPs and alert if they change

**Why →** Elastic IPs on NAT gateways are static for as long as you keep them, unlike instance public IPs.

**Interview tip →** Mention BYOIP if the partner wants IPs your company owns.

</details>

### Q41. How would you let on-prem users resolve and reach private AWS resources over VPN? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Network reachability alone is not enough; DNS has to cross the VPN too.

**What I'd do →**
1. Establish Site-to-Site VPN or Direct Connect and propagate routes in both directions
2. Create a Route 53 Resolver inbound endpoint in the VPC
3. Point on-prem DNS conditional forwarders for `corp.aws.example` to the inbound endpoint IPs
4. Test with `dig @10.0.1.10 db.corp.aws.example` from on-prem
5. Open 53 UDP/TCP in security groups

**Why →** Private hosted zones are visible only inside VPCs, so on-prem needs the resolver endpoint to ask.

**Interview tip →** Mention an outbound endpoint for the reverse direction.

</details>

### Q42. A security team wants to isolate databases so no instance can reach the internet at all. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Combine routing with security controls.

**What I'd do →**
1. Place databases in subnets whose route table has only the local route
2. Remove any NAT or IGW route and use VPC endpoints only for needed services
3. Restrict security groups to inbound from the app tier SG ID only and limit outbound
4. Add an SCP or config rule that flags public subnets with data tier tags

**Why →** Without a route, no packet can leave, regardless of what a security group says.

**Interview tip →** Say this is defense in depth: route, SG, NACL and IAM each independently block.

</details>

### Q43. Design a multi-account landing zone network with centralized inspection and egress. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The expected answer is a hub VPC pattern with Transit Gateway and firewalls.

**What I'd do →**
1. Create an inspection VPC with Gateway Load Balancer endpoints and firewall appliances
2. Attach all spoke VPCs to a TGW with appliance mode enabled on the inspection attachment
3. Route spoke default routes to TGW, and TGW to inspection, then inspection to a central egress VPC with NAT gateways
4. Use separate route tables for spoke, inspection and egress
5. Manage with Terraform and AWS Network Firewall rule groups

**Why →** Appliance mode keeps both directions of a flow in the same AZ so stateful inspection works.

**Interview tip →** Mention that central egress simplifies allow-lists, but TGW data processing cost can be higher than local NAT.

</details>

### Q44. How do you use Gateway Load Balancer for third-party firewalls and why does it work? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** GWLB uses GENEVE encapsulation to transparently insert appliances.

**What I'd do →**
1. Deploy the appliances behind a GWLB (port 6081 GENEVE)
2. Create a GWLB endpoint service and endpoints in the protected VPCs
3. Route traffic to the GWLB endpoint using route tables (ingress via an edge association)
4. The GWLB hashes flows on a 5-tuple so both directions go to one appliance
5. Health-check targets on the appliance port

**Why →** GWLB combines a transparent bump-in-the-wire with the scaling and health checking of a load balancer.

**Interview tip →** Say you pick GWLB when you need a vendor firewall, and AWS Network Firewall when you want managed.

</details>

### Q45. Two AWS regions must work in active-active. How do you design the network? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Think connectivity, traffic steering and failure domains.

**What I'd do →**
1. Give each region non-overlapping VPC CIDRs and connect with TGW inter-region peering
2. Use Route 53 latency or weighted records with health checks, or Global Accelerator for anycast entry
3. Keep data replication on private paths (Aurora Global, DynamoDB global tables)
4. Make each region self-sufficient for DNS, NAT and endpoints
5. Test region evacuation with game days

**Why →** Active-active only works if each region can stand alone when the link or the other region fails.

**Interview tip →** Mention the egress cost of replication traffic.

</details>

### Q46. Connect an AWS VPC to a GCP VPC privately. What are your options? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-cloud connectivity has a few standard patterns, each with a cost and latency profile.

**What I'd do →**
1. Site-to-Site VPN with HA VPN on GCP and two tunnels on the AWS side, using BGP
2. For more bandwidth use a colocation provider with Direct Connect and Cloud Interconnect cross-connected
3. Use a neutral fabric provider (Equinix, Megaport) to avoid managing hardware
4. Keep CIDRs unique and run BGP with a clear ASN plan
5. Monitor tunnel health and failover

**Why →** VPN is fast to set up and encrypted but limited in throughput per tunnel (around 1.25 Gbps on AWS).

**Interview tip →** Say you compare cost of egress before deciding to move data across clouds.

</details>

### Q47. Replace the corporate VPN with a zero-trust model for engineers reaching internal apps. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The ask is identity-based access per application instead of network-level access.

**What I'd do →**
1. Put an identity-aware proxy in front of apps (for example Cloudflare Access, Google IAP, AWS Verified Access)
2. Integrate SSO with MFA and device posture
3. Remove broad network routes; allow only the proxy to reach apps
4. Use short-lived certs or tokens for SSH and databases (for example `teleport` or SSM)
5. Roll out app by app and keep VPN as fallback until usage drops

**Why →** Network location no longer implies trust; each request is authenticated and authorized.

**Interview tip →** Be honest about migration cost: legacy protocols may need a connector.

</details>

### Q48. Your NAT gateway shows ErrorPortAllocation. What is happening and how do you scale it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A NAT gateway supports about 55,000 simultaneous connections per destination IP, port and protocol; too many to one endpoint exhausts the ports.

**What I'd do →**
1. Check CloudWatch `ErrorPortAllocation` and `ActiveConnectionCount`
2. Find the destination causing it with flow logs
3. Spread across multiple NAT gateways (one per AZ or per subnet group)
4. Attach secondary EIPs to the NAT gateway
5. Reduce connection churn with keep-alive and connection pooling in the app

**Why →** The limit is per unique destination 3-tuple, so one popular API endpoint is the bottleneck.

**Interview tip →** Mention that connection reuse in the client is the cheapest fix.

</details>

### Q49. How do you design for DDoS resilience at the network edge? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Layered defense: absorb at the edge, filter, then scale origin.

**What I'd do →**
1. Front the app with CloudFront or a CDN and AWS Shield Advanced where justified
2. Add AWS WAF rate-based rules and geo or ASN blocks
3. Expose only the CDN to the origin (security group with the CloudFront managed prefix list)
4. Use Global Accelerator or anycast for non-HTTP traffic
5. Autoscale and set up alarms and a runbook with the provider's DRT contact

**Why →** Hiding the origin means attackers can only reach what the edge forwards.

**Interview tip →** Mention SYN cookies and conntrack limits on origin hosts.

</details>

### Q50. You operate multiple AWS accounts and need to audit whether any security group is open to 0.0.0.0/0 on SSH. How do you do it at scale? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need continuous detection, not a one-off script.

**What I'd do →**
1. Enable AWS Config rule `restricted-ssh` in all accounts via an aggregator
2. Query with `aws ec2 describe-security-groups --filters Name=ip-permission.from-port,Values=22 Name=ip-permission.cidr,Values=0.0.0.0/0`
3. Use Security Hub findings and EventBridge to auto-remediate or page
4. Block creation with an SCP or Terraform policy check (OPA, Checkov)

**Why →** Detective control plus preventive control in CI closes the loop.

**Interview tip →** Say remediation by auto-revoking is only safe after teams were warned.

</details>

## DNS & Service Discovery

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. A customer types www.example.com in a browser. Walk through how DNS resolves it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tests whether you know the resolver hierarchy and caching, not just "DNS turns names into IPs".

**What I'd do →**
1. Browser and OS check their caches and `/etc/hosts` first
2. The stub resolver asks the configured recursive resolver (for example 10.0.0.2 in a VPC or 1.1.1.1)
3. On a cache miss the recursive resolver asks a root server, then the `.com` TLD servers, then the authoritative nameserver of example.com
4. The answer returns with a TTL and is cached at every layer
5. Watch it with `dig +trace www.example.com`

**Why →** Each hop only knows who to ask next, and TTLs make the whole system fast and cheap.

**Interview tip →** Name the roles: stub, recursive, authoritative.

</details>

### Q52. What is the difference between an A record and a CNAME, and when do you use each? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** An A record maps a name to an IP; a CNAME maps a name to another name.

**What I'd do →**
1. Use A (or AAAA) when you control a fixed IP, for example `app.example.com A 203.0.113.10`
2. Use CNAME when the target is a name owned by someone else, for example `www CNAME myapp.herokuapp.com` or an ALB DNS name
3. Verify with `dig app.example.com` and `dig +short CNAME www.example.com`
4. Never put other record types next to a CNAME on the same name

**Why →** A CNAME lets the provider change IPs without you touching DNS.

**Interview tip →** Mention that a CNAME adds a resolution hop, so it costs one extra lookup on cold cache.

</details>

### Q53. Why can't you put a CNAME at the zone apex, and what do you use instead? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The apex (example.com) must hold SOA and NS records, and a CNAME cannot coexist with other records.

**What I'd do →**
1. Use a Route 53 ALIAS record to the ALB or CloudFront distribution for `example.com`
2. Other providers call it ANAME or CNAME flattening
3. Or use A records pointing to static IPs
4. Verify with `dig example.com A +short` and confirm it returns IPs

**Why →** ALIAS is resolved by the DNS provider at query time and returned as plain A records, so it is valid at the apex.

**Interview tip →** Add that Route 53 ALIAS queries to AWS resources are free.

</details>

### Q54. You are moving a site to a new server. What do you do with DNS to make the cutover smooth? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** DNS caching makes cutovers slow unless you prepare.

**What I'd do →**
1. Lower the TTL (for example from 86400 to 60 seconds) at least one old-TTL period before the change
2. Bring the new server up and test it with `curl --resolve www.example.com:443:203.0.113.20 https://www.example.com`
3. Change the record
4. Watch both servers' logs until traffic drains from the old one, then raise the TTL again

**Why →** Resolvers honor the TTL they cached, so you must wait out the old value before the low TTL takes effect.

**Interview tip →** Mention that some ISPs and apps ignore low TTLs, so keep the old server alive for a while.

</details>

### Q55. How do you find which nameservers are authoritative for a domain and query them directly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Troubleshooting basics with dig.

**What I'd do →**
1. Run `dig NS example.com +short`
2. Query one directly: `dig @ns1.example.com www.example.com` to bypass caches
3. Compare answers with a public resolver: `dig @1.1.1.1 www.example.com`
4. Look at the `flags` for `aa` (authoritative answer)
5. Use `dig +trace` to see the delegation chain

**Why →** Querying the authority directly separates "the record is wrong" from "a cache has stale data".

**Interview tip →** Say that comparing authoritative and recursive answers is your first move for any "DNS is wrong" ticket.

</details>

### Q56. What DNS records do you set up for company email? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** MX plus the three email authentication records.

**What I'd do →**
1. Add MX records pointing to the mail provider with priorities
2. Add an SPF TXT: `v=spf1 include:_spf.google.com ~all`
3. Add DKIM as a TXT or CNAME record at `selector._domainkey.example.com`
4. Add DMARC: `_dmarc TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"`
5. Check with `dig TXT example.com +short` and `dig MX example.com +short`

**Why →** SPF, DKIM and DMARC together keep your mail out of spam and stop spoofing of your domain.

**Interview tip →** Say DMARC should start at p=none with reporting, then move to quarantine and reject.

</details>

### Q57. When do you use /etc/hosts and when DNS? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hosts file is local, static and overrides DNS for that machine only.

**What I'd do →**
1. Use `/etc/hosts` for local testing, bootstrapping, or pinning a name before DNS exists
2. Use DNS for anything shared across hosts
3. Order is controlled by `hosts:` in `/etc/nsswitch.conf` (files dns)
4. Test with `getent hosts myapp` which follows nsswitch rather than only querying DNS

**Why →** `dig` ignores the hosts file, so it can disagree with what applications see; `getent` shows the application's view.

**Interview tip →** Mention that stale hosts entries are a classic cause of "works on my machine".

</details>

### Q58. Your Kubernetes pods see slow DNS and sometimes SERVFAIL on external names. Where do you look? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Typical CoreDNS issues: ndots, resolver overload, conntrack race on UDP.

**What I'd do →**
1. Check CoreDNS pods and logs: `kubectl -n kube-system logs deploy/coredns` and metrics for latency
2. Look at the pod `/etc/resolv.conf`: `options ndots:5` means `api.example.com` is tried with 4 search suffixes first
3. Lower it per pod with `dnsConfig: options: [{name: ndots, value: "2"}]` or use fully qualified names ending with a dot
4. Deploy NodeLocal DNSCache to cut latency and avoid conntrack
5. Scale CoreDNS by replicas

**Why →** With ndots 5 every external lookup generates several failing queries, multiplying load on CoreDNS.

**Interview tip →** Say you check `kubectl exec -it pod -- nslookup api.example.com` timing before and after.

</details>

### Q59. What is split-horizon DNS and when do you need it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The same name returns different answers inside and outside the network.

**What I'd do →**
1. Create a public hosted zone `example.com` with the public ALB and a private hosted zone with the same name associated to the VPC
2. Inside the VPC `app.example.com` returns a private IP
3. Outside it returns the public one
4. Verify from an instance with `dig app.example.com` and from home with the same command
5. Be careful: records missing in the private zone become NXDOMAIN inside the VPC

**Why →** Private zones shadow the public zone completely for associated VPCs, so every needed record must exist in both.

**Interview tip →** Name the failure mode: forgot to copy a record into the private zone.

</details>

### Q60. How do Route 53 weighted and failover routing differ and when do you use each? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both are DNS-level traffic policies with different goals.

**What I'd do →**
1. Weighted: split traffic by weights, for canary releases such as 95/5
2. Failover: primary answers while its health check is healthy, otherwise the secondary is returned
3. Attach health checks with `aws route53 create-health-check`
4. Set TTL low (30 to 60s) so shifts take effect fast
5. Latency and geolocation policies cover regional routing

**Why →** DNS routing decisions are cached, so TTL defines how quickly a change reaches clients.

**Interview tip →** Mention that DNS failover is coarse and not instant; pair with load balancer health checks.

</details>

### Q61. A deleted record is still being returned by some users hours later. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Caching at resolvers, and for deleted names negative caching also applies.

**What I'd do →**
1. Check the record's TTL and the SOA minimum field, which defines negative caching TTL
2. Query the authoritative server to confirm the new truth: `dig @ns1.example.com name`
3. Query public resolvers and see their remaining TTL in the answer
4. Flush a local resolver: `resolvectl flush-caches`
5. Wait out the TTL or ask the resolver operator to purge (Google and Cloudflare have purge pages)

**Why →** Resolvers cache both positive and negative answers for as long as the zone says.

**Interview tip →** Say you lower TTL before changes, not after.

</details>

### Q62. How do you delegate a subdomain to another team's AWS account? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Subdomain delegation lets the owning team manage their zone independently.

**What I'd do →**
1. In the child account create the hosted zone `dev.example.com` and note its four nameservers
2. In the parent zone add an NS record `dev.example.com` with those four values
3. Verify with `dig NS dev.example.com +short`
4. Remove the delegation to undo, and never keep conflicting A records for the same name in the parent zone

**Why →** NS records at the parent create the delegation; the child zone becomes authoritative.

**Interview tip →** Mention this is also how you split blast radius in a large organization.

</details>

### Q63. Your CI needs to obtain a wildcard TLS certificate from Let's Encrypt. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Wildcards require the DNS-01 challenge.

**What I'd do →**
1. Use certbot or lego with a DNS plugin for Route 53: `certbot certonly --dns-route53 -d '*.example.com' -d example.com`
2. Give the runner a narrowly scoped IAM role allowing `route53:ChangeResourceRecordSets` on that zone
3. The tool creates `_acme-challenge` TXT, Let's Encrypt validates it, and the record is deleted
4. Renew automatically with `certbot renew` on a timer

**Why →** HTTP-01 cannot prove control of a wildcard; DNS-01 proves control of the whole zone.

**Interview tip →** Say you scope the IAM permission to the zone or delegate `_acme-challenge` via a CNAME to a dedicated zone.

</details>

### Q64. How do you debug "it resolves for me but not for the customer"? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Likely different resolver, cached data, split horizon or a propagation issue.

**What I'd do →**
1. Ask the customer for the output of `nslookup name` and the resolver IP they used
2. Query from several places: `dig @8.8.8.8 name`, `dig @1.1.1.1 name`, `dig @<authoritative> name`
3. Compare answers and TTLs
4. Check for IPv6 AAAA records without a working IPv6 path
5. Check if the customer's network uses DNS filtering or a corporate resolver

**Why →** Isolating which resolver gives a different answer tells you whether the zone, the cache or the network is at fault.

**Interview tip →** Use online multi-location checkers as a second opinion.

</details>

### Q65. What are the limits of DNS-based load balancing? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** DNS rotation spreads clients roughly but does not know about load or health in real time.

**What I'd do →**
1. Round-robin returns several A records in rotating order, but clients and resolvers cache them
2. Some clients always pick the first record
3. With health checks you can remove dead IPs, but only after TTL expiry
4. Use a real load balancer or anycast for fine-grained balancing
5. Verify with repeated `dig +short name`

**Why →** DNS balances connection setup at best, and only as fast as caches expire.

**Interview tip →** Say DNS is for steering regions, not balancing requests.

</details>

### Q66. What happens when a service uses a hardcoded IP and the provider changes it, and how do you prevent it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pinned IPs in configs and firewall rules break silently when cloud endpoints rotate.

**What I'd do →**
1. Replace IPs with DNS names in config and resolve at connect time
2. For firewall rules use FQDN-based rules (for example Network Firewall domain lists) or provider prefix lists
3. Make sure the app respects DNS TTL: JVMs cache DNS forever unless `networkaddress.cache.ttl` is set
4. Alert on connection failures to the old IP

**Why →** Cloud load balancers and CDNs change IPs often; the name is the stable contract.

**Interview tip →** Mention the JVM DNS cache example; interviewers love it.

</details>

### Q67. A Route 53 private hosted zone isn't resolving from a second VPC. What do you check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A private zone only answers for VPCs it is associated with.

**What I'd do →**
1. Associate the VPC: `aws route53 associate-vpc-with-hosted-zone` (cross-account needs `create-vpc-association-authorization` first)
2. Ensure `enableDnsSupport` and `enableDnsHostnames` are true on the VPC
3. Test with `dig name` from an instance and look at the resolver 169.254.169.253 or VPC+2
4. Check no overlapping private zone with the same name in another association

**Why →** Without association the VPC resolver does not know the zone.

**Interview tip →** Mention that cross-account association needs a two-step authorization.

</details>

### Q68. Your DNS provider has an outage and your site is unreachable. How do you design to survive that? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** DNS is a global single point of failure unless you plan multi-provider.

**What I'd do →**
1. Run two authoritative providers (for example Route 53 and NS1 or Cloudflare) and publish both sets of NS records
2. Keep zones in sync with `octodns` or `dnscontrol` from Git
3. Keep TTL on NS and critical records reasonable (hours, not seconds) so resolvers keep answers during an outage
4. Monitor from outside with synthetic checks against each provider
5. Practice a drill

**Why →** Resolvers will fall through to the second provider's nameservers if the first stops answering.

**Interview tip →** Mention dnscontrol, with zone files as code reviewed through PRs.

</details>

### Q69. How do you roll out DNSSEC and what can go wrong? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** DNSSEC signs records so resolvers can validate them, and mistakes cause total outage for validating resolvers.

**What I'd do →**
1. Enable signing at the DNS provider (Route 53 uses a KMS key in us-east-1)
2. Create the DS record at the registrar from the DNSKEY
3. Verify with `dig +dnssec example.com` and `delv example.com`
4. Pre-lower DS TTL and plan key rollover; monitor expiry of signatures
5. If you must roll back, remove the DS record first and wait for its TTL before disabling signing

**Why →** A DS record with no matching key makes validating resolvers return SERVFAIL for your whole domain.

**Interview tip →** Say the order matters: remove DS before disabling signing.

</details>

### Q70. Your open resolver is used in a DNS amplification attack. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Open recursive resolvers answer anyone and amplify small queries into big responses.

**What I'd do →**
1. Restrict recursion to your networks (BIND `allow-recursion { 10.0.0.0/8; };`)
2. Enable response rate limiting (`rate-limit` in BIND options)
3. Use security groups and firewall rules to allow port 53 only from needed sources
4. Check with an external scan, and `dig +short ANY` shows large responses
5. Prefer managed resolvers

**Why →** Attackers spoof the victim's IP as the source; the resolver sends big responses at the victim.

**Interview tip →** Mention BCP38 egress filtering as the root fix at ISP level.

</details>

### Q71. How do you choose between DNS, Consul and Kubernetes services for microservice discovery? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each solves discovery at a different level of dynamism.

**What I'd do →**
1. Use Kubernetes Services and CoreDNS inside a cluster: `svc.namespace.svc.cluster.local`
2. Use Route 53 or an internal DNS for stable cross-cluster entry points
3. Use Consul or a service mesh when you need cross-platform discovery (VMs and pods), health-aware routing and mTLS
4. Keep the lowest layer that works, because every extra control plane is an outage source

**Why →** DNS is simple and universal but slow to change; registries react in seconds but need agents.

**Interview tip →** Say you avoid building discovery that platform already provides.

</details>

### Q72. Design DNS for hybrid multi-VPC, multi-account AWS with on-prem integration. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need a central design so teams don't each create overlapping private zones.

**What I'd do →**
1. Create a shared-services VPC with Route 53 Resolver inbound and outbound endpoints
2. Share resolver rules with RAM to all accounts and associate them with VPCs
3. Use private hosted zones per domain, associated to all VPCs that need them
4. Forward on-prem domains with outbound rules, and on-prem forwards AWS domains to inbound endpoint IPs
5. Log queries with Resolver query logging

**Why →** Central resolvers with shared rules avoid per-account sprawl and loops.

**Interview tip →** Mention avoiding forwarding loops between on-prem and AWS.

</details>

### Q73. How do you prevent subdomain takeover from dangling DNS records? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A CNAME pointing to a deleted cloud resource can be claimed by an attacker.

**What I'd do →**
1. Inventory all DNS records through exports from Route 53: `aws route53 list-resource-record-sets --hosted-zone-id Z123`
2. Scan CNAMEs for NXDOMAIN or provider takeover fingerprints with tools like `subjack` or `nuclei`
3. Make decommission steps delete DNS first
4. Put DNS in Terraform so resource and record share a lifecycle
5. Alert on new CNAMEs to third-party domains

**Why →** The record outlives the resource, and anyone who can claim the target name gets your domain's trust.

**Interview tip →** Say DNS belongs in the same pull request as the infrastructure it points to.

</details>

### Q74. How would you do a DNS-based canary release for a global app, and what are its risks? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Weighted DNS records provide a gradual shift but are coarse.

**What I'd do →**
1. Create two weighted records (for example 99 to the stable ALB and 1 to the canary ALB) with health checks
2. Increase weight in steps while watching error rates and latency
3. Keep TTL at 30 to 60 seconds
4. Know the risk: caching means a client may stay on canary longer, and weights are by DNS query, not by request
5. Prefer load-balancer weighted target groups for per-request control

**Why →** DNS shifts by resolver query, so low-volume clients may see skewed ratios.

**Interview tip →** Say you use DNS for cross-cluster shifts and ALB weights for within-region.

</details>

### Q75. Why would you use Anycast for DNS or edge services? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Anycast announces the same IP from many locations; routing delivers the user to the nearest.

**What I'd do →**
1. Announce the same prefix from several PoPs via BGP
2. Users reach the topologically nearest one, reducing latency
3. If a PoP fails, its announcement is withdrawn and traffic moves to the next
4. Works well for UDP DNS, and for TCP with care because route changes can break long connections
5. Examples: Cloudflare, Route 53, AWS Global Accelerator

**Why →** The network, not DNS logic, picks the closest site and handles failure.

**Interview tip →** Mention the TCP caveat: route flaps can reset a long-lived connection.

</details>

## Load Balancing, Proxies & CDN

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. What is the difference between a Layer 4 and a Layer 7 load balancer? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The layer decides what the balancer can see and therefore what it can route on.

**What I'd do →**
1. L4 (NLB, HAProxy in `mode tcp`) balances on IP and port without reading the payload, so it is fast and protocol agnostic
2. L7 (ALB, Nginx, Envoy) understands HTTP and can route on host, path, header or cookie
3. L7 can terminate TLS, rewrite headers and do WAF or redirects
4. Choose L4 for raw TCP/UDP, extreme throughput or static IPs, and L7 for web traffic

**Why →** The more of the packet the balancer parses, the smarter and the slower the decision gets.

**Interview tip →** Mention that NLB preserves the client source IP, while ALB does not and passes it in `X-Forwarded-For`.

</details>

### Q77. When do you choose an ALB and when an NLB in AWS? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Different products for different protocols and needs.

**What I'd do →**
1. ALB for HTTP/HTTPS, path or host routing, gRPC, WebSockets, authentication, WAF integration
2. NLB for TCP, UDP or TLS passthrough, static or Elastic IPs per AZ, millions of requests per second, ultra low latency
3. Use NLB in front of an ALB if you need static IPs for an HTTP app
4. Check with `aws elbv2 describe-load-balancers`

**Why →** ALB works at layer 7 with rich routing; NLB works at layer 4 with performance and IP guarantees.

**Interview tip →** Mention PrivateLink requires an NLB or GWLB, which is a common reason to pick NLB.

</details>

### Q78. Forward proxy versus reverse proxy: what is the difference? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** They sit on opposite sides of the connection.

**What I'd do →**
1. A forward proxy (Squid, corporate proxy) sits in front of clients and controls their outbound access, for example `HTTPS_PROXY=http://proxy:3128`
2. A reverse proxy (Nginx, Envoy, ALB) sits in front of servers, hiding them and handling TLS, caching and routing
3. Test a forward proxy with `curl -x http://proxy:3128 https://example.com`
4. Reverse proxy sits at the public edge of your apps

**Why →** A forward proxy represents clients to the world; a reverse proxy represents servers to the world.

**Interview tip →** Mention that service meshes are reverse proxies in sidecar form.

</details>

### Q79. How do health checks work on a load balancer and what makes a good one? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A health check decides which targets receive traffic.

**What I'd do →**
1. The LB probes each target periodically, for example HTTP GET `/healthz` every 10 seconds
2. A target is marked unhealthy after N consecutive failures and healthy after M successes
3. Keep the endpoint cheap and check only what this instance needs (can it serve?), not every downstream
4. Configure in AWS with `aws elbv2 modify-target-group --health-check-path /healthz --healthy-threshold-count 2`
5. Return 200 only after the app is warmed up

**Why →** Deep health checks that fail when the database is down can take the entire fleet out at once.

**Interview tip →** Say you separate liveness from readiness in Kubernetes for the same reason.

</details>

### Q80. What are sticky sessions and why would you avoid them? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Stickiness pins a client to one backend, usually via a cookie.

**What I'd do →**
1. ALB stickiness uses `AWSALB` cookie, enabled per target group with `stickiness.enabled=true`
2. Use only if the app keeps session state in memory
3. Better: move session state to Redis or use signed JWTs so any node can serve any request
4. Without stickiness, scaling in and rolling deployments are smooth

**Why →** Stickiness leads to uneven load and lost sessions when a node dies.

**Interview tip →** Say stateless services are the real answer; stickiness is a tactical workaround.

</details>

### Q81. What does a CDN do and when does it help? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A CDN caches content at edge locations close to users.

**What I'd do →**
1. Static assets (images, JS, CSS, video) are cached at the edge and served without touching the origin
2. Cache hits cut latency and origin load
3. Set `Cache-Control: public, max-age=31536000, immutable` on hashed assets
4. Dynamic content can still benefit from edge TLS termination and persistent origin connections
5. Check with `curl -sI https://cdn.example.com/app.js | grep -i x-cache`

**Why →** The speed of light is the real bottleneck for global users, and a nearby cache removes the round trip.

**Interview tip →** Mention cache keys, since mistakes there cause cache misses or wrong content for other users.

</details>

### Q82. Your app sees the load balancer's IP instead of the real client IP. How do you get the real one? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Proxies replace the source address, so the original goes in a header or the PROXY protocol.

**What I'd do →**
1. Read `X-Forwarded-For` (the left-most is the original client, but only trust entries added by your own proxies)
2. In Nginx use `set_real_ip_from 10.0.0.0/16; real_ip_header X-Forwarded-For;`
3. On NLB enable client IP preservation or PROXY protocol v2
4. Log `$http_x_forwarded_for` and verify with `curl -H 'X-Forwarded-For: 1.2.3.4'` to confirm spoofing is handled

**Why →** A client can send any XFF value, so trust only the hops you control.

**Interview tip →** Say that rate limiting and allow lists on a spoofable header are a security bug.

</details>

### Q83. Round robin versus least connections: when does it matter? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Algorithm choice matters when requests have uneven cost.

**What I'd do →**
1. Round robin is fine when requests are short and equal
2. Least connections (`least_conn` in Nginx, `leastconn` in HAProxy) is better when requests vary in duration, such as uploads and WebSockets
3. ALB offers round robin and least outstanding requests
4. Switch algorithm and compare p99 latency per backend

**Why →** Counting active work spreads load by real effort instead of by turn.

**Interview tip →** Mention weighted variants for mixed instance sizes.

</details>

### Q84. How do you route multiple microservices behind one domain with an ALB? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Path and host based listener rules.

**What I'd do →**
1. Create one ALB listener on 443 with a default fixed-response or main target group
2. Add rules: `/api/*` to the api target group, `/auth/*` to the auth target group, `Host: admin.example.com` to the admin group
3. Use `aws elbv2 create-rule --listener-arn <arn> --priority 10 --conditions Field=path-pattern,Values='/api/*' --actions Type=forward,TargetGroupArn=<tg>`
4. Set health checks per target group
5. Use IP target type for Kubernetes pods

**Why →** One public entry point reduces certificates, DNS records and cost.

**Interview tip →** Mention the 100 rules per listener default quota when the service count grows.

</details>

### Q85. A user reports that long-lived WebSocket connections drop after 60 seconds behind your ALB. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The ALB idle timeout default is 60 seconds and WebSocket idle periods hit it.

**What I'd do →**
1. Raise the ALB idle timeout: `aws elbv2 modify-load-balancer-attributes --load-balancer-arn <arn> --attributes Key=idle_timeout.timeout_seconds,Value=3600`
2. Send application-level pings every 20 to 30 seconds
3. Make sure backend and proxies in between (Nginx `proxy_read_timeout`) allow the same duration
4. Reconnect with backoff on the client

**Why →** The weakest timeout in the chain wins; every hop needs a matching value.

**Interview tip →** Say the keepalive from the app must be shorter than the LB idle timeout.

</details>

### Q86. Deployments cause 502 errors for a few seconds behind the load balancer. How do you stop that? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Targets are being removed while requests are in flight, or new targets get traffic before they are ready.

**What I'd do →**
1. Set the target group deregistration delay (default 300s) to cover your longest request: `aws elbv2 modify-target-group-attributes --attributes Key=deregistration_delay.timeout_seconds,Value=30`
2. On shutdown handle SIGTERM: stop accepting new work, finish in-flight requests, then exit
3. In Kubernetes add a `preStop` sleep and a readiness probe
4. Ensure the app keep-alive timeout is longer than the LB idle timeout to avoid reused connection resets

**Why →** 502s on deploy are race conditions between LB state and app lifecycle.

**Interview tip →** Mention the keep-alive mismatch as the most common hidden cause.

</details>

### Q87. How do you do blue/green with weighted target groups on an ALB? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Weighted forwarding lets you shift traffic gradually without DNS changes.

**What I'd do →**
1. Create two target groups, blue and green
2. Set a listener rule action forwarding with weights `blue=100 green=0`
3. Deploy green, test it through a header-based rule
4. Shift 10, 50, 100 percent while watching 5xx and latency
5. Roll back by setting the weights back, which is instant

**Why →** Weights are enforced per request, so shifts take effect immediately, unlike DNS.

**Interview tip →** Mention CodeDeploy or Argo Rollouts can automate these steps.

</details>

### Q88. Design Nginx as a reverse proxy for two app servers with keepalive, timeouts and a health fallback. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Typical hands-on config question.

**What I'd do →**
1. Define the upstream: `upstream app { server 10.0.1.10:8080 max_fails=3 fail_timeout=10s; server 10.0.1.11:8080; keepalive 32; }`
2. Proxy: `location / { proxy_pass http://app; proxy_http_version 1.1; proxy_set_header Connection ""; proxy_set_header Host $host; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; proxy_connect_timeout 3s; proxy_read_timeout 30s; }`
3. Validate with `nginx -t` and reload with `nginx -s reload`
4. Test failover by stopping one backend and watching `access.log`

**Why →** Upstream keepalive avoids a new TCP handshake per request, and explicit timeouts prevent hung workers.

**Interview tip →** Say open source Nginx has only passive health checks.

</details>

### Q89. Where do you terminate TLS: at the load balancer, at the app, or both? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Trade-off between simplicity, inspection and end-to-end encryption.

**What I'd do →**
1. Terminate at the ALB or CDN when you want WAF inspection, central cert management (ACM) and lower app complexity
2. Use TLS passthrough on NLB when the app must hold the key or you need mTLS end to end
3. Re-encrypt to the backend (ALB to HTTPS target group) if compliance requires in-transit encryption inside the VPC
4. Use ACM certs for the LB and a private CA for internal hops

**Why →** Terminating at the edge lets you inspect traffic; passthrough keeps the key out of the LB.

**Interview tip →** Say compliance requirements, not preference, usually decides.

</details>

### Q90. How do you invalidate a CDN cache after a deployment? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You must clear stale objects or avoid needing to.

**What I'd do →**
1. Prefer versioned file names (`app.4f3a9c.js`) with long TTLs and a short TTL on `index.html`
2. For emergencies invalidate paths: `aws cloudfront create-invalidation --distribution-id E123 --paths '/index.html' '/api/config*'`
3. Watch cost: first 1,000 paths a month are free, then billed
4. Verify with `curl -sI` and check `X-Cache: Hit from cloudfront` or `Miss`

**Why →** Immutable asset names turn invalidation from a process into a non-event.

**Interview tip →** Say you design cache keys so deploys never need a global purge.

</details>

### Q91. A cross-zone load balancing question: what changes when you enable it on an NLB? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Without cross-zone each LB node only sends traffic to targets in its own AZ.

**What I'd do →**
1. Without it an AZ with fewer targets gets the same share of traffic as one with many, causing hot spots
2. Enable: `aws elbv2 modify-load-balancer-attributes --attributes Key=load_balancing.cross_zone.enabled,Value=true`
3. ALB has it on by default, NLB and GWLB off by default
4. Remember cross-AZ data transfer charges apply on NLB when enabled

**Why →** Cross-zone evens load but adds inter-AZ cost and a shared failure surface.

**Interview tip →** Mention that you keep target counts balanced across AZs when it's off.

</details>

### Q92. How do you rate-limit abusive clients at the edge? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The limit belongs as close to the attacker as possible.

**What I'd do →**
1. Use AWS WAF rate-based rules, for example 2000 requests per 5 minutes per IP, on CloudFront or the ALB
2. Or Nginx: `limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;` plus `limit_req zone=api burst=20 nodelay;`
3. Return 429 with `Retry-After`
4. Key on API key or user when many clients share NAT IPs
5. Monitor blocked counts

**Why →** Limits on source IP are crude for mobile carriers and offices behind NAT, so keys matter.

**Interview tip →** Say you always start in count mode before blocking.

</details>

### Q93. Explain how you would load balance gRPC services properly. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** gRPC uses long-lived HTTP/2 connections, so connection-level balancing sends all requests from one client to one backend.

**What I'd do →**
1. Use an L7 balancer that balances per request (ALB supports gRPC, or Envoy)
2. In Kubernetes use a headless service with client-side balancing or a mesh such as Istio or Linkerd
3. Configure health checks with the gRPC health protocol
4. Set max connection age on servers so connections rotate
5. Verify distribution by comparing per-pod request counters

**Why →** A plain L4 balancer sees one connection and cannot spread the multiplexed requests inside it.

**Interview tip →** Mention `MAX_CONNECTION_AGE` as the hidden hero.

</details>

### Q94. Compare CloudFront and Global Accelerator for a global application. When do you use each? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** One is a caching CDN, the other is an anycast network onto the AWS backbone.

**What I'd do →**
1. CloudFront caches HTTP content at the edge and terminates TLS near users, good for static and cacheable APIs
2. Global Accelerator gives two static anycast IPs and routes TCP/UDP over the AWS backbone to the closest healthy regional endpoint, with no caching
3. Use GA for non-HTTP protocols, gaming, static IPs and fast regional failover
4. Both can be combined for different paths
5. Verify with `aws globalaccelerator list-accelerators`

**Why →** CDN reduces work; accelerator reduces path length and failover time.

**Interview tip →** Say you decide by cacheability and protocol.

</details>

### Q95. A cache stampede hits your origin when a popular object expires. How do you protect the origin? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Many edge nodes request the same object simultaneously on a miss.

**What I'd do →**
1. Enable origin shield (CloudFront) or a regional cache layer to collapse requests
2. Use request collapsing and `stale-while-revalidate` / `stale-if-error` headers
3. Add jitter to TTLs so objects don't expire together
4. Rate-limit and autoscale the origin
5. Pre-warm important URLs after deploy

**Why →** Collapsing concurrent misses into one origin fetch turns a thundering herd into a single request.

**Interview tip →** Mention stale-if-error as a cheap availability feature.

</details>

### Q96. You need to design load balancing for 1 million requests per second. What changes? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** At this scale the balancer itself, connection limits and warm-up matter.

**What I'd do →**
1. Use NLB or a fleet of Envoy or HAProxy nodes behind anycast or ECMP routing
2. Pre-warm ALBs with AWS Support or use many smaller ALBs by service
3. Prefer connection reuse and HTTP/2 or HTTP/3 to cut handshakes
4. Scale targets across AZs with cross-zone on
5. Load test ramps gradually with `k6` or `vegeta` and watch ephemeral port, conntrack and file-descriptor limits

**Why →** At extreme scale, the bottleneck is usually state: sockets, conntrack tables and TLS handshakes.

**Interview tip →** Say you remove the shared state first and the load balancer second.

</details>

### Q97. What is the PROXY protocol and when do you need it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** It carries the original client address across an L4 hop.

**What I'd do →**
1. Enable PROXY protocol v2 on the NLB target group: `proxy_protocol_v2.enabled=true`
2. Configure the backend to expect it: Nginx `listen 443 ssl proxy_protocol;` with `set_real_ip_from` and `real_ip_header proxy_protocol;`
3. Do not enable it on one side only, because the backend then sees garbage in the first bytes and rejects with 400
4. Verify with `tcpdump -A` on the first packet

**Why →** An L4 balancer cannot add HTTP headers, so the PROXY header is prepended to the TCP stream.

**Interview tip →** Say mismatched settings cause immediate protocol errors.

</details>

### Q98. You must migrate from one load balancer to another with zero downtime. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Run both in parallel and move traffic with DNS weights.

**What I'd do →**
1. Build the new LB with identical rules and targets and test with `curl --resolve`
2. Lower the DNS TTL in advance
3. Use weighted records to shift 1, 10, 50, 100 percent while watching 5xx and latency
4. Keep the old LB for a full TTL after 100 percent
5. Update allow lists that contain LB IPs or hostnames

**Why →** Parallel run with weighted DNS makes rollback instant at any step.

**Interview tip →** Mention that clients with stale DNS keep hitting the old one for TTL, so don't decommission early.

</details>

## TLS, HTTP & Application Layer

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q99. Explain what happens in a TLS 1.3 handshake in a few sentences. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Interviewers want the sequence and what it achieves, not the byte layout.

**What I'd do →**
1. Client sends ClientHello with supported ciphers, key share and SNI (the hostname)
2. Server replies with ServerHello, its key share, certificate and a signed Finished message
3. Both derive session keys from the key exchange (ECDHE), so the rest is encrypted
4. Client validates the certificate chain, hostname and expiry, then sends Finished
5. Inspect with `openssl s_client -connect example.com:443 -servername example.com -tls1_3`

**Why →** TLS 1.3 needs one round trip and gives forward secrecy by default.

**Interview tip →** Mention that TLS 1.2 needs two round trips and TLS 1.3 removed weak ciphers.

</details>

### Q100. Your site must redirect all HTTP traffic to HTTPS. How do you do it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A simple but common task, with a security extra.

**What I'd do →**
1. On the ALB add a listener on port 80 with a redirect action to HTTPS 443 and status HTTP_301
2. On Nginx use `server { listen 80; return 301 https://$host$request_uri; }`
3. Add HSTS on the HTTPS side: `add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;`
4. Test with `curl -sI http://example.com`

**Why →** A redirect protects the first request; HSTS makes browsers skip HTTP afterward.

**Interview tip →** Say you start HSTS with a short max-age before enabling preload.

</details>

### Q101. What is the difference between 4xx and 5xx errors, and which ones do you see most in networking problems? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Client versus server errors, and which of them point at infrastructure.

**What I'd do →**
1. 4xx means the request is the problem: 400 bad request, 401 and 403 auth, 404 not found, 429 rate limited
2. 5xx means the server or path failed: 500 app error, 502 bad gateway (invalid response from upstream), 503 unavailable (no healthy targets or overload), 504 gateway timeout (upstream too slow)
3. From the proxy log compare `upstream_status` and `status`
4. Test the backend directly with `curl -v http://10.0.1.10:8080/health`

**Why →** 502, 503 and 504 almost always mean the proxy or load balancer cannot get a good answer from its backend.

**Interview tip →** Say you map 502 to bad response, 503 to no capacity and 504 to slow response.

</details>

### Q102. HTTP/1.1, HTTP/2 and HTTP/3: what changed and why does it matter in operations? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each version fixed head-of-line blocking at different layers.

**What I'd do →**
1. HTTP/1.1 needs multiple connections for parallelism
2. HTTP/2 multiplexes streams over one TCP connection with header compression, but one lost packet blocks all streams (TCP head-of-line blocking)
3. HTTP/3 runs over QUIC on UDP 443 and avoids that, with faster handshakes
4. Test with `curl --http2 -I https://example.com` and `curl --http3 -I https://example.com`
5. Ensure firewalls allow UDP 443

**Why →** Fewer connections and fewer round trips reduce latency, especially on mobile networks.

**Interview tip →** Mention that gRPC requires HTTP/2.

</details>

### Q103. The browser console shows a CORS error when your frontend calls your API. What is it and how do you fix it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** CORS is enforced by the browser, not the network; the API must say which origins may read the response.

**What I'd do →**
1. Check the error: `has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header`
2. Inspect the preflight: `curl -i -X OPTIONS https://api.example.com/v1/items -H 'Origin: https://app.example.com' -H 'Access-Control-Request-Method: POST'`
3. Return `Access-Control-Allow-Origin: https://app.example.com`, allowed methods and headers
4. Make sure the OPTIONS request is not blocked by auth or the gateway

**Why →** The server opts in to cross-origin reads with headers; a wildcard with credentials is rejected by browsers.

**Interview tip →** Say CORS is not a security control for the API itself.

</details>

### Q104. How do you check which certificate a server presents and when it expires? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Basic operational skill.

**What I'd do →**
1. Run `openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates`
2. Check SANs with `-ext subjectAltName`
3. Or use `curl -vI https://example.com` and read the TLS section
4. Alert on `ssl_expiry` with Prometheus blackbox exporter

**Why →** The `-servername` flag matters because servers pick a certificate by SNI.

**Interview tip →** Mention that checking from outside the network also reveals CDN or LB certificates.

</details>

### Q105. What is the difference between HTTP keep-alive and a new TCP connection per request? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Connection reuse saves handshakes.

**What I'd do →**
1. A new connection costs a TCP handshake plus TLS handshake, around 2 to 3 round trips
2. Keep-alive reuses the connection for many requests
3. Configure with Nginx `keepalive_timeout 65;` for clients and `keepalive 32;` for upstream
4. Confirm reuse with `curl -v` showing `Re-using existing connection`

**Why →** Connection setup often costs more time than the request itself.

**Interview tip →** Say the server's keep-alive timeout should exceed the LB's idle timeout.

</details>

### Q106. Design mutual TLS between two internal services. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both sides authenticate with certificates; the hard part is issuance and rotation.

**What I'd do →**
1. Run an internal CA (Vault PKI, AWS Private CA, or cert-manager with a CA issuer)
2. Issue short-lived certs (24 hours) to each workload through a sidecar or SPIFFE
3. Configure the server to require client certs: Nginx `ssl_verify_client on; ssl_client_certificate ca.pem;`
4. Test with `curl --cert client.crt --key client.key --cacert ca.crt https://svc:8443`
5. Prefer a mesh (Istio, Linkerd) to automate it

**Why →** Short-lived certs make revocation mostly unnecessary.

**Interview tip →** Say manual mTLS fails at rotation time, which is why you automate.

</details>

### Q107. How do you rotate a TLS certificate without downtime? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Overlap old and new and reload gracefully.

**What I'd do →**
1. Issue the new certificate before the old one expires, at least 30 days ahead
2. Deploy the chain and key in the same file set and reload gracefully: `nginx -t && nginx -s reload`
3. On ALB import or replace the cert in ACM and the listener updates without drop
4. Verify with `openssl s_client` from outside
5. Automate with certbot or cert-manager renewals and alert at 14 days

**Why →** Graceful reload keeps existing connections alive while new ones use the new cert.

**Interview tip →** Mention that forgotten certs on non-HTTP services (SMTP, LDAP) are the usual outage.

</details>

### Q108. Browsers trust your server but `curl` and Java clients fail with "unable to get local issuer certificate". What happened? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The server sends only the leaf certificate and browsers fix it by fetching the intermediate.

**What I'd do →**
1. Check the chain: `openssl s_client -showcerts -connect example.com:443 -servername example.com`
2. Confirm only one certificate is sent
3. Configure the server with the full chain: in Nginx `ssl_certificate fullchain.pem;`
4. Re-test with `curl -v` and with an SSL checker

**Why →** Browsers cache or fetch intermediates (AIA), but most libraries do not.

**Interview tip →** Say the leaf must come first and the chain follows in order.

</details>

### Q109. How do you serve multiple HTTPS domains on one IP? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** SNI allows the client to say which hostname it wants during the handshake.

**What I'd do →**
1. Configure one server block per domain with its own certificate in Nginx `server_name`
2. On an ALB attach multiple certificates to the listener, and it picks by SNI
3. Test with `openssl s_client -connect IP:443 -servername a.example.com` and again with another name
4. Use wildcard or SAN certificates to reduce certificate count

**Why →** SNI is sent in the ClientHello, before encryption begins.

**Interview tip →** Mention that very old clients without SNI get the default certificate.

</details>

### Q110. Uploads of 5 MB fail with 413 behind your reverse proxy. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Proxies enforce body size limits, and each hop has its own.

**What I'd do →**
1. Nginx: set `client_max_body_size 50m;` in the `http`, `server` or `location`
2. Check ingress-nginx annotation `nginx.ingress.kubernetes.io/proxy-body-size: "50m"`
3. Check ALB (1 MB limit for Lambda targets) and WAF body inspection limits
4. Test with `curl -F file=@big.bin https://example.com/upload -v`

**Why →** The smallest limit in the chain rejects the request, and the error comes from that hop.

**Interview tip →** Say you confirm which hop answered by looking at response headers.

</details>

### Q111. Your proxy retries failed POST requests and customers are charged twice. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Retries are only safe for idempotent requests.

**What I'd do →**
1. Disable automatic retries for non-idempotent methods in the proxy (Nginx `proxy_next_upstream off;` or limit to `error timeout` with `non_idempotent` not set)
2. In Envoy set `retry_on` carefully and avoid POST
3. Add idempotency keys in the API so retries are safe
4. Log request IDs through every hop to find duplicates

**Why →** A timeout does not mean the request failed; the backend may have completed it.

**Interview tip →** Say safe retries need idempotency keys, not just luck.

</details>

### Q112. How do you run Kubernetes certificates automatically with cert-manager? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** cert-manager issues and renews certificates as Kubernetes resources.

**What I'd do →**
1. Install it: `helm install cert-manager jetstack/cert-manager --set crds.enabled=true -n cert-manager --create-namespace`
2. Create a `ClusterIssuer` for Let's Encrypt with HTTP-01 or DNS-01 solvers
3. Annotate the Ingress with `cert-manager.io/cluster-issuer: letsencrypt` and set `tls.secretName`
4. Watch `kubectl get certificate,certificaterequest,order,challenge -A`
5. Alert on `certmanager_certificate_expiration_timestamp_seconds`

**Why →** The controller renews at two thirds of the lifetime by default, so expiry becomes a monitoring problem rather than a calendar one.

**Interview tip →** Say rate limits of Let's Encrypt are why you use staging first.

</details>

### Q113. What TLS policy do you set on a public load balancer? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Balance security and client compatibility.

**What I'd do →**
1. Allow TLS 1.2 and 1.3 only and disable TLS 1.0 and 1.1
2. On ALB choose a policy such as `ELBSecurityPolicy-TLS13-1-2-2021-06`
3. Prefer ECDHE ciphers with AES-GCM or ChaCha20
4. Scan with `testssl.sh example.com` or `nmap --script ssl-enum-ciphers -p 443 example.com`
5. Check access logs for old TLS versions before tightening

**Why →** Dropping old versions removes known attacks, but must be gated on client data.

**Interview tip →** Say you measure client TLS versions before changing policy.

</details>

### Q114. A health check to an HTTPS backend fails after enabling TLS on the app. What do you check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The load balancer must trust or ignore the backend certificate.

**What I'd do →**
1. Confirm the health check protocol is HTTPS and the port matches
2. Test locally: `curl -vk https://10.0.1.10:8443/health`
3. ALB does not validate the backend certificate, but NLB TLS and Envoy do
4. Check the path and expected status code `200-299`
5. Look at the TargetHealth reason with `aws elbv2 describe-target-health`

**Why →** A protocol mismatch looks like a timeout or a TLS alert at the LB.

**Interview tip →** Say target health reason codes name the failure exactly.

</details>

### Q115. Design an internal PKI for a microservice platform. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Think root protection, issuing CA, automation, revocation and trust distribution.

**What I'd do →**
1. Keep an offline root CA and an online intermediate in Vault PKI or AWS Private CA
2. Issue short-lived workload certs (hours) through SPIFFE/SPIRE or the mesh
3. Distribute the root via config management and container base images
4. Limit certificate SANs and key usage with roles
5. Monitor expirations and CA health, and test CA rotation regularly

**Why →** Short lifetimes plus automation beat CRLs and manual renewal.

**Interview tip →** Say the intermediate rotation is the test of your design.

</details>

### Q116. How do you minimize TLS handshake latency for a global application? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cut round trips and cryptographic work.

**What I'd do →**
1. Enable TLS 1.3 and session resumption
2. Terminate TLS at edge locations close to users (CDN)
3. Use OCSP stapling so clients don't fetch revocation info
4. Use ECDSA certificates, which are cheaper than RSA
5. Use HTTP/2 or HTTP/3 and long-lived connections to origin
6. Measure with `curl -w '%{time_connect} %{time_appconnect}\n' -so /dev/null https://example.com`

**Why →** Every round trip costs the full RTT, so the cheapest handshake is the one you skip.

**Interview tip →** Mention that 0-RTT in TLS 1.3 has replay risks, so only for idempotent requests.

</details>

### Q117. Explain HTTP request smuggling and how infrastructure choices affect it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Front-end and back-end disagree about where one request ends.

**What I'd do →**
1. Understand the cause: conflicting `Content-Length` and `Transfer-Encoding` headers parsed differently by proxy and backend
2. Use the same HTTP stack at both layers or normalize requests at the edge
3. Prefer HTTP/2 end to end, and reject ambiguous requests
4. Keep proxies, WAF and servers patched
5. Test with tools like Burp's HTTP Request Smuggler in a staging environment

**Why →** An attacker hides a second request inside the first, which the backend processes with another user's context.

**Interview tip →** Say you disable connection reuse between mismatched components as a mitigation.

</details>

### Q118. You are rolling out HTTP/3 and some users report the site is slower. Why? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** QUIC runs over UDP, which some networks throttle or block.

**What I'd do →**
1. Check clients fall back: browsers retry over TCP when QUIC fails, with a delay
2. Verify the UDP 443 path in security groups, firewalls and the LB
3. Advertise with `Alt-Svc: h3=":443"; ma=86400` and watch the share of h3 traffic
4. Compare real-user metrics by protocol
5. If corporate networks block UDP, they will use h2 anyway

**Why →** Fallback handles blocking, but the fallback delay and middlebox throttling can hurt.

**Interview tip →** Say you roll out h3 gradually and compare RUM data.

</details>

### Q119. A retry storm takes down a dependency. How do you design the network and client layers to prevent it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Retries multiply load at the moment the service is least able to handle it.

**What I'd do →**
1. Use exponential backoff with jitter on clients
2. Add retry budgets, for example max 10 percent extra requests (Envoy `retry_budget`)
3. Use circuit breakers and outlier detection to eject unhealthy hosts
4. Apply timeouts at every hop, shorter at each layer toward the user
5. Load shed with 429 or 503 and `Retry-After`

**Why →** Retries without limits turn a small failure into a self-inflicted outage.

**Interview tip →** Say the answer is budgets, not more retries.

</details>

### Q120. You must rotate the root CA used by every service in a mesh. How do you avoid an outage? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Trust must overlap: new CA is trusted before it is used to sign.

**What I'd do →**
1. Distribute a trust bundle containing both old and new roots to all workloads
2. Wait until every workload has the combined bundle
3. Switch the issuing CA to the new one and let short-lived certs rotate
4. Remove the old root only after its issued certs have expired
5. Monitor handshake failures per service, and have a rollback to the old issuer

**Why →** Trust first, then issue, then retire; any other order breaks verification.

**Interview tip →** Say you rehearse this in staging because CA rotation is rare and scary.

</details>

## Security: Firewalls, SGs, NACLs & VPN

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q121. Security group versus NACL: explain the differences. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** One of the most common interview questions; the key is stateful versus stateless.

**What I'd do →**
1. Security groups attach to ENIs, are stateful, support only allow rules and evaluate all rules together
2. NACLs attach to subnets, are stateless, support allow and deny, and evaluate rules in number order
3. With a NACL you must allow return traffic yourself, including ephemeral ports 1024-65535
4. Use SGs for normal access control and NACLs as a coarse subnet-level backstop or to block a bad IP
5. Inspect with `aws ec2 describe-network-acls`

**Why →** Statefulness means SG return traffic is automatic, while a NACL treats each packet independently.

**Interview tip →** Say NACLs are rarely the first tool; you use them for explicit denies.

</details>

### Q122. How do you let only your office IP SSH into a server? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Least privilege on port 22.

**What I'd do →**
1. Edit the security group inbound rule: `aws ec2 authorize-security-group-ingress --group-id sg-0abc --protocol tcp --port 22 --cidr 203.0.113.7/32`
2. Remove any `0.0.0.0/0` rule for 22
3. Better: use SSM Session Manager and close port 22
4. Verify from outside with `nc -vz server 22` and from another network, which should time out

**Why →** A /32 limits access to one address; a timeout from other sources shows the SG is dropping, not rejecting.

**Interview tip →** Mention that office IPs change, so a VPN with a fixed egress IP is more stable.

</details>

### Q123. What does it mean that a security group can reference another security group? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Group references make rules follow instances instead of IPs.

**What I'd do →**
1. Allow the database SG to accept 5432 from the app SG: `--source-group sg-app`
2. Any instance with the app SG may connect, even as IPs change
3. It also works across peered VPCs in the same region
4. Check rules in the console or `aws ec2 describe-security-group-rules --filters Name=group-id,Values=sg-db`

**Why →** IP lists rot as autoscaling changes instances, while group membership is always current.

**Interview tip →** Say this is how you build tiered access without CIDRs.

</details>

### Q124. Stateful versus stateless firewalls: what is the practical difference? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Do you track connections or evaluate each packet alone?

**What I'd do →**
1. A stateful firewall remembers outbound connections and automatically allows the replies (security groups, `iptables -m conntrack --ctstate ESTABLISHED,RELATED`)
2. A stateless firewall needs explicit rules both ways (NACLs, router ACLs)
3. Stateful devices have connection table limits
4. Check with `conntrack -L | wc -l` and `sysctl net.netfilter.nf_conntrack_max`

**Why →** State lets you write simple rules, at the cost of memory and a table that can fill.

**Interview tip →** Say you remember conntrack table exhaustion as the failure mode.

</details>

### Q125. What is a WAF and where do you place it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** It filters HTTP requests for application attacks.

**What I'd do →**
1. A WAF inspects layer 7 for SQL injection, XSS, bad bots and rate abuse
2. Attach AWS WAF to CloudFront, an ALB or API Gateway
3. Start with managed rule groups (`AWSManagedRulesCommonRuleSet`) in count mode, then block
4. Review sampled requests and tune false positives
5. It does not replace secure code or network controls

**Why →** It blocks known bad patterns before they reach your application, but only the patterns it knows.

**Interview tip →** Say count mode first, because false positives can take your own app down.

</details>

### Q126. What is the difference between a site-to-site VPN and a client VPN? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** One joins networks, the other joins individual users to a network.

**What I'd do →**
1. Site-to-site connects an office or data center router to a VPC with IPsec tunnels (AWS provides two tunnels per connection)
2. Client VPN gives each user an encrypted connection from a laptop (OpenVPN, WireGuard, AWS Client VPN)
3. Site-to-site is always-on and routed, client VPN is per user with authentication
4. Test with `ip route`, `ping` across the tunnel and `wg show` or `ipsec status`

**Why →** The first is network to network; the second is device to network.

**Interview tip →** Mention both tunnels in site-to-site VPN must be configured for HA.

</details>

### Q127. How does DNAT or port forwarding work on a Linux gateway? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rewrites the destination address of incoming packets.

**What I'd do →**
1. Enable forwarding: `sysctl -w net.ipv4.ip_forward=1`
2. Add the rule: `iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.1.20:80`
3. Allow the forwarded traffic: `iptables -A FORWARD -p tcp -d 10.0.1.20 --dport 80 -j ACCEPT`
4. Add MASQUERADE if the return path would not come back through the gateway
5. List with `iptables -t nat -L -n -v`

**Why →** PREROUTING runs before the routing decision, so rewriting there sends the packet to the new host.

**Interview tip →** Say nftables is the modern replacement for iptables.

</details>

### Q128. Why restrict outbound traffic in security groups as well as inbound? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default SGs allow all outbound, which helps attackers after a compromise.

**What I'd do →**
1. Remove the default allow-all egress rule on sensitive tiers
2. Allow only the required destinations, for example 443 to the NAT, 5432 to the database SG
3. Use VPC endpoints and prefix lists for AWS services
4. Test that the app still works, and watch flow logs for REJECT lines that reveal missing rules

**Why →** Egress filtering stops data exfiltration and reverse shells after an app compromise.

**Interview tip →** Say it's high value but expensive in effort, so apply it first to data and PCI tiers.

</details>

### Q129. Your database tier must be reachable only from the app tier. How do you implement it across layers? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Defense in depth: routing, SG, NACL and DB auth.

**What I'd do →**
1. Put the DB in private subnets without internet routes
2. Security group allows 5432 only from the app SG
3. Optional NACL allows 5432 from the app subnet CIDRs and ephemeral ports back
4. Require TLS and a DB user per application
5. Test from an unrelated instance, which must time out, and from the app, which must connect

**Why →** Each layer should block on its own so one mistake does not expose the data.

**Interview tip →** Say you test the negative case; people only test that the app works.

</details>

### Q130. A NACL change broke working connections. What commonly goes wrong? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Stateless rules need both directions.

**What I'd do →**
1. Check inbound rule for the service port and the outbound rule for ephemeral ports 1024-65535 (and vice versa for client connections)
2. Check rule numbers: a lower-numbered deny overrides a later allow
3. The default NACL allows everything, a custom one denies everything by default
4. Use VPC Flow Logs to see REJECT with the NACL's subnet ENI
5. Roll back with the last known-good rule set

**Why →** NACLs forget connections; reply packets are evaluated as new traffic.

**Interview tip →** Say you keep NACLs simple and use SGs for logic.

</details>

### Q131. You hit "RulesPerSecurityGroupLimitExceeded". What are your options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default 60 inbound and 60 outbound rules per SG, and 5 SGs per ENI.

**What I'd do →**
1. Replace many CIDR rules with a managed prefix list, which counts as one rule by default (`aws ec2 create-managed-prefix-list`)
2. Reference security groups instead of CIDRs
3. Split rules across several SGs attached to the same ENI
4. Request a quota increase, noting that rules x groups per ENI has a ceiling
5. Clean unused rules with an audit script

**Why →** Prefix lists collapse hundreds of CIDRs into one reusable object.

**Interview tip →** Mention prefix lists are shareable across accounts with RAM.

</details>

### Q132. Set up WireGuard to give an engineer access to a private network. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** WireGuard is a lightweight VPN with a very small config.

**What I'd do →**
1. On the server install `wireguard-tools`, generate keys with `wg genkey | tee server.key | wg pubkey > server.pub`
2. Write `/etc/wireguard/wg0.conf` with `[Interface] Address=10.8.0.1/24 ListenPort=51820 PrivateKey=...` and a `[Peer]` with `PublicKey` and `AllowedIPs = 10.8.0.2/32`
3. Enable forwarding and NAT for the private network
4. Start with `wg-quick up wg0` and check `wg show`
5. Open UDP 51820 in the security group

**Why →** AllowedIPs doubles as routing table and access control for each peer.

**Interview tip →** Say rotating keys and removing a peer are one-line operations, which is why it's popular.

</details>

### Q133. Compare OpenVPN, WireGuard and IPsec for different uses. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Different strengths, maturity and operational cost.

**What I'd do →**
1. IPsec/IKEv2 is the standard for site-to-site between devices from different vendors, and native in clouds
2. OpenVPN works over TCP or UDP 443 and passes restrictive firewalls, with certificate-based auth
3. WireGuard is the simplest and fastest, UDP only, minimal config
4. Choose IPsec for cloud gateways, WireGuard for user or node mesh, and OpenVPN where only 443/TCP is open
5. Verify with `openvpn --version`, `wg --version`, `ipsec --version`

**Why →** Pick by interoperability and network constraints, not by benchmark.

**Interview tip →** Say WireGuard's small codebase is easier to audit.

</details>

### Q134. A site-to-site IPsec tunnel is down. How do you troubleshoot? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Phase 1 (IKE) or phase 2 (IPsec) mismatch, NAT-T or routing issue.

**What I'd do →**
1. Check tunnel status in the AWS console or `aws ec2 describe-vpn-connections --query 'VpnConnections[0].VgwTelemetry'`
2. Compare IKE version, encryption, hash, DH group and lifetimes on both sides
3. Verify UDP 500 and 4500 and ESP are not blocked
4. On Linux gateways read `ipsec statusall` and the logs for `NO_PROPOSAL_CHOSEN` or `AUTH_FAILED`
5. Confirm routes and interesting traffic definitions match

**Why →** Both sides must agree on parameters exactly or negotiation fails before any traffic flows.

**Interview tip →** Say tunnel up with no traffic means routes, not crypto.

</details>

### Q135. How do you make iptables rules survive a reboot and avoid locking yourself out? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rules in memory vanish and mistakes on remote hosts cut your session.

**What I'd do →**
1. Apply rules while keeping an escape: `iptables -A INPUT -p tcp --dport 22 -j ACCEPT` before setting `-P INPUT DROP`
2. Use a timed rollback: `(sleep 120; iptables -F; iptables -P INPUT ACCEPT) &` then cancel it once verified
3. Save with `iptables-save > /etc/iptables/rules.v4` and `netfilter-persistent`
4. Prefer nftables or config management
5. Keep console access

**Why →** Rule order and default policy decide whether you can still log in.

**Interview tip →** Say always allow ESTABLISHED,RELATED and SSH first.

</details>

### Q136. Many SSH brute-force attempts hit a public host. How do you stop them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reduce exposure, then rate-limit.

**What I'd do →**
1. Close the port to the world and use SSM or a VPN
2. If SSH must be public, disable passwords in `/etc/ssh/sshd_config`: `PasswordAuthentication no` and `PermitRootLogin no`
3. Install fail2ban: `fail2ban-client status sshd`
4. Use security group allow lists
5. Review `journalctl -u ssh` and `lastb`

**Why →** Key-only auth makes brute force pointless, and allow lists remove the noise.

**Interview tip →** Say moving SSH to another port is cosmetic, not security.

</details>

### Q137. You need to allow outbound traffic only to specific domains from a private subnet. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Security groups work on IPs and cannot do domain names.

**What I'd do →**
1. Use AWS Network Firewall with a stateful domain list rule group (allow `.github.com`, `.amazonaws.com`, default deny)
2. Or a proxy (Squid) allow-listing domains and set `HTTPS_PROXY`
3. Route private subnets through the firewall endpoint
4. Look at alert and flow logs for blocked domains and tune
5. Note that it inspects SNI, so ECH or domain fronting reduces visibility

**Why →** Domain filtering needs a device that reads SNI or Host headers.

**Interview tip →** Say you start in alert mode before deny.

</details>

### Q138. Docker publishes a port and bypasses your ufw rules. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Docker inserts its own iptables rules ahead of ufw's chains.

**What I'd do →**
1. Understand that `-p 8080:80` creates DNAT in the nat table and ACCEPT in `DOCKER` chain, used before ufw's INPUT rules
2. Check with `iptables -L DOCKER-USER -n`
3. Add restrictions in the `DOCKER-USER` chain: `iptables -I DOCKER-USER -i eth0 ! -s 203.0.113.0/24 -j DROP`
4. Or bind to localhost: `-p 127.0.0.1:8080:80`
5. Use cloud security groups as the real boundary

**Why →** Published container ports are forwarded traffic, which doesn't pass through INPUT.

**Interview tip →** Say you never rely on host firewalls alone for containers.

</details>

### Q139. How do you reach private servers over SSH through a bastion in one command? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** ProxyJump chains SSH connections transparently.

**What I'd do →**
1. Use `ssh -J ec2-user@bastion.example.com ec2-user@10.0.2.15`
2. Persist in `~/.ssh/config`: `Host private-*` with `ProxyJump bastion`
3. Use agent forwarding carefully, prefer ProxyJump so keys never touch the bastion
4. Port forward a DB: `ssh -L 5432:db.internal:5432 bastion`
5. Audit with SSH logs

**Why →** ProxyJump keeps your private key on your laptop.

**Interview tip →** Say SSM port forwarding replaces the bastion entirely.

</details>

### Q140. Design network microsegmentation for a platform with 200 services. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Thousands of SG rules can't be hand-managed; you need identity or label based policy.

**What I'd do →**
1. Use Kubernetes NetworkPolicies with default-deny per namespace, then allow by labels
2. Use Cilium or Calico for L3/L4 and L7 policy, and identity-based rules
3. Add a mesh for mTLS and authorization policies by service identity
4. Outside Kubernetes use SGs referencing SGs per service
5. Observe actual flows with Hubble or flow logs before enforcing
6. Roll out in audit mode

**Why →** Policy by identity survives IP churn; policy by IP does not.

**Interview tip →** Say you observe first, enforce second.

</details>

### Q141. What is your WAF strategy for a public API? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Layers of rules with lifecycle management.

**What I'd do →**
1. Attach WAF at CloudFront or the ALB with managed core, known-bad-inputs and IP reputation groups
2. Add rate-based rules by IP and by API key header
3. Add custom rules for your path and method patterns, and body size limits
4. Log to S3 or Kinesis and analyze with Athena
5. Deploy changes through IaC with a count-mode stage and review false positives weekly

**Why →** A WAF is a tuned control, not a one-time setting.

**Interview tip →** Say bot and API abuse need behavior-based protection beyond signatures.

</details>

### Q142. How do you stop data exfiltration from compromised workloads? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Control egress and detect anomalies.

**What I'd do →**
1. Default-deny egress with explicit allow lists via security groups, Network Firewall or a proxy
2. Force all outbound through a central point with logging
3. Use VPC endpoint policies to restrict S3 to your own buckets
4. Alert on DNS tunneling patterns using Route 53 Resolver DNS Firewall and query logs
5. Use GuardDuty findings for unusual destinations

**Why →** Attackers need an outbound path; remove it, and the stolen data has nowhere to go.

**Interview tip →** Say DNS is the forgotten exfiltration channel.

</details>

### Q143. A SYN flood is saturating a public service. How do you respond? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** SYN flood fills the half-open connection queue.

**What I'd do →**
1. Verify with `ss -s` and `netstat -n | grep -c SYN_RECV`
2. Enable SYN cookies: `sysctl -w net.ipv4.tcp_syncookies=1` and raise `tcp_max_syn_backlog`
3. Place the service behind a CDN, NLB or Shield Advanced which absorb SYN floods
4. Rate-limit at the edge, block sources with NACL or upstream filters
5. Engage the provider's DDoS team

**Why →** Edge infrastructure has the capacity to absorb it, a single host does not.

**Interview tip →** Say host tuning is the last line, not the plan.

</details>

### Q144. The security team wants a TLS-inspecting proxy for all outbound traffic. What breaks and how do you handle it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Man-in-the-middle inspection needs a trusted CA on every client, and some traffic must be exempt.

**What I'd do →**
1. Distribute the proxy CA to endpoints, base images and language truststores (Java, Python `certifi`, Node `NODE_EXTRA_CA_CERTS`)
2. Exempt certificate-pinned apps, financial or medical sites, and package managers that fail
3. Expect errors like `x509: certificate signed by unknown authority`
4. Monitor proxy CPU and latency
5. Keep a bypass list under change control

**Why →** Each runtime keeps its own trust store, which is why inspection projects stall.

**Interview tip →** Say pinned apps and mTLS clients cannot be inspected.

</details>

### Q145. Explain how you would implement IDS/IPS in a cloud network. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Inspection must see traffic; cloud has no span port by default.

**What I'd do →**
1. Use VPC Traffic Mirroring to send traffic to a Suricata or Zeek fleet behind an NLB for detection
2. For inline prevention use Gateway Load Balancer with IPS appliances or AWS Network Firewall (Suricata-compatible rules)
3. Tune signatures and keep rules updated
4. Forward alerts to the SIEM and tie to automation
5. Watch throughput limits and false positives

**Why →** Out-of-band detection is safe and scalable; inline prevention gives control but can break traffic.

**Interview tip →** Say prevention is only worth it once detection is quiet and tuned.

</details>

## Kubernetes Networking

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q146. Explain the Kubernetes networking model in one minute. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The model has three rules that every CNI must satisfy.

**What I'd do →**
1. Every pod gets its own IP
2. Pods can reach all other pods without NAT, across nodes
3. Agents on a node can reach all pods on that node
4. Services give a stable virtual IP and DNS name in front of changing pods
5. Check with `kubectl get pods -o wide` to see pod IPs and nodes

**Why →** A flat pod network lets applications use plain ports and IPs without per-host port mapping.

**Interview tip →** Say the CNI plugin implements the model; Kubernetes only defines it.

</details>

### Q147. ClusterIP, NodePort and LoadBalancer: when do you use each? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** These are the Service types that expose workloads at different scopes.

**What I'd do →**
1. ClusterIP (default) is internal only, used for service-to-service traffic
2. NodePort opens a port (30000-32767) on every node, mostly a building block or for dev
3. LoadBalancer asks the cloud to create an external load balancer pointing at the service
4. For HTTP apps use one Ingress or Gateway in front of several ClusterIP services
5. Check with `kubectl get svc -o wide`

**Why →** Each type builds on the previous: LoadBalancer includes a NodePort and a ClusterIP.

**Interview tip →** Say one LoadBalancer per service gets expensive, so you use Ingress.

</details>

### Q148. How does a pod find another service by name? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** CoreDNS gives every Service a DNS name.

**What I'd do →**
1. The name is `<service>.<namespace>.svc.cluster.local`, and within the same namespace `<service>` is enough
2. The pod's `/etc/resolv.conf` points to the CoreDNS ClusterIP and has search domains
3. Test: `kubectl exec -it pod -- nslookup my-svc.prod`
4. Headless services return pod IPs directly

**Why →** Search domains in resolv.conf allow short names to expand to the full service FQDN.

**Interview tip →** Mention cross-namespace needs at least `svc.namespace`.

</details>

### Q149. Service versus Ingress: what is the difference? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A Service is L4 load balancing inside the cluster; an Ingress is L7 HTTP routing at the edge.

**What I'd do →**
1. A Service gives a stable IP and balances across pods
2. An Ingress defines host and path rules routed to Services
3. An Ingress needs a controller (ingress-nginx, AWS Load Balancer Controller, Traefik) to act
4. Check with `kubectl get ingress` and `kubectl describe ingress app`
5. TLS is configured in `spec.tls`

**Why →** The Ingress object is only configuration; the controller is the actual proxy.

**Interview tip →** Mention Gateway API is the newer, more expressive successor.

</details>

### Q150. What does a CNI plugin do and which ones do you know? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** CNI configures pod networking when a pod starts.

**What I'd do →**
1. Kubelet calls the CNI binary to create the pod's network namespace interface, assign an IP and set routes
2. Popular choices: AWS VPC CNI (pod gets a VPC IP), Calico (BGP or VXLAN, network policy), Cilium (eBPF, observability, policy), Flannel (simple overlay)
3. Check the config in `/etc/cni/net.d/`
4. List the plugin pods with `kubectl -n kube-system get pods`

**Why →** Without a working CNI, nodes stay NotReady and pods remain ContainerCreating.

**Interview tip →** Say the choice affects policy support, scale and cloud integration.

</details>

### Q151. How do you reach a pod from your laptop for debugging without exposing it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** kubectl port-forward tunnels through the API server.

**What I'd do →**
1. Run `kubectl port-forward pod/web-7d4f 8080:80` or `svc/web 8080:80`
2. Open `http://localhost:8080`
3. It is a debugging tool, not for production traffic
4. Use `kubectl exec` or an ephemeral debug container: `kubectl debug -it pod/web --image=nicolaka/netshoot --target=web`

**Why →** The tunnel uses the API server connection, so no extra network path is needed.

**Interview tip →** Say netshoot is your network toolbox in Kubernetes.

</details>

### Q152. By default, can any pod talk to any other pod, and how do you change that? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Kubernetes is open by default; isolation requires NetworkPolicy and a CNI that enforces it.

**What I'd do →**
1. Without policies all pod-to-pod traffic is allowed
2. Create a default-deny: `apiVersion: networking.k8s.io/v1 kind: NetworkPolicy spec: podSelector: {} policyTypes: [Ingress]`
3. Then add allow policies by label
4. Check the CNI supports policy (Calico, Cilium; the VPC CNI needs the network policy agent)
5. Test with `kubectl exec` and `curl` between pods

**Why →** A NetworkPolicy object does nothing unless the CNI enforces it.

**Interview tip →** Say you remember to allow DNS egress when you add default-deny egress.

</details>

### Q153. You must expose a web app on the internet with HTTPS on EKS. What components do you wire up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Typical production path: Ingress controller, certificate, DNS.

**What I'd do →**
1. Install the AWS Load Balancer Controller with IRSA
2. Create an Ingress with `ingressClassName: alb` and annotations `alb.ingress.kubernetes.io/scheme: internet-facing` and `target-type: ip`
3. Attach an ACM certificate with `alb.ingress.kubernetes.io/certificate-arn`
4. Let ExternalDNS create the Route 53 record
5. Verify with `kubectl get ingress` and `curl -I https://app.example.com`

**Why →** IP target type sends traffic directly to pod IPs, skipping node hops.

**Interview tip →** Say you pick the controller by what the cloud provides natively.

</details>

### Q154. Your app needs the real client IP behind a Kubernetes LoadBalancer service. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** kube-proxy SNATs traffic when it forwards across nodes, hiding the client IP.

**What I'd do →**
1. Set `externalTrafficPolicy: Local` on the Service, which sends traffic only to nodes with local pods and keeps the source IP
2. The cloud health check then marks nodes without pods unhealthy automatically
3. Or use PROXY protocol or `X-Forwarded-For` on an L7 LB
4. Verify by logging the remote address in the app
5. Spread pods across nodes with topology spread constraints

**Why →** `Cluster` policy balances evenly but loses the client IP; `Local` keeps it with possible imbalance.

**Interview tip →** Say the trade-off is uneven load.

</details>

### Q155. When do you use a headless service? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** It returns pod IPs instead of a virtual IP.

**What I'd do →**
1. Set `clusterIP: None` in the Service
2. DNS returns A records for each ready pod, and for StatefulSets stable names like `db-0.db.prod.svc.cluster.local`
3. Use it for databases, Kafka or any client-side discovery
4. Check with `nslookup db.prod.svc.cluster.local`
5. Clients must handle multiple records and reconnect on changes

**Why →** No virtual IP means no load balancing by kube-proxy; the client sees the real endpoints.

**Interview tip →** Say StatefulSets need it for stable network identity.

</details>

### Q156. Compare kube-proxy iptables mode and IPVS mode. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both implement Services on each node, with different scaling profiles.

**What I'd do →**
1. iptables mode creates chains of rules per service and endpoint, evaluated linearly, and updates rewrite large rule sets
2. IPVS mode uses kernel hash tables and supports more algorithms (rr, lc, sh) with O(1) lookups at scale
3. Check mode: `kubectl -n kube-system get cm kube-proxy -o yaml | grep mode`
4. List with `ipvsadm -Ln` or `iptables -t nat -L KUBE-SERVICES | head`
5. Cilium can replace kube-proxy with eBPF

**Why →** At thousands of services, linear iptables lookups and full rewrites become slow.

**Interview tip →** Say you move to IPVS or eBPF once the service count is in the thousands.

</details>

### Q157. Isolate namespaces so only the frontend namespace can reach the backend namespace on port 8080. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Needs a NetworkPolicy with namespace and pod selectors.

**What I'd do →**
1. Label namespaces: `kubectl label ns frontend team=frontend`
2. In the backend namespace create a policy: `podSelector: matchLabels: app=api`, ingress from `namespaceSelector: matchLabels: team=frontend`, port 8080
3. Add a default-deny ingress policy
4. Test with `kubectl exec -n frontend deploy/web -- curl -m3 api.backend:8080` and from another namespace to confirm timeout

**Why →** Network policies are additive allow lists on top of a default deny.

**Interview tip →** Say namespaceSelector and podSelector in one `from` entry are ANDed, in separate entries ORed.

</details>

### Q158. Your pods need a fixed outbound IP for a partner's allow list. How do you do it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pod IPs change and node IPs vary, so egress needs a stable exit.

**What I'd do →**
1. Route pod egress through NAT gateways with Elastic IPs, which are stable per AZ
2. With the VPC CNI custom networking SNAT happens at the node; check `AWS_VPC_K8S_CNI_EXTERNALSNAT`
3. For finer control use an egress gateway (Cilium Egress Gateway, Istio egress gateway) pinned to specific nodes with EIPs
4. Provide the partner with the IPs
5. Test with `kubectl run -it --rm curl --image=curlimages/curl -- curl -s ifconfig.me`

**Why →** The source IP the partner sees is whatever NAT sits last in the path.

**Interview tip →** Say you verify with an external echo service.

</details>

### Q159. Why would you add a service mesh and what is the networking cost? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A mesh moves cross-cutting network features into sidecars or node proxies.

**What I'd do →**
1. Gains: mTLS between services, retries and timeouts, traffic shifting, rich L7 metrics
2. Cost: extra proxy hop and CPU or memory per pod, complexity, new failure modes
3. Install Istio or Linkerd and enable injection: `kubectl label ns prod istio-injection=enabled`
4. Verify mTLS with `istioctl x describe pod <pod>`
5. Evaluate ambient or eBPF modes to reduce overhead

**Why →** A mesh trades compute and operational complexity for uniform security and observability.

**Interview tip →** Say you adopt a mesh for a specific problem, not by default.

</details>

### Q160. Compare Ingress and Gateway API. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Gateway API is the role-oriented successor to Ingress.

**What I'd do →**
1. Ingress has one resource and relies on controller-specific annotations for most features
2. Gateway API separates `GatewayClass`, `Gateway` (infra owner) and `HTTPRoute` (app owner)
3. It supports header matching, weighted traffic splitting, TCP and gRPC routes natively
4. Check support with `kubectl get gatewayclass`
5. Migrate gradually since both can coexist

**Why →** Splitting roles lets platform teams own the listener while app teams own routes.

**Interview tip →** Say you choose by controller support and needed features.

</details>

### Q161. What does hostNetwork: true do and why is it risky? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The pod shares the node's network namespace.

**What I'd do →**
1. The pod uses the node IP and ports directly, bypassing the pod network
2. Used by CNI agents, node exporters and some ingress controllers
3. Risks: port conflicts, bypasses NetworkPolicy, can see node traffic
4. Check usage with `kubectl get pods -A -o jsonpath='{range .items[?(@.spec.hostNetwork==true)]}{.metadata.name}{"\n"}{end}'`
5. Use admission policy to restrict

**Why →** It breaks the isolation boundary that namespaces and policies rely on.

**Interview tip →** Say you allow it only for system components.

</details>

### Q162. How do you automate DNS records for Kubernetes Ingress and Services? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** ExternalDNS watches resources and writes DNS records.

**What I'd do →**
1. Install ExternalDNS with a role limited to the hosted zone
2. Annotate the Service or use the Ingress host: `external-dns.alpha.kubernetes.io/hostname: app.example.com`
3. Set `--policy=upsert-only` and `--txt-owner-id=cluster-a` to protect existing records
4. Check `kubectl logs deploy/external-dns` and `dig app.example.com`
5. Use a dedicated subdomain zone for the cluster

**Why →** Records follow the lifecycle of the Kubernetes objects, preventing stale entries.

**Interview tip →** Say TXT ownership records stop two clusters fighting over a name.

</details>

### Q163. How do you size the pod CIDR and service CIDR for a new cluster? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** They are hard to change later and must not overlap with anything reachable.

**What I'd do →**
1. Estimate max nodes x pods per node, for example 500 nodes x 110 pods needs about 55,000 IPs, so a /16 for pods
2. Choose a service CIDR like `10.96.0.0/16` that doesn't overlap with VPC, on-prem or peer ranges
3. With the VPC CNI check subnet capacity instead
4. In kubeadm: `--pod-network-cidr` and `--service-cidr`
5. Record choices in the IPAM

**Why →** Overlaps cause unreachable destinations that are very hard to diagnose later.

**Interview tip →** Say you plan for growth and keep ranges outside corporate space.

</details>

### Q164. Cilium replaces kube-proxy in your cluster. What changes and what do you gain? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** eBPF handles service load balancing in the kernel datapath.

**What I'd do →**
1. Install with `kubeProxyReplacement=true` and verify with `cilium status`
2. Services are translated at socket level or at the TC hook, avoiding iptables and conntrack for service traffic
3. Gain: lower latency, better scalability, native Hubble flow visibility and L7 policy
4. Risks: kernel version requirements and fewer people who know it
5. Validate with `cilium connectivity test`

**Why →** eBPF maps replace linear rule chains, which scales better and exposes flow-level visibility.

**Interview tip →** Say you check kernel version and cloud support before adopting.

</details>

### Q165. How do you connect services across two Kubernetes clusters? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Options range from simple DNS to full multi-cluster meshes.

**What I'd do →**
1. Simple: expose services through internal load balancers and private DNS names
2. Network-level: non-overlapping pod CIDRs plus routing (TGW, peering) or Cilium Cluster Mesh
3. Service-level: Istio multi-cluster or the Multi-Cluster Services API (`ServiceExport`)
4. Keep identity consistent with shared root CA
5. Test failover by killing a cluster's endpoints

**Why →** Cross-cluster calls multiply failure modes, so start with the simplest path that meets the requirement.

**Interview tip →** Say overlapping pod CIDRs are why you plan before the first cluster.

</details>

### Q166. A cluster with 10,000 Services has slow network programming and high CPU in kube-proxy. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** iptables mode does not scale linearly.

**What I'd do →**
1. Confirm with `iptables-save | wc -l` and kube-proxy sync duration metrics
2. Switch to IPVS or eBPF replacement
3. Reduce services through consolidation with Ingress
4. Use EndpointSlices (default) and avoid churn from flapping readiness
5. Tune `conntrack` sizes

**Why →** The cost is per rule, and Services times endpoints grows fast.

**Interview tip →** Say endpoint churn is as important as raw counts.

</details>

### Q167. Pods with the AWS VPC CNI take long to start and sometimes stay in ContainerCreating with "failed to assign an IP address". What is the design fix? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** IP warm pool or subnet exhaustion.

**What I'd do →**
1. Check `kubectl logs -n kube-system ds/aws-node` and `ipamd` messages
2. Verify subnet free IPs and ENI limits per instance type
3. Enable prefix delegation to get /28 prefixes per ENI
4. Tune `WARM_IP_TARGET`, `MINIMUM_IP_TARGET`
5. Use custom networking with secondary CIDR
6. Choose instance types with more ENIs or use Karpenter with right sizing

**Why →** Each instance type caps ENIs and IPs per ENI, so density and subnet size both limit pods.

**Interview tip →** Say max pods formula depends on instance type unless prefix delegation is enabled.

</details>

### Q168. How do you roll out Ingress controller upgrades with zero downtime? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The Ingress controller is your data plane; treat it like production infrastructure.

**What I'd do →**
1. Run at least 3 replicas across AZs with a PodDisruptionBudget
2. Use `maxUnavailable: 0` rolling updates and readiness probes
3. Add `preStop` sleep and graceful shutdown so connections drain
4. Test the new version on a second ingress class and shift via DNS or weighted ALB
5. Watch 5xx and latency during the rollout

**Why →** Connection draining and surge capacity keep in-flight requests alive.

**Interview tip →** Say canary the controller itself, not just the apps behind it.

</details>

### Q169. Design pod egress with centralized inspection and per-team allow lists. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need identity-aware egress per namespace.

**What I'd do →**
1. Use Cilium egress gateway or Istio egress gateway with per-namespace policies
2. Enforce FQDN policies using CiliumNetworkPolicy `toFQDNs`
3. Route egress through a central NAT and firewall with logging
4. Give each team a policy through GitOps pull requests
5. Alert on denied flows with Hubble

**Why →** Pod IPs are ephemeral, so egress rules must follow labels and names, not addresses.

**Interview tip →** Say audit logs of denied flows drive onboarding of new destinations.

</details>

## Performance, MTU, Latency & Tools

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q170. Latency, bandwidth and throughput: what is the difference and why does it matter? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** People mix these up and then buy bandwidth to fix a latency problem.

**What I'd do →**
1. Latency is the time for a packet to travel (RTT in ms)
2. Bandwidth is the link capacity (Mbps or Gbps)
3. Throughput is what you actually achieve, limited by latency, window size, loss and CPU
4. Measure RTT with `ping -c 10 host` and throughput with `iperf3 -c host`
5. A 1 Gbps link with 200 ms RTT still gives slow single-stream transfers

**Why →** TCP needs acknowledgements, so high RTT caps throughput regardless of link speed.

**Interview tip →** Say the formula: throughput is at most window size divided by RTT.

</details>

### Q171. When do you use ping, traceroute and mtr? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each answers a different question.

**What I'd do →**
1. `ping` tests reachability and RTT to one host
2. `traceroute -n host` (or `tracepath`) shows each hop to find where the path breaks or slows
3. `mtr -rwzc 100 host` combines both over time, showing loss and latency per hop
4. Remember many routers deprioritize ICMP, so loss on a middle hop that doesn't continue to the end is usually noise

**Why →** Only loss that carries through to the final hop is real loss.

**Interview tip →** Say you read mtr from the last hop backward.

</details>

### Q172. What is MTU and what is the usual value? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The maximum packet size without fragmentation on a link.

**What I'd do →**
1. Standard Ethernet MTU is 1500 bytes
2. AWS supports 9001 inside a VPC (jumbo frames), but 1500 over the internet and VPN
3. Overlays and tunnels reduce it (VXLAN loses 50 bytes, WireGuard about 60-80)
4. Check with `ip link show eth0` and test with `ping -M do -s 1472 host`

**Why →** If packets exceed the MTU along the path, they're fragmented or dropped.

**Interview tip →** Mention 1472 plus 28 bytes of headers equals 1500.

</details>

### Q173. How do you check what is listening on a port and who is connected? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Everyday server diagnosis.

**What I'd do →**
1. `ss -tulpn` lists listening TCP and UDP sockets with process names
2. `ss -tn state established '( dport = :443 )'` shows active connections
3. `lsof -i :8080` shows the owning process
4. From another host use `nc -zv host 8080` or `curl -v telnet://host:8080`
5. `netstat` is deprecated in favour of `ss`

**Why →** `ss` reads kernel socket tables directly and is faster than netstat.

**Interview tip →** Say listening on 127.0.0.1 versus 0.0.0.0 is the first thing to read.

</details>

### Q174. How do you measure where time goes in a slow HTTP request? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Break the request into DNS, connect, TLS, first byte and transfer.

**What I'd do →**
1. Run `curl -o /dev/null -s -w 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' https://example.com`
2. High dns means resolver trouble
3. High connect means network path or SYN drops
4. High tls means handshake cost
5. High ttfb minus tls means the backend is slow

**Why →** Each phase points to a different layer and a different owner.

**Interview tip →** Say you keep a curl format file for this in your toolkit.

</details>

### Q175. What does tcpdump do and what are the first filters you use? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Packet capture is the final source of truth.

**What I'd do →**
1. Capture on an interface: `sudo tcpdump -ni eth0 host 10.0.1.5 and port 443`
2. Save for Wireshark: `sudo tcpdump -ni any -w /tmp/cap.pcap -c 10000`
3. Show only SYN packets: `tcpdump -ni eth0 'tcp[tcpflags] & tcp-syn != 0'`
4. Use `-nn` to skip DNS and port name lookups
5. Keep captures short and filtered

**Why →** Capturing at the host proves whether a packet really arrived or left.

**Interview tip →** Say you capture on both ends to prove where a packet disappears.

</details>

### Q176. TCP or UDP: how do you choose for an application? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reliability and ordering versus latency and overhead.

**What I'd do →**
1. TCP gives ordered, reliable delivery with congestion control, good for web, databases, APIs
2. UDP has no handshake or retransmit, good for DNS, real-time media, gaming and QUIC (which builds reliability on top)
3. Test UDP reachability with `nc -u -vz host 53`
4. Open the correct protocol in security groups, since TCP rules don't cover UDP

**Why →** TCP trades latency for guarantees; UDP leaves reliability to the application.

**Interview tip →** Say HTTP/3 is reliable but runs on UDP through QUIC.

</details>

### Q177. What does the TCP three-way handshake look like in a capture? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reading SYN, SYN-ACK, ACK in tcpdump helps spot where a connection fails.

**What I'd do →**
1. Client sends SYN, server returns SYN-ACK, client sends ACK
2. In tcpdump look for flags `[S]`, `[S.]`, `[.]`
3. Only `[S]` repeated means packets are dropped or the server is unreachable
4. `[R.]` after `[S]` means the port is closed
5. Capture with `tcpdump -ni eth0 'host 10.0.1.5 and tcp port 22'`

**Why →** The pattern of flags tells you whether the problem is a drop, a reject or an application issue.

**Interview tip →** Say repeated SYN means firewall drop; RST means closed port.

</details>

### Q178. Large uploads over your VPN hang while small requests work. What is likely going on? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic path MTU problem: tunnel overhead reduces effective MTU and ICMP is filtered.

**What I'd do →**
1. Test: `ping -M do -s 1400 host` and reduce size until it passes, finding the real path MTU
2. Check `tracepath host` for `pmtu`
3. Clamp MSS on the tunnel: `iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu`
4. Set interface MTU explicitly, for example 1380 on WireGuard
5. Allow ICMP type 3 code 4 in firewalls

**Why →** Large packets with DF set are dropped silently when ICMP "fragmentation needed" messages are blocked.

**Interview tip →** Say "small works, big hangs" is the signature of MTU.

</details>

### Q179. When do you use jumbo frames and what are the risks? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Jumbo frames cut per-packet overhead but need the whole path to support them.

**What I'd do →**
1. Use 9001 MTU within an AWS VPC and cluster placement groups for storage, HPC or heavy replication
2. Set it with `sudo ip link set dev eth0 mtu 9001`
3. Remember traffic leaving the VPC, over VPN or between regions is limited to 1500 (or 8500 over TGW)
4. Test each path with `ping -M do -s 8973 host`
5. Mismatched MTU yields drops

**Why →** Jumbo only helps if every hop supports it; otherwise path MTU discovery or drops follow.

**Interview tip →** Say you validate end to end rather than assuming.

</details>

### Q180. How do you measure real network bandwidth between two servers? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use a synthetic test with controlled parameters.

**What I'd do →**
1. Start a server: `iperf3 -s`
2. Run the client: `iperf3 -c 10.0.1.20 -P 4 -t 30` (4 parallel streams)
3. Add `-R` for reverse direction and `-u -b 500M` for UDP
4. Compare with the instance type's baseline and burst network limits
5. Test across AZs and regions separately

**Why →** A single TCP stream is limited by window and RTT, parallel streams reveal the pipe capacity.

**Interview tip →** Say cloud instance types have bandwidth caps and burst credits.

</details>

### Q181. Why does a transfer across regions seem slow even with plenty of bandwidth? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bandwidth-delay product: long RTT needs a large TCP window.

**What I'd do →**
1. Measure RTT, for example 180 ms, and target 1 Gbps: BDP is about 22 MB
2. Check window scaling: `sysctl net.ipv4.tcp_window_scaling` and buffers `net.ipv4.tcp_rmem` and `tcp_wmem`
3. Increase max buffers: `sysctl -w net.core.rmem_max=33554432`
4. Use parallel streams, or multipart S3 upload
5. Consider BBR: `sysctl -w net.ipv4.tcp_congestion_control=bbr`

**Why →** A sender can have only a window of unacknowledged data in flight; small windows waste a long pipe.

**Interview tip →** Say BDP equals bandwidth times RTT.

</details>

### Q182. mtr shows 40% loss at hop 5 but 0% at the destination. Is there a problem? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Probably not; routers rate-limit ICMP replies.

**What I'd do →**
1. Look at the last hop first: 0% there means end-to-end delivery is fine
2. Loss that persists on all following hops is real
3. Verify with application-level tests: `curl` timings and `iperf3 -u` for loss
4. Run `mtr -T -P 443 host` to probe with TCP instead of ICMP
5. Check `ss -ti` retransmit counters on the endpoint

**Why →** Intermediate routers deprioritize replies to probes, but forward real traffic normally.

**Interview tip →** Say evidence comes from the endpoints, not from a mid-path router.

</details>

### Q183. How do you spot TCP retransmissions and what do they tell you? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Retransmits show packet loss or congestion.

**What I'd do →**
1. Per host counters: `nstat -az TcpRetransSegs` or `netstat -s | grep -i retrans`
2. Per connection: `ss -ti dst 10.0.1.20` and read `retrans:` and `rtt:`
3. In Wireshark filter `tcp.analysis.retransmission`
4. Rate of retransmits above about 1% is worth investigating
5. Correlate with interface drops: `ip -s link show eth0` and `ethtool -S eth0`

**Why →** Retransmits are TCP telling you the network lost or delayed its packets.

**Interview tip →** Say you check NIC drops and queue overruns before blaming the network.

</details>

### Q184. Your server shows thousands of sockets in TIME_WAIT. Is that a problem? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Usually not, unless it exhausts ephemeral ports on a client.

**What I'd do →**
1. Count states: `ss -s` and `ss -tan state time-wait | wc -l`
2. TIME_WAIT lasts 60 seconds on Linux and protects against late packets
3. The real problem is port exhaustion on a client making many outbound connections to one destination
4. Widen the range: `sysctl -w net.ipv4.ip_local_port_range="10240 65535"`
5. Use keep-alive and connection pooling instead of tweaking `tcp_tw_reuse`

**Why →** TIME_WAIT is a feature, and fewer short connections is the actual fix.

**Interview tip →** Say never enable the removed `tcp_tw_recycle`.

</details>

### Q185. How do you simulate latency and packet loss to test application behavior? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Chaos testing at the network layer.

**What I'd do →**
1. Use `tc netem`: `sudo tc qdisc add dev eth0 root netem delay 100ms 20ms loss 1%`
2. Verify with `ping` and `tc qdisc show dev eth0`
3. Remove with `sudo tc qdisc del dev eth0 root`
4. Use toxiproxy or Chaos Mesh for containers and Kubernetes
5. Run tests against timeouts and retry settings

**Why →** Applications that look fine on a LAN often fail under 100 ms RTT or 1% loss.

**Interview tip →** Say you test timeouts, not only throughput.

</details>

### Q186. How do you capture traffic from a Kubernetes pod without changing the image? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use an ephemeral debug container sharing the pod's network namespace.

**What I'd do →**
1. Run `kubectl debug -it pod/web --image=nicolaka/netshoot --target=web -- bash`
2. Inside run `tcpdump -ni eth0 port 8080 -w /tmp/c.pcap`
3. Copy out with `kubectl cp web:/tmp/c.pcap ./c.pcap -c debugger-xyz`
4. Alternatively use `kubectl sniff` (krew plugin) to stream into Wireshark
5. Capture on the node with `tcpdump -i any` to see the CNI path

**Why →** A shared network namespace gives full visibility without modifying the application image.

**Interview tip →** Say production images shouldn't carry tcpdump.

</details>

### Q187. How do you monitor network health continuously? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Combine active probes and passive metrics.

**What I'd do →**
1. Run Prometheus blackbox exporter for HTTP, TCP, ICMP and DNS probes from several locations
2. Collect node metrics (`node_network_receive_errs_total`, `node_netstat_Tcp_RetransSegs`, conntrack usage)
3. Enable VPC flow logs and ELB access logs for analysis
4. Alert on error rate, latency p99 and certificate expiry
5. Track SLOs for DNS and connection success

**Why →** Probes show what users experience; host metrics show why.

**Interview tip →** Say you probe from outside your own network.

</details>

### Q188. Why does cross-AZ traffic matter for cost and latency? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** AZs are separate data centers in a region with small latency and per-GB charges.

**What I'd do →**
1. Latency within an AZ is typically under 1 ms, cross-AZ about 1 to 2 ms
2. AWS charges for data transferred between AZs in both directions
3. Keep chatty paths (app to cache) in the same AZ with topology-aware routing
4. In Kubernetes use `topology.kubernetes.io/zone` and topology aware routing on Services
5. Check cost with Cost Explorer filtered to `DataTransfer-Regional-Bytes`

**Why →** Spreading for availability is right, but chatty cross-AZ calls cost money and latency.

**Interview tip →** Say you balance availability against locality deliberately.

</details>

### Q189. Your conntrack table is full: "nf_conntrack: table full, dropping packet". What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The kernel tracks every flow and drops new ones at the limit.

**What I'd do →**
1. Check usage: `sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max`
2. Raise: `sysctl -w net.netfilter.nf_conntrack_max=1048576` and the hash size
3. Lower timeouts for idle states like `nf_conntrack_tcp_timeout_time_wait`
4. Reduce flows with keep-alive and pooling
5. Exempt high-volume traffic from tracking with `iptables -t raw -j NOTRACK`

**Why →** Each tracked flow uses memory, and a busy NAT or Kubernetes node hits the cap sooner than expected.

**Interview tip →** Say you alert on conntrack usage above 80 percent.

</details>

### Q190. Users report intermittent slowness only in the evening. How do you hunt for microbursts and congestion? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Averages hide bursts shorter than the monitoring interval.

**What I'd do →**
1. Look at fine-grained metrics: interface drops and `tc -s qdisc show dev eth0` backlog, and `ethtool -S eth0` for `rx_missed` or `tx_dropped`
2. Use one-second sampling (`sar -n DEV 1`) rather than minute averages
3. Check instance bandwidth allowance exceeded with `ethtool -S eth0 | grep allowance_exceeded` on AWS ENA
4. Compare with flow logs to find talkers
5. Add capacity or shape traffic

**Why →** A link can be 20 percent utilized on average and still drop packets in 50 ms bursts.

**Interview tip →** Say ENA allowance counters are the AWS-specific hint.

</details>

### Q191. Tune a Linux server to handle 100,000 concurrent connections. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Think file descriptors, kernel buffers and the accept queue.

**What I'd do →**
1. Raise limits: `ulimit -n 1048576` and `fs.file-max`
2. Tune the accept queue: `net.core.somaxconn=65535` and `net.ipv4.tcp_max_syn_backlog=65535`
3. Widen ports and enable `tcp_syncookies`
4. Set `net.core.netdev_max_backlog`, use multi-queue NICs and RSS/RPS for CPU spread
5. Use epoll-based servers (Nginx, Envoy)
6. Verify with `wrk` or `k6` while watching `ss -s`

**Why →** Each connection costs memory and a descriptor; the defaults assume far lower scale.

**Interview tip →** Say you measure after every change, not guess.

</details>

### Q192. How do you find the cause of a request that is slow only 1 in 1000 times? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tail latency usually comes from retransmits, DNS timeouts or GC, not average load.

**What I'd do →**
1. Add timing breakdown to logs (connect, TLS, TTFB) and trace IDs
2. Check for 1 second or 3 second multiples (TCP SYN retransmit at 1s, DNS timeout at 5s)
3. Capture with tcpdump around the slow event
4. Use `bpftrace` or `tcpretrans` from bcc to log retransmits live
5. Correlate with GC pauses and noisy neighbours

**Why →** Delays in exact multiples of the retransmit or resolver timeouts are a fingerprint.

**Interview tip →** Say 1s, 3s and 5s latencies point at network timeouts, not CPU.

</details>

### Q193. UDP-based real-time service (voice, game) is chosen. How do you handle loss, jitter and NAT? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** No retransmission means the application manages quality.

**What I'd do →**
1. Use RTP or QUIC with forward error correction and jitter buffers
2. Prefer anycast or regional edge so RTT stays low
3. Handle NAT traversal with STUN, TURN and ICE for peer paths
4. Mark DSCP values (EF for voice) where networks honour them
5. Open UDP port ranges and set NLB UDP health checks properly
6. Monitor jitter and loss with probe tools

**Why →** Real-time media prefers a late-lost packet over a delayed one, so the app tolerates loss.

**Interview tip →** Say TURN relays cost bandwidth and are the fallback.

</details>

### Q194. Trace network behavior at scale in Kubernetes without sampling gaps. What do you use? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** eBPF-based observability sees every flow with low overhead.

**What I'd do →**
1. Deploy Cilium with Hubble: `hubble observe --namespace prod --verdict DROPPED`
2. View service dependency maps in Hubble UI
3. Export flow metrics to Prometheus
4. Use `bpftrace` one-liners for node-level debugging like `tcpretrans`
5. Retain flow logs for forensics
6. Compare with the policies in git to find mismatches

**Why →** Kernel-level hooks see flows regardless of application instrumentation.

**Interview tip →** Say Hubble answers "why was this dropped" with a policy name.

</details>

## Network Operations & Change Management

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q195. How do you roll out a security group or firewall change in production safely? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Network changes can take down everything at once and are often hard to see failing.

**What I'd do →**
1. Express the change as code and review it as a pull request showing `terraform plan` output
2. Add the new allow rule first, verify traffic, then remove the old rule
3. Apply in a change window with a rollback command ready (`git revert` and re-apply)
4. Watch flow logs REJECT counts and application error rate during and after
5. Announce in the incident or change channel

**Why →** Adding before removing keeps both paths working, so mistakes show up before an outage.

**Interview tip →** Say you never delete a rule you haven't seen unused in flow logs.

</details>

### Q196. How do you manage VPCs and network resources with Terraform without drift or accidents? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shared network code has a large blast radius.

**What I'd do →**
1. Keep network in its own state, with remote backend and locking (S3 plus DynamoDB or native S3 locking)
2. Use a reviewed module for VPC, subnets and routes with `cidrsubnet`
3. Add `prevent_destroy` on VPCs and TGWs
4. Run `terraform plan` on a schedule to detect drift
5. Restrict who can apply via CI roles

**Why →** A small, isolated state limits what one bad apply can destroy.

**Interview tip →** Say you separate network from application stacks for blast radius reasons.

</details>

### Q197. A route table was changed by mistake and production lost internet access. How do you recover and learn? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** First restore, then find the cause.

**What I'd do →**
1. Find the change: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=ReplaceRoute`
2. Restore the route (`aws ec2 create-route --route-table-id rtb-0abc --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-0abc`)
3. Verify from an instance with `curl -sI https://example.com`
4. Reconcile with Terraform to remove drift
5. Write a post-incident review and restrict console edits with SCPs or IAM

**Why →** CloudTrail records who changed what and when, which makes rollback exact.

**Interview tip →** Say manual console fixes must be backported to code immediately.

</details>

### Q198. Data transfer charges are 30% of the AWS bill. How do you reduce them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The costs come from a few patterns: NAT, cross-AZ, cross-region and internet egress.

**What I'd do →**
1. Break down with Cost Explorer grouped by usage type, and flow logs for top talkers
2. Add gateway and interface endpoints to bypass NAT
3. Keep chatty services in one AZ or use topology aware routing
4. Put a CDN in front of public egress, which is often cheaper per GB
5. Compress, cache and use private links between regions only when needed

**Why →** Know which of the four categories is the bill before changing anything.

**Interview tip →** Say measure first; most savings come from one or two top flows.

</details>

### Q199. What is your checklist for onboarding a new office to your cloud network over VPN? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** It's a repeatable process that should be a template.

**What I'd do →**
1. Allocate a unique CIDR from IPAM, no overlap with existing sites
2. Create the customer gateway, VPN connection and two tunnels, with BGP where possible
3. Add routes to the VPC or TGW route tables and security rules for the office CIDR
4. Configure split DNS to private zones
5. Test with ping, traceroute, DNS and an application, then monitor tunnel state and set alarms

**Why →** A template avoids snowflake configurations and forgotten steps such as DNS.

**Interview tip →** Say you document the tunnel parameters, because the other side's config is rarely in your repo.

</details>

### Q200. Design network disaster recovery for the loss of an entire region. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** It covers entry traffic, data and connectivity, and it must be tested.

**What I'd do →**
1. Pre-build a standby VPC and infrastructure in a second region with non-overlapping CIDRs, connected by TGW peering
2. Use Route 53 health checks with failover records or Global Accelerator to move users
3. Replicate data with cross-region replication
4. Keep DNS, certificates and secrets available in both regions
5. Define RTO and RPO per service and run game-day failover tests twice a year

**Why →** An untested failover plan is a hypothesis; the dependency you forgot shows up during the real event.

**Interview tip →** Say the first thing that breaks is a hidden dependency on the failed region, such as a single-region DNS or IAM endpoint.

</details>

<!-- DEVOPS-SHACK:FOOTER -->
---

<div align="center">

### 🚀 Made with ❤️ by [DevOps Shack](https://www.youtube.com/@devopsshack)

**Learn DevOps, DevSecOps, Cloud and Kubernetes the practical way.**

[▶️ YouTube — DevOps Shack](https://www.youtube.com/@devopsshack) · [▶️ Hindi — @shackverse](https://www.youtube.com/@shackverse) · [🎓 devopsshack.com](https://devopsshack.com) · [📸 Instagram @devopsshack](https://www.instagram.com/devopsshack)

[🏠 Back to Home](../README.md) · [⭐ Star the original repo](https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack)

<sub>© DevOps Shack · Licensed CC BY-NC-SA 4.0 · If you found this copy anywhere other than <a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack">github.com/jaiswaladi246</a>, the original and latest version is maintained by DevOps Shack.</sub>

</div>
<!-- /DEVOPS-SHACK:FOOTER -->
