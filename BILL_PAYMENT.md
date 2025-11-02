# Understanding UDS Billing & Payments

This guide explains how UDS calculates patient bills and handles payments. Whether you're a developer maintaining the system or trying to understand how bills are calculated, this document will help you understand the process step by step.

## How Billing Works - Simple Overview

When a patient receives medical care:

1. Every billable item (like medicines, tests, or services) gets added to their session
2. The system checks their insurance (assurance) coverage:
   - What percentage the insurance covers (like 90% insurance, 10% patient)
   - Which items are not covered by insurance (restricted items)
3. The system creates a bill splitting the total between:
   - What the patient needs to pay
   - What the insurance company will pay

## Understanding the Data

### 1. Insurance (Assurance) Information
The system stores insurance information including:
- Coverage percentage (example: 90% means insurance pays 90%, patient pays 10%)
- Lists of items not covered by insurance, such as:
  - Restricted medicines
  - Restricted tests
  - Restricted operations
  - Restricted services
  - Restricted equipment
  
These "restricted" items must be paid 100% by the patient, regardless of their insurance coverage.

### 2. Patient Session Data
When a patient visits, we track:
- All medicines prescribed
- Tests performed
- Medical services provided
- Equipment used
- Operations performed

For each item we store:
- The item ID
- The price
- Whether the item has been given to the patient ("served out")
  - Items marked as "served out" are not included in the bill

### 3. Hospital Price Lists
Each hospital maintains its own inventory with current prices. These prices are used when:
- Creating new bills
- Recalculating existing bills (prices may change over time)

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

### Creating a New Bill

When a new patient session is created (`src/controllers/patient.session.controller.js`), here's what happens:

#### Step 1: Collect All Items
The system gathers everything that needs to be billed:
- Medicines prescribed
- Tests performed
- Services provided
- Equipment used
- Operations performed

#### Step 2: Calculate the Bill
1. Get current prices from the hospital's inventory
2. Calculate how much insurance will pay and how much the patient owes
3. For medicines, we also track the purchase price (this helps with inventory management)

#### Step 3: Create the Payment Record
The system creates a payment record with:
- Total amount the patient needs to pay
- Total amount insurance will pay
- How much the patient has paid so far
- Payment status:
  - "paid" if patient pays immediately
  - "awaiting payment" if patient will pay later
  - Insurance status starts as "pending payment"
  - Sometimes insurance is marked "paid" automatically (like for 100% coverage)

### Updating an Existing Bill

Sometimes we need to recalculate a bill that was already created. This happens in `calculateSessionTotals` function and here's how it works:

#### When This Happens
- When prices in the hospital inventory change
- When insurance coverage changes
- When adding new items to an existing session
- When applying special discounts or adjustments

#### How It Works
1. Get the latest information:
   - All items in the session
   - Current prices from hospital inventory
   - Current insurance coverage

2. Recalculate everything:
   - Calculate new insurance and patient amounts
   - Include purchase prices for inventory tracking
   - Update payment status (paid, partially paid, etc.)

This is also where we can add special rules like:
- Discounts for certain patients
- Maximum out-of-pocket limits
- Special holiday rates
- VAT or other taxes

### Other Important Functions

#### Payment Processing
We have several functions that handle different parts of the payment process:
- Processing a payment from a patient
- Approving an insurance payment
- Marking medicines as "served" (given to patient)

#### Payment Status Changes
- When a patient makes a payment → Update their payment status
- When insurance pays → Update insurance payment status
- When medicine is given to patient → Mark as "served" (won't be billed again)

## Adding New Features

Here are common scenarios and how to implement them:

### Adding a New Type of Billable Item
Let's say you want to add "therapy sessions" as a new billable item:

1. Add to Insurance System:
   - Add a new list for restricted therapy sessions
   - This lets insurance companies specify which therapies they don't cover

2. Update Patient Records:
   - Add therapy sessions to patient history
   - Track prices in hospital inventory

3. Update Bill Calculator:
   - Make it check insurance coverage for therapy sessions
   - Include therapy costs in bill calculations

4. Update User Interface:
   - Add fields to enter therapy information
   - Show therapy costs in bills

### Adding Special Pricing Rules

#### Discounts and Copays
You can add special pricing rules in two ways:

1. In the main calculator:
   - Add rules that affect how costs are split between patient and insurance
   - Good for simple discounts that apply everywhere

2. In the bill recalculation:
   - Add rules after the basic calculation
   - Better for complex rules that need extra information
   - Easier to track and log changes

Examples:
- Fixed copay: Patient pays 2000 RWF for each medicine
- Maximum limit: Patient never pays more than 50,000 RWF total
- Senior discount: 10% off for patients over 65

#### Adding Taxes
When adding VAT or other taxes:
1. Calculate the basic split between patient and insurance
2. Add taxes afterward in the bill recalculation
This keeps the basic calculator simple and makes tax calculations easier to track

#### Rounding Numbers
Important decisions:
- Round each item separately? Or round the final total?
- Always round to nearest RWF
- Apply rounding just before saving the final bill

## Known Issues to Watch Out For

### 1. Equipment Insurance Coverage Bug
There's a bug in how equipment coverage is checked. The system is looking at the wrong restriction list:

```js
// In calculate.payments.controller.js
restrictedItems = {
    medicines: assuranceInfo.rstrct_medicines,
    tests: assuranceInfo.rstrct_tests,
    operations: assuranceInfo.rstrct_operations,
    equipments: assuranceInfo.rstrct_medicines,  // BUG: Using medicines list instead of equipment list!
    services: assuranceInfo.rstrct_services,
}
```

How to fix it:
```js
equipments: assuranceInfo.rstrct_equipments,  // Use the correct equipment list
```

### 2. Payment Calculation Type Parameter
When calling the payment calculator:
- You must pass a "type" parameter (like 'all' or 'type')
- If you don't, it returns nothing
- To fix this in your code, always pass 'all' as the type

### 3. Database Query Improvement
Current situation:
- System makes complex database queries to get insurance information
- Uses multiple table joins which can be slow

Suggested improvement:
- Just get the insurance record directly
- Parse the restriction lists from that record
- This would be faster and simpler

## Real-World Examples

### Example 1: Basic Insurance Coverage
A patient with 80% insurance coverage gets:
- A test costing 200 RWF
- Medicine costing 50 RWF
- Total bill = 250 RWF

How it's split:
- Insurance pays: 80% of 250 = 200 RWF
- Patient pays: 20% of 250 = 50 RWF

### Example 2: Restricted Medicine
Same patient (80% coverage) gets:
- Medicine that costs 50 RWF
- But this medicine is on the insurance's restricted list

How it's split:
- Insurance pays: 0 RWF (doesn't cover restricted items)
- Patient pays: 50 RWF (must pay full amount)

## Understanding the Code Flow

### Main Files and Their Relationships

```
[Frontend]
  ↓
src/resources/assets/js/
  session.controller.js  (Creates/updates sessions)
           ↓
[Backend API Routes]
  ↓
src/controllers/
  patient.session.controller.js  (Main entry point)
           ↓
  |–––––––––––––––––|–––––––––––––––|
  ↓                 ↓               ↓
calculate.payments   inventory    payment.status
controller.js      service.js    service.js
(Splits costs)    (Gets prices)  (Updates status)
```

### How Functions Work Together

1. **Starting a New Session**:
   ```
   session.controller.js (Frontend)
   → Creates session with items
   → Sends to backend API
   → patient.session.controller.js handles request
   → Calls calculatePayments() for billing
   → Creates payment record in database
   ```

2. **Adding Items to Session**:
   ```
   session.controller.js (Frontend)
   → Updates session items
   → patient.session.controller.js
   → calculateSessionTotals()
     → Fetches current prices
     → Calls calculatePayments()
     → Updates payment record
   ```

3. **Processing Payments**:
   ```
   payment.controller.js (Frontend)
   → Sends payment info
   → processPayment() in backend
   → Updates payment status
   → Triggers any notifications
   ```

### Key Integration Points

1. **Price Updates**:
   - Inventory service → calculateSessionTotals → payment records
   - Any price changes automatically flow through this path

2. **Insurance Changes**:
   - Insurance service → calculatePayments → payment records
   - Coverage updates affect all future calculations

3. **Status Updates**:
   - Payment processing → payment status → session status
   - Status changes can trigger notifications or reports

### Where to Make Changes

This flow helps you understand where to add new features:

1. **New Item Types**:
   - Add to frontend session controller
   - Update backend session controller
   - Modify calculatePayments for new type
   - Update database schemas

2. **Price Rules**:
   - Simple rules: Add in calculatePayments
   - Complex rules: Add in calculateSessionTotals
   - UI rules: Add in frontend session controller

3. **Payment Processing**:
   - Payment validation: Add in processPayment
   - Status logic: Add in payment status service
   - Notifications: Add in payment controller

## Quick Guide: Where to Add New Features

1. For quick changes to bill totals:
   - Add them in `calculateSessionTotals`
   - Good for discounts, special offers, seasonal adjustments

2. For changes to how costs are split:
   - Add them in `calculatePayments`
   - Good for new insurance rules or coverage types

3. For changes to payment processing:
   - Add them in payment approval functions
   - Good for adding receipts, notifications, or audit logs

## Testing Your Changes

Before deploying changes, check these scenarios:

✅ Patient with no insurance
- Should pay 100% of all costs

✅ Patient with partial insurance
- Costs should split correctly by percentage

✅ Patient with full coverage
- Should pay nothing (unless items are restricted)

✅ Restricted items
- Patient should pay 100%

✅ Items marked as "served out"
- Should not appear in bill

✅ Changing prices
- Bills should update with new prices

✅ Payment status
- Should update correctly when payments are made

---

Revision date: 2025-11-02
