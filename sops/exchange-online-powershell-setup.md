# SOP: Install and Connect to Exchange Online PowerShell

**Purpose:** Standard steps for installing the Exchange Online PowerShell module and connecting to a Microsoft 365 tenant, for troubleshooting mailbox/calendar configuration issues (e.g., online meeting provider settings).

**Scope:** Any team member with an admin account needing Exchange Online PowerShell access from a Windows PC.

---

## Prerequisites

- Windows PowerShell 5.1 or PowerShell 7
- An account with Exchange Online admin permissions
- MFA method available (Authenticator app, SMS, etc.)

---

## Step 1: Install the Exchange Online PowerShell Module

Open PowerShell (as Administrator if possible) and run:

```powershell
Install-Module -Name ExchangeOnlineManagement -Force
```

If prompted about the NuGet provider, type **Y** and press Enter:

```
NuGet provider is required to continue
[Y] Yes  [N] No  [S] Suspend  [?] Help (default is "Y"): Y
```

If you can't run PowerShell as Administrator, install for your user only:

```powershell
Install-Module -Name ExchangeOnlineManagement -Force -AllowClobber -Scope CurrentUser
```

**Verify the install:**

```powershell
Get-Module -ListAvailable ExchangeOnlineManagement
```

If this returns nothing, the install did not complete — rerun the install command above.

---

## Step 2: Fix Execution Policy (if blocked)

If importing the module fails with an error like:

```
File ...ExchangeOnlineManagement.psm1 cannot be loaded because running scripts is disabled on this system.
```

Check current policy:

```powershell
Get-ExecutionPolicy -List
```

If all scopes show `Undefined` (defaults to Restricted), fix it for your account only — no admin rights required:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force
```

---

## Step 3: Import the Module

```powershell
Import-Module ExchangeOnlineManagement
```

If this still fails, rerun with `-Verbose` to see the real underlying error (rather than a generic "could not be loaded" message):

```powershell
Import-Module ExchangeOnlineManagement -Verbose
```

---

## Step 4: Connect to Exchange Online

**Standard method (recommended)** — opens a browser/sign-in window that handles password + MFA automatically:

```powershell
Connect-ExchangeOnline -UserPrincipalName admin@sjultra.com
```

**Device code method** — only needed if the machine has no browser access (e.g., headless/remote session):

```powershell
Connect-ExchangeOnline -Device
```

This prints a code and a URL (`https://microsoft.com/devicelogin`). Open that URL on any device with a browser, enter the code, and sign in there to complete the connection.

---

## Step 5: Verify the Connection

```powershell
Get-AcceptedDomain
```

If this returns results with no errors, you're connected.

---

## Step 6: Disconnect When Finished

Always disconnect when done to avoid using up available sessions:

```powershell
Disconnect-ExchangeOnline -Confirm:$false
```

---

## Troubleshooting Reference

| Error | Cause | Fix |
|---|---|---|
| `NuGet provider is required to continue` | Missing NuGet provider | Type `Y` when prompted, or run `Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force` |
| `...could not be loaded. For more information, run 'Import-Module...'` | Generic autoload failure | Run `Import-Module ExchangeOnlineManagement -Verbose` to see the real error |
| `cannot be loaded because running scripts is disabled on this system` | Execution policy set to Restricted | `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force` |

---

*Last updated: 2026-07-22*
