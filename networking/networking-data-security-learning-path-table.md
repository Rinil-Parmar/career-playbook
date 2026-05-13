# Networking and Data Security Learning Path (Phase Table)

This is the quick reference table version of the full playbook.

| Phase | Focus | What you must learn | Hands-on outcome |
|---|---|---|---|
| 0 | Setup & baseline | OSI/TCP-IP overview, Linux CLI basics, IP math basics | Install Wireshark, Packet Tracer/GNS3, VirtualBox, Kali/Ubuntu lab |
| 1 | Networking fundamentals | IPv4/IPv6, subnetting, ARP, ICMP, VLAN basics | Design subnet plan for a small company and validate with ping/traceroute |
| 2 | Switching & routing | MAC table, STP, inter-VLAN routing, static vs dynamic routing (OSPF intro) | Build multi-VLAN + routed network in simulator |
| 3 | Core protocols | DNS, DHCP, NAT/PAT, TCP/UDP internals, HTTP/HTTPS flow | Capture and explain full browser request flow in Wireshark |
| 4 | Network operations | ACLs, firewall basics, QoS basics, monitoring (SNMP/syslog/NetFlow) | Implement ACL + firewall policy and test allow/deny paths |
| 5 | Security foundations | CIA triad, threat modeling, attack surface, identity concepts | Create threat model for your own lab network |
| 6 | Cryptography essentials | Symmetric/asymmetric crypto, hashing, signatures, PKI, TLS | Generate certs, inspect TLS handshake, verify chain/trust |
| 7 | Network security | Segmentation, VPN, IDS/IPS, zero trust basics, secure architecture | Deploy pfSense + Suricata and detect blocked malicious traffic |
| 8 | Data security lifecycle | Data classification, encryption at rest/in transit/in use, key mgmt, DLP | Define data classes + controls matrix (who can access what) |
| 9 | App/API data security | AuthN/AuthZ, OAuth2/OIDC basics, secrets mgmt, SQLi/XSS prevention | Secure a sample API with TLS, token auth, and secret rotation |
| 10 | Cloud & modern infra | VPC/VNet security, security groups, IAM least privilege, KMS | Build secure cloud-like network blueprint and policy set |
| 11 | Governance & resilience | Logging/SIEM, incident response, backup strategy, RPO/RTO, compliance mapping | Write incident runbook + backup/restore drill |
| 12 | Capstone | End-to-end secure enterprise mini-architecture | Final architecture + attack simulation + hardening report |
