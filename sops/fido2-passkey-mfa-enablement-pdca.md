# PDCA Plan: Enable FIDO2 / Passkey MFA in Microsoft Entra ID

**Related SOP:** [Enable FIDO2 / Passkey MFA in Microsoft Entra ID](SOP%20-%20Enable%20FIDO2%20Passkey%20MFA%20(Microsoft%20Entra%20ID).md) — this document is the Plan-Do-Check-Act framing behind that SOP: the reasoning, sequencing, and decision points for a first rollout.

**Client:** CCOM Internal

---

## Plan

**Current state:** The "Add a sign-in method" screen only offers Microsoft Authenticator, Hardware token, Phone, Alternate phone, and Office phone. FIDO2 security key and passkey are not showing because the Passkey (FIDO2) authentication method is disabled (or not opted into passkey profiles) in the tenant's Authentication methods policy. This is a tenant-level admin setting, not a per-user setting — no user will see the option until an admin enables it.

**Objective:** Enable phishing-resistant passkey (FIDO2) sign-in as an available MFA/authentication method in the Microsoft Entra ID tenant, piloted first, then rolled out broadly.

**Roles required:**
- Authentication Policy Administrator — enable/configure the Passkey (FIDO2) policy and profiles
- Conditional Access Administrator — required only if enforcing passkey sign-in via authentication strength (Act phase)

**Prerequisites:**
- Confirm license: Passkeys (FIDO2) work on all Entra ID editions, including Free — no extra licensing needed
- Inventory target devices/OS: Entra-joined Windows needs 10 v1903+; hybrid-joined needs Windows 10 v2004+
- Decide passkey types to support: device-bound (FIDO2 hardware keys, Microsoft Authenticator) and/or synced (Apple iCloud Keychain, Google Password Manager, 3rd-party like 1Password/Bitwarden)
- If hardware keys are in scope, select a vendor and confirm AAGUID / attestation support
- Identify a pilot group (e.g., IT admins) before org-wide rollout

**Key decisions:**
1. Device-bound vs. synced vs. both — device-bound (hardware keys, Authenticator) gives stronger assurance and is recommended for admins/privileged accounts; synced passkeys are lower-cost and better UX for general staff but don't support attestation.
2. Attestation enforcement — Yes = only verified/genuine device-bound authenticators allowed (blocks synced passkeys for that profile); No = broader compatibility but no vendor verification.
3. Key restrictions (AAGUID allow/block list) — optionally lock registration to specific approved key models.
4. Rollout scope — pilot group first, then phased expansion, vs. enabling for all users at once.

**Anticipated risks / constraints:**
- Passkey (FIDO2) policy has a 20 KB size limit — plan profile/target count accordingly (base policy ~1.44 KB, each profile target ~0.23–0.4 KB).
- Guest/B2B users cannot register FIDO2 passkeys.
- If a user's UPN changes, existing passkeys can't be auto-updated — user must delete and re-register.
- Removing an AAGUID from an allow-list retroactively blocks users already using that key model.
- Admin bulk-provisioning of FIDO2 keys via Microsoft Graph is currently in preview (requires Authentication Administrator role or app permission `UserAuthenticationMethod.ReadWrite.All`).

---

## Do

Steps taken, in order:

1. **Opt in to passkey profiles.** Entra admin center > Entra ID > Security > Authentication methods > Policies > Passkey (FIDO2) > select the banner link to opt in. This is a one-way action (cannot opt out once enabled). On the Configure tab, set Allow self-service set up to Yes.
2. **Configure the profile.** Open the Default passkey profile (or add a new one via + Add passkey profile). Set Passkey types (Device-bound and/or Synced), set Enforce attestation per the Plan decision, and optionally configure a Key Restriction Policy (AAGUID allow/block list). Save.
3. **Target the pilot group.** Enable and Target tab > confirm Enable = On > Add target > select the pilot group (not All users) > assign the profile > Save.
4. **Run the pilot.** Pilot users complete MFA (required within 5 minutes prior) and register a passkey via [Security info](https://mysignins.microsoft.com/security-info). Test registration and sign-in for each passkey type in scope, across representative OS/browser combinations. Confirm the helpdesk/account-recovery process for a lost key.

---

## Check

Verification steps run after the pilot:

- [ ] Pilot users can see "Passkey" as an option under Security info > Add sign-in method
- [ ] Registration succeeds for each targeted passkey type
- [ ] Sign-in succeeds using the registered passkey
- [ ] Helpdesk has a tested process for lost-key account recovery
- [ ] No unexpected lockouts or support-ticket spikes during the pilot window

**Review and adjust:** Collect pilot user feedback and adjust profile settings (attestation, AAGUID restrictions) as needed. Confirm the chosen passkey types and attestation setting still meet security requirements after real-world testing, then decide go/no-go for expanding beyond the pilot group.

---

## Act

- **Expand rollout:** Add additional target groups incrementally, or switch target to All users once validated. Communicate to end users what a passkey is, how to register, and where to get hardware keys if applicable. Update onboarding/IT documentation and helpdesk runbooks.
- **(Optional) Enforce for sensitive resources:** As Conditional Access Administrator, go to Entra ID > Authentication methods > Authentication strengths. Use the built-in Phishing-resistant strength, or create a New authentication strength scoped to Passkeys (FIDO2), optionally restricted to specific AAGUIDs. Apply via a Conditional Access policy to sensitive apps/roles.
- **Rollback plan:** Passkey profiles, once opted in, cannot be fully disabled tenant-wide, but individual profiles/targets can be removed or set to exclude a group to stop new registrations for that group. Existing registered passkeys can be deleted per-user from Authentication methods on the user object if needed.
- **Standardize:** Once stable, the working configuration is captured in the [Enable FIDO2 / Passkey MFA SOP](SOP%20-%20Enable%20FIDO2%20Passkey%20MFA%20(Microsoft%20Entra%20ID).md) for future tenants/rollouts. Set a recurring review (e.g., annually or after policy changes) to revisit profile settings and AAGUID allow-lists.

---

*Last updated: 2026-07-30*
