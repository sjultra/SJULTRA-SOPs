## Process Document: Reclaim a Departed Employee's Email Address as a Distribution List (Hybrid AD + Exchange Online)

**Owner:** Richard Rives | **Last Updated:** 2026-09-03 | **Review Cadence:** Annually, or after any change to Entra Connect write-back status

### Purpose

Free up a shared business email address (e.g. `info@domain.com`) that currently lives on a departed employee's mailbox, and hand it to a distribution list instead — so no former employee's account has to stay active just to keep a business address working, and current staff can receive and send as that address without a shared mailbox to manage. Written for environments where on-prem Active Directory is synced to Microsoft Entra ID via Entra Connect but **password/attribute write-back is not configured** — meaning on-prem AD is the sole source of authority for mail attributes on synced objects.

### Scope

**In scope:** Identifying and freeing a single SMTP address from a directory-synced mailbox; making that change on-premises when Exchange Online refuses it; creating a cloud-only distribution group to take over the address; granting Send As; opening the group to external senders when the address needs to receive mail from outside the organization; safely retiring the departed employee's on-prem account without data loss.

**Out of scope:** Full mailbox export or legal-hold procedures; on-prem Exchange hybrid administration beyond basic AD attribute edits; cleanup of unrelated legacy aliases discovered sitting on the same mailbox (see Exceptions) — those get inventoried and flagged for a separate decision with the client, not resolved in this pass.

### RACI Matrix

| Step | Responsible | Accountable | Consulted | Informed |
|------|------------|-------------|-----------|----------|
| Inventory current address ownership | IT Administrator | IT Manager | — | — |
| Decide fate of the rest of the mailbox (other aliases) | IT Administrator | IT Manager | Client/Customer Stakeholder | — |
| Free the address (cloud attempt, then on-prem if blocked) | IT Administrator | IT Manager | — | — |
| Force/await directory sync | IT Administrator | — | — | — |
| Create distribution group, grant Send As | IT Administrator | IT Manager | Receiving employee(s) | — |
| Open group to external senders (if required) | IT Administrator | IT Manager | — | — |
| Test and verify | IT Administrator | — | Receiving employee(s) | — |
| Retire departed employee's account | IT Administrator | IT Manager | — | Client/Customer Stakeholder |

### Process Flow

```
Confirm the address and current owner (Get-Mailbox ... EmailAddresses)
        |
Decide: does anything else on this mailbox need preserving?
        |
Attempt to free the address in Exchange Online
        |
Fails with "out of the current user's write scope"? --Yes--> Object is directory-synced;
        |No                                                   mail attributes are on-prem-authoritative.
        |                                                      Make the edit in AD instead (Set-ADUser,
        |                                                      elevated PowerShell, ActiveDirectory module).
        v                                                              |
Address freed in Exchange Online <------------------------------------+
   (force delta sync from the Entra Connect server, or wait ~30 min for the default cycle)
        |
Create distribution group with the freed address as primary SMTP
        |
Add member(s); grant Send As
        |
Address must accept external mail? --Yes--> Set RequireSenderAuthenticationEnabled = $false
        |No
        v
Test both directions (external send-in, Send As send-out)
        |
Retire departed employee's account
   - Other addresses/history still live on the mailbox? --> Disable AD account, do NOT delete
   - Nothing else of value on it? --> Safe to fully decommission
        |
Flag any unrelated legacy aliases found for a separate client conversation
```

### Detailed Steps

#### Step 1: Inventory current address ownership
- **Who**: IT Administrator
- **When**: Before touching anything
- **How**: `Get-Mailbox <alias> | fl PrimarySmtpAddress,EmailAddresses` in Exchange Online PowerShell. Read the full list, not just the primary — mailboxes that have existed for years frequently accumulate unrelated aliases (former coworkers, name-spelling variants, secondary/typo-catch domains) that have nothing to do with the address you're actually here for.
- **Output**: Confirmed current owner and full address list for the mailbox.

#### Step 2: Decide what happens to the rest of the mailbox
- **Who**: IT Administrator, with IT Manager and Client/Customer Stakeholder input
- **When**: After Step 1, before any address changes
- **How**: If Step 1 turned up only the one address in question, proceed directly. If it turned up other addresses still presumably in active use (other employees, other name variants), do **not** delete or convert the mailbox in this pass — only the specific address being reclaimed gets touched. Note everything else for a follow-up conversation (see Exceptions).
- **Output**: A clear boundary on what this pass will and won't touch.

#### Step 3: Attempt to free the address in Exchange Online
- **Who**: IT Administrator
- **When**: After Step 2
- **How**: `Set-Mailbox` has **no** `-PrimarySmtpAddress` parameter (that exists on `Set-MailUser`/`Set-RemoteMailbox`, not `Set-Mailbox`) — use the `-EmailAddresses` hashtable syntax instead, with a capital `SMTP:` prefix marking the new primary:
  ```powershell
  Set-Mailbox <alias> -EmailAddresses @{Add="SMTP:<newprimary>@domain.com"}
  ```
  In the Exchange admin center, the equivalent is Recipients > Mailboxes > the mailbox > Email addresses — but note the address currently marked as the bold/default reply address can't be deleted directly; you have to promote a different address to primary first, which demotes the old one to a plain (deletable) secondary.
- **Output**: Either success (skip to Step 5), or the specific error in Step 4.

#### Step 4: Handle a directory-sync write-scope block
- **Who**: IT Administrator
- **When**: If Step 3 fails with `...it's out of the current user's write scope. ... This action should be performed on the object in your on-premises organization.`
- **How**: This means the mailbox object is directory-synced and mail attributes (`proxyAddresses`, `mail`) are on-prem-authoritative — Exchange Online will not accept the write, by design, because Entra Connect would just overwrite it back on the next sync anyway. Make the change on the domain controller instead, using the `ActiveDirectory` PowerShell module (`Import-Module ActiveDirectory`; this and `ExchangeOnlineManagement` can both be loaded in the same PowerShell session with no conflict):
  ```powershell
  Set-ADUser <alias> `
    -Remove @{proxyAddresses=@("SMTP:<oldprimary>@domain.com","smtp:<newprimary>@domain.com")} `
    -Add    @{proxyAddresses=@("smtp:<oldprimary>@domain.com","SMTP:<newprimary>@domain.com")}
  ```
  Check the existing `proxyAddresses` list first (`Get-ADUser <alias> -Properties proxyAddresses | Select -ExpandProperty proxyAddresses`) — the new primary may already be sitting there as a lowercase secondary alias, in which case you're only flipping case on two existing entries, not adding a new one.
- **Output**: The change applied on-prem, pending sync.

#### Step 5: Resolve "Insufficient access rights" if it appears
- **Who**: IT Administrator
- **When**: If the on-prem `Set-ADUser` in Step 4 itself fails
- **How**: There are three distinct causes that produce a similar-looking error — check in this order:
  1. **RBAC parameter scoping** (Exchange Online side only) — shows as `A parameter cannot be found that matches parameter name '...'`. Not a real permissions error; it usually means the wrong parameter was used in the first place (see Step 3's note on `-PrimarySmtpAddress`).
  2. **Exchange-specific AD ACLs** — even a genuine Domain Admin account can be blocked from writing mail attributes if on-prem Exchange setup locked `proxyAddresses`/`mail` down to `Exchange Recipient Administrators` or similar. Check with `dsacls <DN>` on the target object; look for whether the account's actual groups (not just what `Get-ADUser -Properties MemberOf` shows) carry write rights.
  3. **UAC token filtering** — the most common cause in practice. Confirm with `whoami /groups | findstr /i "domain admins"` (or the relevant privileged group). If it shows `Group used for deny only`, the current PowerShell window isn't running elevated, so Windows stripped the privileged group from the active token even though the account genuinely has the membership. Fix: close the window and reopen PowerShell **as Administrator**, reload the `ActiveDirectory` module, and retry — no group membership changes needed.
- **Output**: Identified root cause and a working elevated session.

#### Step 6: Force or await directory sync
- **Who**: IT Administrator
- **When**: After Step 4 succeeds
- **How**: If you have access to the Entra Connect sync server, force it rather than waiting:
  ```powershell
  Import-Module ADSync
  Start-ADSyncSyncCycle -PolicyType Delta
  ```
  If you don't have access to that server, no action is required — the default scheduler runs a delta sync automatically every 30 minutes on its own.
- **Output**: On-prem change replicated to Entra ID / Exchange Online.

#### Step 7: Verify the address is free in Exchange Online
- **Who**: IT Administrator
- **When**: A few minutes after Step 6
- **How**: `Get-Mailbox <alias> | fl PrimarySmtpAddress,EmailAddresses` — confirm the target address no longer appears anywhere in the list (not even as a lowercase secondary).
- **Output**: Confirmed the address is unclaimed and ready to reassign.

#### Step 8: Create the distribution group and grant Send As
- **Who**: IT Administrator
- **When**: Immediately after Step 7
- **How**:
  ```powershell
  New-DistributionGroup -Name "<GroupName>" -DisplayName "<GroupName>" -PrimarySmtpAddress <address> -Members <useralias>
  Add-RecipientPermission -Identity "<GroupName>" -Trustee <useralias> -AccessRights SendAs
  ```
  Plain distribution groups support Send As in Exchange Online despite having no mailbox of their own — no mail-enabled security group required. Add any secondary domain variants of the address the same way it was inventoried in Step 1 (`Set-DistributionGroup -Identity "<GroupName>" -EmailAddresses @{Add="smtp:<variant>@domain2.com"}`), so nothing that used to land on the old mailbox starts bouncing.
- **Output**: A working distribution list receiving mail and able to send as the address.

#### Step 9: Open the group to external senders, if required
- **Who**: IT Administrator
- **When**: Before considering the cutover complete, if the address is customer/vendor-facing
- **How**: New distribution groups in Exchange Online default to internal-senders-only. Check and clear it if the address needs to take mail from outside the org:
  ```powershell
  Get-DistributionGroup "<GroupName>" | fl RequireSenderAuthenticationEnabled
  Set-DistributionGroup "<GroupName>" -RequireSenderAuthenticationEnabled $false
  ```
- **Output**: External mail delivers instead of bouncing with a delivery-restriction NDR.

#### Step 10: Test both directions
- **Who**: IT Administrator, with the receiving employee(s)
- **When**: After Steps 8–9
- **How**: Send an external test message to the address and confirm it lands in the member's inbox. Have the member compose a new message with From set to the address and confirm it sends correctly — Send As can take up to an hour to fully propagate, so don't troubleshoot prematurely if it doesn't work immediately.
- **Output**: Confirmed working in both directions.

#### Step 11: Retire the departed employee's account
- **Who**: IT Administrator
- **When**: After Step 10 confirms the cutover
- **How**: If Step 2 found other addresses or mail history on the mailbox still worth keeping, **disable** the on-prem AD account rather than deleting it — deleting it would sync a delete up to Entra ID and remove the mailbox (and everything on it) along with the account. If nothing else of value remains, it's safe to fully decommission. Either way, confirm the account's license has been reclaimed.
- **Output**: Departed employee's account no longer active; nothing else lost in the process.

### Exceptions and Edge Cases

| Scenario | What to Do |
|----------|-----------|
| `Set-Mailbox -PrimarySmtpAddress` throws "parameter cannot be found" | That parameter doesn't exist on `Set-Mailbox`. Use `-EmailAddresses @{Add="SMTP:..."}` instead (see Step 3). |
| EAC shows the address as deletable but it's still there after "removing" it | The bold/default reply address can't be deleted directly — promote a different address to primary first, which demotes the old one to a deletable secondary. |
| "Out of the current user's write scope" in Exchange Online | The object is directory-synced; make the change on-prem instead (Step 4). |
| Domain Admin account still gets "Insufficient access rights" on-prem | Don't assume it's a real permissions gap — check `whoami /groups` for "Group used for deny only" (UAC filtering) before escalating anything (Step 5). |
| No access to the Entra Connect sync server | Not required — the default 30-minute delta sync cycle will pick up the change on its own (Step 6). |
| The source mailbox turns out to already be a Shared mailbox, or gets converted mid-process | No impact on this procedure — Shared vs. regular mailbox doesn't change address-removal mechanics, and Shared consumes no license, which is a net positive if the mailbox needs to stick around for other addresses. |
| The mailbox being freed also holds several unrelated legacy aliases (other people's names, other name-spelling variants, a secondary/typo-catch domain) | Do not clean those up in the same pass. Capture the full list from Step 1 and raise it with the client/customer as a separate decision — some may represent other departed staff whose mail is silently landing nowhere anyone checks. |
| Object shows `{This object is protected from inheriting permissions from the parent}` in `dsacls` output | Sign the object has `adminCount=1` — it was (or still is) a member of a privileged AD group at some point, which is why its ACL looks unusually locked-down. Informational, not necessarily a blocker, but worth noting for the client if unexpected. |

### Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Mail downtime during cutover | 0 minutes (address always owned by something) | Compare timestamp address left old owner vs. joined new DL |
| Time from start to verified working (both directions) | < 1 business day, excluding sync-server access blockers | Ticket/session timestamps |
| NDRs generated during cutover window | 0 | Message trace on the address across the cutover |
| Legacy aliases discovered and flagged (not silently dropped) | 100% | Compare final client-facing inventory to Step 1's full `EmailAddresses` capture |

### Related Documents

- [Manage permissions for recipients in Exchange Online (Send As / Send on Behalf)](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Add-RecipientPermission (PowerShell reference)](https://learn.microsoft.com/en-us/powershell/module/exchange/add-recipientpermission)
- [Install and Connect to Exchange Online PowerShell](exchange-online-powershell-setup.md)
- Example engagement this SOP was written from: a departed employee's on-prem-synced mailbox holding a customer-facing `info@` address, in a hybrid AD/Exchange Online environment with Entra Connect write-back not yet configured — client name omitted per internal documentation policy.
