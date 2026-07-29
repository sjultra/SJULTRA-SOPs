## Process Document: Enable FIDO2 / Passkey MFA in Microsoft Entra ID

**Owner:** IT/Security Administrator (Authentication Policy Administrator role) | **Last Updated:** 2026-07-29 | **Review Cadence:** Annually or after any Microsoft Entra authentication-methods policy change

### Purpose

Enable phishing-resistant passkey (FIDO2) sign-in as an MFA/authentication method in a Microsoft Entra ID tenant, so users can register and use FIDO2 hardware keys, Microsoft Authenticator passkeys, or synced passkeys (iCloud Keychain, Google Password Manager, etc.) instead of, or alongside, weaker methods like SMS/phone.

### Scope

**In scope:** Tenant-level configuration of the Passkey (FIDO2) authentication method policy, passkey profile creation, pilot rollout, org-wide rollout, and optional Conditional Access enforcement for sensitive resources.

**Out of scope:** End-user device/browser troubleshooting unrelated to Entra configuration; procurement of hardware FIDO2 keys (handled as a separate purchasing task); non-Microsoft identity providers.

### RACI Matrix

| Step | Responsible | Accountable | Consulted | Informed |
|------|------------|-------------|-----------|----------|
| Opt in to passkey profiles | Authentication Policy Administrator | IT Manager | Security/Compliance | All admins |
| Configure passkey profile(s) | Authentication Policy Administrator | IT Manager | Security/Compliance | Helpdesk |
| Target pilot group | Authentication Policy Administrator | IT Manager | Pilot users | Helpdesk |
| Pilot testing | Pilot users, Helpdesk | IT Manager | Authentication Policy Administrator | IT Manager |
| Org-wide rollout | Authentication Policy Administrator | IT Manager | Helpdesk, Comms | All users |
| Conditional Access enforcement (optional) | Conditional Access Administrator | IT Manager | Security/Compliance | Affected users |

### Process Flow

```
Confirm prereqs & roles
        |
Opt in to Passkey (FIDO2) profiles  --> (irreversible opt-in)
        |
Configure Default / new passkey profile
   (types, attestation, AAGUID restrictions)
        |
Enable & Target -> Pilot group
        |
Pilot test registration + sign-in --> issues? --> adjust profile --> retest
        |
Expand target to additional groups / All users
        |
Communicate to end users + update helpdesk docs
        |
(Optional) Enforce via Conditional Access authentication strength
```

### Detailed Steps

#### Step 1: Confirm prerequisites and roles
- **Who**: Authentication Policy Administrator
- **When**: Before starting any tenant configuration
- **How**: Confirm Entra ID license (Passkeys work on all editions, including Free — no extra license needed); inventory target OS/devices (Entra-joined Windows needs 10 v1903+, hybrid-joined needs 10 v2004+); decide device-bound vs. synced vs. both; select a pilot group; if hardware keys are in scope, confirm vendor AAGUID/attestation support.
- **Output**: Documented scope decision (which passkey types, which users, attestation on/off).

#### Step 2: Opt in to passkey profiles
- **Who**: Authentication Policy Administrator
- **When**: Once prerequisites are confirmed
- **How**: Entra admin center > **Entra ID > Security > Authentication methods > Policies > Passkey (FIDO2)** > select the banner link to opt in to passkey profiles. This is a one-way action (cannot opt out once enabled). On the **Configure** tab, set **Allow self-service set up** to **Yes**.
- **Output**: Passkey profiles enabled; existing global FIDO2 settings auto-migrated to a **Default passkey profile**.

#### Step 3: Configure passkey profile(s)
- **Who**: Authentication Policy Administrator
- **When**: Immediately after opting in
- **How**: Open the **Default passkey profile** (or **+ Add passkey profile** for up to 2 additional profiles). Set **Passkey types** (Device-bound and/or Synced), set **Enforce attestation** (Yes = device-bound only, verified vendors; No = broader compatibility, no vendor verification), and optionally configure a **Key Restriction Policy** (AAGUID allow/block list). Save.
- **Output**: One or more passkey profiles configured to organizational requirements (e.g., stricter profile for admins, lighter-touch profile for general staff).

#### Step 4: Target and enable for pilot group
- **Who**: Authentication Policy Administrator
- **When**: After profile configuration
- **How**: **Enable and Target** tab > confirm **Enable = On** > **Add target** > select the pilot group (not All users) > assign the appropriate profile > Save.
- **Output**: Pilot group can register and sign in with passkeys; all other users unaffected.

#### Step 5: Pilot test
- **Who**: Pilot users, with Helpdesk support
- **When**: Immediately following Step 4
- **How**: Pilot users complete MFA (required within 5 minutes prior) and register a passkey via [Security info](https://mysignins.microsoft.com/security-info). Test registration and sign-in for each passkey type in scope, across representative OS/browser combinations. Validate the helpdesk/account-recovery process for a lost key.
- **Output**: Confirmed working configuration, or a list of adjustments needed to profile settings.

#### Step 6: Expand rollout
- **Who**: Authentication Policy Administrator
- **When**: After successful pilot sign-off
- **How**: Add additional target groups incrementally, or switch target to **All users**. Communicate to end users what a passkey is, how to register, and where to get hardware keys if applicable. Update onboarding and helpdesk runbooks.
- **Output**: Org-wide (or fully scoped) passkey availability.

#### Step 7 (Optional): Enforce for sensitive resources
- **Who**: Conditional Access Administrator
- **When**: After rollout is stable
- **How**: **Entra ID > Authentication methods > Authentication strengths** > use the built-in **Phishing-resistant** strength or create a **New authentication strength** scoped to **Passkeys (FIDO2)** (optionally restricted by AAGUID). Apply via a Conditional Access policy to sensitive apps/roles (e.g., admin portals, privileged roles).
- **Output**: Passkey sign-in required for designated high-risk access.

### Exceptions and Edge Cases

| Scenario | What to Do |
|----------|-----------|
| User's UPN changes after passkey registration | Passkey can't be auto-updated; user deletes old passkey and re-registers via Security info. |
| Guest/B2B user needs a passkey | Not supported — FIDO2 passkey registration is unavailable for internal or external guest users. |
| Admin wants to remove an AAGUID from the allow-list | Warn first — this retroactively blocks sign-in for anyone already using that key model. |
| Passkey (FIDO2) policy approaching 20 KB size limit | Consolidate profiles/targets before adding more (reference: base policy ~1.44 KB, each profile target ~0.23–0.4 KB, each profile with 10 AAGUIDs ~0.3 KB). |
| Need to bulk-provision hardware keys centrally instead of self-service | Use the Microsoft Graph FIDO2 provisioning API (preview) — requires Authentication Administrator role or `UserAuthenticationMethod.ReadWrite.All` app permission. |
| Need to remove a specific user's passkey | Entra admin center > find user > **Authentication methods** > right-click **Passkey (device-bound)** > **Delete**. |

### Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Pilot registration success rate | ≥ 95% | Registrations completed / pilot group size |
| Helpdesk tickets related to passkey setup (first 30 days post-rollout) | Trending down week over week | Ticket count tagged "passkey" or "FIDO2" |
| % of eligible users with a registered passkey | Org-defined target (e.g., 80% within 90 days) | Authentication methods report in Entra admin center |
| Sign-ins using passkey vs. legacy MFA | Increasing trend | Entra sign-in logs, filtered by auth method |

### Related Documents

- [FIDO2 / Passkey MFA Enablement Plan (project-specific plan)](./FIDO2-Passkey-MFA-Enablement-Plan.md)
- [How to enable passkeys (FIDO2) in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/authentication/how-to-authentication-passkeys-fido2)
- [Passkeys (FIDO2) authentication method in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-passkeys-fido2)
- [Enable and support passkeys in Authenticator for Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/authentication/how-to-enable-authenticator-passkey)
