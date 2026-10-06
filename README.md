# 🛡️ Enterprise IDS/IPS Deployment & Automated DoS Mitigation Lab

## 📌 Executive Summary
This project demonstrates a production-grade Network Security architecture deploying **Suricata** as an inline **IDS/IPS (Intrusion Detection & Prevention System)** to defend an Ubuntu server against active network infrastructure attacks. 

The lab lifecycle covers advanced attacker reconnaissance, live volumetric **TCP SYN Flooding**, firewall tuning via **Netfilter (`iptables`)**, and custom signature engineering. The final phase demonstrates an evolution from passive alerting to active inline remediation—effectively dropping subsequent repeated attacks, protecting system sockets, and keeping the target web application completely online.

---

## 🏗️ Technical Lab Topology & Architecture
```text
  [ Attacker Host: Kali Linux ]            [ Target Victim: Ubuntu Server ]
   Interface: eth0                          Interface: enp0s3
   IP: 192.168.56.103                       IP: 192.168.56.102
           │                                        │
           └───────────────► [ Host-Only ] ◄────────┘
                             Network Matrix
                                   │
                           (Promiscuous Mode)
                                   ▼
                     ┌───────────────────────────┐
                     │ Suricata Security Engine  │
                     │  Mode: Passive -> Inline  │
                     └───────────────────────────┘
```

---

## 🗺️ 1. Phase 1: Adversarial Reconnaissance & Scanning
Before launching the denial of service simulation, aggressive network footprinting was conducted from the Kali Linux host using **Nmap** to detect exposed target applications and fingerprint the operating system layer.

```bash
nmap -A 192.168.56.102
```

### 📋 Discovered Host Surface Telemetry
* **Target State:** Host is up (0.0064s latency).
* **Open Port:** `80/tcp` (HTTP)
* **Service Version:** Apache httpd `2.4.66` (Ubuntu default landing page).
* **Hardware MAC Address:** `08:00:27:34:BD:D7` (Oracle VirtualBox virtual NIC).

---

## ⚔️ 2. Phase 2: Volumetric DoS Attack Simulation
Using the footprinting intelligence, a high-frequency **TCP SYN Flood / Invalid ACK attack** was initiated from the Kali machine to exhaust target resources and force web application dropouts.

```bash
sudo hping3 -S --flood -p 80 192.168.56.102
```

### 📋 Live Traffic Output Statistics
```text
--- 192.168.56.102 hping statistic ---
3601571 packets transmitted, 0 packets received, 100% packet loss
round-trip min/avg/max = 0.0/0.0/0.0 ms
```
* **Analysis:** High-frequency, unacknowledged packets flooded interface `enp0s3`. Initial real-time logging captures (`tail -f /var/log/suricata/fast.log`) caught the behavior under signature handles `1:2210046:2` and `1:2210063:2`.

---

## 🛠️ 3. Phase 3: Infrastructure Diagnostics
To isolate system network interfaces and evaluate the exact layer impact of the live volumetric flood, system diagnostics were executed.

### Network Interface Verification (`ip -br addr`)
```bash
ip -br addr
```
* **`lo` (Loopback):** Local host testing matrix (`127.0.0.1/8`).
* **`enp0s3` / `eth0`:** Active Host-Only infrastructure framework nodes (`192.168.56.102` and `192.168.56.103`), which handled the execution of the attack matrix.

### Active Socket Layer Diagnostics (`ss -s`)
```bash
ss -s
```
**Diagnostic Observation:** Socket diagnostics registered exactly **4 established** TCP connections during peak attack waves. This verifies that the underlying Linux Kernel safely throttled unauthorized half-open connection attempts without filling up or overflowing the operating system backlog queues.

---

## 🛡️ 4. Phase 4: Defensive Configuration & Active IPS Containment
To transition the deployment from passive alert notifications to real-time defensive remediation, the Suricata architecture was bound inline using the Linux netfilter framework.

### Step 1: Netfilter Engine Interception Rules (`iptables`)
The host firewall routing rules were completely flushed and updated to forward raw untrusted interface traffic through Suricata's internal inspection queue:

```bash
# Flush legacy runtime rules and custom user chains
sudo iptables -F
sudo iptables -X

# Intercept and pipeline incoming/forwarding traffic into NFQUEUE 0
sudo iptables -I INPUT -j NFQUEUE --queue-num 0
sudo iptables -I FORWARD -j NFQUEUE --queue-num 0

# Whitelist statefully verified connections to maintain host performance
sudo iptables -I INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -I OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### Step 2: Inline Mode Activation (`-q 0`)
Suricata was re-initialized to pull natively from Netfilter Queue `0`. Running processes were audited to verify inline enforcement:
```bash
ps aux | grep suricata
```
**Process Layout Result:**
```text
root  6973 102 4.8 210784 16964 ? Rs 08:40 0:22 suricata -c /etc/suricata/suricata.yaml -q 0 -D --pidfile /run/suricata.pid
```

---

## 🔄 5. Phase 5: Testing Drop Rules & Repeat Attack Validation
With the inline architecture active, **custom drop rules** were implemented, and the attacks were repeated from the Kali machine to validate mitigation efficacy.

### Custom Drop Rule Configuration (`/etc/suricata/rules/local.rules`)
Signatures were evolved from traditional passive `alert` frameworks into authoritative dropping frameworks:
```text
drop tcp $EXTERNAL_NET any -> 192.168.56.102 80 (msg:"IPS ENFORCED: AUTOMATIC SYN FLOOD DROP"; flags:S; threshold: type threshold, track by_src, count 100, seconds 1; classtype:attempted-dos; sid:1000001; rev:2;)
```

### Repeat Attack Verification
When the `hping3 --flood` sequences were re-run from Kali Linux, the defensive engine actively intervened:
1. **Detection Blocked at Firewall:** Suricata captured packets inside `NFQUEUE 0` and dropped them instantly.
2. **Zero Resource Drain:** The malicious requests were dropped before ever touching the web server applications, neutralizing CPU spikes and avoiding memory resource fatigue.
3. **Application Preservation:** The Apache web server remained completely accessible and responsive to legitimate traffic during subsequent attack waves.

---

## 🚀 6. Advanced Proactive Hardening (Extended Scope)
*The following industrial-grade configurations protect this architecture long-term against structural resource depletion:*

### A. Persistent Netfilter Controls
To prevent runtime rule losses during system updates or hardware reboots, the network filtering maps were hardcoded into a permanent schema using the `iptables-persistent` automation controller:
```bash
sudo apt-get install iptables-persistent -y
sudo netfilter-persist save
```

### B. Automated Log Management & Rotation
To protect the server disk space against massive log expansions during volumetric DoS actions, a tailored log rotation routine was injected into `/etc/logrotate.d/suricata`:
```text
/var/log/suricata/*.log /var/log/suricata/*.json {
    daily
    rotate 7
    missingok
    compress
    delaycompress
    sharedscripts
    postrotate
        /bin/kill -HUP `cat /var/run/suricata.pid 2>/dev/null` 2> /dev/null || true
    endscript
}
```

---

## 📁 7. Repository Structure
```text
├── README.md                 # Complete incident investigation and deployment blueprint
├── rules/
│   └── local.rules           # Exported custom Suricata drop rulesets
├── config/
│   ├── suricata.yaml         # Optimized engine runtime configurations
│   └── iptables.rules        # Persistent Netfilter firewall routing schema
└── images/
    ├── network_topology.png   # Architectural engineering layout blueprint
    └── fast_log_alerts.png   # Forensics validation capture from fast.log
```
