# Port Scan Detection: Nmap Recon vs. Wazuh Visibility

**Lab environment:** Kali Linux (attacker) → Windows 11 (target, Wazuh agent) → Ubuntu (Wazuh manager)
**Date:** 21/09/2026
**MITRE ATT&CK mapping:** T1046 — Network Service Discovery

## Objective

Establish a baseline of what an Nmap port scan actually looks like from the defender's
side, and build a working Wazuh detection rule for it — starting from raw recon, through
several dead ends, to a firing alert.

## Environment Setup

- **Attacker:** Kali Linux, IP `192.168.163.3`
- **Target:** Windows 11, IP `192.168.163.4`, Wazuh agent installed, Sysmon (SwiftOnSecurity config)
- **SIEM:** Ubuntu, Wazuh manager + dashboard

## Step 1 — Baseline

Confirmed the Wazuh dashboard showed no meaningful events for the Windows host prior to
any scan activity, so any subsequent events could be attributed to the exercise.

## Step 2 — Initial Recon

Ran three progressively detailed Nmap scans from Kali against the Windows target:

```bash
sudo nmap -sS 192.168.163.4
sudo nmap -sV 192.168.163.4
sudo nmap -A 192.168.163.4
```

**Result:** 999 ports filtered (no response), port 445 (SMB) open.

## Step 3 — First Dead End: No Sysmon or Security Log Visibility

Expected to see scan activity in Sysmon and/or Windows Security event log. Found neither.

**Root cause investigation:**
- Windows Security log: `auditpol /get /category:*` showed `Filtering Platform Connection`
  set to **No Auditing** — Event ID 5156 will never be generated without this enabled.
- Sysmon: despite using the well-known SwiftOnSecurity config, still saw nothing for port
  445 (SMB) traffic specifically. Inspecting the config XML confirmed it **intentionally
  excludes ports 135/139/445** from `NetworkConnect` logging to reduce noise from normal
  enterprise SMB chatter.
- For the 999 filtered ports: Windows Firewall dropped these before any process/socket was
  involved, so there was nothing for Sysmon to attribute a connection to in the first place.

**Lesson:** a scan against mostly-closed/filtered ports on a default Windows install is
close to invisible without additional logging configured. "Nothing happened" is itself
useful defensive information, not a broken lab.

## Step 4 — Second Dead End: Windows Firewall Log Under Burst Load

Enabled Windows Firewall logging (`netsh advfirewall set allprofiles logging
droppedconnections enable`) to capture the filtered-port traffic directly.

**First attempt** (`nmap -sV` at default timing) against ~999 filtered ports produced only
**4 DROP log lines** — far fewer than expected.

**Diagnosis:** re-ran the same scan with `-T2` (slow/"polite" timing). The slower scan
produced a complete, proportional set of DROP entries. This confirms Windows Firewall's
`pfirewall.log` **silently drops log entries under high write-burst load** — a real,
documented limitation, not a configuration mistake.

**Trade-off this creates:**
| Scan speed | Logging completeness | Detection feasibility |
|---|---|---|
| Fast (default/`-T4`) | Incomplete — entries lost to throttling | Good — events cluster in time, easier for a frequency-based rule to catch |
| Slow (`-T2`) | Complete | Poor — events too spread out to cluster within a short detection window |

**Lesson:** relying on a single log source has real, measurable limits under load — a
finding worth calling out in any real detection design, not just this lab.

## Step 5 — Building the Detection

- Configured the Wazuh agent to monitor `pfirewall.log` via a `<localfile>` block in `ossec.conf`.
- Initially wrote a **custom decoder** — hit a syntax error (`(DROP|ALLOW)` alternation
  isn't supported by Wazuh's regex engine) and an attribute error (`after_prematch` vs.
  the correct `after_parent` for a child decoder).
- **Third dead end:** after fixing the custom decoder, `wazuh-logtest` showed the event was
  actually being matched by Wazuh's **built-in** `windows-firewall` decoder and rule `4101`
  — a near-identical name to the custom one, which meant the custom decoder/rule never
  actually fired. Rebuilt the correlation rule on top of the built-in rule `4101` instead
  of reinventing parsing that Wazuh already provides out of the box.

**Final rule** (`/var/ossec/etc/rules/local_rules.xml`):

```xml
<group name="firewall,windows_firewall,">

  <rule id="100011" level="10" frequency="[fill in final value]" timeframe="[fill in final value]">
    <if_matched_sid>4101</if_matched_sid>
    <same_source_ip />
    <different_dst_port />
    <description>Possible port scan detected from $(srcip) - multiple ports probed in $(timeframe)s</description>
    <mitre>
      <id>T1046</id>
    </mitre>
  </rule>

</group>
```

## Step 6 — Validation

Confirmed the rule fires on a live scan. Sample alert:

```
rule.id: 100011
rule.level: 10
rule.description: Possible port scan detected from 192.168.163.3 - multiple ports probed in 30s
rule.mitre.id: T1046
rule.mitre.tactic: Discovery
```


## Threshold Tuning Notes

- Started at `frequency=15, timeframe=30` — never fired, due to the log-loss issue in
  Step 4 reducing effective event volume below the threshold.
- Loosened to `frequency=5, timeframe=60` to confirm the correlation logic itself was
  sound — fired successfully.


## False Positive Considerations

- A low frequency threshold risks firing on benign bursts (e.g. a browser or VPN client
  retrying a connection to a few ports in quick succession).


## Key Takeaways

1. Default Windows logging (Security audit + Sysmon) captures far less than you'd assume —
   visibility has to be deliberately configured, not assumed.
2. Log completeness and detection timing work against each other under load — a faster,
   more "obvious" attack can paradoxically be easier to detect than a slow, careful one,
   because logging keeps up better spread out, while correlation windows need events
   clustered in time.
3. Check for built-in decoders/rules before writing custom ones — Wazuh ships with broad
   out-of-the-box coverage, and a near-duplicate custom decoder name can silently shadow
   or be shadowed without an obvious error.

## Next Steps

- [ ] Tighten the detection rule with a properly justified threshold
- [ ] Test against legitimate/benign traffic to validate false-positive rate
- [ ] Move to Exercise 2: Metasploit exploitation → Windows Event Log analysis
