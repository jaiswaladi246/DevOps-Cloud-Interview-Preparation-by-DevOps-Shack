# Networking for DevOps — Assignments

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

> 25 practical assignments: 10 🟢 Beginner, 10 🟡 Intermediate, 5 🔴 Advanced. Each has an objective, tasks, acceptance criteria, collapsible hints and a bonus.

[⬅ Back to Networking](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [🟢 Beginner (A1–A10)](#-beginner)
- [🟡 Intermediate (A11–A20)](#-intermediate)
- [🔴 Advanced (A21–A25)](#-advanced)

Most assignments run on any Linux VM or WSL. AWS ones can use the VPC from the [hands-on lab](./hands-on-lab.md). Do them in order within a level.

---

## 🟢 Beginner

### A1. Know your host's network 🟢

**Objective:** Read a Linux host's network configuration without a GUI.

**Tasks:**
1. List interfaces and addresses, and identify the primary IP and its prefix length.
2. Print the routing table and name the default gateway.
3. Find the DNS resolver your host uses.
4. Find the MAC address of the default gateway.

**Expected outcome:** A short text file listing IP/prefix, gateway, resolver and gateway MAC, with the command that produced each.

<details>
<summary>Hints</summary>

`ip -br addr`, `ip route`, `resolvectl status` or `cat /etc/resolv.conf`, `ip neigh show`. Ping the gateway first so it appears in the neighbor table.

</details>

**Bonus:** Explain why `127.0.0.53` appears in `/etc/resolv.conf` on Ubuntu.

### A2. Subnet math 🟢

**Objective:** Do CIDR calculations by hand and verify with a tool.

**Tasks:**
1. For `10.20.30.0/24`, `10.20.30.64/26` and `172.16.0.0/20` write the network, broadcast, first and last host, and host count.
2. Split `10.0.0.0/16` into four equal subnets and list them.
3. Decide whether `10.0.5.9` is inside `10.0.4.0/22`.

**Expected outcome:** All answers correct, checked with `ipcalc` or Python's `ipaddress` module.

<details>
<summary>Hints</summary>

Hosts = 2^(32 - prefix) - 2 (AWS subtracts 5). `python3 -c "import ipaddress as i; print(list(i.ip_network('10.0.0.0/16').subnets(prefixlen_diff=2)))"`.

</details>

**Bonus:** How many usable IPs does an AWS `/28` subnet give you, and why is that too small for an EKS node group?

### A3. Read a DNS answer 🟢

**Objective:** Query DNS and interpret each section of the output.

**Tasks:**
1. Run `dig example.com` and label the question, answer, authority and additional sections.
2. Fetch `A`, `AAAA`, `MX`, `NS` and `TXT` records for a domain you own or a public one.
3. Query the same name against `@1.1.1.1` and `@8.8.8.8` and compare TTLs.
4. Run it twice against your local resolver and watch the TTL count down.

**Expected outcome:** A table of record types with values and TTLs, plus a sentence explaining why the TTL decreased on the second query.

<details>
<summary>Hints</summary>

`dig +short`, `dig +noall +answer`, `dig NS example.com`. A counting-down TTL means you were served from the resolver's cache.

</details>

**Bonus:** Use `dig -x` for a reverse lookup of an IP and explain what a PTR record is for.

### A4. See the phases of an HTTP request 🟢

**Objective:** Use curl to separate DNS, connect, TLS and server time.

**Tasks:**
1. Run `curl -v https://example.com -o /dev/null` and mark where DNS, TCP connect, TLS and the HTTP request/response happen.
2. Use `-w` with `time_namelookup`, `time_connect`, `time_appconnect`, `time_starttransfer` and `time_total`.
3. Compare HTTP/1.1 (`--http1.1`) with HTTP/2 (`--http2`).

**Expected outcome:** A timing breakdown for three sites and a note on which phase dominates each.

<details>
<summary>Hints</summary>

`curl -s -o /dev/null -w 'dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' https://example.com`

</details>

**Bonus:** Write a shell function `curltime URL` that prints this line.

### A5. Listening ports and who owns them 🟢

**Objective:** Map sockets to processes.

**Tasks:**
1. Start `python3 -m http.server 9000` in the background.
2. Find it with `ss -ltnp` and with `lsof -i :9000`.
3. Bind a second server to the same port and capture the exact error.
4. Restart the first on `127.0.0.1` only, and test it from another machine or with the host's non-loopback IP.

**Expected outcome:** You can explain `0.0.0.0` vs `127.0.0.1` vs `::` and show that a loopback-only service is unreachable remotely.

<details>
<summary>Hints</summary>

The error is `OSError: [Errno 98] Address already in use`. `python3 -m http.server 9000 --bind 127.0.0.1`.

</details>

**Bonus:** Use `ss -s` to count sockets by state and name three TCP states you see.

### A6. Test connectivity layer by layer 🟢

**Objective:** Distinguish "can't resolve", "can't route", "closed" and "filtered".

**Tasks:**
1. `ping` a host, then `ping` a name that does not exist and record both failures.
2. Use `nc -vz host port` against an open port, a closed port on a reachable host, and a port dropped by a firewall (use `iptables -A INPUT -p tcp --dport 9001 -j DROP` on a lab VM).
3. Record the exact output and the time taken for each.

**Expected outcome:** A table mapping each failure to its message (`Name or service not known`, `Connection refused`, `timed out`) and what it means.

<details>
<summary>Hints</summary>

Refused is instant (RST). Timeout means a silent drop. Use `-w 3` to cap the wait. Remove your iptables rule afterwards with `-D`.

</details>

**Bonus:** Show the difference with `REJECT` instead of `DROP`.

### A7. Trace a path 🟢

**Objective:** Use traceroute and mtr to see hops and latency.

**Tasks:**
1. Run `traceroute -n` and `mtr -rwn -c 20` to a public host.
2. Explain the `* * *` lines and why they may not mean a problem.
3. Compare the path to two destinations in different regions.

**Expected outcome:** A report with hop count, where latency jumps, and the explanation of ICMP rate limiting on intermediate routers.

<details>
<summary>Hints</summary>

Loss at a middle hop that does not continue to the final hop is usually rate limiting, not real loss. Loss that persists to the end is real.

</details>

**Bonus:** Run `traceroute -T -p 443` and explain why TCP probes sometimes succeed where ICMP fails.

### A8. Edit /etc/hosts and understand resolution order 🟢

**Objective:** See how a name is resolved on Linux.

**Tasks:**
1. Read `/etc/nsswitch.conf` and find the `hosts:` line.
2. Add `127.0.0.1 shop.local` to `/etc/hosts` and test it with `getent hosts shop.local`, `ping` and `curl`.
3. Show that `dig shop.local` does not see it.

**Expected outcome:** You can explain why `getent` and `curl` use `/etc/hosts` but `dig` does not.

<details>
<summary>Hints</summary>

`dig` talks straight to a DNS server. Applications call the libc resolver, which follows `nsswitch.conf` (files, then DNS).

</details>

**Bonus:** Run a local server on port 8000 and reach it as `http://shop.local:8000`.

### A9. Serve a page with nginx 🟢

**Objective:** Install nginx and serve a static site.

**Tasks:**
1. Install nginx, create `/var/www/demo/index.html` and a server block listening on port 8081.
2. Validate with `nginx -t`, reload, and test with curl.
3. Read `access.log` and `error.log` and find your request.
4. Request a missing file and note the status and the log line.

**Expected outcome:** `curl -i localhost:8081` returns `200` with your page and `curl -i localhost:8081/nope` returns `404`, both visible in the logs.

<details>
<summary>Hints</summary>

`root /var/www/demo;` inside `server { listen 8081; }`. Config goes in `/etc/nginx/conf.d/demo.conf`. Reload with `systemctl reload nginx`.

</details>

**Bonus:** Add `add_header X-Served-By demo always;` and confirm it with `curl -I`.

### A10. Draw and explain a public/private subnet design 🟢

**Objective:** Be able to explain a basic VPC on a whiteboard.

**Tasks:**
1. Draw a VPC with one public and one private subnet, IGW, NAT gateway, route tables and the two security groups.
2. Write each route table's rows.
3. Answer: what makes a subnet public? Where must the NAT gateway live? Why can the app not be reached from the internet?

**Expected outcome:** A clear diagram (Mermaid or image) and three correct written answers.

<details>
<summary>Hints</summary>

A route to an IGW makes a subnet public. The NAT gateway sits in a public subnet because it needs the IGW. Private instances have no public IP and no inbound route.

</details>

**Bonus:** Add a second AZ and explain how many NAT gateways you would run and why.

---

## 🟡 Intermediate

### A11. Build the VPC with Terraform 🟡

**Objective:** Codify the lab network.

**Tasks:**
1. Write Terraform for a VPC, two public and two private subnets across two AZs, an IGW, a NAT gateway per AZ and route tables.
2. Use variables for CIDR and AZ count, and outputs for subnet IDs.
3. Run `plan`, `apply`, then `destroy`.

**Expected outcome:** `terraform apply` yields a working network; instances in private subnets reach the internet through their own AZ's NAT.

<details>
<summary>Hints</summary>

Use `cidrsubnet(var.vpc_cidr, 8, count.index)`. Each private route table points to the NAT in the same AZ. `aws_eip` with `domain = "vpc"`.

</details>

**Bonus:** Replace the per-AZ NATs with a single NAT behind a variable toggle for dev, and document the failure trade-off.

### A12. nginx reverse proxy with security headers and rate limiting 🟡

**Objective:** Harden a proxy.

**Tasks:**
1. Proxy `/api/` to a backend on `127.0.0.1:8080` and `/` to static files.
2. Add `limit_req_zone` at 5 requests per second per client IP with a burst of 10.
3. Add `Strict-Transport-Security`, `X-Content-Type-Options` and `X-Frame-Options` headers.
4. Prove rate limiting with a loop of requests and count the `503` responses (or set `limit_req_status 429`).

**Expected outcome:** Normal requests pass; a burst of 50 requests shows rejections, and headers appear on every response.

<details>
<summary>Hints</summary>

`limit_req_zone $binary_remote_addr zone=api:10m rate=5r/s;` in `http {}`, then `limit_req zone=api burst=10 nodelay;` in the location. Test: `for i in $(seq 50); do curl -s -o /dev/null -w '%{http_code}\n' localhost/api/; done | sort | uniq -c`.

</details>

**Bonus:** Behind a load balancer, make nginx rate limit on the real client IP using `real_ip_header X-Forwarded-For` and `set_real_ip_from`.

### A13. TLS with Let's Encrypt and auto-renewal 🟡

**Objective:** Issue and renew a trusted certificate.

**Tasks:**
1. Point a real DNS name at a server and issue a certificate with certbot (`--nginx` or `--webroot`).
2. Confirm the chain with `openssl s_client` and the expiry with `openssl x509 -noout -dates`.
3. Test renewal with `certbot renew --dry-run` and find the renewal timer.
4. Add a deploy hook that reloads nginx.

**Expected outcome:** Browser shows a valid certificate; dry-run renewal succeeds; the timer is active.

<details>
<summary>Hints</summary>

`systemctl list-timers | grep certbot`. Hooks live in `/etc/letsencrypt/renewal-hooks/deploy/`. Rate limits apply, so use `--staging` while experimenting.

</details>

**Bonus:** Get a wildcard certificate using the DNS-01 challenge with the Route 53 plugin.

### A14. Capture and read a TCP session 🟡

**Objective:** Use tcpdump to explain a connection from SYN to FIN.

**Tasks:**
1. Capture `curl http://localhost:8081/` on the loopback with `tcpdump -nn -i lo port 8081`.
2. Identify the handshake, the request, the response, and the teardown flags.
3. Write the capture to a pcap and open it with `tshark -r` or Wireshark.
4. Find the TCP window size and MSS in the SYN.

**Expected outcome:** An annotated capture listing each packet with its meaning.

<details>
<summary>Hints</summary>

`-nn` avoids DNS and port name lookups; `-A` shows ASCII payload; `-S` shows absolute sequence numbers. MSS appears as `mss 65495` on loopback.

</details>

**Bonus:** Capture a refused connection and show the RST packet; capture a firewalled one and show repeated SYN retransmits.

### A15. DNS records and failover in Route 53 🟡

**Objective:** Operate DNS beyond a single A record.

**Tasks:**
1. Create two servers (or two IPs) and a health check against each.
2. Create a failover record pair (primary and secondary) with a 60 second TTL.
3. Stop the primary and measure how long until `dig` returns the secondary.
4. Create a weighted pair 90/10 and verify the distribution by sampling 100 queries.

**Expected outcome:** Failover completes within roughly health-check interval times threshold plus TTL; weighted answers are close to 90/10.

<details>
<summary>Hints</summary>

Health check: HTTP, interval 10 s, failure threshold 3. Add a polling loop: `while true; do date +%T; dig +short app.example.com; sleep 5; done`. Resolver caching extends perceived failover.

</details>

**Bonus:** Add a private hosted zone attached to your VPC that overrides the same name with an internal IP (split-horizon DNS).

### A16. Security groups vs NACLs experiment 🟡

**Objective:** Prove statefulness by experiment.

**Tasks:**
1. In the lab VPC, allow inbound 8080 on a security group only and confirm it works.
2. Add a custom NACL on the subnet allowing inbound 8080 but denying all outbound, and observe the failure.
3. Fix it by allowing outbound ephemeral ports `1024-65535`.
4. Add an explicit NACL deny for your own IP and show that the security group alone cannot do that.

**Expected outcome:** A written explanation, backed by results, of why NACLs need return-traffic rules and security groups do not.

<details>
<summary>Hints</summary>

Security groups are stateful (return traffic allowed automatically). NACLs are stateless and evaluated by rule number, lowest first. Remember the default `*` deny rule.

</details>

**Bonus:** Turn on VPC Flow Logs and find the `REJECT` records for your blocked packets.

### A17. Load balance two backends with health checks 🟡

**Objective:** Run an upstream pool and observe failover.

**Tasks:**
1. Start two backends that return their hostname.
2. Configure an nginx `upstream` with both, `max_fails=2 fail_timeout=10s`.
3. Run `curl` 20 times and confirm round robin; then stop one backend and confirm no errors reach the client.
4. Switch to `least_conn` and `ip_hash` and describe when each fits.

**Expected outcome:** Zero client-visible errors after the first failed attempt, with `proxy_next_upstream` retrying on the other node.

<details>
<summary>Hints</summary>

`proxy_next_upstream error timeout http_502;` only retries idempotent requests by default. Watch `/var/log/nginx/error.log` for `upstream` messages.

</details>

**Bonus:** Replace nginx with HAProxy using `option httpchk GET /healthz` for true active checks.

### A18. Debug an intentionally broken stack 🟡

**Objective:** Apply the layered workflow to unknown faults.

**Tasks:**
1. Have a teammate (or a script) break five things in your lab: a DNS record, a security group rule, an nginx `proxy_pass` port, the certificate's SAN and a stopped service.
2. Without being told which, find each in under five minutes, using only `dig`, `curl -v`, `ss`, `tcpdump` and `openssl`.
3. For each, write symptom, command, cause, fix.

**Expected outcome:** Five documented diagnoses, each ordered resolve, connect, handshake, request.

<details>
<summary>Hints</summary>

Start at DNS and move up. Timeout points to firewall or routing; refused points to a stopped listener; TLS errors point to certificate or SNI; 502 points to the upstream.

</details>

**Bonus:** Write a `netcheck.sh` that runs all these checks for a hostname and prints a pass or fail line per layer.

### A19. Docker and container networking 🟡

**Objective:** Understand bridge networks, port publishing and DNS between containers.

**Tasks:**
1. Run two containers on the default bridge and on a user-defined bridge; show that name resolution works only on the user-defined one.
2. Publish a port with `-p 8080:80` and find the iptables DNAT rule that implements it.
3. Inspect the network with `docker network inspect` and find the container IPs.
4. Run a container with `--network host` and explain the trade-off.

**Expected outcome:** A short write-up with the commands and the iptables output for the DNAT rule.

<details>
<summary>Hints</summary>

`sudo iptables -t nat -L DOCKER -n` shows DNAT rules; `ip link` shows the `docker0` and `br-*` bridges and veth pairs. User-defined networks have an embedded DNS at `127.0.0.11`.

</details>

**Bonus:** Create a network with `--internal` and prove containers on it cannot reach the internet.

### A20. Kubernetes Service and DNS walkthrough 🟡

**Objective:** Trace how a pod reaches another pod by name.

**Tasks:**
1. On a local cluster (kind or minikube) deploy two replicas of nginx behind a ClusterIP Service.
2. From a debug pod run `nslookup web.default.svc.cluster.local` and `curl http://web`.
3. Read `/etc/resolv.conf` in the pod and explain `search` and `ndots:5`.
4. Find the Service's endpoints and the kube-proxy rules that load balance to them.

**Expected outcome:** You can explain every step from name to pod IP, with `kubectl get endpointslices` and the iptables or IPVS output as evidence.

<details>
<summary>Hints</summary>

`kubectl run dbg --rm -it --image=nicolaka/netshoot -- bash`. On the node: `iptables-save | grep web`. CoreDNS config is the `coredns` ConfigMap in `kube-system`.

</details>

**Bonus:** Apply a default-deny NetworkPolicy and then allow only the frontend pods to reach the backend.

---

## 🔴 Advanced

### A21. Multi-VPC connectivity design and build 🔴

**Objective:** Connect networks without overlap and without transitive surprises.

**Tasks:**
1. Create two VPCs with non-overlapping CIDRs and peer them; add routes on both sides and security group rules referencing the peer CIDR.
2. Add a third VPC peered only to the second and prove that VPC 1 cannot reach VPC 3.
3. Replace the three peerings with a Transit Gateway and re-test.
4. Write a one-page comparison of peering, Transit Gateway and PrivateLink covering transitivity, cost, scale and use cases.

**Expected outcome:** Working reachability matrix before and after, and a comparison that justifies when you would choose each.

<details>
<summary>Hints</summary>

VPC peering is non-transitive. Both route tables need routes to the peer CIDR. Transit Gateway needs attachments, route table associations and propagations. Use `aws ec2 describe-route-tables` to check for missing routes.

</details>

**Bonus:** Expose a service from VPC 2 to VPC 1 using an NLB and PrivateLink, with no routes between the VPCs.

### A22. Diagnose and fix MTU and conntrack problems 🔴

**Objective:** Reproduce two subtle, production-grade network faults.

**Tasks:**
1. Lower the MTU of one interface on a lab VM (`ip link set dev eth0 mtu 1200`) behind a path that blocks ICMP, and show that small requests work while large downloads hang. Prove it with `ping -M do -s 1400 host`.
2. Fix it with correct MTU or MSS clamping (`iptables -t mangle ... --clamp-mss-to-pmtu`), and re-test.
3. On another VM set `net.netfilter.nf_conntrack_max` very low, generate load with `ab` or `wrk`, and capture the `nf_conntrack: table full, dropping packet` message.
4. Raise the limit sensibly and show the current usage with `conntrack -C`.

**Expected outcome:** Both faults reproduced, diagnosed with the right command and fixed, with a note on how you would alert on each in production.

<details>
<summary>Hints</summary>

`tracepath host` reveals the path MTU. The symptom of PMTUD failure: handshake works, big responses stall. Watch `dmesg -w` and `/proc/sys/net/netfilter/nf_conntrack_count`. Do not leave the low limit in place.

</details>

**Bonus:** Show ephemeral port exhaustion by making many connections to a single destination from one source IP and tuning `net.ipv4.ip_local_port_range`.

### A23. Build a NAT gateway and router by hand on Linux 🔴

**Objective:** Understand what a NAT gateway does by building one.

**Tasks:**
1. Use three VMs (or network namespaces): client (private), router (two interfaces), server (public).
2. Enable forwarding with `sysctl net.ipv4.ip_forward=1` and add `iptables -t nat -A POSTROUTING -o <public-if> -j MASQUERADE`.
3. Prove the server sees the router's address and not the client's, and inspect the translation with `conntrack -L`.
4. Add DNAT port forwarding from the router's port 8080 to a service on the client.
5. Disable forwarding and show exactly how the failure looks in `tcpdump`.

**Expected outcome:** Working SNAT and DNAT, with captures proving addresses change at the router.

<details>
<summary>Hints</summary>

With namespaces: `ip netns add`, `ip link add veth0 type veth peer name veth1`, `ip link set veth1 netns client`. Remember the default route in the client namespace and `rp_filter` settings if replies disappear.

</details>

**Bonus:** Rewrite the rules with `nft` and compare readability.

### A24. Zero-downtime TLS and certificate lifecycle 🔴

**Objective:** Run certificates like production, with automation, monitoring and mTLS.

**Tasks:**
1. Create a private CA with `openssl` (or `step-ca`), issue a server certificate with SAN and a client certificate.
2. Configure nginx for mutual TLS: `ssl_client_certificate`, `ssl_verify_client on`.
3. Show a request with the client certificate succeeding and without it failing (`400 No required SSL certificate was sent`).
4. Write a script or Prometheus blackbox probe that alerts when any certificate has less than 14 days left.
5. Rotate the server certificate with `nginx -s reload` while a load generator runs, and prove zero failed requests.

**Expected outcome:** mTLS working, alert rule written, and a reload under load showing 0 errors.

<details>
<summary>Hints</summary>

`curl --cert client.crt --key client.key --cacert ca.crt https://host`. Expiry check: `openssl x509 -checkend 1209600 -noout`. blackbox_exporter exposes `probe_ssl_earliest_cert_expiry`. Reload is graceful; restart is not.

</details>

**Bonus:** Use cert-manager with a self-signed ClusterIssuer in Kubernetes and watch automatic renewal by shortening `duration` and `renewBefore`.

### A25. Network design review: three-tier production architecture 🔴

**Objective:** Produce a design you can defend in a system design interview.

**Tasks:**
1. Design a three-tier app (web, app, database) across three AZs: CIDR plan with room to grow, subnet tiers, route tables, NAT strategy, security group chain, NACL stance.
2. Add an ALB with ACM certificate and WAF, private connectivity to AWS services with VPC endpoints, and a VPN or Direct Connect path for on-prem.
3. Add observability: VPC Flow Logs, ALB access logs, DNS query logs, and the alerts you would set.
4. Estimate monthly NAT and cross-AZ data transfer cost for 5 TB outbound and explain how endpoints reduce it.
5. List five failure modes (AZ loss, NAT failure, DNS outage, expired certificate, route table mistake) with detection and mitigation for each.

**Expected outcome:** A Mermaid diagram plus a two-page write-up with the numbers and trade-offs, reviewed by a peer.

<details>
<summary>Hints</summary>

Use a `/16` per VPC with `/20` subnets so tiers can scale. Reference security groups, not CIDRs. NAT cost is the hourly charge per gateway plus a per-GB processing fee, so S3 and DynamoDB gateway endpoints (free) cut the biggest share. Check current pricing on the AWS pricing page.

</details>

**Bonus:** Describe how the design changes for multi-region active-passive, including DNS failover and data replication.

---

[⬅ Back to Networking](./README.md) · [🏠 Home](../README.md)

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
