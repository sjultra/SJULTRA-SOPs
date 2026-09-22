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

---

## Check

Fixes applied and verified:

- **Scoped the rule** with `RecipientDomainIs = ibenit.com, sjultra.com, vzxy.net`, so it can never evaluate against the tenant's other Authoritative domains.
- **Rebuilt the filters** on `all-sjultra`, `all-ibenit`, and `all-vzxy` to match on recipient type (`UserMailbox`, `MailUser`, `MailContact`) and domain, instead of the broken `Company`/unscoped-`Alias` logic — including both `smtp:` and `SMTP:` proxy-address prefixes to cover primary-only addresses like User B's.
- **Cleaned up the individual `ExceptIfSentTo` list**: removed stale test-account entries (`user01`–`user10@vzxy.net`, since retired) and deduplicated repeated addresses.
- **Removed the now-redundant scaffolding groups** (`DDG-AllRecipients-*`) created during diagnosis once the original `all-*` groups were fixed to do the same job — avoided leaving two parallel, identical mechanisms in place.
- **Verified with live external test messages** to previously-affected shared mailboxes (e.g. `user-c@sjultra.com`) via message trace — confirmed delivery with no redirect.
- **Added `user-b@sjultra.com` to the individual exception list** as an immediate stopgap once his report came in — his corrected-filter fix hadn't yet taken effect in mail-flow evaluation (see Act, below), so the individual list unblocked him right away without waiting on it.

---

## Act

- The `all-sjultra` filter was corrected to match both `smtp:` and `SMTP:` proxy-address prefixes, which should now cover `user-b@sjultra.com` (and anyone else whose only domain address is their primary, uppercase-prefixed SMTP address). A retest immediately after the fix still showed him getting redirected, even though the corrected logic checks out.
- **Most likely explanation:** Dynamic Distribution Group filter changes appear to propagate to live directory queries (`Get-Mailbox`, `Get-Recipient`) faster than they do to the separate, periodically-refreshed membership expansion that mail-flow / transport-rule evaluation uses. Since the corrected filter was only applied minutes before the retest, that propagation gap is the leading theory — **not yet confirmed.**
- **Outstanding follow-up:** after a few hours (or the next business day), temporarily remove `user-b@sjultra.com` from the individual `ExceptIfSentTo` list and send another external test message.
  - If it delivers cleanly without the individual entry — the group fix has propagated and is working as designed. No further action needed.
  - If it's still getting redirected after a full day — propagation lag isn't the explanation, and this needs a different long-term fix (e.g., a scheduled script that maintains a static, non-dynamic group membership instead of relying on dynamic filter expansion for mail-flow evaluation).
- **Standing practice going forward:** if anyone reports this issue again, add the affected address to `ExceptIfSentTo` immediately to unblock the person, then let the dynamic group catch up in the background — no need to re-diagnose from scratch each time.

---

*Last updated: 2026-09-22*
