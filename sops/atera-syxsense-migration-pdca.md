# PDCA: Atera MSP Environment Setup — Replacing Syxsense

**Owner:** SJULTRA
**Date:** 2026-09-01
**Status:** In progress — kicked off 2026-09-01

### Key Dates
| Date | Event |
|---|---|
| **2026-09-01** | Kickoff — sign up for Atera 30-day free trial, begin environment build |
| **2026-10-01** | Atera 30-day trial expires — must convert to paid ("go live") before this date |
| **2026-10-06 → 2026-10-20** | Owner on vacation, unreachable — no active migration work during this window |
| **2026-10-21** | Owner back to work |
| **2026-10-30** | **Hard cutoff — Syxsense must be fully decommissioned by this date** |

**Real constraint:** only 9 days between returning from vacation (10/21) and the Syxsense cutoff (10/30). Treat **2026-10-05** as the true internal deadline to have as many customers/devices as possible fully migrated and stable — the post-vacation window should be verification and decommission only, not a migration scramble.

## Background

SJULTRA currently uses Syxsense for customer endpoint management (RMM/patch/device management). We are replacing Syxsense with Atera as our MSP platform.

**Why the switch:**
- **Cost:** Atera is more affordable than Syxsense at our current customer/endpoint volume.
- **RMM capability:** Atera's remote monitoring and management functionality is stronger than what Syxsense offers today, even though Atera is not a feature-for-feature match on every capability Syxsense provides.

**Environment profile:**
- Primary OS coverage: **Windows** and **macOS**.
- **Linux**: very few devices today, possibly none at some customers — treat as a minor/edge-case coverage path, not a primary platform.

---

## PLAN

### Objective
Stand up a production-ready Atera environment, migrate all managed customers/devices off Syxsense, and decommission Syxsense — with no gap in monitoring, patching, or remote support coverage during the transition.

### Scope
- All current Syxsense-managed customers and endpoints.
- Windows workstations/servers, macOS devices, and any incidental Linux endpoints.
- RMM functions in current use today: monitoring/alerting, patch management, remote access, scripting/automation, software deployment, reporting.
- Any current PSA/ticketing or billing integration tied to Syxsense.

### Out of scope (for this PDCA cycle)
- Re-evaluating other RMM vendors — Atera is the selected platform.
- Non-RMM tooling changes (e.g., AV/EDR, backup) unless they are bundled through Syxsense today and need a replacement as a side effect of the switch.

### Known gap risk — Atera is not an exact match for Syxsense
Before migrating a customer, confirm parity (or an accepted gap) for anything that customer actively relies on today, particularly:
- Any Syxsense-specific patch policies, compliance reporting, or vulnerability scanning features in active use.
- Linux management depth — Atera's Linux agent/scripting support is lighter than its Windows/macOS support. For scattered Linux devices, plan on basic monitoring + SSH-based scripting rather than full patch-policy automation, and confirm this is acceptable per device.
- Any custom scripts, automation policies, or alert rules built in Syxsense that need to be rebuilt (not just copied) in Atera.

### Stakeholders
- SJULTRA technicians/engineers (day-to-day RMM users)
- Customers with devices under management (notified of maintenance windows only if needed — no expected end-user impact)
- Billing/admin (Atera subscription setup, Syxsense cancellation timing)

### Success Criteria
- 100% of previously Syxsense-managed devices are checked in and actively monitored in Atera.
- Patch management is running on schedule for Windows and macOS devices.
- Alerting reaches the same notification channel(s) technicians currently rely on.
- Remote access/remote control is verified working per platform.
- No customer-reported gap in support coverage during or after cutover.
- Syxsense contract is not renewed/is cancelled once cutover is confirmed complete.

### Risks & Mitigations
| Risk | Mitigation |
|---|---|
| Coverage gap during parallel run (device monitored nowhere, or double-alerting) | Run a documented cutover checklist per device/customer; track migration status centrally until 100% complete |
| Feature gap discovered mid-migration (e.g., a Syxsense automation with no Atera equivalent) | Pilot first with a low-risk customer before full rollout; surface gaps early |
| Linux devices under-managed post-migration | Explicitly identify every Linux endpoint up front and decide monitoring-only vs. scripted approach per device before cutover |
| Technician unfamiliarity with Atera UI/workflows slows support during transition | Build a short internal quick-reference SOP before full rollout; pilot phase doubles as training |
| Syxsense cancelled too early, before migration is verified complete | Do not cancel/downgrade Syxsense until Check phase confirms 100% migration and a stability window has passed |
| Trial expires (10/1) or account needs attention while owner is on vacation (10/6–10/20) | Convert trial to paid well before departure — target by 09/30 — so nothing needs action while away |
| Migration not finished before vacation, leaving only 9 days after return to hit the 10/30 cutoff | Treat 10/05 as the real internal deadline for full migration (see Key Dates); anything not done by then is high priority in the 10/21–10/29 window |
| No one covering Atera/Syxsense alerts while owner is away 10/6–10/20 | Decide before departure whether Syxsense (for anything not yet migrated) or Atera (for migrated devices) is the system of record for alerts during the gap, and confirm someone/something is watching it, or accept and document the gap |

### Timeline (dated)
| Window | Focus |
|---|---|
| **09/01 – 09/07** | Sign up for Atera 30-day trial; core environment config (company profile, technician accounts, MFA, notification channels) |
| **09/08 – 09/14** | Customer/site structure; agent deployment packages (Windows/macOS/Linux); core RMM policies (patch, monitoring/alerts, scripting library) |
| **09/15 – 09/21** | Integrations (PSA/ticketing); pilot customer migration; validate pilot |
| **09/22 – 09/29** | Convert trial to paid ("go live") once pilot is validated; begin full phased migration |
| **09/30 – 10/05** | Push full phased migration — goal is 100% of customers/devices migrated and stable before vacation |
| **10/06 – 10/20** | Vacation — no active migration; anything not yet migrated stays untouched/stable on whichever platform it's currently on rather than left half-cut-over |
| **10/21 – 10/29** | Finish any remaining migrations; run full Check-phase validation; decommission Syxsense |
| **10/30** | Hard cutoff — Syxsense fully decommissioned/cancelled |

---

## DO

### 1. Sign up for Atera (2026-09-01)
- Start the **30-day free trial** (select the plan/tier based on current endpoint count and required features — confirm patch management and remote access are included at the tier you'll eventually convert to).
- Note the trial expiration date (~2026-10-01) — this account is the one that gets converted to paid, not a separate signup.
- Technician account for this trial: `partners@sjultra.com` (sole technician).
- **2FA is off by default — turn it on manually.** It is *not* auto-enforced the way Atera's help docs describe; there's an **Enable 2FA** toggle in **Admin → Users and security → Security and authentication** that was disabled out of the box. Enable it now rather than waiting.
- **Before 09/30:** convert trial to a paid subscription ("go live") — do this ahead of the 10/6 vacation departure so the account needs no attention while away.

### 2. Core environment configuration
- Set company profile, branding, and default technician roles/permissions.
- Configure notification channels (email, and any chat/PSA integration used for alerts).
- Set default working hours / after-hours alert routing if applicable.

### 3. Customer/site structure
- Recreate the current Syxsense customer/org list as Atera "Customers."
- Map any per-customer groupings (departments, sites, device tags) needed for reporting or policy targeting.

### 4. Agent deployment packages
- Build Windows agent deployment package(s) — confirm silent install method for remote push.
- Build macOS agent deployment package(s) — confirm any macOS-specific permissions (e.g., full disk access, accessibility) required for remote control/scripting.
- Build a Linux install path for the scattered Linux devices (manual/SSH install is acceptable given low volume).

### 5. RMM policy configuration
- Patch management policies: Windows Update and macOS software update policies (schedule, approval rules, reboot behavior).
- Monitoring thresholds and alert rules (disk, CPU, memory, service down, offline agent, etc.) — replicate what's actively used in Syxsense today, not everything Syxsense is capable of.
- Scripting/automation library: rebuild any recurring scripts/tasks currently run through Syxsense.
- Software deployment packages for commonly pushed applications.
- Remote access/remote control configuration and testing.

### 6. Integrations
- Connect PSA/ticketing integration if one is in use (confirm ticket creation from alerts works as expected).
- Connect any documentation or password-management integration currently tied to Syxsense workflows.

### 7. Pilot migration
- Select one low-risk customer (or a handful of internal/test devices).
- Install Atera agent alongside existing Syxsense agent (parallel run).
- Validate monitoring, patching, alerting, and remote access before removing Syxsense from pilot devices.

### 8. Full migration (phased, customer by customer)
- Deploy Atera agent to each customer's devices.
- Run parallel with Syxsense for a short stability window per customer.
- Uninstall Syxsense agent once that customer's devices are confirmed stable in Atera.

### 9. Decommission Syxsense
- Confirm zero active Syxsense agents remain.
- Export/archive any historical Syxsense reporting or asset data worth retaining.
- Cancel or let the Syxsense contract lapse per its terms.

---

## CHECK

- [ ] Every device previously managed in Syxsense is present and checked in on Atera.
- [ ] Patch management is applying updates on the defined schedule for Windows and macOS.
- [ ] Alerts fire correctly and land in the expected channel — test with a deliberate trigger (e.g., simulate a disk-space or offline alert).
- [ ] Remote access/remote control works reliably on a sample of Windows and macOS devices.
- [ ] Any PSA/ticketing integration correctly creates/updates tickets from Atera alerts.
- [ ] Scattered Linux devices have an agreed-upon, documented management approach (monitoring-only vs. scripted) and it's working as intended.
- [ ] Technicians report no missing capability that blocks day-to-day support (collect feedback after pilot and after full rollout).
- [ ] No customer has reported a support or monitoring gap during the transition.
- [ ] Actual Atera cost matches (or beats) the projected savings vs. Syxsense.
- [ ] All rebuilt automations/scripts produce the same result as their Syxsense originals.

## ACT

- Turn the working Atera configuration (agent packages, patch policies, alert rules, scripts) into a repeatable template so new customers are onboarded consistently.
- Write an internal SOP for "Onboarding a new customer into Atera," following the existing SOP creation workflow (see the SJULTRA SOPs process for format/publishing conventions).
- Document any accepted feature gaps vs. Syxsense (especially Linux depth) so they're a known, deliberate tradeoff rather than a surprise later.
- Set a recurring review (e.g., quarterly) of Atera configuration to catch drift and pick up new Atera features worth adopting.
- Confirm Syxsense is fully decommissioned/cancelled and close out the contract.
- Share a short internal recap of what worked and what to do differently next time an RMM/PSA platform changes.
