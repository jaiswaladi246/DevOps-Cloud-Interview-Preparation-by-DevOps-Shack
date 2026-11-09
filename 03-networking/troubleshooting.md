# Networking — Troubleshooting Drills

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

> Realistic network symptoms with exact error text, the commands we run to narrow them down, and the fixes that hold in production · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Networking](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Connectivity: refused, timed out, no route](#connectivity-refused-timed-out-no-route) — Q1–Q25
- [DNS resolution failures](#dns-resolution-failures) — Q26–Q50
- [TLS and certificate errors](#tls-and-certificate-errors) — Q51–Q75
- [MTU, routing, packet loss and latency](#mtu-routing-packet-loss-and-latency) — Q76–Q100
- [Sockets, ports, TIME_WAIT and conntrack](#sockets-ports-time_wait-and-conntrack) — Q101–Q125
- [Load balancers and reverse proxies](#load-balancers-and-reverse-proxies) — Q126–Q150
- [Cloud networking: VPC, firewalls, NAT, VPN and proxies](#cloud-networking-vpc-firewalls-nat-vpn-and-proxies) — Q151–Q175
- [Containers, Kubernetes, IPv6 and packet captures](#containers-kubernetes-ipv6-and-packet-captures) — Q176–Q200

---

## Connectivity: refused, timed out, no route

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. `curl: (7) Failed to connect to 10.0.2.15 port 8080: Connection refused` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The packet reached the host and the kernel answered with a TCP RST: nothing is listening on that port, the app is bound to `127.0.0.1` only, or a local firewall rule uses `REJECT --reject-with tcp-reset`.

**Diagnose →**
```bash
# on the server
ss -ltnp | grep 8080
systemctl status myapp
sudo iptables -S | grep -i reject
```

**Fix →** Start the service, or change the bind address from `127.0.0.1` to `0.0.0.0` (or the specific interface IP). Remove the reject rule if it is unintended.

**Prevent →** Add a readiness check that verifies the port is listening, and alert on process restarts.

</details>

### Q2. `curl: (28) Failed to connect to api.internal port 443 after 130001 ms: Connection timed out` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SYN packets got no reply at all. Something silently dropped them: security group, NACL, host firewall with `DROP`, a missing route back, or the host is down.

**Diagnose →**
```bash
nc -vz -w 3 api.internal 443
traceroute -T -p 443 api.internal
# on the target, are SYNs arriving?
sudo tcpdump -ni any 'tcp port 443 and tcp[tcpflags] & tcp-syn != 0'
```

**Fix →** If SYNs never arrive, fix the path (SG, NACL, route). If they arrive but no SYN-ACK leaves, fix the host firewall or listening service. If SYN-ACK leaves but the client never sees it, look for asymmetric return routing.

**Prevent →** Use short connect timeouts (`curl --connect-timeout 5`) so failures surface fast, and keep firewall rules in code.

</details>

### Q3. `ssh: connect to host 10.1.4.20 port 22: No route to host` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Either the local kernel has no route, ARP for the target (or gateway) failed on the local subnet, or a router/firewall replied with ICMP host-unreachable. firewalld's default reject also returns `icmp-host-prohibited`, which shows up as "No route to host".

**Diagnose →**
```bash
ip route get 10.1.4.20
ip neigh show 10.1.4.20        # FAILED / INCOMPLETE means ARP gave no answer
sudo tcpdump -ni any icmp      # look for "host unreachable" or "admin prohibited"
```

**Fix →** Add the missing route, bring the target interface up, or open the port in firewalld (`firewall-cmd --add-service=ssh --permanent && firewall-cmd --reload`).

**Prevent →** Know which hosts run firewalld/ufw and manage those rules with config management.

</details>

### Q4. Service works with `curl localhost:3000` on the box but times out from other hosts. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** App bound to loopback, host firewall blocks the port, or the cloud security group does not allow it.

**Diagnose →**
```bash
ss -ltnp | grep 3000     # 127.0.0.1:3000 vs 0.0.0.0:3000 / *:3000
sudo ufw status verbose; sudo nft list ruleset | head -50
```

**Fix →** Bind to `0.0.0.0` (Node: `app.listen(3000, '0.0.0.0')`; Flask: `--host 0.0.0.0`). Open the port in ufw/nftables and the security group from the right source CIDR only.

**Prevent →** Make bind address a config value and test from a second host in CI smoke tests.

</details>

### Q5. `ping` works but `curl` to port 80 on the same host times out. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ICMP is allowed but TCP/80 is filtered (SG, NACL, host firewall), or the service is down and a firewall drops instead of rejecting.

**Diagnose →**
```bash
nc -vz -w 3 host 80
sudo tcpdump -ni eth0 port 80     # on the server
```

**Fix →** Allow TCP/80 in the relevant filter. Ping only proves layer 3 reachability; it says nothing about a TCP port.

**Prevent →** Health-check the actual port and protocol, not ICMP.

</details>

### Q6. `curl` to a public site works, but `ping 8.8.8.8` shows 100% packet loss. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ICMP is blocked outbound by a security group/NACL or corporate firewall, or the cloud provider does not route ICMP through that egress path.

**Diagnose →**
```bash
curl -sS -o /dev/null -w '%{http_code}\n' https://example.com
traceroute -T -p 443 8.8.8.8      # TCP traceroute sidesteps ICMP echo filtering
```

**Fix →** Nothing to fix for apps. If you need ICMP for diagnostics, allow ICMP echo outbound and replies inbound in the NACL.

**Prevent →** Don't use ping as the single signal of "network up" in monitoring.

</details>

### Q7. `nc: connect to db.internal port 5432 (tcp) failed: Connection refused`, yet PostgreSQL is running. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `listen_addresses = 'localhost'` in `postgresql.conf`, Postgres listening on a different port, or you resolved to a different host than you think.

**Diagnose →**
```bash
sudo ss -ltnp | grep postgres
sudo -u postgres psql -c 'show listen_addresses; show port;'
getent hosts db.internal
```

**Fix →** Set `listen_addresses = '*'` (or the private IP), restart, then allow the client in `pg_hba.conf`. Note that a `pg_hba.conf` miss gives `no pg_hba.conf entry for host`, not "refused".

**Prevent →** Template DB configs and verify from an app host after every change.

</details>

### Q8. Connections to a server alternate between working and "Connection refused" on every other attempt. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The name or VIP maps to two backends and one has the service down; a load balancer with one dead target and no health check; duplicate IP on the subnet (two MACs answering ARP).

**Diagnose →**
```bash
dig +short app.internal                 # multiple A records?
for i in $(seq 10); do curl -s -o /dev/null -w '%{remote_ip} %{http_code}\n' http://app.internal; done
arping -c 4 -I eth0 10.0.1.50           # replies from two MACs = duplicate IP
```

**Fix →** Restart or remove the bad backend; fix the health check; remove the duplicate IP assignment.

**Prevent →** Use DHCP/IPAM to avoid manual IP reuse; health checks on every pool.

</details>

### Q9. A client gets `Connection reset by peer` in the middle of a request. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The server process crashed or closed the socket with unread data, an idle timeout on a middlebox (NAT, firewall, LB) expired and it sent RST, or a proxy hit a limit (request size, header size).

**Diagnose →**
```bash
sudo tcpdump -ni any host 10.0.3.7 and 'tcp[tcpflags] & tcp-rst != 0' -w rst.pcap
journalctl -u myapp --since -10min
dmesg -T | grep -i -E 'oom|killed'
```
Capture on both ends: if the server never sent the RST, a middlebox did.

**Fix →** Fix the crash, raise the middlebox idle timeout, or enable TCP keepalives below that timeout.

**Prevent →** Keep client idle timeouts shorter than every middlebox timeout in the path.

</details>

### Q10. `curl: (52) Empty reply from server` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** TCP connected but the server closed without sending any HTTP response: speaking HTTP to a TLS port, app crashed on the request, or a proxy dropped it.

**Diagnose →**
```bash
curl -v http://host:443/        # plain HTTP to a TLS port often gives this
curl -vk https://host:443/
journalctl -u myapp -n 50
```

**Fix →** Use the right scheme for the port, or fix the crashing handler.

**Prevent →** Separate ports cleanly (80 plain, 443 TLS) and log request start in the app.

</details>

### Q11. `curl: (56) Recv failure: Connection reset by peer` right after connecting through a corporate network. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A DPI firewall or IPS is resetting the flow based on SNI, destination or protocol, or an egress proxy is required and direct connections are being killed.

**Diagnose →**
```bash
curl -v https://target.example.com 2>&1 | head -30
curl -v -x http://proxy.corp:3128 https://target.example.com
sudo tcpdump -ni any host target.example.com   # RST arriving with odd TTL = injected by middlebox
```

**Fix →** Use the corporate proxy, or ask the network team to allow the destination category.

**Prevent →** Document egress requirements for each service and test from the real network early.

</details>

### Q12. A connection to a remote port hangs for exactly 21 seconds on Windows clients or ~127 seconds on Linux before failing. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SYN retransmissions with exponential backoff. Linux default `net.ipv4.tcp_syn_retries = 6` gives roughly 127 s. This always means silent drops, never a reachable closed port.

**Diagnose →**
```bash
sysctl net.ipv4.tcp_syn_retries
sudo tcpdump -ni any 'tcp[tcpflags] == tcp-syn' and host <dst>
```

**Fix →** Find the dropping filter. In the app, set connect timeouts explicitly rather than relying on kernel retries.

**Prevent →** Every client library call gets a connect timeout of a few seconds.

</details>

### Q13. `telnet: Unable to connect to remote host: Network is unreachable` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The local kernel has no route that matches the destination (often no default route), or the interface is down. This is `ENETUNREACH` and is raised locally before any packet leaves.

**Diagnose →**
```bash
ip route
ip -br link
ip route get 8.8.8.8
```

**Fix →** Bring the interface up, renew DHCP (`dhclient -v eth0` or `networkctl renew eth0`), or add the default route: `ip route add default via 10.0.0.1`.

**Prevent →** Persist routes in netplan/NetworkManager/systemd-networkd, not ad-hoc `ip route` commands.

</details>

### Q14. After a reboot, a server has no network although the interface shows UP. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Interface renamed (predictable names changed after a kernel/driver update), netplan or ifcfg references the old name, DHCP client not started, or cloud-init regenerated config.

**Diagnose →**
```bash
ip -br addr
journalctl -b -u systemd-networkd -u NetworkManager --no-pager | tail -50
cat /etc/netplan/*.yaml
```

**Fix →** Match on MAC in netplan (`match: macaddress:`) and `netplan apply`, or fix the name in the config. Use the serial console if SSH is gone.

**Prevent →** Match interfaces by MAC or driver, and test kernel upgrades on a canary host.

</details>

### Q15. Intermittent `Connection timed out` to a backend that is up and healthy, only under load. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SYN queue or accept queue overflow on the server, conntrack table full on a hop, NAT port exhaustion, or `tcp_tw_recycle`-style timestamp issues (old kernels) behind NAT.

**Diagnose →**
```bash
# server
nstat -az | grep -E 'ListenOverflows|ListenDrops|TcpExtSyncookies'
ss -ltn 'sport = :8080'     # Recv-Q near Send-Q = accept backlog full
dmesg -T | grep -E 'conntrack|SYN flooding'
```

**Fix →** Raise the app's listen backlog and `net.core.somaxconn`, add workers so `accept()` keeps up, raise `nf_conntrack_max`, or spread egress across more NAT IPs.

**Prevent →** Alert on `ListenOverflows` and conntrack usage; load-test with realistic connection churn.

</details>

### Q16. A TCP connection establishes, but the first request after it has sat idle for an hour hangs and eventually times out. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A stateful middlebox (NAT gateway, firewall, LB) evicted the idle flow. The client still thinks the connection exists, sends data, and packets are dropped silently. AWS NAT gateway idle timeout is 350 s; many firewalls use 3600 s.

**Diagnose →**
```bash
ss -tno state established '( dport = :5432 )'   # timer:(keepalive,...) shows if keepalives are on
sysctl net.ipv4.tcp_keepalive_time
```

**Fix →** Enable TCP keepalive at an interval shorter than the shortest idle timeout (e.g. `tcp_keepalive_time=300`, or `keepalives_idle=60` in the libpq connection string). Configure connection pools to validate or recycle idle connections.

**Prevent →** Document idle timeouts of every hop and set pool `max_idle_time` below them.

</details>

### Q17. Clients can connect from one subnet but not from another to the same service. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Source-based filtering: SG/NACL allows only one CIDR, the host firewall whitelists a range, the app has an allow list, or return routing for the second subnet is missing.

**Diagnose →**
```bash
sudo tcpdump -ni eth0 'src net 10.0.8.0/24 and tcp port 443'   # does it arrive?
ip route get 10.0.8.10                                           # how does the reply leave?
```

**Fix →** Allow the second CIDR, or add the return route. If the reply exits a different interface, enable policy routing or fix `rp_filter`.

**Prevent →** Reference security groups instead of CIDRs where possible.

</details>

### Q18. `connect: Cannot assign requested address` when the app opens outbound connections. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Ephemeral port exhaustion toward one destination IP:port (`EADDRNOTAVAIL`), or the app tries to bind to a source IP not on the host.

**Diagnose →**
```bash
sysctl net.ipv4.ip_local_port_range
ss -tan state time-wait | wc -l
ss -tan dst 10.0.5.9:443 | wc -l
```

**Fix →** Reuse connections (HTTP keep-alive, pooling), widen `ip_local_port_range` (e.g. `1024 65000`), enable `net.ipv4.tcp_tw_reuse=1` for outbound, or spread traffic across more destination IPs.

**Prevent →** Track outbound connection rate and TIME_WAIT counts; insist on pooling in client libraries.

</details>

### Q19. `bind: Address already in use` when restarting a service. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Old process still running, another service owns the port, or the listening socket is lingering and the app doesn't set `SO_REUSEADDR`.

**Diagnose →**
```bash
sudo ss -ltnp 'sport = :8080'
sudo lsof -i :8080
```

**Fix →** Stop the old process or change the port. In your own code set `SO_REUSEADDR` on listening sockets.

**Prevent →** Use systemd with proper `ExecStop` and avoid `kill -9`, which leaves orphaned children holding sockets.

</details>

### Q20. A service is reachable by IP but not by hostname. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** DNS problem, wrong `/etc/hosts` entry, or a virtual-host-based server returns a different site for the Host header.

**Diagnose →**
```bash
getent hosts app.example.com
dig app.example.com +short
curl -v --resolve app.example.com:443:203.0.113.10 https://app.example.com/
```

**Fix →** Fix the record or hosts file. If DNS is right but the response differs, check the server's `server_name` / vhost config.

**Prevent →** Monitor the public name, not just the IP.

</details>

### Q21. Traffic to a peer host works one way only: A can SSH to B, but B cannot SSH to A. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Inbound filters on A (SG, host firewall), sshd not listening on A's interface, or NAT in one direction only.

**Diagnose →**
```bash
# on A
sudo ss -ltnp | grep :22
sudo tcpdump -ni any tcp port 22 and host <B>
```

**Fix →** Open inbound 22 on A from B. Remember stateful filters let replies back automatically; new connections in the other direction still need explicit rules.

**Prevent →** Model flows as "who initiates" when writing rules.

</details>

### Q22. Packets arrive on the server (seen in tcpdump), but the application never gets them. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** tcpdump captures before netfilter on ingress, so a host firewall `INPUT` drop is invisible to it. Also: reverse-path filter (`rp_filter`) drops packets whose source would route out another interface, or the destination IP isn't on that host.

**Diagnose →**
```bash
sudo iptables -L INPUT -v -n --line-numbers   # watch packet counters climb on a DROP rule
sysctl net.ipv4.conf.all.rp_filter net.ipv4.conf.eth1.rp_filter
nstat -az | grep -i IPReversePathFilter
```

**Fix →** Remove or adjust the drop rule; set `rp_filter=2` (loose) on multi-homed hosts or add policy routing so replies use the same interface.

**Prevent →** Remember the order: capture point → netfilter → socket. Use `nft monitor trace` or iptables `TRACE` to follow a packet.

</details>

### Q23. `curl -v` hangs at `* Trying 2606:4700::6810:84e5:443...` for a long time, then succeeds. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The host has an IPv6 address and route, tries AAAA first, but IPv6 is broken somewhere upstream. curl's Happy Eyeballs falls back to IPv4 after a delay; older clients just wait for the full timeout.

**Diagnose →**
```bash
curl -6 -v --connect-timeout 5 https://example.com
curl -4 -v https://example.com
ip -6 route; ping -6 -c 3 2606:4700:4700::1111
```

**Fix →** Fix IPv6 routing/firewall, or if IPv6 isn't supported, remove the IPv6 default route or prefer IPv4 via `/etc/gai.conf` (`precedence ::ffff:0:0/96 100`).

**Prevent →** Treat IPv6 as a first-class path in monitoring if it is enabled anywhere.

</details>

### Q24. You need to prove whether a specific firewall is dropping or rejecting traffic. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You suspect a filter but don't know which hop.

**Diagnose →**
```bash
# Refused vs timeout tells reject vs drop
nc -vz -w 3 target 8443
# TCP traceroute on the real port shows the last hop that answers
sudo traceroute -T -p 8443 target
sudo mtr -T -P 8443 -rwc 50 target
# Capture both sides of the suspected firewall at the same time
sudo tcpdump -ni eth0 host target and port 8443
```
Packets visible before the firewall but not after prove it drops them. An RST or ICMP admin-prohibited with the firewall's IP or TTL proves it rejects.

**Fix →** Change the rule with evidence attached.

**Prevent →** Enable firewall deny logging (flow logs, `nft log`) so the next time you can read the drop directly.

</details>

### Q25. Users see `ERR_CONNECTION_REFUSED` in the browser for your site, but `curl` from your laptop works. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Users resolve to a different IP (old DNS, geo routing, IPv6 AAAA pointing to a host that doesn't listen on v6), or a browser extension/proxy.

**Diagnose →**
```bash
dig A app.example.com +short; dig AAAA app.example.com +short
curl -6 -v https://app.example.com
# Ask a user for: nslookup app.example.com and the IP shown in browser devtools (Network > Remote Address)
```

**Fix →** Remove a stale AAAA record or make the service listen on IPv6 (`listen [::]:443 ssl;` in nginx). Fix the stale record.

**Prevent →** Synthetic checks from multiple regions over both IPv4 and IPv6.

</details>

---

## DNS resolution failures

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. `dig app.example.com` returns `status: NXDOMAIN`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The name doesn't exist in the zone: typo, record not created, record in the wrong zone (e.g. private hosted zone not associated), or delegation pointing to servers that don't have it.

**Diagnose →**
```bash
dig app.example.com +noall +answer +authority
dig NS example.com +short
dig @$(dig NS example.com +short | head -1) app.example.com   # ask authoritative directly
```

**Fix →** Create the record in the zone that the registrar actually delegates to. Compare registrar NS with the hosted zone's NS set.

**Prevent →** Manage records in Terraform/OctoDNS and check delegation in CI.

</details>

### Q27. `dig` returns `status: SERVFAIL` for one domain while other domains resolve fine. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The resolver can't get a valid answer from the authoritative servers: all of them are unreachable, lame delegation, or DNSSEC validation failure (expired RRSIG, wrong DS at the parent).

**Diagnose →**
```bash
dig app.example.com @1.1.1.1
dig app.example.com @1.1.1.1 +cd        # +cd disables validation; works now = DNSSEC problem
dig +trace app.example.com
delv app.example.com                   # explains validation failures
```

**Fix →** For DNSSEC: fix signing or remove the DS record at the registrar. For lame delegation: make NS records at the parent match servers that really serve the zone.

**Prevent →** Monitor DNSSEC signature expiry and NS consistency externally.

</details>

### Q28. `curl: (6) Could not resolve host: api.example.com` inside a container but it resolves on the host. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The container's `/etc/resolv.conf` points at `127.0.0.53` (systemd-resolved stub) copied from the host, which isn't reachable inside the container namespace; or a custom network's embedded DNS (127.0.0.11) can't reach upstream.

**Diagnose →**
```bash
docker exec app cat /etc/resolv.conf
docker exec app getent hosts api.example.com
resolvectl status
```

**Fix →** Set daemon DNS in `/etc/docker/daemon.json` (`"dns": ["10.0.0.2"]`) or per-container `--dns`. Docker normally rewrites a loopback resolver to defaults; corporate DNS needs to be explicit.

**Prevent →** Pin resolver configuration for the runtime rather than inheriting host defaults.

</details>

### Q29. A DNS change was made 2 hours ago, but some users still reach the old IP. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Resolvers cached the old record for its previous TTL; some clients (JVM, browsers, app-level caches) cache longer; or the change wasn't made on all authoritative servers.

**Diagnose →**
```bash
for ns in $(dig NS example.com +short); do dig @$ns app.example.com +short; done
dig app.example.com @8.8.8.8 +noall +answer   # remaining TTL shown in the answer
```

**Fix →** Wait out the old TTL, flush what you control (`resolvectl flush-caches`, restart the JVM), and keep the old endpoint running until traffic drains.

**Prevent →** Lower the TTL to 60 s a day before planned changes.

</details>

### Q30. Java service keeps connecting to an old IP long after the DNS record changed. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** JVM DNS cache. With a security manager installed, `networkaddress.cache.ttl` defaults to forever; otherwise 30 s. Connection pools also keep existing connections to the old IP.

**Diagnose →**
```bash
jcmd <pid> VM.system_properties | grep -i networkaddress
grep -i networkaddress $JAVA_HOME/conf/security/java.security
ss -tnp | grep java
```

**Fix →** Set `networkaddress.cache.ttl=60` in `java.security` or `-Dsun.net.inetaddr.ttl=60`, and set pool max connection lifetime so connections reopen.

**Prevent →** Standardize JVM DNS TTL in base images.

</details>

### Q31. In Kubernetes, external lookups from pods are slow and CoreDNS logs show many NXDOMAIN queries like `api.stripe.com.default.svc.cluster.local`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `ndots:5` in pod `resolv.conf`. Any name with fewer than five dots is tried with every search domain first, so one external lookup becomes up to 4–5 queries times A and AAAA.

**Diagnose →**
```bash
kubectl exec -it pod -- cat /etc/resolv.conf
kubectl -n kube-system logs -l k8s-app=kube-dns | grep NXDOMAIN | head
```

**Fix →** Use fully qualified names with a trailing dot (`api.stripe.com.`), or lower ndots per pod:
```yaml
dnsConfig:
  options:
    - name: ndots
      value: "2"
```

**Prevent →** Deploy NodeLocal DNSCache and enable CoreDNS `autopath` if it fits; set ndots in workload templates.

</details>

### Q32. Pods intermittently log `dial tcp: lookup orders on 10.96.0.10:53: i/o timeout`, about 5 seconds per failure. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The classic conntrack race: glibc sends A and AAAA queries in parallel from the same socket, both UDP packets hit DNAT at once, one loses the conntrack insert and is dropped, and the resolver waits its 5 s timeout. Also CoreDNS overload or too few replicas.

**Diagnose →**
```bash
conntrack -S | grep insert_failed          # on nodes; rising = the race
kubectl -n kube-system top pod -l k8s-app=kube-dns
```

**Fix →** Add `options single-request-reopen` (glibc) via `dnsConfig`, deploy NodeLocal DNSCache (avoids DNAT for DNS), or switch to a CNI that avoids the race. Scale CoreDNS.

**Prevent →** Alert on CoreDNS latency and `insert_failed`; run NodeLocal DNSCache by default.

</details>

### Q33. `nslookup` works but the application says `Name or service not known`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `nslookup` talks to the DNS server directly; the app goes through glibc NSS (`/etc/nsswitch.conf`, `/etc/hosts`, search domains). A broken hosts entry, missing `dns` in nsswitch, or a short name without the right search domain breaks the app only.

**Diagnose →**
```bash
getent ahosts myservice          # resolves the same way the app does
grep hosts /etc/nsswitch.conf
cat /etc/resolv.conf
```

**Fix →** Fix nsswitch/hosts/search domains, or use FQDNs in app config.

**Prevent →** Test with `getent`, not `nslookup`/`dig`, when debugging application resolution.

</details>

### Q34. After enabling systemd-resolved, `dig` to `127.0.0.53` works, but VPN-internal names don't resolve. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Split DNS isn't configured: the VPN's DNS server is attached to the tunnel link but no routing domain (`~corp.internal`) tells resolved to send those names there.

**Diagnose →**
```bash
resolvectl status
resolvectl query host.corp.internal
```

**Fix →**
```bash
sudo resolvectl dns tun0 10.8.0.1
sudo resolvectl domain tun0 '~corp.internal'
```
Most VPN clients do this via a NetworkManager plugin or `update-systemd-resolved` script.

**Prevent →** Ship the VPN client with resolved integration configured.

</details>

### Q35. Route 53 private hosted zone records return NXDOMAIN from an EC2 instance. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The PHZ isn't associated with this VPC, `enableDnsHostnames`/`enableDnsSupport` is off, the instance uses a custom resolver instead of the VPC resolver (`VPC CIDR base + 2`), or a public zone with the same name shadows it.

**Diagnose →**
```bash
cat /etc/resolv.conf
dig @169.254.169.253 db.internal.example.com
aws route53 get-hosted-zone --id Z123 --query 'VPCs'
aws ec2 describe-vpc-attribute --vpc-id vpc-abc --attribute enableDnsSupport
```

**Fix →** Associate the zone with the VPC (cross-account needs authorization + association), enable DNS attributes, or forward the domain from the custom resolver to the VPC resolver.

**Prevent →** Associate PHZs in the same Terraform module that creates VPCs.

</details>

### Q36. `dig +short` returns the right answer, but `dig @8.8.8.8` returns a different (old) IP. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Your local resolver has the new answer; Google's cache still holds the old one until its TTL expires. Or split-horizon: your local resolver serves an internal view.

**Diagnose →**
```bash
dig app.example.com @8.8.8.8 +noall +answer    # check remaining TTL
dig app.example.com @<authoritative-ns> +short
```

**Fix →** Wait for the TTL, or use Google's cache flush page for public names. If it's split-horizon, that's expected behaviour.

**Prevent →** Lower TTLs before migrations.

</details>

### Q37. A newly created record returns NXDOMAIN for 15 minutes even after it exists on the authoritative servers. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Negative caching. Someone queried the name before it existed, and resolvers cached the NXDOMAIN for the SOA minimum / negative TTL.

**Diagnose →**
```bash
dig SOA example.com +noall +answer     # last field = negative TTL
dig app.example.com @<resolver> +noall +authority
```

**Fix →** Wait, or flush the resolvers you control (`unbound-control flush app.example.com`, `rndc flushname`).

**Prevent →** Create records before anything queries them, and keep the SOA negative TTL modest (300 s).

</details>

### Q38. DNS lookups fail with `;; Truncated, retrying in TCP mode` then `connection timed out; no servers could be reached`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The response is larger than the UDP limit (big TXT, many records, DNSSEC), the resolver retries over TCP/53, and TCP/53 is blocked by a firewall or NACL.

**Diagnose →**
```bash
dig TXT example.com +notcp
dig TXT example.com +tcp
dig TXT example.com +bufsize=1232
```

**Fix →** Allow TCP/53 alongside UDP/53 everywhere DNS flows. It is required, not optional.

**Prevent →** Firewall templates always pair UDP and TCP 53; keep TXT records trimmed.

</details>

### Q39. CoreDNS pods are in CrashLoopBackOff with `Loop (127.0.0.1:XXXX -> :53) detected for zone "."`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CoreDNS forwards to `/etc/resolv.conf` of the node, which points to `127.0.0.53` (systemd-resolved) or itself, so queries loop.

**Diagnose →**
```bash
kubectl -n kube-system logs deploy/coredns
cat /etc/resolv.conf     # on the node
```

**Fix →** Point kubelet at the real upstream file: `resolvConf: /run/systemd/resolve/resolv.conf` in the kubelet config, or change the Corefile `forward . 10.0.0.2`.

**Prevent →** Bake the correct kubelet `resolvConf` into node images.

</details>

### Q40. Every DNS lookup on a Linux host takes exactly 5 seconds before succeeding. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The first nameserver in `resolv.conf` is dead and glibc waits `timeout:5` before trying the next, or AAAA queries are dropped (parallel A/AAAA issue).

**Diagnose →**
```bash
cat /etc/resolv.conf
for ns in $(awk '/^nameserver/{print $2}' /etc/resolv.conf); do dig @$ns example.com +time=2 +tries=1 | grep 'Query time'; done
strace -f -e trace=network getent hosts example.com 2>&1 | head -40
```

**Fix →** Remove the dead server. Add `options timeout:1 attempts:2 rotate` while you fix the root cause, or `single-request-reopen` for the A/AAAA case.

**Prevent →** Monitor each resolver individually.

</details>

### Q41. Internal names resolve on the office network but return public IPs (or NXDOMAIN) over the VPN. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** VPN clients use public DNS instead of the internal resolver, so they get the external view of a split-horizon zone.

**Diagnose →**
```bash
scutil --dns            # macOS
resolvectl status       # Linux
nslookup intranet.example.com
```

**Fix →** Push the internal DNS server and search domains in the VPN profile (`push "dhcp-option DNS 10.0.0.2"` in OpenVPN; `DNS =` in WireGuard config).

**Prevent →** Include DNS checks in the VPN onboarding checklist.

</details>

### Q42. Your domain stopped resolving everywhere at once after a registrar renewal. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Domain expired or went into hold, the registrar reset nameservers to parking, or a DS record mismatch after the renewal.

**Diagnose →**
```bash
whois example.com | grep -i -E 'status|expir|name server'
dig NS example.com @a.gtld-servers.net
```

**Fix →** Renew and restore the correct NS at the registrar. Remove or fix the DS record if DNSSEC breaks.

**Prevent →** Auto-renew, registrar lock, expiry monitoring, and more than one person on the registrar account.

</details>

### Q43. `dig` shows the right record from all authoritative NS, but one region's users get SERVFAIL. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** One of your anycast/authoritative instances is unreachable or serving stale data in that region, EDNS/fragmentation issues on that path (large responses dropped), or an ISP resolver with broken DNSSEC handling.

**Diagnose →**
```bash
dig app.example.com @ns1.example.com +nsid     # which instance answered
dig app.example.com +bufsize=1232 +dnssec
# Use RIPE Atlas or a multi-region checker to query from that region
```

**Fix →** Fix or withdraw the bad instance; cap EDNS buffer at 1232 bytes on authoritative servers to avoid fragmentation.

**Prevent →** Multi-region DNS monitoring that checks every NS from multiple vantage points.

</details>

### Q44. A Kubernetes pod can resolve `orders` but not `orders.payments`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `orders` resolves in the pod's own namespace; the Service in namespace `payments` doesn't exist, has a different name, or a NetworkPolicy blocks DNS from that pod.

**Diagnose →**
```bash
kubectl get svc -n payments
kubectl exec pod -- nslookup orders.payments.svc.cluster.local
```

**Fix →** Use the right name/namespace or create the Service.

**Prevent →** Prefer `<svc>.<ns>.svc.cluster.local` in cross-namespace config.

</details>

### Q45. After applying a default-deny NetworkPolicy, every pod fails with `could not resolve host`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Egress to CoreDNS (UDP and TCP 53 in `kube-system`) is now blocked.

**Diagnose →**
```bash
kubectl get networkpolicy -A
kubectl exec pod -- nslookup kubernetes.default
```

**Fix →**
```yaml
egress:
  - to:
      - namespaceSelector:
          matchLabels: { kubernetes.io/metadata.name: kube-system }
        podSelector:
          matchLabels: { k8s-app: kube-dns }
    ports:
      - { protocol: UDP, port: 53 }
      - { protocol: TCP, port: 53 }
```

**Prevent →** Ship an "allow DNS" policy alongside every default-deny.

</details>

### Q46. `resolv.conf` keeps being overwritten and your manual nameserver change disappears. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** NetworkManager, systemd-resolved, dhclient or cloud-init manage the file.

**Diagnose →**
```bash
ls -l /etc/resolv.conf     # symlink to ../run/systemd/resolve/stub-resolv.conf?
head -3 /etc/resolv.conf   # "Generated by ..." line
```

**Fix →** Configure DNS at the owner: netplan `nameservers:`, `nmcli con mod eth0 ipv4.dns 10.0.0.2`, or `/etc/systemd/resolved.conf` `DNS=`.

**Prevent →** Never edit generated files; manage the source config with Ansible.

</details>

### Q47. An internal resolver stopped answering and `/var/log/syslog` shows `named: client ... query (cache) denied`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** BIND `allow-query` / `allow-recursion` ACLs don't include the client's subnet (new VPC or subnet added).

**Diagnose →**
```bash
named-checkconf
grep -A5 -E 'allow-(query|recursion)' /etc/bind/named.conf.options
dig @resolver example.com      # status: REFUSED
```

**Fix →** Add the new CIDR to the ACL and `rndc reconfig`.

**Prevent →** Generate resolver ACLs from the IPAM source of truth.

</details>

### Q48. A pod's DNS queries succeed but take 1–2 seconds when the cluster is busy. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CoreDNS CPU throttling (low CPU limit), too few replicas, upstream resolver latency, or the VPC resolver per-ENI limit (1024 packets/s per ENI in AWS) being hit.

**Diagnose →**
```bash
kubectl -n kube-system top pod -l k8s-app=kube-dns
# Prometheus: coredns_dns_request_duration_seconds, coredns_forward_request_duration_seconds
# AWS: ethtool -S eth0 | grep linklocal_allowance_exceeded
```

**Fix →** Remove CPU limits or raise them, scale CoreDNS (cluster-proportional-autoscaler), raise cache TTL, deploy NodeLocal DNSCache to spread queries across nodes.

**Prevent →** Dashboards for CoreDNS latency and `linklocal_allowance_exceeded`.

</details>

### Q49. `dig` against a public name returns `status: REFUSED`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You asked a server that isn't recursive for you: an authoritative-only server for a zone it doesn't host, or a resolver with ACLs that exclude you.

**Diagnose →**
```bash
dig example.org @ns1.mycompany.com    # authoritative-only server -> REFUSED
dig example.org @10.0.0.2
```

**Fix →** Query a recursive resolver you are allowed to use, or add your network to its ACL.

**Prevent →** Keep authoritative and recursive roles on separate servers.

</details>

### Q50. CNAME chain resolves, but the final name returns NXDOMAIN for some users. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CNAME target (e.g. a CDN or SaaS hostname) was deleted or is regional; dangling CNAME after deprovisioning; or resolvers cached a negative answer for the target.

**Diagnose →**
```bash
dig www.example.com +noall +answer    # follow each hop
dig <target-hostname> @8.8.8.8
```

**Fix →** Point the CNAME at a live target or remove it immediately. A dangling CNAME to a released cloud resource is a subdomain takeover risk.

**Prevent →** Delete DNS records in the same change that removes the resource; scan for dangling records.

</details>

---
## TLS and certificate errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. `curl: (60) SSL certificate problem: certificate has expired` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The leaf certificate passed its `notAfter`, an intermediate in the served chain expired, or the client clock is wrong.

**Diagnose →**
```bash
echo | openssl s_client -connect app.example.com:443 -servername app.example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
date -u; timedatectl
```

**Fix →** Renew and deploy the certificate, then reload the server (`nginx -s reload`). If the client clock is off, fix NTP (`chronyc tracking`).

**Prevent →** Automate renewal (ACME/cert-manager/ACM) and alert at 30 and 14 days before expiry from an external probe.

</details>

### Q52. `curl: (60) SSL certificate problem: unable to get local issuer certificate` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The server sends only the leaf without the intermediate, the client's CA bundle lacks the root (private CA, old OS), or a TLS-inspecting proxy re-signs traffic with a corporate CA the client doesn't trust.

**Diagnose →**
```bash
openssl s_client -connect app.example.com:443 -servername app.example.com -showcerts </dev/null | grep -E 's:|i:'
# Only one certificate shown = missing intermediate
```

**Fix →** Serve the full chain (`ssl_certificate fullchain.pem` in nginx). For a private or proxy CA, add it to the trust store: `cp corp-ca.crt /usr/local/share/ca-certificates/ && update-ca-certificates`.

**Prevent →** Test endpoints with `openssl s_client -verify_return_error` in CI, not just a browser (browsers fetch missing intermediates themselves).

</details>

### Q53. `SSL: no alternative certificate subject name matches target host name 'api.example.com'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The certificate's SAN list doesn't include the hostname used, you connected by IP, or the server returned its default certificate because SNI didn't match any vhost.

**Diagnose →**
```bash
echo | openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null \
  | openssl x509 -noout -ext subjectAltName
```

**Fix →** Issue a certificate that includes the name (wildcards cover one label only: `*.example.com` doesn't match `a.b.example.com`), or fix the vhost so it serves the right certificate.

**Prevent →** Generate SANs from the same config that defines hostnames.

</details>

### Q54. Some clients get the wrong certificate (another site's) from a shared IP. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Clients that don't send SNI (very old clients, some health checkers, scripts using raw sockets) get the default server block's certificate. Or `server_name` doesn't match and nginx falls back to the default.

**Diagnose →**
```bash
openssl s_client -connect 203.0.113.10:443 </dev/null 2>/dev/null | openssl x509 -noout -subject      # no SNI
openssl s_client -connect 203.0.113.10:443 -servername shop.example.com </dev/null 2>/dev/null | openssl x509 -noout -subject
```

**Fix →** Fix `server_name`, make the most important site the `default_server`, or upgrade clients to send SNI.

**Prevent →** Include a no-SNI check in synthetic monitoring if legacy clients matter.

</details>

### Q55. `SSL routines::sslv3 alert handshake failure` from `openssl` or curl. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No overlap in protocol versions or cipher suites, the server requires a client certificate (mTLS), or SNI is missing/wrong and the server has no default cert.

**Diagnose →**
```bash
openssl s_client -connect host:443 -servername host -tls1_2
openssl s_client -connect host:443 -servername host -tls1_3
nmap --script ssl-enum-ciphers -p 443 host
```

**Fix →** Enable a common protocol (TLS 1.2 + 1.3) and modern ECDHE ciphers on both sides. If the server asks for a client certificate, send it (`curl --cert client.crt --key client.key`).

**Prevent →** Use a standard TLS profile (Mozilla "intermediate") and test old clients before tightening.

</details>

### Q56. Python `requests` fails with `SSLError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: self-signed certificate in certificate chain`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A corporate TLS-inspection proxy or a private CA. `requests` uses `certifi`'s bundle, not the OS trust store.

**Diagnose →**
```bash
python -c "import certifi; print(certifi.where())"
openssl s_client -connect pypi.org:443 -servername pypi.org -showcerts </dev/null | grep 'i:'
```

**Fix →** Point at the right bundle: `export REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt` (after adding the corporate CA to the OS store), or install `truststore`/`pip-system-certs`. Never set `verify=False` in production.

**Prevent →** Bake the corporate CA into base images and set `REQUESTS_CA_BUNDLE`, `NODE_EXTRA_CA_CERTS` and the Java truststore in one place.

</details>

### Q57. Node.js app logs `Error: unable to verify the first certificate` calling an internal API. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The internal API serves a leaf without its intermediate, or is signed by a private CA Node doesn't know (Node ships its own CA list).

**Diagnose →**
```bash
openssl s_client -connect internal-api:443 -servername internal-api -showcerts </dev/null
```

**Fix →** Serve the full chain on the API, or start Node with `NODE_EXTRA_CA_CERTS=/etc/ssl/private-ca.pem`. Avoid `NODE_TLS_REJECT_UNAUTHORIZED=0`.

**Prevent →** Issue internal certs from a CA whose root is distributed to all images.

</details>

### Q58. Java client fails with `PKIX path building failed: unable to find valid certification path to requested target`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The JVM truststore (`cacerts`) lacks the issuing CA, the server sends an incomplete chain, or the app uses a custom truststore that overrides defaults.

**Diagnose →**
```bash
keytool -list -cacerts -storepass changeit | grep -i myca
java -Djavax.net.debug=ssl:handshake -jar app.jar 2>&1 | grep -i -A3 'certificate'
```

**Fix →** Import the CA: `keytool -importcert -cacerts -storepass changeit -alias myca -file myca.crt -noprompt`, or fix the server chain.

**Prevent →** Build images with the CA imported; check `-Djavax.net.ssl.trustStore` overrides.

</details>

### Q59. Everything broke at once for clients on old Android/OpenSSL versions after a CA root expired. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Your chain depends on a cross-signed or expired root that those clients still try to build a path to (the 2021 Let's Encrypt DST Root CA X3 expiry is the classic case). OpenSSL 1.0.x fails if any path ends at an expired root.

**Diagnose →**
```bash
openssl s_client -connect host:443 -servername host -showcerts </dev/null | grep -E 's:|i:'
openssl version     # on the failing client
```

**Fix →** Serve a chain that ends at a root the clients have, update client CA bundles (`ca-certificates` package), or remove the expired root from client trust stores.

**Prevent →** Track CA announcements; test with old clients in a compatibility matrix.

</details>

### Q60. Certificates renewed by certbot, but browsers still show the old expiry date. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** nginx/Apache wasn't reloaded so the old cert stays in memory, the server config points at a copied file rather than `/etc/letsencrypt/live/...`, or a CDN/LB in front has its own certificate.

**Diagnose →**
```bash
certbot certificates
grep -R ssl_certificate /etc/nginx/
echo | openssl s_client -connect host:443 -servername host 2>/dev/null | openssl x509 -noout -dates
```

**Fix →** `systemctl reload nginx`, point config at the `live/` symlinks, and add a deploy hook: `certbot renew --deploy-hook "systemctl reload nginx"`.

**Prevent →** Monitor the served expiry date, not the file on disk.

</details>

### Q61. cert-manager Certificate stuck with `Waiting for HTTP-01 challenge propagation: wrong status code '404', expected '200'`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The solver Ingress isn't picked up (wrong `ingressClassName`), DNS for the name doesn't point at the ingress yet, an HTTP→HTTPS redirect or auth middleware intercepts `/.well-known/acme-challenge/`, or a CDN caches the path.

**Diagnose →**
```bash
kubectl describe certificate app-tls
kubectl get challenges -A
kubectl describe challenge <name>
curl -v http://app.example.com/.well-known/acme-challenge/test
```

**Fix →** Set the right ingress class on the Issuer solver, fix DNS, exclude the challenge path from redirects and auth. For wildcards or private names use DNS-01.

**Prevent →** Alert on Certificates not Ready; prefer DNS-01 for internal hosts.

</details>

### Q62. Let's Encrypt renewal fails with `too many certificates already issued for exact set of domains`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rate limit hit (duplicate certificate limit), usually from a loop that requests a new certificate on every deploy or a pod that keeps no persistent state.

**Diagnose →**
```bash
kubectl get certificaterequests -A | wc -l
# Check crt.sh for how many certs were issued for the name recently
```

**Fix →** Stop the loop, persist certificates (cert-manager Secrets, certbot volume), and wait for the window. Use the staging endpoint while testing.

**Prevent →** Test ACME flows against staging; never issue certs in container entrypoints.

</details>

### Q63. mTLS between services fails with `tls: bad certificate` on the server side logs. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The client presented no cert or one signed by a CA not in the server's client-CA bundle, the client cert lacks `clientAuth` in Extended Key Usage, or it expired.

**Diagnose →**
```bash
openssl x509 -in client.crt -noout -issuer -dates -ext extendedKeyUsage
openssl s_client -connect svc:8443 -servername svc -cert client.crt -key client.key -CAfile ca.crt
```

**Fix →** Issue the client cert from the trusted CA with `extendedKeyUsage = clientAuth`, and put that CA in the server's `ssl_client_certificate` / `ClientCAs`.

**Prevent →** Automate workload certificates (SPIFFE/SPIRE, mesh) and monitor rotation.

</details>

### Q64. `curl: (35) error:0A00010B:SSL routines::wrong version number` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Client speaks TLS to a port that answers plain HTTP (e.g. `https://host:80`, or a proxy port expecting HTTP CONNECT), or `https_proxy` is set to `https://proxy:3128` when the proxy speaks plain HTTP.

**Diagnose →**
```bash
curl -v http://host:8080/
env | grep -i proxy
```

**Fix →** Use the correct scheme/port; set `https_proxy=http://proxy:3128` (the proxy URL scheme describes how you talk to the proxy).

**Prevent →** Keep port conventions consistent and document proxy variables.

</details>

### Q65. TLS handshake takes 2–3 seconds from some locations. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Long RTT multiplied by round trips (TLS 1.2 = 2 RTT), OCSP checks by the client to a slow responder, a huge chain, or the server is CPU-bound on RSA 4096.

**Diagnose →**
```bash
curl -o /dev/null -s -w 'dns:%{time_namelookup} tcp:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer}\n' https://host/
openssl s_client -connect host:443 -servername host -status </dev/null | grep -A2 'OCSP'
```

**Fix →** Enable TLS 1.3, OCSP stapling, session resumption, ECDSA certificates, and terminate TLS closer to users (CDN).

**Prevent →** Track `time_appconnect` from multiple regions in synthetic checks.

</details>

### Q66. After switching to an ECDSA-only certificate, some legacy clients fail to connect. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Old clients support only RSA key exchange/signatures.

**Diagnose →**
```bash
openssl s_client -connect host:443 -servername host -sigalgs RSA+SHA256 </dev/null
# Check access logs or LB metrics for TLS negotiation errors by client type
```

**Fix →** Serve dual certificates (RSA + ECDSA). nginx supports two `ssl_certificate` directives; most LBs accept both.

**Prevent →** Test changes against the client compatibility list before rollout.

</details>

### Q67. Through an ALB, clients see `ERR_SSL_PROTOCOL_ERROR` after you attached a new certificate. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The listener's security policy is too strict for clients, the new ACM cert is for a different name so SNI selects no match, or the cert in ACM is still `PENDING_VALIDATION`.

**Diagnose →**
```bash
aws acm describe-certificate --certificate-arn <arn> --query 'Certificate.[Status,DomainName,SubjectAlternativeNames]'
aws elbv2 describe-listeners --load-balancer-arn <lb> --query 'Listeners[].SslPolicy'
openssl s_client -connect lb-dns:443 -servername app.example.com
```

**Fix →** Validate the certificate (DNS CNAME), add it to the listener certificate list, and pick a policy like `ELBSecurityPolicy-TLS13-1-2-2021-06`.

**Prevent →** Use ACM DNS validation records in Terraform so renewals stay automatic.

</details>

### Q68. `x509: certificate signed by unknown authority` from a Go binary in a scratch/distroless image. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The image has no CA bundle at `/etc/ssl/certs/ca-certificates.crt`.

**Diagnose →**
```bash
docker run --rm --entrypoint ls myimage /etc/ssl/certs    # fails or empty in scratch
```

**Fix →** Copy the bundle in a multi-stage build:
```dockerfile
COPY --from=alpine:3.20 /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
```
or use `gcr.io/distroless/static`, which includes CA certificates.

**Prevent →** Use base images with CA bundles and test outbound HTTPS in CI.

</details>

### Q69. A backend behind nginx (`proxy_pass https://...`) fails with `upstream SSL certificate does not match "10.0.4.12"`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `proxy_ssl_verify on` with `proxy_pass` to an IP or upstream name, and nginx isn't sending the right SNI / verifying against the right name.

**Diagnose →**
```bash
grep -n -E 'proxy_ssl|proxy_pass' /etc/nginx/conf.d/*.conf
tail -n 20 /var/log/nginx/error.log
```

**Fix →**
```nginx
proxy_ssl_server_name on;
proxy_ssl_name backend.internal.example.com;
proxy_ssl_trusted_certificate /etc/nginx/ca.pem;
proxy_ssl_verify on;
```

**Prevent →** Use hostnames for TLS upstreams and keep verification on.

</details>

### Q70. A browser shows `NET::ERR_CERT_COMMON_NAME_INVALID` even though the CN matches the hostname. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Browsers ignore CN; they only check the Subject Alternative Name extension. Self-made certificates often have CN only.

**Diagnose →**
```bash
openssl x509 -in cert.pem -noout -ext subjectAltName
```

**Fix →** Reissue with SANs:
```bash
openssl req -x509 -newkey rsa:2048 -nodes -keyout k.pem -out c.pem -days 365 \
  -subj "/CN=app.local" -addext "subjectAltName=DNS:app.local,IP:10.0.0.5"
```

**Prevent →** Always populate SANs in internal CA templates.

</details>

### Q71. `SSL_ERROR_RX_RECORD_TOO_LONG` in Firefox for your HTTPS site. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Port 443 is serving plain HTTP: missing `ssl` on the nginx `listen 443` line, or a load balancer forwards HTTPS to an HTTP port without terminating TLS.

**Diagnose →**
```bash
curl -v http://host:443/
grep -n 'listen' /etc/nginx/sites-enabled/*
```

**Fix →** `listen 443 ssl;` with certificate directives, or terminate TLS at the LB.

**Prevent →** Config tests in CI (`nginx -t`) plus an HTTPS smoke test after deploy.

</details>

### Q72. Clients behind a corporate TLS-inspection proxy fail only for one SaaS endpoint, with handshake failures. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SaaS client pins its certificate or uses mTLS, so a re-signed certificate is rejected; or the proxy doesn't support TLS 1.3 / the endpoint's ciphers.

**Diagnose →**
```bash
openssl s_client -connect saas.example.com:443 -servername saas.example.com -showcerts </dev/null | grep 'i:'
# issuer = corporate CA => inspected
```

**Fix →** Add the domain to the proxy's TLS-inspection bypass list.

**Prevent →** Maintain a bypass list for pinned/mTLS destinations and review it during SaaS onboarding.

</details>

### Q73. After a key rotation, nginx refuses to start: `SSL_CTX_use_PrivateKey ... key values mismatch`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The new certificate was paired with the old key, or the files were swapped.

**Diagnose →**
```bash
openssl x509 -noout -pubkey -in cert.pem | sha256sum
openssl pkey -pubout -in key.pem | sha256sum      # hashes must match
```

**Fix →** Pair the matching key and cert, `nginx -t`, then reload.

**Prevent →** Validate key/cert pairs in the deployment pipeline before pushing.

</details>

### Q74. gRPC client returns `transport: authentication handshake failed: x509: certificate is valid for ingress.local, not api.example.com`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ingress controller returns its default fake certificate because no Ingress/TLS secret matches the SNI, or the TLS secret is in the wrong namespace.

**Diagnose →**
```bash
kubectl get ingress -A -o wide | grep api.example.com
kubectl describe ingress api -n app     # TLS section, secret name
kubectl get secret api-tls -n app
```

**Fix →** Put the TLS Secret in the Ingress's namespace and list the host under `spec.tls.hosts`. For gRPC on ingress-nginx also set `nginx.ingress.kubernetes.io/backend-protocol: "GRPC"`.

**Prevent →** Alert when the controller serves its default certificate (synthetic check on the issuer).

</details>

### Q75. A client fails with `certificate is not yet valid` right after a new certificate was issued. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Client clock is behind (VMs restored from snapshot, IoT devices, containers on a host without NTP), or the CA backdates less than the skew.

**Diagnose →**
```bash
date -u; chronyc tracking
openssl x509 -in cert.pem -noout -startdate
```

**Fix →** Fix time sync (`systemctl enable --now chronyd`, `chronyc makestep`).

**Prevent →** Monitor clock offset on every host; alert above 1 s.

</details>

---

## MTU, routing, packet loss and latency

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. SSH connects, but the session freezes as soon as you run `ls` in a big directory or `cat` a large file. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** MTU black hole: small packets get through, full-size packets exceed a smaller MTU on the path (VPN, GRE, VXLAN) and are dropped, while ICMP "fragmentation needed" is filtered so PMTUD never works.

**Diagnose →**
```bash
ping -M do -s 1472 target     # 1472 + 28 = 1500
ping -M do -s 1372 target     # step down until it works
tracepath target              # reports pmtu per hop
```

**Fix →** Lower the interface MTU (`ip link set dev eth0 mtu 1400`) or clamp MSS on the tunnel router:
```bash
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```
Allow ICMP type 3 code 4 (and ICMPv6 type 2) through firewalls.

**Prevent →** Set MTU and MSS clamping as part of every tunnel design.

</details>

### Q77. HTTPS to a site over the VPN hangs after `Client hello`, while HTTP works. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The server's certificate chain response is large and fills full-size packets that get dropped by an MTU black hole; small HTTP responses fit.

**Diagnose →**
```bash
curl -v https://host/        # stalls after "TLSv1.3 (OUT), TLS handshake, Client hello"
sudo tcpdump -ni tun0 host host   # repeated retransmits of large segments
```

**Fix →** MSS clamping on the VPN gateway or lower tunnel MTU (WireGuard: `MTU = 1380` in `[Interface]`).

**Prevent →** Test large transfers (not just ping) when validating a tunnel.

</details>

### Q78. `ping: local error: message too long, mtu=1500` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You set `-M do` (don't fragment) with a size larger than the local interface MTU.

**Diagnose →**
```bash
ip link show eth0 | grep mtu
```

**Fix →** Use a size of MTU minus 28 for IPv4 (20 IP + 8 ICMP): `ping -M do -s 1472`. This is just a test sizing error, not a network problem.

**Prevent →** Remember the header math when probing PMTU.

</details>

### Q79. Pods on different nodes can ping each other, but HTTP responses larger than ~1.4 KB never arrive. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Overlay MTU mismatch: VXLAN adds 50 bytes, Geneve more, but the pod interface MTU is still 1500 on a 1500 underlay. Or the underlay is 1500 while nodes were configured with 9001 jumbo frames on only some hosts.

**Diagnose →**
```bash
kubectl exec pod-a -- ip link show eth0
ip link show flannel.1; ip link show vxlan.calico
kubectl exec pod-a -- ping -M do -s 1422 <pod-b-ip>
```

**Fix →** Set the CNI MTU to underlay MTU minus encapsulation overhead (Calico `veth_mtu`, Cilium `--mtu`), roll the pods so new veths pick it up.

**Prevent →** Let the CNI auto-detect MTU where supported, and test a large payload in cluster acceptance tests.

</details>

### Q80. `mtr` shows 30% loss at hop 6, but 0% at the final destination. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The router at hop 6 rate-limits ICMP replies to itself (control-plane policing). Forwarded traffic is fine.

**Diagnose →**
```bash
mtr -rwzc 200 target
mtr -T -P 443 -rwc 200 target     # TCP mode
```

**Fix →** Nothing. Loss only matters if it continues to every later hop including the destination.

**Prevent →** Read mtr from the destination backward; first loss that persists to the end is the real one.

</details>

### Q81. `mtr` shows loss starting at hop 4 and continuing to the destination at 8–10%. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Real loss on the link between hop 3 and hop 4 or in hop 4: congestion, a bad optic/cable, or a policer.

**Diagnose →**
```bash
mtr -rwzc 500 -i 0.2 target        # include AS numbers (-z)
mtr -rwzc 500 target_from_other_direction   # loss is often one-way; run it back from the target
```

**Fix →** If the hop is yours, check interface errors (`ip -s link`, `ethtool -S`). If it's an ISP or transit, open a ticket with both mtr reports and timestamps.

**Prevent →** Continuous loss monitoring (smokeping, ThousandEyes, CloudWatch Network Monitor) to have baselines.

</details>

### Q82. Latency between two app servers in the same region jumped from 0.5 ms to 40 ms. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Traffic now flows through a different path: a new route through a central inspection VPC or a VPN, one host moved to another AZ/region, or a DNS change points to a far endpoint.

**Diagnose →**
```bash
ip route get <peer-ip>
traceroute -n <peer-ip>
mtr -rwc 50 <peer-ip>
```

**Fix →** Restore the direct route (TGW/peering route table), co-locate or fix DNS.

**Prevent →** Alert on inter-service RTT percentiles and review route table changes.

</details>

### Q83. Replies to traffic coming in on `eth1` leave via `eth0`, and the connection never completes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Asymmetric routing on a multi-homed host: the default route is on `eth0`, so replies to `eth1`'s clients go out the wrong interface with `eth1`'s source IP. Upstream stateful firewalls or `rp_filter` drop them.

**Diagnose →**
```bash
sudo tcpdump -ni any host <client> -e     # see which interface each direction uses
ip route get <client> from <eth1-ip>
```

**Fix →** Policy routing:
```bash
ip route add default via 10.1.0.1 dev eth1 table 100
ip rule add from 10.1.0.20/32 table 100
```

**Prevent →** For multi-NIC instances, configure source-based routing in network config (netplan `routing-policy`).

</details>

### Q84. Through a pair of firewalls, connections randomly get reset or hang after a failover event. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Asymmetric paths: SYN goes through firewall A, SYN-ACK returns through firewall B, which has no state and drops it. Or state isn't synced between the HA pair.

**Diagnose →**
```bash
# Capture on both firewalls (or their logs) for the same 5-tuple
tcpdump -ni any host <client> and port 443
# Check routing on both sides of the pair for ECMP or different preferred paths
```

**Fix →** Make return routing symmetric (same next hop both ways), enable state synchronization, or use GWLB which keeps flow symmetry.

**Prevent →** Design inspection paths with symmetric routing and test failovers under traffic.

</details>

### Q85. File copies between two data centers get 50 Mbit/s on a 1 Gbit/s link with no loss. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Bandwidth-delay product: with 60 ms RTT, a single TCP flow needs ~7.5 MB of window for 1 Gbit/s. Small socket buffers or disabled window scaling cap throughput.

**Diagnose →**
```bash
ss -tin dst <peer>        # check rcv_space, cwnd, rtt
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem net.core.rmem_max
iperf3 -c <peer> -P 1; iperf3 -c <peer> -P 8
```

**Fix →** Raise buffer maximums (`net.core.rmem_max=67108864`, `net.ipv4.tcp_rmem="4096 87380 67108864"`), try BBR (`net.ipv4.tcp_congestion_control=bbr`), or use parallel streams.

**Prevent →** Tune long-fat-network hosts deliberately and benchmark with iperf3 before migrations.

</details>

### Q86. `traceroute` shows only `* * *` after the first hop. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** UDP probes (Linux default, ports 33434+) or ICMP time-exceeded replies are blocked; cloud networks often don't return them.

**Diagnose →**
```bash
traceroute -I target           # ICMP echo
sudo traceroute -T -p 443 target   # TCP SYN on a port that's allowed
```

**Fix →** Use TCP traceroute on the application port. Stars alone don't mean traffic is dropped.

**Prevent →** Know which probe types each environment allows.

</details>

### Q87. A host has two default routes and outbound traffic is randomly slow or fails. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** DHCP on two interfaces added two defaults; the kernel uses the lower metric, or both with equal metrics, sending traffic out a path with no NAT or a wrong source IP.

**Diagnose →**
```bash
ip route show default
ip route get 1.1.1.1
```

**Fix →** Remove one default or set metrics (netplan `dhcp4-overrides: { route-metric: 200 }`), plus policy routing if both are needed.

**Prevent →** Only one interface should provide the default unless you design for multi-path.

</details>

### Q88. `ip route add 10.20.0.0/16 via 10.0.5.1` fails with `Error: Nexthop has invalid gateway.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The gateway isn't on any directly connected subnet of the host.

**Diagnose →**
```bash
ip -br addr
ip route get 10.0.5.1
```

**Fix →** Use a gateway that is in a local subnet, or add `dev eth1 onlink` if you know the next hop is reachable at layer 2.

**Prevent →** Model next hops as directly connected neighbours when planning static routes.

</details>

### Q89. Interface counters show rising `RX errors` and `dropped`, and the app sees retransmits. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Errors: bad cable, optic, duplex mismatch. Drops: RX ring buffer overflow or softirq can't keep up (CPU, single queue).

**Diagnose →**
```bash
ip -s link show eth0
ethtool eth0 | grep -E 'Speed|Duplex'
ethtool -S eth0 | grep -i -E 'err|drop|miss|fifo'
ethtool -g eth0
cat /proc/net/softnet_stat     # 2nd column = drops
```

**Fix →** Replace hardware or fix duplex; raise ring size (`ethtool -G eth0 rx 4096`), enable RSS/RPS across CPUs, raise `net.core.netdev_max_backlog`.

**Prevent →** Export NIC counters to monitoring (node_exporter does this).

</details>

### Q90. TCP throughput drops sharply under load, and `nstat` shows `TcpRetransSegs` climbing. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Packet loss somewhere on the path (congestion, policer, bad link) or receiver-side drops.

**Diagnose →**
```bash
nstat -az TcpRetransSegs TcpExtTCPLostRetransmit TcpExtTCPTimeouts
ss -ti dst <peer> | grep -o 'retrans:[0-9/]*'
sudo tcpdump -ni eth0 host <peer> -w retrans.pcap   # in Wireshark: tcp.analysis.retransmission
```

**Fix →** Locate the loss with mtr and interface counters; fix the bottleneck or shape traffic below a policer rate.

**Prevent →** Alert on retransmit rate as a percentage of segments sent.

</details>

### Q91. On AWS, an instance hits throughput limits and `ethtool -S eth0` shows `bw_out_allowance_exceeded` increasing. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The instance type's network bandwidth allowance is exceeded (burst credits consumed); packets are queued or dropped by ENA. Related counters: `pps_allowance_exceeded`, `conntrack_allowance_exceeded`, `linklocal_allowance_exceeded`.

**Diagnose →**
```bash
ethtool -S eth0 | grep allowance
```

**Fix →** Move to a larger or network-optimized instance type (`n` suffix), spread load across instances, or reduce chatty small-packet traffic.

**Prevent →** Export ENA allowance counters to CloudWatch (CloudWatch agent supports them) and alert on any increase.

</details>

### Q92. VPN users report the internet is "slow" since full-tunnel VPN was enabled. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** All traffic hairpins through the VPN concentrator and corporate egress, adding RTT and sharing a congested link.

**Diagnose →**
```bash
ip route        # default via tun0?
traceroute -n 1.1.1.1
curl -o /dev/null -w '%{time_total}\n' https://example.com
```

**Fix →** Split tunnelling for SaaS/internet traffic if policy allows, or add VPN concentrator capacity and regional egress.

**Prevent →** Size concentrators and egress for peak; measure user experience after VPN policy changes.

</details>

### Q93. After adding a static route on one router, a subnet becomes unreachable and `traceroute` shows the same two hops repeating. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Routing loop: router A sends the prefix to B, and B's route for it points back to A. Packets bounce until TTL hits zero.

**Diagnose →**
```bash
traceroute -n 10.30.1.5         # 10.0.0.1, 10.0.0.2, 10.0.0.1, 10.0.0.2 ...
ip route get 10.30.1.5          # on each router
```

**Fix →** Correct the next hop on one side; remove the conflicting static route.

**Prevent →** Peer review route changes and prefer dynamic routing with proper filters over scattered static routes.

</details>

### Q94. A Linux box acting as router doesn't forward packets between its interfaces. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** IP forwarding disabled, `FORWARD` chain policy `DROP`, or missing NAT for return traffic. On AWS also source/destination check is enabled.

**Diagnose →**
```bash
sysctl net.ipv4.ip_forward
sudo iptables -L FORWARD -v -n
aws ec2 describe-instance-attribute --instance-id i-123 --attribute sourceDestCheck
```

**Fix →** `sysctl -w net.ipv4.ip_forward=1` (persist in `/etc/sysctl.d/`), allow forwarding, add `MASQUERADE` if needed, disable source/dest check.

**Prevent →** Configure routers via code with all four pieces together.

</details>

### Q95. Latency to a remote service has high jitter: p50 is 20 ms but p99 is 600 ms. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Bufferbloat or microbursts on a congested link, retransmission timeouts (200 ms minimum RTO), CPU steal or GC pauses on endpoints, or Nagle + delayed ACK interaction (40 ms stalls).

**Diagnose →**
```bash
mtr -rwc 500 -i 0.1 target
ss -ti dst target | grep -E 'rto|retrans|rtt'
sar -n DEV 1; vmstat 1    # steal (st) column
```

**Fix →** Remove the bottleneck or apply fq_codel/cake queueing, set `TCP_NODELAY` for request/response protocols, fix endpoint pauses.

**Prevent →** Track latency histograms, not averages, and correlate with retransmits.

</details>

### Q96. Large UDP packets (e.g. syslog or DNS over 1500 bytes) are lost while small ones arrive. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Large UDP datagrams are IP-fragmented, and a firewall, NAT or cloud network drops fragments (non-first fragments have no port info).

**Diagnose →**
```bash
sudo tcpdump -ni eth0 'ip[6:2] & 0x3fff != 0'     # fragments
nstat -az IpReasmFails IpFragFails
```

**Fix →** Keep UDP payloads under path MTU (EDNS buffer 1232 for DNS), switch to TCP (syslog over TCP/RELP), or allow fragments.

**Prevent →** Design UDP protocols to avoid fragmentation.

</details>

### Q97. Throughput between two EC2 instances is capped at 5 Gbit/s despite both being 25 Gbit/s types. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Single-flow limit: AWS caps one flow at 5 Gbit/s outside a cluster placement group (10 Gbit/s inside). Cross-region or via IGW/NAT also has lower limits.

**Diagnose →**
```bash
iperf3 -c <peer> -P 1
iperf3 -c <peer> -P 8 -t 30
```

**Fix →** Use multiple flows, or put instances in a cluster placement group for the 10 Gbit/s single-flow limit.

**Prevent →** Know per-flow limits when designing data replication.

</details>

### Q98. After enabling jumbo frames (MTU 9001) on EC2, traffic to on-prem over VPN breaks for large transfers. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Jumbo frames work inside a VPC but VPN, IGW, peering across regions and Transit Gateway paths have smaller MTUs (VPN ~1399 effective), and PMTUD depends on ICMP being allowed.

**Diagnose →**
```bash
tracepath <onprem-ip>
ping -M do -s 1372 <onprem-ip>
```

**Fix →** Allow ICMP "fragmentation needed" in security groups, or set MTU 1500 on instances that talk across these paths.

**Prevent →** Only use jumbo frames on intra-VPC paths that you've tested end to end.

</details>

### Q99. `ping` to a host on the same subnet fails, but `ip neigh` shows the entry as `FAILED`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ARP gets no answer: host down, wrong subnet mask, different VLAN, or the host's firewall drops ARP (rare).

**Diagnose →**
```bash
ip neigh show
sudo arping -I eth0 -c 3 10.0.1.40
sudo tcpdump -ni eth0 arp
```

**Fix →** Bring the host up, fix the mask/VLAN, or fix the switch port.

**Prevent →** IPAM-managed addressing and consistent subnet masks.

</details>

### Q100. After moving a VIP between two servers, clients keep sending traffic to the old server for minutes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Neighbours and the gateway still have the old MAC in their ARP caches; the new owner didn't send gratuitous ARP, or the network ignores it.

**Diagnose →**
```bash
ip neigh show 10.0.1.100        # on a client/gateway
sudo tcpdump -ni eth0 arp and host 10.0.1.100
```

**Fix →** Send gratuitous ARP on takeover (`arping -U -I eth0 -c 3 10.0.1.100`); keepalived does this by default (`garp_master_delay`).

**Prevent →** Use keepalived/VRRP properly; in cloud, use API-based IP reassignment instead of ARP tricks.

</details>

---


## Sockets, ports, TIME_WAIT and conntrack

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. `dmesg` is full of `nf_conntrack: nf_conntrack: table full, dropping packet` and new connections randomly time out. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The conntrack table hit `nf_conntrack_max`. Every tracked flow (including short DNS and health-check flows) holds an entry until its timeout expires, so a traffic spike, a scan, or long `TIME_WAIT`/UDP timeouts fill it. When full, the kernel drops new flows silently, which looks like random timeouts.

**Diagnose →**
```bash
sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
sudo conntrack -S | head                       # look at drop / early_drop counters
sudo conntrack -L -o extended | awk '{print $3}' | sort | uniq -c | sort -rn   # by protocol
sudo conntrack -L | awk '{print $5}' | sort | uniq -c | sort -rn | head        # top sources
```

**Fix →** Raise the limit and the hash size together, and shorten timeouts for flows you don't need held long:
```bash
sudo sysctl -w net.netfilter.nf_conntrack_max=524288
echo 131072 | sudo tee /sys/module/nf_conntrack/parameters/hashsize
sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
```
For high-volume stateless traffic (e.g. a DNS server), exempt it with `iptables -t raw -A PREROUTING -p udp --dport 53 -j NOTRACK`.

**Prevent →** Persist values in `/etc/sysctl.d/`, export `nf_conntrack_count / nf_conntrack_max` to monitoring and alert at 75%.

</details>

### Q102. `ss -s` on a reverse proxy shows 40,000 sockets in `TIME_WAIT`. Is that a problem? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `TIME_WAIT` is normal: the side that closes a TCP connection first holds it for 2×MSL (60 s on Linux) so late packets don't corrupt a new connection. A proxy that opens a new upstream connection per request and closes it first will accumulate many. Each costs a few hundred bytes, so 40k is harmless memory-wise. It only becomes a problem when they exhaust ephemeral ports towards the *same* destination IP:port.

**Diagnose →**
```bash
ss -s
ss -tan state time-wait | awk '{print $4}' | sort | uniq -c | sort -rn | head   # per destination
sysctl net.ipv4.ip_local_port_range
```
If one upstream has ~28k entries (the default range size), you are close to `Cannot assign requested address`.

**Fix →** Reuse upstream connections (nginx `keepalive 64;` in the `upstream` block with `proxy_http_version 1.1;` and `proxy_set_header Connection "";`). If still needed, `net.ipv4.tcp_tw_reuse=1` lets outgoing connections reuse `TIME_WAIT` ports safely.

**Prevent →** Don't "fix" `TIME_WAIT` counts by themselves; monitor per-destination port usage instead.

</details>

### Q103. An app server's `CLOSE_WAIT` count grows steadily until the service logs `Too many open files`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `CLOSE_WAIT` means the peer closed the connection and the kernel is waiting for *your application* to call `close()`. It is always an application bug: a leaked HTTP client response body, a DB connection not returned to the pool, or a thread blocked forever before cleanup. The kernel never times these out.

**Diagnose →**
```bash
ss -tanp state close-wait | awk '{print $4, $NF}' | sort | uniq -c | sort -rn | head
ls /proc/$(pgrep -f myapp)/fd | wc -l
# Java: jstack <pid> to see threads stuck around the socket; Go: check for missing resp.Body.Close()
```
The remote address tells you which dependency's connections are leaking.

**Fix →** Fix the code path (close response bodies, use `try-with-resources`/`defer`, set client timeouts so blocked calls return). Restarting the process clears them only temporarily.

**Prevent →** Alert on `CLOSE_WAIT` count per process and on open FD count versus the limit.

</details>

### Q104. Logs show `accept4(): Too many open files` and the service stops accepting connections. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Every socket is a file descriptor. The process hit its `RLIMIT_NOFILE` (often 1024 soft by default), either through normal load or a socket/file leak.

**Diagnose →**
```bash
pid=$(pgrep -f myapp | head -1)
grep 'open files' /proc/$pid/limits
ls /proc/$pid/fd | wc -l
sudo lsof -p $pid | awk '{print $5}' | sort | uniq -c   # IPv4/IPv6 vs REG files
cat /proc/sys/fs/file-nr                                 # system-wide usage
```

**Fix →** Raise the limit where the process is started (systemd `LimitNOFILE=65536`, Docker `--ulimit nofile=65536:65536`, Kubernetes inherits from the runtime). If the count keeps climbing without load, look for a leak (see `CLOSE_WAIT`).

**Prevent →** Set explicit limits in unit files, and monitor FDs as a percentage of the limit.

</details>

### Q105. `dmesg` shows `TCP: request_sock_TCP: Possible SYN flooding on port 443. Sending cookies.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SYN (half-open) queue for that listener overflowed and the kernel fell back to SYN cookies. Either it's a real SYN flood, or a legitimate burst (deploy, reconnect storm) with a small backlog. SYN cookies keep you up but drop TCP options such as window scaling in some cases.

**Diagnose →**
```bash
nstat -az | grep -E 'TcpExtSyncookiesSent|TcpExtListenOverflows|TcpExtListenDrops'
ss -ltn 'sport = :443'
sysctl net.ipv4.tcp_max_syn_backlog net.core.somaxconn net.ipv4.tcp_syncookies
sudo tcpdump -ni eth0 'tcp[tcpflags] == tcp-syn and dst port 443' -c 200 | awk '{print $3}' | sort | uniq -c | sort -rn | head
```
Many distinct spoofed sources that never complete the handshake means a flood.

**Fix →** For bursts: raise `tcp_max_syn_backlog`, `somaxconn` and the app's `listen()` backlog (nginx `listen 443 ssl backlog=4096;`). For floods: put DDoS protection (AWS Shield, Cloudflare) or an L4 LB in front; keep `tcp_syncookies=1`.

**Prevent →** Alert on `SyncookiesSent` and `ListenDrops` rates.

</details>

### Q106. `ss -ltn` shows the listener for port 8080 with `Recv-Q 129` and `Send-Q 128`, and clients hang on connect. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** For a listening socket, `Recv-Q` is the current accept queue length and `Send-Q` is its maximum. Recv-Q above Send-Q means the accept queue is full: the handshake completed in the kernel, but the app isn't calling `accept()` fast enough (event loop blocked, all worker threads busy, GC pause). Further final ACKs are dropped and clients retransmit.

**Diagnose →**
```bash
watch -n1 "ss -ltn 'sport = :8080'"
nstat -az TcpExtListenOverflows TcpExtListenDrops
top -H -p $(pgrep -f myapp)         # are all workers pegged?
```

**Fix →** Fix the slow consumer first (more workers, non-blocking I/O, offload slow work). Then raise the backlog: the effective value is `min(listen() backlog, net.core.somaxconn)`, so increase both.
```bash
sudo sysctl -w net.core.somaxconn=4096
# and in the app, e.g. gunicorn --backlog 4096
```

**Prevent →** Export `ListenOverflows` and alert on any increase; load test with realistic concurrency.

</details>

### Q107. A blog post told you to set `net.ipv4.tcp_tw_recycle=1`, but `sysctl` returns `cannot stat /proc/sys/net/ipv4/tcp_tw_recycle`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `tcp_tw_recycle` was removed in Linux 4.12. It broke clients behind NAT (it rejected SYNs with lower TCP timestamps from the same IP), so the advice is outdated.

**Diagnose →**
```bash
uname -r
sysctl -a 2>/dev/null | grep -E 'tcp_tw_reuse|ip_local_port_range|tcp_fin_timeout'
```

**Fix →** Don't try to re-add it. If the real goal is avoiding port exhaustion on outgoing connections, use `net.ipv4.tcp_tw_reuse=1`, widen `ip_local_port_range`, and reuse connections with keepalive/pooling.

**Prevent →** Review sysctl tuning in config management against the running kernel version; remove stale keys so `sysctl --system` doesn't fail on boot.

</details>

### Q108. `ss -tan` shows thousands of sockets in `FIN-WAIT-2` on a server. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Your side closed and the peer ACKed the FIN, but the peer never sent its own FIN. The remote application is not closing its end (its own `CLOSE_WAIT` leak), or a middlebox dropped the peer's FIN. Orphaned FIN-WAIT-2 sockets time out after `tcp_fin_timeout` (60 s default); ones still owned by a process can live longer.

**Diagnose →**
```bash
ss -tano state fin-wait-2 | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head
sysctl net.ipv4.tcp_fin_timeout
```
Then check the peer host for matching `CLOSE_WAIT` sockets.

**Fix →** Fix the peer's connection handling. As a mitigation, lower `net.ipv4.tcp_fin_timeout` to 15–30 s and make sure your app closes sockets fully (not just `shutdown(SHUT_WR)` and then forgets them).

**Prevent →** Track TCP state counts per host as a dashboard; asymmetry between peers points straight at the leaking side.

</details>

### Q109. Port 9090 is in use, but `ss -ltnp | grep 9090` shows no process name. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You ran `ss` without root, so it can't read other users' process info; or the socket belongs to a different network namespace (a container), where it's not visible at all; or it's a kernel-owned socket (e.g. NFS, WireGuard).

**Diagnose →**
```bash
sudo ss -ltnp 'sport = :9090'
sudo lsof -nP -iTCP:9090 -sTCP:LISTEN
sudo fuser -v 9090/tcp
docker ps --format '{{.Names}} {{.Ports}}' | grep 9090
# inside a specific namespace:
sudo nsenter -t <container-pid> -n ss -ltnp
```

**Fix →** Stop or reconfigure the owner. If it is `docker-proxy`, a container publishes the port; change its `-p` mapping.

**Prevent →** Document port allocations per host and use `sudo` for socket inspection by habit.

</details>

### Q110. A Node.js service fails to start with `Error: listen EACCES: permission denied 0.0.0.0:80`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Ports below 1024 are privileged; a non-root process needs `CAP_NET_BIND_SERVICE`, or the system must lower `net.ipv4.ip_unprivileged_port_start`.

**Diagnose →**
```bash
id
sysctl net.ipv4.ip_unprivileged_port_start
getcap $(which node)
```

**Fix →** Pick one:
```bash
# systemd unit
AmbientCapabilities=CAP_NET_BIND_SERVICE
# or listen on 8080 and put nginx / an LB on 80
# or (containers) run on 8080 and map -p 80:8080
```
Avoid `setcap` on the shared `node` binary; it grants the capability to every Node script.

**Prevent →** Run apps on high ports behind a reverse proxy or LB; never run them as root just to bind 80.

</details>

### Q111. A Java service shows `LISTEN *:8080` (or `:::8080`), yet IPv4 clients get `Connection refused` on one host only. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The app opened an IPv6 socket on `::`. Normally that also accepts IPv4 via mapped addresses, but on that host `net.ipv6.bindv6only=1` (or the app set `IPV6_V6ONLY`), so it is IPv6-only. Other hosts have the default `0`.

**Diagnose →**
```bash
ss -ltn 'sport = :8080'           # [::]:8080 only, no 0.0.0.0:8080
sysctl net.ipv6.bindv6only
curl -4 -v http://127.0.0.1:8080/ ; curl -6 -v http://[::1]:8080/
```

**Fix →** Set `net.ipv6.bindv6only=0`, or bind the app explicitly to `0.0.0.0` (Java: `-Djava.net.preferIPv4Stack=true`, Spring: `server.address=0.0.0.0`), or open both an IPv4 and IPv6 listener.

**Prevent →** Keep sysctl baselines identical across a fleet and check `ss -ltn` output in smoke tests.

</details>

### Q112. After a DNS server behind a UDP load-balancing iptables rule was replaced, some clients keep failing lookups for minutes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** UDP has no close, so conntrack keeps the existing DNAT mapping (client IP:port → old backend) alive as long as packets keep arriving within `nf_conntrack_udp_timeout_stream` (120 s by default, refreshed on each packet). Clients reusing the same source port are pinned to the dead backend. Kubernetes hit the same issue with CoreDNS pod replacement.

**Diagnose →**
```bash
sudo conntrack -L -p udp --dport 53 | grep <old-backend-ip>
sysctl net.netfilter.nf_conntrack_udp_timeout net.netfilter.nf_conntrack_udp_timeout_stream
```

**Fix →** Flush stale entries for the old backend:
```bash
sudo conntrack -D -p udp --orig-dst <vip> --dport 53
sudo conntrack -D -p udp --reply-src <old-backend-ip>
```

**Prevent →** Make backend changes clear conntrack automatically (kube-proxy does this for UDP endpoints), shorten UDP timeouts on DNS-heavy nodes, or use NodeLocal DNSCache so clients talk to a stable local IP.

</details>

### Q113. Under load, `conntrack -S` shows `insert_failed` climbing, and outbound connections from containers fail sporadically. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A race in SNAT/MASQUERADE: two connections from different containers are assigned the same source port at the same moment; the second conntrack insert fails and the packet is dropped. The client sees a timeout (often 1 s or 5 s extra latency on retry). Very common for high-rate outbound calls and for UDP DNS from pods.

**Diagnose →**
```bash
sudo conntrack -S | grep -o 'insert_failed=[0-9]*'
sudo iptables -t nat -S POSTROUTING | grep MASQUERADE
```

**Fix →** Use fully random port allocation on the masquerade rule (`--random-fully`, which recent Docker, kube-proxy and most CNIs apply), reduce SNAT by using VPC-native pod IPs, and reuse connections with keepalive/pooling. For DNS, use NodeLocal DNSCache or `options single-request-reopen`.

**Prevent →** Graph `insert_failed` per node and include it in node health dashboards.

</details>

### Q114. `ss -tn` shows one client connection with a large and growing `Send-Q`, and the app's write calls block. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** On an established socket, `Send-Q` is data not yet ACKed by the peer. A growing value means the receiver is slow (advertising a zero or tiny window because its app isn't reading), or the path is losing packets.

**Diagnose →**
```bash
ss -tni dst <client-ip>          # check snd_wnd / rcv_space / retrans / rto
sudo tcpdump -ni eth0 host <client-ip> -c 50 | grep -E 'win 0|win [0-9]{1,3},'
```
`win 0` from the client = zero window (receiver problem). Retransmits with a normal window = network loss.

**Fix →** Fix the slow consumer on the client side, or apply per-connection write timeouts on the server so one slow client can't tie up a worker (nginx `send_timeout`, app-level write deadlines).

**Prevent →** Use non-blocking writes with deadlines, and protect against slow readers with timeouts at the proxy layer.

</details>

### Q115. Long-lived connections through a Linux NAT box get reset, and conntrack logs show packets classified as `INVALID`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Conntrack tracks TCP windows strictly. Packets that fall outside the tracked window (after heavy reordering, a conntrack table restart, or a flow that started before the box took over) are marked INVALID, and a typical ruleset drops them or doesn't NAT them, so they leak with the private source and the peer answers with RST.

**Diagnose →**
```bash
sudo conntrack -S | grep -o 'invalid=[0-9]*'
sudo iptables -L FORWARD -v -n | grep INVALID
sudo sysctl -w net.netfilter.nf_conntrack_log_invalid=6   # log TCP invalids, then check dmesg
```

**Fix →** Drop INVALID explicitly so they never leave un-NATed (`iptables -A FORWARD -m conntrack --ctstate INVALID -j DROP`), and if reordering is expected, set `net.netfilter.nf_conntrack_tcp_be_liberal=1` so only out-of-window RSTs count as invalid.

**Prevent →** Sync conntrack state between HA NAT nodes (`conntrackd`) and avoid ECMP paths that reorder single flows.

</details>

### Q116. keepalived moves a VIP to the backup node, but HAProxy on that node fails to start with `cannot bind socket [10.0.1.100:443]`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** HAProxy starts before the VIP is assigned to the node (it's on the master), and Linux refuses to bind to an address that isn't local (`EADDRNOTAVAIL`).

**Diagnose →**
```bash
ip addr show | grep 10.0.1.100
sysctl net.ipv4.ip_nonlocal_bind
journalctl -u haproxy | grep -i bind
```

**Fix →** Allow binding to non-local addresses so HAProxy runs on both nodes all the time:
```bash
echo 'net.ipv4.ip_nonlocal_bind = 1' | sudo tee /etc/sysctl.d/90-vip.conf
sudo sysctl --system && sudo systemctl restart haproxy
```
Alternatively bind to `*:443` and filter by destination in ACLs.

**Prevent →** Test failover regularly; with `ip_nonlocal_bind` both proxies are warm and failover is only an ARP update.

</details>

### Q117. During a rolling binary upgrade with `SO_REUSEPORT`, a fraction of requests still hit the old version and some get reset. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With `SO_REUSEPORT` the kernel hashes new connections across *all* listening sockets in the group. While the old process is alive, it keeps receiving new connections. When it exits, connections already sitting in its accept queue (handshake complete, not yet accepted) are reset.

**Diagnose →**
```bash
sudo ss -ltnp 'sport = :8080'      # two PIDs listening on the same port
nstat -az TcpExtTCPAbortOnClose
```

**Fix →** Stop the old process's listener before it exits: close the listening socket, keep serving in-flight requests, then exit (graceful drain). Tools like nginx (`nginx -s reload`), HAProxy (`-x` socket transfer / seamless reload) and Envoy hot restart pass the listening FD instead of racing.

**Prevent →** Use FD passing / socket activation (systemd) for zero-downtime restarts, and test upgrades under load.

</details>

### Q118. On a client, `ss -tan state syn-sent` shows hundreds of connections to the same backend stuck there. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The client sent SYNs and got nothing back: a firewall/security group drops them, the server is down, there's no return route, or the server's SYN queue is overflowing.

**Diagnose →**
```bash
ss -tan state syn-sent | awk '{print $4}' | sort | uniq -c
sudo tcpdump -ni eth0 'host <backend> and tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'
nc -vz -w 3 <backend> <port>
```
Run the same tcpdump on the backend: SYNs not arriving = path/firewall; arriving but no SYN-ACK = server-side issue.

**Fix →** Open the port in the SG/firewall, fix routing, or fix the server backlog. Set sane connect timeouts in clients so they fail fast instead of piling up.

**Prevent →** Synthetic TCP checks to critical dependencies and connect-timeout metrics in the client.

</details>

### Q119. A service on port 40001 fails to start after boot about once a week with `Address already in use`, though nothing listens there when you check. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Port 40001 lies inside the ephemeral range (`ip_local_port_range`, default 32768–60999). Occasionally an outbound connection opened before the service starts grabs it as its source port.

**Diagnose →**
```bash
sysctl net.ipv4.ip_local_port_range
ss -tanp | grep ':40001 '
```

**Fix →** Reserve the port so the kernel never uses it as an ephemeral source port:
```bash
echo 'net.ipv4.ip_local_reserved_ports = 40001' | sudo tee /etc/sysctl.d/91-reserved.conf
sudo sysctl --system
```
Or move the service out of the ephemeral range.

**Prevent →** Keep listening ports below the ephemeral range; note that Kubernetes NodePorts (30000–32767) overlap with some ranges, so reserve them on nodes.

</details>

### Q120. The app logs `write: broken pipe` (or `EPIPE`) on some responses. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The app wrote to a socket the peer had already closed: the client gave up (timeout, user navigated away), or a proxy/LB in between closed the connection because its timeout was shorter than the app's processing time.

**Diagnose →**
```bash
# correlate with proxy logs: nginx 499 or ALB 460 at the same timestamps
grep ' 499 ' /var/log/nginx/access.log | tail
```
Measure how long the failing requests took versus each layer's timeout.

**Fix →** Occasional `EPIPE` from impatient clients is normal; log it at a lower level. If it correlates with slow endpoints, speed them up or align timeouts so each inner layer times out before the outer one.

**Prevent →** Define a timeout budget from client → LB → proxy → app → DB, decreasing at each hop.

</details>

### Q121. Clients that vanished (laptops closed, mobile networks switched) leave connections `ESTAB` on the server for days. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** TCP has no traffic on an idle connection, so if the client disappears without a FIN/RST, the server never notices. These half-open connections hold FDs and memory. TCP keepalive is off by default per socket, and even when on, the default first probe is after 7200 s.

**Diagnose →**
```bash
ss -tno state established '( sport = :5000 )' | grep -c keepalive
sysctl net.ipv4.tcp_keepalive_time net.ipv4.tcp_keepalive_intvl net.ipv4.tcp_keepalive_probes
```
`ss -o` shows `timer:(keepalive,...)` only for sockets with `SO_KEEPALIVE` set.

**Fix →** Enable `SO_KEEPALIVE` in the app and tune it, e.g. 60 s idle, 10 s interval, 6 probes:
```bash
sudo sysctl -w net.ipv4.tcp_keepalive_time=60 net.ipv4.tcp_keepalive_intvl=10 net.ipv4.tcp_keepalive_probes=6
```
Better still, add application-level heartbeats/idle timeouts (WebSocket pings, gRPC keepalive).

**Prevent →** Set idle timeouts at every layer and keepalive intervals shorter than the shortest middlebox idle timeout.

</details>

### Q122. A small request/response RPC protocol shows exactly ~40 ms extra latency per call, but only for some message sizes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Nagle's algorithm interacting with delayed ACKs. The sender writes a request in two small `write()`s; Nagle holds the second segment until the first is ACKed, and the receiver delays its ACK (up to 40 ms on Linux) waiting for more data. Result: a 40 ms stall.

**Diagnose →**
```bash
sudo tcpdump -ni eth0 -ttt host <peer> and port 7000
# look for ~0.040000 s gaps between a small segment and the next one
strace -f -e trace=write,sendto -p <pid>     # multiple small writes per message?
```

**Fix →** Set `TCP_NODELAY` on the socket (most HTTP and gRPC libraries already do), and coalesce writes into one buffer per message (`writev` or a buffered writer).

**Prevent →** Treat any repeated 40 ms or 200 ms latency step as a Nagle/delayed-ACK red flag; include p99 latency checks in protocol tests.

</details>

### Q123. A UDP metrics collector loses data under load, and `netstat -su` shows `receive buffer errors` increasing. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Packets arrive faster than the app reads them; the socket receive buffer fills and the kernel drops new datagrams. UDP gives no backpressure, so the sender never knows.

**Diagnose →**
```bash
netstat -su | grep -iE 'receive buffer errors|packet receive errors'
nstat -az UdpRcvbufErrors UdpInErrors
ss -uamn 'sport = :8125'        # skmem: rb = buffer size, d = drops
sysctl net.core.rmem_max net.core.rmem_default
```

**Fix →** Raise the max and request a bigger buffer in the app (`SO_RCVBUF`), and read faster (more reader threads, `SO_REUSEPORT` to spread across sockets):
```bash
sudo sysctl -w net.core.rmem_max=26214400 net.core.rmem_default=8388608
```

**Prevent →** Alert on `UdpRcvbufErrors`; for data you can't lose, switch to TCP or a queue.

</details>

### Q124. You set `nofile 65535` in `/etc/security/limits.conf`, but the systemd service still dies at 1024 open sockets. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `limits.conf` is applied by PAM to login sessions only. Services started by systemd ignore it and use `LimitNOFILE` from the unit (or `DefaultLimitNOFILE` in `system.conf`).

**Diagnose →**
```bash
systemctl show myapp -p LimitNOFILE -p LimitNOFILESoft
grep 'open files' /proc/$(systemctl show -p MainPID --value myapp)/limits
```

**Fix →**
```bash
sudo systemctl edit myapp
# [Service]
# LimitNOFILE=65536
sudo systemctl daemon-reload && sudo systemctl restart myapp
```

**Prevent →** Put limits in the unit file managed by config management, and verify via `/proc/<pid>/limits`, not `ulimit -n` in your shell.

</details>

### Q125. You raised `nf_conntrack_max` on Kubernetes nodes via sysctl, but after a reboot or kube-proxy restart the value is back to something else. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** kube-proxy sets `nf_conntrack_max` itself on startup based on `--conntrack-max-per-core` × CPU cores (with `--conntrack-min` as a floor), overriding your sysctl. Also, if `/etc/sysctl.d` is applied before the `nf_conntrack` module loads, the key doesn't exist yet and is skipped.

**Diagnose →**
```bash
sysctl net.netfilter.nf_conntrack_max
kubectl -n kube-system get cm kube-proxy -o yaml | grep -A3 conntrack
journalctl -b | grep -i 'nf_conntrack'
```

**Fix →** Set it in kube-proxy's config (`conntrack.maxPerCore`, `conntrack.min` in `KubeProxyConfiguration`) and restart the DaemonSet. For non-kube-proxy hosts, load the module early (`echo nf_conntrack > /etc/modules-load.d/conntrack.conf`).

**Prevent →** Verify node sysctls after boot with a DaemonSet check or node-problem-detector, not just at provisioning time.

</details>

---

## Load balancers and reverse proxies

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. nginx returns `502 Bad Gateway` and the error log says `connect() failed (111: Connection refused) while connecting to upstream`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** nginx reached the upstream host but nothing accepted on that port: the app is down or restarting, listens on a different port, or is bound to `127.0.0.1` while nginx connects via another IP (or vice versa, e.g. `localhost` resolving to `::1` while the app listens on IPv4 only).

**Diagnose →**
```bash
sudo tail -n 20 /var/log/nginx/error.log      # shows the exact upstream address
sudo ss -ltnp | grep <port>
curl -v http://<upstream-address>/health       # from the nginx host
```

**Fix →** Start/fix the app, or point `proxy_pass` at the address it actually listens on (use `127.0.0.1` instead of `localhost` to avoid the IPv6 surprise).

**Prevent →** Health checks with retries (`max_fails`, `fail_timeout`), process supervision, and readiness checks before nginx sends traffic.

</details>

### Q127. nginx returns `504 Gateway Time-out` with `upstream timed out (110: Connection timed out) while reading response header from upstream`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The connection to the upstream succeeded but the app didn't send response headers within `proxy_read_timeout` (60 s default). The request is slow (heavy query, lock, external call) or the app is overloaded.

**Diagnose →**
```bash
grep 'upstream timed out' /var/log/nginx/error.log | awk -F'request: ' '{print $2}' | sort | uniq -c | sort -rn | head
# add $upstream_response_time and $request_time to log_format to see distribution
```

**Fix →** Fix the slow endpoint (indexes, caching, async jobs for long work). Raise `proxy_read_timeout` only for specific known-long endpoints:
```nginx
location /reports/ {
    proxy_read_timeout 300s;
    proxy_pass http://app;
}
```

**Prevent →** Log upstream timings, set SLO-based alerts on p99, and move long tasks to background workers with polling or callbacks.

</details>

### Q128. An ALB target group shows all targets `unhealthy` with `Health checks failed with these codes: [404]`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The health check path doesn't exist on the app (default is `/`), the app requires a specific `Host` header, or a framework redirects/serves 404 for unknown hosts (the ALB sends the target IP as Host).

**Diagnose →**
```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn>
aws elbv2 describe-target-groups --target-group-arns <tg-arn> \
  --query 'TargetGroups[0].[HealthCheckPath,HealthCheckPort,Matcher]'
curl -i http://<target-private-ip>:<port>/health   # from a host in the VPC
```

**Fix →** Point the health check at a real lightweight endpoint (`/healthz`) and set the matcher (`200` or `200-399`). Allow the app to answer health checks for any Host (e.g. Django `ALLOWED_HOSTS`, Rails `config.hosts`).

**Prevent →** Ship a dedicated health endpoint in every service template and test it in CI.

</details>

### Q129. An ALB returns sporadic `502`s, and access logs show `elb_status_code=502` with `target_status_code=-`. Load is low. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target closed an idle keep-alive connection at the same moment the ALB sent a new request on it. This happens when the backend's keep-alive timeout is *shorter* than the ALB idle timeout (60 s default), e.g. Node.js `keepAliveTimeout` 5 s, gunicorn `--keep-alive 2`.

**Diagnose →**
```bash
aws elbv2 describe-load-balancer-attributes --load-balancer-arn <arn> \
  --query "Attributes[?Key=='idle_timeout.timeout_seconds']"
# check access logs: target_processing_time = -1 and 502 on low-traffic periods
```

**Fix →** Make the backend keep-alive longer than the ALB's idle timeout:
```js
server.keepAliveTimeout = 65000;
server.headersTimeout = 66000;
```
For nginx targets: `keepalive_timeout 75s;`.

**Prevent →** Rule of thumb: each inner hop's idle timeout must exceed the outer hop's.

</details>

### Q130. nginx logs `upstream prematurely closed connection while reading response header from upstream`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The upstream accepted the request and then closed the connection without sending a full response: the app worker crashed (OOM, segfault), hit its own timeout (gunicorn `--timeout`, PHP-FPM `request_terminate_timeout`), or closed a reused keepalive connection.

**Diagnose →**
```bash
grep 'prematurely closed' /var/log/nginx/error.log | tail
journalctl -u myapp --since '10 min ago' | grep -iE 'killed|timeout|worker'
dmesg -T | grep -i 'out of memory'
```

**Fix →** Fix the crash or raise the app's worker timeout above `proxy_read_timeout` if long requests are legitimate. If it's keepalive-related, raise the upstream's keepalive timeout or set `proxy_next_upstream error timeout;` for idempotent requests.

**Prevent →** Align app worker timeouts with proxy timeouts, and alert on worker restarts.

</details>

### Q131. nginx error log shows `no live upstreams while connecting to upstream` and every request gets 502. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Every server in the `upstream` block is marked failed by passive health checking (`max_fails` within `fail_timeout`). One short outage or a burst of errors can mark all of them down at once, and nginx then refuses to try any until `fail_timeout` expires.

**Diagnose →**
```bash
grep -B3 'no live upstreams' /var/log/nginx/error.log | head -20   # what failed first?
nginx -T | grep -A10 'upstream app'
```

**Fix →** Fix the underlying failure first. Then tune: `server 10.0.1.10:8080 max_fails=3 fail_timeout=10s;`, and avoid tiny upstream pools. Note that `proxy_next_upstream` codes such as `http_500` also count as failures.

**Prevent →** Use active health checks (NGINX Plus, HAProxy, Envoy, or an LB) and alert when more than half the pool is down.

</details>

### Q132. An NLB shows all targets healthy, but clients from the internet time out connecting through it. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With instance targets (or IP targets with client IP preservation on), the NLB passes the *client's* IP through. Health checks come from the NLB's private IPs (allowed), but the target's security group doesn't allow the client CIDRs, so real traffic is dropped.

**Diagnose →**
```bash
aws elbv2 describe-target-group-attributes --target-group-arn <tg> \
  --query "Attributes[?Key=='preserve_client_ip.enabled']"
aws ec2 describe-security-groups --group-ids <target-sg> --query 'SecurityGroups[0].IpPermissions'
# VPC Flow Logs on the target ENI will show REJECT for client IPs
```

**Fix →** Allow the client CIDRs (or `0.0.0.0/0` for public services) on the target SG for the listener port, or attach a security group to the NLB and reference it from the target SG.

**Prevent →** Use NLB security groups (supported since 2023) and template SG rules together with the listener.

</details>

### Q133. Behind a load balancer, users get logged out randomly or lose their cart between clicks. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Sessions are stored in the memory of each app instance, and the LB spreads requests across instances. Each request may land on an instance that doesn't know the session.

**Diagnose →**
```bash
curl -s -c /tmp/c -b /tmp/c -I https://app.example.com/ | grep -i -E 'set-cookie|x-served-by'
# repeat a few times and see if the backend changes along with the session
```

**Fix →** Short term: enable stickiness (ALB `stickiness.enabled=true` with `lb_cookie`). Proper fix: move sessions to a shared store (Redis, database) or use stateless signed tokens, so any instance can serve any request.

**Prevent →** Design apps stateless from day one; stickiness hurts scale-in and uneven load.

</details>

### Q134. HAProxy returns `503 Service Unavailable — No server is available to handle this request.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Every server in the backend is DOWN (failed health checks) or in maintenance, or the request matched a backend with no servers because of an ACL/`use_backend` mistake.

**Diagnose →**
```bash
echo "show servers state" | sudo socat stdio /run/haproxy/admin.sock
echo "show stat" | sudo socat stdio /run/haproxy/admin.sock | cut -d, -f1,2,18,37 | column -s, -t
sudo journalctl -u haproxy | grep -i 'is DOWN'
```

**Fix →** Fix the backend or the health check (`option httpchk GET /healthz`, correct port), or correct the routing ACL. A server in `MAINT` needs `set server be_app/web1 state ready`.

**Prevent →** Enable the stats page or Prometheus exporter and alert when a backend's active server count drops.

</details>

### Q135. nginx returns 502 for login requests only, with `upstream sent too big header while reading response header from upstream`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The upstream's response headers exceed `proxy_buffer_size` (4 KB or 8 KB by default). Login responses often carry large `Set-Cookie` headers (JWTs, SSO cookies) that push them over.

**Diagnose →**
```bash
curl -s -o /dev/null -D - http://127.0.0.1:8080/login -d 'u=x&p=y' | wc -c
```

**Fix →**
```nginx
proxy_buffer_size 16k;
proxy_buffers 8 16k;
proxy_busy_buffers_size 32k;
```
For FastCGI the equivalents are `fastcgi_buffer_size` / `fastcgi_buffers`.

**Prevent →** Keep cookies small (store session data server side, reference by ID) and include auth flows in load tests.

</details>

### Q136. After scaling a backend from 3 to 6 instances, the 3 new instances get almost no traffic. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Clients (or an upstream proxy) hold long-lived keep-alive or HTTP/2/gRPC connections to the original instances. An L4 LB balances *connections*, not requests, so existing connections never move. Least-outstanding-requests or slow start can also skew early traffic.

**Diagnose →**
```bash
# per-target connection counts
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name RequestCountPerTarget ...
ss -tn state established '( sport = :8080 )' | wc -l    # on old vs new instances
```

**Fix →** Use an L7 balancer that balances per request (ALB, Envoy, nginx with upstream keepalive pool). For gRPC, use client-side balancing or a proxy that understands HTTP/2 streams, and set `max_connection_age` on servers so clients reconnect periodically.

**Prevent →** Cap connection lifetime on servers and verify traffic distribution after every scale event.

</details>

### Q137. A backend behind nginx moved to a new IP (DNS updated), but nginx keeps sending traffic to the old IP until reload. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** When `proxy_pass` uses a literal hostname (or an `upstream` with hostnames in open-source nginx), nginx resolves it once at startup/reload and caches the IPs forever, ignoring TTL. Common with ELB hostnames, whose IPs change.

**Diagnose →**
```bash
dig +short backend.internal
sudo ss -tnp | grep nginx | awk '{print $5}' | sort | uniq -c   # still connecting to old IP
```

**Fix →** Force runtime resolution by using a variable plus a `resolver`:
```nginx
resolver 10.0.0.2 valid=30s ipv6=off;
location / {
    set $backend "http://backend.internal:8080";
    proxy_pass $backend;
}
```
nginx 1.27.3+ also supports `server backend.internal resolve;` inside an `upstream` block with a `zone`.

**Prevent →** Never proxy to ELB/cloud hostnames without dynamic resolution; reload nginx in deploy pipelines when upstreams change.

</details>

### Q138. After moving TLS termination to the load balancer, the browser shows `ERR_TOO_MANY_REDIRECTS`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The LB now talks HTTP to the app. The app sees a plain-HTTP request and redirects to HTTPS; the browser follows, the LB forwards it again as HTTP, and the loop repeats.

**Diagnose →**
```bash
curl -sIL https://app.example.com/ | grep -iE '^(HTTP|location)'
```

**Fix →** Make the app trust `X-Forwarded-Proto` from the LB (Django `SECURE_PROXY_SSL_HEADER`, Express `app.set('trust proxy', 1)`, Spring `server.forward-headers-strategy=native`), or do the HTTP→HTTPS redirect on the LB (ALB listener rule on port 80) and remove it from the app. In nginx behind an LB: `proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;`.

**Prevent →** Decide which single layer owns the HTTPS redirect and document it.

</details>

### Q139. ALB access logs show a growing number of `460` status codes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `460` means the client closed the connection before the ALB's idle timeout, usually because the client's own timeout is shorter than the time the backend takes. Typical: a mobile app or API gateway with a 10–30 s timeout calling a slow endpoint.

**Diagnose →**
```sql
-- Athena over ALB logs
SELECT request_url, count(*), avg(target_processing_time)
FROM alb_logs WHERE elb_status_code = 460
GROUP BY request_url ORDER BY 2 DESC LIMIT 20;
```

**Fix →** Speed up the endpoints involved, or make long operations asynchronous (202 + polling). Align client timeouts with the realistic processing time.

**Prevent →** Track 460 as a client-experienced error in dashboards, even though it is not a 5xx.

</details>

### Q140. Some requests through an ALB fail with HTTP `463`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ALB received an `X-Forwarded-For` header with more than 30 IP addresses. This happens when requests loop through proxies (a CDN → proxy → ALB → app that calls back into the same ALB), each hop appending an IP, or when a misbehaving client sends a huge header.

**Diagnose →**
```bash
# in ALB access logs, find the client and path
grep ' 463 ' alb.log | awk '{print $4, $13}' | sort | uniq -c
# reproduce
curl -s -o /dev/null -w '%{http_code}\n' -H "X-Forwarded-For: $(seq -s, 1 31 | sed 's/[0-9]*/1.1.1.1/g')" https://app.example.com/
```

**Fix →** Break the request loop (an app calling its own public URL, or a redirect chain between proxies) and make intermediate proxies replace instead of append untrusted XFF values.

**Prevent →** Map the request path through CDN/proxies/LBs and avoid hairpinning internal calls through the public entry point.

</details>

### Q141. nginx access logs suddenly show a spike of `499` status codes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `499` is nginx-specific: the client closed the connection before nginx sent a response. Either the upstream got slow and clients (or an LB in front) gave up, or a client/load generator uses aggressive timeouts.

**Diagnose →**
```bash
awk '$9 == 499 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
# with $request_time in log_format, check how long the 499 requests waited
```
If the 499s all cluster at e.g. exactly 60 s, an LB in front has a 60 s timeout.

**Fix →** Fix upstream latency, or align timeouts so nginx and the app time out before the outer layer does. If you need the upstream to finish work anyway, `proxy_ignore_client_abort on;` (use carefully).

**Prevent →** Alert on 499 rate together with upstream response time; they move together.

</details>

### Q142. After enabling PROXY protocol on nginx, every request fails and the log shows `broken header: "GET / HTTP/1.1"` while reading PROXY protocol. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** nginx expects a PROXY protocol header (`listen 80 proxy_protocol;`) but the sender (LB, health checker, or you testing with curl) sends plain HTTP. Both sides must agree.

**Diagnose →**
```bash
aws elbv2 describe-target-group-attributes --target-group-arn <tg> \
  --query "Attributes[?Key=='proxy_protocol_v2.enabled']"
curl -v --haproxy-protocol http://127.0.0.1/   # test as if from a PROXY-enabled LB
```

**Fix →** Enable PROXY protocol v2 on the NLB target group (or `send-proxy-v2` in HAProxy), or remove `proxy_protocol` from the listen directive. If health checks can't send it, put them on a separate listen port without PROXY protocol.

**Prevent →** Change the sender and receiver in the same rollout, behind a separate port first.

</details>

### Q143. A gRPC service behind nginx returns 502 and the log says `upstream sent no valid HTTP/1.0 header while reading response header from upstream`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** nginx is configured with `proxy_pass`, which speaks HTTP/1.x to upstreams, but the gRPC server only speaks HTTP/2 (h2c). The server answers with HTTP/2 frames, which nginx can't parse as an HTTP/1 response.

**Diagnose →**
```bash
nginx -T | grep -B5 -A5 'proxy_pass.*50051'
grpcurl -plaintext 127.0.0.1:50051 list     # confirm the backend works directly
```

**Fix →**
```nginx
server {
    listen 443 ssl;
    http2 on;
    location / {
        grpc_pass grpc://127.0.0.1:50051;   # grpcs:// if the backend uses TLS
        grpc_read_timeout 300s;
    }
}
```

**Prevent →** Use protocol-aware proxies for gRPC (nginx `grpc_pass`, Envoy, ALB target groups with protocol version `GRPC`).

</details>

### Q144. A database blip made every target fail its "deep" health check, the ALB marked them all unhealthy, and the whole site went down. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The health check endpoint tests downstream dependencies (DB, cache, other services). A shared dependency failing marks *every* instance unhealthy at once, so a partial outage becomes a full one. (An ALB fails open and routes to all targets when all are unhealthy, but other LBs, Kubernetes readiness, and DNS failover often don't.)

**Diagnose →**
```bash
curl -s http://<target>/healthz | jq .   # what does it check?
aws elbv2 describe-target-health --target-group-arn <tg> --query 'TargetHealthDescriptions[].TargetHealth'
```

**Fix →** Split checks: a shallow liveness/LB health check (process up, can serve) and a separate readiness or dependency status used for alerting. Let the app degrade gracefully (cached responses, feature flags) instead of failing the health check.

**Prevent →** Rule: an LB health check should only fail for problems that replacing *this instance* would fix.

</details>

### Q145. An app on an EC2 instance calls its own internal NLB, and connections time out only when the NLB picks that same instance. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Hairpinning with client IP preservation. The packet arrives at the instance with source = its own IP and destination = its own IP; the instance replies locally instead of via the NLB, the client socket sees a reply from an unexpected address, and the connection never completes.

**Diagnose →**
```bash
aws elbv2 describe-target-group-attributes --target-group-arn <tg> \
  --query "Attributes[?Key=='preserve_client_ip.enabled']"
sudo tcpdump -ni any host <instance-ip> and port <port>   # SYN from self, SYN-ACK goes direct
```
Roughly 1/N of calls fail, where N is the number of targets.

**Fix →** Disable client IP preservation on that target group (use PROXY protocol v2 if you need the client IP), use IP targets without preservation, or call the local service directly instead of through the NLB.

**Prevent →** Avoid services calling themselves through their own LB; document this NLB limitation in service templates.

</details>

### Q146. nginx refuses to start: `nginx: [emerg] host not found in upstream "backend:8080" in /etc/nginx/conf.d/app.conf:5`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** nginx resolves upstream hostnames at startup and fails if any don't resolve. In Docker Compose/Kubernetes this happens when nginx starts before the backend container/service exists, or the name is wrong (wrong network, wrong namespace).

**Diagnose →**
```bash
docker compose exec nginx getent hosts backend
docker network inspect <network> | grep -A3 backend
nginx -t
```

**Fix →** Put both containers on the same network and use the correct service name, add `depends_on` with a healthcheck condition, or use a variable with `resolver 127.0.0.11 valid=10s;` (Docker's embedded DNS) so resolution happens per request instead of at startup.

**Prevent →** Run `nginx -t` in CI against the target environment's names, and avoid hard startup dependencies between containers.

</details>

### Q147. Large downloads through nginx are slow to start, and the error log warns `an upstream response is buffered to a temporary file`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With `proxy_buffering on` (default), nginx reads the upstream response into memory buffers and then spills to disk (`proxy_temp_path`) when the response exceeds them. Large files cause disk I/O and can fill the disk; slow disks delay delivery.

**Diagnose →**
```bash
grep 'buffered to a temporary file' /var/log/nginx/error.log | wc -l
du -sh /var/lib/nginx/proxy 2>/dev/null || du -sh /var/cache/nginx/proxy_temp
df -h /var
```

**Fix →** For large downloads, stream instead of buffering to disk:
```nginx
location /downloads/ {
    proxy_max_temp_file_size 0;   # never spill to disk
    proxy_pass http://files;
}
```
Better: serve big static objects from S3/CloudFront or let the app return `X-Accel-Redirect` so nginx serves the file directly.

**Prevent →** Monitor the temp path's disk usage and keep large-object delivery off the app path.

</details>

### Q148. Server-Sent Events / streaming responses work directly against the app, but through nginx they arrive all at once at the end. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** nginx buffers upstream responses by default and only flushes when buffers fill or the response ends. Compression (`gzip on`) and some LBs add more buffering.

**Diagnose →**
```bash
curl -N http://127.0.0.1:8080/stream       # direct: events trickle in
curl -N https://app.example.com/stream     # via nginx: all at the end
```

**Fix →**
```nginx
location /stream {
    proxy_pass http://app;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_buffering off;
    proxy_read_timeout 1h;
    gzip off;
}
```
Or have the app send `X-Accel-Buffering: no` for streaming responses.

**Prevent →** Mark streaming endpoints explicitly in proxy config and include them in end-to-end tests through the real proxy chain.

</details>

### Q149. Rate limiting by client IP isn't working: attackers rotate the IP your app sees by sending their own `X-Forwarded-For` header. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The app (or nginx `real_ip`) takes the *left-most* XFF value, which the client fully controls. Or `set_real_ip_from` trusts `0.0.0.0/0`, so any sender can set the "real" IP.

**Diagnose →**
```bash
curl -s -H 'X-Forwarded-For: 1.2.3.4' https://app.example.com/whoami
nginx -T | grep -E 'real_ip|set_real_ip_from'
```
If `/whoami` echoes `1.2.3.4`, the header is trusted blindly.

**Fix →** Trust only your own proxies and take the right-most untrusted address:
```nginx
set_real_ip_from 10.0.0.0/16;        # your LB subnets only
set_real_ip_from 173.245.48.0/20;    # e.g. CDN ranges, kept up to date
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

**Prevent →** Do rate limiting at the edge (CDN/WAF/ALB) where the TCP peer is the real client, and document which hops are trusted.

</details>

### Q150. HAProxy logs show `504` responses with termination state `sH--`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** HAProxy's termination flags read: first letter `s` = server-side timeout, second `H` = while waiting for response headers. So the backend accepted the request but didn't send headers within `timeout server`. (Compare `cD` = client-side timeout during data, `SC` = server refused the connection.)

**Diagnose →**
```bash
grep ' sH-- ' /var/log/haproxy.log | awk '{print $9, $NF}' | sort | uniq -c | sort -rn | head
# Tr (server response time) in the timers field shows how long the server took
```

**Fix →** Fix the slow backend, or raise `timeout server` for that backend only. Make sure the backend's own timeout is shorter so it fails cleanly instead of HAProxy cutting it.

**Prevent →** Learn the termination flags table; parse them into your log pipeline as fields and alert on `s*` timeouts per backend.

</details>

---

## Cloud networking: VPC, firewalls, NAT, VPN and proxies

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. An EC2 instance in a private subnet hangs on `apt-get update` at `Connecting to archive.ubuntu.com`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The private subnet's route table has no `0.0.0.0/0` route to a NAT gateway, the NAT gateway itself sits in a private subnet (it must be in a public subnet with an IGW route), or the instance SG/NACL blocks outbound 80/443.

**Diagnose →**
```bash
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id> \
  --query 'RouteTables[].Routes'
aws ec2 describe-nat-gateways --nat-gateway-ids <nat-id> --query 'NatGateways[0].[State,SubnetId]'
curl -m 5 -sI https://archive.ubuntu.com    # from the instance
```

**Fix →** Add `0.0.0.0/0 → nat-xxxx` to the private route table, place the NAT gateway in a public subnet, and allow outbound HTTPS in the SG.

**Prevent →** Build VPCs from a tested Terraform module; add a VPC endpoint for S3 so package mirrors on S3 don't need NAT at all.

</details>

### Q152. A new EC2 instance in a "public" subnet can't be reached over SSH from the internet; the SG allows port 22 from your IP. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The instance has no public IPv4 (subnet doesn't auto-assign and none was requested), the subnet's route table has no `0.0.0.0/0 → igw-xxxx`, a NACL blocks it, or your IP changed.

**Diagnose →**
```bash
aws ec2 describe-instances --instance-ids <id> \
  --query 'Reservations[0].Instances[0].[PublicIpAddress,SubnetId]'
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>
curl -s https://checkip.amazonaws.com
```
Use VPC Reachability Analyzer from the IGW to the ENI on port 22 for a definitive answer.

**Fix →** Attach an Elastic IP or enable auto-assign public IP, add the IGW route, fix the NACL, update the SG with your current IP.

**Prevent →** Prefer SSM Session Manager over public SSH entirely; no inbound port, no public IP needed.

</details>

### Q153. The SG allows inbound 443 and the app works from inside the VPC, but internet clients time out. The subnet NACL allows inbound 443. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** NACLs are stateless. Responses go back to the client's ephemeral port (1024–65535), and the NACL's outbound rules don't allow that, so replies are dropped. Security groups are stateful and don't have this problem.

**Diagnose →**
```bash
aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id> \
  --query 'NetworkAcls[0].Entries'
# Flow Logs: ACCEPT on the inbound record, REJECT on the outbound response
```

**Fix →** Add an outbound NACL rule allowing TCP 1024–65535 to `0.0.0.0/0` (and the matching inbound rule for responses to the instance's own outbound calls).

**Prevent →** Keep NACLs simple (default allow) and use SGs for fine-grained control unless compliance demands NACLs.

</details>

### Q154. A VPC peering connection is `Active`, but instances in the two VPCs can't ping each other. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Peering does not add routes. Both VPCs' route tables need a route to the other's CIDR via the `pcx-` connection, and both SGs must allow the traffic (ICMP for ping). Also: peering is not transitive.

**Diagnose →**
```bash
aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-a> \
  --query 'RouteTables[].Routes[?VpcPeeringConnectionId]'
aws ec2 describe-security-groups --group-ids <sg-b>
```

**Fix →** Add `10.1.0.0/16 → pcx-xxxx` in VPC A's route tables and `10.0.0.0/16 → pcx-xxxx` in VPC B's, then allow the source CIDR (or the peer SG, same region only) in the SGs.

**Prevent →** Create peering, routes and SG rules in the same Terraform module so none can be forgotten.

</details>

### Q155. ALB targets are `unhealthy` with reason `Target.Timeout`, and VPC Flow Logs show `REJECT` from the ALB's IPs to the instances. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The instance SG doesn't allow traffic from the ALB on the target/health check port. Often the SG allows only `0.0.0.0/0:443` while the app listens on 8080, or it references the wrong SG.

**Diagnose →**
```bash
aws elbv2 describe-load-balancers --names <alb> --query 'LoadBalancers[0].SecurityGroups'
aws ec2 describe-security-groups --group-ids <instance-sg> --query 'SecurityGroups[0].IpPermissions'
```

**Fix →** Allow the target port (and health check port if different) from the ALB's security group:
```bash
aws ec2 authorize-security-group-ingress --group-id <instance-sg> \
  --protocol tcp --port 8080 --source-group <alb-sg>
```

**Prevent →** Always reference the LB SG rather than CIDRs; manage both in one module.

</details>

### Q156. Two VPCs are attached to a Transit Gateway, but traffic between them fails. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Three places to get right: (1) each VPC's subnet route tables need a route to the other CIDR via `tgw-xxxx`; (2) each attachment must be associated with a TGW route table; (3) that TGW route table needs routes (propagated or static) to the other attachment. Plus the TGW attachment must have a subnet in each AZ the traffic uses.

**Diagnose →**
```bash
aws ec2 get-transit-gateway-route-table-associations --transit-gateway-route-table-id <rtb>
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <rtb> \
  --filters Name=type,Values=static,propagated
aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc> --query 'RouteTables[].Routes[?TransitGatewayId]'
```
Reachability Analyzer and TGW Flow Logs help confirm which hop drops.

**Fix →** Add the missing VPC route, association or propagation; add attachment subnets in missing AZs.

**Prevent →** Use TGW Network Manager / Route Analyzer and codify TGW route tables, especially when segmenting prod and non-prod.

</details>

### Q157. An AWS Site-to-Site VPN tunnel stays `DOWN`, and the on-prem firewall logs show `NO_PROPOSAL_CHOSEN`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** IKE (phase 1) or IPsec (phase 2) parameters don't match: encryption (AES128/256, GCM), integrity (SHA1/SHA2), DH group, IKE version, or lifetime. AWS accepts a set; the on-prem side may propose something outside it.

**Diagnose →**
```bash
aws ec2 describe-vpn-connections --vpn-connection-ids <vpn> \
  --query 'VpnConnections[0].VgwTelemetry'
# enable tunnel logging to CloudWatch (TunnelOptions.LogOptions) and read the IKE negotiation
# on strongSwan: swanctl --log / journalctl -u strongswan
```

**Fix →** Align proposals on both sides, e.g. IKEv2, AES256, SHA2-256, DH group 14+ for both phases. You can restrict AWS's tunnel options to exactly what on-prem supports (`modify-vpn-tunnel-options`).

**Prevent →** Keep the tunnel parameters in code on both sides and use the downloadable vendor configuration from AWS as the baseline.

</details>

### Q158. The VPN tunnel is `UP`, but no traffic passes between on-prem and the VPC. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Routing, not IPsec: VPC route tables don't have the on-prem prefix (route propagation from the VGW off, no static route), BGP isn't advertising the right prefixes, the on-prem side routes the VPC CIDR elsewhere, or a policy-based device has a traffic selector (encryption domain) that doesn't match (`TS_UNACCEPTABLE`). SGs/NACLs can also block.

**Diagnose →**
```bash
aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc> --query 'RouteTables[].PropagatingVgws'
aws ec2 describe-vpn-connections --vpn-connection-ids <vpn> --query 'VpnConnections[0].VgwTelemetry[].[Status,AcceptedRouteCount]'
```
`AcceptedRouteCount: 0` means BGP isn't sending prefixes.

**Fix →** Enable route propagation or add static routes, advertise the right prefixes via BGP, align traffic selectors (AWS uses `0.0.0.0/0` by default on its side unless you set local/remote network CIDRs), and open SGs for the on-prem CIDR.

**Prevent →** Use route-based (BGP) VPNs over policy-based ones and monitor `TunnelState` plus accepted route count.

</details>

### Q159. A Site-to-Site VPN tunnel goes down every few hours and comes back only when on-prem sends traffic. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rekey or DPD mismatches. AWS's side waits for the customer gateway to initiate by default; if the tunnel idles and DPD times out (or a rekey fails because of lifetime/proposal mismatch), the tunnel drops and nobody re-initiates until on-prem traffic appears.

**Diagnose →**
```bash
aws ec2 describe-vpn-connections --vpn-connection-ids <vpn> \
  --query 'VpnConnections[0].Options.TunnelOptions[].[DpdTimeoutAction,StartupAction,Phase1LifetimeSeconds,Phase2LifetimeSeconds]'
# CloudWatch tunnel logs: look for DPD timeout or rekey failures before each drop
```

**Fix →** Set `DpdTimeoutAction=restart` and `StartupAction=start` on the AWS side so AWS re-initiates, match phase 1/2 lifetimes with on-prem, and enable DPD on the customer gateway.

**Prevent →** Alarm on `TunnelState` per tunnel, run both tunnels active with BGP, and keep a lightweight keepalive (e.g. a monitoring ping across the tunnel).

</details>

### Q160. With `HTTPS_PROXY` set on a server, internet calls work but calls to `https://api.internal.corp` now fail with `502` from the proxy. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The internal call is also being sent to the corporate proxy, which can't reach internal names. `NO_PROXY` doesn't include the internal domain or IP ranges.

**Diagnose →**
```bash
env | grep -i _proxy
curl -v https://api.internal.corp 2>&1 | grep -iE 'proxy|connected to'
```

**Fix →**
```bash
export NO_PROXY=localhost,127.0.0.1,.internal.corp,10.0.0.0/8,169.254.169.254
export no_proxy=$NO_PROXY
```
Use the leading-dot domain form; CIDR support varies by tool (curl and Go support it, some older tools don't), so also list key IPs if needed.

**Prevent →** Set both upper- and lowercase variants centrally (e.g. `/etc/environment`, systemd drop-ins), and always include the metadata IP and cluster ranges.

</details>

### Q161. `docker pull` fails behind the corporate proxy with `dial tcp: lookup registry-1.docker.io: i/o timeout`, although `curl` works in your shell. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Docker daemon is a systemd service; it doesn't inherit your shell's `HTTPS_PROXY`. It tries to go direct, and direct DNS/egress is blocked.

**Diagnose →**
```bash
systemctl show docker --property=Environment
docker info | grep -i proxy
```

**Fix →**
```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf <<'CONF'
[Service]
Environment="HTTP_PROXY=http://proxy.corp:3128"
Environment="HTTPS_PROXY=http://proxy.corp:3128"
Environment="NO_PROXY=localhost,127.0.0.1,.corp,registry.internal"
CONF
sudo systemctl daemon-reload && sudo systemctl restart docker
```
Containers need their own proxy settings at build/run time (`~/.docker/config.json` `proxies` block).

**Prevent →** Configure daemon proxy and a pull-through registry mirror in the base image so builds don't depend on the corporate proxy.

</details>

### Q162. `curl: (56) CONNECT tunnel failed, response 407` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The forward proxy requires authentication (`407 Proxy Authentication Required`) and curl sent none, or wrong credentials, or the proxy wants NTLM/Kerberos instead of Basic.

**Diagnose →**
```bash
curl -v -x http://proxy.corp:8080 https://example.com 2>&1 | grep -i 'proxy-authenticate'
```
The `Proxy-Authenticate` header tells you the scheme (Basic, NTLM, Negotiate).

**Fix →**
```bash
curl -x http://proxy.corp:8080 --proxy-user 'svc_user:pass' https://example.com
curl -x http://proxy.corp:8080 --proxy-ntlm --proxy-user 'DOMAIN\svc_user' https://example.com
```
URL-encode special characters if credentials are inside `HTTPS_PROXY=http://user:p%40ss@proxy:8080`.

**Prevent →** For servers, ask for IP-based allow listing on the proxy or a dedicated service account, not personal credentials in env vars.

</details>

### Q163. After adding an S3 gateway endpoint, EC2 instances get `AccessDenied` on buckets they could read before. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Traffic now goes through the endpoint, and either the endpoint policy is restrictive (only allows certain buckets), or the bucket policy uses `aws:SourceIp` conditions that no longer match (requests via a gateway endpoint carry private IPs, not the NAT EIP).

**Diagnose →**
```bash
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <vpce> --query 'VpcEndpoints[0].PolicyDocument'
aws s3api get-bucket-policy --bucket <bucket> --query Policy --output text | jq .
```

**Fix →** Allow the bucket in the endpoint policy, and replace `aws:SourceIp` conditions with `aws:SourceVpce` or `aws:SourceVpc`.

**Prevent →** Use `aws:SourceVpce` from the start for private-access buckets and test endpoint rollouts in one subnet first.

</details>

### Q164. You created an interface VPC endpoint for Secrets Manager, but `dig secretsmanager.ap-south-1.amazonaws.com` still returns public IPs from the instance. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Private DNS isn't enabled on the endpoint, the VPC lacks `enableDnsSupport`/`enableDnsHostnames`, or the instance uses a custom DNS server that doesn't forward to the VPC resolver (`169.254.169.253` / VPC+2), so the private hosted zone the endpoint creates is never consulted.

**Diagnose →**
```bash
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <vpce> --query 'VpcEndpoints[0].PrivateDnsEnabled'
aws ec2 describe-vpc-attribute --vpc-id <vpc> --attribute enableDnsHostnames
cat /etc/resolv.conf
dig +short secretsmanager.ap-south-1.amazonaws.com @169.254.169.253
```

**Fix →** Enable private DNS on the endpoint and the VPC DNS attributes, and forward AWS service domains from custom resolvers to the VPC resolver (or Route 53 Resolver inbound endpoint for on-prem).

**Prevent →** Make endpoints with `private_dns_enabled = true` in Terraform, and verify resolution as part of the rollout.

</details>

### Q165. A Lambda function attached to a VPC can reach RDS but times out on any call to the internet. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Lambda ENIs in a VPC never get public IPs, even in a public subnet. Internet access needs the function in private subnets with a route to a NAT gateway (or VPC endpoints for AWS services).

**Diagnose →**
```bash
aws lambda get-function-configuration --function-name <fn> --query 'VpcConfig'
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>
```

**Fix →** Move the function to private subnets with `0.0.0.0/0 → nat-xxxx`, allow outbound in its SG, and add VPC endpoints for AWS APIs it calls (S3, DynamoDB, Secrets Manager) to avoid NAT costs.

**Prevent →** Only attach Lambda to a VPC when it needs private resources, and use a standard "private subnets with NAT" pattern when you do.

</details>

### Q166. A GCP VM without an external IP can't reach the internet, and an ingress firewall rule you added for port 8080 doesn't work either. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No Cloud NAT configured on the Cloud Router for that region/subnet, so there's no egress path. For the ingress rule: GCP firewall rules apply via target tags or service accounts; the VM doesn't have the tag, or a higher-priority deny rule wins.

**Diagnose →**
```bash
gcloud compute routers nats list --router=<router> --region=<region>
gcloud compute instances describe <vm> --zone=<zone> --format='value(tags.items)'
gcloud compute firewall-rules list --filter="network=<vpc>" \
  --format='table(name,direction,priority,sourceRanges.list(),allowed[].map().firewall_rule().list(),targetTags.list())'
```
Connectivity Tests (`gcloud network-management connectivity-tests create`) shows which rule blocks.

**Fix →** Create a Cloud Router + Cloud NAT for the subnet, and add the right network tag to the VM (or fix rule priority).

**Prevent →** Use service-account-based firewall targets and enable firewall rule logging on critical rules.

</details>

### Q167. In Azure, the NSG on a VM's NIC allows inbound 443, but traffic is still blocked. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Azure evaluates NSGs at both the subnet and the NIC level; inbound traffic must be allowed by both. A subnet-level NSG (or a lower-numbered deny rule) is blocking. A UDR sending traffic to a firewall appliance can also drop it.

**Diagnose →**
```bash
az network nic list-effective-nsg -g <rg> -n <nic>
az network watcher test-ip-flow -g <rg> --vm <vm> --direction Inbound \
  --protocol TCP --local <vm-ip>:443 --remote <client-ip>:50000
az network nic show-effective-route-table -g <rg> -n <nic> -o table
```

**Fix →** Allow 443 in the subnet NSG too, or remove the conflicting deny rule; fix the UDR/firewall policy if traffic is forced through an NVA.

**Prevent →** Pick one level (usually subnet) for NSGs as a team convention, and use IP flow verify in troubleshooting runbooks.

</details>

### Q168. Right after a new partner VPN came up, part of your VPC (10.0.8.0/22) became unreachable from your own office. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The partner advertises an overlapping, more specific prefix over BGP (e.g. `10.0.8.0/24`). Longest prefix match wins everywhere those routes propagate (VGW/TGW route tables, on-prem routers), so traffic for that range now goes to the partner.

**Diagnose →**
```bash
aws ec2 describe-route-tables --route-table-ids <rtb> --query 'RouteTables[0].Routes[?Origin==`EnableVgwRoutePropagation`]'
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <rtb> --filters Name=route-search.longest-prefix-match,Values=10.0.8.10/32
```

**Fix →** Filter what you accept from the partner (prefix lists, separate TGW route table for partner attachments with no propagation into internal tables), and ask the partner to advertise only agreed prefixes. Where overlap is permanent, use NAT (private NAT gateway) to translate.

**Prevent →** Isolate third-party connectivity in its own TGW route domain and maintain an IPAM registry of all allocated ranges.

</details>

### Q169. Traffic between two VPCs through a central inspection VPC with firewall appliances works for some flows and silently drops others. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Asymmetric routing across AZs. The Transit Gateway picks the firewall attachment in the AZ of the source for one direction and a different AZ for the return. Stateful firewalls in different AZs don't share state, so the return packet is dropped.

**Diagnose →**
```bash
aws ec2 describe-transit-gateway-vpc-attachments --transit-gateway-attachment-ids <inspection-attach> \
  --query 'TransitGatewayVpcAttachments[0].Options.ApplianceModeSupport'
# firewall logs: SYN seen in AZ-a, SYN-ACK arriving in AZ-b with no state
```

**Fix →** Enable appliance mode on the inspection VPC attachment so TGW keeps both directions of a flow on the same AZ:
```bash
aws ec2 modify-transit-gateway-vpc-attachment --transit-gateway-attachment-id <id> \
  --options ApplianceModeSupport=enable
```
Or use Gateway Load Balancer, which keeps flow symmetry by design.

**Prevent →** Make appliance mode part of the inspection VPC module and test cross-AZ flows explicitly.

</details>

### Q170. A self-managed NAT instance is in place and routes point to it, but private instances still can't reach the internet. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Source/destination check is enabled on the NAT instance's ENI (AWS drops traffic not addressed to/from the instance itself), IP forwarding is off, or there's no MASQUERADE rule.

**Diagnose →**
```bash
aws ec2 describe-instance-attribute --instance-id <nat> --attribute sourceDestCheck
sysctl net.ipv4.ip_forward
sudo iptables -t nat -S POSTROUTING
```

**Fix →**
```bash
aws ec2 modify-instance-attribute --instance-id <nat> --no-source-dest-check
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o eth0 -s 10.0.0.0/16 -j MASQUERADE
```
Persist both settings.

**Prevent →** Use managed NAT gateways unless cost forces otherwise; if you run NAT instances, bake them from a tested image with an ASG for recovery.

</details>

### Q171. Traffic steered to third-party firewalls behind a Gateway Load Balancer is all dropped, and GWLB targets show `unhealthy`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** GWLB sends traffic to appliances encapsulated in GENEVE on UDP 6081. The appliance SG/NACL doesn't allow UDP 6081 from the GWLB, the appliance doesn't have GENEVE enabled, or the health check port (separate, typically TCP/HTTP) isn't answered. Also common: GWLB endpoint routes are missing in the ingress (edge) route table, so return traffic bypasses inspection.

**Diagnose →**
```bash
aws elbv2 describe-target-health --target-group-arn <gwlb-tg>
aws ec2 describe-security-groups --group-ids <appliance-sg> --query 'SecurityGroups[0].IpPermissions'
sudo tcpdump -ni eth0 udp port 6081 -c 20      # on the appliance
```

**Fix →** Allow UDP 6081 and the health check port from the GWLB subnets, enable GENEVE on the appliance, and add symmetric routes: subnet → GWLBe → IGW, and an IGW edge route table sending the subnet CIDR back via the GWLBe.

**Prevent →** Use the vendor's GWLB reference architecture and verify both traffic directions with Reachability Analyzer.

</details>

### Q172. `terraform apply` fails with `RulesPerSecurityGroupLimitExceeded: The maximum number of rules per security group has been reached`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An SG has a default quota of 60 inbound and 60 outbound rules (each CIDR counts as a rule). Allow-listing many partner IPs or CDN ranges one by one hits it quickly.

**Diagnose →**
```bash
aws ec2 describe-security-groups --group-ids <sg> --query 'length(SecurityGroups[0].IpPermissions[].IpRanges[])'
aws service-quotas get-service-quota --service-code vpc --quota-code L-0EA8095F
```

**Fix →** Use managed prefix lists (one reference counts as rules equal to its max entries, but is much easier to manage), AWS-managed prefix lists (e.g. CloudFront origin-facing), split rules across SGs, or request a quota increase (rules × SGs per ENI is capped at 1000).

**Prevent →** Reference SGs and prefix lists instead of raw CIDRs; review rules that grow from ad-hoc requests.

</details>

### Q173. AWS Client VPN users connect successfully but can't reach anything in the VPC. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Connecting only proves authentication. Access also needs an authorization rule for the destination CIDR (for that user group), a route in the Client VPN route table, a target network association, and SGs on resources allowing the Client VPN endpoint's SG. If the client CIDR overlaps the user's home LAN or the VPC, routing breaks too.

**Diagnose →**
```bash
aws ec2 describe-client-vpn-authorization-rules --client-vpn-endpoint-id <cvpn>
aws ec2 describe-client-vpn-routes --client-vpn-endpoint-id <cvpn>
aws ec2 describe-client-vpn-target-networks --client-vpn-endpoint-id <cvpn>
```

**Fix →** Add `authorize-client-vpn-ingress` for the VPC CIDR (scoped to an AD/SAML group), add routes, allow the endpoint SG on target resources, and choose a client CIDR that doesn't overlap.

**Prevent →** Template the full Client VPN setup and enable connection logs to CloudWatch.

</details>

### Q174. A Direct Connect virtual interface shows the BGP session stuck in `Idle`/`Active`, never `Established`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Layer 2/3 mismatch on the VIF: wrong VLAN tag, wrong peer IPs (/30 or /31), wrong ASN on either side, MD5 auth key mismatch, or the router not reaching the AWS peer IP at all. `Active` usually means TCP 179 attempts aren't completing.

**Diagnose →**
```bash
aws directconnect describe-virtual-interfaces --virtual-interface-id <vif> \
  --query 'virtualInterfaces[0].[vlan,asn,amazonSideAsn,amazonAddress,customerAddress,bgpPeers[0].bgpStatus]'
# on the router (Cisco-style)
show ip bgp summary
ping <amazon-peer-ip> source <customer-peer-ip>
```
If ping fails, it's L2/L3 (VLAN, IP, cross-connect); if ping works, it's BGP config (ASN, MD5).

**Fix →** Match VLAN, IPs, ASN and MD5 key exactly with the AWS VIF configuration; check the cross-connect light levels with the provider if the link is down.

**Prevent →** Generate router config from the VIF data, keep a VPN as backup, and alarm on `ConnectionState` and BGP status.

</details>

### Q175. A partner says some of your API calls arrive from an IP they never allow-listed, so they block them. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You run one NAT gateway per AZ (correct for resilience), each with its own Elastic IP, but gave the partner only one. Or a new NAT gateway / recreated EIP changed the egress IP, or some workloads egress via a different path (VPC endpoints, IPv6, a second VPC).

**Diagnose →**
```bash
aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=<vpc> \
  --query 'NatGateways[].NatGatewayAddresses[].PublicIp'
# from instances in each AZ
curl -s https://checkip.amazonaws.com
```

**Fix →** Give the partner all NAT EIPs, or route partner-bound traffic through a single egress (a proxy or specific NAT) if they insist on one IP. Protect EIPs from deletion in Terraform (`prevent_destroy`).

**Prevent →** Keep an "egress IPs" inventory per environment and treat EIP changes as a change-managed event.

</details>

---

## Containers, Kubernetes, IPv6 and packet captures

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. After someone ran `iptables -F` (or restarted firewalld), all Docker containers lost outbound connectivity. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Docker programs its own iptables chains (`DOCKER`, `DOCKER-USER`, NAT `MASQUERADE` for `172.17.0.0/16`). Flushing rules or a firewalld reload removed them, so container traffic is no longer NATed. A reset of `net.ipv4.ip_forward=0` has the same effect.

**Diagnose →**
```bash
sudo iptables -t nat -S POSTROUTING | grep MASQUERADE
sudo iptables -S FORWARD | head
sysctl net.ipv4.ip_forward
docker run --rm busybox ping -c2 1.1.1.1
```

**Fix →** `sudo systemctl restart docker` recreates the chains (it restarts containers unless `live-restore` is set). Re-enable forwarding if needed.

**Prevent →** Never flush iptables on Docker/Kubernetes hosts; put your own rules in the `DOCKER-USER` chain and manage host firewalls with tools that preserve Docker's rules.

</details>

### Q177. `docker run -p 8080:80 myapp` starts fine, but `curl localhost:8080` returns `Connection reset by peer`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Docker's port mapping accepts the connection, but inside the container nothing listens on `0.0.0.0:80`: the app listens on `127.0.0.1` (only reachable from inside the container) or on a different port (e.g. 3000).

**Diagnose →**
```bash
docker exec myapp ss -ltn      # or: docker exec myapp netstat -ltn
docker logs myapp | grep -i listen
docker port myapp
```

**Fix →** Bind the app to `0.0.0.0` (Flask `--host=0.0.0.0`, Vite `--host`, Rails `-b 0.0.0.0`) and map the right container port (`-p 8080:3000`).

**Prevent →** `EXPOSE` the correct port in the Dockerfile and add a `HEALTHCHECK` that hits it.

</details>

### Q178. Two containers on the default bridge network can't reach each other by container name. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The default `bridge` network has no embedded DNS for container names. Name resolution works only on user-defined networks (and in Compose, which creates one).

**Diagnose →**
```bash
docker network inspect bridge --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
docker exec app getent hosts db
```

**Fix →**
```bash
docker network create appnet
docker run -d --name db --network appnet postgres:17
docker run -d --name app --network appnet myapp
docker exec app getent hosts db
```

**Prevent →** Always use user-defined networks or Compose; never rely on the deprecated `--link`.

</details>

### Q179. `docker run -p 80:80 nginx` fails with `Bind for 0.0.0.0:80 failed: port is already allocated`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another container already publishes host port 80, or a host process (nginx, Apache) listens on it.

**Diagnose →**
```bash
docker ps --format '{{.Names}}\t{{.Ports}}' | grep ':80->'
sudo ss -ltnp 'sport = :80'
```

**Fix →** Stop the conflicting container/service, or publish a different host port (`-p 8081:80`). If no container shows it but the error persists, a stale `docker-proxy` may hold it; restarting Docker clears that.

**Prevent →** Put a single reverse proxy on 80/443 and keep app containers on unpublished ports in a shared network.

</details>

### Q180. A new Kubernetes node stays `NotReady` with `container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CNI plugin isn't installed or running on that node: the CNI DaemonSet pod (Calico, Cilium, aws-node) is failing, can't be scheduled due to taints, or `/etc/cni/net.d/` has no config. On a fresh cluster, the CNI was never applied.

**Diagnose →**
```bash
kubectl describe node <node> | grep -A5 Conditions
kubectl -n kube-system get pods -o wide --field-selector spec.nodeName=<node>
kubectl -n kube-system logs <cni-pod> --all-containers --tail=50
ls /etc/cni/net.d/ /opt/cni/bin/        # on the node
```

**Fix →** Install/repair the CNI, fix the failing CNI pod (image pull, IAM permissions for aws-node, wrong pod CIDR), and make sure its DaemonSet tolerates the node's taints.

**Prevent →** Treat the CNI as part of cluster bootstrap and alert on DaemonSet `numberUnavailable > 0`.

</details>

### Q181. A Service returns `Connection refused` / nothing, and `kubectl get endpoints myapp` shows `<none>`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No ready pods match the Service: the `selector` labels don't match the pod labels, pods are failing readiness probes, or they are in another namespace.

**Diagnose →**
```bash
kubectl get svc myapp -o jsonpath='{.spec.selector}'; echo
kubectl get pods -l app=myapp --show-labels
kubectl get endpointslices -l kubernetes.io/service-name=myapp
kubectl describe pod <pod> | grep -A5 Readiness
```

**Fix →** Align the selector with the pod template labels, or fix the readiness probe (wrong path/port, too short `initialDelaySeconds`).

**Prevent →** Template Service and Deployment labels together (Helm/Kustomize) and add a check that every Service has endpoints after deploy.

</details>

### Q182. `curl` to a pod IP works, but `curl` to the Service's ClusterIP times out. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Service's `targetPort` doesn't match the container port, or kube-proxy (or its eBPF replacement) isn't programming rules on that node: crashed, wrong mode, or rules out of sync.

**Diagnose →**
```bash
kubectl get svc myapp -o jsonpath='{.spec.ports}'; echo
kubectl get endpointslices -l kubernetes.io/service-name=myapp -o yaml | grep -A3 ports
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
sudo iptables -t nat -S KUBE-SERVICES | grep <cluster-ip>      # iptables mode
sudo ipvsadm -Ln | grep -A3 <cluster-ip>                         # ipvs mode
```

**Fix →** Correct `targetPort` (name the container port and reference it by name), or restart/fix kube-proxy on the affected node.

**Prevent →** Use named ports and alert on kube-proxy `sync_proxy_rules` errors.

</details>

### Q183. A NodePort Service answers on some nodes and times out on others. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `externalTrafficPolicy: Local` is set: only nodes running a ready pod for that Service answer; others drop the traffic (by design, to preserve client IP). Otherwise, security groups or host firewalls differ between nodes.

**Diagnose →**
```bash
kubectl get svc myapp -o jsonpath='{.spec.externalTrafficPolicy}'; echo
kubectl get pods -l app=myapp -o wide           # which nodes have pods
curl -m 3 http://<node-ip>:<nodeport>/          # per node
```

**Fix →** Either use `Cluster` policy (any node forwards, client IP is SNATed), or keep `Local` and make sure the LB health-checks `healthCheckNodePort` so it only sends traffic to nodes with pods.

**Prevent →** Use `Local` only behind an LB that honours the health-check node port; spread pods with topology spread constraints.

</details>

### Q184. Requests to your Ingress hostname get `404 Not Found` from the ingress-nginx default backend. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The controller didn't match any rule: missing or wrong `ingressClassName` (so the controller ignores the Ingress), the `host` doesn't match the request's Host header, or the path/`pathType` doesn't match.

**Diagnose →**
```bash
kubectl get ingress -A
kubectl get ingressclass
kubectl describe ingress myapp       # events: "Scheduled for sync" means the controller picked it up
curl -v -H 'Host: app.example.com' http://<ingress-lb>/
```

**Fix →** Set `spec.ingressClassName: nginx`, correct the host, and use `pathType: Prefix` with `path: /` for a catch-all.

**Prevent →** Lint manifests (kubeconform) and have one default IngressClass annotated `ingressclass.kubernetes.io/is-default-class: "true"`.

</details>

### Q185. ingress-nginx returns `503 Service Temporarily Unavailable` for one host while others work. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Ingress rule matched, but the backend Service has no ready endpoints (pods crashing, failing readiness, selector mismatch), or the Ingress references a wrong Service name/port.

**Diagnose →**
```bash
kubectl describe ingress myapp | grep -A3 Backend
kubectl get endpointslices -l kubernetes.io/service-name=myapp
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller | grep -i 'endpoints\|myapp' | tail
```
The controller logs `Service "ns/myapp" does not have any active Endpoint` in this case.

**Fix →** Fix the pods/readiness or the Service reference (name and `port.number`/`port.name` in the Ingress backend).

**Prevent →** Use PodDisruptionBudgets and rolling update settings so a deploy never takes all endpoints away at once.

</details>

### Q186. After adding a NetworkPolicy that allows ingress to `backend` from `frontend`, the frontend still can't connect. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A default-deny *egress* policy in the frontend namespace blocks the outgoing side (both ends must allow), the selectors are wrong, or the rule semantics are off: `namespaceSelector` and `podSelector` in the *same* `from` item mean AND, in separate items they mean OR. Also, the policy needs the pod port (`targetPort`), not the Service port.

**Diagnose →**
```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy allow-frontend -n backend
kubectl get ns frontend --show-labels       # kubernetes.io/metadata.name=frontend
kubectl exec -n frontend deploy/frontend -- nc -vz -w3 backend.backend.svc 8080
```

**Fix →**
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels: {kubernetes.io/metadata.name: frontend}
    podSelector:
      matchLabels: {app: frontend}
  ports:
  - port: 8080
```
And add an egress policy in `frontend` allowing traffic to `backend` on 8080 (plus DNS to kube-system on 53).

**Prevent →** Use a policy visualiser (Cilium editor, `kubectl np-viewer`) and policy tests in CI.

</details>

### Q187. A `type: LoadBalancer` Service stays with `EXTERNAL-IP <pending>` forever. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No controller can provision a load balancer: a bare-metal/kind/minikube cluster without MetalLB, a missing or broken AWS Load Balancer Controller / cloud controller manager, missing IAM permissions, or subnets not tagged for LB discovery.

**Diagnose →**
```bash
kubectl describe svc myapp | tail -20                # events show the controller error
kubectl -n kube-system logs deploy/aws-load-balancer-controller | tail
aws ec2 describe-subnets --filters Name=tag:kubernetes.io/role/elb,Values=1 --query 'Subnets[].SubnetId'
```

**Fix →** Install MetalLB (bare metal) or the cloud's controller, fix its IAM role (IRSA/Pod Identity), and tag public subnets `kubernetes.io/role/elb=1` (private: `kubernetes.io/role/internal-elb=1`).

**Prevent →** Include the LB controller and subnet tags in cluster provisioning code and smoke-test a LoadBalancer Service on every new cluster.

</details>

### Q188. A pod can't reach the EC2 instance metadata service: `curl http://169.254.169.254/latest/meta-data/` hangs, while it works from the node. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** IMDSv2 with a PUT response hop limit of 1. The token response can't make the extra hop from the node into the pod network namespace, so SDKs hang waiting for a token. This is often intentional, to stop pods from using the node's IAM role.

**Diagnose →**
```bash
aws ec2 describe-instances --instance-ids <id> \
  --query 'Reservations[0].Instances[0].MetadataOptions'
kubectl exec <pod> -- curl -s -m3 -X PUT http://169.254.169.254/latest/api/token \
  -H 'X-aws-ec2-metadata-token-ttl-seconds: 60'
```

**Fix →** Preferred: give the pod its own IAM role via EKS Pod Identity or IRSA, so it doesn't need IMDS credentials. If it truly needs IMDS (e.g. some agents), set `HttpPutResponseHopLimit=2` on the launch template.

**Prevent →** Keep hop limit 1 by default, block IMDS with a NetworkPolicy for app namespaces, and use per-pod identities.

</details>

### Q189. Pods on the same node talk fine, but cross-node pod traffic fails on a self-managed cluster running Calico or Flannel. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The overlay/underlay protocol is blocked between nodes: VXLAN uses UDP 4789 (Flannel's default VXLAN port is 8472 on Linux), Calico IP-in-IP uses IP protocol 4, Calico BGP needs TCP 179, WireGuard UDP 51820. Security groups or host firewalls often allow only TCP/UDP. On AWS, source/dest check also breaks non-encapsulated routing.

**Diagnose →**
```bash
kubectl get pods -o wide            # pick two pods on different nodes
sudo tcpdump -ni eth0 'udp port 4789 or udp port 8472 or ip proto 4' -c 20   # on the receiving node
kubectl -n kube-system exec <calico-node-pod> -- calicoctl node status       # BGP peers Established?
```
Encapsulated packets leave node A but never arrive at node B = firewall in between.

**Fix →** Allow the overlay protocol between node SGs (self-referencing SG rule for all traffic between nodes is common), or disable source/dest check if routing without encapsulation.

**Prevent →** Codify node-to-node SG rules with the CNI choice and run a cross-node connectivity check (e.g. `cilium connectivity test` or a simple DaemonSet ping mesh).

</details>

### Q190. `kubectl logs` and `kubectl exec` fail with `dial tcp 10.0.3.15:10250: i/o timeout`, while `kubectl get pods` works. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `get` only talks to the API server, but `logs`/`exec`/`port-forward` require the API server to connect to the kubelet on the node (TCP 10250). The node security group or firewall blocks 10250 from the control plane, or the node's kubelet is down.

**Diagnose →**
```bash
kubectl get nodes -o wide
kubectl get --raw /api/v1/nodes/<node>/proxy/healthz
aws ec2 describe-security-groups --group-ids <node-sg> --query 'SecurityGroups[0].IpPermissions[?ToPort==`10250`]'
sudo systemctl status kubelet        # on the node
```

**Fix →** Allow TCP 10250 from the control plane SG (EKS cluster SG) to the nodes, and restart kubelet if it's down.

**Prevent →** Use the managed node group or eksctl default SGs, and don't strip "unused looking" rules from node SGs.

</details>

### Q191. After enabling Istio sidecar injection, a service returns `upstream connect error or disconnect/reset before headers. reset reason: connection failure`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Envoy can't connect to the destination: mTLS mode mismatch (STRICT on the server, client without a sidecar), a `DestinationRule` with the wrong TLS mode, the app listening on `127.0.0.1` only (sidecar forwards to the pod IP), a port named with the wrong protocol prefix, or the app starting before the sidecar is ready.

**Diagnose →**
```bash
istioctl analyze -n myns
istioctl proxy-config endpoints deploy/client -n myns | grep myservice
kubectl logs deploy/myservice -c istio-proxy -n myns | tail
kubectl get peerauthentication,destinationrule -A
```

**Fix →** Align PeerAuthentication and DestinationRule TLS settings (`ISTIO_MUTUAL`), make the app listen on `0.0.0.0`, name ports correctly (`http-`, `grpc-`) or set `appProtocol`, and enable `holdApplicationUntilProxyStarts: true`.

**Prevent →** Roll out mTLS with `PERMISSIVE` first, then `STRICT` per namespace after checking traffic, and run `istioctl analyze` in CI.

</details>

### Q192. On a dual-stack server, `curl -6 https://example.com` fails with `Network is unreachable`, but IPv4 works. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The host has no IPv6 default route. Typical reasons: router advertisements aren't accepted because `net.ipv6.conf.eth0.forwarding=1` while `accept_ra=1` (RA is ignored on forwarding interfaces unless `accept_ra=2`), DHCPv6/RA disabled, or the cloud subnet has no `::/0` route to an IGW/egress-only IGW.

**Diagnose →**
```bash
ip -6 addr show scope global
ip -6 route show default
sysctl net.ipv6.conf.eth0.accept_ra net.ipv6.conf.eth0.forwarding
ping -6 -c2 2606:4700:4700::1111
```

**Fix →** Set `accept_ra=2` on hosts that forward (Docker/Kubernetes nodes), or configure a static IPv6 default route; in AWS add `::/0 → igw` (public) or `::/0 → eigw` (private) to the route table.

**Prevent →** If IPv6 isn't supported end to end, don't publish AAAA records or leave half-configured addresses; apps then try IPv6 first and stall.

</details>

### Q193. IPv6-only pods (or an IPv6-only subnet) can't reach `github.com`, which has no AAAA record. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An IPv6-only client can't talk to an IPv4-only server without translation. You need DNS64 (synthesises AAAA records in `64:ff9b::/96`) plus NAT64 (translates those to IPv4). Without them, resolution returns only A records the client can't use.

**Diagnose →**
```bash
dig +short AAAA github.com              # empty = IPv4-only service
dig +short AAAA github.com @<resolver>  # 64:ff9b::... if DNS64 works
ip -6 route get 64:ff9b::8c52:7904
```

**Fix →** In AWS: enable DNS64 on the subnet (`aws ec2 modify-subnet-attribute --subnet-id <id> --enable-dns64`) and route `64:ff9b::/96` to a NAT gateway. On EKS IPv6 clusters, pods also get an IPv4 egress path via the node (egress-only SNAT by the VPC CNI).

**Prevent →** Inventory IPv4-only dependencies (registries, SaaS APIs) before going IPv6-only, and test them from an IPv6-only subnet first.

</details>

### Q194. After hardening `ip6tables` to "deny all except SSH and HTTPS", IPv6 connectivity works for a few minutes and then dies. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** IPv6 depends on ICMPv6 for Neighbor Discovery (the IPv6 replacement for ARP: neighbor solicitation/advertisement, router advertisement) and for Path MTU Discovery (`Packet Too Big`). Blocking ICMPv6 lets cached neighbors work until they expire, then nothing resolves.

**Diagnose →**
```bash
sudo ip6tables -S INPUT
ip -6 neigh show                 # entries go FAILED / INCOMPLETE
sudo tcpdump -ni eth0 icmp6
```

**Fix →** Allow the essential ICMPv6 types:
```bash
sudo ip6tables -I INPUT -p ipv6-icmp --icmpv6-type neighbour-solicitation -j ACCEPT
sudo ip6tables -I INPUT -p ipv6-icmp --icmpv6-type neighbour-advertisement -j ACCEPT
sudo ip6tables -I INPUT -p ipv6-icmp --icmpv6-type router-advertisement -j ACCEPT
sudo ip6tables -I INPUT -p ipv6-icmp --icmpv6-type packet-too-big -j ACCEPT
```
(Or simply allow all ICMPv6, which most guidance accepts.)

**Prevent →** Base IPv6 firewall policy on RFC 4890 recommendations and keep IPv4/IPv6 rules reviewed together (nftables `inet` family helps).

</details>

### Q195. `tcpdump -i eth0` fails with `tcpdump: eth0: You don't have permission to capture on that device (socket: Operation not permitted)`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Packet capture needs `CAP_NET_RAW` (and `CAP_NET_ADMIN` for promiscuous mode). You're not root, or you're in a container that drops those capabilities.

**Diagnose →**
```bash
id
getcap $(which tcpdump)
capsh --print | grep -i net_raw      # inside a container
```

**Fix →** Run with `sudo`. In containers, use a debug container with the capability rather than adding it to the app:
```bash
docker run --rm -it --net container:myapp --cap-add NET_RAW --cap-add NET_ADMIN nicolaka/netshoot tcpdump -ni eth0
```

**Prevent →** Keep a known debugging image (netshoot) and a documented, audited way to use it, instead of shipping tcpdump in app images.

</details>

### Q196. A capture taken on the sending host shows `cksum 0x1c46 (incorrect -> 0x7a3f)` on every outgoing TCP packet. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Checksum offload. The kernel leaves the TCP/UDP checksum for the NIC to fill in after the packet passes the capture point, so tcpdump/Wireshark sees a placeholder. This is harmless and not the cause of your problem.

**Diagnose →**
```bash
sudo tcpdump -ni eth0 -vv tcp port 443 -c 5 | grep cksum
ethtool -k eth0 | grep -i checksum
```
Captured on the *receiving* side, the same packets have correct checksums.

**Fix →** Nothing to fix. Disable checksum validation in Wireshark (TCP protocol preferences) or capture on the receiver. Only turn off offload temporarily (`ethtool -K eth0 tx off`) if you specifically suspect a NIC/driver bug.

**Prevent →** Know which capture artifacts are normal: incorrect checksums on TX, and huge "packets" larger than MTU caused by TSO/GRO.

</details>

### Q197. You need to capture traffic from a Kubernetes pod whose image has no shell or tcpdump. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Distroless or minimal images don't include tools, and you shouldn't add them. The pod's network namespace is shared by all containers in the pod, so you can attach a tool container to it.

**Diagnose →**
```bash
# ephemeral debug container sharing the pod network namespace
kubectl debug -it pod/myapp-7d9f -n prod --image=nicolaka/netshoot --target=app -- tcpdump -ni eth0 -w /tmp/cap.pcap port 8080
# or from the node
crictl inspect <container-id> | grep -m1 '"pid"'
sudo nsenter -t <pid> -n tcpdump -ni eth0 port 8080 -w /tmp/cap.pcap
```

**Fix →** Copy the capture out for analysis: `kubectl cp prod/myapp-7d9f:/tmp/cap.pcap ./cap.pcap -c <debug-container-name>`, or stream it live: `kubectl debug ... -- tcpdump -ni eth0 -U -w - > cap.pcap`.

**Prevent →** Allow `kubectl debug` through RBAC for on-call engineers only, and document the procedure in the runbook.

</details>

### Q198. On a busy load balancer host, tcpdump ends with `52341 packets dropped by kernel`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** tcpdump couldn't copy packets out of the kernel buffer fast enough: no filter (capturing everything), full packet payloads, name resolution slowing output, or writing to a slow terminal.

**Diagnose →** The summary lines show the ratio of `captured` to `dropped`. Check CPU on the tcpdump process and disk speed if writing a file.

**Fix →**
```bash
sudo tcpdump -ni eth0 -s 128 -B 65536 -w /tmp/lb.pcap 'host 10.0.4.12 and tcp port 443'
```
`-n` skips DNS, a tight BPF filter cuts volume, `-s 128` keeps headers only, `-B` enlarges the buffer (KiB), and `-w` writes binary instead of printing. Rotate with `-C 100 -W 10` for long captures.

**Prevent →** Always capture with a filter and to a file in production; for sustained high-rate analysis use flow logs, eBPF tools, or port mirroring instead.

</details>

### Q199. Requests fail intermittently with a 400 in production only, and you need to see the HTTP payload inside the TLS connection between your service and its upstream. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You can't read TLS 1.3 traffic from a capture with just the server's private key (forward secrecy). You need the per-session secrets from one of the endpoints.

**Diagnose →**
```bash
# 1. make the client log session keys (supported by curl, OpenSSL-based apps that honour it, Firefox/Chrome, Go via tls.Config.KeyLogWriter)
export SSLKEYLOGFILE=/tmp/tls-keys.log
# 2. capture at the same time
sudo tcpdump -ni eth0 -w /tmp/tls.pcap host upstream.example.com and port 443
# 3. decrypt
tshark -r /tmp/tls.pcap -o tls.keylog_file:/tmp/tls-keys.log -Y http -V | less
```
Alternatively, add a debug log of request headers at the client, or route through a mitm proxy in staging.

**Fix →** Once decrypted, you usually find the real cause: a malformed header, a duplicated `Content-Length`, a missing `Host`/SNI, or a body encoding mismatch.

**Prevent →** Treat key log files as secrets: enable briefly, protect, and delete afterwards. Add structured request/response logging (with redaction) at service boundaries so you rarely need decryption.

</details>

### Q200. Pods on one node log `dial tcp 10.96.0.1:443: i/o timeout` when talking to the Kubernetes API, while pods on other nodes are fine. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `10.96.0.1` is the `kubernetes` Service ClusterIP. On that node, Service translation or the path to the control plane is broken: kube-proxy/eBPF dataplane not running or out of sync, the CNI isn't ready, the node's SG/firewall blocks egress to the API endpoint, or conntrack/iptables rules were flushed.

**Diagnose →**
```bash
kubectl get endpoints kubernetes                 # real API server IPs
kubectl -n kube-system get pods -o wide --field-selector spec.nodeName=<node>
# on the node
sudo iptables -t nat -S KUBE-SERVICES | grep 10.96.0.1
curl -k -m 5 https://<apiserver-endpoint-ip>:443/livez
```
If direct `curl` to the endpoint IP works from the node but the ClusterIP doesn't, it's the Service dataplane; if neither works, it's network/SG.

**Fix →** Restart kube-proxy / the CNI agent on the node, restore node SG egress to the control plane, or cordon, drain and replace the node if its state is broken.

**Prevent →** Monitor kube-proxy and CNI agent health per node, use node auto-repair (managed node groups, Karpenter node replacement), and never run manual `iptables -F` on cluster nodes.

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
