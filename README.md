# Entra-ID-hardening-Lab
# Entra ID Security Hardening & Incident Detection Lab

This lab focuses on the identity security layer of a cloud environment. Often the first thing attackers target since a single compromised account can bypass network defenses entirely. Using a Microsoft Entra ID tenant, I configured Conditional Access policies to enforce multi-factor authentication, block legacy authentication protocols, and require compliant devices for privileged roles. I then reviewed sign in logs to understand how Entra ID evaluates and logs each authentication attempt against these policies.

To test detection capability rather than just prevention, I simulated a compromised account scenario (atypical sign in location/impossible travel) and used Microsoft Entra ID Protection to detect the risky sign in, investigate the risk signal, and walk through remediation steps including password reset and session revocation. The goal was to build hands on experience with the full identity security lifecycle: harden, monitor, detect, and respond.

This is the first of two labs building out a full Entra ID security pipeline. This one covers hardening, monitoring, and detection/response. The follow up lab (Sentinel + KQL automated detection) builds on this same tenant.

## Architecture

* Identity provider: Microsoft Entra ID (Free + P2 trial)
* Global Administrator: `testadmin`
* Standard test users: `testuser1`, `testuser2`
* Break-glass admin: primary tenant account (excluded from all Conditional Access policies)

## What I Built

**Conditional Access & MFA**

* Enabled a baseline policy requiring MFA for all users, all cloud apps
* Registered a test account with Microsoft Authenticator and verified the end-to-end MFA challenge
* Built a Block Legacy Authentication policy targeting outdated client protocols
* Built a Risk-Based Sign-In MFA policy (Sign-in risk: Medium and above)
* Built a Require Compliant Device policy scoped to the Global Administrator role, kept in Report-only since no compliant device exists in this lab

**Sign-in Log Review**

* Reviewed sign-in and audit logs to see how Entra ID evaluates each policy per individual sign-in rather than uniformly across every event
* Confirmed via the Conditional Access and Device info tabs which specific conditions caused a policy to apply, not apply, or succeed

**Simulated Account Compromise**

* Attempted to trigger impossible travel and anonymous IP detections using Proton VPN and Windscribe across three countries
* Documented a real impossible-travel-qualifying sign-in pair (New York to Amsterdam in under two hours) that still did not trigger a detection, consistent with Microsoft's own documented limitations for risk simulation
* Successfully triggered a real-time, High-risk Anonymous IP address detection using Tor Browser, following Microsoft's official simulation guidance

**Detection & Response**

* Investigated the flagged sign-in through Identity Protection's Risk detection details, reviewing detection type, attack type, and sign-in metadata
* Confirmed the sign-in as compromised, escalating the account to High risk
* Remediated by revoking all active sessions and resetting the account's password
* Received and reviewed Microsoft's automated "user at risk" email alert

## Skills Demonstrated

* Conditional Access policy design (MFA enforcement, legacy auth blocking, risk-based access, device compliance)
* Identity Protection investigation and remediation workflow (confirm compromise, reset password, revoke sessions)
* Sign-in log and audit log analysis
* Real-world detection limitations and troubleshooting: P2 licensing access, tenant domain naming, unreliable risk detection across multiple VPN providers, and following official Microsoft simulation methodology to achieve a valid detection

## Full Documentation

The complete step-by-step write-up, including screenshots for every phase, is here Entra ID (Azure AD) security hardening lab.pdf

## Author

## Author

Jim Fullah — Cybersecurity student at GMU
