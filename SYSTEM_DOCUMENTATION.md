# UDS (Ubuzima Digital System) - Complete System Documentation

## Table of Contents
1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Technology Stack](#technology-stack)
4. [Project Structure](#project-structure)
5. [Backend Architecture](#backend-architecture)
6. [Frontend Architecture](#frontend-architecture)
7. [Authentication & Authorization](#authentication--authorization)
8. [Database Schema](#database-schema)
9. [Key Features & Workflows](#key-features--workflows)
10. [API Endpoints](#api-endpoints)
11. [AI Integration (NEXUN)](#ai-integration-nexun)
12. [Real-time Communication](#real-time-communication)
13. [External Integrations](#external-integrations)
14. [Development Guidelines](#development-guidelines)
15. [Adding New Features](#adding-new-features)
16. [Testing & Debugging](#testing--debugging)
17. [Security Considerations](#security-considerations)
18. [Deployment](#deployment)

---

## Overview

**UDS (Ubuzima Digital System)** is a comprehensive healthcare management platform that digitizes hospital operations, patient care, and medical workflows. The system integrates **NEXUN**, an AI-powered clinical decision support system that assists healthcare providers with diagnoses, test recommendations, and treatment planning.

### Core Purpose
- Digitize healthcare processes in hospitals
- Manage patient records, appointments, and medical sessions
- Provide AI-assisted medical decision support
- Enable multi-role access (doctors, nurses, pharmacists, lab scientists, cashiers, patients)
- Generate reports and analytics for healthcare management

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                          │
│  (Web Browsers - Chrome, Firefox, Safari, Edge)             │
│  - Patient Portal                                            │
│  - Healthcare Provider Interface                             │
│  - Admin Dashboard                                           │
└─────────────────────┬───────────────────────────────────────┘
                      │ HTTPS Requests
                      │ WebSocket (Socket.IO)
┌─────────────────────▼───────────────────────────────────────┐
│                   APPLICATION LAYER                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Express.js Server (Node.js)                         │   │
│  │  - REST API Endpoints                                │   │
│  │  - WebSocket Server                                  │   │
│  │  - Authentication & Authorization                    │   │
│  │  - Business Logic Controllers                        │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────┬───────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
┌───────▼────┐  ┌────▼─────┐  ┌───▼──────────┐
│  MySQL DB  │  │  AWS S3  │  │ External APIs│
│  (Primary) │  │ (Storage)│  │  - OpenAI    │
│            │  │          │  │  - Azure AI  │
│            │  │          │  │  - Twilio    │
│            │  │          │  │  - Africa's  │
└────────────┘  └──────────┘  │    Talking   │
                               └──────────────┘
```

### Request Flow

```
1. User Action (Frontend)
   ↓
2. HTTP Request / WebSocket Event
   ↓
3. Route Handler (index.route.js)
   ↓
4. Middleware Chain:
   - CORS validation
   - Authentication (token.verifier.controller.js)
   - Authorization (role/permission check)
   - Input validation (body.schema.middleware.js)
   - Action logging
   ↓
5. Controller (Business Logic)
   ↓
6. Database Query (query.controller.js → MySQL)
   ↓
7. Response Generation
   ↓
8. Event Emission (if needed - notifications, logs)
   ↓
9. Response to Client (JSON)
```

---

## Technology Stack

### Backend
- **Runtime**: Node.js (v14+)
- **Framework**: Express.js v4.18.2
- **Language**: JavaScript (ES6+ with ESM modules)
- **Database**: MySQL (via mysql2/promise)
- **Real-time**: Socket.IO v4.6.0
- **Authentication**: JWT (jsonwebtoken)
- **Password Hashing**: bcryptjs
- **Environment Variables**: dotenv

### Frontend
- **Languages**: HTML5, CSS3, JavaScript (ES6+)
- **UI Framework**: Custom (Bootstrap-based)
- **AJAX**: Fetch API / XMLHttpRequest
- **Real-time**: Socket.IO Client
- **Charts**: Chart.js (likely)
- **Date Handling**: Luxon

### External Services
- **Cloud Storage**: AWS S3
- **AI Services**: 
  - OpenAI GPT (NEXUN AI assistant)
  - Azure Cognitive Services (OCR, Image Analysis)
  - Hugging Face (Medical models)
- **SMS**: Twilio, Africa's Talking
- **Email**: Zoho Mail
- **Medical Coding**: WHO ICD-11 API

### DevOps
- **Process Manager**: Nodemon (development)
- **Testing**: Mocha, Chai, Should
- **Code Coverage**: NYC
- **Build Tools**: Babel (ES6+ transpilation)

---

## Project Structure

```
UDS-ubuzima-digital-system/
│
├── handler.js                  # Server initialization, port binding
├── app.js                      # Express app config, DB connection, middleware setup
├── package.json                # Dependencies and scripts
├── .env                        # Environment variables (CREATE THIS)
│
├── src/
│   ├── controllers/            # Business logic layer
│   │   ├── login.controller.js
│   │   ├── signup.controller.js
│   │   ├── patients.controller.js
│   │   ├── employee.controller.js
│   │   ├── patient.session.controller.js
│   │   ├── nexun.controller.js        # AI integration
│   │   ├── 2FA.*.controller.js        # Two-factor authentication
│   │   ├── hospital.controller.js
│   │   ├── appointments.controller.js
│   │   ├── medicine.controller.js
│   │   ├── tests.controller.js
│   │   ├── uploads.controller.js      # File handling
│   │   └── ... (50+ controllers)
│   │
│   ├── routes/
│   │   └── index.route.js      # All API endpoint definitions
│   │
│   ├── middlewares/            # Request interceptors
│   │   ├── users.authoriser.middleware.js    # Role-based access
│   │   ├── roles.authorizer.middleware.js
│   │   ├── action.logger.js                  # Activity logging
│   │   ├── body.schema.middleware.js         # Input validation
│   │   ├── api.key.authorizer.js
│   │   └── ... (17 middlewares)
│   │
│   ├── services/               # Reusable business services
│   │   └── notification.service.js    # Email/SMS sending
│   │
│   ├── events/                 # Event-driven architecture
│   │   ├── eventsEmitter.js           # Central event emitter
│   │   └── notifications.listener.js   # Event handlers
│   │
│   ├── schemas/                # Data validation schemas
│   │
│   ├── pages/                  # HTML frontend files
│   │   ├── auth-login.html
│   │   ├── dashboard.html
│   │   ├── admin-*.html        # Admin dashboards
│   │   ├── hcp-index.html      # Healthcare provider
│   │   ├── patient-index.html
│   │   ├── pharmacist-index.html
│   │   ├── laboratory-index.html
│   │   └── ... (40+ pages)
│   │
│   ├── templates/              # Email/SMS templates
│   │
│   ├── socket.io/              # WebSocket handlers
│   │   ├── connector.socket.io.js
│   │   ├── holder.socket.js
│   │   └── recs.socket.js
│   │
│   ├── Database/               # SQL schema files
│   │   ├── uds.sql
│   │   ├── services.sql
│   │   └── tests.sql
│   │
│   ├── process/                # Background jobs
│   │   └── process.controller.js
│   │
│   ├── resources/              # Static assets
│   │
│   └── utils/                  # Helper functions
│
├── public/                     # Publicly accessible files
│   ├── admin/                  # Admin dashboard assets
│   │   ├── asset/
│   │   │   ├── js/             # JavaScript files
│   │   │   │   ├── auth.js     # Authentication logic
│   │   │   │   ├── api.js      # API wrapper
│   │   │   │   └── ...
│   │   │   ├── css/            # Stylesheets
│   │   │   └── images/
│   │   └── *.html
│   │
│   ├── patient/                # Patient portal assets
│   ├── uploads/                # User-uploaded files
│   └── ...
│
├── certificates/               # SSL/TLS certificates
├── AWS/                        # AWS deployment files
└── fingerprint-sdk/            # Biometric integration
```

---

## Backend Architecture

### Application Entry Point

#### 1. `handler.js` - Server Bootstrap
```javascript
// Loads environment variables FIRST
dotenv.config();

// Creates Express app instance
export let app = express();

// Starts HTTP server on configured port
const port = process.env.PORT || 7000;
export const server = app.listen(port, () => {
    console.log(`connected to port ${port}`)
});
```

#### 2. `app.js` - Application Configuration
```javascript
// Establishes MySQL connection pool
const connection = mysql.createPool({
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    password: process.env.DB_PASS,
    database: process.env.DB_NAME
});

// Registers middleware
app.use(bodyParser.json());
app.use(cors());
app.use(express.static('public'));

// Mounts all routes
app.use(router);

// Imports event listeners (notifications)
import "./src/events/notifications.listener.js";

// Starts background processes
import { weeklyProcess } from './src/process/process.controller.js';
```

### Controllers Layer

Controllers contain business logic and orchestrate data flow between routes and database.

**Example: Login Controller**
```javascript
// src/controllers/login.controller.js

async function login(req, res) {
    try {
        const { username, password } = req.body;
        
        // Query user from database
        const user = await query(
            'SELECT * FROM users WHERE username = ?', 
            [username]
        );
        
        if (!user.length) {
            return res.status(404).json({ 
                message: "User not found" 
            });
        }
        
        // Verify password
        const isValid = await bcrypt.compare(password, user[0].password);
        
        if (!isValid) {
            return res.status(401).json({ 
                message: "Invalid credentials" 
            });
        }
        
        // Generate JWT token
        const token = jwt.sign(
            { 
                id: user[0].id,
                role: user[0].role,
                username: user[0].username 
            },
            process.env.SECRET_KEY,
            { expiresIn: '24h' }
        );
        
        // Send 2FA code
        await send2FACode(user[0].email);
        
        res.status(200).json({
            status: 200,
            message: "Login successful",
            data: { ...user[0], token }
        });
        
    } catch (error) {
        console.error(error);
        res.status(500).json({ message: "Server error" });
    }
}

export default login;
```

### Routes Layer

All routes are defined in `src/routes/index.route.js`. Routes map HTTP endpoints to controllers with middleware chains.

**Route Structure:**
```javascript
router.post(
    '/endpoint',
    middleware1,        // Authentication
    middleware2,        // Authorization
    middleware3,        // Input validation
    controllerFunction  // Business logic
);
```

**Example Routes:**
```javascript
// Authentication
router.post('/login', login);
router.post('/signup', signup);
router.post('/verify-2fa', verification);

// Patient Management (Protected)
router.post('/add-patient', 
    authorizeHc_provider,      // Must be healthcare provider
    addpatient                 // Controller
);

router.get('/get-patients/:hospital', 
    authorizeMultipleRoles,    // Multiple roles allowed
    getHpatients
);

// Session Management
router.post('/add-session', 
    authorizeHc_provider,
    addSession
);

router.post('/close-session',
    authorizeHc_provider,
    closeSession
);

// AI Assistant
router.post('/nexun/generate',
    authorizeHc_provider,
    generateCompletion
);
```

### Middleware Layer

Middlewares intercept requests for cross-cutting concerns.

#### 1. Authentication Middleware
```javascript
// src/middlewares/users.authoriser.middleware.js

export const authorizeAdmin = async (req, res, next) => {
    try {
        // Extract and verify JWT token
        const token = GAVToken(req);
        if (!token.success) {
            return res.status(401).send({ 
                message: "Invalid token" 
            });
        }
        
        // Check user role in database
        const user = await query(
            'SELECT role FROM users WHERE id = ?', 
            [token.token.id]
        );
        
        if (user[0].role !== 'admin') {
            return res.status(403).send({ 
                message: "Forbidden" 
            });
        }
        
        // Attach user to request
        req.user = token.token;
        next();
        
    } catch (error) {
        res.status(500).send({ message: "Server error" });
    }
};
```

#### 2. Action Logger Middleware
```javascript
// src/middlewares/action.logger.js

// Logs every action to database for audit trail
export const logAction = async (req, res, next) => {
    const userId = req.user?.id;
    const action = `${req.method} ${req.path}`;
    const timestamp = new Date();
    
    await query(
        'INSERT INTO action_logs (user_id, action, timestamp) VALUES (?, ?, ?)',
        [userId, action, timestamp]
    );
    
    next();
};
```

### Database Layer

#### Query Controller
```javascript
// src/controllers/query.controller.js

import connection from '../../app.js';

async function query(sql, params) {
    try {
        const [rows] = await connection.execute(sql, params);
        return rows;
    } catch (error) {
        console.error('Database error:', error);
        throw error;
    }
}

export default query;
```

**Usage Pattern:**
```javascript
// SELECT
const users = await query('SELECT * FROM users WHERE role = ?', ['doctor']);

// INSERT
await query(
    'INSERT INTO patients (name, email, phone) VALUES (?, ?, ?)',
    [name, email, phone]
);

// UPDATE
await query(
    'UPDATE sessions SET status = ? WHERE id = ?',
    ['closed', sessionId]
);

// DELETE
await query('DELETE FROM appointments WHERE id = ?', [appointmentId]);
```

---

## Frontend Architecture

### File Organization

```
public/
├── admin/                      # Admin dashboard
│   ├── dashboard.html
│   ├── asset/
│   │   ├── js/
│   │   │   ├── auth.js         # Login/logout logic
│   │   │   ├── api.js          # API wrapper functions
│   │   │   ├── patients.js     # Patient management
│   │   │   ├── sessions.js     # Session management
│   │   │   └── ...
│   │   └── css/
│   └── *.html
│
├── patient/                    # Patient portal
│   ├── index.html
│   ├── my-sessions.html
│   └── asset/
│
└── shared/                     # Shared resources
    ├── plugins/
    └── utils/
```

### Frontend Request Flow

#### 1. Authentication (Login Example)
```javascript
// admin/asset/js/auth.js

async function login() {
    const username = document.getElementById('username').value;
    const password = document.getElementById('password').value;
    
    try {
        const response = await fetch('http://127.0.0.1:7000/login', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ username, password })
        });
        
        const data = await response.json();
        
        if (data.status === 200) {
            // Store token and user data
            localStorage.setItem('token', data.data.token);
            localStorage.setItem('role', data.data.role);
            localStorage.setItem('userData', JSON.stringify(data.data));
            
            // Redirect to dashboard
            window.location.href = '/admin/dashboard.html';
        } else {
            alert(data.message);
        }
        
    } catch (error) {
        console.error('Login error:', error);
        alert('Login failed');
    }
}
```

#### 2. Authenticated API Calls
```javascript
// admin/asset/js/api.js

async function apiCall(endpoint, method = 'GET', body = null) {
    const token = localStorage.getItem('token');
    
    const options = {
        method: method,
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
        }
    };
    
    if (body && method !== 'GET') {
        options.body = JSON.stringify(body);
    }
    
    try {
        const response = await fetch(
            `http://127.0.0.1:7000${endpoint}`, 
            options
        );
        return await response.json();
        
    } catch (error) {
        console.error('API call failed:', error);
        throw error;
    }
}

// Usage
const patients = await apiCall('/get-patients/12345', 'GET');
await apiCall('/add-patient', 'POST', { name, email, phone });
```

#### 3. Real-time Updates (Socket.IO)
```javascript
// Connect to WebSocket server
const socket = io('http://127.0.0.1:7000', {
    auth: {
        token: localStorage.getItem('token')
    }
});

// Listen for notifications
socket.on('notification', (data) => {
    showNotification(data.message);
});

// Listen for session updates
socket.on('session:updated', (session) => {
    updateSessionUI(session);
});

// Emit events
socket.emit('join:room', { roomId: 'hospital-123' });
```

---

## Authentication & Authorization

### Authentication Flow

```
1. User submits credentials (username + password)
   ↓
2. Server validates credentials against database
   ↓
3. If valid, generate JWT token with user data
   ↓
4. Send 2FA code via email/SMS
   ↓
5. User enters 2FA code
   ↓
6. Server verifies 2FA code
   ↓
7. Return token to client
   ↓
8. Client stores token in localStorage
   ↓
9. All subsequent requests include token in Authorization header
```

### Token Structure

```javascript
// Token payload
{
    id: "user_id",
    Full_name: "John Doe",
    role: "hc_provider",
    status: "available",
    email: "john@example.com",
    phone: "+250788123456",
    username: "johndoe",
    iat: 1738579814,           // Issued at
    exp: 1738666214            // Expires at (24 hours later)
}
```

### Token Verification

```javascript
// src/controllers/token.verifier.controller.js

export function GAVToken(req) {
    try {
        // Extract token from Authorization header
        const authHeader = req.headers.authorization;
        const token = authHeader?.split(' ')[1];  // "Bearer <token>"
        
        if (!token) {
            return { success: false, message: 'No token provided' };
        }
        
        // Verify and decode token
        const decoded = jwt.verify(token, process.env.SECRET_KEY);
        
        return { success: true, token: decoded };
        
    } catch (error) {
        return { success: false, message: 'Invalid token' };
    }
}
```

### Role-Based Access Control (RBAC)

**System Roles:**
- `admin` - System administrator
- `super_admin` - Super administrator with all permissions
- `hc_provider` - Healthcare provider (doctor/nurse)
- `pharmacist` - Pharmacy staff
- `laboratory_scientist` - Lab technician
- `cashier` - Billing staff
- `receptionist` - Front desk
- `assurance_manager` - Insurance manager
- `patient` - Patient user
- `system` - System service account

**Role Middleware Examples:**
```javascript
// Single role authorization
export const authorizeAdmin = async (req, res, next) => {
    const token = GAVToken(req);
    const user = await query('SELECT role FROM users WHERE id = ?', [token.token.id]);
    
    if (user[0].role !== 'admin') {
        return res.status(403).send({ message: "Forbidden" });
    }
    next();
};

// Multiple roles authorization
export const authorizeMultipleRoles = async (req, res, next) => {
    const allowedRoles = ['admin', 'hc_provider', 'pharmacist'];
    const token = GAVToken(req);
    const user = await query('SELECT role FROM users WHERE id = ?', [token.token.id]);
    
    if (!allowedRoles.includes(user[0].role)) {
        return res.status(403).send({ message: "Forbidden" });
    }
    next();
};
```

### Capability-Based Authorization

Beyond roles, the system supports fine-grained capabilities:

```javascript
// User can have additional capabilities per hospital
{
    role: "nurse",
    extra: {
        additionalCapabilities: [
            {
                hospitalId: "12345",
                task: "prescribe_medicine",
                subtasks: ["antibiotics", "painkillers"]
            },
            {
                hospitalId: "12345",
                task: "order_tests",
                subtasks: ["*"]  // All test types
            }
        ]
    }
}
```

---

## Database Schema

### Core Tables

#### 1. Users Table
```sql
CREATE TABLE users (
    id VARCHAR(50) PRIMARY KEY,
    Full_name VARCHAR(255),
    username VARCHAR(100) UNIQUE,
    email VARCHAR(255),
    phone VARCHAR(20),
    password VARCHAR(255),  -- bcrypt hashed
    role ENUM('admin', 'hc_provider', 'pharmacist', ...),
    status ENUM('available', 'unavailable', 'busy'),
    hospital VARCHAR(50),   -- Hospital ID
    department VARCHAR(50),
    extra JSON,             -- Additional data (capabilities, preferences)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

#### 2. Patients Table
```sql
CREATE TABLE patients (
    id VARCHAR(50) PRIMARY KEY,
    patientName VARCHAR(255),
    patientID VARCHAR(100) UNIQUE,  -- National ID or hospital ID
    email VARCHAR(255),
    phone VARCHAR(20),
    DOB DATE,
    gender ENUM('M', 'F', 'Other'),
    address TEXT,
    bloodGroup VARCHAR(5),
    allergies JSON,
    defNchannel ENUM('email', 'sms'),  -- Preferred notification channel
    fingerprint BLOB,      -- Biometric data
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 3. Sessions Table (Medical Sessions)
```sql
CREATE TABLE sessions (
    id VARCHAR(50) PRIMARY KEY,
    patient VARCHAR(50),   -- Patient ID
    hospital VARCHAR(50),
    hc_provider VARCHAR(50),  -- Healthcare provider ID
    type ENUM('consultation', 'emergency', 'followup'),
    dtype ENUM('outpatient', 'inpatient'),
    status ENUM('open', 'closed', 'transferred'),
    insurance VARCHAR(50),  -- Insurance/Assurance ID
    
    -- Session data (JSON fields)
    symptoms JSON,         -- Chief complaints
    vitals JSON,           -- Vital signs (BP, temp, pulse, etc.)
    physical_exam JSON,    -- Physical examination findings
    history JSON,          -- Medical history
    diagnosis JSON,        -- Diagnoses (ICD codes)
    tests JSON,            -- Ordered tests
    medicines JSON,        -- Prescriptions
    operations JSON,       -- Procedures/operations
    
    nexun_report TEXT,     -- AI-generated report
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    closed_at TIMESTAMP NULL
);
```

#### 4. Hospitals Table
```sql
CREATE TABLE hospitals (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255),
    type ENUM('hospital', 'clinic', 'health_center'),
    location JSON,         -- Province, district, sector, cell
    phone VARCHAR(20),
    email VARCHAR(255),
    tin_number VARCHAR(50),
    logo VARCHAR(255),     -- Logo file path
    created_at TIMESTAMP
);
```

#### 5. Medicines Table
```sql
CREATE TABLE medicines (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255),
    generic_name VARCHAR(255),
    category VARCHAR(100),
    form ENUM('tablet', 'capsule', 'syrup', 'injection'),
    strength VARCHAR(50),
    manufacturer VARCHAR(255),
    price DECIMAL(10, 2),
    stock INT,
    hospital VARCHAR(50)
);
```

#### 6. Tests Table
```sql
CREATE TABLE tests (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255),
    category VARCHAR(100),
    department VARCHAR(50),  -- Lab, Radiology, etc.
    price DECIMAL(10, 2),
    turnaround_time INT,     -- Hours
    hospital VARCHAR(50)
);
```

#### 7. Appointments Table
```sql
CREATE TABLE appointments (
    id VARCHAR(50) PRIMARY KEY,
    patient VARCHAR(50),
    hc_provider VARCHAR(50),
    hospital VARCHAR(50),
    appointment_date DATETIME,
    type ENUM('consultation', 'followup', 'procedure'),
    status ENUM('scheduled', 'completed', 'cancelled'),
    notes TEXT,
    created_at TIMESTAMP
);
```

---

## Key Features & Workflows

### 1. Patient Registration & Management

**Workflow:**
```
1. Receptionist/HCP accesses patient registration form
2. Enters patient demographics (name, ID, DOB, contact)
3. Optional: Captures fingerprint for biometric ID
4. System generates unique patient ID
5. Patient record saved to database
6. Patient can now be searched and sessions created
```

**API Endpoint:**
```
POST /add-patient
Authorization: Bearer <token>
Role: hc_provider, receptionist

Body:
{
    "patientName": "John Doe",
    "patientID": "1199780012345",  // National ID
    "email": "john@example.com",
    "phone": "+250788123456",
    "DOB": "1997-01-15",
    "gender": "M",
    "bloodGroup": "O+",
    "allergies": ["penicillin"],
    "defNchannel": "sms"
}
```

### 2. Medical Session Workflow

**Workflow:**
```
1. Patient arrives at hospital
2. Receptionist/HCP creates new session
3. HCP opens session and records:
   - Chief complaints (symptoms)
   - Vital signs (BP, temp, pulse, O2 sat)
   - Physical examination
   - Medical history
4. HCP uses NEXUN AI for diagnostic assistance
5. HCP records:
   - Diagnosis (with ICD codes)
   - Test orders
   - Prescriptions
   - Procedures/operations
6. Tests are performed (lab/radiology)
7. Test results uploaded to session
8. HCP reviews results, adjusts treatment
9. Session closed
10. Patient notification sent
11. Billing processed
```

**Key API Endpoints:**
```
POST /add-session              - Create new session
POST /add-session-symptoms     - Record symptoms
POST /add-session-vitals       - Record vital signs
POST /add-session-diagnosis    - Add diagnosis
POST /add-session-tests        - Order tests
POST /add-session-medicine     - Prescribe medication
POST /close-session            - Close session
GET /get-session/:id           - Retrieve session data
```

### 3. NEXUN AI Assistant

**Workflow:**
```
1. HCP opens patient session
2. Clicks "Ask NEXUN" button
3. NEXUN receives:
   - Patient demographics
   - Current symptoms
   - Vital signs
   - Medical history
   - Previous diagnoses
   - Uploaded images/documents
4. NEXUN analyzes data using:
   - OpenAI GPT-4
   - Azure AI (image analysis)
   - Medical knowledge bases
5. NEXUN generates report with:
   - Differential diagnoses (ranked by likelihood)
   - Recommended tests
   - Suggested medications
   - Patient advice
   - Safety alerts (red flags)
6. HCP reviews NEXUN report
7. HCP makes final clinical decisions
8. Report saved to session
```

**API Endpoint:**
```
POST /nexun/generate
Authorization: Bearer <token>
Role: hc_provider

Body:
{
    "sessionId": "session_123",
    "conv": {
        "messages": [
            {
                "role": "user",
                "content": "Patient presents with fever, cough, difficulty breathing"
            }
        ],
        "responseFormat": "raw"
    }
}

Response:
{
    "report": "<article class='nexun-report'>...</article>",
    "signature": "nexun_response_id"
}
```

### 4. Pharmacy Management

**Workflow:**
```
1. Session closed with prescriptions
2. Patient goes to pharmacy
3. Pharmacist views patient's active prescriptions
4. Pharmacist dispenses medicines
5. Inventory updated automatically
6. Prescription marked as "dispensed"
7. Patient billed
```

### 5. Laboratory Management

**Workflow:**
```
1. HCP orders tests in session
2. Lab scientist views pending tests
3. Tests performed
4. Results entered/uploaded
5. HCP receives notification
6. HCP reviews results in session
```

### 6. Billing & Insurance

**Workflow:**
```
1. Session closed
2. System calculates total:
   - Consultation fee
   - Tests performed
   - Medicines dispensed
   - Procedures/operations
3. If patient has insurance:
   - Insurance coverage applied
   - Claim generated
4. Cashier collects patient payment
5. Receipt generated
```

---

## AI Integration (NEXUN)

### NEXUN Architecture

```
Patient Session Data
        ↓
Context Preparation
        ↓
┌───────────────────────┐
│   NEXUN Controller    │
│ (nexun.controller.js) │
└──────────┬────────────┘
           │
    ┌──────┴──────┐
    │             │
    ▼             ▼
┌─────────┐  ┌─────────────┐
│ OpenAI  │  │  Azure AI   │
│ GPT-4   │  │  (OCR,      │
│         │  │   Image)    │
└─────────┘  └─────────────┘
    │             │
    └──────┬──────┘
           │
     HTML Report
           ↓
   Save to Session
```

### NEXUN Capabilities

1. **Clinical Decision Support**
   - Differential diagnosis generation
   - Test recommendations
   - Treatment suggestions

2. **Medical Document Analysis**
   - PDF reports (lab, imaging, discharge summaries)
   - Word documents (.doc, .docx)
   - Prescription extraction

3. **Medical Image Analysis**
   - X-rays, CT scans, MRIs
   - Dermatology images
   - Wound photos
   - ECG strips

4. **Safety Features**
   - Red flag detection (emergency conditions)
   - Drug interaction warnings
   - Contraindication alerts
   - Dosage validation

### NEXUN Prompt Engineering

The system uses a sophisticated prompt structure:

```
System Role: Clinical decision support assistant
Scope: Healthcare only (medical symptoms, diagnoses, tests)
Safety: Mandatory red flag detection
Medication: Dose validation with contraindications
Output: Strict HTML format (no JSON, no scripts)
```

**Output Format:**
```html
<article class="nexun-report" data-triage="self-care|routine|urgent|emergent">
    <h2>Brief</h2>
    <ul><li>Key findings...</li></ul>
    
    <h2>Differential</h2>
    <ol><li><strong>Diagnosis A</strong> — likelihood: high; rationale: ...</li></ol>
    
    <h2>Recommended Workup</h2>
    <ul><li>Test — reason</li></ul>
    
    <h2>Medications (Consider if Safe)</h2>
    <ul><li>Drug — dose; cautions</li></ul>
    
    <h2>Patient Advice</h2>
    <ul><li>Next steps...</li></ul>
    
    <h2>Citations</h2>
    <ol><li>Reference — source</li></ol>
</article>
```

---

## Real-time Communication

### Socket.IO Implementation

**Server Setup:**
```javascript
// src/socket.io/connector.socket.io.js

import { Server } from 'socket.io';
import { server } from '../../handler.js';

const io = new Server(server, {
    cors: {
        origin: process.env.API_SERVER_ORIGIN,
        methods: ["GET", "POST"]
    }
});

// Authentication middleware
io.use((socket, next) => {
    const token = socket.handshake.auth.token;
    
    try {
        const decoded = jwt.verify(token, process.env.SECRET_KEY);
        socket.userId = decoded.id;
        socket.userRole = decoded.role;
        next();
    } catch (error) {
        next(new Error('Authentication failed'));
    }
});

// Connection handler
io.on('connection', (socket) => {
    console.log(`User connected: ${socket.userId}`);
    
    // Join user-specific room
    socket.join(`user:${socket.userId}`);
    
    // Join hospital room if applicable
    if (socket.userHospital) {
        socket.join(`hospital:${socket.userHospital}`);
    }
    
    // Handle disconnection
    socket.on('disconnect', () => {
        console.log(`User disconnected: ${socket.userId}`);
    });
});

export default io;
```

**Use Cases:**
1. **Real-time Notifications**
   - Test results ready
   - Appointment reminders
   - Session updates

2. **Live Updates**
   - New patient registrations
   - Session status changes
   - Inventory alerts

3. **Chat/Messaging**
   - HCP to patient communication
   - Internal staff messaging

---

## External Integrations

### 1. AWS Services

**S3 Storage:**
```javascript
// src/controllers/uploads.controller.js

import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

const s3Client = new S3Client({
    region: process.env.AWS_REGION,
    credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
    }
});

export async function uploadToS3(file, key) {
    const command = new PutObjectCommand({
        Bucket: process.env.AWS_BUCKET_NAME,
        Key: key,
        Body: file.buffer,
        ContentType: file.mimetype
    });
    
    await s3Client.send(command);
    return `https://${process.env.AWS_BUCKET_NAME}.s3.amazonaws.com/${key}`;
}
```

**AWS HealthImaging (DICOM):**
- Medical imaging storage
- DICOM conversion
- Image retrieval

### 2. Azure Cognitive Services

**Form Recognizer (OCR):**
```javascript
import { FormRecognizerClient } from "@azure/ai-form-recognizer";

const client = new FormRecognizerClient(
    process.env.AZURE_ENDPOINT,
    { key: process.env.AZURE_API_KEY }
);

// Extract text from medical documents
const poller = await client.beginRecognizeContent(documentBuffer);
const pages = await poller.pollUntilDone();
```

### 3. WHO ICD-11 API

**Disease Code Lookup:**
```javascript
// src/controllers/token.signer.controller.js

// Authenticate with WHO ICD API
const tokenResponse = await axios.post(
    'https://icdaccessmanagement.who.int/connect/token',
    {
        client_id: process.env.ICD_CLIENT_ID,
        client_secret: process.env.ICD_CLIENT_SECRET,
        grant_type: 'client_credentials',
        scope: 'icdapi_access'
    }
);

// Search diseases
const searchResults = await axios.get(
    'https://id.who.int/icd/release/11/2023-01/mms/search',
    {
        params: { q: 'malaria' },
        headers: { Authorization: `Bearer ${tokenResponse.data.access_token}` }
    }
);
```

### 4. Communication Services

**Twilio (SMS):**
```javascript
// src/controllers/twilio.setup.contoller.js

import twilio from 'twilio';

const client = twilio(
    process.env.TWILIO_SID,
    process.env.TWILIO_TOKEN
);

export async function sendSMS(to, message) {
    await client.messages.create({
        body: message,
        from: process.env.TWILIO_PHONE_NUMBER,
        to: to
    });
}
```

**Zoho Mail (Email):**
```javascript
// src/services/notification.service.js

import nodemailer from 'nodemailer';

const transporter = nodemailer.createTransport({
    host: 'smtp.zoho.com',
    port: 465,
    secure: true,
    auth: {
        user: process.env.ZOHO_MAIL,
        pass: process.env.ZOHO_MAIL_PASSWORD
    }
});

export async function sendEmail(to, subject, html) {
    await transporter.sendMail({
        from: process.env.ZOHO_MAIL,
        to: to,
        subject: subject,
        html: html
    });
}
```

**Africa's Talking (SMS for Africa):**
```javascript
import AfricasTalking from 'africastalking';

const africastalking = AfricasTalking({
    apiKey: process.env.AT_API_KEY,
    username: process.env.AT_USERNAME
});

const sms = africastalking.SMS;

await sms.send({
    to: ['+250788123456'],
    message: 'Your appointment is confirmed'
});
```

---

## Development Guidelines

### Environment Setup

1. **Clone Repository:**
```bash
git clone https://github.com/mogul250/UDS-ubuzima-digital-system.git
cd UDS-ubuzima-digital-system
```

2. **Install Dependencies:**
```bash
npm install
```

3. **Create .env File:**
```bash
# Copy template
cp .env.example .env

# Edit with your credentials
notepad .env
```

4. **Database Setup:**
```bash
# Import SQL schemas
mysql -u your_user -p your_database < src/Database/uds.sql
mysql -u your_user -p your_database < src/Database/services.sql
mysql -u your_user -p your_database < src/Database/tests.sql
```

5. **Start Development Server:**
```bash
npm run dev    # Uses nodemon for auto-restart
```

### Code Style

**ES6+ Modules:**
```javascript
// Use import/export (not require/module.exports)
import express from 'express';
export default function myFunction() { }
```

**Async/Await:**
```javascript
// Prefer async/await over .then()
async function getData() {
    try {
        const result = await query('SELECT * FROM users');
        return result;
    } catch (error) {
        console.error(error);
        throw error;
    }
}
```

**Error Handling:**
```javascript
// Always use try-catch in controllers
export async function myController(req, res) {
    try {
        // Logic here
        res.status(200).json({ success: true, data });
    } catch (error) {
        console.error(error);
        res.status(500).json({ 
            success: false, 
            message: "Internal server error" 
        });
    }
}
```

### Naming Conventions

- **Files**: `kebab-case.controller.js`, `camelCase.service.js`
- **Functions**: `camelCase` (e.g., `addPatient`, `getSessionData`)
- **Variables**: `camelCase` (e.g., `userId`, `patientData`)
- **Constants**: `UPPER_SNAKE_CASE` (e.g., `SECRET_KEY`, `DB_HOST`)
- **Database Tables**: `snake_case` (e.g., `users`, `patient_sessions`)

---

## Adding New Features

### Step-by-Step Guide

#### Example: Adding "Lab Test Results Notification" Feature

**1. Create Controller** (`src/controllers/lab-results.controller.js`)
```javascript
import query from './query.controller.js';
import eventEmitter from '../events/eventsEmitter.js';

export async function uploadLabResults(req, res) {
    try {
        const { sessionId, testId, results, files } = req.body;
        
        // Validate input
        if (!sessionId || !testId || !results) {
            return res.status(400).json({ 
                message: "Missing required fields" 
            });
        }
        
        // Update test results in database
        await query(
            'UPDATE session_tests SET results = ?, files = ?, status = ? WHERE id = ?',
            [JSON.stringify(results), JSON.stringify(files), 'completed', testId]
        );
        
        // Get patient and session info
        const session = await query(
            'SELECT patient, hc_provider FROM sessions WHERE id = ?',
            [sessionId]
        );
        
        const patient = await query(
            'SELECT patientName, email, phone, defNchannel FROM patients WHERE id = ?',
            [session[0].patient]
        );
        
        // Emit event for notification
        eventEmitter.emit('results:ready', {
            patient: patient[0],
            session: sessionId
        });
        
        res.status(200).json({
            success: true,
            message: "Lab results uploaded successfully"
        });
        
    } catch (error) {
        console.error(error);
        res.status(500).json({ 
            success: false, 
            message: "Failed to upload results" 
        });
    }
}
```

**2. Add Route** (`src/routes/index.route.js`)
```javascript
import { uploadLabResults } from '../controllers/lab-results.controller.js';
import { authorizeLaboratory_scientist } from '../middlewares/users.authoriser.middleware.js';

// Add this line with other routes
router.post('/upload-lab-results', 
    authorizeLaboratory_scientist,  // Authorization
    uploadLabResults                // Controller
);
```

**3. Add Event Listener** (`src/events/notifications.listener.js`)
```javascript
// Add this event handler
eventEmitter.on("results:ready", ({ patient, session }) => {
    sendNotification({
        type: patient.defNchannel === "email" 
            ? "MAIL_RESULTS_READY" 
            : "SMS_RESULTS_READY",
        to: patient.defNchannel === "email" 
            ? patient.email 
            : patient.phone,
        templateData: {
            patientName: patient.patientName,
            link: `${process.env.SERVER_LINK}/patient/my-sessions`,
            sessionId: session
        }
    });
});
```

**4. Create Notification Template** (`src/templates/results-ready.html`)
```html
<!DOCTYPE html>
<html>
<head>
    <title>Lab Results Ready</title>
</head>
<body>
    <h1>Hello {{patientName}},</h1>
    <p>Your lab test results are now available.</p>
    <p><a href="{{link}}">View Results</a></p>
</body>
</html>
```

**5. Update Frontend** (`public/admin/asset/js/lab-results.js`)
```javascript
async function uploadResults() {
    const sessionId = document.getElementById('sessionId').value;
    const testId = document.getElementById('testId').value;
    const results = document.getElementById('results').value;
    
    const response = await fetch('http://127.0.0.1:7000/upload-lab-results', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${localStorage.getItem('token')}`
        },
        body: JSON.stringify({ sessionId, testId, results })
    });
    
    const data = await response.json();
    
    if (data.success) {
        alert('Results uploaded successfully!');
        loadPendingTests();  // Refresh list
    } else {
        alert('Upload failed: ' + data.message);
    }
}
```

**6. Test the Feature**
```bash
# Start server
npm run dev

# Test with Postman or curl
curl -X POST http://127.0.0.1:7000/upload-lab-results \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "sessionId": "session_123",
    "testId": "test_456",
    "results": {"hemoglobin": "14.5 g/dL"}
  }'
```

---

## Testing & Debugging

### Debugging Tips

1. **Check Server Logs:**
```javascript
// Add detailed logging in controllers
console.log('Request body:', req.body);
console.log('User token:', req.user);
console.log('Query result:', result);
```

2. **Test Database Queries:**
```javascript
// Test queries in isolation
import query from './src/controllers/query.controller.js';

const result = await query('SELECT * FROM users WHERE id = ?', ['123']);
console.log(result);
```

3. **Monitor Network Requests:**
- Use browser DevTools (Network tab)
- Check request/response headers
- Verify token is being sent

4. **Common Issues:**

| Issue | Solution |
|-------|----------|
| 401 Unauthorized | Check token validity, ensure Authorization header present |
| 403 Forbidden | Verify user role, check middleware authorization |
| 500 Server Error | Check server logs, verify database connection |
| CORS Error | Update CORS origin in app.js |
| Token expired | Re-login, check EXPIRING_TIME in .env |

### Testing API Endpoints

**Using Postman:**
```
1. Create new request
2. Set method (GET, POST, etc.)
3. Set URL: http://127.0.0.1:7000/endpoint
4. Add headers:
   - Content-Type: application/json
   - Authorization: Bearer <your_token>
5. Add body (for POST/PUT)
6. Send request
```

**Using VS Code REST Client:**
Create `api-tests.http`:
```http
### Login
POST http://127.0.0.1:7000/login
Content-Type: application/json

{
    "username": "admin",
    "password": "password123"
}

### Get Patients (requires token from login)
GET http://127.0.0.1:7000/get-patients/hospital_123
Authorization: Bearer <paste_token_here>
```

---

## Security Considerations

### Current Security Measures

1. **Password Hashing:**
   - bcryptjs with salt rounds
   - Never store plain-text passwords

2. **JWT Authentication:**
   - Token expiration (24 hours default)
   - Secret key from environment variable

3. **CORS Protection:**
   - Configured allowed origins
   - Prevents unauthorized cross-origin requests

4. **SQL Injection Prevention:**
   - Parameterized queries
   - Never concatenate user input into SQL

5. **Role-Based Access:**
   - Middleware authorization
   - Route-level permission checks

### Security Recommendations

**Before Deployment:**

1. **Environment Variables:**
   - Never commit .env to git
   - Use different secrets for production
   - Rotate API keys regularly

2. **HTTPS:**
   - Use SSL/TLS certificates
   - Force HTTPS redirect
   - Update SERVER_LINK to https://

3. **Database Security:**
   - Use read-only user for queries
   - Separate write permissions
   - Enable audit logging

4. **Rate Limiting:**
```javascript
import rateLimit from 'express-rate-limit';

const limiter = rateLimit({
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 100  // limit each IP to 100 requests per windowMs
});

app.use('/api/', limiter);
```

5. **Input Validation:**
```javascript
import { body, validationResult } from 'express-validator';

router.post('/add-patient',
    body('email').isEmail(),
    body('phone').isMobilePhone(),
    body('patientName').trim().isLength({ min: 2 }),
    (req, res, next) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ errors: errors.array() });
        }
        next();
    },
    addpatient
);
```

6. **Sensitive Data:**
   - Mask passwords in logs
   - Encrypt sensitive fields in database
   - Use HTTPS for all API calls

---

## Deployment

### Production Checklist

- [ ] Create production .env with secure credentials
- [ ] Change SECRET_KEY to strong random value
- [ ] Update SERVER_LINK to production domain
- [ ] Set NODE_ENV=production
- [ ] Enable HTTPS/SSL
- [ ] Configure firewall rules
- [ ] Set up database backups
- [ ] Configure logging (winston, morgan)
- [ ] Set up monitoring (PM2, New Relic)
- [ ] Test all critical workflows
- [ ] Document admin credentials securely

### Deployment Options

**1. Traditional Server (VPS):**
```bash
# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Clone repository
git clone https://github.com/mogul250/UDS-ubuzima-digital-system.git
cd UDS-ubuzima-digital-system

# Install dependencies
npm install --production

# Set up PM2 process manager
npm install -g pm2
pm2 start handler.js --name uds-server
pm2 startup
pm2 save

# Configure Nginx reverse proxy
sudo nano /etc/nginx/sites-available/uds
```

**2. AWS Deployment:**
- EC2 for application server
- RDS for MySQL database
- S3 for file storage (already configured)
- CloudFront for CDN
- Route 53 for DNS

**3. Docker:**
```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 7000
CMD ["node", "handler.js"]
```

---

## Quick Reference

### Essential Commands

```bash
# Start development server
npm run dev

# Start production server
npm start

# Run tests
npm test

# Check dependencies
npm list

# Update dependencies
npm update
```

### Key Files to Know

| File | Purpose |
|------|---------|
| `handler.js` | Server entry point |
| `app.js` | Express configuration |
| `src/routes/index.route.js` | All API routes |
| `src/controllers/login.controller.js` | Authentication logic |
| `src/controllers/nexun.controller.js` | AI assistant |
| `src/middlewares/users.authoriser.middleware.js` | Authorization |
| `src/events/notifications.listener.js` | Event handlers |
| `src/services/notification.service.js` | Email/SMS service |

### Environment Variables Quick Reference

```env
# Server
PORT=7000
NODE_ENV=development
SERVER_LINK=http://127.0.0.1:7000

# Database
DB_HOST=mysql.freehostia.com
DB_USER=ingtec5_uds
DB_PASS=Hh@0790861884
DB_NAME=ingtec5_uds

# Authentication
SECRET_KEY=84JFTK00mE
EXPIRING_TIME=10

# AI Services
OPENAI_KEY=sk-proj-...

# AWS
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=l46e...
AWS_REGION=us-east-1
AWS_BUCKET_NAME=udsbucket250

# Communication
ZOHO_MAIL=info@ubuzima.digital
ZOHO_MAIL_PASSWORD=1rLeLk0akkN7
TWILIO_SID=ACd8...
TWILIO_TOKEN=4d11...
TWILIO_PHONE_NUMBER=+17742249159
```

### Common Tasks

**Add a new user role:**
1. Update database ENUM in users table
2. Create authorization middleware in `users.authoriser.middleware.js`
3. Add role-specific routes in `index.route.js`
4. Create frontend pages for role

**Add a new API endpoint:**
1. Create controller function
2. Add route in `index.route.js`
3. Add middleware (auth, validation)
4. Test with Postman
5. Update frontend to call endpoint

**Modify database schema:**
1. Update SQL file in `src/Database/`
2. Run migration script
3. Update relevant controllers
4. Test thoroughly

---

## Developer Tips & Best Practices

### 1. Before Making Changes
- **Understand the flow:** Trace the request from route → middleware → controller → database
- **Check dependencies:** See what other features might be affected
- **Read existing code:** Follow the patterns already established

### 2. When Adding Features
- **Start small:** Build incrementally, test each piece
- **Use events:** For side effects (notifications, logging), use EventEmitter
- **Validate input:** Always validate and sanitize user input
- **Handle errors:** Comprehensive try-catch blocks
- **Log appropriately:** Help future debugging

### 3. Code Organization
- **One function, one purpose:** Keep functions focused
- **DRY principle:** Don't repeat yourself, extract common logic
- **Consistent naming:** Follow project conventions
- **Comment complex logic:** Explain why, not just what

### 4. Database Queries
- **Use parameters:** Always use `?` placeholders
- **Optimize queries:** Select only needed columns
- **Handle errors:** Database can fail, plan for it
- **Transaction support:** For multi-step operations

### 5. Testing Strategy
- **Test locally first:** Before pushing to production
- **Test edge cases:** Empty inputs, invalid data, extreme values
- **Test permissions:** Verify authorization works correctly
- **Test notifications:** Ensure events fire correctly

---

## Troubleshooting Guide

### Server Won't Start

**Issue:** `Error: listen EADDRINUSE: address already in use :::7000`
**Solution:**
```powershell
# Find process using port 7000
netstat -ano | findstr :7000

# Kill the process
taskkill /PID <process_id> /F

# Or change PORT in .env
```

### Database Connection Failed

**Issue:** `Error connecting to database`
**Solution:**
- Check .env database credentials
- Verify MySQL server is running
- Test connection manually:
```bash
mysql -h mysql.freehostia.com -u ingtec5_uds -p ingtec5_uds
```

### Token Errors

**Issue:** `Invalid token` or `Token expired`
**Solution:**
- Re-login to get fresh token
- Check SECRET_KEY matches in .env
- Verify token is sent in Authorization header

### CORS Errors

**Issue:** `Access-Control-Allow-Origin` error
**Solution:**
```javascript
// In app.js, update CORS config
app.use(cors({
    origin: 'http://your-frontend-domain.com',
    credentials: true
}));
```

---

## Next Steps for You

Now that you understand the system:

### Immediate Actions
1. ✅ Create `.env` file with provided credentials
2. ✅ Install dependencies: `npm install`
3. ✅ Test database connection
4. ✅ Start development server: `npm run dev`
5. ✅ Login and explore the UI

### Learning Path
1. **Week 1:** Understand authentication flow
2. **Week 2:** Explore patient and session management
3. **Week 3:** Study NEXUN AI integration
4. **Week 4:** Practice adding small features

### Resources
- **Project README:** `readme.md`
- **TODO List:** `TODO.md` (check pending tasks)
- **Database Schema:** `src/Database/uds.sql`
- **API Routes:** `src/routes/index.route.js`

### When You Need Help
1. Check this documentation first
2. Read the specific controller code
3. Check logs in console
4. Test in isolation (Postman)
5. Ask specific questions with error messages

---

## Conclusion

UDS is a well-structured, feature-rich healthcare management system. The architecture follows clear separation of concerns with controllers, routes, middlewares, and services.

**Key Strengths:**
- Modular architecture (easy to extend)
- Role-based access control
- Event-driven notifications
- AI integration for clinical support
- Comprehensive medical workflows

**Areas to Understand Deeply:**
- Authentication & authorization flow
- Patient session lifecycle
- NEXUN AI assistant usage
- Event emitter pattern
- Database schema relationships

**Remember:**
- Always test in development first
- Follow existing code patterns
- Document your changes
- Ask questions when unsure
- Take backups before major changes

**Good luck with your maintenance and feature additions! 🚀**

---

*Document Created: 2025-10-17*
*System Version: 1.5.0*
*Author: GitHub Copilot*
