## Process Document: Release and Allowlist False-Positive Phishing Quarantine (Microsoft 365 Defender)

**Owner:** IT/Security Administrator (Security & Compliance Administrator role) | **Last Updated:** 2026-08-25 | **Review Cadence:** Quarterly, or after any Tenant Allow/Block List policy change

### Purpose

Restore delivery of legitimate business email that Microsoft 365 Defender for Office 365 blocks to Quarantine as a false-positive Phish (or Spam/Bulk) verdict, using the Tenant Allow/Block List so the fix is durable and auditable — without weakening protection against genuinely spoofed mail.

### Scope

**In scope:** Diagnosing a Quarantine verdict in the Defender portal, verifying SPF/DKIM/DMARC/Composite authentication results, creating and renewing Tenant Allow/Block List allow entries for sender addresses or domains, releasing individually quarantined messages, and submitting false-positive reports to Microsoft.

**Out of scope:** Malware verdicts (never overridden by this procedure); confirmed spoofing or authentication-failure cases (handled by fixing the sending domain's DNS records, not by allow-listing); general end-user phishing-awareness training.

### RACI Matrix

| Step | Responsible | Accountable | Consulted | Informed |
|------|------------|-------------|-----------|----------|
| Diagnose quarantine reason | IT/Security Administrator | IT Manager | Reporting client/employee | — |
| Verify authentication results | IT/Security Administrator | IT Manager | — | — |
| Create Tenant Allow/Block List entry | IT/Security Administrator | IT Manager | Security/Compliance | — |
| Release quarantined message | IT/Security Administrator | IT Manager | — | Affected recipient(s) |
| Submit false positive to Microsoft | IT/Security Administrator | — | — | — |
| Review/renew allow entry before expiration | IT/Security Administrator | IT Manager | — | — |

### Process Flow

```
Report received (legitimate mail missing / stuck)
        |
Locate the message in Quarantine (Defender portal)
        |
Review quarantine detail: verdict + Authentication (SPF/DKIM/DMARC/Composite)
        |
Verdict = Malware? --Yes--> STOP. Never override malware verdicts.
        |No
Composite authentication = Pass? --No--> Do NOT allow-list.
        |Yes                              Escalate: coordinate SPF/DKIM/DMARC
        |                                 fix with the sending domain's owner.
Add Tenant Allow/Block List entry
   (sender address or domain, Allow, note + review/expiration date)
        |
Submit message via Submissions ("Should not have been blocked")
        |
Release the quarantined message for affected recipient(s)
        |
Notify reporter; monitor for recurrence
        |
Set reminder to review/renew allow entry before it expires
```

### Detailed Steps

#### Step 1: Confirm the report
- **Who**: IT/Security Administrator
- **When**: On report of missing or delayed legitimate email
- **How**: Get the sender address, approximate date/time, and affected recipient(s) from the reporter. Ask whether other recipients at the same organization did or didn't receive it — uneven delivery across recipients is itself a useful diagnostic signal (see Exceptions).
- **Output**: Enough detail to search Quarantine for the specific message.

#### Step 2: Locate the message in Quarantine
- **Who**: IT/Security Administrator
- **When**: Immediately after Step 1
- **How**: **security.microsoft.com > Email & collaboration > Review > Quarantine**. Search/filter by sender or recipient and date range. If the message doesn't appear in the tenant-wide Quarantine list, check the affected mailbox's own quarantine view — visibility can differ by role/permissions.
- **Output**: The specific quarantined message record.

#### Step 3: Review the quarantine detail and authentication results
- **Who**: IT/Security Administrator
- **When**: Immediately after locating the message
- **How**: Open the message and record the **Original/Latest Threats** verdict (Phish / Spam / Bulk / Malware), the **Detection technology** (e.g. Advanced filter), and the **Authentication** section (SPF, DKIM, DMARC, Composite authentication).
- **Output**: A clear verdict + authentication pass/fail combination to branch on.

#### Step 4: Branch on authentication result
- **Who**: IT/Security Administrator
- **When**: After Step 3
- **How**:
  - **Malware verdict** → stop. Never override, regardless of authentication result.
  - **Composite authentication = Pass**, verdict Phish/Spam/Bulk from Advanced Filter (content/heuristic) → genuine false positive, proceed to Step 5. Content-based heuristics commonly flag legitimate transactional email that mimics common phishing patterns — e-signature "click to sign" notifications (ZohoSign, DocuSign, Adobe Sign, etc.) are a frequent example even when authentication is clean.
  - **Composite authentication = Fail** → do **not** allow-list. This may be genuine spoofing. Escalate to fixing the sending domain's SPF/DKIM/DMARC records with the sender's own IT team.
- **Output**: Decision to proceed with allow-listing, or to escalate an authentication fix instead.

#### Step 5: Add a Tenant Allow/Block List entry
- **Who**: IT/Security Administrator
- **When**: After confirming Step 4 is a genuine false positive
- **How**: **Threat policies > Rules > Tenant Allow/Block Lists > Domains & addresses** tab > **Add**. Enter the sender's email address (narrowest scope) or domain (if the same legitimate vendor sends from multiple addresses on one domain). Set action to **Allow**, and add a note with the reason, ticket reference, and date. Set a review/expiration reminder — allow entries expire on a fixed window by design (commonly ~30 days), so this is not a permanent fix.
- **Output**: Future mail from this sender bypasses the phishing/spam/bulk filters that produced the false positive. Malware scanning is not bypassed.

#### Step 6: Submit the message to Microsoft as a false positive
- **Who**: IT/Security Administrator
- **When**: Same session as Step 5
- **How**: From the quarantined message, use **Submissions** → **report as "Should not have been blocked"** → **Allow**. This both helps release the message and feeds Microsoft's detection model, reducing future false positives tenant-wide.
- **Output**: False positive reported to Microsoft; message flagged for release.

#### Step 7: Release the quarantined message
- **Who**: IT/Security Administrator
- **When**: After Steps 5–6
- **How**: From the Quarantine list, select the message(s) for the affected recipient(s) and choose **Release** (or release to all recipients if more than one was affected). Confirm with the reporter that it now appears in their inbox.
- **Output**: Legitimate email delivered.

#### Step 8: Notify and monitor
- **Who**: IT/Security Administrator
- **When**: After release
- **How**: Tell the reporter/sender the issue is resolved, and that they can resend if needed. Watch Quarantine for the same sender over the following few sends to confirm the allow entry is working for all recipients, not just the one released manually.
- **Output**: Confirmed resolution.

#### Step 9: Review and renew before expiration
- **Who**: IT/Security Administrator
- **When**: Before the allow entry's expiration date set in Step 5
- **How**: Track allow-list entries as recurring reminders — the Tenant Allow/Block Lists page shows each entry's expiration date. Before expiry, re-verify the sender is still legitimate and renew, or let it lapse if it's no longer needed.
- **Output**: No recurrence of the false-positive block for an active legitimate sender.

### Exceptions and Edge Cases

| Scenario | What to Do |
|----------|-----------|
| Authentication fails (SPF/DKIM/DMARC) | Do not allow-list — this may be a real spoof. Escalate to the sending domain's owner to fix DNS records. |
| Verdict is Malware | Never override — malware detections are not eligible for allow-list bypass. |
| Delivers fine for some recipients but not others, despite one shared policy | Likely Defender's per-mailbox "mailbox intelligence" — recipients with prior interaction history get a lower phish score than first-time recipients. A tenant-wide allow entry (Step 5) resolves it for everyone regardless of individual history. |
| One vendor sends from many addresses/subdomains on a shared platform | Allow by domain rather than a single address, but document the broader scope and set a shorter review interval. |
| Client insists on a "permanent" whitelist | Explain that Tenant Allow/Block List entries expire by design and must be periodically renewed — there is no indefinite override for phishing verdicts, since that would itself be a security gap. |
| Quarantined message isn't visible in the tenant-wide Quarantine view | Check the affected mailbox's own quarantine view — visibility can differ by admin role/permissions. |

### Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Time from report to release | < 1 business hour | Ticket timestamps (report vs. release) |
| Recurrence rate for previously allow-listed senders | 0 recurrences before entry expiration | Quarantine search by sender, post-fix |
| Allow-list entries reviewed before expiration | 100% | Tenant Allow/Block List expiration dates vs. review log |

### Related Documents

- [Tenant Allow/Block List (Microsoft Learn)](https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-about)
- [Quarantined email messages in EOP and Microsoft Defender for Office 365](https://learn.microsoft.com/en-us/defender-office-365/quarantine-about)
- [Report false positives and false negatives to Microsoft](https://learn.microsoft.com/en-us/defender-office-365/submissions-admin)
- Example incident this SOP was written from: an e-signature vendor's transactional notification email (Advanced Filter Phish/High verdict, SPF/DKIM/DMARC/Composite all Pass) quarantined for most recipients at a client tenant, Aug 2026 — client name omitted here per internal documentation policy.
