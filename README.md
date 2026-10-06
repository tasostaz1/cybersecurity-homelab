# Cybersecurity Lab Journey

Documenting my hands-on path into cybersecurity through a home lab: attacking, detecting,
and analyzing in a self-built red/blue environment.

## Lab Environment

| Role | OS | Purpose |
|---|---|---|
| Attacker | Kali Linux | Recon, exploitation, attack simulation |
| Target | Windows 11 | Victim machine, Wazuh agent + Sysmon installed |
| SIEM | Ubuntu Server | Wazuh manager + dashboard for detection/analysis |

All VMs run on an isolated internal network — no traffic leaves the lab.

## Skills Demonstrated

- Network reconnaissance (Nmap)
- Windows logging & auditing configuration (Event Viewer, Sysmon, `auditpol`)
- SIEM detection engineering (Wazuh custom decoders/rules)
- Log analysis and correlation rule tuning
- Incident documentation / write-up discipline

## Write-ups

| # | Exercise | Techniques | MITRE ATT&CK | Status |
|---|---|---|---|---|
| 01 | [Nmap Recon + Wazuh Visibility Baseline](./01-nmap-wazuh-port-scan-detection.md) | Port scanning, firewall logging, custom detection rule | T1046 – Network Service Discovery | ✅ Complete |
| 02 | Metasploit Exploitation → Windows Event Log Analysis | Exploitation, Sysmon/Event Log tracing, kill chain reconstruction | TBD | ✅ Complete |
| 03 | Brute-Force / Credential Attack Detection Engineering | Hydra/Crowbar, Sysmon, custom Wazuh alerting | T1110 – Brute Force | ✅ Complete |

## Repo Structure

```
.
├── README.md                                  # you are here
├── 01-nmap-wazuh-port-scan-detection.md
├── 02-metasploit-event-log-analysis.md         
├── 03-bruteforce-detection-engineering.md     

```

## Key Lessons So Far

- [ ] Default Windows logging captures far less than assumed — visibility must be
      deliberately configured.
- [ ] Log completeness and real-time detection can work against each other under load.
- [ ] Always check for a SIEM's built-in detections before writing custom ones from scratch.


## About Me

I'm switching careers into cybersecurity and just started — this repo is where I'm documenting that journey from day one, mistakes included.

To get hands-on fast, I built a home lab (Kali Linux attacker, Windows 11 target, and Ubuntu running Wazuh as a SIEM) so I could practice both attacking and detecting rather than just reading theory. I'm still deciding between blue team (SOC/detection) and red team (pentesting) work, so early on I'm deliberately practicing both sides before specializing.

Each write-up here covers a full exercise start to finish — including the dead ends and misconfigurations I hit along the way, because working through why something didn't work the first time taught me more than the parts that went smoothly.

## Contact

https://www.linkedin.com/in/anastasios-nikolaidis-123877170/
