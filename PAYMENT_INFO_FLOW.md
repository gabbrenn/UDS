# Payment Info Calculation Flow

## Overview
This document shows where `payment_info` is calculated and how it relates to `partP` (individual entity payment calculations).

## 1. Initial payment_info Setup (Lines 3978-4002)

`payment_info` is **initially loaded from the database**:

```javascript
// Line 3978-4002
let parsedPayment = payment ? {
  id: payment.id,
  date: payment.date,
  extra: safeParse(payment.extra, []),
  status: payment.status,
  a_amount: payment.a_amount || 0,        // From database
  datepaid: payment.datepaid || null,
  p_amount: payment.p_amount || 0,        // From database
  pati_p_amount: payment.pati_p_amount || 0, // From database
  paymentSummary: {},
  pp_amount: payment.pp_amount || 0        // From database (if exists)
} : { /* defaults with 0 values */ };
response.payment_info = parsedPayment;
```

**Key Point**: `payment_info` values (`a_amount`, `p_amount`, `pp_amount`) come from the **database**, NOT from aggregating `partP` results.

---

## 2. Individual Entity Calculations (partP)

Each entity (medicines, services, equipments, tests) calculates its own `partP` using `calculatePayments()`:

### Medicines (Lines 4147-4159)
```javascript
let partP = await calculatePayments(
  response.assurance_info.id,
  { medicines: [{ ...medicine, quantity: rawM.quantity || 0, price: finalPrice, servedOut: false }] },
  'medicines',
  { hospitalId: response.hp_info.id }
);
Object.assign(medicine, {
  pati_amount: partP.patient_amount,      // Assigned to individual medicine
  assurance_amount: partP.assurance_amount, // Assigned to individual medicine
  private_amount: partP.private_amount || 0 // Assigned to individual medicine
});
totalPPrice += partP.private_amount;       // Accumulated for pp_amount
```

### Services (Lines 4212-4226)
```javascript
let partP = await calculatePayments(
  response.assurance_info.id,
  { services: [{ ...services, quantity: sQty, price: finalPrice, total_amount: sTotal }] },
  'services',
  { hospitalId: response.hp_info.id }
);
Object.assign(services, {
  pati_amount: partP.patient_amount,       // Assigned to individual service
  assurance_amount: partP.assurance_amount, // Assigned to individual service
  private_amount: partP.private_amount || 0  // Assigned to individual service
});
totalPPrice += partP.private_amount;        // Accumulated for pp_amount
```

### Equipments (Lines 4258-4273)
```javascript
let partP = await calculatePayments(
  response.assurance_info.id,
  { equipments: [{ ...equipment, quantity: eQty, price: finalPrice, total_amount: eTotal }] },
  'equipments',
  { hospitalId: response.hp_info.id }
);
Object.assign(equipment, {
  pati_amount: partP.patient_amount,       // Assigned to individual equipment
  assurance_amount: partP.assurance_amount, // Assigned to individual equipment
  private_amount: partP.private_amount || 0  // Assigned to individual equipment
});
totalPPrice += partP.private_amount;        // Accumulated for pp_amount
```

### Tests (Lines 4326-4338 for director_general, 4381-4384 for cashier/dof/etc)
```javascript
let partP = await calculatePayments(
  response.assurance_info.id,
  { tests: [tests] },
  'tests',
  { hospitalId: response.hp_info.id }
);
Object.assign(tests, {
  pati_amount: partP.patient_amount,       // Assigned to individual test
  assurance_amount: partP.assurance_amount, // Assigned to individual test
  private_amount: partP.private_amount || 0  // Assigned to individual test
});
totalPPrice += partP.private_amount || 0;  // Accumulated for pp_amount
```

**Key Point**: Each `partP` result is assigned to the **individual entity** (medicine, service, equipment, test), NOT aggregated into `payment_info`.

---

## 3. totalPPrice Accumulation

`totalPPrice` accumulates **only the private_amount** from all entities:
- Medicines: `totalPPrice += partP.private_amount` (line 4158)
- Services: `totalPPrice += partP.private_amount` (line 4225)
- Equipments: `totalPPrice += partP.private_amount` (line 4272)
- Tests (director_general): `totalPPrice += partP.private_amount || 0` (line 4337)
- Tests (cashier/dof/etc): `totalPPrice += partP.private_amount || 0` (line 4384)

---

## 4. Final payment_info.pp_amount Update (Lines 4487-4490)

**Only `pp_amount` is updated** from `totalPPrice`:

```javascript
// Line 4487-4490
// pp_amount is computed above from rules; retain existing totalPPrice only if pp_amount wasn't set
if (typeof response.payment_info.pp_amount !== 'number' || response.payment_info.pp_amount === 0) {
  response.payment_info.pp_amount = totalPPrice;
}
```

**Key Point**: 
- If `pp_amount` exists in database and is > 0 (computed from rules), it's kept
- If `pp_amount` is 0 or not set, it uses `totalPPrice` (sum of all private amounts)

---

## 5. Commented-Out Aggregation (Lines 4419-4438)

There's a **commented-out section** that would aggregate ALL `partP` results into `payment_info`:

```javascript
// Lines 4419-4438 (COMMENTED OUT)
// const computedTotals = await calculatePayments(
//   response.assurance_info.id,
//   {
//     medicines: Array.isArray(response.medicines) ? response.medicines.map(m => ({ ...m, servedOut: false })) : [],
//     services: Array.isArray(response.services) ? response.services : [],
//     equipments: Array.isArray(response.equipments) ? response.equipments : [],
//     tests: Array.isArray(response.tests) ? response.tests : [],
//   },
//   'all',
//   { hospitalId: response.hp_info.id }
// );
// response.payment_info.a_amount = computedTotals.assurance_amount || 0;
// response.payment_info.p_amount = computedTotals.patient_amount || 0;
// response.payment_info.pp_amount = computedTotals.private_amount || 0;
```

**This is NOT currently used** - `payment_info` values come from the database, not from aggregating `partP` results.

---

## Summary

### Current Flow:
1. **payment_info** is loaded from database (lines 3978-4002)
2. **Each entity** calculates its own `partP` and assigns amounts to itself (lines 4147-4387)
3. **totalPPrice** accumulates only `private_amount` from all entities
4. **payment_info.pp_amount** is updated from `totalPPrice` ONLY if it wasn't already set (line 4489)
5. **payment_info.a_amount** and **payment_info.p_amount** remain from database, NOT from `partP` aggregation

### Relationship:
- `partP` results → Assigned to **individual entities** (medicine, service, equipment, test)
- `partP.private_amount` → Accumulated into `totalPPrice` → Used for `payment_info.pp_amount`
- `payment_info.a_amount` and `payment_info.p_amount` → From **database**, NOT from `partP`

### The Issue:
Currently, `payment_info.a_amount` and `payment_info.p_amount` come from the database and may not match the sum of all `partP` results. The commented-out section (lines 4419-4438) would fix this by aggregating all `partP` results, but it's disabled.

