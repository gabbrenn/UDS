# Insurance Fixed Coverage Rules

This feature lets hospitals define a per-item, unit-based coverage cap for a given insurance. If a rule exists, the insurer will only cover up to the capped amount (per quantity), and any excess is counted as a private/top-up amount.

## What changed

- Payment calculator now applies a fixed coverage cap when rules exist and the hospital id is provided.
- Session flows pass the current hospital id so rules are considered.
- We record the computed private/top-up amount in `payments.extra` on initial session creation (and in recalculation results).

## Data model

New table: `insurance_rules` (see `src/Database/insurance_rules.sql`)
- id: bigint (PK, auto-increment)
- hospital_id: string
- insurance_id: string
- entity_type: string — one of: `medicine`, `test`, `service`, `equipment`, `operation`
- entity_id: string (the item’s id)
- coverage_amount: decimal(12,2) — unit-based cap in RWF
- created_at, updated_at

Primary key: `id`
Uniqueness: (`hospital_id`, `insurance_id`, `entity_type`, `entity_id`)

Example insert
- INSERT INTO insurance_rules (hospital_id, insurance_id, entity_type, entity_id, coverage_amount)
  VALUES ('<hp_id>', '<assurance_id>', 'medicine', '<medicine_id>', 500.00);

## Code flow

- Calculator: `src/utils/calculate.payments.controller.js`
  - New helper `getInsuranceRule(hospitalId, assuranceId, entityType, entityId)` queries `insurance_rules`.
  - `calculatePayments(assuranceId, items, type, { hospitalId })` now:
    - Computes the line’s actual total.
    - If a rule exists, caps insurer coverage to `coverage_amount * quantity`.
    - Adds any excess above the cap into `sum.private_amount`.
    - Still applies percentage coverage to the (capped) insurance base.

- Controllers updated to pass `{ hospitalId }` to `calculatePayments`:
  - `addSession` (already passing)
  - Add test/medicine to session endpoints
  - Session viewer aggregation for medicines/services/equipments/tests
  - Change-insurance recalculation
  - Recalculation helper `calculateSessionTotals` (already passing)

## Persistence

- On session creation, `payments.extra` stores an entry:
  - `{ type: 'private-amount', amount, id, date }`
- On recalculation (via `calculateSessionTotals`) the returned `totals` includes `private_amount`. You can choose to persist it similarly if needed.

## Edge cases
- No rule: coverage remains percentage-only (no private top-up added).
- Zero or negative coverage_amount: treated as no rule.
- Quantities: cap scales as `coverage_amount * quantity`.
- Served-out items: excluded from insurer coverage as before.

## How to deploy
1. Apply SQL:
   - Run `src/Database/insurance_rules.sql` on your database.
  - If you previously created the table with a composite primary key, migrate:
    1) Add id as AUTO_INCREMENT with a temporary unique index
      ALTER TABLE insurance_rules ADD COLUMN id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT FIRST, ADD UNIQUE KEY uk_tmp_id (id);
    2) Switch PK and keep logical uniqueness
      ALTER TABLE insurance_rules DROP PRIMARY KEY, ADD PRIMARY KEY (id), ADD UNIQUE KEY uk_insurance_rules (hospital_id, insurance_id, entity_type, entity_id);
    3) Drop temporary index
      ALTER TABLE insurance_rules DROP INDEX uk_tmp_id;
2. Restart/redeploy backend.
3. Optional: Add UI for DoF to manage rules (CRUD) and expose private/top-up on cashier reports.

### New API endpoints (admin/dof)
- POST `/get-insurance-rules`
  - body: { hospital_id (required), insurance_id?, entity_type?, entity_id?, limit?, offset? }
  - returns: list of matching rules
- POST `/upsert-insurance-rule`
  - body: { hospital_id, insurance_id, entity_type, entity_id, coverage_amount }
  - behavior: insert or update coverage for the tuple; returns the saved row
- POST `/delete-insurance-rule`
  - body: { id } or { hospital_id, insurance_id, entity_type, entity_id }
  - behavior: deletes a single rule; returns affected count

## Files touched
- `src/utils/calculate.payments.controller.js`: rule helper, cap logic, `private_amount` tracking.
- `src/controllers/patient.session.controller.js`: pass hospital id to all calls to `calculatePayments`; store private amount on initial create.
- `src/Database/insurance_rules.sql`: new table schema.

## Notes
- The code expects `entity_type` singular names: `medicine`, `test`, `service`, `equipment`, `operation`.
- If your IDs are numeric in your DB, passing them as strings is okay; they’re matched using `String(entity_id)`.

---

## Payment workflow (current)

This section documents the live behavior of pricing, totals, and approvals (November 2025), with special focus on insurance rules and percentage coverage.

### What totals mean on the page

- Insurance’s fees: sum of what the insurer pays across all lines (after applying any per-unit caps and the insurance percentage).
- Patient’s fees: sum of co‑pays on the covered base (does NOT include top‑up/private).
- Patient’s private fees: sum of top‑ups (amounts above the per-unit cap). If no cap applies, this is 0.
- Total Amount (gross): Insurance’s fees + Patient’s fees + Patient’s private fees.
- What patient actually owes: Patient’s fees + Patient’s private fees − Patient’s paid fees.

### How line items are calculated (cap + percent)

For each non-served-out line:
1) Actual line total: `actual_line_total = unit_price × quantity`.
2) Apply fixed coverage rule (if exists):
   - `rule_line_total = coverage_amount × quantity`.
   - `insurance_base = min(actual_line_total, rule_line_total)`.
   - `top_up = max(0, actual_line_total − insurance_base)` → accumulated into `private_amount`.
   - If no rule, `insurance_base = actual_line_total`, `top_up = 0`.
3) Apply percentage coverage (from Assurance):
   - `insurance_pays += insurance_base × (percentage_coverage / 100)` → `assurance_amount`.
   - `patient_copay += insurance_base × (1 − percentage_coverage/100)` → `patient_amount`.
4) Gross total accumulates as `total_amount += actual_line_total`.

Additional medicine patient price: When recomputing with `calculateSessionTotals`, the system adds the configured `p_price` for medicines to the patient portion: `totals.patient_amount += totalPPrice` (where `totalPPrice = Σ medicine.p_price × qty`).

### Functions and return shapes

- calculatePayments
  - File: `src/utils/calculate.payments.controller.js`
  - Signature: `calculatePayments(assuranceId, items, type, { hospitalId })`
  - Inputs:
    - `assuranceId`: insurance id.
    - `items`: an object like `{ medicines: [], services: [], equipments: [], tests: [], operations: [] }` with entries shaped like `{ id, price, quantity?, servedOut? }`. `price` is the line total (unit × qty) used when provided.
    - `type`: category or `'all'`.
    - `hospitalId`: used to look up per-unit cap rules.
  - Returns:
    - `{ assurance_amount, patient_amount, private_amount, total_amount }`.

- calculateSessionTotals
  - File: `src/controllers/patient.session.controller.js`
  - Signature: `calculateSessionTotals(sessionId, overrideAssurance?)`
  - Behavior:
    - Loads session, inventories, computes line `price` values with FPPA/FPPA2, builds `addins` payload.
    - Calls `calculatePayments` with `{ hospitalId: session.hospital }`.
    - Adds `totalPPrice` (medicines’ `p_price × qty`) to `totals.patient_amount`.
    - Reads current `payments` row to derive a session-level `status` only (paid/partially/awaiting) based on `pati_p_amount` vs. `amount + pp_amount` and 100% coverage.
  - Returns: `{ totals, totalPPrice, assurance, hospital, status }`.

### Approve Payment (server)

- Endpoint: `POST /approve-payment`
- Guards: user role + `authorizeSession('ismyfacilty')` (user and session must belong to same hospital unless `system`).
- Logic (file: `src/controllers/patient.session.controller.js`):
  - Loads `payments` row and computes `totalDue = amount + pp_amount`.
  - Defaults `amount` to `totalDue − pati_p_amount` when omitted.
  - Prevents overpay; updates `pati_p_amount` and `extra` (append `{ type: 'patient-payment', method, amount, approver, date }`).
  - Status set to `paid` if `assurance.percentage_coverage == 100` OR `pati_p_amount ≥ totalDue`; otherwise `partially paid`.

### UI behavior (reverted default for now)

- Approve Payment modal (file: `src/resources/assets/js/components/sessionpopups.js`):
  - Default amount now uses only the patient’s co‑pay: `p_amount − pati_p_amount`.
  - Note: This default excludes private/top‑up (`pp_amount`). Users can still manually enter a different amount if the workflow requires collecting top‑up at approval time.

- Request Payment popup (file: `src/utils/payments.popup.controller.js`):
  - Remaining amount (`ra`) now uses the same co‑pay logic: `pmD.p_amount − pmD.pati_p_amount`.

This revert keeps the UI defaults conservative while the backend keeps full awareness of top‑up (`pp_amount`) for totals and “fully paid” checks.

### Quick example

Insurance% = 85%, DoF cap = 6,000 per unit, Qty = 1, Actual price = 10,000:

- Covered base = `min(10,000, 6,000) = 6,000`
- Private/Top‑up = `10,000 − 6,000 = 4,000`
- Insurance pays = `6,000 × 85% = 5,100`
- Patient’s fees = `6,000 × 15% = 900`
- Patient owes = `900 + 4,000 = 4,900` (minus any amount already paid)

### Edge cases

- No rule: coverage remains percentage-only (no private top‑up added).
- Zero or negative `coverage_amount`: treated as no rule.
- Quantities: cap scales as `coverage_amount × quantity`.
- Served-out items: fully charged to patient; insurer share not applied to served-out lines.

### Files changed recently and why

- `src/utils/calculate.payments.controller.js`
  - Added fixed-cap logic and `private_amount` accumulation; returns a full breakdown.

- `src/controllers/patient.session.controller.js`
  - Passes `{ hospitalId }` into `calculatePayments` in all relevant flows (session creation, add entities, viewer, change-insurance, recalculation).
  - Persists a summary entry to `payments.extra` including a `private-amount` record for transparency on session creation.
  - Approve Payment computes `totalDue = amount + pp_amount` to decide paid/partial.

- UI popups
  - `src/resources/assets/js/components/sessionpopups.js`: reverted modal default payable amount to co‑pay only.
  - `src/utils/payments.popup.controller.js`: reverted requested amount (`ra`) to co‑pay only.

These changes align the totals and reports with fixed-cap rules while keeping the cashier UI uncomplicated by default.
