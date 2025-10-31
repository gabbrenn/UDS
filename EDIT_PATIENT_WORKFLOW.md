## Receptionist: Search → Patient Details Modal → Edit Patient (including Gender & DOB)

This document explains, in depth, how a receptionist searches for a patient, how the patient details modal is rendered, and how patient information is edited end to end. It also details how gender and date of birth (DOB) are or can be edited, with precise file paths and API endpoints.

### At a glance

- Entry UI: `src/resources/assets/js/components/searchpatient.js`
- Profile modal: `src/utils/user.profile.controller.js` (functions `addUprofile`, `uprofileStf`, class `Profile`)
- Generic edit endpoint (covers gender/DOB): `POST /edit-patient-profile/:patient?` → `src/controllers/patients.controller.js::editPatientProfile`
- Specific edit endpoints: Name, Occupation, Location, NID, etc. in `patients.controller.js` and wired in `src/routes/index.route.js`


## 1) Searching for a patient

File: `src/resources/assets/js/components/searchpatient.js`

The receptionist navigates to the “Search patient” page (menu item). The page component builds a search form and optional fingerprint search button.

- Text search form
	- Input accepts NID, phone number, or email.
	- On submit: performs `request('patient/${value}', postschema)` and, on success, calls `uprofileStf(response, users)` to open the patient details modal.
	- If the URL already has an ID like `/receptionist/search-patient/{patientId}`, the component auto-loads that patient on mount.

- Live suggestions (typeahead)
	- As the user types (≥ 5 chars), it emits a socket event `searchForRecs` with `entity: 'patients'` and `datatofetch: 'Full_name,dob,id,gender'`.
	- Incoming results (`socket.on('RecsRes', ...)`) display a pick list showing name plus `gender, age`. Clicking a suggestion sets the input to the selected patient ID and triggers the form submit.

- Fingerprint search
	- Clicking the fingerprint icon triggers `showFingerprintDiv('search')` to capture fingerprint.
	- On success it calls `request('patient/', postschema)` with `{ fp_data, type: 'fp' }` and, if found, opens the modal via `uprofileStf(...)`.


## 2) Patient details modal (profile)

Primary builder: `src/utils/user.profile.controller.js::addUprofile(data)`

After search success, `uprofileStf()` calls `addUprofile()` which constructs a full-screen modal overlay with patient information and actions. Key sections and controls:

- About
	- Full Name: shows name and an edit button (`#edit-name`)
	- Gender: displayed as text (no default edit control in current UI)
	- DOB: displayed as text (formatted via `extractTime(dob, 'date')`; no default edit control)
	- NID: shows value, copy icon, and an edit button (`#edit-nid`) that opens a correction popup
	- Role (Title): shows role and a `#switch-role` control to toggle patient ↔ householder
	- Occupation: visible only when role = householder, with edit button (`#edit-occupation`)

- Location
	- Shows province, district, sector, cell, village
	- Edit button (`#edit-location`) opens a form with typeahead for administrative levels

- Contacts
	- Phone: shows value and edit button (`#edit-phone`)
	- Email: shows value and edit button (`#edit-email`)

- Health Information
	- Known Allergies: a “View Allergies” button opens a read-only view (there is a backend API to update allergies)

- Insurances
	- Cards for existing assurances; remove buttons per insurance
	- “Add Insurance” opens a chip-based selector and posts additions

- Family
	- If householder: “Add Beneficiary” and a list of beneficiaries (click to navigate; trash to remove)
	- If not householder and no existing householder: “Add Parent” to attach to a householder

- Actions
	- Add fingerprint button to register a new fingerprint
	- Load medical history list (with lazy load button)
	- “Notify consultant” (select target and priority) and “Create session” actions

Important: the event handlers for these controls are attached right after the modal is rendered in the same file (`user.profile.controller.js`), near where the DOM nodes are queried (e.g., `const editName = a.querySelector('#edit-name'); ...`).


## 3) Editing patient information

Edits are performed via specific endpoints or a generic edit endpoint, wired in `src/routes/index.route.js` and implemented in `src/controllers/patients.controller.js`.

### 3.1 Name
- UI: `#edit-name` prompts for a new full name.
- API: `PUT /edit-pati-name/:id` → `editPatientProfileName`
- Frontend: updates `.full-name-hol` on success.

### 3.2 Occupation (householder only)
- UI: `#edit-occupation` prompts for occupation.
- API: `PUT /edit-pati-occupation/:id` → `editPatientOccupation` (stores under `patients.extra.occupation`)
- Frontend: updates `.occupation-hol` on success.

### 3.3 Location
- UI: `#edit-location` opens a modal with inputs for province→village. Inputs are “typeahead with IDs” so values post as IDs.
- API: `PUT /edit-patient-location/:id` → `editPatientLocation`
- Frontend: updates the location spans (`[data-hold=province|district|...]`).

### 3.4 NID (with reason)
- UI: `#edit-nid` opens `showNidCorrectionPopup` with current NID, new NID, and a mandatory reason.
- API: `PUT /update-patient-nid/:id` → `editPatientNID`
- Frontend: updates NID text and the associated copy button’s `data-id`.

### 3.5 Phone and Email
- UI: `#edit-phone`, `#edit-email` call `Profile.editPhone()` / `Profile.editEmail()` which open a small modal via `showContactEditModal`.
- API: `POST /edit-patient-profile/:patient` → `editPatientProfile` with `{ type: 'phone'|'email', value }`
- Backend validation: uniqueness checks for email/phone; receptionist is allowed without password (password is required only if the editor is the patient/householder themselves).
- Frontend: updates neighboring spans in the profile.

### 3.6 Gender (how to edit)

Current UI status: Gender is displayed but there’s no default edit control in the modal. However, the backend supports updating gender via the generic edit endpoint.

- Supported by backend
	- API: `POST /edit-patient-profile/:patient` → `editPatientProfile`
	- Payload: `{ type: 'gender', value: 'male'|'female' }` (consistent with stored values used across analytics and UI)
	- Authorization: Receptionist role can perform this without providing the patient’s password (password check is enforced only for `patient` or `householder` editing themselves).
	- Logging: All edits are recorded via `actionLogger` with action `EDIT_PROFILE`.

- Minimal UI addition (recommended)
	- Add a small edit button next to the gender line (e.g., `#edit-gender`).
	- On click, show a select with allowed values (male, female). Submit to the endpoint above. On success, update the gender text node in the modal.

### 3.7 DOB (how to edit)

Current UI status: DOB is displayed using `extractTime(dob, 'date')`, but there’s no default edit control. The backend supports updating DOB via the generic edit endpoint.

- Supported by backend
	- API: `POST /edit-patient-profile/:patient` → `editPatientProfile`
	- Payload: `{ type: 'dob', value: 'YYYY-MM-DD' }` (store a proper date that matches how DOB is queried elsewhere, e.g., age calculations using `calcTime`)
	- Authorization: Receptionist can perform this without the patient’s password.
	- Logging: `actionLogger` with `EDIT_PROFILE`.

- Minimal UI addition (recommended)
	- Add a small edit button next to the DOB line (e.g., `#edit-dob`).
	- On click, open a modal with an `<input type="date">`. Validate that the new date is sensible (not in the future). Submit as above, then re-render or update the DOB span (use the same formatting as the display code).

Notes:
- No backend uniqueness check applies to gender/DOB (unlike email/phone).
- Ensure the date format aligns with DB and display utilities (`extractTime`, `calcTime`).


## 4) End-to-end flow summary

1) Receptionist opens “Search patient” page.
2) Enters identifier (or uses fingerprint) → frontend calls `GET patient/{id}` or `POST patient/` with `fp_data`.
3) On success, `uprofileStf()` opens the patient profile modal built by `addUprofile()`.
4) Receptionist edits fields as needed:
	 - Name → `PUT /edit-pati-name/:id`
	 - Occupation → `PUT /edit-pati-occupation/:id`
	 - Location → `PUT /edit-patient-location/:id`
	 - NID → `PUT /update-patient-nid/:id` (with reason)
	 - Phone/Email → `POST /edit-patient-profile/:id` with `type` and `value`
	 - Gender/DOB (if enabled in UI) → `POST /edit-patient-profile/:id` with `type: 'gender'|'dob'`
5) On success, UI updates corresponding nodes; most handlers already mutate DOM (see `user.profile.controller.js`).


## 5) Related files and key symbols

- `src/resources/assets/js/components/searchpatient.js`
	- `SearchPatientPage()` – builds page, search form, fingerprint and suggestions
	- Emits `searchForRecs` to socket; renders suggestions; opens profile via `uprofileStf`

- `src/utils/user.profile.controller.js`
	- `addUprofile(data)` – constructs modal HTML (About, Location, Contacts, Health, Actions)
	- `uprofileStf(r, users)` – opens modal and wires global action buttons (notify, create session)
	- Event handlers: edit name, occupation, location, NID, phone, email; family and insurance management; medical history loader
	- `Profile` class: `editLocation`, `addBeneficiary`, `addParent`, `editEmail`, `editPhone`

- `src/controllers/patients.controller.js`
	- `editPatientProfile` – generic editor for arbitrary column via `{ type, value }` (use for gender, dob, email, phone, username, etc.)
	- `editPatientProfileName`, `editPatientOccupation`, `editPatientLocation`, `editPatientNID`, `updatePatientAllergies`
	- Uses `actionLogger` for audit trail; enforces email/phone/username uniqueness; password requirement applies only when the editor is the patient/householder.

- `src/routes/index.route.js`
	- Routes for all edits, including `POST /edit-patient-profile/:patient?` used by the generic edit pattern.


## 6) API reference (receptionist-relevant)

- `GET /patient/{id}` – fetch patient by ID/NID/phone/email
- `POST /patient/` – identify by fingerprint: `{ fp_data, type: 'fp' }`
- `PUT /edit-pati-name/:id` – update full name
- `PUT /edit-pati-occupation/:id` – update occupation (in `extra`)
- `PUT /edit-patient-location/:id` – update location (IDs for administrative levels)
- `PUT /update-patient-nid/:id` – update NID with reason
- `POST /edit-patient-profile/:patient?` – generic editing: `{ type, value }` (use for gender, dob, email, phone, username)


## 7) Enabling gender & DOB editing in the UI (safe pattern)

The backend already supports these fields via the generic endpoint; to expose them safely to receptionists:

1) Add buttons next to the fields in `addUprofile()` markup, e.g.,
	 - Gender line → append `<span class="ml-10p btn btn-sm btn-primary" id="edit-gender">...`.
	 - DOB line → append `<span class="ml-10p btn btn-sm btn-primary" id="edit-dob">...`.

2) Attach handlers (near existing `#edit-phone`/`#edit-email` wiring):
	 - For gender: open a small modal with select options: male, female. On submit, call `request('edit-patient-profile/{id}', { type: 'gender', value })`, then update the text node.
	 - For DOB: open a small modal with `<input type="date">`. Validate not future, call `request('edit-patient-profile/{id}', { type: 'dob', value })`, and update the displayed date using the same formatting helper (`extractTime`).

3) Audit and roles: Edits are logged (`actionLogger`). Receptionists don’t need the patient’s password for these updates (password check applies to patient/householder self-edits).

4) Downstream checks: Age is computed with `calcTime(dob)` across the app (suggestion lists, history, analytics). Ensure DOB format stays compatible (`YYYY-MM-DD`).


## 8) Edge cases and validations

- Not found: If `GET /patient/{id}` or fingerprint lookup fails, a toast with the error message is shown and the modal isn’t opened.
- Email/Phone duplicates: Backend returns a uniqueness error for these fields in `editPatientProfile`.
- NID update: Requires a non-trivial reason; the UI enforces minimum length and non-equality with the old NID.
- Location consistency: The editor clears dependent fields when a higher-level geo changes; IDs are posted to the server.
- Permissions: Route guards require authenticated roles; specific routes limit which roles can edit which fields (receptionist is permitted on the routes listed above).


## 9) Quality notes

- Display helpers: DOB is normalized with `extractTime(dob, 'date')`; ages and timelines use `calcTime`.
- Real-time: Some updates (like self-edit by patients) refresh tokens via socket. For receptionist edits to other patients, no token change is needed.
- Audit: All profile edits are logged with patient ID, type, and value.


— End —

