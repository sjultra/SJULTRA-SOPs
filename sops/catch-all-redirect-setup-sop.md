## Process Document: Build a Domain-Scoped Catch-All/Redirect Rule with Exempted Real Recipients (Exchange Online)

**Owner:** IT/Security Administrator (Exchange Administrator role) | **Last Updated:** 2026-10-02 | **Review Cadence:** Whenever a new domain needs catch-all coverage, and after any change to the rule's exception list

### Purpose

Stand up (or extend) a mail flow rule that redirects mail addressed to non-existent recipients on specific accepted domains to a human triage mailbox, while guaranteeing that real recipients — including shared mailboxes, mail contacts, and anything created after the rule was built — are never caught.

### Scope

**In scope:** The accepted-domain prerequisite, scoping the transport rule, building the exemption group that protects real recipients, validating it with live tests, and keeping its membership current.

**Out of scope:** True catch-all on an Authoritative domain (not possible in Exchange Online — see Background); spam/phishing filtering (Microsoft Defender policies, not transport rules); on-premises/hybrid catch-all relay configurations; the automated membership sync (separate companion SOP, not yet written — see Open Items).

---

### Triage Summary (Incident Record)

This section records the incident that produced this procedure, so the reasoning behind each step is traceable. Mailbox names are aliased because this repo is public; domains are real.

#### What happened

A transport rule (`ibenit-catch-all`) redirects external mail on three domains (`ibenit.com`, `sjultra.com`, `vzxy.net`) to a triage mailbox (`user-a@sjultra.com`), tagging the subject `[O365FWD]`. It was meant to catch mail to addresses that don't exist, with real recipients exempted. Instead it was repeatedly catching mail to **real mailboxes** — first shared mailboxes, then regular user mailboxes, then a distribution group — and redirecting it. Each report arrived as "I'm not getting mail" and each was fixed one person at a time by adding that address to the rule's individual exception list.

#### What we found

1. **The exemption mechanism was broken at the source.** Exemptions came from three "allusers@" dynamic distribution groups plus a hand-maintained address list. One group filtered on the `Company` attribute, which was blank on every mailbox (zero real members); the other two filtered only on `Alias -ne $null`, with no domain scoping.
2. **The rule itself was unscoped.** It had no recipient-domain condition, so it evaluated mail for every accepted domain in the tenant, not only the three where catch-all behavior is possible.
3. **Rebuilding the dynamic groups correctly did not fix it.** We rewrote the filters to match recipient type plus domain (`EmailAddresses`), confirmed the filter text on the group, and confirmed by live PowerShell enumeration that the affected mailboxes matched it — yet mail to them was still redirected.
4. **Our first explanation did not hold up.** We suspected a proxy-address casing gap (`SMTP:` primary vs `smtp:` alias). It fit the first few failing mailboxes, but later failing mailboxes had a matching alias and failed anyway, so it cannot be the cause.
5. **A forced "re-bind" (removing and re-adding the group on the rule) was inconsistent.** It appeared to fix one mailbox, then a different mailbox failed a week later, and another failed 43 minutes after a re-bind. It is not part of this procedure.
6. **`Get-Recipient -RecipientPreviewFilter` was misleading.** It returned empty results for every wildcard (`-like`) filter we tried, including ones that worked in live mail flow. It should not be used to judge these groups.
7. **A static group was honored where the dynamic groups were not.** When the same mailbox was added to an ordinary mail-enabled security group and that group was added to the rule's exceptions, mail was delivered normally.

We did **not** identify why the rule fails to honor the dynamic groups. The procedure below is a validated workaround, not a root-cause fix.

#### What we did to fix it

- Scoped the rule to the three catch-all domains (`RecipientDomainIs`).
- Created one static mail-enabled security group (`CatchAll-Exempt-Recipients`), hidden from address lists, and populated it from a live enumeration of every user mailbox, shared mailbox, room/equipment mailbox, mail user, and mail contact on those domains (49 members).
- Added that group to the rule's `ExceptIfSentToMemberOf` list.
- Pruned stale test entries and duplicates from the individual exception list.
- Left the legacy dynamic groups on the rule for now (harmless); they can be removed after a stability period.

#### How we tested it

All tests used an external Gmail sender and message trace; a redirect shows `ibenit-catch-all` events and a `[O365FWD]` subject, a clean delivery shows neither.

| Test | Setup | Result |
|------|-------|--------|
| User mailbox A, dynamic groups only | 43 min after last rule save | Redirected |
| User mailbox A, single-member static group | 13 min after rule save | Delivered |
| User mailbox A, dynamic groups only again | 43 min after removing the static group | Redirected |
| User mailboxes A and B, 49-member static group | ~19 min after rule save; neither on any individual exception | Delivered |
| User mailbox C | Delivered, but on the individual exception list | Not counted |
| Distribution group `ops@` | Delivered, but separately listed in the rule | Weak evidence only (see Open Items) |
| Brand-new shared mailbox, not yet in the group | Baseline | Redirected (expected) |
| Same mailbox, added to the group | Tested at 6, 15, and 37 min after adding | Redirected each time |
| Same mailbox, rule re-saved at ~55 min | Next test at 3 h 7 min after adding (2 h 12 min after the re-save) | Delivered |

What this establishes: a static group's *existing* members are honored within about 20 minutes of a rule save; a *newly added* member took longer than 37 minutes and was covered by about 3 hours. We did not test between 55 minutes and 3 hours, so we do not know whether the re-save was required or the member simply took a few hours.

---

### Background

Exchange Online has no native "catch-all mailbox" feature. A redirect-style catch-all rule only works on accepted domains set to **InternalRelay** — on an **Authoritative** domain, Exchange Online rejects mail to a non-existent recipient at the SMTP level, before any transport rule runs. Confirm or set the domain type before doing anything else in this procedure.

### RACI Matrix

| Step | Responsible | Accountable | Consulted | Informed |
|------|------------|-------------|-----------|----------|
| Confirm/set accepted domain type | IT/Exchange Administrator | IT Manager | — | — |
| Scope the transport rule | IT/Exchange Administrator | IT Manager | — | — |
| Build and populate the exemption group | IT/Exchange Administrator | IT Manager | — | — |
| Verify with live tests | IT/Exchange Administrator | — | — | — |
| Add new mailboxes to the group | IT/Exchange Administrator (at provisioning) | IT Manager | — | — |

### Process Flow

```
New domain needs catch-all coverage
        |
Accepted domain type = InternalRelay? --No--> Set it (or STOP: Authoritative domains
        |                                      can never support this pattern)
        |Yes
Scope transport rule: RecipientDomainIs = [only the intended domain(s)]
        |
Create static mail-enabled security group (hidden)
        |
Populate with every real recipient on those domains
        |
Verify count + no skipped recipients
        |
Add the group to the rule's ExceptIfSentToMemberOf
        |
Wait 20-30 min --> live external test to a real mailbox --> Delivered clean? --Yes--> Done
        |No
Confirm group membership, confirm group is on the rule, wait longer, retest
        |
Still failing --> individual exception (ExceptIfSentTo) as stopgap, escalate
```

### Procedure

1. **Confirm the accepted domain type.**
   ```powershell
   Get-AcceptedDomain | Select-Object DomainName, DomainType
   ```
   If the domain is `Authoritative`, stop — a redirect rule cannot intercept mail to non-existent recipients there. Changing a domain to `InternalRelay` has broader mail-routing implications; confirm nothing else depends on `Authoritative` first:
   ```powershell
   Set-AcceptedDomain -Identity "example.com" -DomainType InternalRelay
   ```

2. **Scope the transport rule to only the domain(s) that need it.** An unscoped rule evaluates every accepted domain in the tenant.
   ```powershell
   Set-TransportRule -Identity "<rule-name>" -RecipientDomainIs @("example.com")
   ```

3. **Create the exemption group** — a *static* mail-enabled security group. Do not use a dynamic distribution group here (see Known Limitations).
   ```powershell
   $group = "CatchAll-Exempt-Recipients"
   New-DistributionGroup -Name $group -Type Security
   Set-DistributionGroup -Identity $group -HiddenFromAddressListsEnabled $true
   ```

4. **Populate it from a live enumeration** of every real recipient on the covered domains, then verify nothing was skipped.
   ```powershell
   $domains = @("example.com")

   $recipients = Get-Recipient -ResultSize Unlimited -RecipientTypeDetails UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox,MailUser,MailContact |
       Where-Object { $a = $_.EmailAddresses; $domains | Where-Object { $a -like "*@$_" } }

   foreach ($r in $recipients) {
       Add-DistributionGroupMember -Identity $group -Member $r.PrimarySmtpAddress -ErrorAction SilentlyContinue
   }

   $recipients.Count
   $members = (Get-DistributionGroupMember -Identity $group -ResultSize Unlimited).PrimarySmtpAddress
   $recipients | Where-Object { $members -notcontains $_.PrimarySmtpAddress } | Select-Object Name, PrimarySmtpAddress, RecipientTypeDetails
   ```
   The last command must print nothing. `-ErrorAction SilentlyContinue` hides failed adds, so this check is required, not optional.

5. **Add the group to the rule's exceptions.**
   ```powershell
   $rule = Get-TransportRule -Identity "<rule-name>"
   Set-TransportRule -Identity "<rule-name>" -ExceptIfSentToMemberOf ($rule.ExceptIfSentToMemberOf + $group)
   (Get-TransportRule -Identity "<rule-name>").WhenChanged
   ```
   Note the `WhenChanged` time. Group entries are stored as full addresses (e.g. `CatchAll-Exempt-Recipients@tenant.onmicrosoft.com`), so if you ever remove one, match with a wildcard (`-notlike "CatchAll-Exempt-Recipients*"`); an exact-name match silently removes nothing.

6. **Verify with live external tests.** Wait at least **20–30 minutes** after any save to the rule, then send an external test to a real user mailbox and a real shared mailbox that are **not** on the rule's individual exception list. Check the message trace: delivery should show no `ibenit-catch-all`-style rule events and no `[O365FWD]`-style subject tag. A mailbox on the individual exception list proves nothing — check `(Get-TransportRule -Identity "<rule-name>").ExceptIfSentTo` first.

7. **Keep membership current.** Newly created mailboxes are not covered until they are in the group, and our tests showed a delay of up to a few hours after adding. Therefore:
   - **At provisioning time**, add every new mailbox on a covered domain to the group:
     ```powershell
     Add-DistributionGroupMember -Identity "CatchAll-Exempt-Recipients" -Member new.mailbox@example.com
     ```
   - **Safety net**: re-run the step 4 population block on a schedule (manually until the automated sync SOP exists).
   - Until a new mailbox is covered, its mail is redirected to the triage mailbox, not lost.

8. **Individual exception as an always-available stopgap.** If a specific person is blocked right now, don't wait on the group:
   ```powershell
   $rule = Get-TransportRule -Identity "<rule-name>"
   Set-TransportRule -Identity "<rule-name>" -ExceptIfSentTo ($rule.ExceptIfSentTo + "person@example.com")
   ```
   Remove the entry once the group covers them, so the individual list doesn't grow back into the thing this procedure replaces.

### Known Limitations

- **Do not use dynamic distribution groups as the exemption mechanism for this rule.** In this tenant the rule did not reliably honor them, even when their filters matched the affected mailboxes by direct enumeration, and a static group worked. We do not know why. If you find an explanation, update this SOP.
- **Saving the rule restarts the wait.** Do not make further changes to the rule while a test is in progress, or you won't know what the result reflects.
- **`Get-Recipient -RecipientPreviewFilter` is not a valid test.** It returns empty results for wildcard filters regardless of whether the group works in mail flow.
- **Message trace can show one message as two records** (joined by `TRANSFER` events). We saw it when the test address was typed in a different case than the mailbox's primary address; that is our best guess at the trigger, not something we confirmed. The record that ends in `Deliver` is the real one; the stub that shows "Not delivered" is an artifact.

### Open Items

- **Automated membership sync** is not built. A scheduled job (Azure Automation runbook, not a machine that has to stay on) should run step 4 on an interval. Companion SOP to follow once it has been built and tested.
- **Group recipients** (distribution lists and Microsoft 365 Groups addressed directly, e.g. `info@`) are not validated. The one test (`ops@`) was delivered but is separately listed in the rule, so it proves little. Before relying on this for group addresses, test an unlisted distribution list; if it is redirected, exempt group addresses by address using `ExceptIfSentTo` instead of by membership.
- **Re-save vs. latency** for newly added members is unresolved (see the last two rows of the test table).
- **Legacy dynamic groups** still on the rule should be removed after a stability period.

*Last updated: 2026-10-02*
