# Phishing Triage Report — Two Real-World Samples

**Date:** 07/10/2026
**Samples analyzed:** 2 unsolicited emails received in personal inbox, flagged as suspicious

---

## Sample 1 — Fake Amazon Order Confirmation

### Headers

```
Authentication-Results: spf=pass; dkim=pass; dmarc=pass
header.d=al-educacao-sp-gov-br.20251104.gappssmtp.com
header.from=al.educacao.sp.gov.br
sender IP: 74.125.230.67
```

### Analysis

| Check | Result | Notes |
|---|---|---|
| SPF | Pass | Sending IP authorized for this domain |
| DKIM | Pass | Signed via Google Workspace (`.gappssmtp.com`) |
| DMARC | Pass | Policy satisfied |
| Claimed sender | "Amazon" | Order confirmation branding |
| Actual authenticated domain | `al.educacao.sp.gov.br` | A Brazilian state government education department |
| Verdict signal | **Major mismatch** | No legitimate reason for this domain to send Amazon-branded mail |

### Content red flags

- Phone number listed (`1-813-494-8359`) does not match any official Amazon support
  line — consistent with a "callback scam" pattern designed to get the victim to call
  and be socially engineered over the phone
- Card number listed as "Visa ending in 91761" — 5 digits, not the standard 4-digit
  format real card-ending references always use, indicating sloppily generated template
  content
- No link was present in this sample; the attack vector appears to be the phone number
  itself

### Root cause assessment

**All three authentication checks passing does not indicate this email is safe.** The
most likely explanation is a **compromised mailbox** within the legitimate
`al.educacao.sp.gov.br` Google Workspace environment — an attacker with access to a real
account can send mail that passes every technical authentication check, because as far
as the receiving mail server is concerned, it genuinely did come from that domain's
authorized infrastructure. This is a materially different (and harder to detect) threat
model than a spoofed/look-alike domain.

### Verdict: **Phishing / scam (callback scam via compromised third-party mailbox)**

---

## Sample 2 — Fake Walmart Prize Notification

### Headers

```
Authentication-Results: spf=none; dkim=none; dmarc=none
smtp.helo=hukm.vangcentrum.com
header.from=JKHAGS0C1OPTV7HX8RXULBGCOWGJ
sender IP: 172.173.110.48
```

### Analysis

| Check | Result | Notes |
|---|---|---|
| SPF | None | No SPF record published for sending domain at all |
| DKIM | None | Message not signed |
| DMARC | None | No policy published |
| From header | Garbled random string | Not a valid email address format — spam-kit artifact |
| Sending IP | `172.173.110.48` | Microsoft Azure datacenter/hosting range |
| AbuseIPDB result | 0% confidence, 0 reports | **Not meaningful** — cloud-hosted IPs are rotated frequently by spammers before accumulating reports |

### Content red flags

- Generic "you've won" prize bait (free Walmart e-bike) — classic high-volume scam template
- Username shown as a placeholder-style string (`lpf_a_n`), not the recipient's actual
  name, consistent with a mass/templated send rather than a targeted message
- All links (including the "unsubscribe" link) point to an unrelated third-party domain
  (`newswiftgarden.com`), not `walmart.com` or any Walmart-owned domain
- Urgency language ("Hurry up! Your Reward is Ready") typical of social-engineering bait

### Link analysis

Submitted the "Claim Reward" link to urlscan.io. The scan returned a page stating the
"offer is not available in your country" rather than the actual payload.

**This is not a clean result** — it's consistent with **geofencing/geo-targeted
evasion**, where the destination infrastructure checks the visitor's apparent location
(or scanning-service origin) before serving malicious content, and serves a benign
placeholder to anything that doesn't match the intended target region. This is a
deliberate anti-analysis technique, not evidence the link is safe.

### Verdict: **Phishing / scam (mass-distributed prize scam, geo-targeted payload)**

---

## Cross-Sample Comparison

| | Sample 1 (Amazon) | Sample 2 (Walmart) |
|---|---|---|
| Technical sophistication | High — real infrastructure, valid signing | Low — no authentication, garbled headers |
| Likely root cause | Compromised legitimate mailbox | Disposable spam/scam infrastructure |
| Primary hook | Fear/urgency (unexpected charge) | Greed/urgency (free prize) |
| Attack vector | Phone callback | Malicious link (geo-fenced) |
| Hardest part to detect | Passing SPF/DKIM/DMARC despite being malicious | N/A — detectable via headers alone without deep analysis |

## Key Takeaways

1. **Passing SPF/DKIM/DMARC is necessary but not sufficient for trust.** These checks
   confirm the technical sending path was authorized — they say nothing about whether the
   account or human behind that path has been compromised. Content and context analysis
   remain essential even when authentication passes cleanly.
2. **A clean IP reputation score on a datacenter/cloud-hosted IP is a weak signal.**
   Spammers using Azure/AWS/etc. rotate IPs faster than abuse databases can catch up —
   "0 reports" should be read as "not yet reported," not "confirmed safe," especially for
   hosting-provider IP ranges.
3. **Automated sandbox tools can be evaded by the target themselves.** Geofencing and
   similar checks mean a scan result of "nothing malicious found" doesn't always reflect
   what an actual intended victim would see — analysts should document this limitation
   explicitly rather than treating a sandbox's non-finding as a clean bill of health.
4. **Sender/content mismatch remains one of the most reliable manual signals** — in both
   samples, the authenticated sending domain had no plausible relationship to the brand
   being impersonated, which was detectable before any deeper technical analysis was done.

## Next Steps

- [ ] Report both samples to the respective brands' phishing-report addresses
      (`reportphishing@apwg.org` is also a general-purpose option) and to Google
      Safe Browsing / Microsoft SmartScreen if applicable
- [ ] Consider setting up a dedicated throwaway inbox to collect a larger phishing
      sample set over time for future analysis practice
- [ ] Move to the Boss of the SOC (BOTS) dataset investigation as the next real-world
      scenario project
