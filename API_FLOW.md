# EMS Complete API Flow - User to Database and Back

This guide shows the **complete round-trip**: how a request goes from the user through every backend layer to MongoDB, and then **returns back through all layers to the user's screen**.

---

## 🔐 Complete Login Flow (Start → End)

### **OUTGOING: User → Backend → Database**

#### 1. Frontend: User enters credentials

**File:** `frontend/src/pages/Login.tsx`

```tsx
function Login() {
    const handleSubmit = async (e: React.FormEvent) => {
        // User clicked "Login" button
        // STEP 1: Send email and password to backend
        const data = await AuthService.login(email, password)
    }
}
```

#### 2. Frontend: AuthService makes HTTP request

**File:** `frontend/src/services/authService.ts`

```typescript
export const AuthService = {
    login: (email: string, password: string): Promise<LoginResponse> => {
        // STEP 2: POST request to backend
        return api.post<LoginResponse>('/auth/login', { email, password })
            .then(r => r.data)
    }
}
```

**HTTP Request leaves browser:**

```http
POST http://localhost:8000/auth/login
Content-Type: application/json

{
  "email": "admin@company.com",
  "password": "secure123"
}
```

#### 3. Backend Router receives request

**File:** `backend/app/router/auth.py`

```python
@router.post("/login", response_model=LoginResponse)
async def login_user(
    payload: LoginRequest,  # FastAPI validates JSON here
    controller: AuthController = Depends(get_auth_controller)
) -> LoginResponse:
    # STEP 3: Router passes validated data to controller
    return await controller.login_user(payload)
```

#### 4. Backend Controller processes logic

**File:** `backend/app/controller/auth.py`

```python
async def login_user(self, payload: LoginRequest) -> LoginResponse:
    # STEP 4a: Call repository to find user by email
    user = await self.user_repo.find_user_by_email(payload.email)
    
    # STEP 4b: Verify password
    if not user or not verify_password(payload.password, user.hashed_password):
        raise HTTPException(status_code=401, detail="Invalid email or password")
    
    # STEP 4c: Log login action
    await self.user_repo.append_activity_log(user.email, ActivityLogEntry(...))
    
    # STEP 4d: Create JWT token
    token = create_access_token(email=user.email, role=user.role)
    
    # STEP 4e: Return response object (not HTTP response yet)
    return LoginResponse(access_token=token, token_type="bearer")
```

#### 5. Backend Repository queries database

**File:** `backend/app/repository/users.py`

```python
async def find_user_by_email(self, email: str) -> UserInDB | None:
    # STEP 5a: Query MongoDB for user by email
    user_data = await self.collection.find_one({"email": email})
    
    # STEP 5b: Return user document or None
    if user_data is None:
        return None
    
    user_data["id"] = str(user_data["_id"])
    return UserInDB(**user_data)
```

#### 6. MongoDB executes query

```javascript
// STEP 6: MongoDB queries its collection
db.users.findOne({ email: "admin@company.com" })

// Returns document:
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "email": "admin@company.com",
  "hashed_password": "$argon2id$v=19$m=65540,t=3,p=4$...",
  "role": "admin",
  "activity_log": [...]
}
```

---

### **RETURN: Database → Backend → User**

#### 7. Repository receives data from MongoDB

```python
# MongoDB connection responds with user document
# Motor async driver converts MongoDB cursor to Python dict
user_data = await self.collection.find_one({"email": email})
# Returns: {"_id": ObjectId(...), "email": "admin@...", "hashed_password": "...", ...}

# STEP 7: Repository transforms to UserInDB Pydantic model
return UserInDB(**user_data)  # Returns to Controller
```

#### 8. Controller receives repository result

```python
# Repository returned UserInDB object with user data
user = await self.user_repo.find_user_by_email(payload.email)
# user = UserInDB(id="507f...", email="admin@...", role="admin", ...)

# Controller verifies password
if not verify_password(payload.password, user.hashed_password):
    # Password matches! Continue...
    
# Create JWT token
token = create_access_token(email=user.email, role=user.role)

# STEP 8: Return LoginResponse object to Router
return LoginResponse(
    access_token="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbkBjb21wYW55...",
    token_type="bearer"
)
```

#### 9. Router receives response from Controller

```python
# Controller returned LoginResponse object
# FastAPI automatically:
# 1. Validates response matches LoginResponse model
# 2. Serializes to JSON
# 3. Sets HTTP status code (200 OK)
# 4. Sets Content-Type header

# STEP 9: FastAPI returns to client
return LoginResponse(...)  # FastAPI converts to JSON
```

#### 10. HTTP Response sent to browser

```http
HTTP/1.1 200 OK
Content-Type: application/json
Date: Wed, 22 May 2026 15:30:00 GMT

{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbkBjb21wYW55LmNvbSIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTcxNjQwMjIwMCwiZXhwIjoxNzE2NDA0MDAwfQ.abcdef123456...",
  "token_type": "bearer"
}
```

#### 11. Axios receives HTTP response

**File:** `frontend/src/api/axios.ts`

```typescript
// Axios gets 200 response
api.post('/auth/login', { email, password })
    .then(response => {
        // STEP 11: Response interceptor checks status
        // Status 200 = success, pass through
        return response
    })
```

#### 12. Frontend component receives data

```tsx
// AuthService.login() promise resolves
const data = await AuthService.login(email, password)

// STEP 12: data contains:
// {
//   access_token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
//   token_type: "bearer"
// }
```

#### 13. Frontend stores token in LocalStorage

**File:** `frontend/src/context/AuthContext.tsx`

```tsx
// Login page received the response data
const data = await AuthService.login(email, password)

// STEP 13a: Call AuthContext.login() to store token
login(data.access_token)

// AuthContext implementation:
const login = (newToken: string) => {
    // Save to localStorage (persists across page refreshes)
    localStorage.setItem('token', newToken)
    
    // Save to React state (immediate UI update)
    setToken(newToken)
}
```

#### 14. Frontend redirects user to employees page

```tsx
// Token is stored, now redirect
navigate('/employees')

// Browser navigates to new URL: http://localhost:5173/employees
```

#### 15. Frontend renders login success (UI updates)

```tsx
// Login.tsx component:
// - login() succeeded → no error shown
// - navigate() executed → page redirects
// - User sees "Loading employees..." on new page

// User is now authenticated ✅
```

---

## 📋 GET /employees - Protected Request (Return Path Detailed)

### **OUTGOING: User's browser → Backend → Database**

#### 1. Frontend: Employee page mounts

```tsx
function EmployeesList() {
    const { data: employees } = useEmployees()  // Hook runs on mount
}
```

#### 2. Hook calls EmployeeService

```typescript
// useEmployees hook:
useEffect(() => {
    const refetch = async () => {
        // STEP 1: Call API
        const data = await EmployeeService.list()
        setAll(data)
    }
    refetch()
}, [])
```

#### 3. Axios interceptor adds token to request

**File:** `frontend/src/api/axios.ts`

```typescript
api.interceptors.request.use((config) => {
    // STEP 2: Before sending request, read token from localStorage
    const token = localStorage.getItem('token')
    // token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    
    // Add Authorization header
    if (token) {
        config.headers['Authorization'] = `Bearer ${token}`
    }
    
    return config  // Request now has Authorization header
})
```

#### 4. HTTP request sent with token

```http
GET http://localhost:8000/employees
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbkBjb21wYW55LmNvbSIsInJvbGUiOiJhZG1pbiJ9...
Accept: application/json
```

#### 5. Backend dependency validates token

**File:** `backend/app/dependencies/users.py`

```python
async def get_current_user(request: Request, user_repo: UserRepository = Depends(...)):
    # STEP 3: Extract Bearer token from Authorization header
    auth_header = request.headers.get("Authorization")
    # auth_header = "Bearer eyJhbGciOi..."
    
    token = auth_header.split(" ")[1]
    # token = "eyJhbGciOi..."
    
    # Decode and verify JWT signature
    payload = decode_access_token(token)
    # payload = {"sub": "admin@company.com", "role": "admin", "exp": ..., "iat": ...}
    
    email = payload.get("sub")  # "admin@company.com"
    
    # Ensure user still exists in database
    user = await user_repo.find_user_by_email(email)
    
    # Returns authenticated user to route handler
    return user  # Passes to next layer
```

#### 6. Router receives authentication, calls controller

```python
@router.get("", response_model=List[EmployeeResponse])
async def get_all_employees(
    controller: EmployeeController = Depends(get_employee_controller)
) -> List[EmployeeResponse]:
    # STEP 4: get_current_user already validated token
    # This function only runs if authentication passed
    # (If token invalid, dependency raises 401 and this function never runs)
    
    return await controller.get_all_employees()
```

#### 7. Controller calls repository

```python
async def get_all_employees(self):
    # STEP 5: Query repository for all employees
    employees = await self.repository.get_all_employees()
    
    # Convert to response models
    return [EmployeeResponse(**emp) for emp in employees]
```

#### 8. Repository queries MongoDB

```python
async def get_all_employees(self) -> list:
    # STEP 6: Query MongoDB
    cursor = self.db.employees.find()
    
    # Collect documents
    employees = []
    async for employee in cursor:
        employees.append(employee)
    
    return employees
```

#### 9. MongoDB returns documents

```javascript
// MongoDB query
db.employees.find({})

// Returns all employee documents
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

---

### **RETURN: Database → Backend → Axios → Frontend → User's Screen**

#### 10. Repository receives MongoDB documents

```python
async def get_all_employees(self):
    cursor = self.db.employees.find()
    
    # STEP 7: Motor async driver returns documents
    employees = []
    async for employee in cursor:
        # employee is a dict from MongoDB
        # {"_id": ObjectId(...), "employee_id": "EMP00001", ...}
        employees.append(employee)
    
    # STEP 8: Return list to Controller
    return employees  # [dict, dict, dict, ...]
```

#### 11. Controller receives list from repository

```python
async def get_all_employees(self):
    # Repository returned list of dicts
    employees = await self.repository.get_all_employees()
    # employees = [{"_id": ObjectId(...), "employee_id": "EMP00001", ...}, ...]
    
    # STEP 9: Transform to response models
    response_models = [EmployeeResponse(**emp) for emp in employees]
    # response_models = [EmployeeResponse(employee_id="EMP00001", name="John Doe", ...), ...]
    
    # STEP 10: Return to Router
    return response_models
```

#### 12. Router receives response from controller

```python
@router.get("", response_model=List[EmployeeResponse])
async def get_all_employees(controller: EmployeeController = Depends(...)):
    # Controller returned list of EmployeeResponse objects
    result = await controller.get_all_employees()
    # result = [EmployeeResponse(...), EmployeeResponse(...), ...]
    
    # STEP 11: FastAPI automatically:
    # 1. Validates result matches List[EmployeeResponse] model
    # 2. Serializes each object to JSON dict
    # 3. Sets HTTP status 200 OK
    
    return result
```

#### 13. HTTP response built and sent to browser

```http
HTTP/1.1 200 OK
Content-Type: application/json
Date: Wed, 22 May 2026 15:35:00 GMT
Content-Length: 1247

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

#### 14. Axios receives response

```typescript
api.interceptors.response.use(
    (response) => {
        // STEP 12: Response interceptor receives 200 OK
        // response.status = 200
        // response.data = [employee objects]
        
        // Response is valid, pass it through
        return response
    }
)
```

#### 15. Frontend promise resolves with data

```typescript
// useEmployees hook:
const refetch = async () => {
    // API call completes
    const data = await EmployeeService.list()
    
    // STEP 13: data contains employee array
    // data = [
    //   {employee_id: "EMP00001", name: "John Doe", ...},
    //   {employee_id: "EMP00002", name: "Jane Smith", ...}
    // ]
    
    // STEP 14: Store in React state
    setAll(data)
}
```

#### 16. React re-renders with employee data

```tsx
function EmployeesList() {
    // State updated with employees
    const { data: employees } = useEmployees()
    // employees = [
    //   {employee_id: "EMP00001", name: "John Doe", ...},
    //   {employee_id: "EMP00002", name: "Jane Smith", ...}
    // ]
    
    // STEP 15: Component re-renders
    return (
        <table>
            <tbody>
                {employees.map(emp => (  // STEP 16: Map over employees
                    <tr key={emp.employee_id}>
                        <td>{emp.employee_id}</td>
                        <td>{emp.name}</td>
                        <td>{emp.email}</td>
                        <td>{emp.department}</td>
                        <td>{emp.position}</td>
                        <td>{emp.status}</td>
                    </tr>
                ))}
            </tbody>
        </table>
    )
}
```

#### 17. Browser renders HTML table on user's screen

```
┌─────────────────────────────────────────────────────┐
│ Employee Management                                  │
├──────────────┬──────────────┬──────────────────────┤
│ Employee ID  │ Name         │ Department           │
├──────────────┼──────────────┼──────────────────────┤
│ EMP00001     │ John Doe     │ IT                   │
│ EMP00002     │ Jane Smith   │ HR                   │
└──────────────┴──────────────┴──────────────────────┘

✅ USER CAN NOW SEE EMPLOYEES ON SCREEN
```

---

## Summary: Complete Round-Trip Journey

```
USER'S BROWSER
    ↓
    └→ Frontend Component (Login.tsx)
         └→ AuthService.login(email, password)
              └→ Axios HTTP Request
                   └→ NETWORK (internet)
                        └→ Backend Router (/auth/login)
                             └→ Controller.login_user()
                                  └→ Repository.find_user_by_email()
                                       └→ MONGODB query
                                            └→ Database returns document
                                                 └→ Repository returns UserInDB
                                                      └→ Controller creates JWT
                                                           └→ Router returns LoginResponse
                                                                └→ FastAPI serializes to JSON
                                                                     └→ HTTP 200 Response
                                                                          └→ NETWORK (internet)
                                                                               └→ Axios interceptor
                                                                                    └→ Frontend promise resolves
                                                                                         └→ AuthContext stores token
                                                                                              └→ React state updates
                                                                                                   └→ Navigate to /employees
                                                                                                        └→ useEmployees hook
                                                                                                             └→ API request with token
                                                                                                                  └→ [SAME JOURNEY ABOVE]
                                                                                                                       └→ Data received
                                                                                                                            └→ setAll(employees)
                                                                                                                                 └→ Component re-renders
                                                                                                                                      └→ Browser renders HTML table
                                                                                                                                           ↓
                                                                                                                                    USER'S SCREEN ✅
```

**Every response takes the SAME PATH BACK as the request came, but in reverse!**
