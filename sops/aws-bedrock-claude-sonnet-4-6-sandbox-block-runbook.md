## Runbook: Block anthropic.claude-sonnet-4-6 in Sandbox Account (279199950628)

**Owner:** Richard Rives | **Frequency:** As Needed
**Last Updated:** 2026-09-02 | **Last Run:** 2026-09-02 — completed successfully

### Purpose

Stop all further Bedrock usage/cost for the `anthropic.claude-sonnet-4-6` model in the Sandbox account (279199950628), following the AWS Cost Anomaly Detection alert on 2026-08-29 ($27.24 single-day impact; $84.67 for this model across August, 83% of which was prompt-caching overhead). This is the specific execution of the general [SOP - Block Amazon Bedrock Model Access (SCP + IAM Deny)](SOP%20-%20Block%20Amazon%20Bedrock%20Model%20Access%20%28SCP%20%2B%20IAM%20Deny%29.md), scoped to just this account per the 2026-09-02 decision — Sandbox only, no other accounts were affected.

### Prerequisites

- [x] AWS Organizations management-account access (required to create/attach an SCP) — confirmed to be account **Iben Rodriguez (169952954347)**; signed in via IAM Identity Center, permission set `AdministratorAccess`
- [x] Confirmed model ID: `anthropic.claude-sonnet-4-6` (from Bedrock Model catalog)
- [x] Confirmed target account: Sandbox, 279199950628
- [x] Service control policies enabled as a policy type in this Organization (Organizations console -> **Settings** -> **Enable service control policies**) — the org was already in "All features" mode, but the SCP policy type itself still had to be turned on before **Create policy** became available. Do this check first if Create policy / Edit in AWS Organizations appears disabled.

### Procedure

#### Step 1: Create the SCP

```
AWS Organizations console -> Policies -> Service control policies -> Create policy
Name: Deny-Bedrock-anthropic-claude-sonnet-4-6-Sandbox
Description: Blocks all invocation of anthropic.claude-sonnet-4-6 (direct + cross-region
inference profile) in the Sandbox account, per AWS Cost Anomaly alert 2026-08-29.
```

Paste this JSON as the policy content:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyClaudeSonnet46Direct",
      "Effect": "Deny",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream",
        "bedrock:CreateModelInvocationJob",
        "bedrock:GetModelInvocationJob"
      ],
      "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.claude-sonnet-4-6*"
    },
    {
      "Sid": "DenyClaudeSonnet46CrossRegionProfile",
      "Effect": "Deny",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "arn:aws:bedrock:*:*:inference-profile/*anthropic.claude-sonnet-4-6*"
    }
  ]
}
```

**Expected result:** Policy saved and listed under Service control policies.
**If it fails:** "Create policy" is grayed out, or you get a permission error — you're not in the management account, or your role lacks `organizations:CreatePolicy`. Escalate to whoever holds Organizations admin (see Escalation below).

#### Step 2: Attach the SCP to the Sandbox account only

```
Policies -> Service control policies -> Deny-Bedrock-anthropic-claude-sonnet-4-6-Sandbox -> Attach
Target: Account 279199950628 (Sandbox)
```

**Expected result:** Policy shows as attached to account 279199950628, and only that account.
**If it fails:** Target account not listed — confirm 279199950628 is the correct account ID and that it's a member of this organization (not the management account).

#### Step 3: Confirm it's active

```
Organizations console -> Account 279199950628 -> Policies tab -> Service control policies
```

**Expected result:** `Deny-Bedrock-anthropic-claude-sonnet-4-6-Sandbox` listed as an attached/applied policy.

### Verification

- [ ] From within the Sandbox account, attempt an `InvokeModel` call against `anthropic.claude-sonnet-4-6` (CLI, SDK, or Bedrock playground) — expect `AccessDeniedException` citing the SCP.
- [ ] Cost Explorer, filtered to Service = Bedrock, Model = Claude Sonnet 4.6, Linked account = Sandbox — confirm no new daily cost appears after the attach date.
- [ ] Re-check after 24–48 hours in case any in-flight or cached calls briefly succeeded during propagation.

### Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Calls still succeed right after attaching | SCP propagation delay | Wait a few minutes and retest; if still succeeding after 15+ min, re-check the policy is attached to the correct account ID |
| Calls to a *different* Claude model still show cost | This policy only covers `anthropic.claude-sonnet-4-6*` | Confirm which model is actually driving the new cost before assuming the block failed |
| A legitimate workload in Sandbox breaks | Something else in that account depended on this model | Pause and confirm with the requester before rolling back — don't silently detach |

### Rollback

Detach (do not delete) the SCP: Organizations console -> Account 279199950628 -> Policies tab -> Service control policies -> `Deny-Bedrock-anthropic-claude-sonnet-4-6-Sandbox` -> Detach. Fully reversible, takes effect within minutes. Keep the policy itself (just detached) in case it's needed again, rather than recreating it from scratch.

### Escalation

| Situation | Contact | Method |
|-----------|---------|--------|
| Need Organizations management-account access | AWS Org owner (confirm current holder) | Direct request |
| Block appears not to be working after 24 hrs | Iben Rodriguez / Trytan Oluwadare | Email/Teams |
| Legitimate workload broken by this block | Requester of that workload | Email/Teams, before any rollback |

### History

| Date | Run By | Notes |
|------|--------|-------|
| 2026-09-02 | — | Runbook created following AWS Cost Anomaly alert (2026-08-29, $27.24) and decision to block the Sandbox account only. |
| 2026-09-02 | Richard Rives | Executed successfully. Signed into management account (Iben Rodriguez, 169952954347) via IAM Identity Center. Had to enable the "Service control policies" policy type in Organizations Settings first (was off despite All-features mode). Policy created and attached to Sandbox (279199950628). Policy ARN: `arn:aws:organizations::169952954347:policy/o-i953e90ayx/service_control_policy/p-o49yu0dk`. Pending: post-attach verification (InvokeModel denial test + Cost Explorer check over next 24-48h). |
