# Issue 1 — Create insurance_rules database table
### Description
Add a new table to store insurance coverage caps per hospital, per insurance, per entity.

### Tasks
- [ ] Create migration for `insurance_rules` table
- [ ] Fields: id, hospital_id, insurance_id, entity_type, entity_id, coverage_amount, timestamps
- [ ] Add model or query helper if applicable
- [ ] Add DB index: (hospital_id, insurance_id, entity_type, entity_id)

### Acceptance Criteria
- Table exists in DB with correct schema
- Able to insert/read/update/delete a rule

### Labels
backend, db, enhancement

# Issue 2 — Add helper function getInsuranceRule()
### Description
Create helper inside calculate.payments.controller.js to fetch the coverage rule.

### Tasks
- [ ] Implement getInsuranceRule(hospital_id, insurance_id, entity_type, entity_id)
- [ ] Return { coverage_amount } or null
- [ ] Add basic error handling + fallback

### Acceptance Criteria
- Function returns coverage based on matching DB row
- Logs or silently ignores missing rules (no crash)

### Labels
backend, logic

# Issue 3 — Apply rule logic inside calculatePayments()
### Description
Modify existing payment calculation so that insurance payments are capped by rule.

### Tasks
- [ ] Before calculation, lookup rule via getInsuranceRule()
- [ ] Compute insurance_base and private/top_up amount
- [ ] Replace old logic that assumed full or percentage-based tariff
- [ ] Add private_amount into returned object

### Acceptance Criteria
- Insurance does not exceed rule × quantity
- Extra amount is assigned to patient as private/top-up
- No change for entities without insurance rules

### Labels
backend, payments

# Issue 4 — Update all addSession* controllers to store private_amount
### Description
Store private_amount alongside existing assurance_amount and pati_p_amount.

### Tasks
- [ ] Modify addSession(), addSessionTests(), services, operations, equipment, medicines
- [ ] Make sure edit/delete flows also recalc private_amount
- [ ] Update models if needed (payments.extra field)

### Acceptance Criteria
- Private/top-up amount is saved and appears in session payments
- No breaking change for existing payments

### Labels
backend, controller

# Issue 5 — Update DOF financial API responses
### Description
Expose 3-way split in finance API responses: insurance, patient, private_amount.

### Tasks
- [ ] Modify API responses used in DoF and cashier views
- [ ] Add missing field in JSON output
- [ ] Ensure sorting/filtering still works

### Acceptance Criteria
- API returns 3 fields: assurance_amount, pati_p_amount, private_amount
- No changes required in frontend until UI update

### Labels
backend, reporting

✅ Repository: Frontend (DoF Portal / Cashier UI)
# Issue 6 — Create Insurance Rule Management UI
### Description
Add new page in DoF portal to manage insurance rules per entity type.

### Tasks
- [ ] Add tab layout: Services, Tests, Medicines, Operations, Equipment
- [ ] List insurances for this hospital
- [ ] Show rule table: Entity | Price | Coverage | Difference | Actions
- [ ] Action buttons: Add Rule, Edit Rule, Delete Rule
- [ ] Use existing showRecs() for entity picker
- [ ] Use existing modal framework for popup forms

### Acceptance Criteria
- DoF can add/edit/remove rules
- Rules correctly sync with backend via API
- UI follows current DoF styling

### Labels
frontend, feature, UI

# Issue 7 — Update cashier + DoF billing screens to display 3-way split
### Description
Show new private/top-up amount beside insurance and normal patient amount.

### Tasks
- [ ] Update payment breakdown tables
- [ ] Add new column or badge for private amount
- [ ] Add totals footer reflecting 3 values
- [ ] Color-code or style private amount distinctly

### Acceptance Criteria
- Users can clearly see three values per bill
- No layout break on smaller screens

### Labels
frontend, UI, payments

# Issue 8 — Permissions: restrict UI to DoF & Finance roles
### Description
Ensure only users with finance permissions can manage insurance rules.

### Tasks
- [ ] Hook into existing auth/role system
- [ ] Hide menu item if user lacks permission
- [ ] Prevent rule endpoints from loading if unauthorized

### Acceptance Criteria
- Non-finance staff cannot view or edit rules
- Permission denied behavior is consistent with rest of system

### Labels
frontend, security

✅ Shared / DevOps
# Issue 9 — Write migration + deployment notes
### Tasks
- [ ] Document DB changes
- [ ] Add instructions for applying migration
- [ ] Add rollback notes
- [ ] Add sample seed data for testing

### Labels
docs, devops

# Issue 10 — Regression testing & edge cases
### Description
Verify billing still works for all existing insurance types with no rule.

### Tests to cover
- [ ] Sessions with no insurance
- [ ] Insurance with percentage model (no rule)
- [ ] Insurance with fixed rule lower than tariff
- [ ] Rule higher than tariff (cap ignored)
- [ ] Multiple entities, mixed coverage rules

### Labels
testing, QA
