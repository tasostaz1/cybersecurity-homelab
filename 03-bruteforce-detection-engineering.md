# Brute-Force Detection: RDP/Local Account Attack vs. Windows + Wazuh

**Lab environment:** Kali Linux (attacker) → Windows 11 (target, Wazuh agent)
**Date:** 06/10/2026
**MITRE ATT&CK mapping:** T1110 — Brute Force, T1531 — Account Access Removal

## Objective

Simulate a credential brute-force attack against a Windows login surface and verify what
Windows and Wazuh actually capture natively — both for individual failed attempts and for
the broader attack pattern — without assuming tooling would work cleanly on the first try.

## Environment Setup

- **Attacker:** Kali Linux, IP `192.168.163.7`
- **Target:** Windows 11, IP `192.168.163.4`, Wazuh agent already configured (Exercises 1–2)
- **SIEM:** Ubuntu, Wazuh manager + dashboard

## Step 1 — Confirm Logon Auditing Was Enabled

```powershell
auditpol /get /subcategory:"Logon"
```

**Result:** `Success and Failure` already enabled — no changes needed. (Unlike Exercise 1,
where a different audit subcategory had to be manually turned on, this one was already
correctly configured.)

## Step 2 — Enable RDP and Confirm Reachability

Enabled Remote Desktop via Windows Settings, then confirmed from Kali:

```bash
nmap -p 3389 192.168.163.4
```

**Result:** `3389/tcp open ms-wbt-server` — reachable.

## Step 3 — Tooling Dead Ends

Tried two standard brute-force tools before finding one that actually worked reliably
against this target:

- **Hydra against RDP** (`hydra -l administrator -P passwords.txt rdp://...`) — failed
  immediately with `freerdp: The connection failed to establish`. Diagnosed as a **Network
  Level Authentication (NLA)** compatibility issue: Hydra's RDP module is explicitly
  labeled experimental and frequently fails to even establish a connection when NLA is
  enabled. Disabling NLA on the target did not resolve it — Hydra's RDP support proved
  unreliable regardless.
- **Hydra against SMB** (`hydra -l administrator -P passwords.txt smb://...`) — failed
  with `invalid reply from target`, consistent with Windows 11's newer SMB
  signing/encryption requirements not being supported by Hydra's older SMB module.
- **ncrack against RDP** — connected but produced no visible output or errors even after
  several minutes, suggesting it was hanging rather than failing outright.

**What worked:** a direct `xfreerdp` connection succeeded immediately and returned a
proper authentication rejection (`ERRCONNECT_LOGON_FAILURE`) rather than a connection
error — confirming the service itself was fine, and the issue was tooling compatibility,
not the target. Scripted a simple loop around `xfreerdp` to brute-force manually:

```bash
while read pass; do
  xfreerdp /v:192.168.163.4 /u:administrator /p:"$pass" /cert:ignore +auth-only \
    2>&1 | grep -i "logon\|failure\|success"
done < passwords.txt
```

**Lesson:** purpose-built brute-force tools aren't always the most reliable option against
a given target/protocol combination — a well-understood, "legitimate" client tool scripted
in a loop can be a more dependable substitute than fighting an experimental module. Worth
testing the simplest possible manual connection before assuming a target is misconfigured.

## Step 4 — Verifying Individual Failed-Logon Detection

A single manual `xfreerdp` attempt with a wrong password was enough to confirm the full
pipeline before running the whole list:

- **Windows Event Viewer** → Security log → **Event ID 4625** generated correctly,
  correctly recording the attacker's IP (`192.168.163.7`), target username, and NTLM
  authentication details.
- **Wazuh** picked this up immediately via its native `EventChannel` collector — **no
  custom decoder was needed** (same lesson as Exercise 1: check built-in coverage before
  writing custom rules). A built-in rule fired automatically:

```
rule.id: 60122
rule.description: Logon Failure - Unknown user or bad password
rule.groups: windows, windows_security, authentication_failed
rule.level: 5
```

## Step 5 — Running the Full Attempt List and Observing the Brute-Force Detection

Ran the full `xfreerdp` loop against the password list. Windows' own **account lockout
policy** triggered partway through, generating:

```
Event ID: 4740 — A user account was locked out
Account Name: Administrator
Caller Computer Name: kali
```

Wazuh matched this to a built-in correlation-style rule:

```
rule.id: 60115
rule.description: User account locked out (multiple login errors)
rule.groups: windows, windows_security, authentication_failures
rule.level: 9
rule.mitre.id: T1110, T1531
rule.mitre.tactic: Credential Access, Impact
```

**Key observation:** unlike the port-scan detection rule built by hand in Exercise 1
(a Wazuh frequency/timeframe correlation rule counting raw firewall events), this
detection required no correlation logic in Wazuh at all. **Windows' own account lockout
policy performed the correlation** — counting repeated failures and generating a single,
distinct event (4740) once a threshold was crossed. Wazuh's job was simply to recognize
that one event via an existing built-in rule. This is a meaningfully different and, in
some ways, more elegant detection path than building frequency-based correlation from
scratch: **the OS's own security control did the pattern-matching; the SIEM only needed
to surface the result.**

## Key Takeaways

1. **Tool reliability varies a lot by target/protocol combination** — Hydra and ncrack
   both struggled against this specific RDP/SMB setup, while a scripted loop around a
   standard RDP client (`xfreerdp`) worked immediately. Don't assume a failed brute-force
   tool means a hardened target; it may just mean the wrong tool.
2. **Not every detection needs a custom correlation rule.** Windows' native account
   lockout policy already performs brute-force correlation at the OS level — Wazuh's
   built-in ruleset was already mapped to recognize the resulting event, MITRE
   classification included, with zero custom configuration.
3. **Individual-event and pattern-level detections are both valuable and distinct.**
   Event 4625 (one failed logon) and Event 4740 (lockout after many failures) represent
   two different severity levels and two different investigative questions — "did someone
   fail to log in" versus "is someone actively trying to brute-force this account" — and
   a mature detection setup should surface both, not just the aggregate.

## Comparison Across Exercises

| | Exercise 1 (Port Scan) | Exercise 2 (Pivot) | Exercise 3 (Brute Force) |
|---|---|---|---|
| Log source | `pfirewall.log` (file) | `pfirewall.log` (file) | Windows Security `EventChannel` |
| Detection logic | Custom Wazuh correlation rule | Same custom rule (reused) | Built-in Wazuh rule, OS does correlation |
| Main obstacle | Log write-throttling under burst load | Same throttling issue, confirmed via pivot | Brute-force tool compatibility (Hydra/ncrack) |
| MITRE mapping | T1046 | T1046 | T1110, T1531 |