# SJULTRA SOP's

Standard Operating Procedures for SJULTRA IT operations. This repo is a living index — new SOPs will be added over time.

## Index

| SOP | Description |
|---|---|
| [Install and Connect to Exchange Online PowerShell](sops/exchange-online-powershell-setup.md) | Installing the Exchange Online PowerShell module, fixing execution policy issues, and connecting to a Microsoft 365 tenant. |
| [Fix User Unable to Schedule Teams Meetings in Outlook/OWA](sops/teams-meeting-scheduling-permission-fix.md) | Diagnosing and resolving the "You do not have permissions to invite others" error — mailbox meeting provider and misconfigured resource account checks. |
| [Case Study (PDCA): User Unable to Schedule Teams Meetings](sops/teams-meeting-scheduling-pdca-case-study.md) | Full Plan-Do-Check-Act record of the troubleshooting behind the SOP above, including dead ends and what didn't work. |
| [Enable FIDO2 / Passkey MFA in Microsoft Entra ID](sops/SOP%20-%20Enable%20FIDO2%20Passkey%20MFA%20(Microsoft%20Entra%20ID).md) | Opting in to passkey profiles, configuring device-bound/synced passkey settings, piloting, and rolling out phishing-resistant FIDO2 sign-in org-wide. |
| [PDCA Plan: Enable FIDO2 / Passkey MFA in Microsoft Entra ID](sops/fido2-passkey-mfa-enablement-pdca.md) | Plan-Do-Check-Act framing behind the FIDO2/Passkey SOP above — objective, key decisions, pilot steps, verification checklist, and rollout/rollback actions. |
| [Release and Allowlist False-Positive Phishing Quarantine (Microsoft 365 Defender)](sops/SOP%20-%20Release%20and%20Allowlist%20False-Positive%20Phishing%20Quarantine%20(Microsoft%20365%20Defender).md) | Diagnosing a Defender quarantine verdict, telling a false-positive phishing block apart from real spoofing via SPF/DKIM/DMARC, and safely overriding it with a Tenant Allow/Block List entry. |
| [PDCA Plan: Atera MSP Environment Setup (Replacing Syxsense)](sops/atera-syxsense-migration-pdca.md) | Plan-Do-Check-Act migration plan for replacing Syxsense with Atera as the MSP/RMM platform — objective, scope, feature-gap risks, dated timeline, and decommission checklist. |
| [Block Access to an Amazon Bedrock Foundation Model (SCP + IAM Deny)](sops/SOP%20-%20Block%20Amazon%20Bedrock%20Model%20Access%20%28SCP%20%2B%20IAM%20Deny%29.md) | How to block a Bedrock foundation model (SCP and/or IAM Deny) now that AWS has retired the self-service Model access page — single account or org-wide. |
| [Runbook: Block anthropic.claude-sonnet-4-6 in Sandbox Account (279199950628)](sops/aws-bedrock-claude-sonnet-4-6-sandbox-block-runbook.md) | Specific execution of the SOP above — exact SCP JSON, attach/verify/rollback steps for the Aug 2026 Bedrock cost anomaly in the Sandbox account. |
| [Reclaim a Departed Employee's Email Address as a Distribution List (Hybrid AD + Exchange Online)](sops/SOP%20-%20Reclaim%20a%20Departed%20Employee%27s%20Email%20Address%20as%20a%20Distribution%20List%20%28Hybrid%20AD%20%2B%20Exchange%20Online%29.md) | Freeing a shared address (e.g. info@) off a departed employee's directory-synced mailbox and handing it to a distribution list — covers the on-prem write-scope block, UAC token-filtering false alarm, and default-primary-address gotchas. |
| [Case Study (PDCA): Real Mailboxes Redirected by the `ibenit-catch-all` Transport Rule](sops/catch-all-redirect-incident-pdca.md) | Diagnosing why real and shared mailboxes were getting caught by a catch-all/redirect transport rule — broken dynamic-group filters, missing recipient-domain scoping, an `SMTP:`/`smtp:` proxy-address casing gap, and a dynamic-group filter change that silently doesn't propagate to the rule until it's re-bound. Resolved. |
| [Build a Domain-Scoped Catch-All/Redirect Rule with Self-Maintaining Exceptions (Exchange Online)](sops/catch-all-redirect-setup-sop.md) | Repeatable procedure for standing up this pattern anywhere: accepted-domain (InternalRelay) prerequisite, scoping the transport rule, building a self-maintaining dynamic exception group per domain, safe validation, and the mandatory group re-bind step after any filter edit. |

## Adding a new SOP

1. Add the new `.md` file to the `sops/` folder.
2. Add a row to the table above linking to it.
