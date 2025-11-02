# Billing & Payments – Technical Overview

This document explains how patient payments are computed in UDS, the related data and code paths, how to extend or adjust the logic, and common pitfalls to avoid.

## High-level flow

1. A session is created or updated with billable entities (tests, medicines, services, equipments, operations).
2. The system computes the split between patient and assurance (insurer) using assurance coverage and restriction lists.
3. A record is inserted in `payments` for the session with the computed totals; follow-up operations update payment status as money is received.

## Key data sources

- Table `assurances`
	- `percentage_coverage` – e.g., 90 means insurer covers 90%, patient 10%.
	- JSON columns of restricted items: `rstctd_medicines`, `rstctd_tests`, `rstctd_operations`, `rstctd_services`, `rstctd_equipments`.
		- Each is a JSON array of item IDs not covered by the assurance; restricted items are paid 100% by the patient.

- Session entities (stored in table `medical_history`)
	- Arrays per entity type: `medicines`, `tests`, `services`, `equipments`, `operations`.
	- Each item includes at least: `{ id, price, ... }` and may include flags like `servedOut` which excludes the item from patient/assurance computation (see rules below).

- Inventories per hospital
	- Used when recalculating totals to resolve current prices and metadata.

## Core functions and files

### 1) Payment calculator

File: `src/utils/calculate.payments.controller.js`

Exported API:

```js
export async function calculatePayments(assuranceId, itemGroups, type)
```

Inputs:
- `assuranceId` – the active assurance for the patient (required).
- `itemGroups` – object mapping entity names to arrays of items:
	- `{ medicines: [...], tests: [...], operations: [...], services: [...], equipments: [...] }`
- `type` – any truthy value triggers computation; falsy returns nothing (by current implementation).

Returns (when `type` is truthy):

```js
{ assurance_amount: number, patient_amount: number }
```

Algorithm summary:
- Load assurance coverage and restriction lists:
	- Query joins `assurances` with item tables (via JSON_CONTAINS) and groups by assurance id.
	- Parse JSON arrays from `assurances` into `restrictedItems` per entity type.
- If no assurance record is found, everything is billed to the patient (non-served items only).
- Otherwise, iterate each entity list and each item:
	- Skip items with `servedOut === true`.
	- If the item id is in the entity’s restriction list, bill 100% to patient.
	- Else split the item price into insurer and patient shares by `percentage_coverage`.

Formula:
- Let coverage in percent be `c`, price be `p`.
- Insurer portion: `p_insurer = p * c / 100`
- Patient portion: `p_patient = p * (100 - c) / 100`

Notes and caveats:
- The function only returns a value when `type` is truthy (e.g. `'all'`, `'type'`). If called with a falsy `type`, it currently returns `undefined`.
- Items with `servedOut` are ignored in both totals.
- Known issue (bug): the mapping for `equipments` uses `rstrct_medicines` instead of `rstrct_equipments`. See “Known issues” for the fix.

### 2) Session creation and payment insertion

File: `src/controllers/patient.session.controller.js` (function `addSession`)

Relevant steps:
1. Collect entities: `medicines`, `tests`, `services`, `equipments`, `operations`.
2. Build `addins` with the entities (after inventory normalization):
	 ```js
	 const addins = { medicines, tests, operations, services, equipments };
	 ```
3. Compute totals:
	 ```js
	 const pts = await calculatePayments(assurance, addins, 'all');
	 ```
4. Some flows also compute a `totalPPrice` (e.g., cost of medicines from inventory purchase price) and add it to `pts.patient_amount` before persisting.
5. Insert into `payments`:
	 ```sql
	 insert into payments(id,user,session,amount,assurance_amount,pati_p_amount,status,assu_paym_status,date,assurance,approver,extra)
	 ```
	 - `amount` = `pts.patient_amount`
	 - `assurance_amount` = `pts.assurance_amount`
	 - `pati_p_amount` = initial patient-paid amount (set to `amount` when action is `create-n-pay`)
	 - `status` = `'paid'` in `create-n-pay` flow else `'awaiting payment'` (unless assurance covers 100%).
	 - `assu_paym_status` = `'pending payment'` by default, or `'paid'` if a special case (e.g. full coverage or specific assurance id).

### 3) Recalculation for an existing session

File: `src/controllers/patient.session.controller.js` (function `calculateSessionTotals`)

Purpose: Recompute totals for an existing session, using current prices and assurance (or an override).

Process:
- Load session, parse the entity arrays.
- Load inventories per entity for the session’s hospital.
- Build a fresh `addins` array of items with prices (possibly updated from inventories).
- Compute totals with `calculatePayments(assuranceId, addins, 'all')`.
- Add `totalPPrice` (e.g., accumulated purchase prices) to `totals.patient_amount`.
- Determine a session-level status (e.g., `'partially paid'`, `'paid'`) using existing `payments` row(s) and coverage.

This function is a good place to centralize future business rules that need to be applied across the entire session (discounts, caps, VAT, etc.).

### 4) Other related endpoints

- `processPayment`, `approvePayment`, `approveAssuPayment` – update payment records and statuses.
- `markMedicineAsServed` – toggles serving state for medicines; served items are excluded from the totals in `calculatePayments`.

## Extending the payment logic

Below are common scenarios and where to change the code.

### Add a new billable entity type
1. Schema: add a JSON column `rstctd_<entity>` to `assurances` (e.g., `rstctd_therapies`).
2. Backend models:
	 - Include the new entity in session storage (`medical_history`).
	 - Add inventory retrieval for the new entity.
3. Calculator:
	 - Update `calculatePayments` to:
		 - Parse `rstctd_<entity>` into `restrictedItems.<entity>`.
		 - Loop the new entity array in `itemGroups`.
4. UI/API:
	 - Ensure the entity carries `{ id, price, servedOut? }`.
	 - Include it in `addins` where totals are computed.

### Apply discounts, copays, or caps
- Add a post-processing step after the base split:
	- Option A: Modify `calculatePayments` to accept an optional policy descriptor and adjust `assurance_amount`/`patient_amount` accordingly.
	- Option B: Apply adjustments in `calculateSessionTotals` after calling `calculatePayments` (easier to centralize and log policy effects).
- Examples:
	- Fixed co-pay: add a per-item or per-session fixed amount to `patient_amount`.
	- Out-of-pocket cap: clamp `patient_amount` to a max and reassign excess to `assurance_amount`.

### Taxes (VAT) or surcharges
- Compute tax from the split or on gross line-item price, depending on policy.
- Prefer applying in `calculateSessionTotals` to keep the base calculator simple and reusable.

### Rounding rules
- Decide whether to round per-item or at the session aggregate.
- Implement consistent rounding (e.g., to the nearest RWF) right before persisting totals.

## Known issues and recommendations

1) Equipment restriction mapping
- In `calculate.payments.controller.js`, `restrictedItems` for `equipments` is incorrectly sourced from `rstrct_medicines`:

```js
restrictedItems = {
	medicines: assuranceInfo.rstrct_medicines,
	tests: assuranceInfo.rstrct_tests,
	operations: assuranceInfo.rstrct_operations,
	equipments: assuranceInfo.rstrct_medicines, // BUG: should be rstrct_equipments
	services: assuranceInfo.rstrct_services,
}
```

Fix:

```js
equipments: assuranceInfo.rstrct_equipments,
```

2) Result on falsy `type`
- `calculatePayments` only returns a sum when `type` is truthy. If you need the function to return totals by default, add a default truthy value or adjust callers.

3) Query shape
- The calculator loads assurance JSON lists via a `SELECT` with multiple LEFT JOINs and `JSON_CONTAINS` filters, then `GROUP BY assurances.id`.
- You can simplify this to a single-row `SELECT` from `assurances` by `id` and parse the JSON arrays directly, since coverage and restriction arrays are properties of the assurance, not of individual joined rows.

## Examples

### Example 1: Standard split

Input:

```js
assuranceId = 123;
itemGroups = {
	tests: [{ id: 10, price: 200 }],
	medicines: [{ id: 99, price: 50 }]
};
coverage = 80; // from assurances
restricted lists = all empty
```

Output:

```js
{
	assurance_amount: 0.8 * (200 + 50) = 200,
	patient_amount:   0.2 * (200 + 50) = 50
}
```

### Example 2: Restricted medicine

Input:

```js
restricted.medicines = [99]
itemGroups.medicines = [{ id: 99, price: 50 }]
coverage = 80
```

Output:

```js
{
	assurance_amount: 0,      // insurer does not cover restricted items
	patient_amount:   50
}
```

## Where to plug in new features

- Quick adjustments to totals: `calculateSessionTotals` (centralized, safe for cross-entity rules).
- Deep per-item logic: `calculatePayments` (when the rule affects the base split and/or restricted logic).
- Payment status transitions: `processPayment`, `approvePayment`, `approveAssuPayment` (e.g., add audit logs, receipts, notifications).

## Testing checklist

- No-assurance path: All items billed to patient.
- Partial coverage: Correct split by coverage percent.
- 100% coverage: Patient share should be 0 unless restricted.
- Restricted items: Billed entirely to patient.
- `servedOut` items: Excluded.
- Recalculation: `calculateSessionTotals` reflects inventory price updates and policy adjustments.
- Status transitions: Verify session status updates when patient or assurance payments are approved.

---

Revision date: 2025-11-02
