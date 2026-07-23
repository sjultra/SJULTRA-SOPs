# SOP: Fix User Unable to Schedule Teams Meetings in Outlook/OWA

**Purpose:** Diagnose and resolve the error "You do not have permissions to invite others. Please contact your administrator." when a user tries to add a Teams meeting to a calendar invite in OWA or Outlook (Windows/Mac).

**Scope:** Any user who can otherwise sign in and use email/calendar normally, but gets blocked specifically when adding a Teams meeting to an event.

---

## Prerequisites

- Exchange Online PowerShell connected (see the separate [Install and Connect to Exchange Online PowerShell](exchange-online-powershell-setup.md) SOP for that setup)
- Microsoft Teams PowerShell module installed and connected (see Step 0 below if not already set up)

---

## Step 0: Install and Connect Microsoft Teams PowerShell (if not already done)

```powershell
Install-Module -Name MicrosoftTeams -Force
Connect-MicrosoftTeams
```

This opens a browser sign-in window for the admin account (MFA included), same as Exchange Online.

---

## Step 1: Check the Mailbox's Default Online Meeting Provider

The most common cause: the affected user's mailbox is configured to use a different meeting provider (e.g., Zoom) instead of Teams.

```powershell
Get-MailboxCalendarConfiguration -Identity user@domain.com | Format-List DefaultOnlineMeetingProvider
```

If this does **not** return `TeamsForBusiness`, fix it:

```powershell
Set-MailboxCalendarConfiguration -Identity user@domain.com -DefaultOnlineMeetingProvider TeamsForBusiness
```

Have the user fully sign out of OWA (clear browser cache/cookies) and restart Outlook, then test again.

---

## Step 2: Check Whether the Account Is Misconfigured as a Resource Account

If Step 1 doesn't resolve it, or if the user is also missing from the Teams admin center **Users** list entirely, check how Teams is interpreting the account type:

```powershell
Get-CsOnlineUser -Identity user@domain.com | Format-List DisplayName, InterpretedUserType, TeamsUpgradeEffectiveMode, FeatureTypes, AccountEnabled
```

A normal user should show `InterpretedUserType` as `PureOnlineTeamsOnlyUser` (or similar normal user type).

If it instead shows **`PureOnlineApplicationInstance`**, the account has been misconfigured as a Teams resource account (the object type used for Auto Attendants, Call Queues, or Teams Phone Agents) — this explains both the missing-from-admin-center symptom and the inability to schedule meetings normally.

---

## Step 3: Fix a Misconfigured Resource Account

1. In the Microsoft 365 admin center, go to **Users → Active users**, select the affected user, and open the **Licenses and apps** tab.
2. Look for **Microsoft Teams Phone Resource Account** in their assigned licenses. This license is meant only for actual resource accounts (Auto Attendants/Call Queues) — if it's present on a real user's account, that's the misconfiguration.
3. Uncheck it and save.
4. Wait a few minutes for the change to sync, then re-check:

```powershell
Get-CsOnlineUser -Identity user@domain.com | Format-List InterpretedUserType
```

It should now return a normal user type (e.g., `PureOnlineTeamsOnlyUser`).

**Caution:** Do not run any `Remove-CsOnlineApplicationInstance` or similar resource-account deletion cmdlets against a real user's account. Those commands are meant for actual resource accounts and can have unintended effects. Removing the mistaken license is the correct and sufficient fix in this scenario.

---

## Step 4: Verify

1. Confirm `InterpretedUserType` is back to normal (Step 3).
2. Confirm `DefaultOnlineMeetingProvider` is `TeamsForBusiness` (Step 1).
3. Confirm the user now appears in the Teams admin center **Users** list.
4. Have the user try scheduling a Teams meeting again in both OWA and Outlook desktop/Mac.

---

## Troubleshooting Reference

| Symptom | Cause | Fix |
|---|---|---|
| "You do not have permissions to invite others" in OWA/Outlook | Mailbox's default online meeting provider isn't Teams | `Set-MailboxCalendarConfiguration -Identity user@domain.com -DefaultOnlineMeetingProvider TeamsForBusiness` |
| User missing from Teams admin center Users list | Account misconfigured as a resource account (`PureOnlineApplicationInstance`) | Remove the mistakenly assigned **Microsoft Teams Phone Resource Account** license via Licenses and apps |
| Both of the above together | Both issues can co-occur — check both before assuming a fix is complete | Run through Steps 1–3 in order |

---

*Last updated: 2026-07-23*
