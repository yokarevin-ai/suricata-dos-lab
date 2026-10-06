# Network Security Incident Report: DoS Investigation

## Executive Summary
On **October 6, 2026**, a security simulation lab was conducted inside an Oracle VirtualBox environment using an **Ubuntu VM** monitored by a **Suricata Intrusion Detection System (IDS)**. During the monitoring window, a network anomaly was identified, analyzed, and mitigated.

---

## 1. Initial Detection & Log Analysis
The Suricata IDS fired high-frequency alerts indicating potential malicious network activity. The real-time log tracking command used was:
`sudo tail -f /var/log/suricata/fast.log`

### Collected Log Evidence
```text
10/06/2026-09:02:51.980532  [**] [1:2210063:2] SURICATA STREAM 3way handshake excessive different SYNs [**] [Classification: Generic Protocol Command Decode] [Priority: 3] {TCP} 192.168.56.103:19053 -> 192.168.56.102:80
```

### Forensic Analysis Matrix

| Metric | Details |
| :--- | :--- |
| **Identified Threat** | TCP SYN Flood Denial of Service (DoS) |
| **Attacker Source IP** | `192.168.56.103` |
| **Target Victim IP** | `192.168.56.102` (HTTP Web Server Port 80) |
| **Signature ID Triggered** | `1:2210063:2` |

---

## 2. Infrastructure Diagnostics
To evaluate server resource impact and queue conditions, socket layer diagnostics were executed:

```bash
# Executed command to review active socket distributions
ss -s
```

**Observation:** Socket counts showed `4 established` TCP connections during active logging, proving that the operating system was successfully managing table allocation or dropping unauthorized half-open connection attempts.

---

## 3. Mitigation & Containment Action
To protect the server infrastructure and suppress the high-volume alert traffic, a host-based firewall policy was applied to drop all traffic originating from the malicious node:

```bash
sudo iptables -A INPUT -s 192.168.56.103 -j DROP
```

### Verification
Following the application of the `iptables` rule, the live Suricata alert stream immediately stopped, validating that the firewall successfully intercepted and dropped the traffic at the network boundary.
