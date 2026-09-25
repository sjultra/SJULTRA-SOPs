# Case Study (PDCA): Real Mailboxes Redirected by the `ibenit-catch-all` Transport Rule

**Affected account context:** Started as a report of shared mailboxes on `sjultra.com` getting external mail redirected instead of delivered; widened partway through to include a regular user mailbox, which surfaced a second, unrelated root cause.

---

## Plan

**Symptom:** External mail to shared mailboxes on `sjultra.com` (and later a regular user mailbox) was getting tagged `[O365FWD]` in the subject and redirected to `user-a@sjultra.com` instead of landing in the actual mailbox.

**Background:** `ibenit-catch-all` is a mail flow (transport) rule that redirects mail from external senders to `user-a@sjultra.com` for manual triage, with a list of exceptions for real recipients who should just receive their mail normally. Exception handling relied on two mechanisms: a hand-maintained list of individual addresses, and three "allusers@" groups (one per domain: `ibenit.com`, `sjultra.com`, `vzxy.net`). New mailboxes — especially shared mailboxes, which don't go through the same onboarding checklist as user accounts — were never being added to either, so they fell through and got redirected.

**Initial hypothesis:** The exception groups were static distribution lists that simply weren't being kept up to date with new shared mailboxes.

A look at the tenant's accepted domains explained the domain scope for this rule: only `ibenit.com`, `sjultra.com`, and `vzxy.net` are set to **InternalRelay** (the rest are **Authoritative**). Only InternalRelay domains let Exchange Online accept mail to an address that doesn't match a real recipient — which is the only reason a catch-all/redirect rule can function at all. The rule itself had no recipient-domain condition, so in principle it was evaluating against all 11 accepted domains in the tenant, not just the three where catch-all logic makes sense.

---

## Do

Steps taken, in order:

1. **Inspected the rule in EAC**, then pulled its live definition via Exchange Online PowerShell (`Get-TransportRule`) to see the actual conditions, actions, and exceptions rather than the summarized EAC text.
2. **Checked the tenant's accepted domains** to understand why only three domains were in scope for catch-all behavior (InternalRelay vs Authoritative, above).
3. **Inspected the three `allusers@` exception groups** and found they were already *Dynamic Distribution Groups* (not static lists, as first assumed) — but with broken filters:
   - `all-sjultra` filtered on `Company -eq 'SJULTRA'`. A scan of every `sjultra.com` mailbox showed the `Company` attribute was blank on **all of them** — the group had zero real members, so it was providing no protection at all. Initial hypothesis (dead end) invalidated.
   - `all-ibenit` / `all-vzxy` filtered only on `Alias -ne $null`, with no domain restriction — so broad they matched almost the entire tenant, meaning they weren't actually scoped to their domain (not the cause of the bug, but not doing their job either).
4. **Built and tested new domain-based dynamic groups** (`RecipientType` + `EmailAddresses -like` on the domain) as a control, validated their membership against a reliable method (`Get-Mailbox | Where-Object`, since `Get-Recipient -RecipientPreviewFilter` proved unreliable for wildcard matches on `EmailAddresses` in this tenant — a dead end worth noting so it isn't retried), then folded that same corrected logic into the original `all-*` groups instead of keeping duplicate infrastructure.
5. **Added a recipient-domain condition** (`RecipientDomainIs`) to the rule itself, so it can only ever evaluate against `ibenit.com` / `sjultra.com` / `vzxy.net`, never the other 8 (Authoritative) accepted domains in the tenant.
6. **A later report** (`user-b@sjultra.com` unable to send/receive) led to a second, more subtle finding: his mailbox's *only* address in the domain was the primary `SMTP:user-b@sjultra.com` (uppercase prefix), with no secondary alias. Our filter's `EmailAddresses -like 'smtp:*@sjultra.com'` clause only matched the **lowercase** `smtp:` prefix Exchange uses for secondary aliases — so anyone without a lowercase alias in that domain, including User B, was still falling through. Confirmed by comparing against `user-c@sjultra.com` (a working case), who has two lowercase aliases and was matching by coincidence, not because the domain logic was fully correct.
7. **Fixed the filter to match both prefixes** (`smtp:` and `SMTP:`), confirmed the corrected filter text was saved on the group — but a retest minutes later still showed User B getting redirected. Initial hypothesis was propagation lag; added him to the individual `ExceptIfSentTo` list as an immediate stopgap while that was investigated further.
8. **A second, independent recurrence** three days later: a different shared mailbox (`user-d@sjultra.com`, previously protected only by an individual exception that had since been cleared) hit the identical failure mode — only an uppercase-primary address in the domain, no lowercase alias. Three days is far longer than any reasonable propagation delay, which ruled out "just wait longer" as the explanation.
9. **Tried to validate group membership via `Get-Recipient -RecipientPreviewFilter`** using both `EmailAddresses -like` and, to isolate whether the multi-valued property was the issue, a plain `PrimarySmtpAddress -like` filter. Both returned empty results — even though the exact same logic run live via `Get-Mailbox | Where-Object` returned correct results, and an `-eq` (exact match) filter through the same preview cmdlet worked fine. Concluded this cmdlet is simply unreliable for `-like` filters in this tenant and should never be trusted to validate these groups — a real dead end, now documented as a standing caution.
10. **Root-caused the actual failure**: the transport rule does not automatically pick up a change to a referenced dynamic group's `RecipientFilter`, even after several days. Confirmed by forcing a re-bind — removing the group from the rule's `ExceptIfSentToMemberOf` and immediately adding it back in the same session — which resolved delivery for the affected mailbox on the next live test, with no further filter changes.

---

## Check

Fixes applied and verified:

- **Scoped the rule** with `RecipientDomainIs = ibenit.com, sjultra.com, vzxy.net`, so it can never evaluate against the tenant's other Authoritative domains.
- **Rebuilt the filters** on `all-sjultra`, `all-ibenit`, and `all-vzxy` to match on recipient type (`UserMailbox`, `MailUser`, `MailContact`) and domain, instead of the broken `Company`/unscoped-`Alias` logic — including both `smtp:` and `SMTP:` proxy-address prefixes to cover primary-only addresses like User B's.
- **Cleaned up the individual `ExceptIfSentTo` list**: removed stale test-account entries (`user01`–`user10@vzxy.net`, since retired) and deduplicated repeated addresses.
- **Removed the now-redundant scaffolding groups** (`DDG-AllRecipients-*`) created during diagnosis once the original `all-*` groups were fixed to do the same job — avoided leaving two parallel, identical mechanisms in place.
- **Verified with live external test messages** to previously-affected shared mailboxes (e.g. `user-c@sjultra.com`) via message trace — confirmed delivery with no redirect.
- **Confirmed the real fix** — removing and re-adding the affected group in the rule's `ExceptIfSentToMemberOf` — resolved delivery for every mailbox previously falling through, verified via live external test messages after the re-bind.

---

## Act

- **Resolved.** The root cause was not propagation lag: an Exchange Online transport rule does not automatically re-evaluate a referenced dynamic distribution group's `RecipientFilter` after it changes, even after multiple days. The rule must be forced to re-bind by removing the group from `ExceptIfSentToMemberOf` and adding it back in the same session, immediately after any future filter edit.
- **Standardized into a repeatable procedure**: [Build a Domain-Scoped Catch-All/Redirect Rule with Self-Maintaining Exceptions](catch-all-redirect-setup-sop.md), so this same pattern (and both gotchas found here — the `SMTP:`/`smtp:` casing gap and the mandatory group re-bind step) can be applied to any other domain or tenant without repeating this diagnosis.
- **Standing practice going forward:** if anyone reports this issue again, add the affected address to `ExceptIfSentTo` immediately to unblock the person, then apply the group re-bind step from the SOP above — no need to re-diagnose from scratch each time.
- **Documented dead end:** `Get-Recipient -RecipientPreviewFilter` cannot be used to validate these groups' membership in this tenant — it returns empty results for `-like` filters regardless of the property being matched, even when the group is working correctly in live mail flow. Use a `Get-Mailbox | Where-Object` simulation instead.

---

*Last updated: 2026-09-25*
