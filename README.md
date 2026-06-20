# Active_Response_Wazuh_SocHomeLab
SOC home lab: detected a live SSH brute-force attack with Wazuh SIEM, mapped the full MITRE ATT&amp;CK kill chain, and built automated firewall containment via Active Response.

# 🛡️ SOC Home Lab — SSH Brute Force Detection & Automated Response with Wazuh SIEM

> A full blue team / red team home lab: deploying a SIEM, attacking my own infrastructure with a real SSH brute-force tool, detecting and threat-hunting the full MITRE ATT&CK kill chain, then closing the loop with automated containment via Wazuh Active Response.

![Status](https://img.shields.io/badge/status-complete-brightgreen)
![Wazuh](https://img.shields.io/badge/Wazuh-v4.14.5-1A93C2)
![Kali](https://img.shields.io/badge/Kali-Linux-557C94)
![MITRE](https://img.shields.io/badge/MITRE-ATT%26CK-red)
![Active Response](https://img.shields.io/badge/Active%20Response-firewall--drop-orange)

---

## 📌 Overview

This project documents an end-to-end SOC analyst workflow built entirely in a self-owned, isolated home lab:

1. Deployed **Wazuh SIEM** and enrolled an Ubuntu endpoint agent
2. Performed reconnaissance and a **live SSH brute-force attack** from Kali Linux using **Hydra**
3. Detected the attack in Wazuh — from individual failed-login alerts to the single composite alert proving compromise
4. Threat-hunted the full kill chain and mapped it to **MITRE ATT&CK**
5. Configured **Wazuh Active Response** to automatically contain the attacker via `firewall-drop`, then re-ran the attack to validate the block
6. Cross-referenced the critical alert against **5 compliance frameworks** (GDPR, NIST 800-53, PCI-DSS, HIPAA, GPG13)

> ⚠️ **Legal & Ethical Note:** All testing was performed exclusively within an isolated VirtualBox lab I own and control. Unauthorized scanning or brute-forcing of systems you do not own or have explicit written permission to test is illegal.

---

## 🏗️ Lab Architecture

| Component | Role | OS / Version | IP Address |
|---|---|---|---|
| **Attacker** | Offensive ops (Hydra, nmap, arp-scan) | Kali Linux | `192.168.1.4` |
| **Target** | Victim endpoint w/ Wazuh agent | Ubuntu 26.04 LTS (OpenSSH 10.2p1) | `192.168.1.7` |
| **SIEM Server** | Wazuh manager — detection & analysis | Wazuh v4.14.5 | `192.168.1.18` |
| **Hypervisor** | Oracle VirtualBox, host-only network | — | `192.168.1.0/24` |

**Tools used:** Wazuh v4.14.5 · Hydra v9.6 · nmap · arp-scan · MITRE ATT&CK Navigator

---

## 📊 Key Results

| Metric | Result |
|---|---|
| Total events captured | **1,464** |
| Authentication failures logged | **1,327** |
| Successful authentications | **13** |
| Critical composite alert fired (Rule 40112) | **1** |
| Medium-severity brute-force alerts | **284** |
| MITRE ATT&CK techniques identified | `T1110`, `T1110.001`, `T1078`, `T1021` |
| Compliance frameworks auto-mapped | GDPR, NIST 800-53, PCI-DSS, HIPAA, GPG13 |
| Active Response mechanism | `firewall-drop` (tiered: 300s / 600s) |
| Repeat attack after Active Response | **Blocked — connection attempts timed out** |

---

## 🔬 Methodology

### Phase 1 — Building the Monitored Environment
- Verified Wazuh manager health (API, alert/monitoring/statistics index patterns — all ✅)
- Recorded a **pre-attack baseline** (0 agents, Medium: 6 / Low: 23 alerts) to prove any later spike is attributable to the attack, not noise
- Confirmed SSH active on the Ubuntu target, added a UFW allow rule
- Deployed the Wazuh agent (DEB package) via `wget` + `dpkg`, enabled via `systemd`
- Verified all 5 agent daemons running cleanly (`execd`, `agentd`, `syscheckd`, `logcollector`, `modulesd`)

<p align="center">
  <img src="images/01_baseline_dashboard.jpeg" alt="Wazuh pre-attack baseline dashboard — no agents, Medium:6 / Low:23" width="800">
  <br><em>Pre-attack baseline — Medium: 6 / Low: 23, zero agents registered</em>
</p>

<p align="center">
  <img src="images/02_agent_enrolled.jpeg" alt="Wazuh Endpoints view showing agent enrolled and active" width="800">
  <br><em>Ubuntu target enrolled and reporting active (Agent ID 003)</em>
</p>

### Phase 2 — Simulating the Attack
- **Recon:** `arp-scan` discovered 5 live hosts on the `/24`; `nmap -A` fingerprinted the Wazuh server itself and confirmed SSH open on the target
- **Validation:** manual SSH login confirmed password authentication was enabled (a prerequisite for the attack to succeed)
- **Attack:** launched Hydra v9.6 against `root@192.168.1.7` using the `fasttrack.txt` wordlist (262 candidates, 4 parallel threads)

```bash
hydra -l root -P /usr/share/wordlists/fasttrack.txt ssh://192.168.1.7 -t 4 -vV
```

<p align="center">
  <img src="images/03_hydra_attack.jpeg" alt="Hydra brute-force attack in progress against root@192.168.1.7" width="800">
  <br><em>Hydra v9.6 dictionary attack in progress — 262 candidate passwords, 4 parallel threads</em>
</p>

### Phase 3 — Detection & Alert Analysis
- Observed the alert spike: Medium severity jumped from a baseline of **6 → 84**
- Identified **Rule 2502** (`syslog: User missed the password more than one time`) firing 83 times, all sourced from the attacker IP
- Isolated the single most important event in the lab: **Rule 40112** — a composite rule that only fires when repeated auth failures are immediately followed by a success. This is definitive proof of compromise, and it auto-mapped to MITRE, GDPR, NIST 800-53, PCI-DSS, and HIPAA in one shot
- Confirmed shell access via the PAM `session opened` log entry for the `ubuntu` user

<p align="center">
  <img src="images/04_alert_spike.jpeg" alt="Wazuh dashboard during the attack — Medium severity spikes to 84" width="800">
  <br><em>During the attack — Medium severity jumps from 6 → 84, Low from 23 → 34</em>
</p>

<p align="center">
  <img src="images/05_rule_40112_critical_alert.jpeg" alt="Rule 40112 forensic detail — multiple authentication failures followed by a success" width="800">
  <br><em>Rule 40112 — the composite alert proving the brute force succeeded, auto-mapped to 5 compliance frameworks</em>
</p>

### Phase 4 — Threat Hunting & MITRE ATT&CK Mapping
- Used Wazuh's Threat Hunting dashboard to view the attack at scale: 1,464 total events, 1,327 failures, 13 successes
- Used the MITRE ATT&CK module to reconstruct the full kill chain:

```
T1110 (Brute Force)
   → T1110.001 (Password Guessing)
      → T1078 (Valid Accounts) + T1021 (Remote Services)
```

<p align="center">
  <img src="images/06_threat_hunting_dashboard.jpeg" alt="Wazuh Threat Hunting dashboard filtered to medium-severity brute force events" width="800">
  <br><em>Threat Hunting dashboard — Brute Force as the sole MITRE technique at medium severity</em>
</p>

<p align="center">
  <img src="images/07_mitre_attack_dashboard.jpeg" alt="MITRE ATT&CK dashboard showing tactic breakdown dominated by Credential Access and Lateral Movement" width="800">
  <br><em>MITRE ATT&CK dashboard — full tactic breakdown across the attack</em>
</p>

| Technique ID | Name | Tactic | Wazuh Rules |
|---|---|---|---|
| T1110 | Brute Force | Credential Access | 2502, 5758 |
| T1110.001 | Password Guessing | Credential Access, Lateral Movement | 5760, 5557 |
| T1078 | Valid Accounts | Defense Evasion, Persistence, Privilege Escalation | 5501, 40112 |
| T1021 | Remote Services (SSH) | Lateral Movement | 5715 |

<p align="center">
  <img src="images/08_mitre_kill_chain_table.jpeg" alt="MITRE event table showing the kill chain from T1110 to T1078 to T1021" width="800">
  <br><em>MITRE event table — the kill chain reconstructed chronologically, alert by alert</em>
</p>

### Phase 5 — Active Response: From Detection to Automated Containment

Detection alone doesn't stop an attacker — it just produces a very well-documented compromise. After watching Rule 40112 fire while the attacker still held a live shell, this lab was extended to close that gap: **Wazuh Active Response**, configured to automatically block the attacker's IP at the firewall the moment a brute-force pattern is confirmed.

Two response tiers were configured globally in **`ossec.conf` on the Wazuh manager**, using the built-in **`firewall-drop`** active response — escalating in severity and block duration as the attack progresses:

| Trigger Rule | Stage | Active Response | Block Duration |
|---|---|---|---|
| `5712` / `5720` — repeated SSH auth failures | Early-stage brute force (pre-compromise) | `firewall-drop` | **300s (5 min)** |
| `40112` — auth failures followed by a success | Confirmed compromise | `firewall-drop` | **600s (10 min)** |

```xml
<!-- /var/ossec/etc/ossec.conf — Wazuh manager -->
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5712,5720</rules_id>
  <timeout>300</timeout>
</active-response>

<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>40112</rules_id>
  <timeout>600</timeout>
</active-response>
```

**Why two tiers instead of one:** the lower-severity rule blocks the *attempt* — cutting off a brute-force attack early, before it has a chance to succeed, with a short timeout to avoid over-blocking on noisy networks. Rule 40112 (confirmed compromise) gets a longer timeout because at that point the threat is no longer hypothetical — the attacker already had valid credentials, so the response needs to hold the door shut longer.

**Validation:** the attack was re-run end-to-end after configuring Active Response. Hydra was launched against the target a second time, and this round the brute force was cut off mid-attempt — the manager pushed a `firewall-drop` command to the agent, `iptables` rules were inserted automatically, and subsequent connection attempts from the attacker IP timed out. The block self-expired after the configured window, confirming the response was both effective and non-permanent.

This closes the loop the original lab left open: **Detect → Attribute → Contain**, with the third step now automated and measured in seconds rather than however long it takes a human analyst to notice and act.

---

## 🗺️ Compliance Frameworks Mapped

The single critical alert (Rule 40112) automatically cross-referenced against **six** frameworks simultaneously:

| Framework | Controls / References Triggered |
|---|---|
| MITRE ATT&CK | T1110, T1078, T1021 |
| GDPR | Art. IV_35.7.d (risk assessment), IV_32.2 (security of processing) |
| NIST 800-53 | AU.14, AC.7, SI.4 |
| PCI-DSS | 10.2.4, 10.2.5, 11.4 |
| HIPAA | §164.312(b) — Audit controls |
| GPG13 | 7.1 (audit trail), 7.8 (intrusion detection) |

---

## 🛠️ Skills Demonstrated

**Blue Team / Defensive**
- SIEM deployment, configuration, and health verification
- Endpoint agent enrollment & service management (systemd)
- Log correlation and alert triage across severity levels
- Threat hunting using purpose-built SIEM modules
- MITRE ATT&CK mapping and kill-chain reconstruction
- Multi-framework compliance mapping
- Incident timeline reconstruction from raw logs
- IOC identification
- **Automated incident containment with Wazuh Active Response (`firewall-drop`)**
- **Tiered response design — escalating block duration by alert severity**
- **Response validation — re-attacking post-mitigation to confirm control effectiveness**

**Red Team / Offensive**
- Network reconnaissance with `arp-scan` and `nmap`
- Service enumeration and OS/version fingerprinting
- Credential brute-forcing with Hydra
- SSH exploitation and session verification
- Wordlist-based attack methodology

---

## 🔧 Defensive Recommendations

**High Priority**
- Disable password authentication — `PasswordAuthentication no`, enforce key-based auth only
- Disable root SSH login — `PermitRootLogin no`
- ✅ ~~Configure Wazuh Active Response on Rule 40112 to auto-block the attacker's IP via `firewall-drop`~~ — **implemented in Phase 5**

**Medium Priority**
- Rate limiting / Fail2ban — `MaxAuthTries 3`
- Enforce MFA via PAM (`libpam-google-authenticator`)
- Network segmentation — restrict SSH to trusted management networks

**Long-Term**
- Integrate Wazuh alerts with Slack/Teams/PagerDuty for real-time escalation
- Define a log retention policy meeting compliance requirements
- Expand the lab with a second attacker/target to simulate lateral movement

> 🔁 **Detection used to be where this lab stopped.** Phase 5 closed that gap with automated containment via Active Response. The next open question: how does this hold up against a *distributed* brute force — multiple source IPs, low-and-slow timing — where a simple per-IP `firewall-drop` starts to break down?

---

## 📅 Attack Timeline (condensed)

| Phase | Event |
|---|---|
| T+0 | Wazuh manager deployed, baseline recorded (Medium: 6 / Low: 23) |
| T+0 | Agent enrolled on Ubuntu target, reporting active |
| T+~2h | Recon: `arp-scan` + `nmap` identify target and fingerprint SIEM |
| T+~2h | Manual SSH login confirms password auth is exploitable |
| T+~2h | Hydra brute-force launched — 262 passwords, 4 threads |
| T+~2h | Rule 40112 fires — brute force succeeds, Critical alert |
| T+~3h | PAM session opened — shell access confirmed |
| T+~3h | Threat hunting + MITRE ATT&CK kill chain fully attributed |

*(Full timestamped timeline with dual-clock reconciliation available in the full report.)*

---

## 📁 Repository Structure

```
.
├── README.md                                  # this file
├── IR_Report_Brute_Force_Lab.docx              # full incident response report
├── SSH_Bruteforce_Wazuh_ThreatDetection.pdf    # full 24-page lab walkthrough w/ screenshots
└── images/                                     # screenshots referenced in this README
    ├── 01_baseline_dashboard.jpeg
    ├── 02_agent_enrolled.jpeg
    ├── 03_hydra_attack.jpeg
    ├── 04_alert_spike.jpeg
    ├── 05_rule_40112_critical_alert.jpeg
    ├── 06_threat_hunting_dashboard.jpeg
    ├── 07_mitre_attack_dashboard.jpeg
    └── 08_mitre_kill_chain_table.jpeg
```

---

## 👤 About

Built end-to-end in an isolated Oracle VirtualBox network (`192.168.1.0/24`) using Kali Linux, Ubuntu 26.04 LTS, and Wazuh v4.14.5. Backed by 24 timestamped screenshots captured throughout the exercise.

**Author:** Arjun R Pillai
**Project Date:** June 13, 2026

---

## ⚠️ Legal & Ethical Disclaimer

All testing in this project was performed exclusively within an isolated virtual lab environment that I own and control. Unauthorized scanning or brute-force attacks against systems you do not own or have explicit written permission to test are illegal.
