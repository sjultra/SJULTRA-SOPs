## Process Document: Build a Domain-Scoped Catch-All/Redirect Rule with Self-Maintaining Exceptions (Exchange Online)

**Owner:** IT/Security Administrator (Exchange Administrator role) | **Last Updated:** 2026-09-25 | **Review Cadence:** Whenever a new domain needs catch-all coverage, or immediately after any edit to a group filter this procedure created

### Purpose

Stand up (or extend) a mail flow rule that redirects mail addressed to non-existent recipients on specific accepted domains to a human triage mailbox, while guaranteeing real recipients — including shared mailboxes, mail contacts, and anything created after the rule was built — are never caught, with no ongoing manual exception-list maintenance.

### Scope

**In scope:** Configuring the accepted-domain type prerequisite, scoping the transport rule to the correct domains, building self-maintaining dynamic exception groups, validating them safely, and the mandatory re-bind step after any future filter edit.

**Out of scope:** True catch-all across an Authoritative domain (not possible in Exchange Online — see Background); spam/phishing filtering (handled by Microsoft Defender policies, not transport rules); on-premises/hybrid catch-all relay configurations.

### Background

Exchange Online has no native "catch-all mailbox" feature. A redirect-style catch-all rule only works on accepted domains set to **InternalRelay** — on an **Authoritative** domain, EXO rejects mail to any non-existent recipient at the SMTP level, before any transport rule ever runs, so a rule can never intercept it. Confirm or set the domain type before doing anything else in this procedure.

### RACI Matrix

| Step | Responsible | Accountable | Consulted | Informed |
|------|------------|-------------|-----------|----------|
| Confirm/set accepted domain type | IT/Exchange Administrator | IT Manager | — | — |
| Scope the transport rule | IT/Exchange Administrator | IT Manager | — | — |
| Build dynamic exception group(s) | IT/Exchange Administrator | IT Manager | — | — |
| Validate group membership | IT/Exchange Administrator | IT Manager | — | — |
| Re-bind group after any filter edit | IT/Exchange Administrator | IT Manager | — | Affected recipients (if previously blocked) |
| Verify with live test | IT/Exchange Administrator | — | — | — |

### Process Flow

```
New domain needs catch-all coverage
        |
Accepted domain type = InternalRelay? --No--> Set it (or STOP: Authoritative domains
        |                                      can never support this pattern)
        |Yes
Scope transport rule: RecipientDomainIs = [only the intended domain(s)]
        |
Build one Dynamic Distribution Group per domain
  (RecipientType filter + EmailAddresses -like, both smtp:/SMTP: cases)
        |
Validate membership via Get-Mailbox | Where-Object
  (never trust Get-Recipient -RecipientPreviewFilter for this — see Known Limitations)
        |
Add group(s) to rule's ExceptIfSentToMemberOf
        |
Send live external test mail --> Delivered clean? --Yes--> Done
        |No
Remove the group from ExceptIfSentToMemberOf, then add it back
  (forces the rule to re-bind to the group's current filter)
        |
Re-test --> Delivered clean? --Yes--> Done
        |No
Add the specific recipient to ExceptIfSentTo as an immediate stopgap,
escalate for deeper investigation
```

### Procedure

1. **Confirm the accepted domain type.**
   ```powershell
   Get-AcceptedDomain | Select-Object DomainName, DomainType
   ```
   If the domain you need catch-all coverage for is `Authoritative`, stop — a redirect rule cannot intercept mail to non-existent recipients there. Change it to `InternalRelay` first if that's genuinely intended (this has broader mail-routing implications — confirm no other dependency needs `Authoritative` before changing it):
   ```powershell
   Set-AcceptedDomain -Identity "example.com" -DomainType InternalRelay
   ```

2. **Scope the transport rule to only the domain(s) that need this.** Never leave the rule's recipient condition open to the whole tenant — an unscoped rule evaluates against every accepted domain, including ones where it has no business running.
   ```powershell
   Set-TransportRule -Identity "<rule-name>" -RecipientDomainIs @("example.com")
   ```
   (Add more domains to the array as needed; each additional domain gets its own exception group in the next step.)

3. **Build one Dynamic Distribution Group per protected domain**, using this exact filter template — it's the only version confirmed to reliably cover both a mailbox's primary address and any secondary alias, regardless of Exchange's internal address-prefix casing:
   ```powershell
   New-DynamicDistributionGroup -Name "all-<domain-short-name>" -RecipientFilter "(RecipientType -eq 'UserMailbox' -or RecipientType -eq 'MailUser' -or RecipientType -eq 'MailContact') -and ((EmailAddresses -like 'smtp:*@example.com') -or (EmailAddresses -like 'SMTP:*@example.com'))"
   ```
   **Do not simplify this to one `-like` clause.** Exchange stores a mailbox's primary address with an uppercase `SMTP:` prefix and any secondary aliases with lowercase `smtp:`. A mailbox whose only address in the domain is its primary (no secondary alias) will silently fall through a filter that only checks one casing — this was the actual root cause of two separate real incidents.
   **Do not filter on attributes that require manual per-mailbox tagging** (e.g. the `Company` field, a custom attribute, or an OU/container scope that new mailboxes might not land in by default) — that reintroduces exactly the manual-maintenance problem this pattern exists to eliminate. Only `RecipientType` + `EmailAddresses` should gate membership.

4. **Validate the group's actual membership before trusting it — using a live simulation, not the built-in preview cmdlet:**
   ```powershell
   Get-Mailbox -ResultSize Unlimited | Where-Object { $_.EmailAddresses -like "*@example.com*" } | Select-Object Name, PrimarySmtpAddress, RecipientTypeDetails
   ```
   Cross-check that every real mailbox and shared mailbox you expect appears in this list. See **Known Limitations** below for why `Get-Recipient -RecipientPreviewFilter` should not be used for this check.

5. **Add the group(s) to the rule's exception condition:**
   ```powershell
   $rule = Get-TransportRule -Identity "<rule-name>"
   Set-TransportRule -Identity "<rule-name>" -ExceptIfSentToMemberOf ($rule.ExceptIfSentToMemberOf + "all-<domain-short-name>")
   ```

6. **Send a live external test message** to a real mailbox on the protected domain (ideally a shared mailbox, since those are most likely to be missed) and confirm clean delivery via message trace — no redirect, no `[O365FWD]`-style subject tagging, no `Fail`/`Drop` events in the trace detail.

7. **Mandatory step after any future edit to a group's `RecipientFilter`:** the rule does **not** automatically pick up the change, even after several days — confirmed in production. Force it to re-bind by removing the group from the exception list and immediately adding it back in the same session:
   ```powershell
   $rule = Get-TransportRule -Identity "<rule-name>"
   $without = $rule.ExceptIfSentToMemberOf | Where-Object { $_ -ne "all-<domain-short-name>" }
   Set-TransportRule -Identity "<rule-name>" -ExceptIfSentToMemberOf $without
   Set-TransportRule -Identity "<rule-name>" -ExceptIfSentToMemberOf ($without + "all-<domain-short-name>")
   ```
   Re-test with a live external message afterward. Treat this re-bind as a required part of *editing the filter*, not an optional troubleshooting step to try only if something looks broken.

8. **Keep an individual bypass ready as an always-available immediate stopgap.** If any one person or shared mailbox is actively blocked — during initial rollout, after a filter edit, or for any other reason — add them directly rather than waiting on the group:
   ```powershell
   $rule = Get-TransportRule -Identity "<rule-name>"
   Set-TransportRule -Identity "<rule-name>" -ExceptIfSentTo ($rule.ExceptIfSentTo + "person@example.com")
   ```
   Once the group is confirmed working for that person (step 6/7), the individual entry can be removed — but never leave someone waiting on a group fix, even one you're confident is correct.

### Known Limitations

- **`Get-Recipient -RecipientPreviewFilter` is not reliable for validating these groups.** In practice it returned empty results for every `-like`-based filter tested (against `EmailAddresses` and even the single-valued `PrimarySmtpAddress`), while the exact same logic run live via `Get-Mailbox | Where-Object` returned correct results, and the group worked correctly in actual mail flow. Use step 4's live simulation instead — never conclude a group is broken based on this cmdlet alone.
- **A group's `RecipientFilter` change does not automatically propagate to a transport rule already referencing it**, even after multiple days. This is not a short propagation delay — treat step 7's remove/re-add as mandatory, every time, immediately after any filter edit.
- **Address-prefix casing (`SMTP:` vs `smtp:`) is a real, silent failure mode**, not a theoretical edge case — it has caused two separate real incidents. Always include both cases in the filter template in step 3.

*Last updated: 2026-09-25*
