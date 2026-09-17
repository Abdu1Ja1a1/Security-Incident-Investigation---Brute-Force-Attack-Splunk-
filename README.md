# SOC Incident Investigation — Brute Force Compromise

## Overview

This project documents my investigation of a simulated brute-force attack against a privileged SSH account on `authserver`. The investigation began after a dense cluster of failed login attempts stood out in `auth.log`, and was reconstructed entirely through after-the-fact Splunk log analysis rather than a live alert.

The investigation identified a single-source attacker (`185.220.101.47`), a 47-attempt failed login burst over ~3 minutes, and a subsequent successful login that established a root-level (`uid=0`) session on the `admin` account.

📄 **[Full PDF Report](./soc-incident-report.pdf)**

## Objectives

The main goals of the investigation were to:

- Determine whether the failed login cluster represented a genuine attack
- Confirm whether the attack originated from a single source or a distributed set of IPs
- Establish the exact attack window (first and last attempt)
- Determine whether the attack resulted in a successful compromise
- Determine the privilege level obtained if compromise occurred
- Identify the detection gaps that allowed the attack to go unnoticed in real time
- Document remediation and hardening recommendations

## Tools Used

- Splunk Enterprise (local instance)
- Splunk Search Processing Language (SPL) — `rex`, `stats`, `timechart`
- `auth.log` (SSH authentication log)

## Environment

| | |
|---|---|
| **Log source** | `auth.log` (simulated SSH authentication log) |
| **Host** | `authserver` |
| **SIEM tool** | Splunk Enterprise (local instance) |
| **Date of investigation** | August 9, 2026 |
| **Total events analyzed** | 72 (background traffic + attack) |

## Key Findings

- 47 failed login attempts against `admin` from a single source IP, `185.220.101.47`
- Attempts were evenly spaced (~1 every 4 seconds), consistent with automated tooling rather than manual entry
- Attack window: 04:07:03 – 04:10:10 (failed attempts), followed by a successful login at 04:10:14
- A root-level session (`uid=0`) was opened for `admin` at 04:10:15, confirming full account compromise
- No account lockout, rate-limiting, or real-time alerting was in place to stop or flag the attack while in progress

## Investigation Process

1. Ingested `auth.log` into Splunk as a custom source type
2. Ran an initial search across all events to baseline normal authentication activity
3. Visually identified the anomalous failed-login cluster via the event timeline histogram
4. Filtered for `"Failed password"` events, returning 47 results
5. Extracted source IPs with `rex` and grouped by IP (`stats count by src_ip`) to confirm a single-source attack
6. Calculated first/last attempt timestamps (`earliest`/`latest`) to establish the attack window
7. Searched for the corresponding `"Accepted password for admin"` event to confirm the moment of compromise
8. Searched for the `"session opened for user admin"` event to confirm the privilege level obtained (`uid=0`)
9. Built a `timechart span=30s count` visualization to chart attempt volume over time
10. Documented detection gaps and remediation recommendations
11. Created the final incident report

## Incident Summary

| | |
|---|---|
| **Incident ID** | INC-2026-0809-001 |
| **Classification** | Brute Force Attack — Successful Compromise |
| **Severity** | High |
| **Attacker IP** | `185.220.101.47` |
| **Target account** | `admin` on `authserver` |
| **Attack window** | 04:07:03 – 04:10:15, August 9, 2026 (~3 min 11 sec) |
| **Outcome** | 47 failed attempts → successful login → root-level (`uid=0`) session |
| **Status** | Confirmed / Closed (simulated investigation) |

## Recommendations

1. Implement account lockout after a low threshold of failed attempts, with a cooldown period
2. Implement rate-limiting on the authentication endpoint, independent of lockout
3. Enforce a strong password policy, including checks against breached-password lists
4. Configure real-time SIEM alerting for authentication failure bursts
5. Enable multi-factor authentication (MFA) on privileged/admin accounts
6. Rotate the compromised account's credentials and audit the attacker's session window for further compromise

## Outcome

The investigation confirmed a successful, fully automated brute-force attack against a privileged account. The attack itself was unsophisticated — no evasion, no distributed infrastructure, just a fast, consistent guessing pattern against an unprotected login — which is exactly what made it easy to reconstruct after the fact and exactly what a basic rate-limit or lockout policy would have stopped outright.

The more important finding wasn't the attack technique, it was the **detection gap**: every signal needed to catch this in progress was already in the logs. This reinforced a broader lesson — **exploitability and detectability are separate concerns.** Closing that gap means real-time alerting on failure bursts and account protections (lockout, rate-limiting, MFA), not better logging after the fact.

## Repo Structure

```
soc-incident-brute-force/
├── README.md
├── soc-incident-report.pdf
├── auth.log
└── evidence/
    ├── src_ip_stats.png
    ├── attack_window_stats.png
    ├── accepted_password.png
    ├── session_opened.png
    └── chart.png
```

> Note: the `evidence/` screenshots referenced in the report (Splunk search results) aren't included in this project yet — see the report's Evidence section for placeholders marking where each one belongs.