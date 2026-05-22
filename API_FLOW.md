# EMS Complete API Flow with All Layers

This guide shows two complete end-to-end flows: **Register** and **Login + Protected Request**, tracing through every layer from frontend UI through router → controller → repository → MongoDB and back.

---

## 🔐 Complete Login Flow (Router → Controller → Repository → MongoDB)

### Why the user sends this request

The user has credentials and wants to prove their identity. Backend returns a JWT token that proves authentication for future requests.

### Frontend: User enters credentials and submits

**File:** `frontend/src/pages/Login.tsx`

```tsx
import { useState } from 'react'
import { AuthService } from '../services/authService'
import { useAuth } from '../context/AuthContext'
import { useNavigate } from 'react-router-dom'

function Login() {
    const [email, setEmail] = useState('')
    const [password, setPassword] = useState('')
    const [error, setError] = useState('')
    
    const { login } = useAuth()  // From AuthContext - stores token in localStorage
    const navigate = useNavigate()

    const handleSubmit = async (e: React.FormEvent) => {
        e.preventDefault()
        try {
            // STEP 1: Send credentials to backend via AuthService
            const data = await AuthService.login(email, password)
            
            // STEP 2: Backend returns JWT token
            // STEP 3: Save token to localStorage and React state (AuthContext)
            login(data.access_token)
            
            // STEP 4: Redirect to employees page (user is authenticated)
            navigate('/employees')
        } catch (err: any) {
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
            />
            <input 
                type="password" 
                placeholder="Password" 
                value={password}
                onChange={e => setPassword(e.target.value)}
            />
            <button type="submit">Login</button>
            {error && <p style={{color: 'red'}}>{error}</p>}
        </form>
    )
}
```

### Frontend: AuthService calls API

**File:** `frontend/src/services/authService.ts`

```typescript
import api from '../api/axios'

export interface LoginResponse {
    access_token: string
    token_type: string
}

export const AuthService = {
    login: (email: string, password: string): Promise<LoginResponse> => {
        // Uses shared axios instance (which auto-adds Authorization header)
        return api.post<LoginResponse>('/auth/login', { email, password })
            .then(r => r.data)
    }
}
```

**HTTP Request Sent to Backend:**

```http
POST http://localhost:8000/auth/login
Content-Type: application/json
Accept: application/json

{
  "email": "admin@company.com",
  "password": "secure123"
}
```

### Backend: Router receives and validates

**File:** `backend/app/router/auth.py`

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
    payload: LoginRequest,  # ← FastAPI validates JSON against this model
    controller: AuthController = Depends(get_auth_controller)  # ← Inject controller
) -> LoginResponse:
    """
    FastAPI validates request:
    1. Email field exists and is EmailStr format
    2. Password field exists and is non-empty string
    3. If invalid → 422 Unprocessable Entity
    
    Then calls controller with validated data
    """
    return await controller.login_user(payload)
```

**Request validation (Pydantic model):**

```python
from pydantic import BaseModel, EmailStr

class LoginRequest(BaseModel):
    email: EmailStr  # Must be valid email format
    password: str    # Must be non-empty
```

### Backend: Controller implements business logic

**File:** `backend/app/controller/auth.py`

```python
from app.repository.users import UserRepository
from app.auth.utils import verify_password, create_access_token
from app.models.users import LoginRequest, LoginResponse, ActivityLogEntry
from datetime import datetime, timezone
from fastapi import HTTPException, status

class AuthController:
    def __init__(self, user_repo: UserRepository):
        self.user_repo = user_repo  # Injected, not created here

    async def login_user(self, payload: LoginRequest) -> LoginResponse:
        """
        Login business logic:
        1. Find user in database by email
        2. Verify password matches the hash
        3. Log the login action
        4. Generate JWT token
        5. Return token to frontend
        """
        
        # STEP 1: Query repository to find user by email
        user = await self.user_repo.find_user_by_email(payload.email)
        
        # STEP 2: Check if user exists AND password is correct
        # IMPORTANT: Use constant-time comparison to prevent timing attacks
        if not user or not verify_password(payload.password, user.hashed_password):
            # Generic error message - don't reveal if email exists or password wrong
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid email or password"
            )
        
        # STEP 3: Log this login for audit trail
        await self.user_repo.append_activity_log(
            user.email,
            ActivityLogEntry(
                action="login",
                timestamp=datetime.now(timezone.utc)
            )
        )
        
        # STEP 4: Create JWT token with user's email and role
        token = create_access_token(email=user.email, role=user.role)
        
        # STEP 5: Return token to frontend
        return LoginResponse(access_token=token, token_type="bearer")
```

### Backend: Repository queries database

**File:** `backend/app/repository/users.py`

```python
from motor.motor_asyncio import AsyncIOMotorDatabase
from app.models.users import UserInDB

class UserRepository:
    def __init__(self, db: AsyncIOMotorDatabase):
        self.collection = db["users"]  # MongoDB collection handle

    async def find_user_by_email(self, email: str) -> UserInDB | None:
        """
        Query MongoDB for a user by email.
        
        This is called by controller.login_user() to find the user
        before password verification.
        """
        
        # MongoDB query: find_one({email})
        user_data = await self.collection.find_one({"email": email})
        
        # If no match found, return None
        if user_data is None:
            return None
        
        # Convert MongoDB _id (ObjectId) to string
        user_data["id"] = str(user_data["_id"])
        
        # Construct UserInDB Pydantic model from database document
        return UserInDB(**user_data)

    async def append_activity_log(self, email: str, entry: ActivityLogEntry) -> None:
        """
        Add a new activity log entry to user's activity log array.
        
        Called by controller after successful login to record the action.
        """
        
        # MongoDB update: append to activity_log array
        await self.collection.update_one(
            {"email": email},
            {"$push": {"activity_log": entry.model_dump()}}
        )
```

### Backend: MongoDB operations

```javascript
// Step 1: Find user by email
db.users.findOne({ email: "admin@company.com" })

// Returns this document (if found):
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "email": "admin@company.com",
  "hashed_password": "$argon2id$v=19$m=65540,t=3,p=4$...",
  "role": "admin",
  "activity_log": [
    {
      "action": "register",
      "timestamp": ISODate("2026-05-20T08:00:00Z")
    }
  ]
}

// Step 2: After password verification, append login event
db.users.updateOne(
  { email: "admin@company.com" },
  {
    $push: {
      activity_log: {
        action: "login",
        timestamp: ISODate("2026-05-22T14:30:00Z")
      }
    }
  }
)

// Now activity_log has two entries
```

### Backend: JWT Token Creation

**File:** `backend/app/auth/utils.py`

```python
from jose import jwt
from datetime import datetime, timedelta, timezone
from app.core.settings import settings

def create_access_token(email: str, role: str) -> str:
    """
    Create signed JWT token containing user info.
    
    Token payload includes:
    - sub: email (subject - who the token is for)
    - role: admin or user
    - iat: issued at timestamp
    - exp: expiration timestamp
    
    Signature proves token hasn't been tampered with.
    """
    
    # Calculate expiration: now + 30 minutes
    now = datetime.now(timezone.utc)
    expire = now + timedelta(minutes=settings.JWT_ACCESS_TOKEN_EXPIRE_MINUTES)
    
    # Create payload dict
    payload = {
        "sub": email,
        "role": role,
        "iat": now,
        "exp": expire
    }
    
    # Sign with secret key using HS256 algorithm
    # Only server with secret key can create valid tokens
    token = jwt.encode(
        payload,
        settings.JWT_SECRET_KEY,
        algorithm=settings.JWT_ALGORITHM
    )
    
    return token

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """
    Verify plain password matches the Argon2 hash.
    
    Uses constant-time comparison to prevent timing attacks.
    """
    from passlib.context import CryptContext
    pwd_context = CryptContext(schemes=["argon2"], deprecated="auto")
    return pwd_context.verify(plain_password, hashed_password)
```

**Example JWT Token:**

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiJhZG1pbkBjb21wYW55LmNvbSIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTcxNjQwMjIwMCwiZXhwIjoxNzE2NDA0MDAwfQ.
abcdef123456xyz789...

DECODED:
Header: { "alg": "HS256", "typ": "JWT" }
Payload: {
  "sub": "admin@company.com",
  "role": "admin",
  "iat": 1716402200,
  "exp": 1716404000
}
Signature: HMACSHA256(Header.Payload, "your-secret-key")
```

### Frontend: Store token and redirect

**File:** `frontend/src/context/AuthContext.tsx`

```typescript
import { createContext, useContext, useState } from 'react'

interface AuthContextType {
    token: string | null
    login: (newToken: string) => void
    logout: () => void
}

const AuthContext = createContext<AuthContextType>({} as AuthContextType)

export function AuthProvider({ children }: { children: ReactNode }) {
    // Initialize from localStorage so session persists on page refresh
    const [token, setToken] = useState(localStorage.getItem('token'))

    // Called by Login page after successful authentication
    const login = (newToken: string) => {
        // Save to localStorage (survives page refresh)
        localStorage.setItem('token', newToken)
        // Update React state (immediate UI updates)
        setToken(newToken)
    }

    const logout = () => {
        localStorage.removeItem('token')
        setToken(null)
    }

    return (
        <AuthContext.Provider value={{ token, login, logout }}>
            {children}
        </AuthContext.Provider>
    )
}

export const useAuth = () => useContext(AuthContext)
```

**Backend Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbkBjb21wYW55LmNvbSIsInJvbGUiOiJhZG1pbiJ9...",
  "token_type": "bearer"
}
```

**Frontend Storage:**

```javascript
// Token is saved in localStorage
localStorage.getItem("token")  
// → "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...."

// And in React state (AuthContext)
const { token } = useAuth()
// → "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...."

// Then redirected to /employees
```

---

## 📋 GET /employees - Protected Request with Token

### Why the user sends this request

After login, user is redirected to employees page. The page needs to fetch all employees. Token is automatically sent via axios interceptor.

### Frontend: Fetch employees page loads

**File:** `frontend/src/pages/EmployeesList.tsx`

```typescript
import { useEmployees } from '../hooks/useEmployees'

function EmployeesList() {
    // Hook fetches ALL employees in 1 API call
    const { data: employees, query, setQuery, refetch } = useEmployees()

    return (
        <>
            <h2>Employees</h2>
            <input 
                type="text" 
                placeholder="Search..." 
                value={query} 
                onChange={e => setQuery(e.target.value)}
            />
            <table>
                <tbody>
                    {employees.map(emp => (
                        <tr key={emp.employee_id}>
                            <td>{emp.name}</td>
                            <td>{emp.department}</td>
                        </tr>
                    ))}
                </tbody>
            </table>
        </>
    )
}
```

**File:** `frontend/src/hooks/useEmployees.ts`

```typescript
import { useEffect, useState, useMemo } from 'react'
import { EmployeeService } from '../services/employeeService'

export function useEmployees() {
    const [all, setAll] = useState([])
    const [query, setQuery] = useState('')

    useEffect(() => {
        // On component mount: fetch all employees from API
        const refetch = async () => {
            const data = await EmployeeService.list()
            setAll(data)
        }
        refetch()
    }, [])

    // Client-side filtering: instant, no API calls
    const data = useMemo(() => {
        const q = query.trim().toLowerCase()
        if (!q) return all
        return all.filter(e => 
            e.name.toLowerCase().includes(q) ||
            e.department.toLowerCase().includes(q)
        )
    }, [all, query])

    return { data, query, setQuery }
}
```

**File:** `frontend/src/services/employeeService.ts`

```typescript
import api from '../api/axios'

export const EmployeeService = {
    // GET /employees - fetch all employees
    list: (): Promise<Employee[]> =>
        api.get<Employee[]>('/employees').then(r => r.data)
}
```

### Frontend: Axios interceptor adds token to request

**File:** `frontend/src/api/axios.ts`

```typescript
import axios from 'axios'

const api = axios.create({
    baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8000'
})

// REQUEST INTERCEPTOR: Before every request, add JWT token from localStorage
api.interceptors.request.use((config) => {
    // Read token from localStorage (was saved after login)
    const token = localStorage.getItem('token')
    
    // Add to Authorization header in Bearer format
    if (token) {
        config.headers['Authorization'] = `Bearer ${token}`
    }
    
    return config
})

// RESPONSE INTERCEPTOR: If token expired (401), remove it
api.interceptors.response.use(
    response => response,
    error => {
        if (error.response?.status === 401) {
            localStorage.removeItem('token')
            window.location.href = '/login'
        }
        return Promise.reject(error)
    }
)

export default api
```

**HTTP Request sent (via interceptor):**

```http
GET http://localhost:8000/employees
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbkBjb21wYW55LmNvbSIsInJvbGUiOiJhZG1pbiJ9...
Content-Type: application/json
```

### Backend: Router validates authentication

**File:** `backend/app/router/employees.py`

```python
from fastapi import APIRouter, Depends, status
from app.dependencies.users import get_current_user, required_role

# All employees routes require a valid JWT token via get_current_user
# This is applied globally to the router
router = APIRouter(
    prefix="/employees",
    tags=["employees"],
    dependencies=[Depends(get_current_user)]  # ← Required for ALL routes
)

@router.get("", response_model=List[EmployeeResponse])
async def get_all_employees(
    controller: EmployeeController = Depends(get_employee_controller)
) -> List[EmployeeResponse]:
    """
    Fetch all employees.
    
    Before this function runs:
    1. Dependency get_current_user runs
    2. Extracts & validates JWT from Authorization header
    3. Returns authenticated user
    4. Only then does this endpoint execute
    """
    return await controller.get_all_employees()
```

### Backend: Dependency validates JWT token

**File:** `backend/app/dependencies/users.py`

```python
from fastapi import Depends, HTTPException, Request
from jose import JWTError, jwt
from app.auth.utils import decode_access_token

_CREDENTIAL_EXCEPTION = HTTPException(
    status_code=401,
    detail="Invalid credentials or missing token",
    headers={"WWW-Authenticate": "Bearer"}
)

async def get_current_user(
    request: Request,
    user_repo: UserRepository = Depends(get_user_repository)
) -> UserInDB:
    """
    Validate JWT token from Authorization header.
    
    This dependency is called BEFORE the endpoint function runs.
    If token is invalid → raises 401 → endpoint never executes.
    If valid → returns authenticated user → endpoint executes.
    """
    
    # STEP 1: Extract Bearer token from Authorization header
    auth_header = request.headers.get("Authorization")
    if not auth_header or not auth_header.startswith("Bearer "):
        raise _CREDENTIAL_EXCEPTION
    
    # STEP 2: Parse token from "Bearer <token>"
    token = auth_header.split(" ")[1]
    
    try:
        # STEP 3: Decode JWT and verify signature
        # If signature invalid or expired → JWTError → 401
        payload = decode_access_token(token)
        email = payload.get("sub")
        
        if email is None:
            raise _CREDENTIAL_EXCEPTION
        
        # STEP 4: Ensure user still exists in database
        # (user might have been deleted after token was issued)
        user = await user_repo.find_user_by_email(email)
        if user is None:
            raise _CREDENTIAL_EXCEPTION
        
        # STEP 5: Return authenticated user (passed to endpoint)
        return user
        
    except JWTError:
        raise _CREDENTIAL_EXCEPTION
```

**File:** `backend/app/auth/utils.py`

```python
from jose import jwt
from app.core.settings import settings

def decode_access_token(token: str) -> dict:
    """
    Decode and verify JWT signature.
    
    If signature invalid, tampered, or expired → raises JWTError
    """
    return jwt.decode(
        token,
        settings.JWT_SECRET_KEY,
        algorithms=[settings.JWT_ALGORITHM]  # HS256
    )
```

### Backend: Controller fetches employees

**File:** `backend/app/controller/employees.py`

```python
from app.repository.employees import EmployeeRepository
from app.models.employees import EmployeeResponse

class EmployeeController:
    def __init__(self, repository: EmployeeRepository):
        self.repository = repository

    async def get_all_employees(self) -> List[EmployeeResponse]:
        """
        Fetch all employees from database via repository.
        
        At this point, we know the request is authenticated
        (dependency get_current_user already validated JWT).
        """
        # Call repository to fetch from database
        employees = await self.repository.get_all_employees()
        
        # Convert to response model (safe schema)
        return [EmployeeResponse(**emp) for emp in employees]
```

### Backend: Repository queries MongoDB

**File:** `backend/app/repository/employees.py`

```python
from motor.motor_asyncio import AsyncIOMotorDatabase
import datetime

class EmployeeRepository:
    def __init__(self, db: AsyncIOMotorDatabase):
        self.db = db

    async def get_all_employees(self) -> list:
        """
        Query MongoDB employees collection for all documents.
        
        Returns all employee documents in the collection.
        """
        # MongoDB query: db.employees.find({})
        cursor = self.db.employees.find()
        
        # Collect all documents from cursor
        employees = []
        async for employee in cursor:
            # Convert MongoDB _id to string
            employee["id"] = str(employee["_id"])
            employees.append(employee)
        
        return employees
```

### MongoDB: Query executes

```javascript
// MongoDB query
db.employees.find({})

// Returns all documents:
[
  {
    "_id": ObjectId("507f1f77bcf86cd799439011"),
    "employee_id": "EMP00001",
    "name": "John Doe",
    "email": "john@company.com",
    "department": "IT",
    "position": "Developer",
    "status": "Active",
    "created_at": ISODate("2026-05-20T10:00:00Z"),
    "updated_at": null
  },
  {
    "_id": ObjectId("507f1f77bcf86cd799439012"),
    "employee_id": "EMP00002",
    "name": "Jane Smith",
    "email": "jane@company.com",
    "department": "HR",
    "position": "Manager",
    "status": "Active",
    "created_at": ISODate("2026-05-21T09:00:00Z"),
    "updated_at": null
  }
]
```

### Backend: Response returns to frontend

**HTTP Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "employee_id": "EMP00001",
    "name": "John Doe",
    "email": "john@company.com",
    "department": "IT",
    "position": "Developer",
    "status": "Active",
    "created_at": "2026-05-20T10:00:00Z",
    "updated_at": null
  },
  {
    "employee_id": "EMP00002",
    "name": "Jane Smith",
    "email": "jane@company.com",
    "department": "HR",
    "position": "Manager",
    "status": "Active",
    "created_at": "2026-05-21T09:00:00Z",
    "updated_at": null
  }
]
```

### Frontend: Render employees

The useEmployees hook stores the response in state, and the component renders them in a table. User can search client-side (instant, no API calls).

---

## ✍️ Complete Register Flow

### Why the user sends this request

User does not yet have an account, so they send email + password to create one.

### Frontend: Register component

**File:** `frontend/src/pages/Register.tsx`

```tsx
import { useState } from 'react'
import { useNavigate } from 'react-router-dom'
import { AuthService } from '../services/authService'

function Register() {
    const [email, setEmail] = useState('')
    const [password, setPassword] = useState('')
    const [error, setError] = useState('')
    const navigate = useNavigate()

    const handleSubmit = async (e: React.FormEvent) => {
        e.preventDefault()
        try {
            // Call backend to register
            await AuthService.register(email, password)
            
            // On success, redirect to login (user needs to authenticate)
            navigate('/login')
        } catch (err: any) {
            // Show error: "User with this email already exists" or validation error
            setError(err?.response?.data?.detail ?? 'Failed to register')
        }
    }

    return (
        <form onSubmit={handleSubmit}>
            <input 
                type="email" 
                value={email} 
                onChange={e => setEmail(e.target.value)}
                placeholder="Email"
            />
            <input 
                type="password" 
                value={password}
                onChange={e => setPassword(e.target.value)}
                placeholder="Password"
            />
            <button type="submit">Create Account</button>
            {error && <p style={{color: 'red'}}>{error}</p>}
        </form>
    )
}
```

### Frontend: API call via AuthService

```typescript
export const AuthService = {
    register: (email: string, password: string): Promise<UserResponse> => {
        return api.post<UserResponse>('/auth/register', { email, password })
            .then(r => r.data)
    }
}
```

**HTTP Request:**

```http
POST http://localhost:8000/auth/register
Content-Type: application/json

{
  "email": "newuser@company.com",
  "password": "secure123"
}
```

### Backend: Router and Controller

**Router validation:**

```python
@router.post(
    "/register",
    status_code=status.HTTP_201_CREATED,
    response_model=UserResponse
)
async def register_user(
    payload: UserCreate,  # ← Validated by Pydantic
    controller: AuthController = Depends(get_auth_controller)
) -> UserResponse:
    return await controller.register_user(payload)
```

**Controller logic:**

```python
async def register_user(self, payload: UserCreate) -> UserResponse:
    """
    Register a new user account.
    
    1. Check if email already exists (409 if duplicate)
    2. Hash password
    3. Create user document
    4. Insert into database
    """
    
    # STEP 1: Fast fail if email already registered
    existing_user = await self.user_repo.find_user_by_email(payload.email)
    if existing_user:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="User with this email already exists"
        )
    
    # STEP 2: Hash password (never store plain text)
    hashed_password = hash_password(payload.password)
    
    # STEP 3: Create user document with default role "user"
    user = UserInDB(
        email=payload.email,
        hashed_password=hashed_password,
        role="user",  # New users always get "user" role (not admin)
        activity_logs=[
            ActivityLogEntry(action="register", timestamp=datetime.now(timezone.utc))
        ]
    )
    
    # STEP 4: Insert into database via repository
    user_id = await self.user_repo.create_user(user)
    
    if not user_id:
        # Race condition: email was registered between check and insert
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="User with this email already exists"
        )
    
    # STEP 5: Return safe response (no password exposed!)
    return UserResponse(id=user_id, email=user.email, role=user.role)
```

### Backend: Repository inserts user

**File:** `backend/app/repository/users.py`

```python
from pymongo.errors import DuplicateKeyError

async def create_user(self, user: UserInDB) -> str:
    """
    Insert new user document into MongoDB users collection.
    
    Returns: MongoDB generated _id (as string)
    Raises: DuplicateKeyError if email already exists (unique index)
    """
    
    try:
        # MongoDB insert: db.users.insertOne(user_document)
        result = await self.collection.insert_one(
            user.model_dump(exclude={"id"})  # Don't send id field, let MongoDB generate
        )
        
        # Return the MongoDB-generated ObjectId as string
        return str(result.inserted_id)
        
    except DuplicateKeyError:
        # Unique index violation: email already exists
        # Repository returns None to indicate failure
        return None
```

### MongoDB: User document inserted

```javascript
// MongoDB insert operation
db.users.insertOne({
  email: "newuser@company.com",
  hashed_password: "$argon2id$v=19$m=65540,t=3,p=4$...base64hash...",
  role: "user",
  activity_log: [
    {
      action: "register",
      timestamp: ISODate("2026-05-22T15:00:00Z")
    }
  ]
})

// Returns
{
  acknowledged: true,
  insertedId: ObjectId("507f1f77bcf86cd799439050")
}

// Collection now has unique index on email:
db.users.getIndexes()
// [
//   { key: { _id: 1 } },
//   { key: { email: 1 }, unique: true, name: "uq_email_index" }
// ]
```

### Backend: Response returns

**HTTP Response:**

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": "507f1f77bcf86cd799439050",
  "email": "newuser@company.com",
  "role": "user"
}
```

### Frontend: Redirect to login

After successful 201 registration response, frontend redirects to /login page so user can authenticate.

---

## Summary: Why Each Layer Exists

| Layer | Why |
|-------|-----|
| **Frontend Component** | User interface - collect input, show results |
| **Frontend Service** | Encapsulate API calls - reusable across components |
| **Axios Interceptor** | Auto-add token to every request, handle 401 responses |
| **Router** | HTTP validation, dependency injection setup |
| **Controller** | Business logic - passwords, tokens, authorization |
| **Repository** | Database queries - CRUD operations only |
| **MongoDB** | Persistent data storage, indexes, unique constraints |

Each layer has one responsibility. Testable in isolation. Easy to modify without affecting others.
