# PNM Device Trail — Context & Open Items

**Last updated:** Apr 2026  
**HTML file:** `pnm_open_items.html` (same folder)

---

## Objective

Wiom must know — at any point in time — the identity, current partner assignment, status, and full trail of every PNM device. PNM devices must never be installed on a customer. Partner-level accountability must be defined and enforced in the system.

---

## What is a PNM Device?

A router placed at a partner's location solely to monitor internet health. Cannot be installed on a customer. Distinguished from regular devices (GX, SY series) by device ID pattern (PNM001…).

---

## Assumption Taken

**Wiom holds full ownership and responsibility for PNM devices.** Partner has no financial liability — no deposit collected, no loss process applies to the partner. Needs Dhruv's confirmation to finalise.

---

## Fundamental Questions

| # | Question | Status |
|---|----------|--------|
| 1 | What is a PNM device? | Sorted |
| 2 | Is the partner liable, or is it solely Wiom's responsibility? | Assumed (Wiom owns) — confirm with Dhruv |
| 3 | How is a PNM device identified in the system? | Open — naming pattern + explicit SSOT flag needed |
| 4 | Which partner has which PNM device, and where does that mapping live? | Open — Shariq's sheet exists, needs to move into system |
| 5 | What states can a PNM device be in? | Defined: `in_warehouse`, `in_transit`, `idle` |
| 6 | Can a PNM device be accidentally installed on a customer? | Checklist — validate customer team's checks |
| 7 | How does Wiom monitor a PNM device's internet health? | Open — NAS ID auto-generation to be built; Nikhil monitors |
| 8 | What happens when a PNM device is lost or damaged? | Defined: escalate to Shariq + city head + agent for offline check |
| 9 | How does a PNM device get returned to Wiom? | Defined: follows CSP router return flow (logistics only) |

---

## Key Design Decisions

### PNM Identity in System
- **Naming pattern** (PNM001…) = natural source at procurement
- **Explicit flag in SSOT** (`device_type = 'PNM'`) = enforced identity for all system logic
- Flag must be set at NAS ID creation — not patched later
- All checks (installation guard, loss process, partner assignment) must check the flag

### SSOT Fields
- `lco_account_id` stays as **Wiom PNM partner** throughout
- New field `pnm_partner_id` = real partner where device is physically located
- `state` field: valid values for PNM devices only — `in_warehouse`, `in_transit`, `idle`
- Any activity attempted outside these states (PUT, swap, migrate, customer install) → **BLOCKED**
- TBD with Rahul on exact state-check mechanism

### Partner Liability (Assumption)
- No ₹200 deposit
- No PSF impact on device return
- No change to partner's device order eligibility on return
- Wiom absorbs loss if device is not recovered

### NAS ID Automation
- Once a device enters PNM DB → NAS ID auto-generated and added to Remote T_Device
- Asif's input needed for one-time setup, not ongoing
- NAS ID generation fires an event → SSOT `nas_id` updated against `device_id` automatically
- Eliminates the manual Jaivin → Rahul update step

---

## Ideal Process Flow

### Stage 1: Ordering
- Shariq raises request on Partner App
- Warehouse prepares devices — picks PNM devices only per Shariq's request

### Stage 2: Dispatch
- Pyrops identifies PNM devices via PNM flag, dispatches to City Heads
- Auto-notifies Wiom on dispatch
- **SSOT updated:** `state = in_transit`, `device_type = PNM`

### Stage 3: Assignment on Wiom Hub (new tab)
- Wiom Hub shows all dispatched PNM devices in a new PNM Devices tab
- Shariq assigns `partner_id` to each device (single or bulk)
- Shariq assigns City Head to each device/partner
  - *Later:* auto-assign logic if partner → city head mapping is deterministic
- **PNM DB updated:** `partner_id`, `city_head_id`
- **SSOT updated:** `pnm_partner_id` (new field)

### Stage 4: Agent Assignment & Installation
- City Head assigns device to agent on Hub
- **PNM DB updated:** `agent_id`, `assigned_at`
- Agent receives device from City Head
- Device enters PNM DB → NAS ID auto-generated → added to Remote T_Device
- **PNM DB updated:** `nas_id`; **SSOT updated:** `nas_id` via event

### Stage 5: Installation Verification
- Agent installs device at partner office, uploads photo proof
- **PNM DB updated:** `photo_url`, `install_attempted_at`, `install_partner_id`
- Backend checks: is device online?
  - **Yes** → Installation verified. SSOT: `state = idle`, `pnm_partner_id` confirmed. PNM DB: `verified_at`, `is_verified = true`. Nikhil monitors health.
  - **No (X hours elapsed)** → Alert to City Head + Agent → re-attempt installation

### Stage 6: Active (Device at Partner)
- Device sits at partner office — no action required
- State: `idle`
- Any activity attempted (customer install, PUT, swap, migrate, scan) → **BLOCKED** via SSOT state check

### Stage 7: Terminal States

**Device Returned to Wiom:**
- Follows CSP router return flow (logistics only)
- No PSF impact, no change to partner's order eligibility (no deposit was collected)
- Open question: can this device be reused as a normal device?

**Partner Declares Device Lost:**
- Escalate to Shariq + assigned City Head + Agent
- Offline asset location check
  - Device found → recovery triggers return flow
  - Device not found → mark as lost in SSOT (further process TBD)

---

## PNM DB — Data Points

| Field | Captured at |
|-------|-------------|
| `device_id` | Procurement |
| `partner_id` | Shariq assigns on Hub |
| `city_head_id` | Shariq assigns on Hub |
| `agent_id` | City Head assigns on Hub |
| `assigned_at` | City Head assigns on Hub |
| `nas_id` | Auto-generated on PNM DB entry |
| `photo_url` | Agent uploads at installation |
| `install_attempted_at` | Agent uploads at installation |
| `install_partner_id` | Agent uploads at installation |
| `verified_at` | Backend confirms device online |
| `is_verified` | Backend confirms device online |

---

## Open Items

| Item | Owner | Status |
|------|-------|--------|
| Confirm ownership assumption | Jaivin → Dhruv | Pending |
| PNM identity flag in SSOT — align schema | Jaivin → Rahul + Asif | Pending |
| Migrate Shariq's sheet into system | Jaivin + Shariq + Rahul | Pending (unblocked after Dhruv) |
| State-check blocking mechanism in SSOT | Jaivin → Rahul | TBD |
| NAS ID auto-generation setup | Jaivin → Asif | Pending |
| Validate installation guard (customer team) | Jaivin → customer team | Checklist |
| Can returned PNM device be reused as normal? | Jaivin → Dhruv | Open |
| Lost device — full process (if not recovered) | Jaivin | TBD |
| Auto-assign city head logic (if deterministic) | Jaivin | Later |
| Ask Nikhil: how is device-plugged-in verified today? | Jaivin → Nikhil | Pending |
| Visit PNM dashboard and map to new process | Jaivin | Pending (needs Playwright / screenshots) |

---

## People

| Name | Role |
|------|------|
| Shariq | Owns Wiom PNM partner, places orders, assigns partner IDs |
| Dhruv | Business decision-maker, aligns with Asif |
| Asif | Generates NAS IDs, adds to Remote T_Device |
| Rahul | SSOT table owner |
| Nikhil | PNM health monitoring |
| Fahad | City Head — Delhi |
| Ajinkya | City Head — Mumbai |
| Anurag | City Head — Bharat |
| Jaivin | Product — driving this initiative |
