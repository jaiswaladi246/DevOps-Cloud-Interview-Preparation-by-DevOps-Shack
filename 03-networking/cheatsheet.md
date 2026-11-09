# Networking for DevOps — Cheatsheet

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

> Dense command and config reference for debugging and building networks. Grouped by task.

[⬅ Back to Networking](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Host networking (ip, ss)](#host-networking-ip-ss)
- [Connectivity testing](#connectivity-testing)
- [DNS (dig)](#dns-dig)
- [curl and HTTP](#curl-and-http)
- [TLS and openssl](#tls-and-openssl)
- [tcpdump](#tcpdump)
- [Firewall and NAT](#firewall-and-nat)
- [nginx reverse proxy](#nginx-reverse-proxy)
- [AWS VPC](#aws-vpc)
- [Kubernetes networking](#kubernetes-networking)
- [Reference tables](#reference-tables)
- [Top 15 interview one-liners](#top-15-interview-one-liners)

---

## Host networking (ip, ss)

| Task | Command |
|------|---------|
| Interfaces and IPs (brief) | `ip -br addr` |
| Link state, MTU | `ip -br link` / `ip link show eth0` |
| Routing table | `ip route` |
| Which route is used for a destination | `ip route get 8.8.8.8` |
| Neighbor (ARP) table | `ip neigh` |
| Add a static route | `sudo ip route add 10.5.0.0/16 via 10.0.1.1` |
| Add a temporary IP | `sudo ip addr add 10.0.1.50/24 dev eth0` |
| Set MTU | `sudo ip link set dev eth0 mtu 1400` |
| Listening TCP/UDP with process | `sudo ss -tulpn` |
| Established connections | `ss -tn state established` |
| Socket summary | `ss -s` |
| Connections to a port | `ss -tn dst :443` |
| Count sockets by state | `ss -tan \| awk 'NR>1{print $1}' \| sort \| uniq -c` |
| Who has a port | `sudo lsof -i :8080` |
| Bandwidth per interface | `ip -s link show eth0` |
| Resolver in use (systemd) | `resolvectl status` |

## Connectivity testing

| Task | Command |
|------|---------|
| ICMP reachability | `ping -c 4 host` |
| Path MTU probe (don't fragment) | `ping -M do -s 1472 host` |
| TCP port open? | `nc -vz -w 3 host 443` |
| UDP port check | `nc -vzu -w 3 host 53` |
| Path with latency per hop | `mtr -rwn -c 20 host` |
| Traceroute without DNS | `traceroute -n host` |
| Traceroute over TCP | `sudo traceroute -T -p 443 host` |
| Discover path MTU | `tracepath host` |
| Raw TCP test, no tools | `timeout 3 bash -c '</dev/tcp/host/443' && echo open` |
| Scan a host's ports | `nmap -Pn -p 22,80,443 host` |
| Name resolution as apps see it | `getent hosts name` |

How to read the result: instant `Connection refused` means the host answered with RST (port closed, path fine). A hang until timeout means packets are dropped (firewall, security group, route). `No route to host` or `Network is unreachable` means the local routing table or ARP failed.

## DNS (dig)

| Task | Command |
|------|---------|
| Basic lookup | `dig example.com` |
| Short answer | `dig +short example.com` |
| Answer section only | `dig +noall +answer example.com` |
| Specific type | `dig MX example.com` / `dig TXT example.com` / `dig AAAA example.com` |
| Ask a specific server | `dig @1.1.1.1 example.com` |
| Walk from the root | `dig +trace example.com` |
| Reverse lookup | `dig -x 8.8.8.8 +short` |
| Authoritative servers | `dig NS example.com +short` |
| Query authoritative directly | `dig @ns1.example.com example.com +norecurse` |
| SOA (serial, negative TTL) | `dig SOA example.com +short` |
| Over TCP | `dig +tcp example.com` |
| DNSSEC data | `dig +dnssec example.com` |
| Show query time and server | `dig example.com \| grep -E "Query time\|SERVER"` |
| Kubernetes in-cluster test | `kubectl exec -it pod -- nslookup kubernetes.default` |

Status codes in the header: `NOERROR` (ok, may be empty), `NXDOMAIN` (name does not exist), `SERVFAIL` (resolver or upstream failure, often DNSSEC or broken delegation), `REFUSED` (server will not answer you).

## curl and HTTP

| Task | Command |
|------|---------|
| Verbose (DNS, TLS, headers) | `curl -v https://host/` |
| Headers only | `curl -I https://host/` |
| Silent, status code only | `curl -s -o /dev/null -w '%{http_code}\n' https://host/` |
| Timing breakdown | `curl -s -o /dev/null -w 'dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' URL` |
| Force an IP (bypass DNS) | `curl --resolve app.example.com:443:203.0.113.10 https://app.example.com/` |
| Send a Host header | `curl -H 'Host: app.example.com' http://203.0.113.10/` |
| Follow redirects | `curl -L URL` |
| Ignore cert errors (debug only) | `curl -k URL` |
| Use a CA file | `curl --cacert ca.crt URL` |
| Client certificate (mTLS) | `curl --cert c.crt --key c.key --cacert ca.crt URL` |
| Force HTTP version | `curl --http1.1 URL` / `curl --http2 URL` |
| Set timeouts | `curl --connect-timeout 3 -m 10 URL` |
| POST JSON | `curl -X POST -H 'Content-Type: application/json' -d '{"a":1}' URL` |
| Retry on failure | `curl --retry 3 --retry-connrefused URL` |

HTTP status groups: `2xx` success, `3xx` redirect, `4xx` client error, `5xx` server error. For proxies: `502` bad or no answer from upstream, `503` no healthy upstream or rate limited, `504` upstream timeout.

## TLS and openssl

| Task | Command |
|------|---------|
| Inspect a live certificate | `openssl s_client -connect host:443 -servername host </dev/null \| openssl x509 -noout -subject -issuer -dates -ext subjectAltName` |
| Show the chain | `openssl s_client -connect host:443 -servername host -showcerts </dev/null` |
| Test a protocol version | `openssl s_client -connect host:443 -tls1_2` / `-tls1_3` |
| Verify hostname | `openssl s_client -connect host:443 -servername host -verify_hostname host </dev/null` |
| Read a cert file | `openssl x509 -in cert.pem -noout -text` |
| Fingerprint | `openssl x509 -in cert.pem -noout -fingerprint -sha256` |
| Expires within 14 days? | `openssl x509 -in cert.pem -noout -checkend 1209600` |
| Self-signed cert with SAN | `openssl req -x509 -newkey rsa:2048 -nodes -days 90 -keyout k.pem -out c.pem -subj "/CN=host" -addext "subjectAltName=DNS:host"` |
| Generate key and CSR | `openssl req -new -newkey rsa:2048 -nodes -keyout k.pem -out req.csr -subj "/CN=host"` |
| Cert matches key? | `diff <(openssl x509 -in c.pem -noout -pubkey) <(openssl pkey -in k.pem -pubout)` |
| Verify against a CA | `openssl verify -CAfile ca.pem cert.pem` |
| Let's Encrypt issue | `sudo certbot --nginx -d host -m you@example.com --agree-tos -n` |
| Test renewal | `sudo certbot renew --dry-run` |
| List certbot certs | `sudo certbot certificates` |

Common verify codes from `s_client`: `0 ok`, `10 certificate has expired`, `18 self-signed certificate`, `20 unable to get local issuer certificate` (missing intermediate or unknown CA), `62 hostname mismatch`.

## tcpdump

| Task | Command |
|------|---------|
| Basic, no name lookups | `sudo tcpdump -nn -i any` |
| One host and port | `sudo tcpdump -nn -i any host 10.0.2.10 and tcp port 8080` |
| Only SYN packets | `sudo tcpdump -nn -i any 'tcp[tcpflags] & (tcp-syn) != 0'` |
| Only RST packets | `sudo tcpdump -nn -i any 'tcp[tcpflags] & (tcp-rst) != 0'` |
| DNS traffic | `sudo tcpdump -nn -i any port 53` |
| Show payload as text | `sudo tcpdump -nn -A -i any port 8080` |
| Save to file | `sudo tcpdump -nn -i any -w cap.pcap -c 1000 port 443` |
| Read a file | `tcpdump -nn -r cap.pcap` |
| Rotate captures | `sudo tcpdump -i any -w cap-%H%M.pcap -G 300 -W 12` |
| Exclude your own SSH | `sudo tcpdump -nn -i any 'not port 22'` |
| ICMP | `sudo tcpdump -nn -i any icmp` |

Flag letters: `S` SYN, `.` ACK, `P` PSH, `F` FIN, `R` RST. A healthy open is `[S]`, `[S.]`, `[.]`. Repeated `[S]` with no reply means dropped. `[S]` answered by `[R.]` means refused.

## Firewall and NAT

| Task | Command |
|------|---------|
| List filter rules | `sudo iptables -L -n -v --line-numbers` |
| List NAT rules | `sudo iptables -t nat -L -n -v` |
| Allow a port | `sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT` |
| Drop silently | `sudo iptables -A INPUT -p tcp --dport 9001 -j DROP` |
| Reject with RST | `sudo iptables -A INPUT -p tcp --dport 9001 -j REJECT --reject-with tcp-reset` |
| Delete rule by number | `sudo iptables -D INPUT 3` |
| Enable forwarding | `sudo sysctl -w net.ipv4.ip_forward=1` |
| Masquerade (SNAT) | `sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE` |
| Port forward (DNAT) | `sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.2.10:8080` |
| MSS clamp | `sudo iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu` |
| nftables ruleset | `sudo nft list ruleset` |
| ufw status | `sudo ufw status verbose` |
| Conntrack entries | `sudo conntrack -L` / `sudo conntrack -C` |
| Conntrack limit | `sysctl net.netfilter.nf_conntrack_max` |

## nginx reverse proxy

Minimal TLS reverse proxy:

```nginx
upstream app {
    server 10.0.2.10:8080 max_fails=3 fail_timeout=10s;
    keepalive 16;
}

server {
    listen 80;
    server_name app.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    http2 on;                      # nginx 1.25.1+; older: "listen 443 ssl http2;"
    server_name app.example.com;
    ssl_certificate     /etc/nginx/tls/app.crt;
    ssl_certificate_key /etc/nginx/tls/app.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://app;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 3s;
        proxy_read_timeout 30s;
    }
}
```

| Task | Command or directive |
|------|----------------------|
| Test config | `sudo nginx -t` |
| Graceful reload | `sudo nginx -s reload` or `systemctl reload nginx` |
| Dump effective config | `sudo nginx -T` |
| Error log | `sudo tail -f /var/log/nginx/error.log` |
| Rate limit | `limit_req_zone $binary_remote_addr zone=api:10m rate=5r/s;` then `limit_req zone=api burst=10 nodelay;` |
| Websocket | `proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade";` |
| Big uploads | `client_max_body_size 50m;` |
| Real client IP behind an LB | `set_real_ip_from 10.0.0.0/16; real_ip_header X-Forwarded-For;` |
| Balance by least connections | `least_conn;` inside `upstream` |

Log signatures: `connect() failed (111: Connection refused)` upstream down. `upstream timed out (110: Connection timed out)` upstream slow or firewalled. `no live upstreams` all backends marked failed. `SSL_do_handshake() failed` client or protocol mismatch.

## AWS VPC

| Task | Command |
|------|---------|
| Create VPC | `aws ec2 create-vpc --cidr-block 10.0.0.0/16` |
| Create subnet | `aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a` |
| Create and attach IGW | `aws ec2 create-internet-gateway` then `aws ec2 attach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC` |
| Default route to IGW | `aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW` |
| NAT gateway | `aws ec2 create-nat-gateway --subnet-id $PUB --allocation-id $EIP` |
| Default route to NAT | `aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT` |
| SG rule from another SG | `aws ec2 authorize-security-group-ingress --group-id $SG --protocol tcp --port 8080 --source-group $SG2` |
| Show route tables | `aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPC` |
| Show security group | `aws ec2 describe-security-groups --group-ids $SG` |
| Find ENI for an IP | `aws ec2 describe-network-interfaces --filters Name=addresses.private-ip-address,Values=10.0.2.10` |
| Gateway endpoint for S3 | `aws ec2 create-vpc-endpoint --vpc-id $VPC --service-name com.amazonaws.ap-south-1.s3 --route-table-ids $RT` |
| Enable flow logs | `aws ec2 create-flow-logs --resource-type VPC --resource-ids $VPC --traffic-type ALL --log-destination-type cloud-watch-logs --log-group-name vpc-flow --deliver-logs-permission-arn $ROLE_ARN` |
| Reachability Analyzer | `aws ec2 create-network-insights-path ...` then `start-network-insights-analysis` |
| Instance metadata (IMDSv2) | `TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token -H 'X-aws-ec2-metadata-token-ttl-seconds: 60'); curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4` |

AWS facts: 5 reserved IPs per subnet; VPC resolver at base + 2; `169.254.169.254` is metadata, `169.254.169.253` is the Amazon DNS alternative; security groups are stateful, NACLs are stateless; NAT gateway is zonal; VPC peering is not transitive.

## Kubernetes networking

| Task | Command |
|------|---------|
| Services and ClusterIPs | `kubectl get svc -A` |
| Endpoints behind a Service | `kubectl get endpointslices -l kubernetes.io/service-name=web` |
| Debug pod with tools | `kubectl run dbg --rm -it --image=nicolaka/netshoot -- bash` |
| DNS from a pod | `kubectl exec -it pod -- nslookup web.default.svc.cluster.local` |
| Pod resolver config | `kubectl exec pod -- cat /etc/resolv.conf` |
| CoreDNS logs | `kubectl -n kube-system logs -l k8s-app=kube-dns` |
| CoreDNS config | `kubectl -n kube-system get cm coredns -o yaml` |
| Port forward | `kubectl port-forward svc/web 8080:80` |
| Ingress status | `kubectl describe ingress web` |
| Network policies | `kubectl get networkpolicy -A` |
| kube-proxy rules for a Service | `sudo iptables-save \| grep web` |
| Pod IP and node | `kubectl get pod -o wide` |

Naming: `<svc>.<namespace>.svc.cluster.local`. `ndots:5` means names with fewer than 5 dots try the search domains first, which multiplies DNS queries for external names; use a trailing dot or lower `ndots` for hot paths.

## Reference tables

Common ports:

| Port | Service | Port | Service |
|------|---------|------|---------|
| 22 | SSH | 443 | HTTPS |
| 25 | SMTP | 465 / 587 | SMTPS / submission |
| 53 | DNS (UDP and TCP) | 3306 | MySQL |
| 80 | HTTP | 5432 | PostgreSQL |
| 123 | NTP (UDP) | 6379 | Redis |
| 179 | BGP | 6443 | Kubernetes API |
| 389 / 636 | LDAP / LDAPS | 10250 | kubelet |
| 2049 | NFS | 51820 | WireGuard (UDP) |

CIDR quick table:

| Prefix | Addresses | AWS usable | Prefix | Addresses | AWS usable |
|--------|-----------|------------|--------|-----------|------------|
| /16 | 65,536 | 65,531 | /24 | 256 | 251 |
| /20 | 4,096 | 4,091 | /26 | 64 | 59 |
| /22 | 1,024 | 1,019 | /28 | 16 | 11 |

Private ranges (RFC 1918): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. Link-local: `169.254.0.0/16`. Carrier-grade NAT: `100.64.0.0/10`. Documentation (safe for examples): `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`.

DNS record types: `A` IPv4, `AAAA` IPv6, `CNAME` alias (not at apex), `MX` mail, `TXT` text (SPF, verification), `NS` delegation, `SOA` zone authority, `SRV` service location, `PTR` reverse, `CAA` allowed CAs.

## Top 15 interview one-liners

1. **What happens when you type a URL?** DNS resolves the name, TCP handshakes, TLS negotiates (with SNI), the browser sends an HTTP request, the server responds, and connections may be reused.
2. **TCP vs UDP?** TCP is connection-oriented with ordering, retransmission and congestion control; UDP is connectionless and cheap, so DNS, QUIC and real-time media use it.
3. **Connection refused vs timed out?** Refused means the host replied with RST (path works, nothing listening); timed out means packets were silently dropped (firewall, route, or host down).
4. **What makes a subnet public in AWS?** A route to an internet gateway in its route table, plus public IPs on the instances.
5. **Security group vs NACL?** Security groups are stateful, instance-level, allow-only; NACLs are stateless, subnet-level, ordered, with allow and deny.
6. **What does a NAT gateway do?** Lets private instances start outbound connections by translating their source IP to the gateway's public IP while blocking inbound-initiated traffic.
7. **L4 vs L7 load balancer?** L4 balances TCP/UDP flows fast without reading content; L7 parses HTTP and can route by host, path or header, and terminate TLS.
8. **502 vs 503 vs 504?** 502 the upstream returned garbage or refused; 503 no healthy upstream or overload; 504 the upstream took too long.
9. **How does DNS caching affect a migration?** Resolvers cache for the TTL, so lower the TTL a day or more before the change and raise it after.
10. **CNAME at the zone apex?** Not allowed by the DNS spec because the apex needs SOA and NS; use an ALIAS or ANAME style record (Route 53 alias).
11. **What is SNI?** A TLS extension where the client sends the hostname in the ClientHello so one IP can serve many certificates.
12. **Why are big downloads hanging but small requests fine?** Suspect MTU or PMTUD failure: ICMP "fragmentation needed" is blocked; fix with MSS clamping or correct MTU.
13. **How do pods reach a Service?** The Service's ClusterIP is a virtual address; kube-proxy (iptables, IPVS) or eBPF rewrites it to a ready pod IP from the endpoint slices, and CoreDNS provides the name.
14. **Why VPC endpoints?** Private access to AWS services without NAT or the internet, which cuts NAT data charges and shrinks the attack surface.
15. **First three commands for an unreachable service?** `dig +short name` (resolves?), `nc -vz host port` or `curl -v` (connects?), `ss -ltnp` on the server (listening?), then `tcpdump` to see whether packets arrive.

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
