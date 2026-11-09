# Networking — Core Concepts

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

> Core networking concepts, internals, comparisons and "what happens when…" for DevOps interviews · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Networking](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Network Models, TCP, UDP & Ports](#network-models-tcp-udp--ports) — Q1–Q25
- [IP Addressing, CIDR, Subnetting & Routing](#ip-addressing-cidr-subnetting--routing) — Q26–Q53
- [NAT, Firewalls & VPN](#nat-firewalls--vpn) — Q54–Q73
- [DNS](#dns) — Q74–Q104
- [HTTP, TLS & Certificates](#http-tls--certificates) — Q105–Q136
- [Load Balancers, Proxies & CDNs](#load-balancers-proxies--cdns) — Q137–Q160
- [Command-Line Tools & Diagnostics](#command-line-tools--diagnostics) — Q161–Q180
- [Kubernetes, Containers, MTU & Performance Basics](#kubernetes-containers-mtu--performance-basics) — Q181–Q200

---

## Network Models, TCP, UDP & Ports

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What are the layers of the OSI model, and why does a DevOps engineer care? 🟢

<details>
<summary><b>Answer</b></summary>

Seven layers: Physical, Data Link, Network, Transport, Session, Presentation, Application. In practice we use it as a shared vocabulary for where a problem lives: L2 is MAC/switching/VLANs, L3 is IP/routing, L4 is TCP/UDP ports, L7 is HTTP/DNS/TLS-aware behavior.

When someone says "L4 load balancer" or "L7 WAF" or "security groups are L3/L4", they are telling you what the device can see. An NLB sees IPs and ports only; an ALB can route on host header and path.

</details>

### Q2. How does the TCP/IP model map to the OSI model? 🟢

<details>
<summary><b>Answer</b></summary>

TCP/IP has four layers: Link (OSI 1–2), Internet (OSI 3), Transport (OSI 4) and Application (OSI 5–7). The Internet protocol suite is what is actually implemented; OSI is a teaching model.

Session and presentation concerns (TLS, encoding, session handling) live inside the application or its libraries in the real stack, which is why TLS is awkwardly "between" L4 and L7.

</details>

### Q3. What is the difference between TCP and UDP? 🟢

<details>
<summary><b>Answer</b></summary>

TCP is connection-oriented, reliable and ordered: handshake, sequence numbers, acknowledgements, retransmission, flow and congestion control. UDP is connectionless and best-effort: no handshake, no retransmission, no ordering, just a datagram with source and destination port.

Use TCP when you need every byte in order (HTTP/1.1, HTTP/2, SSH, databases). Use UDP when latency matters more than perfection or the application handles loss itself (DNS, VoIP, QUIC, WireGuard, syslog, metrics via StatsD).

</details>

### Q4. Describe the TCP three-way handshake. 🟢

<details>
<summary><b>Answer</b></summary>

Client sends SYN with its initial sequence number, server replies SYN-ACK with its own sequence number and acknowledges the client's, client sends ACK. After that the connection is ESTABLISHED on both sides and data can flow.

```
Client -> Server : SYN, seq=x
Server -> Client : SYN, ACK, seq=y, ack=x+1
Client -> Server : ACK, ack=y+1
```
It costs one full round trip before the first byte of data (two with TLS 1.2 on top), which is why connection reuse matters.

</details>

### Q5. How does TCP close a connection, and what is TIME_WAIT? 🟡

<details>
<summary><b>Answer</b></summary>

A graceful close is four segments: each side sends FIN and the other ACKs it, so each direction closes independently. The side that closes first enters TIME_WAIT for 2×MSL (60 seconds on Linux) to make sure the last ACK arrived and stale segments from the old connection die out.

Thousands of TIME_WAIT sockets are normal on a busy client or proxy. They only hurt if you run out of ephemeral ports, which you see as `Cannot assign requested address`. The fix is connection pooling or keep-alive, not blindly enabling `tcp_tw_reuse` everywhere.

</details>

### Q6. What is a port, and what are well-known, registered and ephemeral ranges? 🟢

<details>
<summary><b>Answer</b></summary>

A port is a 16-bit number that lets the OS demultiplex traffic to a process. 0–1023 are well-known (80, 443, 22, 53), 1024–49151 are registered, and 49152–65535 are the IANA dynamic range. Linux defaults ephemeral ports to 32768–60999.

```bash
sysctl net.ipv4.ip_local_port_range
```
A connection is identified by the 5-tuple: protocol, source IP, source port, destination IP, destination port.

</details>

### Q7. Which ports should you know by heart for DevOps work? 🟢

<details>
<summary><b>Answer</b></summary>

22 SSH, 53 DNS, 80 HTTP, 443 HTTPS, 25/587 SMTP, 123 NTP, 3306 MySQL, 5432 PostgreSQL, 6379 Redis, 27017 MongoDB, 9200 Elasticsearch, 2379/2380 etcd, 6443 Kubernetes API, 10250 kubelet, 9090 Prometheus, 3000 Grafana, 8080 common app/Jenkins, 3389 RDP, 51820 WireGuard, 1194 OpenVPN.

Interviewers ask these when you configure security groups, so also remember which are TCP versus UDP: DNS uses both 53/udp and 53/tcp.

</details>

### Q8. What is the difference between a socket, a connection and a port? 🟢

<details>
<summary><b>Answer</b></summary>

A socket is an endpoint: an IP plus a port plus protocol, represented as a file descriptor in the process. A listening socket waits on one port; each accepted connection gets its own new socket, but they all share the same local port and are distinguished by the remote IP and port.

That is why one server on port 443 can hold hundreds of thousands of connections. The limits are file descriptors, memory and conntrack, not the number of ports.

</details>

### Q9. How does TCP flow control differ from congestion control? 🟡

<details>
<summary><b>Answer</b></summary>

Flow control protects the receiver: it advertises a receive window (rwnd) telling the sender how much unacknowledged data it can have in flight. Congestion control protects the network: the sender keeps a congestion window (cwnd) and grows or shrinks it based on loss or delay signals.

The sender's effective window is min(rwnd, cwnd). Slow start doubles cwnd per RTT until a threshold, then congestion avoidance grows it linearly.

</details>

### Q10. Compare TCP congestion control algorithms such as CUBIC and BBR. 🔴

<details>
<summary><b>Answer</b></summary>

CUBIC is loss-based: it grows the window until packets drop, then backs off. It is Linux's default and fine for most traffic, but it fills router buffers (bufferbloat) and reacts badly to random loss on long paths. BBR is model-based: it estimates bottleneck bandwidth and minimum RTT and paces to that, so it keeps queues short and holds throughput on lossy, high-latency links.

```bash
sysctl net.ipv4.tcp_available_congestion_control
sysctl -w net.core.default_qdisc=fq net.ipv4.tcp_congestion_control=bbr
```
We have seen BBR help on cross-region and user-facing edge servers; it can be unfair to CUBIC flows sharing a bottleneck, so test before rolling out.

</details>

### Q11. What are TCP sequence numbers and acknowledgements used for? 🟡

<details>
<summary><b>Answer</b></summary>

Every byte in a TCP stream has a sequence number. The sender labels each segment with the number of its first byte, and the receiver acknowledges the next byte it expects (cumulative ACK). Gaps trigger duplicate ACKs, and three duplicate ACKs trigger fast retransmit without waiting for the timer.

Initial sequence numbers are randomized to stop blind spoofing and to avoid confusion with segments from older connections.

</details>

### Q12. What does a TCP RST mean and when do you see it? 🟢

<details>
<summary><b>Answer</b></summary>

RST abruptly aborts a connection. You see it when you connect to a port with no listener (the kernel answers SYN with RST, giving `Connection refused`), when a firewall or load balancer rejects or drops state, when an application closes a socket with unread data, or when an idle connection exceeds a middlebox timeout and the next packet arrives.

`Connection reset by peer` in app logs usually means the other side or something in the path sent RST.

</details>

### Q13. What is the difference between "connection refused", "connection timed out" and "no route to host"? 🟡

<details>
<summary><b>Answer</b></summary>

Refused: the packet reached a host and the kernel replied RST because nothing listens on that port. Timed out: no reply at all, typically a firewall or security group silently dropping, or a dead host. No route to host: the local routing table has no path, or an ICMP host-unreachable came back.

```bash
nc -zv 10.0.1.5 5432
```
Refused means the network path works and the service is the problem. Timeout means suspect filtering first.

</details>

### Q14. What are the TCP connection states you see in ss output? 🟡

<details>
<summary><b>Answer</b></summary>

LISTEN, SYN-SENT, SYN-RECV, ESTABLISHED, FIN-WAIT-1/2, CLOSE-WAIT, LAST-ACK, CLOSING and TIME-WAIT. The ones that signal trouble: many SYN-RECV suggests a SYN flood or a full accept queue, many CLOSE-WAIT means the application is not closing sockets after the peer did (a bug in your code), many TIME-WAIT on one host is usually benign.

```bash
ss -tan state close-wait | wc -l
```

</details>

### Q15. What is the TCP listen backlog and what happens when it overflows? 🔴

<details>
<summary><b>Answer</b></summary>

Two queues per listening socket: the SYN queue for half-open connections and the accept queue for completed handshakes waiting for `accept()`. The accept queue size is min(backlog argument, `net.core.somaxconn`). When it is full, the kernel drops the final ACK or the SYN, and clients see slow connects or timeouts.

```bash
ss -ltn        # Recv-Q on LISTEN = current accept queue, Send-Q = max
nstat -az TcpExtListenOverflows
```

</details>

### Q16. What is a SYN flood and how do you mitigate it? 🟡

<details>
<summary><b>Answer</b></summary>

An attacker sends many SYNs with spoofed sources and never completes the handshake, exhausting the SYN queue. Linux mitigates with SYN cookies, which encode state in the sequence number so no queue entry is needed (`net.ipv4.tcp_syncookies=1`).

At scale you mitigate upstream: cloud providers' DDoS protection (AWS Shield, Cloudflare), rate limits at the edge, and proxies like an NLB or ALB that terminate TCP before it reaches your instances.

</details>

### Q17. What is UDP used for and why is it "unreliable"? 🟢

<details>
<summary><b>Answer</b></summary>

Unreliable means the protocol itself gives no delivery, order or duplicate guarantees. A datagram is sent once; if it is dropped, nobody retries unless the application does. That makes it cheap and low latency.

Common uses: DNS queries, DHCP, NTP, syslog, VoIP and video, game traffic, WireGuard, and QUIC (which builds reliability on top of UDP in user space).

</details>

### Q18. What is a TCP window scale and why does it matter on fast links? 🔴

<details>
<summary><b>Answer</b></summary>

The original 16-bit window caps at 64 KB, so throughput is limited to window / RTT. On a 100 ms link that is about 5 Mbit/s. The window scale option (RFC 7323) multiplies the window by 2^N, negotiated in the SYN, allowing windows of up to about 1 GB.

Bandwidth-delay product is the number to remember: a 1 Gbit/s link at 80 ms RTT needs about 10 MB in flight. Middleboxes that strip TCP options silently cap your throughput.

</details>

### Q19. Explain TCP Fast Open and why it did not become universal. 🔴

<details>
<summary><b>Answer</b></summary>

TFO lets a client send data in the SYN on repeat connections using a cookie obtained in an earlier handshake, saving one RTT. It needs support on both ends and enabling via sysctl and socket option.

It stalled because some middleboxes drop SYNs carrying data, replayable data in SYN is a security problem for non-idempotent requests, and QUIC/TLS 1.3 0-RTT solved the same latency problem at a higher layer.

</details>

### Q20. What is the difference between unicast, multicast and broadcast? 🟢

<details>
<summary><b>Answer</b></summary>

Unicast is one sender to one receiver. Broadcast is one to every host in the L2 domain (ARP requests, DHCP discover). Multicast is one to a subscribed group (224.0.0.0/4), used by routing protocols and some discovery. Anycast is a fourth pattern: the same IP advertised from many places and routed to the nearest.

Cloud VPCs mostly block broadcast and multicast, which affects tools like keepalived/VRRP and some clustering software.

</details>

### Q21. What is ARP and how does it relate to the data link layer? 🟡

<details>
<summary><b>Answer</b></summary>

ARP resolves an IPv4 address to a MAC address on the local subnet. The host broadcasts "who has 10.0.1.5?" and the owner replies with its MAC; the answer is cached in the neighbor table.

```bash
ip neigh show
arping -I eth0 10.0.1.5
```
Off-subnet traffic is sent to the gateway's MAC instead. In cloud VPCs ARP is answered by the virtual network, so you will not see real L2 behavior, and gratuitous ARP tricks for floating IPs generally do not work.

</details>

### Q22. What is a MAC address and how is it different from an IP address? 🟢

<details>
<summary><b>Answer</b></summary>

A MAC is a 48-bit hardware-level identifier used within a single L2 network and rewritten at every router hop. An IP is a logical L3 address used end to end across networks. The MAC says which interface on this segment, the IP says which host in the internetwork.

Containers and VMs get virtual MACs; changing MACs is rarely meaningful in cloud networks, but MAC allow-lists on a physical switch still exist in on-prem environments.

</details>

### Q23. What are VLANs and how do they work? 🟡

<details>
<summary><b>Answer</b></summary>

A VLAN splits one physical switch fabric into multiple isolated L2 broadcast domains by adding an 802.1Q tag (a 12-bit ID, 1–4094) to Ethernet frames. Access ports carry one untagged VLAN; trunk ports carry many tagged VLANs. Traffic between VLANs needs a router (or L3 switch).

Cloud analogues are subnets and VPCs; VLANs show up when you work with on-prem data centers, Direct Connect or ExpressRoute private VIFs, and bare-metal Kubernetes.

</details>

### Q24. What happens at each layer when you curl https://example.com? 🔴

<details>
<summary><b>Answer</b></summary>

App: curl resolves the name (DNS over UDP/53, possibly via systemd-resolved), builds an HTTP request. Transport: TCP handshake to port 443. Then TLS: ClientHello with SNI, certificate validation, key exchange (1 RTT for TLS 1.3). Network: IP packets routed via the default gateway, NAT possibly at the edge. Link: ARP for the gateway MAC, Ethernet framing.

The server replies in reverse. Seeing this end to end with `curl -v --trace-time` and `tcpdump` is how you decide which layer an outage belongs to.

</details>

### Q25. What is the difference between a hub, a switch and a router? 🟢

<details>
<summary><b>Answer</b></summary>

A hub repeats every frame to all ports (dead hardware today). A switch learns MAC addresses and forwards frames only to the right port within a LAN. A router forwards packets between different IP networks based on its routing table.

Modern L3 switches do both; in AWS the VPC router and virtual switching are handled by the Nitro hypervisor and you only configure route tables.

</details>

## IP Addressing, CIDR, Subnetting & Routing

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. What is an IPv4 address and how is it structured? 🟢

<details>
<summary><b>Answer</b></summary>

A 32-bit number written as four octets (192.168.10.25). It has a network part and a host part, and the subnet mask or prefix length says where the split is. 192.168.10.25/24 means the first 24 bits identify the network and the last 8 identify the host.

There are about 4.3 billion addresses, which is why NAT and IPv6 exist.

</details>

### Q27. What are the private IPv4 ranges (RFC 1918)? 🟢

<details>
<summary><b>Answer</b></summary>

10.0.0.0/8, 172.16.0.0/12 (172.16.0.0–172.31.255.255) and 192.168.0.0/16. They are not routable on the public internet and are reused by everyone, which is why overlapping CIDRs between VPCs or with on-prem block peering and VPNs.

Also remember 100.64.0.0/10 (carrier-grade NAT, often used for EKS secondary CIDRs) and 169.254.0.0/16 (link-local; 169.254.169.254 is the cloud metadata endpoint).

</details>

### Q28. What is CIDR notation? 🟢

<details>
<summary><b>Answer</b></summary>

Classless Inter-Domain Routing writes an address with a prefix length: 10.0.0.0/16 means a 16-bit network prefix and 16 host bits, so 65,536 addresses. It replaced the old class A/B/C scheme and lets you size networks to need and aggregate routes.

Quick sizes: /32 = 1, /30 = 4, /28 = 16, /24 = 256, /20 = 4,096, /16 = 65,536 addresses.

</details>

### Q29. How many usable hosts are in a /24, and why minus two? 🟢

<details>
<summary><b>Answer</b></summary>

A /24 has 256 addresses; traditionally the network address (.0) and broadcast (.255) are not assignable, leaving 254. In AWS the VPC reserves five per subnet: network, VPC router, DNS, future use, and broadcast, so a /24 gives 251 usable IPs.

Azure also reserves five and GCP reserves four, so check the cloud's rule before you size small subnets.

</details>

### Q30. Subnet 10.0.0.0/16 into /24 subnets. How many do you get and what are the first three? 🟡

<details>
<summary><b>Answer</b></summary>

Going from /16 to /24 borrows 8 bits, so 2^8 = 256 subnets of 256 addresses each: 10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24, … 10.0.255.0/24.

For a /20 carve-out you step by 16 in the third octet: 10.0.0.0/20, 10.0.16.0/20, 10.0.32.0/20. The block size is 2^(32 − prefix) addresses, and subnets must start at a multiple of the block size.

</details>

### Q31. Given 172.16.37.200/21, what is the network address, broadcast and range? 🟡

<details>
<summary><b>Answer</b></summary>

/21 means a block of 8 in the third octet (255.255.248.0). 37 falls in the 32–39 block, so the network is 172.16.32.0, broadcast is 172.16.39.255, and the usable range is 172.16.32.1–172.16.39.254.

```bash
python3 -c "import ipaddress as i; n=i.ip_network('172.16.37.200/21',strict=False); print(n, n.broadcast_address)"
```

</details>

### Q32. What is a subnet mask and what is the default gateway? 🟢

<details>
<summary><b>Answer</b></summary>

The mask (255.255.255.0 for /24) tells a host which destination addresses are on-link. Anything inside the subnet is reached directly via ARP; anything else is sent to the default gateway, the router on the local subnet that forwards it onward.

If a host has the wrong mask or gateway you get the classic symptom: it reaches some machines but not others.

</details>

### Q33. What is a routing table and how does longest prefix match work? 🟡

<details>
<summary><b>Answer</b></summary>

A routing table maps destination prefixes to a next hop or interface. The router picks the most specific matching route: a /24 beats a /16 beats 0.0.0.0/0. Ties are broken by administrative distance or metric.

```bash
ip route show
ip route get 10.0.2.15      # shows which route and source IP would be used
```
In AWS the same rule applies in VPC route tables: the more specific route wins, and a local route for the VPC CIDR always exists and cannot be overridden by a less specific route.

</details>

### Q34. What is a default route? 🟢

<details>
<summary><b>Answer</b></summary>

0.0.0.0/0 (and ::/0 for IPv6) is the route of last resort, matched when nothing more specific matches. On a host it points to the gateway; in a VPC it points to an internet gateway (public subnet), NAT gateway (private subnet) or a transit gateway/firewall appliance for centralized egress.

A missing default route is why an instance can reach the VPC but not the internet.

</details>

### Q35. What is the difference between static and dynamic routing? 🟡

<details>
<summary><b>Answer</b></summary>

Static routes are manually configured and predictable, ideal for small, stable topologies (a few VPC route table entries). Dynamic routing uses protocols that exchange routes and react to failures: OSPF inside an organization, BGP between organizations and to cloud on-ramps.

AWS Direct Connect and Site-to-Site VPN with a virtual gateway or TGW use BGP so that failover and route advertisement happen without humans editing tables.

</details>

### Q36. How does BGP work at a high level, and where do you meet it as a DevOps engineer? 🔴

<details>
<summary><b>Answer</b></summary>

BGP is a path-vector protocol where autonomous systems (ASNs) advertise IP prefixes to peers over TCP/179. Routers choose paths using attributes: local preference, AS path length, MED and others. It is policy driven rather than shortest-path.

You meet it in Direct Connect/ExpressRoute/Cloud Interconnect, VPN with dynamic routing, anycast setups (advertising the same prefix from multiple sites), and Kubernetes CNIs like Calico or Cilium that peer with top-of-rack switches to announce pod CIDRs.

</details>

### Q37. What is the difference between an internet gateway, NAT gateway and egress-only gateway in AWS? 🟡

<details>
<summary><b>Answer</b></summary>

An internet gateway gives bidirectional internet access to resources with public IPs, and is what makes a subnet public. A NAT gateway lets private instances initiate outbound IPv4 connections while blocking inbound. An egress-only internet gateway does the same for IPv6, where addresses are globally routable and NAT is not used.

A subnet is public or private only because of its route table, not because of a label.

</details>

### Q38. What does IPv6 look like and why is it needed? 🟢

<details>
<summary><b>Answer</b></summary>

A 128-bit address in hex, written like 2001:db8:abcd:12::1. It exists because IPv4 is exhausted. A typical subnet is a /64, and each host gets a globally unique address, so NAT is not needed; security comes from firewalls and security groups instead.

Leading zeros can be dropped and one run of zero groups collapsed with `::`. Dual-stack (IPv4 and IPv6 together) is the realistic production approach.

</details>

### Q39. What are IPv6 addresses types you should recognize? 🟡

<details>
<summary><b>Answer</b></summary>

Global unicast (2000::/3), unique local (fc00::/7, the equivalent of RFC 1918), link-local (fe80::/10, always present on an interface and used for neighbor discovery), multicast (ff00::/8) and loopback (::1). There is no broadcast; ARP is replaced by Neighbor Discovery (NDP) over ICMPv6.

ICMPv6 must not be blocked wholesale: it carries NDP and Packet Too Big messages that path MTU discovery depends on.

</details>

### Q40. What is DHCP and what does a lease contain? 🟢

<details>
<summary><b>Answer</b></summary>

DHCP hands out network configuration automatically using the DORA exchange: Discover, Offer, Request, Acknowledge. A lease includes IP, mask, default gateway, DNS servers, lease time, and optionally domain search list and NTP servers.

In AWS the VPC DHCP option set provides DNS servers and domain name. Wrong `search` domains from DHCP are a quiet cause of slow name lookups.

</details>

### Q41. What is the loopback address and what is 0.0.0.0? 🟢

<details>
<summary><b>Answer</b></summary>

127.0.0.1 (the whole 127.0.0.0/8) is loopback: traffic never leaves the host. `localhost` resolves to it. 0.0.0.0 as a listen address means "all IPv4 interfaces", while as a destination in a route it means default.

Common bug: an app bound to 127.0.0.1 inside a container is unreachable from outside the container; it must bind to 0.0.0.0.

</details>

### Q42. What is link-local 169.254.169.254? 🟡

<details>
<summary><b>Answer</b></summary>

The instance metadata service address on AWS, Azure and GCP. It is reachable only from the instance itself, serves instance identity and temporary IAM credentials, and is why SSRF vulnerabilities are so dangerous in cloud workloads.

On AWS, require IMDSv2 (session token, with hop limit 1 for containers if possible):
```bash
aws ec2 modify-instance-metadata-options --instance-id i-0abc --http-tokens required --http-put-response-hop-limit 1
```

</details>

### Q43. What is a supernet and what is route summarization? 🟡

<details>
<summary><b>Answer</b></summary>

Summarization (aggregation) replaces several contiguous prefixes with one larger prefix: 10.0.0.0/24 through 10.0.3.0/24 collapse to 10.0.0.0/22. It keeps routing tables small and hides instability behind a stable summary.

It only works if your address plan is hierarchical. That is the real reason we allocate VPC CIDRs per region and per environment from a clean block instead of random ranges.

</details>

### Q44. How do you check which interface and IP your machine will use to reach a destination? 🟢

<details>
<summary><b>Answer</b></summary>

Use `ip route get`. It reports the next hop, outgoing device and chosen source address.

```bash
ip route get 8.8.8.8
# 8.8.8.8 via 10.0.1.1 dev eth0 src 10.0.1.23 uid 1000
```
Combine with `ip -br addr` for a one-line-per-interface view.

</details>

### Q45. How does a host decide whether a destination is local or remote? 🟡

<details>
<summary><b>Answer</b></summary>

It ANDs the destination IP with its own mask and compares to its own network. If they match, the destination is on-link and the host ARPs for it directly. If not, the host sends the frame to the default gateway's MAC address with the original destination IP intact.

The IP header stays end to end (apart from NAT); the Ethernet header is rebuilt at each hop.

</details>

### Q46. What is TTL and how do traceroute and loops relate to it? 🟢

<details>
<summary><b>Answer</b></summary>

Every IP packet has a Time To Live (hop limit in IPv6) decremented by each router; at zero the router discards it and sends ICMP Time Exceeded. This stops routing loops from circulating forever. Traceroute sends probes with TTL 1, 2, 3… and records which router answers each time.

Typical initial TTLs are 64 (Linux), 128 (Windows) and 255 (network gear), which lets you guess the OS or number of hops from a received TTL.

</details>

### Q47. What is ICMP and which messages matter operationally? 🟡

<details>
<summary><b>Answer</b></summary>

ICMP is the control and diagnostic companion of IP. Echo request/reply is ping; Destination Unreachable (including "fragmentation needed") tells senders about failures; Time Exceeded powers traceroute; Redirect and others are mostly disabled.

Blocking all ICMP breaks Path MTU Discovery and produces mysterious hangs on large transfers. Allow at least type 3 code 4 (fragmentation needed) in security groups and firewalls.

</details>

### Q48. What is anycast and how is it different from unicast? 🟡

<details>
<summary><b>Answer</b></summary>

Anycast announces the same IP prefix from several locations; routing delivers each client to the topologically nearest one. It gives low latency and built-in failover: if one site withdraws the route, traffic shifts.

Public DNS resolvers (8.8.8.8, 1.1.1.1), root servers, CDNs and AWS Global Accelerator rely on it. It suits stateless or short-lived UDP/TCP flows; long TCP connections can break if routing flips mid-session.

</details>

### Q49. How would you design an IP address plan for a multi-account, multi-region AWS organization? 🔴

<details>
<summary><b>Answer</b></summary>

Pick one non-overlapping supernet from RFC 1918 (for example 10.0.0.0/8 or a slice of it), allocate a /16 or /18 per VPC from a central registry, group by region and environment so routes summarize (10.16.0.0/12 for eu-west-1, and so on), and leave growth room. Reserve ranges for on-prem and partners.

Track allocations in AWS IPAM or a Git-managed file; overlapping CIDRs discovered after peering are painful because VPC CIDRs cannot be changed, only extended with secondary blocks.

</details>

### Q50. What is the difference between VPC peering and Transit Gateway? 🟡

<details>
<summary><b>Answer</b></summary>

Peering is a one-to-one, non-transitive connection between two VPCs: A–B and B–C does not give A–C. It is cheap and has no bandwidth bottleneck. Transit Gateway is a regional hub where many VPCs and VPNs attach, with route tables that control reachability, and it supports transitivity.

Past a handful of VPCs, full-mesh peering becomes unmanageable and TGW wins, at the cost of per-attachment and per-GB charges.

</details>

### Q51. What is a bastion host and how does it relate to routing? 🟢

<details>
<summary><b>Answer</b></summary>

A hardened host in a public subnet used as the single SSH entry point to private instances. It works because it has a public IP and a route to the internet gateway while also having a local route into the private subnets.

Today we prefer SSM Session Manager or EC2 Instance Connect Endpoint, which need no inbound ports and give audit logs.

</details>

### Q52. What is policy-based routing and when would you use it? 🔴

<details>
<summary><b>Answer</b></summary>

Normal routing uses only the destination address. Policy-based routing chooses a table by source IP, mark, interface or other fields using `ip rule`. Typical cases: multi-homed servers with two NICs where replies must leave via the interface the request arrived on, or steering certain traffic through a VPN.

```bash
ip rule add from 10.0.2.10 table 100
ip route add default via 10.0.2.1 dev eth1 table 100
```
Asymmetric routing on multi-NIC instances is the usual reason you need it.

</details>

### Q53. What is asymmetric routing and why does it break stateful devices? 🔴

<details>
<summary><b>Answer</b></summary>

Asymmetric routing means the request and reply take different paths. Stateful firewalls, NAT gateways and load balancers only see half the conversation, never see the handshake complete, and drop the reply as invalid.

It shows up with two ISPs, multi-NIC hosts, TGW appliance-mode misconfiguration (traffic returning through a different AZ's firewall) and Direct Connect plus VPN failover. Fix with symmetric design: appliance mode on TGW, source-based policy routing, or SNAT at the stateful hop.

</details>

## NAT, Firewalls & VPN

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q54. What is NAT and why is it used? 🟢

<details>
<summary><b>Answer</b></summary>

Network Address Translation rewrites IP addresses (and often ports) as packets cross a boundary. It lets many private hosts share one public IP and hides internal addressing. The device keeps a translation table so replies are mapped back.

It is not a security feature by itself; it just happens to block unsolicited inbound connections because there is no mapping for them.

</details>

### Q55. What is the difference between SNAT, DNAT and PAT? 🔴

<details>
<summary><b>Answer</b></summary>

SNAT rewrites the source address (outbound internet access). DNAT rewrites the destination address (port forwarding, publishing an internal service). PAT, or NAT overload, maps many private IP:port pairs to one public IP using different source ports.

```bash
iptables -t nat -A POSTROUTING -s 10.0.0.0/16 -o eth0 -j MASQUERADE
iptables -t nat -A PREROUTING -p tcp --dport 8443 -j DNAT --to-destination 10.0.1.10:443
```

</details>

### Q56. How does an AWS NAT gateway work, and what are its limits? 🔴

<details>
<summary><b>Answer</b></summary>

It sits in a public subnet with an Elastic IP; private subnets route 0.0.0.0/0 to it. It performs PAT and supports up to 55,000 simultaneous connections to a single destination IP:port, scaling bandwidth to 100 Gbps automatically.

It is zonal, so use one per AZ to avoid cross-AZ charges and failure coupling. Errors like `ErrorPortAllocation` in CloudWatch mean you exhausted ports to one destination.

</details>

### Q57. Why is NAT gateway cost a common surprise and how do you reduce it? 🟡

<details>
<summary><b>Answer</b></summary>

You pay an hourly fee plus a per-GB processing fee on all traffic through it, and also normal data transfer. Pulling container images, S3 or DynamoDB traffic through NAT adds up quickly.

Reduce with gateway VPC endpoints for S3 and DynamoDB (free), interface endpoints for ECR, STS, CloudWatch and similar, same-AZ NAT per subnet, and caching images in a private registry.

</details>

### Q58. What is conntrack and what happens when its table fills up? 🔴

<details>
<summary><b>Answer</b></summary>

Linux connection tracking records the state of every flow so stateful iptables/nftables rules and NAT can work. The table has a maximum size (`net.netfilter.nf_conntrack_max`); when full, new connections are dropped and the kernel logs `nf_conntrack: table full, dropping packet`.

```bash
conntrack -C
sysctl net.netfilter.nf_conntrack_max
```
Busy NAT nodes, kube-proxy nodes and DNS-heavy workloads hit this. Raise the limit, reduce short-lived flows, or bypass tracking for known high-volume traffic.

</details>

### Q59. What is the difference between a stateful and a stateless firewall? 🟢

<details>
<summary><b>Answer</b></summary>

Stateful firewalls track connections, so allowing an outbound request automatically allows its reply. Stateless firewalls evaluate every packet independently, so you must allow both directions explicitly.

AWS security groups are stateful; network ACLs are stateless, which is why NACLs need ephemeral port ranges (1024–65535) open for return traffic.

</details>

### Q60. What is a security group and how does it differ from a network ACL? 🟢

<details>
<summary><b>Answer</b></summary>

A security group is a stateful, allow-only firewall attached to an ENI (instance level). A NACL is a stateless firewall on a subnet with numbered allow and deny rules evaluated in order, first match wins.

Security groups can reference other security groups as sources, which is the main tool for tiered architectures. NACLs are best kept as coarse guardrails, for example to deny a known bad CIDR.

</details>

### Q61. Can a security group reference another security group, and why is that useful? 🟡

<details>
<summary><b>Answer</b></summary>

Yes. A rule's source can be another security group ID, meaning "any ENI that has that group attached". Instances can scale and change IPs without rule edits.

```bash
aws ec2 authorize-security-group-ingress --group-id sg-db --protocol tcp --port 5432 --source-group sg-app
```
That one rule says "only the app tier may reach Postgres", which is the standard three-tier pattern.

</details>

### Q62. What does the iptables chain order look like for a packet? 🟡

<details>
<summary><b>Answer</b></summary>

Incoming packets hit PREROUTING (raw, mangle, nat), then a routing decision: local delivery goes through INPUT, forwarded packets go through FORWARD, then POSTROUTING. Locally generated packets go OUTPUT then POSTROUTING. Within each hook, tables are evaluated in a fixed order (raw, conntrack, mangle, nat, filter).

DNAT happens in PREROUTING before routing, SNAT/MASQUERADE in POSTROUTING after. nftables replaces iptables in modern distros with the same hook model.

</details>

### Q63. What is the difference between iptables and nftables? 🟡

<details>
<summary><b>Answer</b></summary>

nftables is the successor framework in the Linux kernel: one tool (`nft`), one ruleset language, sets and maps for efficient large rule lists, atomic rule replacement, and better performance. iptables on current distributions is often `iptables-nft`, a compatibility layer on top of nftables.

```bash
nft list ruleset
iptables --version     # (nf_tables) means the nft backend
```

</details>

### Q64. What is a firewall's default-deny principle? 🟢

<details>
<summary><b>Answer</b></summary>

Block everything, then explicitly allow only what is needed. Security groups already do this for inbound traffic; the discipline is to apply it to outbound traffic too, and to avoid 0.0.0.0/0 on admin ports like 22, 3389 and database ports.

Restrict egress for sensitive workloads: it limits data exfiltration and callbacks after a compromise.

</details>

### Q65. What is a WAF and how does it differ from a network firewall? 🟡

<details>
<summary><b>Answer</b></summary>

A web application firewall inspects HTTP(S) content at L7: SQL injection, XSS, bad bots, rate limits, geo rules. A network firewall filters by IP, port and protocol at L3/L4.

AWS WAF attaches to CloudFront, ALB, API Gateway; Cloudflare and Akamai provide edge WAFs. Start in count/log mode, tune against false positives, then switch to block.

</details>

### Q66. What is a VPN and what problem does it solve? 🟢

<details>
<summary><b>Answer</b></summary>

A VPN builds an encrypted tunnel over an untrusted network so two sites, or a user and a network, behave as if directly connected. Use cases: remote access for engineers, site-to-site links between an office or data center and a VPC, and connecting clouds.

It provides confidentiality and authentication, but not performance guarantees: it still rides the public internet unless you use a dedicated link such as Direct Connect.

</details>

### Q67. Compare IPsec, OpenVPN and WireGuard. 🟡

<details>
<summary><b>Answer</b></summary>

IPsec is the industry standard for site-to-site (IKEv2 negotiation, ESP encapsulation), supported by every router and cloud VPN gateway but complex to configure. OpenVPN uses TLS over UDP or TCP, flexible and firewall friendly but user-space and slower. WireGuard is a tiny, kernel-level, UDP-only protocol with modern crypto and simple config of public keys and allowed IPs.

```ini
[Interface]
PrivateKey = <key>
Address = 10.100.0.2/32
[Peer]
PublicKey = <peer-key>
Endpoint = vpn.example.com:51820
AllowedIPs = 10.0.0.0/16
```

</details>

### Q68. What is split tunneling versus full tunneling? 🟡

<details>
<summary><b>Answer</b></summary>

Full tunnel sends all client traffic through the VPN, which gives central inspection but adds latency and load. Split tunnel sends only traffic for corporate or VPC prefixes through the tunnel and everything else directly to the internet.

In WireGuard it is just the `AllowedIPs` list: `0.0.0.0/0` is full tunnel, specific CIDRs are split. DNS handling needs care in split mode to avoid leaking internal names.

</details>

### Q69. What is the difference between AWS Site-to-Site VPN and Direct Connect? 🟡

<details>
<summary><b>Answer</b></summary>

Site-to-Site VPN is two IPsec tunnels over the internet; it deploys in minutes, up to roughly 1.25 Gbps per tunnel, with variable latency. Direct Connect is a dedicated physical or hosted connection with consistent latency and bandwidth (1 to 100 Gbps) but weeks of lead time and cost.

Standard resilient design: Direct Connect as primary with VPN as backup, both using BGP, with local-pref set so Direct Connect wins.

</details>

### Q70. What are PFS and IKEv2 phases in IPsec? 🔴

<details>
<summary><b>Answer</b></summary>

IKE negotiates the tunnel in two stages: the first (IKE SA) authenticates peers and sets up a secure channel; the second creates the Child SAs that actually protect traffic with ESP. Perfect Forward Secrecy means each rekey does a fresh Diffie-Hellman exchange, so compromise of a long-term key does not reveal past traffic.

AWS tunnels need matching proposals on both sides, and a mismatch is the top reason a tunnel shows DOWN. Use strong groups such as DH 14+ or ECP and AES-256-GCM.

</details>

### Q71. What is a jump host versus a VPN versus zero-trust access? 🟡

<details>
<summary><b>Answer</b></summary>

A jump host exposes one hardened SSH endpoint. A VPN puts the client on the network and trusts it broadly. Zero-trust access (Tailscale ACLs, Cloudflare Access, AWS Verified Access, Teleport) authenticates user and device per application and does not grant network-wide reachability.

The trend is moving away from flat VPN access to per-service identity-aware proxies with audit trails.

</details>

### Q72. What is a DMZ? 🟡

<details>
<summary><b>Answer</b></summary>

A separate network segment between the internet and the internal network that hosts public-facing services (reverse proxies, bastions). Firewalls on both sides ensure that a compromised DMZ host cannot freely reach internal systems.

In cloud, the equivalent is public subnets containing only load balancers and NAT, with applications and data in private subnets.

</details>

### Q73. What is port knocking and is it a good control? 🔴

<details>
<summary><b>Answer</b></summary>

The firewall hides a port until a client sends a secret sequence of connection attempts to other ports. It adds obscurity, not strong security: sequences can be replayed or sniffed.

We prefer real controls: key-based SSH, SSM or a VPN, IP allow lists and short-lived certificates. Treat port knocking as an extra layer, never the only one.

</details>

## DNS

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q74. What does DNS do and what happens when you type a URL into a browser? 🟢

<details>
<summary><b>Answer</b></summary>

DNS translates names to IP addresses. The browser checks its cache, then the OS cache and `/etc/hosts`, then asks the configured recursive resolver. The resolver, if it has no cached answer, walks the hierarchy: root servers, the TLD servers (.com), then the domain's authoritative servers, and returns the answer to the client, caching it for the TTL.

Only after that does the TCP and TLS work to the returned IP begin.

</details>

### Q75. What are the common DNS record types? 🟢

<details>
<summary><b>Answer</b></summary>

A (IPv4), AAAA (IPv6), CNAME (alias to another name), MX (mail), TXT (arbitrary text, SPF/DKIM/verification), NS (delegation), SOA (zone metadata and serial), PTR (reverse lookup), SRV (service host and port), CAA (which CAs may issue certificates).

```bash
dig +short A example.com
dig +short MX example.com
dig +short TXT example.com
```

</details>

### Q76. What is the difference between a recursive resolver and an authoritative server? 🟡

<details>
<summary><b>Answer</b></summary>

An authoritative server holds the zone data and answers only for domains it is responsible for. A recursive resolver does the legwork for clients: it queries authoritative servers in sequence and caches results. Your VPC resolver, 8.8.8.8 and your corporate DNS are recursive; Route 53 hosted zones and Cloudflare DNS are authoritative.

Mixing them up leads to wrong tests: `dig @ns-123.awsdns-45.org` shows the truth at the source, while `dig @8.8.8.8` shows what the cache believes.

</details>

### Q77. Explain the DNS resolution chain step by step with dig. 🟡

<details>
<summary><b>Answer</b></summary>

`dig +trace` follows delegation from the root without using the cache.

```bash
dig +trace www.example.com
# . NS -> root servers
# com. NS -> a.gtld-servers.net ...
# example.com. NS -> ns-xxx.awsdns-xx.org
# www.example.com. A -> 93.184.216.34
```
Each step is a referral (NS plus glue records when the nameserver lives inside the zone it serves), until an authoritative server returns the final answer with the `aa` flag.

</details>

### Q78. What is TTL in DNS and how does it affect changes? 🟢

<details>
<summary><b>Answer</b></summary>

TTL is the number of seconds a resolver may cache a record. A long TTL (3600–86400) reduces query load and latency; a short TTL (30–60) makes changes propagate quickly but increases queries.

Before a planned migration we lower the TTL a full old-TTL period in advance, do the cutover, then raise it again. "DNS propagation" is really just caches expiring, and some resolvers ignore low TTLs.

</details>

### Q79. What is negative caching in DNS? 🟡

<details>
<summary><b>Answer</b></summary>

Resolvers also cache failures. An NXDOMAIN (name does not exist) or NODATA answer is cached for the SOA record's minimum/negative TTL value. If you query a name before creating it, clients may keep seeing NXDOMAIN after you add the record.

This is a classic cause of "I created the record but it does not resolve". Check the SOA minimum with `dig SOA example.com` and wait, or flush caches.

</details>

### Q80. Why can't you use a CNAME at the zone apex? 🟢

<details>
<summary><b>Answer</b></summary>

A CNAME says "this name is an alias for another name" and cannot coexist with other records at the same name. The apex (example.com) must hold SOA and NS records, so a CNAME there would break the zone.

Providers solve it with ALIAS/ANAME or Route 53 alias records, which are resolved on the server side and return A/AAAA answers. In Route 53, alias records to AWS resources are free and follow IP changes automatically.

</details>

### Q81. What is the difference between A and CNAME records? 🟢

<details>
<summary><b>Answer</b></summary>

An A record maps a name directly to an IPv4 address. A CNAME maps a name to another name, and the resolver follows the chain to eventually find an A/AAAA. CNAMEs are good for pointing `www` or `app` at a load balancer DNS name whose IPs change.

Avoid long CNAME chains: each hop is an extra lookup and they cannot be mixed with other record types at the same name.

</details>

### Q82. What is a Route 53 alias record compared with a CNAME? 🟡

<details>
<summary><b>Answer</b></summary>

Alias is a Route 53 extension: it points to an AWS target (ALB, CloudFront, S3 website, API Gateway, another record) and returns A/AAAA answers directly. It works at the apex, queries to it are free, and it supports health-aware evaluation of targets.

A CNAME is standard DNS, not allowed at the apex, billed per query, and returned to the client as a chain.

</details>

### Q83. What are the Route 53 routing policies? 🔴

<details>
<summary><b>Answer</b></summary>

Simple, weighted (percent split for canary or migration), latency-based (lowest measured latency region), failover (primary/secondary with health checks), geolocation (by user's country/continent), geoproximity (distance with bias) and multivalue answer (up to 8 healthy records).

They can be nested through alias records, for example latency routing to regions with weighted records inside each region. Health checks decide which records are eligible.

</details>

### Q84. How do DNS-based load balancing and failover work, and what are their limits? 🔴

<details>
<summary><b>Answer</b></summary>

The authoritative server returns different or multiple A records, optionally filtered by health. Clients pick one and cache it for the TTL, so failover takes at least TTL plus client retry time, and you have no control over client behavior.

Some clients cache beyond TTL (long-lived JVMs with `networkaddress.cache.ttl`, connection pools). For fast failover use a load balancer or anycast in front and treat DNS as coarse steering.

</details>

### Q85. What are NS records, delegation and glue? 🟡

<details>
<summary><b>Answer</b></summary>

NS records state which servers are authoritative for a zone. Delegation is the parent zone holding NS records for the child (the .com zone lists your NS). Glue records are A/AAAA records in the parent for nameservers that live inside the delegated zone, which avoids a circular dependency.

Changing nameservers at the registrar updates the parent delegation; it can take up to the parent's TTL (often 48h for .com NS) to take effect everywhere.

</details>

### Q86. What are SPF, DKIM and DMARC records? 🟡

<details>
<summary><b>Answer</b></summary>

All are TXT records for email authentication. SPF lists the hosts allowed to send for a domain. DKIM publishes a public key at `selector._domainkey.example.com` used to verify message signatures. DMARC at `_dmarc.example.com` tells receivers what to do when SPF/DKIM alignment fails and where to send reports.

```bash
dig +short TXT _dmarc.example.com
# "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"
```

</details>

### Q87. What is a PTR record and reverse DNS? 🟢

<details>
<summary><b>Answer</b></summary>

PTR maps an IP back to a name, stored under `in-addr.arpa` with octets reversed (5.1.0.10.in-addr.arpa). Mail servers check it, and logging tools use it to display hostnames.

```bash
dig -x 8.8.8.8 +short
```
Cloud providers let you set it for Elastic IPs and public IPs only through their console or API; for private ranges you manage reverse zones yourself.

</details>

### Q88. What is an SRV record and who uses it? 🟡

<details>
<summary><b>Answer</b></summary>

SRV records provide host, port, priority and weight for a named service: `_service._proto.name TTL IN SRV priority weight port target`. Clients choose by priority then weighted randomness.

Used by Kubernetes headless services (named ports), Consul, SIP, LDAP and etcd discovery. Standard HTTP browsers do not use SRV, which is why they rarely appear in public web setups.

</details>

### Q89. What is DNSSEC and what does it protect against? 🔴

<details>
<summary><b>Answer</b></summary>

DNSSEC adds digital signatures to DNS data (RRSIG records) verifiable through a chain of trust: root signs TLD, TLD signs your zone's DS record, and your DNSKEY signs the records. It protects integrity against cache poisoning and spoofed answers, not confidentiality.

A broken DS record or expired signatures makes validating resolvers return SERVFAIL for the whole domain, so key rollover and monitoring are critical. Route 53 supports signing with KMS-backed keys.

</details>

### Q90. What is DNS cache poisoning and how is it mitigated? 🔴

<details>
<summary><b>Answer</b></summary>

An attacker races to inject a forged answer into a resolver's cache so users are sent to a malicious IP. Defenses: randomized source ports and transaction IDs, 0x20 case randomization, DNSSEC validation, and limiting open resolvers.

Operationally, never expose a recursive resolver to the internet and keep resolver software patched.

</details>

### Q91. What is DNS over HTTPS (DoH) and DNS over TLS (DoT)? 🟡

<details>
<summary><b>Answer</b></summary>

Both encrypt resolver traffic to hide queries from the network path. DoT uses TCP/853 and is easy to identify and manage; DoH wraps DNS in HTTPS on 443 and blends with web traffic, which is harder for enterprises to monitor.

In servers we mostly keep plain DNS inside the VPC and rely on the resolver; DoH/DoT matter on end-user devices and for forwarding to public resolvers.

</details>

### Q92. When does DNS use TCP instead of UDP? 🟡

<details>
<summary><b>Answer</b></summary>

UDP/53 is the default for small queries and answers. TCP/53 is used when the response exceeds the size limit and is flagged truncated (TC bit), for zone transfers (AXFR/IXFR), and with DNSSEC or large TXT answers. EDNS0 raises the UDP size to about 1232 bytes to reduce truncation.

Firewalls that allow only 53/udp cause intermittent failures for large responses; allow both.

</details>

### Q93. What is split-horizon DNS? 🔴

<details>
<summary><b>Answer</b></summary>

The same name returns different answers depending on who asks. Internal clients get private IPs; external clients get public ones. In AWS this is a private hosted zone associated with the VPC that overrides a public zone of the same name.

Implementation pitfalls: internal-only records must exist in the private zone too, or names disappear for internal users. Document ownership of both zones.

</details>

### Q94. How does DNS work inside an AWS VPC? 🟡

<details>
<summary><b>Answer</b></summary>

Each VPC has an Amazon-provided resolver at the base of the CIDR plus two (10.0.0.2 for 10.0.0.0/16) and also at 169.254.169.254 for link-local access. It resolves public names, private hosted zones and EC2 internal names. `enableDnsSupport` and `enableDnsHostnames` must be on.

Route 53 Resolver endpoints (inbound and outbound) with forwarding rules connect on-prem DNS to the VPC. The resolver has a limit of 1,024 packets per second per ENI.

</details>

### Q95. What is the ndots setting and why does it affect Kubernetes DNS performance? 🔴

<details>
<summary><b>Answer</b></summary>

`ndots:5` in a pod's `resolv.conf` means any name with fewer than 5 dots is first tried with each search domain appended (`api.example.com.default.svc.cluster.local`, then `...svc.cluster.local`, and so on) before the bare name. External lookups therefore generate several NXDOMAIN queries first.

Mitigate by using trailing-dot FQDNs, lowering ndots with `dnsConfig`, or running NodeLocal DNSCache.
```yaml
dnsConfig:
  options:
    - name: ndots
      value: "2"
```

</details>

### Q96. What does /etc/resolv.conf contain and how does systemd-resolved change it? 🟡

<details>
<summary><b>Answer</b></summary>

`nameserver` lines, `search` domains and `options`. On modern Ubuntu it is a stub pointing to 127.0.0.53, with systemd-resolved doing caching and per-link DNS settings.

```bash
resolvectl status
resolvectl query example.com
```
Editing resolv.conf by hand is overwritten; configure via netplan, NetworkManager or `resolved.conf`.

</details>

### Q97. What is the order of name resolution on Linux? 🟡

<details>
<summary><b>Answer</b></summary>

Controlled by `/etc/nsswitch.conf`, usually `hosts: files dns` (or `files resolve dns`): first `/etc/hosts`, then DNS. Tools differ: `getent hosts` follows nsswitch like most applications, while `dig` and `nslookup` go straight to DNS and ignore `/etc/hosts`.

That difference explains "ping works but dig fails" situations.

</details>

### Q98. What is EDNS Client Subnet and why do CDNs and public resolvers care? 🔴

<details>
<summary><b>Answer</b></summary>

ECS lets a recursive resolver pass a truncated client IP prefix to authoritative servers so they can return the closest CDN edge instead of the one nearest to the resolver. Without it, users on a distant public resolver get suboptimal edges.

Privacy cost: your prefix is disclosed to authoritative servers. Some resolvers (1.1.1.1) deliberately omit it.

</details>

### Q99. What are the failure modes of DNS in production outages? 🔴

<details>
<summary><b>Answer</b></summary>

Expired or lapsed domain registration, stale NS delegation, TTL too long during failover, DNSSEC misconfiguration, resolver overload (conntrack or packet-per-second limits), missing glue, and a single provider outage. The 2016 Dyn attack showed how a single DNS provider takes down many services.

Resilience: two independent DNS providers, sensible TTLs, monitoring of external resolution from several locations, and registry lock on critical domains.

</details>

### Q100. What is the difference between `dig`, `nslookup` and `host`? 🟡

<details>
<summary><b>Answer</b></summary>

`dig` is the most detailed and scriptable: shows sections, flags, TTLs, query time and supports `+trace`, `+short`, `@server`. `nslookup` is older and interactive. `host` gives concise one-line answers.

```bash
dig example.com A +noall +answer
host -t mx example.com
nslookup example.com 1.1.1.1
```

</details>

### Q101. What is a SOA record and what do its fields mean? 🟡

<details>
<summary><b>Answer</b></summary>

Start of Authority: primary nameserver, admin email (with the first dot standing for @), serial, refresh, retry, expire and minimum TTL. Secondary servers compare the serial to decide whether to transfer the zone; the last field is the negative caching TTL.

Serial schemes like YYYYMMDDNN are a convention; managed DNS handles it for you.

</details>

### Q102. What is a wildcard DNS record and what is the risk? 🟡

<details>
<summary><b>Answer</b></summary>

`*.example.com` matches any name that has no more specific record. It is handy for ingress (`*.apps.example.com` to a load balancer) and ephemeral preview environments.

Risks: typos silently resolve, subdomain takeover when a dangling record points to a decommissioned service, and certificate wildcard scope. A wildcard matches one label only, never `a.b.example.com` unless that level also has a wildcard.

</details>

### Q103. What is subdomain takeover and how do you prevent it? 🔴

<details>
<summary><b>Answer</b></summary>

A DNS record (often a CNAME) still points to a cloud resource that was deleted, such as an S3 bucket, Elastic Beanstalk or Azure app name. An attacker claims that resource and now serves content on your subdomain, with your cookies in scope.

Prevent: remove DNS records when decommissioning, inventory records with automated scans (e.g., checking CNAME targets for NXDOMAIN or provider-specific error pages), and manage DNS as code so deletes are tied to resource lifecycle.

</details>

### Q104. What is the hosts file and when do you use it? 🟢

<details>
<summary><b>Answer</b></summary>

`/etc/hosts` maps names to IPs locally and is consulted before DNS. We use it to test a site on a new server before changing DNS, or to pin names in an emergency.

```bash
echo "203.0.113.10 app.example.com" | sudo tee -a /etc/hosts
```
`curl --resolve app.example.com:443:203.0.113.10 https://app.example.com` does the same for one command, without editing files.

</details>

## HTTP, TLS & Certificates

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q105. What is the structure of an HTTP request and response? 🟢

<details>
<summary><b>Answer</b></summary>

A request has a request line (method, path, version), headers, a blank line and an optional body. A response has a status line, headers, a blank line and a body.

```bash
curl -v https://example.com/health
# > GET /health HTTP/2
# > Host: example.com
# < HTTP/2 200
# < content-type: application/json
```
HTTP is stateless: cookies, tokens and sessions carry state.

</details>

### Q106. Which HTTP status code classes and codes should every DevOps engineer know? 🟢

<details>
<summary><b>Answer</b></summary>

1xx informational, 2xx success (200, 201, 204), 3xx redirect (301 permanent, 302/307 temporary, 304 not modified), 4xx client error (400, 401 unauthenticated, 403 forbidden, 404, 408, 429 rate limited), 5xx server error (500, 502 bad gateway, 503 unavailable, 504 gateway timeout).

For load balancers: 502 means the upstream sent an invalid response or closed the connection, 503 means no healthy targets, 504 means the upstream did not answer in time.

</details>

### Q107. What are the common HTTP methods and which are idempotent? 🟢

<details>
<summary><b>Answer</b></summary>

GET, HEAD, OPTIONS are safe (read only). GET, HEAD, PUT, DELETE and OPTIONS are idempotent: repeating them has the same effect. POST and PATCH are not guaranteed idempotent.

It matters for retries: proxies and clients can safely retry idempotent requests after a connection failure; retrying POST can create duplicates unless you use an idempotency key.

</details>

### Q108. What changed between HTTP/1.1, HTTP/2 and HTTP/3? 🟡

<details>
<summary><b>Answer</b></summary>

HTTP/1.1: text protocol, persistent connections, but one request at a time per connection, so browsers open several and suffer head-of-line blocking. HTTP/2: binary framing, multiplexed streams over one TCP connection, HPACK header compression, server push (now deprecated). HTTP/3: same semantics over QUIC (UDP), with per-stream loss recovery, built-in TLS 1.3 and faster handshakes.

HTTP/2 still has TCP-level head-of-line blocking when a packet is lost; HTTP/3 removes it.

</details>

### Q109. How does QUIC differ from TCP+TLS? 🔴

<details>
<summary><b>Answer</b></summary>

QUIC runs in user space over UDP and combines transport and TLS 1.3 in one handshake (1 RTT, 0 RTT on resume). Streams are independent so loss on one does not stall others. Connections are identified by a connection ID, so they survive client IP changes (Wi-Fi to cellular).

Operational costs: UDP/443 must be allowed, some networks throttle UDP, CPU use is higher without hardware offload, and observability tools like tcpdump cannot read payloads. Browsers discover HTTP/3 via the `Alt-Svc` header or HTTPS DNS records.

</details>

### Q110. What are HTTP keep-alive and connection reuse, and why do they matter? 🔴

<details>
<summary><b>Answer</b></summary>

Keep-alive keeps the TCP (and TLS) connection open for multiple requests, avoiding a handshake each time. It is default in HTTP/1.1.

Timeouts must be coordinated: if an upstream closes idle connections sooner than the load balancer does, the LB reuses a dead socket and returns 502. Rule of thumb: the backend keep-alive timeout should exceed the LB idle timeout (ALB default 60s, so set Node/nginx upstream to 65s+).

</details>

### Q111. What is the difference between HTTP and HTTPS? 🟢

<details>
<summary><b>Answer</b></summary>

HTTPS is HTTP carried inside TLS. It provides encryption (no eavesdropping), integrity (no tampering) and server authentication (the certificate proves you reached the right domain). Default port is 443 versus 80.

Add HSTS (`Strict-Transport-Security`) so browsers refuse to downgrade to HTTP.

</details>

### Q112. Walk through a TLS 1.3 handshake. 🔴

<details>
<summary><b>Answer</b></summary>

Client sends ClientHello with supported ciphers, a key share and SNI. Server replies with ServerHello (chosen cipher and its key share), then, already encrypted, its certificate chain, CertificateVerify and Finished. Client verifies the chain and sends Finished. Data flows after 1 RTT.

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_3
```
TLS 1.2 needs 2 RTTs and permits weaker ciphers; TLS 1.3 removes RSA key exchange and uses only forward-secret ciphers.

</details>

### Q113. What is the difference between symmetric and asymmetric encryption in TLS? 🟡

<details>
<summary><b>Answer</b></summary>

Asymmetric crypto (RSA, ECDSA, ECDHE) authenticates the server and agrees on a shared secret. Then the session switches to symmetric encryption (AES-GCM or ChaCha20-Poly1305) because it is orders of magnitude faster.

So the certificate's key is used for signatures and key agreement, while the bulk data is encrypted with ephemeral session keys.

</details>

### Q114. What is a certificate chain and what does a CA do? 🟡

<details>
<summary><b>Answer</b></summary>

A leaf certificate is signed by an intermediate CA, which is signed by a root CA that is in the client's trust store. Servers must send the leaf plus intermediates (not the root). Clients validate signatures, validity dates, name match and revocation.

A missing intermediate gives `unable to verify the first certificate` on some clients while browsers may silently fetch it, producing confusing "works in Chrome, fails in curl" reports.

</details>

### Q115. What is SNI? 🟢

<details>
<summary><b>Answer</b></summary>

Server Name Indication is a TLS extension in the ClientHello containing the hostname the client wants, so one IP and port can serve many certificates. Without it a server would not know which cert to present.

```bash
curl --resolve app.example.com:443:203.0.113.10 https://app.example.com
openssl s_client -connect 203.0.113.10:443 -servername app.example.com
```
Encrypted Client Hello (ECH) hides SNI from observers.

</details>

### Q116. What fields matter in an X.509 certificate? 🟡

<details>
<summary><b>Answer</b></summary>

Subject and Subject Alternative Names (SAN: modern clients validate only SAN, not CN), validity period, issuer, public key, key usage and extended key usage (serverAuth/clientAuth), basic constraints (CA true or false), and signature algorithm.

```bash
openssl x509 -in cert.pem -noout -subject -issuer -dates -ext subjectAltName
```

</details>

### Q117. How do you inspect a remote server's certificate from the command line? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -fingerprint -sha256
# show full chain
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null
```
Check `Verify return code: 0 (ok)` at the end. Use `-verify_hostname example.com` to test name matching.

</details>

### Q118. How does Let's Encrypt and the ACME protocol work? 🟡

<details>
<summary><b>Answer</b></summary>

A client (certbot, acme.sh, cert-manager) proves control of a domain to the ACME server through a challenge: HTTP-01 (serve a token at `/.well-known/acme-challenge/` on port 80), DNS-01 (create a `_acme-challenge` TXT record, required for wildcards) or TLS-ALPN-01. After validation it requests and receives a certificate valid for 90 days today, with shorter lifetimes being phased in.

Automation is mandatory: short lifetimes mean renewal must be hands-off and monitored.

</details>

### Q119. What is mutual TLS (mTLS)? 🔴

<details>
<summary><b>Answer</b></summary>

In normal TLS only the server presents a certificate. In mTLS the server also requests a client certificate and verifies it against a trusted CA, so both sides are authenticated cryptographically.

Used for service-to-service auth in meshes (Istio, Linkerd), API gateways for B2B clients, and Kubernetes control plane components. Nginx example: `ssl_client_certificate ca.pem; ssl_verify_client on;`.

</details>

### Q120. What is a self-signed certificate and when is it acceptable? 🟢

<details>
<summary><b>Answer</b></summary>

A certificate signed by its own key rather than a CA, so no client trusts it by default. Fine for local development, internal tests, or when you explicitly distribute the cert to clients you control.

```bash
openssl req -x509 -newkey rsa:2048 -nodes -keyout key.pem -out cert.pem -days 365 -subj "/CN=localhost"
```
Never ship it to public users; they get `NET::ERR_CERT_AUTHORITY_INVALID`.

</details>

### Q121. What is the difference between TLS termination, passthrough and re-encryption? 🔴

<details>
<summary><b>Answer</b></summary>

Termination: the load balancer decrypts and forwards plain HTTP to backends. Passthrough: an L4 balancer forwards encrypted bytes untouched, so the backend holds the cert and the balancer cannot see HTTP. Re-encryption (bridging): terminate at the LB for L7 features, then open a new TLS connection to the backend.

Choose by need: L7 routing wants termination; end-to-end encryption with compliance wants re-encryption or passthrough; mTLS to the backend often means passthrough.

</details>

### Q122. What are cipher suites and how should TLS be configured on a public endpoint? 🟡

<details>
<summary><b>Answer</b></summary>

A cipher suite names key exchange, authentication, bulk cipher and hash, for example `TLS_AES_128_GCM_SHA256` (1.3) or `ECDHE-RSA-AES128-GCM-SHA256` (1.2). Enable TLS 1.2 and 1.3 only, forward-secret AEAD ciphers, and disable SSLv3, TLS 1.0 and 1.1, RC4, 3DES and export ciphers.

Use Mozilla's SSL configuration generator profile "intermediate" and test with `testssl.sh` or SSL Labs.

</details>

### Q123. What is HSTS and what are its risks? 🟡

<details>
<summary><b>Answer</b></summary>

`Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` tells browsers to use HTTPS only for the domain for the stated period. It defeats SSL-stripping downgrade attacks.

The risk is lock-in: if your certificate breaks or a subdomain does not support HTTPS, users cannot click through. Roll out with a short max-age first and only preload when every subdomain is ready.

</details>

### Q124. What are OCSP, CRL and OCSP stapling? 🟡

<details>
<summary><b>Answer</b></summary>

Both check whether a certificate was revoked. CRL is a downloaded list; OCSP is a per-certificate online query to the CA. Stapling lets the server fetch the signed OCSP response and send it in the handshake, saving client round trips and avoiding leaking browsing to the CA.

In nginx: `ssl_stapling on; ssl_stapling_verify on;`. Let's Encrypt ended OCSP in 2025 in favor of short-lived certs and CRLs, so check current CA guidance.

</details>

### Q125. What is certificate pinning and why is it risky? 🔴

<details>
<summary><b>Answer</b></summary>

The client hard-codes an expected certificate or public key hash so even a trusted CA cannot impersonate the server. It protects against rogue CAs but breaks when you rotate certs or switch CA unless you pin a backup key.

Browsers removed HPKP; mobile apps still use it with fallback pins. In server-to-server code prefer private CA trust over per-cert pinning.

</details>

### Q126. What are HTTP caching headers? 🟡

<details>
<summary><b>Answer</b></summary>

`Cache-Control` (max-age, s-maxage, no-store, no-cache, private, public, immutable), `ETag` and `Last-Modified` with conditional requests (`If-None-Match`, `If-Modified-Since`) answered by 304, and `Vary` which declares which request headers create separate cache entries.

Static assets with hashed filenames: `Cache-Control: public, max-age=31536000, immutable`. HTML: short max-age or `no-cache` with ETag. `no-cache` still stores, it only forces revalidation.

</details>

### Q127. What are the forwarding headers a proxy adds and how are they trusted? 🟡

<details>
<summary><b>Answer</b></summary>

`X-Forwarded-For` (client IP chain), `X-Forwarded-Proto`, `X-Forwarded-Host`, and the standardized `Forwarded` header. The PROXY protocol is an alternative for L4 that prepends connection info.

Applications must trust these only when set by your own proxies; otherwise clients can spoof their IP. Configure real-IP handling explicitly: nginx `set_real_ip_from 10.0.0.0/8; real_ip_header X-Forwarded-For;`.

</details>

### Q128. What is CORS and who enforces it? 🟡

<details>
<summary><b>Answer</b></summary>

Cross-Origin Resource Sharing is a browser mechanism. A page on one origin calling another origin's API needs `Access-Control-Allow-Origin` (and others) in the response; for non-simple requests the browser first sends an OPTIONS preflight.

It is not a server-side security control: curl ignores it. Do not reflect arbitrary Origin values with credentials enabled.

</details>

### Q129. What is HTTP request smuggling? 🔴

<details>
<summary><b>Answer</b></summary>

When a front-end proxy and back-end server disagree about where a request ends (Content-Length vs Transfer-Encoding), an attacker hides a second request in the body that the backend treats as a new request, bypassing front-end controls.

Defenses: use HTTP/2 end to end, reject ambiguous requests, keep proxy and backend versions patched, and configure strict parsing. Mismatched keep-alive reuse between LB and backend increases exposure.

</details>

### Q130. What is the difference between 301, 302, 307 and 308 redirects? 🟡

<details>
<summary><b>Answer</b></summary>

301 and 308 are permanent; 302 and 307 are temporary. 307 and 308 preserve the request method and body, while historically 301 and 302 let clients switch POST to GET.

For SEO and caching, use 301/308 for permanent moves; for HTTP to HTTPS redirects 301 or 308 is standard, and 308 is safer for APIs.

</details>

### Q131. What is WebSocket and how does it get through proxies? 🟡

<details>
<summary><b>Answer</b></summary>

A WebSocket starts as an HTTP/1.1 request with `Upgrade: websocket` and `Connection: Upgrade`, receives 101 Switching Protocols, then becomes a long-lived bidirectional channel. Proxies need to pass those hop-by-hop headers and use long timeouts.

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_read_timeout 3600s;
```

</details>

### Q132. What does the Host header do? 🟢

<details>
<summary><b>Answer</b></summary>

It tells the server which virtual host or domain the request is for, enabling many sites on one IP (name-based virtual hosting). In HTTP/2 the equivalent is the `:authority` pseudo-header.

Testing by IP: `curl -H "Host: app.example.com" http://203.0.113.10/`. Routing in ingress controllers and ALB rules depends on it.

</details>

### Q133. How do gRPC and HTTP/2 interact with load balancers? 🔴

<details>
<summary><b>Answer</b></summary>

gRPC uses HTTP/2 with long-lived multiplexed connections, so an L4 balancer spreads connections, not requests, and one busy client can pin all traffic to one backend. You need an L7 balancer that understands HTTP/2 (ALB with gRPC target groups, Envoy, nginx `grpc_pass`) or client-side load balancing.

In Kubernetes a plain ClusterIP Service gives poor gRPC distribution, so use a headless Service with client balancing or a mesh.

</details>

### Q134. What are cookies and how are they secured? 🟢

<details>
<summary><b>Answer</b></summary>

Small key-value pairs the server sets via `Set-Cookie` and the browser returns. Secure attributes: `Secure` (HTTPS only), `HttpOnly` (not readable by JS), `SameSite=Lax|Strict` (CSRF mitigation), plus `Domain`, `Path` and expiry.

Sticky sessions at a load balancer use a cookie (AWSALB) to pin a client to one target.

</details>

### Q135. What is the difference between authentication at TLS and at the HTTP layer? 🟡

<details>
<summary><b>Answer</b></summary>

TLS authenticates machines: the server by cert, optionally the client by cert (mTLS). The HTTP layer authenticates users and applications: basic auth, bearer tokens (JWT/OAuth2), API keys, signed requests (SigV4).

In production they combine: mTLS for service identity, tokens for user identity and authorization.

</details>

### Q136. How would you debug a TLS handshake failure? 🔴

<details>
<summary><b>Answer</b></summary>

Check with `openssl s_client` which step fails. Typical errors: `certificate verify failed` (chain, expiry or CA), `tlsv1 alert protocol version` (version mismatch), `no shared cipher`, `handshake failure` (SNI missing or client cert required), `hostname mismatch`.

```bash
openssl s_client -connect host:443 -servername host -tls1_2 </dev/null
curl -vI --tlsv1.2 --tls-max 1.2 https://host
```
Then compare dates, SANs, intermediates, and capture with `tcpdump -w` to inspect the alert in Wireshark. Also check the system clock.

</details>

## Load Balancers, Proxies & CDNs

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q137. What is a load balancer and why do we use one? 🟢

<details>
<summary><b>Answer</b></summary>

It distributes incoming traffic across several backends to improve availability, scale horizontally, enable zero-downtime deploys and hide backend failures through health checks. It also gives one stable endpoint while instances come and go.

Types by layer: L4 (TCP/UDP) and L7 (HTTP-aware).

</details>

### Q138. What is the difference between L4 and L7 load balancing? 🟡

<details>
<summary><b>Answer</b></summary>

L4 balances connections using IP and port, without parsing payload: very fast, preserves protocols like raw TCP, TLS passthrough and UDP. L7 terminates the connection and understands HTTP, so it can route by host and path, rewrite headers, handle auth, do canaries and retry.

AWS: NLB is L4 (millions of requests per second, static IPs, preserves source IP), ALB is L7 (host/path rules, WebSockets, gRPC, OIDC auth, WAF).

</details>

### Q139. Compare ALB, NLB, GWLB and Classic Load Balancer in AWS. 🟡

<details>
<summary><b>Answer</b></summary>

ALB: HTTP/HTTPS/gRPC, content routing, Lambda targets. NLB: TCP/UDP/TLS, ultra-low latency, static or Elastic IPs, PrivateLink support. GWLB: transparent L3 gateway to insert third-party firewalls and inspection appliances using GENEVE. CLB: legacy, avoid for new work.

Common pattern: CloudFront to ALB for web apps, NLB for database proxies, TCP services and PrivateLink endpoints.

</details>

### Q140. What load-balancing algorithms are common? 🟢

<details>
<summary><b>Answer</b></summary>

Round robin, weighted round robin, least connections, least response time, IP hash or consistent hashing (session affinity by client or key), and random with two choices ("power of two choices").

ALB uses round robin or least outstanding requests per target group. Least-connections or least-outstanding-requests suit requests with uneven duration; hashing suits caches.

</details>

### Q141. What are health checks and how should they be designed? 🔴

<details>
<summary><b>Answer</b></summary>

The balancer probes each target periodically (TCP connect, HTTP GET, gRPC) and removes it after N failures, adding it back after M successes. Tune interval, timeout, thresholds and path.

Design `/health` to check that the process can serve traffic (and critical dependencies cautiously) but not to cascade: if the endpoint checks a shared database, a DB blip marks every instance unhealthy and takes the whole service down. Separate liveness from readiness.

</details>

### Q142. What is connection draining (deregistration delay)? 🟡

<details>
<summary><b>Answer</b></summary>

When a target is removed or terminated, the balancer stops sending new requests but lets in-flight ones finish for a configured period (ALB default 300 seconds). Without it, deployments cut active requests.

Set it to slightly above your longest typical request, make the app handle SIGTERM gracefully, and stop accepting new work while finishing current work.

</details>

### Q143. What are sticky sessions and what are the drawbacks? 🟡

<details>
<summary><b>Answer</b></summary>

Session affinity pins a client to one backend, via a cookie (ALB) or source IP hash. It helps stateful apps that keep sessions in memory.

Drawbacks: uneven load, lost sessions on scale-in or crash, and complexity for blue/green. Preferred design: stateless services with shared session storage (Redis, DynamoDB) or signed tokens.

</details>

### Q144. What is a reverse proxy and how is it different from a forward proxy? 🟢

<details>
<summary><b>Answer</b></summary>

A reverse proxy sits in front of servers and represents them to clients: nginx, HAProxy, Envoy, ALB. It handles TLS, caching, routing and protection. A forward proxy sits in front of clients and represents them to the internet: Squid, corporate egress proxies.

Same technology, opposite direction. Reverse proxies protect servers; forward proxies control and log client egress.

</details>

### Q145. Show a minimal nginx reverse proxy configuration. 🟡

<details>
<summary><b>Answer</b></summary>

```nginx
upstream app {
    server 10.0.1.10:8080 max_fails=3 fail_timeout=10s;
    server 10.0.1.11:8080;
    keepalive 32;
}
server {
    listen 443 ssl http2;
    server_name app.example.com;
    ssl_certificate     /etc/ssl/app.crt;
    ssl_certificate_key /etc/ssl/app.key;
    location / {
        proxy_pass http://app;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Test with `nginx -t` and reload with `nginx -s reload`.

</details>

### Q146. What is the difference between nginx, HAProxy and Envoy? 🟡

<details>
<summary><b>Answer</b></summary>

nginx: web server and reverse proxy, great for static content, TLS and simple L7 routing. HAProxy: purpose-built load balancer with excellent L4/L7 performance, stick tables and detailed stats. Envoy: modern L7 proxy with dynamic configuration over xDS, rich observability and gRPC support, the data plane of Istio and many gateways.

Pick nginx for web serving plus proxy, HAProxy for pure high-throughput balancing, Envoy for service mesh and dynamic environments.

</details>

### Q147. What are common nginx proxy timeouts and what do 499, 502 and 504 mean there? 🟡

<details>
<summary><b>Answer</b></summary>

`proxy_connect_timeout`, `proxy_send_timeout`, `proxy_read_timeout` (defaults 60s). 502 means the upstream refused, reset, or sent garbage. 504 means the upstream did not respond within the timeout. 499 is nginx-specific: the client closed the connection before the response.

A spike of 499s often means upstream latency made users or an upstream LB give up first.

</details>

### Q148. What is the PROXY protocol? 🟡

<details>
<summary><b>Answer</b></summary>

A small header (v1 text or v2 binary) prepended by an L4 proxy to the TCP stream carrying the original client IP and port, since the backend only sees the proxy's address. The backend must be configured to expect it, or every connection will fail.

nginx: `listen 80 proxy_protocol;`, then `real_ip_header proxy_protocol;`. NLB can enable it per target group when preserving source IP is not possible.

</details>

### Q149. How does the source IP appear behind ALB, NLB and Cloudflare? 🟡

<details>
<summary><b>Answer</b></summary>

Behind an ALB, the backend sees the ALB's IP and gets the client in `X-Forwarded-For`. NLB preserves the client IP for instance targets (not for IP targets by default, where you can use PROXY protocol v2). Behind Cloudflare, use `CF-Connecting-IP` or X-Forwarded-For and trust only Cloudflare's published ranges.

Security groups on backends should allow the LB security group, not the internet.

</details>

### Q150. What is a CDN and what does it improve? 🟢

<details>
<summary><b>Answer</b></summary>

A content delivery network caches content at edge locations near users. It lowers latency, offloads origin traffic, absorbs DDoS and often terminates TLS and runs WAF rules.

Examples: CloudFront, Cloudflare, Fastly, Akamai. Static assets cache well; dynamic content can still benefit from TCP/TLS termination near the user and persistent origin connections.

</details>

### Q151. How does CDN caching work and what is the cache key? 🟡

<details>
<summary><b>Answer</b></summary>

The edge looks up a cache entry using the cache key, by default scheme, host, path and query string, plus any headers or cookies you include. On a miss it fetches from the origin and stores according to `Cache-Control` and CDN rules.

Including too much (all cookies, unneeded query params) fragments the cache and destroys hit ratio. Keep the key minimal and normalize query strings.

</details>

### Q152. How do you invalidate a CDN cache and what is the better approach? 🟡

<details>
<summary><b>Answer</b></summary>

You can issue invalidations (`aws cloudfront create-invalidation --distribution-id E123 --paths "/index.html"`), which are slow-ish and billed after a free quota. Better: versioned immutable asset names (`app.3f9a1c.js`) with a one-year TTL, and a short TTL only for the HTML entry point.

That removes the need to invalidate on most deploys.

</details>

### Q153. How do you secure the origin behind a CDN? 🟡

<details>
<summary><b>Answer</b></summary>

Make the origin unreachable except via the CDN: restrict the security group to the CDN's managed prefix list (CloudFront has an AWS managed prefix list), add a secret custom header the origin checks, use Origin Access Control for S3, and optionally mTLS or a VPC origin.

Otherwise attackers hit the origin IP directly and bypass WAF and caching.

</details>

### Q154. What is the difference between a CDN and Global Accelerator? 🟡

<details>
<summary><b>Answer</b></summary>

CDN caches content and proxies HTTP at the edge. Global Accelerator provides two static anycast IPs that route TCP/UDP traffic onto the AWS backbone to the nearest healthy regional endpoint, with fast failover, but does no caching.

Use CDN for cacheable web content; Global Accelerator for non-HTTP protocols, static IP requirements or multi-region active-active failover without DNS TTL delays.

</details>

### Q155. How would you implement a canary or weighted rollout using load balancers? 🔴

<details>
<summary><b>Answer</b></summary>

With an ALB, create two target groups and a forward action with weights (for example 95/5), watch error rate and latency per target group, then shift gradually. Route 53 weighted records can do it across load balancers but are subject to DNS caching.

```bash
aws elbv2 modify-listener --listener-arn $L --default-actions \
 'Type=forward,ForwardConfig={TargetGroups=[{TargetGroupArn=blue,Weight=95},{TargetGroupArn=green,Weight=5}]}'
```
Automate rollback on SLO breach with CodeDeploy, Argo Rollouts or Flagger.

</details>

### Q156. What happens when a load balancer node or AZ fails? 🔴

<details>
<summary><b>Answer</b></summary>

Cloud LBs run multiple nodes, one or more per enabled AZ, each with a DNS entry. Failed nodes are removed from DNS (short TTL, 60s for ALB). Cross-zone load balancing spreads traffic evenly to all registered targets across AZs; without it, each node only uses targets in its own AZ.

Consequences: keep capacity balanced across AZs, avoid enabling only one subnet, and know that ALB cross-zone is always on while NLB's is off by default and incurs inter-AZ charges.

</details>

### Q157. What is an L4 versus L7 health check, and why can a TCP check pass while the app is broken? 🟡

<details>
<summary><b>Answer</b></summary>

A TCP check only verifies something accepts connections on the port; the process can be deadlocked or return 500 and still pass. An HTTP check requests a path and validates the status code (and optionally body), proving the app code is actually answering.

Use HTTP checks for web apps and reserve TCP for protocols without an HTTP interface.

</details>

### Q158. What is rate limiting and where should it live? 🔴

<details>
<summary><b>Answer</b></summary>

Controlling request rate per client key to protect backends and ensure fairness. Algorithms: token bucket, leaky bucket, fixed and sliding window. Respond with 429 and `Retry-After`.

Layers: CDN/WAF for coarse abuse, API gateway or ingress for per-API-key quotas, application for business rules. nginx: `limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;` plus `limit_req zone=api burst=20 nodelay;`.

</details>

### Q159. What is circuit breaking and retry budgeting in a proxy or mesh? 🔴

<details>
<summary><b>Answer</b></summary>

A circuit breaker stops sending requests to an upstream that is failing or saturated (open state), returning fast errors and giving it time to recover. Retry budgets cap retries to a percentage of live traffic so retries do not turn a partial failure into a retry storm.

Envoy configures this per cluster (max connections, pending requests, outlier detection). Always pair retries with timeouts, jitter and idempotency.

</details>

### Q160. How does consistent hashing help in load balancing and caching? 🔴

<details>
<summary><b>Answer</b></summary>

It maps keys and nodes onto a ring so adding or removing a node only remaps about 1/N of keys, instead of nearly all with modulo hashing. Useful for cache tiers, sharded backends and session affinity with minimal disruption.

HAProxy `balance hash` with `hash-type consistent`, nginx `hash $request_uri consistent;`, Envoy ring hash or Maglev.

</details>

## Command-Line Tools & Diagnostics

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q161. How do you check if a remote port is open? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
nc -zv db.internal 5432          # TCP connect test
nc -zvu 10.0.0.2 53              # UDP (less reliable)
timeout 3 bash -c '</dev/tcp/db.internal/5432' && echo open
curl -sv telnet://db.internal:5432 --max-time 3
```
Interpret: "succeeded" means a handshake completed; "refused" means the host answered RST; a hang means filtering or no route.

</details>

### Q162. How do you list listening ports and the processes that own them? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ss -tulpn          # TCP+UDP, listening, numeric, with process
ss -tlnp 'sport = :443'
lsof -iTCP:8080 -sTCP:LISTEN
```
`ss` replaced `netstat` and reads from netlink directly, so it is much faster on hosts with many sockets. Run it with sudo to see process names for other users.

</details>

### Q163. What does `ping` tell you and what does it not? 🟢

<details>
<summary><b>Answer</b></summary>

It sends ICMP echo and reports round-trip time and loss, proving basic L3 reachability. It does not prove a service works, and many networks and cloud security groups block ICMP, so no reply is not proof of failure.

```bash
ping -c 4 -M do -s 1472 10.0.1.5     # MTU probe with Don't Fragment set
```

</details>

### Q164. How does traceroute work and how do you read the output? 🟢

<details>
<summary><b>Answer</b></summary>

It sends probes with increasing TTL and records the ICMP Time Exceeded replies, listing each hop with three RTTs. Stars (`* * *`) mean a hop did not reply, often just ICMP rate limiting, not a failure; what matters is whether later hops respond.

```bash
traceroute -n 8.8.8.8
traceroute -T -p 443 example.com     # TCP SYN probes, passes more firewalls
```

</details>

### Q165. What is mtr and why is it better than traceroute? 🟡

<details>
<summary><b>Answer</b></summary>

mtr combines ping and traceroute, continuously probing each hop and showing loss and latency statistics.

```bash
mtr -rwzbc 100 example.com
```
Read loss carefully: loss at one hop that does not continue to later hops is ICMP deprioritization, not real loss. Loss that starts at a hop and persists to the destination is real. Send the report to your provider as evidence.

</details>

### Q166. How do you query DNS with dig and read the answer? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
dig example.com A
dig +noall +answer +ttlid example.com
dig @1.1.1.1 example.com AAAA
dig -x 10.0.1.5
dig +short NS example.com
```
Look at `status:` (NOERROR, NXDOMAIN, SERVFAIL, REFUSED), the flags (`aa` authoritative, `ra` recursion available, `tc` truncated), the TTL in the answer, and `Query time`.

</details>

### Q167. How do you use curl to debug an HTTP problem? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
curl -v https://example.com
curl -I https://example.com                        # headers only
curl -sS -o /dev/null -w "dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total} code=%{http_code}\n" https://example.com
curl --resolve example.com:443:1.2.3.4 https://example.com
curl -k https://self-signed.local                  # skip verification (diagnosis only)
```
The `-w` timing breakdown shows whether delay is DNS, TCP, TLS or the application.

</details>

### Q168. How do you capture traffic with tcpdump? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo tcpdump -i any -nn port 443 and host 10.0.1.5
sudo tcpdump -i eth0 -nn -s0 -w capture.pcap 'tcp port 8080'
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'
sudo tcpdump -nn -A -i lo port 8080          # print ASCII payload
```
Use `-nn` to skip name lookups, always filter, and rotate big captures with `-C 100 -W 10`. Open the pcap in Wireshark for deeper analysis.

</details>

### Q169. How do you read tcpdump output for a TCP handshake and a reset? 🔴

<details>
<summary><b>Answer</b></summary>

```
IP 10.0.1.5.51234 > 10.0.2.9.443: Flags [S], seq 100, win 64240
IP 10.0.2.9.443 > 10.0.1.5.51234: Flags [S.], seq 500, ack 101
IP 10.0.1.5.51234 > 10.0.2.9.443: Flags [.], ack 501
```
`S` SYN, `.` ACK, `P` PSH, `F` FIN, `R` RST. SYN with no SYN-ACK suggests filtering. SYN followed by `R.` means connection refused. Repeated retransmitted SYNs are a path problem.

</details>

### Q170. What are the most useful `ss` filters and statistics? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
ss -s                                   # socket summary
ss -tan state established '( dport = :443 )' | wc -l
ss -ti dst 10.0.2.9                     # per-connection RTT, cwnd, retransmits
ss -tn state time-wait | wc -l
ss -lntp                                # check Recv-Q/Send-Q for accept backlog
```
`ss -ti` exposes `rtt`, `cwnd`, `retrans` and `bytes_acked`, enough for quick connection health checks without a packet capture.

</details>

### Q171. How do you view and manage interfaces and addresses with `ip`? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ip -br addr                  # brief addresses
ip link show
ip link set eth0 up
ip addr add 10.0.1.50/24 dev eth0
ip route show
ip neigh
```
`ip` from iproute2 replaces ifconfig, route and arp. Changes made this way are not persistent; use netplan, NetworkManager or systemd-networkd.

</details>

### Q172. How do you test bandwidth between two hosts? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
# server
iperf3 -s
# client: 4 parallel streams for 30 seconds, then reverse direction
iperf3 -c 10.0.1.5 -P 4 -t 30
iperf3 -c 10.0.1.5 -R
iperf3 -c 10.0.1.5 -u -b 500M        # UDP, shows jitter and loss
```
Test between same-AZ, cross-AZ and cross-region instances; single streams are limited by TCP window and instance flow limits (AWS limits a single flow to about 5 Gbps, or 10 Gbps inside a cluster placement group).

</details>

### Q173. How do you inspect an HTTPS server with `openssl` and `curl` for expiry monitoring? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -enddate
openssl x509 -checkend 1209600 -noout -in cert.pem && echo ok || echo "expires within 14 days"
curl -vI https://example.com 2>&1 | grep -E "expire date|subject:|issuer:"
```
Feed the first command into cron or a Prometheus blackbox exporter and alert at 21 and 7 days.

</details>

### Q174. How do you find which process is responsible for a connection or a burst of outbound traffic? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
ss -tnp dst 203.0.113.10
sudo lsof -i @203.0.113.10
sudo nethogs eth0                     # bandwidth per process
sudo bpftrace -e 'tracepoint:tcp:tcp_retransmit_skb { @[comm] = count(); }'
```
For containers, enter the namespace: `nsenter -t <pid> -n ss -tnp`. In Kubernetes use `kubectl debug node/...` or an ephemeral container with netshoot.

</details>

### Q175. How do you trace TCP retransmissions and latency at the kernel level? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
nstat -az | grep -i -E 'retrans|listenoverflows|tcpbacklogdrop'
sudo tcpretrans-bpfcc                   # BCC tool, shows retransmitted flows
sudo tcplife-bpfcc                      # per-connection lifespan and bytes
ss -ti | grep -E 'retrans|rtt'
```
Rising `TcpRetransSegs` points to loss or congestion. Combine with `ethtool -S eth0` to check NIC drops, and `tc -s qdisc show dev eth0` to see queue drops.

</details>

### Q176. What do `ethtool`, `ip -s link` and `/proc/net/dev` show? 🟡

<details>
<summary><b>Answer</b></summary>

Interface-level counters: packets, bytes, errors, drops, overruns and collisions. `ethtool -S eth0` adds driver stats (on AWS ENA, `bw_in_allowance_exceeded` and `pps_allowance_exceeded` reveal instance-level throttling).

```bash
ip -s link show eth0
ethtool -S eth0 | grep -E 'exceeded|drop'
```
Incrementing allowance_exceeded counters mean you need a bigger instance type, not tuning.

</details>

### Q177. How do you debug a network issue in a container without network tools in the image? 🔴

<details>
<summary><b>Answer</b></summary>

Use a debug container or enter the network namespace from the host.

```bash
kubectl debug -it pod/web-7d9 --image=nicolaka/netshoot --target=web
docker inspect -f '{{.State.Pid}}' web          # then:
sudo nsenter -t <PID> -n ss -tulpn
sudo nsenter -t <PID> -n tcpdump -i eth0 -nn
```
The `--target` flag shares the process namespace so you see the app's sockets, and you do not have to rebuild or modify the image.

</details>

### Q178. How do you resolve a name the way the application does? 🟡

<details>
<summary><b>Answer</b></summary>

Use `getent`, which follows nsswitch and the system resolver, unlike dig.

```bash
getent hosts api.internal
getent ahostsv4 example.com
resolvectl query api.internal
```
If dig works but getent fails, suspect `/etc/nsswitch.conf`, `search` domains, systemd-resolved routing per interface, or a stale nscd/sssd cache.

</details>

### Q179. How do you check the public IP of a host and the metadata that explains it? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
curl -s https://checkip.amazonaws.com
curl -s -H "X-aws-ec2-metadata-token: $(curl -s -X PUT http://169.254.169.254/latest/api/token -H 'X-aws-ec2-metadata-token-ttl-seconds: 60')" http://169.254.169.254/latest/meta-data/public-ipv4
```
The first shows the egress address the world sees (a NAT gateway's EIP for private instances), the second the instance's own assigned public IP.

</details>

### Q180. How do you measure packet loss and path MTU correctly? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
ping -c 100 -i 0.2 10.0.2.9 | tail -3
tracepath -n example.com               # reports pmtu per hop
ping -M do -s 1472 example.com          # 1472 + 28 = 1500
ip route get 10.0.2.9                   # may show cached mtu
```
If 1472 fails and smaller sizes work, a hop has a smaller MTU and ICMP "fragmentation needed" is not coming back. Look for tunnels (VPN, VXLAN, GRE) and for blocked ICMP.

</details>

## Kubernetes, Containers, MTU & Performance Basics

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q181. What are the basic Kubernetes networking rules? 🟢

<details>
<summary><b>Answer</b></summary>

Every pod gets its own IP, pods can talk to all other pods without NAT, and nodes can talk to all pods. Services provide stable virtual IPs and DNS names in front of changing pod sets. These rules are implemented by a CNI plugin.

This flat model is why pod CIDRs must not overlap with your VPC or on-prem ranges.

</details>

### Q182. What are the Service types in Kubernetes? 🟢

<details>
<summary><b>Answer</b></summary>

ClusterIP (internal virtual IP, default), NodePort (opens a port 30000–32767 on every node), LoadBalancer (provisions a cloud load balancer), ExternalName (CNAME to an external DNS name), and headless (`clusterIP: None`, DNS returns pod IPs directly).

```bash
kubectl get svc -o wide
kubectl get endpointslices -l kubernetes.io/service-name=web
```

</details>

### Q183. What is the difference between a Service and an Ingress? 🟢

<details>
<summary><b>Answer</b></summary>

A Service exposes a set of pods at L4. An Ingress is an L7 routing resource (host and path rules, TLS) implemented by an ingress controller such as nginx-ingress, Traefik or AWS Load Balancer Controller. The newer Gateway API supersedes Ingress with more expressive, role-oriented resources (GatewayClass, Gateway, HTTPRoute).

Ingress needs a controller running; creating the resource alone does nothing.

</details>

### Q184. How does kube-proxy implement Services? 🔴

<details>
<summary><b>Answer</b></summary>

kube-proxy watches Services and EndpointSlices and programs the node's datapath: iptables mode (default, rules per service and endpoint, random probability load balancing), IPVS mode (kernel L4 balancer with real algorithms, scales better) or nftables mode (newer). A packet to ClusterIP is DNAT'd to a pod IP before routing.

```bash
iptables -t nat -L KUBE-SERVICES -n | head
ipvsadm -Ln
```
Cilium can replace kube-proxy entirely with eBPF.

</details>

### Q185. How does DNS work for Services and pods in Kubernetes? 🟡

<details>
<summary><b>Answer</b></summary>

CoreDNS serves the `cluster.local` zone. A Service `web` in namespace `shop` resolves as `web.shop.svc.cluster.local`; from the same namespace just `web`. Headless services return A records for each ready pod, and StatefulSet pods get stable names like `db-0.db.shop.svc.cluster.local`.

```bash
kubectl exec -it pod -- nslookup web.shop
kubectl -n kube-system logs deploy/coredns
```

</details>

### Q186. What is a CNI plugin and how do the main ones differ? 🟡

<details>
<summary><b>Answer</b></summary>

The Container Network Interface is the plugin API that sets up pod networking when a pod starts. Flannel: simple overlay (VXLAN). Calico: L3 routing with BGP or VXLAN plus network policy. Cilium: eBPF dataplane with policy, observability and kube-proxy replacement. AWS VPC CNI: gives pods real VPC IPs from ENIs.

Choose by need: policy and observability (Cilium, Calico), VPC-native addressing (AWS VPC CNI), simplicity (Flannel).

</details>

### Q187. What are the trade-offs of AWS VPC CNI? 🔴

<details>
<summary><b>Answer</b></summary>

Pods get routable VPC addresses, so security groups, flow logs and direct access from other VPC resources work naturally with no overlay overhead. Downsides: IP exhaustion (each pod consumes a subnet IP), per-instance ENI and IP limits restricting pods per node, and the need for large subnets or secondary CIDRs (100.64.0.0/10 with custom networking).

Mitigate with prefix delegation (`ENABLE_PREFIX_DELEGATION=true`) that assigns /28 blocks per ENI slot, and IPv6 clusters.

</details>

### Q188. What is a Kubernetes NetworkPolicy? 🟡

<details>
<summary><b>Answer</b></summary>

A namespaced resource that restricts pod ingress and egress by pod selector, namespace selector, IP block and port. By default all traffic is allowed; once a pod is selected by any policy, only the allowed flows are permitted. It is enforced by the CNI, so Flannel alone ignores it.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: db-allow-app, namespace: shop}
spec:
  podSelector: {matchLabels: {app: db}}
  policyTypes: [Ingress]
  ingress:
  - from: [{podSelector: {matchLabels: {app: api}}}]
    ports: [{protocol: TCP, port: 5432}]
```

</details>

### Q189. How does a packet travel from one pod to another pod on a different node with an overlay network? 🔴

<details>
<summary><b>Answer</b></summary>

The source pod sends to the destination pod IP via its veth pair to the node. The node's routing sends it to the overlay device (VXLAN `flannel.1` or similar), which encapsulates the packet in UDP (8472 or 4789) addressed to the destination node's IP. The remote node decapsulates and delivers to the pod's veth.

Encapsulation adds about 50 bytes, so the pod MTU must be reduced (for example 1450) to avoid fragmentation. Without overlay (BGP or VPC CNI) the node just routes the packet.

</details>

### Q190. What is MTU and what is the typical problem it causes? 🟡

<details>
<summary><b>Answer</b></summary>

Maximum Transmission Unit is the largest L3 packet an interface sends without fragmentation: 1500 for Ethernet, 9001 for jumbo frames within an AWS VPC, 1500 across the internet. Tunnels and overlays shrink the usable size (IPsec about 1400, VXLAN minus 50, WireGuard minus 60).

Symptoms of MTU mismatch: small requests work, large responses and TLS handshakes with big certificates hang. Fix by lowering MTU or MSS clamping, and letting ICMP fragmentation-needed through.

</details>

### Q191. What is Path MTU Discovery and what is an MSS clamp? 🔴

<details>
<summary><b>Answer</b></summary>

Senders set the Don't Fragment bit; a router that cannot forward the packet returns ICMP "fragmentation needed" with its MTU, and the sender lowers its segment size. If ICMP is filtered, the sender never learns and the connection black-holes.

MSS clamping rewrites the TCP MSS option during the handshake so endpoints never send too-large segments:
```bash
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

</details>

### Q192. What is the difference between latency, bandwidth and throughput? 🟢

<details>
<summary><b>Answer</b></summary>

Latency is the time for a packet to travel (one-way or round trip). Bandwidth is the link's maximum capacity. Throughput is what you actually achieve, limited by latency, loss, window sizes, CPU and the application.

A high-bandwidth, high-latency link can still feel slow because TCP needs a full window in flight to use it (bandwidth-delay product).

</details>

### Q193. What network latency figures should you know? 🟡

<details>
<summary><b>Answer</b></summary>

Same AZ: well under 1 ms. Cross-AZ in a region: about 1–2 ms. Cross-region in the same continent: 20–80 ms. Transatlantic about 70–90 ms, trans-Pacific 120–180 ms. Light in fibre travels roughly 200,000 km/s, so 100 ms RTT is about 10,000 km of physical distance.

These numbers drive design choices: avoid chatty synchronous calls across regions, and place databases and app tiers in the same AZ where possible.

</details>

### Q194. What are the sources of tail latency in network paths? 🔴

<details>
<summary><b>Answer</b></summary>

Packet loss with RTO retransmits (200 ms minimum on Linux), bufferbloat and queueing, DNS timeouts and retries (5 s default), SYN retries (1 s, 2 s, 4 s), conntrack or NAT port exhaustion, noisy neighbors on shared network allowances, and GC pauses mistaken for network trouble.

The classic p99 spikes at 1 s, 3 s and 5 s come from SYN retransmit and DNS timeout defaults, which is the first fingerprint to check.

</details>

### Q195. What are Linux network namespaces and veth pairs? 🟡

<details>
<summary><b>Answer</b></summary>

A network namespace is an isolated copy of the network stack (interfaces, routes, iptables, sockets). A veth pair is a virtual cable with two ends; one end in the container namespace, the other in the host or a bridge. Docker and Kubernetes use them for every container or pod.

```bash
ip netns add demo
ip link add v0 type veth peer name v1
ip link set v1 netns demo
ip netns exec demo ip addr add 10.9.0.2/24 dev v1
```

</details>

### Q196. How do Docker network modes differ? 🟡

<details>
<summary><b>Answer</b></summary>

bridge (default): containers on a private NAT'd bridge (docker0). host: shares the host's network stack, no isolation or port mapping. none: loopback only. overlay: multi-host networking for Swarm. macvlan: gives containers their own MAC and LAN IP.

Port publishing (`-p 8080:80`) creates DNAT rules in iptables; user-defined bridges also give automatic DNS between containers by name.

</details>

### Q197. What is eBPF and how is it changing networking? 🔴

<details>
<summary><b>Answer</b></summary>

eBPF runs verified programs inside the Linux kernel at hooks such as XDP, tc and sockets. Networking uses: Cilium for load balancing and policy without iptables, XDP for line-rate DDoS filtering (Cloudflare, Meta Katran), and observability like Hubble, Pixie and bpftrace with low overhead.

It scales better than iptables, whose rule lookup is linear and whose updates become slow at tens of thousands of services.

</details>

### Q198. What is a service mesh, and what networking problems does it solve? 🔴

<details>
<summary><b>Answer</b></summary>

A mesh adds a proxy (sidecar Envoy, or per-node/ambient mode in Istio) to every workload to provide mTLS, retries, timeouts, traffic splitting, circuit breaking and uniform telemetry without changing application code. A control plane distributes config and certificates.

Cost: added latency (about 1–3 ms per hop with sidecars), CPU and memory per pod, operational complexity. Worth it for many services with security and traffic-management needs; overkill for a handful.

</details>

### Q199. What is a NodePort and when would you use one? 🟢

<details>
<summary><b>Answer</b></summary>

A NodePort Service opens the same port (30000–32767 by default) on every node and forwards it to the Service's pods. It is the building block under LoadBalancer Services and handy for quick tests or when an external load balancer targets nodes directly.

```bash
kubectl expose deploy web --type=NodePort --port=80
curl http://<node-ip>:<nodePort>
```
Do not expose NodePorts to the internet directly; put a load balancer in front and restrict the security group.

</details>

### Q200. What does `externalTrafficPolicy: Local` change on a LoadBalancer Service? 🔴

<details>
<summary><b>Answer</b></summary>

With the default `Cluster`, a node may forward traffic to a pod on another node, adding a hop and SNAT, so the pod loses the client source IP. With `Local`, a node only serves traffic to its own pods, preserving client IP and avoiding the extra hop, and the load balancer's health check removes nodes without local pods.

Trade-off: uneven distribution if pods are not spread evenly across nodes, so combine it with topology spread constraints.

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
