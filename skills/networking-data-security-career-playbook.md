# Networking and Data Security Career Playbook

This playbook is a practical path to become job-ready for networking, security, and secure infrastructure roles.

---

## Outcome Targets

By the end of this playbook, you should be able to:
- design secure network architecture for small-to-mid systems,
- explain packet flow from L2 to L7 with confidence,
- implement core controls (segmentation, ACLs, VPN, TLS, logging),
- and present real security projects for internship and co-op interviews.

---

## 6-Month Learning Path (Detailed)

## Phase 1 (Weeks 1-4): Networking Foundations

### Learn
- OSI and TCP/IP models
- IPv4/IPv6, subnetting, CIDR
- ARP, ICMP, DNS basics
- VLAN fundamentals

### Build
- Lab with Linux VMs + Packet Tracer/GNS3
- Subnet a small "company" network (engineering, HR, guest)
- Validate with `ping`, `traceroute`, `ip route`, `tcpdump`

### Deliverable
- One-page network diagram + subnet table

---

## Phase 2 (Weeks 5-8): Routing, Switching, and Core Services

### Learn
- Switching, MAC learning, STP concepts
- Inter-VLAN routing, static routing, OSPF basics
- DHCP, DNS, NAT/PAT
- TCP vs UDP behavior and handshake basics

### Build
- Multi-VLAN topology with routed access
- DHCP + DNS working across VLANs
- Capture DNS lookup and TCP 3-way handshake in Wireshark

### Deliverable
- Packet-capture walkthrough (DNS + HTTP/HTTPS request path)

---

## Phase 3 (Weeks 9-12): Security Fundamentals + Cryptography

### Learn
- CIA triad, threat modeling, attack surface
- Symmetric/asymmetric encryption, hashing, signatures
- PKI, certificates, TLS handshake flow

### Build
- Generate keys and certs with OpenSSL
- Inspect certificate chain and TLS negotiation
- Create a threat model for your own lab

### Deliverable
- Threat model document (assets, threats, controls, residual risks)

---

## Phase 4 (Weeks 13-16): Network Security Engineering

### Learn
- Firewalls, ACL strategy, segmentation, least privilege
- IDS/IPS basics (signature + anomaly detection)
- VPN concepts (site-to-site vs remote access)
- Zero trust principles in practical terms

### Build
- Deploy pfSense (or equivalent) in lab
- Configure segmentation and deny-by-default rules
- Add Suricata/Snort and trigger/inspect alerts

### Deliverable
- Security policy matrix (source, destination, protocol, action, reason)

---

## Phase 5 (Weeks 17-20): Data Security Lifecycle

### Learn
- Data classification and sensitivity tiers
- Encryption at rest/in transit/in use
- Key management and secret handling
- Data retention, backup integrity, and DLP concepts

### Build
- Classification model for sample business data
- Control mapping: what is encrypted, who can access, how monitored
- Backup + restore drill with evidence

### Deliverable
- Data security control matrix + recovery runbook

---

## Phase 6 (Weeks 21-24): Career Packaging + Interview Readiness

### Learn
- Security architecture storytelling for interviews
- Incident response communication
- Tradeoff-based decision making (cost, risk, performance)

### Build
- Capstone: secure enterprise mini-architecture
- Simulate attack path and show detection + containment
- Prepare project write-ups and interview pitch

### Deliverable
- Portfolio-ready capstone case study (diagram, controls, logs, lessons)

---

## Weekly Execution Template (Repeat Every Week)

1. 4-5 hours theory (one narrow topic)
2. 4-6 hours labs (implement and break/fix)
3. 2 hours notes and architecture diagrams
4. 1 hour interview-style explanation practice

---

## Tool Stack

- Networking: Wireshark, tcpdump, Packet Tracer/GNS3
- Security: pfSense, Suricata/Snort, OpenSSL, nmap
- Systems: Linux VMs (VirtualBox/VMware), GitHub for documentation

---

## Portfolio Projects (Do These in Order)

1. Segmented Office Network with VLAN + ACL policy
2. TLS and PKI Validation Lab (certificate chain + mTLS demo)
3. SOC Mini-Lab with IDS alerts and incident timeline
4. Data Security Blueprint for a fintech-style app

---

## Certification Layer (Optional but Helpful)

- Start: Cisco CCNA (network depth)
- Then: Security+ (security foundations)
- Later: SC-900 / AZ-500 / AWS Security Specialty (cloud security track)

Certifications support credibility; projects prove capability. Prioritize projects.

---

## Job-Ready Milestones

You are ready to apply when you can:
- explain packet and TLS flow without notes,
- design secure segmentation for a new service,
- read logs/alerts and produce a response timeline,
- and defend architecture choices with tradeoffs.

---

## How We Will Learn One by One

We will go phase-by-phase:
1. concept briefing,
2. guided lab,
3. challenge task,
4. short assessment,
5. then move to next phase.

That creates depth without skipping fundamentals.
