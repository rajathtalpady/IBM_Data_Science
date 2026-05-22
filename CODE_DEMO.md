# EMS (Employee Management System) - Complete Demo Walkthrough

## 🎯 System Overview

The EMS is a **full-stack web application** for managing employees with secure authentication and role-based access control.

### Core Functionalities

1. **User Authentication** - Register, Login, Session Management
2. **Role-Based Access Control** - Admin vs User permissions
3. **Employee CRUD** - Create, Read, Update, Delete employees
4. **Data Validation** - All inputs validated before storage
5. **Search & Filtering** - Client-side employee search
6. **Activity Logging** - Track user actions for audit trail
7. **Dashboard** - Real-time employee statistics

---

## 🏢 Why This Architecture?

Before diving into code, here's WHY we built it this way:

### Why JWT Tokens (Instead of Sessions)?

**The Problem with Sessions:**
Every user needs a session record in a database. When they log in, create record. When they log out, delete record. When server has multiple instances, all need access to the same session database. With serverless (AWS Lambda), there's no persistent server anyway.

**The JWT Solution:**
The token itself contains all the information (email, role). The server just needs to verify the signature. Multiple servers can validate the same token without talking to each other. Perfect for stateless, serverless architecture.

- ✅ **Stateless**: No server-side storage needed
- ✅ **Scalable**: Works with serverless (AWS Lambda)
- ✅ **Distributed**: Multiple backend instances can validate without sharing state
- ❌ Sessions would require shared database for token storage

### Why Argon2 Password Hashing?

**The Problem with Old Algorithms:**
MD5 and SHA1 are FAST. That's terrible for passwords. Attackers can try billions of passwords per second on modern GPUs. 

**The Argon2 Solution:**
Argon2 is SLOW and uses lots of memory. It was designed to defeat GPU/ASIC attacks. Even with powerful hardware, cracking takes years per hash.

- ✅ **Memory-hard**: Uses RAM, not just CPU. Makes GPU attacks expensive
- ✅ **Tuneable**: Can increase cost (time/memory) as computers get faster
- ✅ **Industry standard**: Won Password Hashing Competition 2015
- ❌ MD5/SHA1 can be cracked in milliseconds (literally)

### Why MongoDB?

**The Problem with SQL:**
Employee data includes arrays (activity logs). SQL databases aren't great at storing nested arrays. You'd need a separate table, and then JOIN queries get complex.

**The MongoDB Solution:**
MongoDB is document-based. An employee is one document. Their activity logs are an array IN that document. No JOINs needed. And Motor (async MongoDB driver) fits perfectly with FastAPI's async/await pattern.

- ✅ **Flexible schema**: Can store arrays (activity_logs) directly in documents
- ✅ **Async driver (Motor)**: Fits FastAPI async pattern perfectly
- ✅ **Document-oriented**: Employee data is naturally one document
- ❌ SQL databases require more schema planning and complex JOINs

### Why Role-Based Access Control?

**The Problem with Frontend-Only Checks:**
If you hide buttons in React, that's just UI. A user can open DevTools console, find the API endpoint, and call it directly. "Delete /employees/123" doesn't care if the button was hidden.

**The Solution: Enforce at Backend:**
The backend checks the user's role BEFORE executing the action. Non-admins never reach the delete function. It returns 403 Forbidden before any damage is done.

- ✅ **Fine-grained permissions**: Different actions for different users
- ✅ **Enforced at backend**: Frontend hiding buttons is just UX, backend is security
- ✅ **Audit-friendly**: Who deleted what is logged before execution
- ❌ Trusting frontend for security = critical vulnerability

### Why Client-Side Search?

**The Problem with Server-Side Search:**
User types "J" → GET /employees?q=J → Network request (100ms)
User types "Jo" → GET /employees?q=Jo → Network request (100ms)
User types "Joh" → GET /employees?q=Joh → Network request (100ms)
User types "John" → GET /employees?q=John → Network request (100ms)

That's 4 API calls for typing one name. With 100 employees and a 50-keystroke search, you'd make 50 API calls!

**The Client-Side Solution:**
GET /employees (once) → Get all 100 employees → Store in browser
User types "J" → Filter in memory (instant, < 1ms)
User types "Jo" → Filter in memory (instant, < 1ms)
User types "John" → Filter in memory (instant, < 1ms)

Plus useMemo optimization means the filter only recalculates when data changes, not on every render.

- ✅ **Performance**: 1 API call instead of 50+
- ✅ **useMemo optimization**: Only recalculates when data changes
- ✅ **Works offline**: Data already in browser
- ❌ Server-side would create 100+ API calls per search session

### Why Dependency Injection?

**The Problem with Tight Coupling:**
```python
# BAD: Controller creates its own repository
class AuthController:
    def __init__(self):
        self.repo = UserRepository()  # Hard to test! Can't mock
```

To test the controller, you MUST use the real database. Can't inject a fake database for testing. The controller is "tightly coupled" to that specific repository.

**The Dependency Injection Solution:**
```python
# GOOD: Repository is passed in (injected)
class AuthController:
    def __init__(self, user_repo: UserRepository):
        self.user_repo = user_repo  # Easy to test! Can pass mock
```

Now for tests, pass a fake repository. Controller doesn't care where it comes from. For production, FastAPI injects the real one.

- ✅ **Testability**: Easy to mock database/repository for unit tests
- ✅ **Decoupling**: Changes to DB implementation don't affect controller logic
- ✅ **Reusability**: Same controller can work with different repositories
- ❌ Tight coupling = hard to test, can't swap implementations

---

## 🚀 How Backend Starts

**File:** `backend/app/main.py`

This file is the entry point of the entire application. It:
1. Creates the FastAPI app
2. Configures CORS (who's allowed to access)
3. Sets up database connections
4. Registers all route handlers
5. Handles startup/shutdown events

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from mangum import Mangum
from app.router.health import router as health_router
from app.router.employees import router as employees_router
from app.router.auth import router as auth_router
from app.db.mongo_db import ensure_indexes, close_client

@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Lifespan manages startup and shutdown events.
    
    Runs ONCE when the app starts (on_startup)
    Runs ONCE when the app stops (on_shutdown)
    """
    
    # ON STARTUP: Setup database
    db = await get_db()
    
    # Create indexes for fast lookups
    # email MUST be unique (can't register same email twice)
    # employee_id MUST be unique (can't have two employees with same ID)
    await ensure_indexes(db)
    
    yield  # App runs here
    
    # ON SHUTDOWN: Clean up
    await close_client()

def create_app() -> FastAPI:
    """Create and configure the FastAPI application"""
    
    app = FastAPI(
        title="Employee Management System",
        version="1.0.0",
        lifespan=lifespan  # Register our startup/shutdown handler
    )
    
    # CORS: Only allow requests from our frontend
    # Without CORS, browser blocks cross-origin requests
    app.add_middleware(
        CORSMiddleware, 
        allow_origins=[
            "https://dnimi4pxmft70.cloudfront.net",  # Production frontend on CloudFront
            "http://localhost:5173"  # Local dev (Vite runs on 5173)
        ],
        allow_methods=["GET", "POST", "PUT", "DELETE"],  # What HTTP methods allowed
        allow_headers=["*"],  # What headers allowed
        allow_credentials=True  # Allow cookies/auth headers
    )
    
    # Register route handlers
    # Each router is a separate module with related endpoints
    app.include_router(health_router)      # GET /health (for load balancer health checks)
    app.include_router(employees_router)   # /employees CRUD endpoints
    app.include_router(auth_router)        # /auth endpoints (login, register, etc)
    
    return app

# Create the app instance
app = create_app()

# For AWS Lambda deployment via Mangum
# Mangum converts the ASGI app to AWS Lambda handler format
handler = Mangum(app, lifespan="off")
```

**What happens:**
1. **On Startup**: MongoDB connection established, indexes created (ensures email/employee_id are unique)
2. **CORS Setup**: Only the frontend from specific domains can call our API
3. **Routes Registered**: All endpoints are registered with FastAPI (GET /health, POST /auth/login, GET /employees, etc)
4. **Ready to Accept Requests**: The app is now running and listening for HTTP requests
5. **On Shutdown**: MongoDB connection closed, cleanup done

---

## 🔐 Complete Login Flow

### What is Login?
Login is the process of verifying that you are who you claim to be (authentication), and then issuing you credentials (a token) that proves it for future requests.

### 1️⃣ Frontend: User Enters Credentials

**File:** `frontend/src/pages/Login.tsx`

This component:
- Collects email and password from user
- Validates format (email looks like email, password isn't empty)
- Sends to backend
- Stores token if successful
- Redirects to dashboard

```tsx
import { useState } from "react"
import { AuthService } from "../services/authService"
import { useAuth } from "../context/AuthContext"
import { useNavigate } from "react-router-dom"

function Login() {
    const [email, setEmail] = useState('')
    const [password, setPassword] = useState('')
    const [error, setError] = useState<string | null>(null)
    const { login } = useAuth()  // Updates global auth state
    const navigate = useNavigate()

    const handleSubmit = async (e: React.FormEvent) => {
        e.preventDefault()
        
        try {
            // STEP 1: Send credentials to backend
            const response = await AuthService.login(email, password)
            
            // STEP 2: If successful, backend returns a token
            // STEP 3: Store JWT token in localStorage (browser's local storage)
            login(response.access_token)
            
            // STEP 4: Redirect to employees page (user is now logged in)
            navigate('/employees')
            
        } catch (err: any) {
            // Login failed - show error message
            // Don't tell user if email or password is wrong (security)
            setError('Invalid email or password')
        }
    }

    return (
        <form onSubmit={handleSubmit}>
            <input 
                type="email" 
                placeholder="Email" 
                value={email} 
                onChange={e => setEmail(e.target.value)}
                required
            />
            <input 
                type="password" 
                placeholder="Password" 
                value={password}
                onChange={e => setPassword(e.target.value)}
                required
            />
            <button type="submit">Login</button>
            {error && <p style={{color: 'red'}}>{error}</p>}
        </form>
    )   
}
```

### 2️⃣ Frontend: Make API Call

**File:** `frontend/src/services/authService.ts`

This service provides methods for calling auth endpoints. It uses the axios instance (which we'll explain next) so authentication headers are automatically added.

```typescript
import api from '../api/axios'

export type UserRole = 'admin' | 'user'

export interface LoginResponse {
    access_token: string
    token_type: string
}

export const AuthService = {
    // Send email/password to backend
    // Backend returns JWT token if credentials are valid
    login: (email: string, password: string): Promise<LoginResponse> => {
        return api.post<LoginResponse>(
            '/auth/login',  // Backend endpoint
            { email, password }  // Request body
        ).then(r => r.data)  // Extract response data
    }
}
```

**HTTP Request Sent:**
```
POST http://localhost:8000/auth/login
Content-Type: application/json

{
  "email": "admin@company.com",
  "password": "secure123"
}
```

### 3️⃣ Backend: Route Receives Request

**File:** `backend/app/router/auth.py`

The router is like a dispatcher. It:
- Listens for POST requests to /auth/login
- Validates incoming JSON matches LoginRequest Pydantic model
- Injects a controller instance
- Calls the controller to handle the actual logic

```python
from fastapi import APIRouter, Depends, status
from app.models.users import LoginRequest, LoginResponse
from app.controller.auth import AuthController
from app.dependencies.users import get_auth_controller

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post(
    "/login",
    status_code=status.HTTP_200_OK,
    response_model=LoginResponse
)
async def login_user(
    payload: LoginRequest,  # ← FastAPI validates JSON matches this model
    controller: AuthController = Depends(get_auth_controller)  # ← Injects controller
) -> LoginResponse:
    """
    Login endpoint.
    
    Takes email and password, returns JWT token if valid.
    The 'Depends(get_auth_controller)' automatically creates and injects
    an AuthController instance with its dependencies.
    """
    return await controller.login_user(payload)
```

**What FastAPI Does Here:**
1. Receives JSON: `{"email": "...", "password": "..."}`
2. Validates it matches `LoginRequest` model (has email, password fields)
3. If invalid (missing field, wrong type), returns 422 Unprocessable Entity
4. If valid, creates `LoginRequest` object and passes to function
5. Calls `Depends(get_auth_controller)` which creates an `AuthController` instance
6. Calls `controller.login_user(payload)`

### 4️⃣ Backend: Controller Logic

**File:** `backend/app/controller/auth.py`

The controller contains the BUSINESS LOGIC - the actual "what should happen" when someone logs in.

```python
from app.auth.utils import verify_password, create_access_token
from app.repository.users import UserRepository
from app.models.users import LoginRequest, LoginResponse, ActivityLogEntry
from datetime import datetime, timezone
from fastapi import HTTPException, status

class AuthController:
    """
    AuthController handles all authentication logic.
    It doesn't know about HTTP or web requests - it just does the logic.
    """
    
    def __init__(self, user_repo: UserRepository):
        # Repository is injected - we don't create it ourselves
        # This makes testing easy (can inject fake repository)
        self.user_repo = user_repo

    async def login_user(self, payload: LoginRequest) -> LoginResponse:
        """
        Login logic:
        1. Find user in database by email
        2. Verify password matches (using Argon2)
        3. Create JWT token
        4. Return token to frontend
        """
        
        # STEP 1: Find user in MongoDB by email
        user = await self.user_repo.find_user_by_email(payload.email)
        
        # STEP 2: Check TWO things:
        # - User exists (email is registered)
        # - Password matches (Argon2 hash verification)
        if not user or not verify_password(payload.password, user.hashed_password):
            # IMPORTANT: Generic error message!
            # DON'T say "Email not found" or "Password wrong"
            # Attacker could learn which emails are registered
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED, 
                detail="Invalid email or password"
            )

        # STEP 3: Log this login action (audit trail)
        await self.user_repo.append_activity_log(
            user.email, 
            ActivityLogEntry(
                action="login",  # What happened
                timestamp=datetime.now(timezone.utc)  # When
            )
        )
        
        # STEP 4: Generate JWT token containing user's email and role
        # This token will be sent back to frontend
        token = create_access_token(
            email=user.email,  # Include email in token
            role=user.role     # Include role (admin/user)
        )
        
        # STEP 5: Return token to frontend
        return LoginResponse(
            access_token=token,  # The JWT token
            token_type="bearer"  # Type of auth (HTTP Bearer)
        )
```

### 5️⃣ Password Verification (Argon2)

**File:** `backend/app/auth/utils.py`

Password verification is the most security-critical part. Here's how Argon2 works:

```python
from passlib.context import CryptContext

# Set up Argon2 hasher
# schemes=["argon2"]: Use Argon2 algorithm (memory-hard, GPU-resistant)
pwd_context = CryptContext(schemes=["argon2"], deprecated="auto")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """
    Verify that plain_password matches the hashed_password.
    
    Never compares plain text passwords!
    This prevents password from being exposed if database is breached.
    """
    # pwd_context.verify() does:
    # 1. Re-hash the plain password using the salt from hashed_password
    # 2. Compare the two hashes
    # 3. Return True/False
    return pwd_context.verify(plain_password, hashed_password)
```

**How Argon2 Hash Verification Works:**
```
Step 1: User enters "secure123"
Step 2: Stored in DB: "$argon2id$v=19$m=65540,t=3,p=4$abc123def456$..."
Step 3: verify_password() re-hashes "secure123" using same salt (abc123def456)
Step 4: Compares generated hash with stored hash
Step 5: Returns True (they match) or False (they don't)

IMPORTANT: The password is NEVER stored or displayed as plain text!
```

### 6️⃣ JWT Token Generation

**File:** `backend/app/auth/utils.py`

JWT is the token that proves you're logged in. It's like a signed credential.

```python
from jose import jwt
from datetime import datetime, timedelta
from app.core.settings import settings

def create_access_token(email: str, role: str) -> str:
    """
    Create a JWT token with user information.
    
    JWT token contains:
    - email: who you are
    - role: what permissions you have (admin/user)
    - exp: when token expires
    - iat: when token was created
    
    The signature proves nobody has tampered with the token.
    """
    
    # Calculate expiration time: now + 30 minutes
    expire = datetime.utcnow() + timedelta(
        minutes=settings.JWT_ACCESS_TOKEN_EXPIRE_MINUTES
    )
    
    # Create the payload (the data in the token)
    payload = {
        "sub": email,  # subject = who the token is for
        "role": role,  # admin or user
        "exp": expire.timestamp(),  # when it expires (Unix timestamp)
        "iat": datetime.utcnow().timestamp()  # when it was created
    }
    
    # Sign with secret key using HS256 algorithm
    # Only the server with the secret key can create valid tokens
    token = jwt.encode(
        payload,
        settings.JWT_SECRET_KEY,
        algorithm=settings.JWT_ALGORITHM  # HS256
    )
    return token
```

**JWT Token Structure:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiJhZG1pbkBjb21wYW55LmNvbSIsInJvbGUiOiJhZG1pbiIsImV4cCI6MTcxNjMwMDAwMH0.
abcdef123456...

HEADER (Part 1):
  {
    "alg": "HS256",  // Algorithm used for signature
    "typ": "JWT"     // Type of token
  }

PAYLOAD (Part 2):
  {
    "sub": "admin@company.com",  // Who the token is for
    "role": "admin",             // Their permissions
    "exp": 1716300000,           // Expiration time (Unix timestamp)
    "iat": 1716296400            // When created
  }

SIGNATURE (Part 3):
  HMACSHA256(
    Header.Payload,
    "your-secret-key-from-settings"
  )

The signature proves:
  - Token was created by someone who knows the secret key
  - Token hasn't been tampered with
  - Token expiration date is valid
```

### 7️⃣ Frontend: Store Token & Redirect

After successful login, backend returns the token. Frontend stores it and redirects to dashboard.

```typescript
// Response from backend:
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbkBjb21wYW55LmNvbSIsInJvbGUiOiJhZG1pbiJ9...",
  "token_type": "bearer"
}

// Frontend AuthContext stores it in localStorage:
localStorage.setItem("token", "eyJhbGciOiJ...")

// Frontend redirects to /employees
navigate('/employees')

// Now the token is stored and will be sent with every request!
```

---

## 👥 Employees CRUD Operations

### How Every Request Stays Authenticated

**The Problem:** 
After login, how does the frontend prove it's the same person for every request? The server needs to verify the JWT token is valid.

**The Solution - Axios Interceptors:**
Every HTTP request automatically includes the token in the Authorization header. Every response checks if token is expired.

**File:** `frontend/src/api/axios.ts`

This is a custom axios instance that:
1. Automatically adds the token to EVERY request
2. Handles token expiration (401 errors)
3. Works with all API calls (login, employees, etc)

```typescript
import axios, { AxiosInstance } from 'axios'

// Create axios instance with base URL
const api: AxiosInstance = axios.create({
    baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8000',
    headers: {
        'Content-Type': 'application/json'
    }
})

/**
 * REQUEST INTERCEPTOR: Add token to every request
 * 
 * Before every request is sent, this runs:
 * - Get token from localStorage
 * - Add to Authorization header as "Bearer <token>"
 * - Send request with token
 */
api.interceptors.request.use((config) => {
    // Get the JWT token from localStorage
    const token = localStorage.getItem('token')
    
    // If token exists, add it to request headers
    // Format: Authorization: Bearer <token>
    // This is the standard for JWT authentication
    if (token) {
        config.headers.Authorization = `Bearer ${token}`
    }
    
    return config
})

/**
 * RESPONSE INTERCEPTOR: Handle 401 responses
 * 
 * After every response, check if token expired:
 * - 401 = Unauthorized (token expired or invalid)
 * - Clear localStorage
 * - Redirect to login
 */
api.interceptors.response.use(
    // Success response - pass it through
    (response) => {
        return response
    },
    // Error response - check if 401
    (error) => {
        if (error.response && error.response.status === 401) {
            // Token is expired or invalid
            localStorage.removeItem('token')
            
            // Redirect to login page
            // User must login again with new credentials
            window.location.href = '/login'
        }
        
        // Return the error so components can handle it
        return Promise.reject(error)
    }
)

export default api
```

**Example: How a request flows through interceptors:**

```
1. Frontend calls: api.post('/employees', {...})
   ↓
2. REQUEST INTERCEPTOR runs:
   - Gets token from localStorage: "eyJhbGci..."
   - Adds to config.headers.Authorization: "Bearer eyJhbGci..."
   ↓
3. Actual HTTP request sent with header:
   POST /employees
   Authorization: Bearer eyJhbGci...
   Content-Type: application/json
   
   {...payload...}
   ↓
4. Backend receives, validates token signature, checks expiration
   ↓
5. Response returns (200 OK or 401 Unauthorized)
   ↓
6. RESPONSE INTERCEPTOR runs:
   - Check status code
   - If 401: clear token, redirect to /login
   - If 200: pass response to component
```

### GET /employees - Fetch All Employees

**File:** `frontend/src/pages/EmployeesList.tsx`

This page:
- Fetches all employees (1 API call)
- Shows them in a table
- Allows client-side search (instant, no API calls)
- Shows edit/delete buttons only for admins

```tsx
import { Link } from "react-router-dom" 
import { useCurrentUser } from "../hooks/useCurrentUser"
import { useEmployees } from "../hooks/useEmployees"
import { EmployeeService } from "../services/employeeService"

function EmployeesList() {
    // useEmployees hook:
    // - Fetches all employees (1 API call)
    // - Stores in state
    // - Provides search/filter function
    const { data: employees, query, setQuery, refetch } = useEmployees()
    
    // useCurrentUser hook:
    // - Gets current user from context
    // - Provides isAdmin flag
    const { isAdmin, loading: userLoading } = useCurrentUser()

    // Handle delete button click
    const handleDelete = async (id: string) => {
        // Show confirmation dialog
        if (!window.confirm("Delete this employee?")) return
        
        // Call API to delete
        await EmployeeService.remove(id)
        
        // Refresh list
        refetch()
    }

    return (
        <div className="employees-page">
            <div className="employees-card">
                <div className="employees-header">
                    <h2 className="employees-title">Employee Management</h2>
                    
                    {/* Only show Add button if admin */}
                    {!userLoading && isAdmin && (
                        <Link to="/employees/new">
                            <button className="btn-add">+ Add Employee</button>
                        </Link>
                    )}
                </div>

                {/* Search box - client-side filtering */}
                <div className="employees-controls">
                    <input
                        type="text"
                        className="search-input"
                        placeholder="Search by name or department..."
                        value={query}
                        onChange={(e) => setQuery(e.target.value)}
                    />
                </div>

                {/* Display employees in table */}
                <div className="table-wrapper">
                    <table className="employees-table">
                        <thead>
                            <tr>
                                <th>Employee ID</th>
                                <th>Name</th>
                                <th>Email</th>
                                <th>Department</th>
                                <th>Position</th>
                                <th>Status</th>
                                {/* Show Actions column only for admins */}
                                {!userLoading && isAdmin && <th>Actions</th>}
                            </tr>
                        </thead>
                        <tbody>
                            {/* Map filtered employees to table rows */}
                            {employees.map(employee => (
                                <tr key={employee.employee_id}>
                                    <td>{employee.employee_id}</td>
                                    <td>{employee.name}</td>
                                    <td>{employee.email}</td>
                                    <td>{employee.department}</td>
                                    <td>{employee.position}</td>
                                    <td>{employee.status}</td>
                                    
                                    {/* Action buttons (edit/delete) for admins only */}
                                    {!userLoading && isAdmin && (
                                        <td className="action-cell">
                                            <Link to={`/employees/${employee.employee_id}/edit`}>
                                                <button className="btn-edit">Edit</button>
                                            </Link>
                                            <button 
                                                className="btn-delete" 
                                                onClick={() => handleDelete(employee.employee_id)}
                                            >
                                                Delete
                                            </button>
                                        </td>
                                    )}
                                </tr>
                            ))}
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    )
}

export default EmployeesList
```

**What happens when page loads:**
1. Component mounts
2. `useEmployees()` hook runs
3. Calls `GET /employees` with Authorization header
4. Backend returns all employees
5. Stored in component state
6. Rendered in table
7. User can search instantly (no API calls for search)
                value={query}
                onChange={(e) => setQuery(e.target.value)}
            />

            <table>
                <tbody>
                    {employees.map(emp => (
                        <tr key={emp.employee_id}>
                            <td>{emp.employee_id}</td>
                            <td>{emp.name}</td>
                            <td>{emp.department}</td>
                            <td>{emp.status}</td>
                            {isAdmin && (
                                <td>
                                    <button>Edit</button>
                                    <button onClick={() => handleDelete(emp.employee_id)}>Delete</button>
                                </td>
                            )}
                        </tr>
                    ))}
                </tbody>
            </table>
        </div>
    )
}
```

**Behind the scenes - useEmployees hook:**

```typescript
import { useEffect, useState, useMemo } from 'react'
import { EmployeeService } from '../services/employeeService'

export const useEmployees = () => {
    const [employees, setEmployees] = useState([])
    const [query, setQuery] = useState('')

    useEffect(() => {
        // Fetch on mount
        EmployeeService.list().then(setEmployees)
    }, [])

    // Client-side filtering (no API call!)
    const filtered = useMemo(() => {
        return employees.filter(emp => 
            emp.name.toLowerCase().includes(query.toLowerCase()) ||
            emp.department.toLowerCase().includes(query.toLowerCase())
        )
    }, [employees, query])

    return { data: filtered, query, setQuery }
}
```

**Backend - Route:**

```python
# backend/app/router/employees.py

@router.get("", response_model=List[EmployeeResponse])
async def get_all_employees(
    controller: EmployeeController = Depends(get_employee_controller)
) -> List[EmployeeResponse]:
    return await controller.get_all_employees()
```

**Backend - Controller:**

```python
async def get_all_employees(self):
    employees = await self.repository.get_all_employees()
    return [EmployeeResponse(**employee) for employee in employees]
```

**Backend - Repository:**

```python
async def get_all_employees(self):
    cursor = db.employees.find()
    employees = []
    async for emp in cursor:
        emp["_id"] = str(emp["_id"])
        employees.append(emp)
    return employees
```

**MongoDB:**
```javascript
db.employees.find({})

Returns:
[
  {
    "_id": ObjectId("..."),
    "employee_id": "EMP00001",
    "name": "John Doe",
    "email": "john@company.com",
    "department": "IT",
    "position": "Developer",
    "status": "Active",
    "created_at": ISODate("2026-05-21T10:00:00Z")
  },
  ...
]
```

---

### POST /employees - Create Employee (Admin Only)

**HTTP Request:**
```
POST /employees
Authorization: Bearer eyJhbGci...
Content-Type: application/json

{
  "employee_id": "EMP00123",
  "name": "Jane Smith",
  "email": "jane@company.com",
  "department": "HR",
  "position": "Manager",
  "status": "Active"
}
```

**Backend - Route with Role Check:**

```python
@router.post("", 
    status_code=status.HTTP_201_CREATED,
    response_model=EmployeeResponse,
    dependencies=[Depends(required_role("admin"))]  # ← Only admins!
)
async def create_employee(
    payload: EmployeeCreate,
    controller: EmployeeController = Depends(get_employee_controller)
) -> EmployeeResponse:
    return await controller.create_employee(payload)
```

**What `required_role("admin")` does:**

```python
# backend/app/dependencies/users.py

def required_role(role: str):
    async def check_role(current_user: UserInDB = Depends(get_current_user)):
        # Extract & validate JWT
        if current_user.role != role:
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return current_user
    return check_role

async def get_current_user(authorization: str = Header()):
    """Extract user from JWT token"""
    try:
        scheme, token = authorization.split()  # "Bearer <token>"
        
        if scheme != "Bearer":
            raise HTTPException(status_code=401)
        
        # Decode JWT (verify signature & expiry)
        payload = decode_access_token(token)
        
        # Find user in database
        user = await user_repo.find_user_by_email(payload["email"])
        
        if not user:
            raise HTTPException(status_code=401)
        
        return user
        
    except Exception:
        raise HTTPException(status_code=401, detail="Not authenticated")
```

**If non-admin tries:**
```
403 Forbidden
{
  "detail": "Insufficient permissions"
}
```

**If admin - Controller:**

```python
async def create_employee(self, payload: EmployeeCreate) -> EmployeeResponse:
    employee = await self.repository.create_employee(payload)
    
    if not employee:
        raise HTTPException(status_code=409, detail="Employee ID already exists")
    
    return EmployeeResponse(**employee)
```

**Repository - Insert into MongoDB:**

```python
async def create_employee(self, payload: EmployeeCreate):
    employee_dict = payload.model_dump()
    employee_dict["created_at"] = datetime.utcnow()
    
    try:
        result = await db.employees.insert_one(employee_dict)
        return {**employee_dict, "_id": str(result.inserted_id)}
    except DuplicateKeyError:
        # MongoDB unique index on employee_id
        return None
```

**MongoDB:**
```javascript
db.employees.insert_one({
  "employee_id": "EMP00123",
  "name": "Jane Smith",
  "email": "jane@company.com",
  "department": "HR",
  "position": "Manager",
  "status": "Active",
  "created_at": ISODate("2026-05-21T14:30:00Z")
})
```

**Response (201 Created):**
```json
{
  "id": "648a1b2c3d4e5f6g7h8i9j0k",
  "employee_id": "EMP00123",
  "name": "Jane Smith",
  "department": "HR",
  "position": "Manager",
  "status": "Active",
  "created_at": "2026-05-21T14:30:00Z"
}
```

---

### PUT /employees/{id} - Update Employee (Admin Only)

**HTTP Request:**
```
PUT /employees/EMP00123
Authorization: Bearer eyJhbGci...

{
  "position": "Senior Manager"
}
```

**Backend - Controller:**

```python
async def update_employee(self, employee_id: str, payload: EmployeeUpdate):
    # Validate format
    if not re.fullmatch(r"[a-zA-Z0-9_-]+", employee_id):
        raise HTTPException(status_code=422, detail="Invalid format")
    
    # Check exists
    existing = await self.repository.find_employee_by_id(employee_id)
    if not existing:
        raise HTTPException(status_code=404, detail="Not found")
    
    # Extract only provided fields (partial update)
    update_data = payload.model_dump(exclude_none=True)
    update_data.pop("employee_id", None)  # Can't change ID
    
    # Add timestamp
    update_data["updated_at"] = datetime.utcnow()
    
    # Update in MongoDB
    updated = await self.repository.update_employee(employee_id, update_data)
    
    return EmployeeResponse(**updated)
```

**Repository:**

```python
async def update_employee(self, employee_id: str, update_data: dict):
    result = await db.employees.find_one_and_update(
        {"employee_id": employee_id},
        {"$set": update_data},  # Only update fields in update_data
        return_document=ReturnDocument.AFTER
    )
    
    if result:
        result["_id"] = str(result["_id"])
    
    return result
```

**MongoDB:**
```javascript
db.employees.findOneAndUpdate(
  { "employee_id": "EMP00123" },
  { $set: { 
      "position": "Senior Manager",
      "updated_at": ISODate("2026-05-21T15:00:00Z")
    }
  },
  { returnDocument: "after" }
)
```

---

### DELETE /employees/{id} - Delete Employee (Admin Only)

**HTTP Request:**
```
DELETE /employees/EMP00123
Authorization: Bearer eyJhbGci...
```

**Backend - Route:**

```python
@router.delete("/{employee_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    dependencies=[Depends(required_role("admin"))]
)
async def delete_employee(
    employee_id: str,
    controller: EmployeeController = Depends(get_employee_controller)
) -> None:
    return await controller.delete_employee(employee_id)
```

**Controller:**

```python
async def delete_employee(self, employee_id: str):
    if not re.fullmatch(r"[a-zA-Z0-9_-]+", employee_id):
        raise HTTPException(status_code=422, detail="Invalid format")
    
    deleted = await self.repository.delete_employee(employee_id)
    
    if not deleted:
        raise HTTPException(status_code=404, detail="Not found")
```

**Repository:**

```python
async def delete_employee(self, employee_id: str):
    result = await db.employees.delete_one({"employee_id": employee_id})
    return result.deleted_count > 0
```

**MongoDB:**
```javascript
db.employees.deleteOne({ "employee_id": "EMP00123" })
```

**Response:**
```
204 No Content
(no body)
```

---

## 🧪 Tests - How We Validate Everything

**File:** `backend/tests/conftest.py` - Shared test setup:

```python
import pytest
from httpx import AsyncClient
from app.main import create_app

@pytest.fixture
async def app():
    return create_app()

@pytest.fixture
async def client(app):
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

@pytest.fixture
async def registered_user(client):
    """Helper: Create a test user"""
    await client.post("/auth/register", json={
        "email": "user@test.com",
        "password": "password123"
    })
    return {"email": "user@test.com", "password": "password123"}

@pytest.fixture
async def user_client(client, registered_user):
    """Client authenticated as regular user"""
    response = await client.post("/auth/login", json=registered_user)
    token = response.json()["access_token"]
    client.headers["Authorization"] = f"Bearer {token}"
    return client

@pytest.fixture
async def admin_client(client, test_db):
    """Client authenticated as admin"""
    await test_db.users.insert_one({
        "email": "admin@test.com",
        "hashed_password": hash_password("password123"),
        "role": "admin"
    })
    response = await client.post("/auth/login", json={
        "email": "admin@test.com",
        "password": "password123"
    })
    token = response.json()["access_token"]
    client.headers["Authorization"] = f"Bearer {token}"
    return client
```

### Test: Successful Login

**File:** `backend/tests/test_auth_login.py`

```python
async def test_login_with_valid_credentials(client, registered_user):
    """GIVEN: registered user | WHEN: login | THEN: return JWT token"""
    response = await client.post("/auth/login", json=registered_user)
    
    assert response.status_code == 200
    data = response.json()
    assert "access_token" in data
    assert data["token_type"] == "bearer"
    
    # Verify JWT format
    token = data["access_token"]
    assert token.count('.') == 2  # JWT has 2 dots
```

### Test: Failed Login

```python
async def test_login_with_wrong_password(client, registered_user):
    """WHEN: wrong password | THEN: 401 Unauthorized"""
    response = await client.post("/auth/login", json={
        "email": registered_user["email"],
        "password": "wrongpassword"
    })
    
    assert response.status_code == 401
```

### Test: Admin Can Create Employee

**File:** `backend/tests/test_employees_post.py`

```python
async def test_admin_creates_employee(admin_client):
    """GIVEN: admin user | WHEN: POST /employees | THEN: 201 Created"""
    response = await admin_client.post("/employees", json={
        "employee_id": "EMP00001",
        "name": "John Doe",
        "email": "john@company.com",
        "department": "IT",
        "position": "Developer",
        "status": "Active"
    })
    
    assert response.status_code == 201
    assert response.json()["name"] == "John Doe"
    assert "created_at" in response.json()
```

### Test: Non-Admin Cannot Create Employee

```python
async def test_non_admin_cannot_create_employee(user_client):
    """WHEN: regular user tries POST /employees | THEN: 403 Forbidden"""
    response = await user_client.post("/employees", json={...})
    
    assert response.status_code == 403
    assert "Insufficient permissions" in response.json()["detail"]
```

### Test: Get Employees

```python
async def test_get_all_employees(user_client, test_db):
    """WHEN: GET /employees | THEN: return all employees"""
    # Setup: Insert test data
    await test_db.employees.insert_many([
        {"employee_id": "EMP00001", "name": "John", ...},
        {"employee_id": "EMP00002", "name": "Sarah", ...}
    ])
    
    response = await user_client.get("/employees")
    
    assert response.status_code == 200
    employees = response.json()
    assert len(employees) == 2
```

### Running Tests

```bash
# Run all tests
pytest backend/tests/ -v

# Run with coverage
pytest backend/tests/ --cov=backend/app

# Run specific test
pytest backend/tests/test_auth_login.py -v

# Watch mode (auto-rerun on changes)
pytest-watch backend/tests/
```

---

## 📊 Complete Request/Response Flow Summary

### Login Flow

```
1. Frontend Form
   └─> Email: admin@company.com
   └─> Password: secure123

2. HTTP POST /auth/login
   └─> Headers: Content-Type: application/json
   └─> Body: {"email": "...", "password": "..."}

3. Backend Route
   └─> FastAPI deserializes JSON → LoginRequest
   └─> Dependency injection → AuthController instance

4. AuthController.login_user()
   └─> Repository.find_user_by_email() → queries MongoDB
   └─> verify_password() → Argon2 verify
   └─> If valid: create_access_token() → JWT
   └─> append_activity_log() → log action

5. MongoDB Query
   └─> db.users.findOne({email: "..."})
   └─> Return user doc with hashed_password

6. Response (200 OK)
   └─> {
         "access_token": "eyJhbGciOiJIUzI1NiIs...",
         "token_type": "bearer"
       }

7. Frontend
   └─> localStorage.setItem("auth_token", "...")
   └─> Redirect to /employees
```

### Create Employee Flow

```
1. Frontend Form → POST /employees
   └─> Bearer eyJhbGci...
   └─> {employee_id, name, email, department, position, status}

2. Backend - required_role("admin")
   └─> Extract token from Authorization header
   └─> Decode JWT
   └─> Check: current_user.role == "admin"
   └─> If false → 403 Forbidden
   └─> If true → continue

3. Backend - EmployeeController
   └─> Validate with Pydantic
   └─> Check for duplicates

4. Backend - Repository
   └─> await db.employees.insert_one(employee_dict)

5. MongoDB
   └─> Insert document with created_at timestamp
   └─> Unique index on employee_id prevents duplicates

6. Response (201 Created)
   └─> Returns employee data with _id and created_at

7. Frontend
   └─> Employee appears in table
   └─> useEmployees hook refetches data
```

---

## ✅ Key Concepts

| Concept | Implementation |
|---------|-----------------|
| **Authentication** | JWT tokens (expire in 30 min) |
| **Password Security** | Argon2 hashing (never plain text) |
| **Authorization** | Role-based checks (admin/user) |
| **Data Validation** | Pydantic models (both frontend & backend) |
| **Database Uniqueness** | MongoDB unique indexes on email & employee_id |
| **Async Operations** | Motor async driver for MongoDB |
| **Client-Side Perf** | useMemo for filtering (no API calls) |
| **API Consistency** | FastAPI automatic validation & serialization |
| **Testing** | pytest with fixtures for user/admin roles |
| **Error Handling** | HTTP status codes (401, 403, 404, 409, 422) |

---

## 🎬 Demo Script for Tomorrow

### 1. Start Backend & Frontend

```bash
# Terminal 1: Backend
cd backend
source venv/Scripts/activate
uvicorn app.main:app --reload --port 8000

# Terminal 2: Frontend
cd frontend
npm run dev
# Opens localhost:5173
```

### 2. Show Registration & Login

1. Click "Register"
2. Enter `demo@company.com` / `demo123`
3. Verify success
4. Redirect to login
5. Login with same credentials
6. Show token in DevTools localStorage
7. Paste in jwt.io and show payload

### 3. Show Employees List

1. Page loads
2. Open DevTools Network tab
3. Show GET /employees request
4. Show Authorization header with Bearer token
5. Show response with employee array

### 4. Show Search (Client-Side)

1. Type in search box
2. Shows filtered results
3. Point out: NO API call made (watch Network tab)

### 5. Show Create (Admin)

1. Click "+ Add Employee"
2. Fill form (use admin account if available)
3. Click Create
4. Show POST in Network tab
5. Show 201 response
6. Employee appears in table

### 6. Show Edit

1. Click Edit
2. Change position
3. Click Save
4. Show PUT request
5. Show updated_at changed

### 7. Show Delete

1. Click Delete
2. Confirm
3. Show DELETE request → 204 No Content
4. Employee removed

### 8. Show Tests

```bash
cd backend
pytest tests/ -v
```

Show all tests passing

---

**Your system demonstrates enterprise-grade full-stack development! Good luck with the demo! 🚀**
