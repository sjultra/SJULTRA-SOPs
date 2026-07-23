# SJULTRA SOP's

Standard Operating Procedures for SJULTRA IT operations. This repo is a living index — new SOPs will be added over time.

## Index

| SOP | Description |
|---|---|
| [Install and Connect to Exchange Online PowerShell](sops/exchange-online-powershell-setup.md) | Installing the Exchange Online PowerShell module, fixing execution policy issues, and connecting to a Microsoft 365 tenant. |
| [Fix User Unable to Schedule Teams Meetings in Outlook/OWA](sops/teams-meeting-scheduling-permission-fix.md) | Diagnosing and resolving the "You do not have permissions to invite others" error — mailbox meeting provider and misconfigured resource account checks. |
| [Case Study (PDCA): User Unable to Schedule Teams Meetings](sops/teams-meeting-scheduling-pdca-case-study.md) | Full Plan-Do-Check-Act record of the troubleshooting behind the SOP above, including dead ends and what didn't work. |

## Adding a new SOP

1. Add the new `.md` file to the `sops/` folder.
2. Add a row to the table above linking to it.
