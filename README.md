# Entra-ID-hardening-Lab
# Entra ID Security Hardening & Incident Detection Lab

This lab focuses on the identity security layer of a cloud environment. Often the first thing attackers target since a single compromised account can bypass network defenses entirely. Using a Microsoft Entra ID tenant, I configured Conditional Access policies to enforce multi-factor authentication, block legacy authentication protocols, and require compliant devices for privileged roles. I then reviewed sign in logs to understand how Entra ID evaluates and logs each authentication attempt against these policies.

To test detection capability rather than just prevention, I simulated a compromised account scenario (atypical sign in location/impossible travel) and used Microsoft Entra ID Protection to detect the risky sign in, investigate the risk signal, and walk through remediation steps including password reset and session revocation. The goal was to build hands on experience with the full identity security lifecycle: harden, monitor, detect, and respond.

This is the first of two labs building out a full Entra ID security pipeline. This one covers hardening, monitoring, and detection/response. The follow up lab (Sentinel + KQL automated detection) builds on this same tenant.

![Lab flow diagram](images/lab-flow-diagram.png)

## Setup

For the lab I set up a brand new Azure account, which gave me $200 worth in credit to spend on any Azure services for the first 30 days.

Go to the Entra admin center and search Licenses > All products. An option to enroll in the Entra ID P2 30 day trial will appear.

![Azure free account dashboard](images/azure-free-account.png)

![Entra ID P2 trial page](images/entra-p2-trial.png)

Microsoft Entra ID P2 is the licensing tier that enables risk based security features like Identity Protection and Conditional Access risk policies, which this lab relies on to detect and respond to compromised accounts.

![Licensed features list](images/licensed-features.png)

In the Entra admin center, go to Users on the left navigation panel and create 3 new users: 1 admin account (testadmin) and 2 standard user accounts (testuser1 and testuser2).

![Create new user](images/create-new-user.png)

![Users list](images/users-list.png)

Once testadmin is created, go to Roles & admins > search Global Administrator > Add assignments > select testadmin, so it actually has the elevated permissions needed for later scenarios. The Global Administrator role can manage all aspects of Microsoft Entra ID and Microsoft services that use Microsoft Entra identities.

![Global Administrator assignment](images/global-admin-assignment.png)

**Alternative option:** The Microsoft 365 Developer Program gives eligible accounts a free, renewable Microsoft 365 E5 sandbox that includes Entra ID P2 pre populated with 25 fake users for testing attack scenarios, with no billing account required. This would have been my ideal setup, but my account was not eligible for the sandbox.

![Microsoft 365 Developer Program](images/dev-program.png)

## Enable MFA Enforcement

I built a baseline Conditional Access policy that requires MFA for all users on all cloud apps. This makes it so a sign in requires two or more different pieces of evidence to prove identity before gaining access.

**Policy: Require MFA for all Users**
- Users: All users
- Target resources: All resources
- Grant: Require multifactor authentication
- Enable policy: Report only, then switched to On

![New MFA policy](images/mfa-policy-new.png)

Security defaults must be disabled first, or the policy cannot be turned on.

![Disable security defaults](images/disable-security-defaults.png)

I then went to an incognito/private browser window, navigated to portal.azure.com, and signed in as testuser1 with the password that was set.

![Sign in screen](images/signin-testuser1.png)

I was prompted to install Microsoft Authenticator and scan a QR code to link the app.

![Install Authenticator prompt](images/install-authenticator.png)

![Scan QR code](images/scan-qr-code.png)

After setting up Authenticator, I had to update the password on first login. To confirm this was working properly, I signed in and out one more time and approved the sign in request through the app.

![Update password](images/update-password.png)

![Approve sign in request](images/approve-signin.png)

I repeated all steps for testadmin.

![Multiple signed in accounts](images/multiple-accounts-signedin.png)

## Build Layered Conditional Access Policies

### Block Legacy Authentication

This policy stops any sign in attempt that uses older, outdated communication protocols which cannot support multi factor authentication.

**Policy: Block Legacy Authentication**
- Users: All users, excluding the break glass admin
- Target resources: All cloud apps
- Conditions: Client apps, checked only Exchange ActiveSync clients and Other clients
- Grant: Block access
- Enable policy: Report only first, then switched to On

![Block Legacy Authentication policy](images/block-legacy-auth-policy.png)

### Risk Based Sign In Policy

This policy automatically evaluates the threat level of every sign in attempt in real time and triggers security controls like blocking access or requiring MFA.

**Policy: Risk Based Sign In MFA**
- Users: All users, excluding the break glass admin
- Target resources: All cloud apps
- Conditions: Sign in risk set to Medium and above
- Grant: Require multifactor authentication
- Enable policy: Report only first (this one requires P2), then switched to On

![Risk Based Sign In MFA policy](images/risk-based-signin-policy.png)

### Require Compliant Device for Admins

This policy requires that any sign in to a Global Administrator account come from a device enrolled in Intune and marked as compliant with organizational security standards.

**Policy: Require Compliant Device - Admin roles**
- Users: Select users and groups > Directory roles > Global Administrator
- Target resources: All cloud apps
- Grant: Require device to be marked as compliant
- Enable policy: Report only, kept permanently, since no compliant device exists in this lab and turning it On would lock testadmin out entirely

![Require Compliant Device policy](images/compliant-device-policy.png)

All four policies set up:

![All policies list](images/all-policies-list.png)

## Review Sign In Logs

Under Conditional Access > Monitoring, there are two options for viewing logs: Sign in Logs and Audit Logs. Sign in logs record authentication events and login attempts, while audit logs track configuration and administrative changes made within the tenant.

In sign in logs, we can view the user, application, status, IP, and location. We can see how testuser1 and the admin account logged in successfully into the Azure portal.

![Sign in logs list](images/signin-logs-list.png)

Clicking into a log shows additional detail.

![Sign in activity details](images/signin-activity-details.png)

In one case, only "Require MFA for all Users" applied and succeeded. Block Legacy Authentication, Risk Based Sign In, and Require Compliant Device did not trigger, since none of their specific conditions were met for that sign in. This confirms Conditional Access evaluates policies individually per sign in rather than applying every configured policy uniformly.

![Conditional Access tab detail](images/ca-tab-detail.png)

The Device info tab confirmed this sign in came from a device marked Compliant: No, Managed: No. This is the exact condition the "Require Compliant Device - Admin roles" policy checks against, meaning if that policy were switched to On rather than Report only, this sign in would have been blocked. This is why the policy was kept in Report only mode: no device in this lab environment is currently compliant, so enforcing it would lock out testadmin entirely.

![Device info tab](images/device-info-tab.png)

The Audit Logs tab shows the service, category, activity, status, status reason, target, and initiated by. This tracks what changes were made to the system, resources, or users. This is helpful for compliance tracking, governance, and finding out who added a user to a privileged group, deleted an application, or reset a password.

![Audit logs](images/audit-logs.png)

Using the Conditional Access filter in the sign in logs further confirmed that Conditional Access was not applied to every account, and that not every sign in requires multifactor authentication.

![Filtered sign in logs](images/filtered-signin-logs.png)

## Simulate a Compromised Account

In order to simulate a compromised account, I signed in as testuser1 using a VPN in an incognito window to trigger impossible travel. Impossible travel is a risk detection that flags two sign ins from the same account happening in places too far apart for real travel to have occurred in that amount of time. First I logged in normally into testuser1 without any VPN or incognito, then logged into testuser1 from a different country in incognito shortly after.

testuser1 log in completed at 2:47 PM on a standard Chrome tab.

![Normal sign in, Entra dashboard](images/normal-signin-dashboard.png)

I then signed out of the session and immediately connected to Proton VPN.

![Proton VPN connected to Netherlands](images/proton-vpn-netherlands.png)

For this scenario, let's assume the attacker also somehow has access to a method of verification.

![MFA verify identity prompt](images/verify-identity-prompt.png)

After successfully logging into the account, I disconnected the VPN and closed the tab.

In a real world scenario, an account like testuser1 is typically compromised through phishing, credential stuffing (reusing passwords leaked in other breaches), or MFA fatigue attacks (bombarding a user with approval requests until one is accidentally approved). This lab doesn't simulate the initial compromise itself, but instead simulates the behavior an attacker would show after signing in with valid credentials from an unfamiliar location. This is the pattern Identity Protection is designed to catch, no matter how the credentials were originally obtained.

To be thorough, I tested this using two different VPN providers across three locations: Netherlands and Hong Kong, through Proton and Windscribe. This was to rule out the chance that a single provider's IP just wasn't flagged by Microsoft.

![Windscribe connected to Hong Kong](images/windscribe-hongkong.png)

Looking at the sign in logs, one attempt showed a jump from New York to Amsterdam in under two hours, which is a real example of impossible travel by definition. Even so, none of these attempts generated a risk detection in Identity Protection.

![Sign in logs showing location jumps](images/signin-logs-location-jumps.png)

Microsoft's own documentation on simulating risk detections confirms this can happen. Their guide states there is a chance these steps will not trigger a detection, since the risk engine relies on machine learning models and known threat data rather than a simple rule. Consumer VPN IPs are not always on Microsoft's watch list, especially in a brand new trial tenant with very little sign in history to build a baseline from.

Following Microsoft's own recommended method, I tried again using Tor Browser instead on a separate device, signing in as testuser1 through myapps.microsoft.com. This time it worked. The sign in was flagged in real time as an Anonymous IP address detection, with a risk level of High and an attack type of obfuscation and access using a valid account. Tor exit nodes are much more reliably included on Microsoft's threat intelligence list than a typical consumer VPN, which is likely why this attempt succeeded where the others did not.

![Risk detection details, anonymous IP](images/risk-detection-anonymous-ip.png)

## Detect and Respond via Identity Protection

Back in the Jim Fullah admin account, Entra ID Protection has a Risky sign ins section where the compromised account can be reviewed. Three logs appeared for testuser1, all at risk: two medium and one high.

![Risk detections list](images/risk-detections-list.png)

Clicking the detection hyperlink gives additional information about the client sign in device, browser, type of attack, and policies applied. "Risk Based Sign In MFA" and "Require MFA for all Users" were successfully applied, while Block Legacy Authentication was not applicable.

![Risky sign in details, risk info](images/risky-signin-risk-info.png)

![Risk detection details](images/risk-detection-details.png)

![Conditional Access tab on risky sign in](images/risky-signin-ca-tab.png)

After reviewing details, I confirmed the user compromised. This tells Entra the sign in has been investigated and determined to be a real compromise, not a false positive.

![Confirm sign in compromised](images/confirm-compromised.png)

![Confirmation success message](images/confirmation-success.png)

After confirming the compromised account, an email was sent notifying of a detailed report, including a percentage graph of risky users.

![Email alert](images/email-alert.png)

![Risky users report](images/risky-users-report.png)

In real world environments, the best course of action is to reset the password for the user and revoke sessions. Revoking sessions removes all active sessions for the account on all devices. Once the password is reset, the user is delivered a temporary password on next sign in.

![Reset password dialog](images/reset-password-dialog.png)

![Revoke sessions prompt](images/revoke-sessions-prompt.png)

## Conclusion

This lab walked through the full identity security lifecycle in Microsoft Entra ID, from hardening to detection to response. I started by locking down the environment with Conditional Access policies, requiring MFA for all users, blocking legacy authentication, and setting up risk based sign in policies and compliant device requirements for admin roles. I then learned how to read sign in logs and understood that Conditional Access evaluates policies individually per sign in rather than applying every policy uniformly.

The most valuable part of this lab was simulating a real compromised account scenario. My first few attempts using consumer VPNs did not trigger a detection, which turned out to be a real and documented limitation rather than a mistake on my part. Microsoft's own guide on simulating risk detections confirms that these outcomes are not always guaranteed. Switching to Tor Browser and following Microsoft's official method finally triggered a real, high risk anonymous IP address detection in real time.

From there I got to walk through the full detection and response workflow that a SOC analyst would actually use. I investigated the risk detection, confirmed the sign in as compromised, and took remediation action by revoking the user's active sessions and resetting their password. I also got an automated email alert from Microsoft, which showed me how this kind of detection reaches an analyst without them having to actively watch the dashboard.

Overall this lab gave me hands on experience with identity based attacks and defenses that go beyond just reading about them. Troubleshooting the parts that did not work the first time, like the P2 licensing, the domain naming issue, and the failed detection attempts, ended up teaching me just as much as the parts that worked smoothly. This is the kind of practical, real world problem solving I want to bring into a SOC analyst role.

---

**Tools used:** Microsoft Entra ID (Free + P2 trial), Conditional Access, Identity Protection, Proton VPN, Windscribe, Tor Browser
