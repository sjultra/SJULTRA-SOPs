## Process Document: Block Access to an Amazon Bedrock Foundation Model (SCP + IAM Deny)

**Owner:** AWS Account Administrator (Organizations management account access) | **Last Updated:** 2026-09-02 | **Review Cadence:** As needed, when a new model needs to be blocked, or annually to confirm SCPs still match active model usage

### Purpose

Block one or more Amazon Bedrock foundation models from being invoked — at a single account or across the entire AWS Organization — now that AWS has retired the self-service **Model access** console page. Serverless Bedrock foundation models are auto-enabled account-wide the first time they're invoked, so the only remaining controls are IAM identity-based policies (single account/role) and AWS Organizations Service Control Policies (SCPs) (single account through entire org). This SOP documents how to apply either.

### Scope

**In scope:** Denying `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`, and related invocation actions for a specific foundation model or model family, at the account level (SCP or IAM) or org-wide (SCP at Root/OU).

**Out of scope:** Deleting or unpublishing a model from AWS's catalog (not possible for a customer — Bedrock models are AWS/Anthropic-hosted); Bedrock Guardrails content filtering (a different control, governing model *output*, not whether the model can be called at all); AWS Marketplace-subscribed models (different access mechanism from serverless models).

### RACI Matrix

| Step | Responsible | Accountable | Consulted | Informed |
|------|------------|-------------|-----------|----------|
| Identify model ID / ARN to block | Requesting stakeholder (e.g. Cost/FinOps owner) | AWS Account Administrator | — | — |
| Decide scope (single account vs. org-wide) | AWS Account Administrator | IT Manager | Affected account owners | — |
| Draft deny policy JSON | AWS Account Administrator | IT Manager | — | — |
| Create & attach SCP / IAM policy | AWS Organizations Administrator | IT Manager | Security | Affected account owners |
| Verify block is effective | AWS Account Administrator | IT Manager | — | Requesting stakeholder |
| Document & log the change | AWS Account Administrator | IT Manager | — | All stakeholders |

### Process Flow

```
Identify model ID (Bedrock console Model catalog, or aws bedrock list-foundation-models)
        |
Decide scope: single account (SCP on account) vs org-wide (SCP on Root/OU)
        |
Draft Deny policy JSON
   (foundation-model ARN + cross-region inference-profile ARN)
        |
Create policy in AWS Organizations console
        |
Attach to target (account / OU / Root)
        |
Verify: attempt InvokeModel -> expect AccessDeniedException
        |
Confirm in Cost Explorer that new usage stops accruing
        |
Record the change (companion runbook / change log)
```

### Detailed Steps

#### Step 1: Identify the exact model ID

- **Who**: Requesting stakeholder / AWS Account Administrator
- **When**: Before drafting any policy
- **How**: Bedrock console > **Model catalog** > open the model > copy its Model ID (e.g. `anthropic.claude-sonnet-4-6`). The console display name (e.g. "Claude Sonnet 4.6 (Amazon Bedrock Edition)") is not the string used in ARNs.
- **Output**: Confirmed model ID to block.

#### Step 2: Decide the blast radius

- **Who**: AWS Account Administrator, with IT Manager sign-off
- **When**: After Step 1
- **How**: Decide whether the block applies to a single AWS account (attach the SCP directly to that account) or the whole organization (attach at the Root or an OU covering every member account). SCPs never apply to the Organization's management account itself.
- **Output**: Named target(s) for policy attachment.

#### Step 3: Draft the Deny policy

- **Who**: AWS Account Administrator
- **When**: After scope is decided
- **How**: Build a policy denying `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream` against two resource patterns — the direct foundation-model ARN (`arn:aws:bedrock:*::foundation-model/<model-id>*`) and the cross-region inference-profile ARN (`arn:aws:bedrock:*:*:inference-profile/*<model-id>*`), since Bedrock may route calls through either path. A wildcard suffix on the model ID catches all dated/versioned variants.
- **Output**: Policy JSON ready to paste into the console.

#### Step 4: Create and attach the policy

- **Who**: AWS Organizations Administrator (needs management-account access to create/attach SCPs)
- **When**: After JSON is drafted and approved
- **How**: AWS Organizations console > **Policies** > **Service control policies** > **Create policy** > paste JSON > name it descriptively (e.g. `Deny-Bedrock-<model-id>`) > **Policies** > open the new policy > **Attach** > select the target account, OU, or Root decided in Step 2.
- **Output**: Policy attached and active.

#### Step 5: Verify

- **Who**: AWS Account Administrator
- **When**: Immediately after attaching
- **How**: From a principal in the blocked account, attempt an `InvokeModel` call against the model (CLI, SDK, or the Bedrock playground) — expect an `AccessDeniedException` referencing the SCP. Over the following days, confirm in Cost Explorer (filtered to the model/service) that no new usage accrues.
- **Output**: Confirmed block, or a note of remaining leakage (e.g. another account still using it) to fix.

#### Step 6: Record the change

- **Who**: AWS Account Administrator
- **When**: After verification
- **How**: Log the change — the companion runbook documents the specific execution. Record the model ID, target(s), policy name, date, and who approved it.
- **Output**: Auditable record for future reference or rollback.

### Exceptions and Edge Cases

| Scenario | What to Do |
|----------|-----------|
| Model was reached via a cross-region inference profile, not a direct model ARN | Include the `inference-profile` resource pattern in the Deny statement (Step 3) — a Deny on the foundation-model ARN alone may not cover profile-routed calls. |
| Need to block for one account now, org-wide later | Attach the same policy to the single account first; when ready, also attach it (or an identical copy) at the Root/OU — SCPs can be attached to multiple targets. |
| Someone needs the model back temporarily | Detach the SCP from the affected target (don't delete it) — fully reversible; see the companion runbook's Rollback section. |
| AWS Organizations management account also needs to be blocked | SCPs cannot restrict the management account — use an IAM Deny policy directly on roles/users in that account instead. |
| A *different*, unblocked account uses this Anthropic model for the first time | First-time Anthropic model use may still require submitting use-case details in that account, per AWS's current (post model-access-page-retirement) flow — this Deny policy doesn't affect that unrelated opt-in step for other accounts. |
| "Create policy" is missing or disabled in AWS Organizations, even though the org is in "All features" mode | Service control policies must also be explicitly enabled as a policy type (separate from the org's feature set) | Organizations console -> **Settings** -> **Enable service control policies**, then retry Step 4. |

### Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Time from decision to policy attached | < 1 business day | Change log timestamp vs. decision timestamp |
| New usage after block | $0 / 0 invocations | Cost Explorer, filtered to model + account, 7 days post-block |
| False-positive access denials (legitimate use blocked unintentionally) | 0 | Helpdesk/escalation tickets referencing this policy |

### Related Documents

- [Runbook: Block anthropic.claude-sonnet-4-6 in Sandbox Account (279199950628)](aws-bedrock-claude-sonnet-4-6-sandbox-block-runbook.md)
- [Add or remove access to Amazon Bedrock foundation models — AWS docs](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access-modify.html)
- [Example SCPs for Amazon Bedrock — AWS Organizations docs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_examples_bedrock.html)
- [Implementing least privilege access for Amazon Bedrock — AWS Security Blog](https://aws.amazon.com/blogs/security/implementing-least-privilege-access-for-amazon-bedrock/)
