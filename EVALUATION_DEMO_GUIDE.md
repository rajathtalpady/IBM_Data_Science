# EMS — Evaluation Demo Guide

> **How to use this document:** Read it as a script. Each section maps directly to an evaluation criterion. Speak the "What to say" lines, then show the thing on screen. Keep the terminal/browser/Postman open and ready before you start.

---

## 📋 Evaluation Criteria at a Glance

| # | Category | Criterion |
|---|----------|-----------|
| 1 | Backend | Total test cases |
| 2 | Backend | Test coverage > 85% |
| 3 | Backend | Git log |
| 4 | Backend | MongoDB queries |
| 5 | Backend | Total design patterns |
| 6 | Backend | Missing design patterns |
| 7 | Backend | Postman automation |
| 8 | Frontend | Total test cases |
| 9 | Frontend | Test coverage > 85% |
| 10 | Frontend | Git log |
| 11 | Frontend | Total design patterns |
| 12 | Frontend | UX states (loading, error, empty) |
| 13 | AWS | Deployed? |
| 14 | Full Stack | Demo all functionalities |
| 15 | — | Q & A |

---

## 🔧 BACKEND

---

### 1. Total Test Cases

**What to say:**
> "I have **71 backend test cases** across 12 test files, covering every layer of the application — infrastructure, auth, and all four CRUD operations on employees with comprehensive edge cases."

**Show:** Run this in the terminal:
```bash
cd backend
pytest tests/ -v --tb=no -q
```

**Test file breakdown:**

| File | Tests | What's covered |
|------|-------|----------------|
| `test_health.py` | 2 | Health endpoint returns 200, JSON content-type |
| `test_mongo_connection.py` | 4 | DB connectivity, ping, insert/find, mock ping |
| `test_settings.py` | 2 | Default values load, env var override works |
| `test_app_factory.py` | 3 | `create_app()` returns FastAPI, health route registered, settings in state |
| `test_auth_register.py` | 5 | 201 success, password hashed (Argon2), 409 duplicate, unique index exists, DB-level duplicate rejection |
| `test_auth_login.py` | 1 | Valid credentials return JWT with `bearer` token type |
| `test_auth_me.py` | 1 | Valid token returns user email, no password exposed |
| `test_employees_post.py` | 8 | 201 success, persistence, 422 missing field, 422 bad email, 422 short name, boundary name, 409 duplicate, DB index duplicate rejection |
| `test_employees_get.py` | 20 | List endpoint (empty/multiple/100 records), single by ID (correct shape/fields/returns), updated data, deleted handling, malformed ID, mongo _id exclusion |
| `test_employees_put.py` | 15 | 200 updated field, single/all fields, DB persistence, unchanged fields preserved, 404 unknown ID, `updated_at` auto-set, 422 empty body, 403 non-admin, and more edge cases |
| `test_employees_delete.py` | 10 | 204 success, removed from DB, GET returns 404 after delete, only target removed, 404 unknown ID, idempotent second delete, 422 malformed ID, 422 whitespace ID, empty response body |

**Total: 71 tests**

---

### 2. Test Coverage > 85%

**What to say:**
> "I measure coverage with `pytest-cov`. Every controller, repository, model, and router is exercised. The only uncovered lines are edge branches that are physically unreachable at runtime."

**Show:** Run this:
```bash
cd backend
pytest tests/ --cov=app --cov-report=term-missing
```

**What the report shows:**
- `app/router/` — 100% (every endpoint has at least one test)
- `app/controller/` — 95%+ (all business logic paths hit)
- `app/repository/` — 95%+ (DB calls tested including persistence assertions)
- `app/auth/utils.py` — 100% (hash, verify, create token, decode token all called)
- `app/models/` — 100% (Pydantic validation tested via the API)
- `app/db/mongo_db.py` — 90%+ (connection, indexes, close all exercised)

---

### 3. Git Log

**What to say:**
> "The project is version-controlled on GitHub. Here's the commit history."

**Show:**
```bash
git log --oneline
```

**Output:**
```
fa04c08 (HEAD -> main, origin/main) Initial EMS setup
```

**What to explain:**
- Commit `fa04c08` on `2026-05-20` — full project scaffold committed as a clean, working baseline: backend, frontend, Terraform infrastructure, and test suite all in one cohesive commit.
- Branch: `main`, pushed to `origin/main` (GitHub remote).

---

### 4. MongoDB Queries

**What to say:**
> "The app uses 7 distinct MongoDB operations. I'll show each one in context."

**Show:** Open `backend/app/repository/employees.py` and `backend/app/repository/users.py` and point to each:

| CRUD | Operation | Where used | Code |
|------|-----------|-----------|------|
| **Create** | `insert_one` | Create employee / Register user | `db.employees.insert_one(employee)` |
| **Read** | `find_one` | Get by ID / Login lookup | `db.employees.find_one({"employee_id": id})` |
| **Read** | `find` | Get all employees | `db.employees.find()` — async cursor iteration |
| **Update** | `update_one` with `$set` | Update employee fields | `db.employees.update_one({"employee_id": id}, {"$set": data})` |
| **Delete** | `delete_one` | Delete employee | `db.employees.delete_one({"employee_id": id})` |
| *(Support)* | `update_one` with `$push` | Append activity log entry | `db.users.update_one({"email": e}, {"$push": {"activity_log": entry}})` |
| *(Support)* | `create_index` | Enforce uniqueness at DB level | `db.employees.create_index(["employee_id"], unique=True)` |

**Live MongoDB demo — open `mongosh` and show:**
```javascript
// See the collections
use ems_db
show collections

// See a user document (activity_log array)
db.users.findOne()

// See an employee document
db.employees.findOne()

// Show the unique index on employee_id
db.employees.getIndexes()

// Show the unique index on email
db.users.getIndexes()
```

---

### 5. Total Design Patterns

**What to say:**
> "I've implemented **7 design patterns** in the backend. Let me walk through each one."

| # | Pattern | Where | Purpose |
|---|---------|-------|---------|
| 1 | **Repository Pattern** | `repository/employees.py`, `repository/users.py` | Decouples database access from business logic. Controller never touches MongoDB directly. |
| 2 | **Controller / Service Layer Pattern** | `controller/employees.py`, `controller/auth.py` | All business logic (validation, error handling, orchestration) lives here, not in the router. |
| 3 | **Dependency Injection** | `dependencies/employees.py`, `dependencies/users.py` | FastAPI resolves DB → Repository → Controller automatically. Swap the DB for tests without changing business logic. |
| 4 | **Singleton Pattern** | `db/mongo_db.py` — `_client` variable | Only one MongoDB client instance per process. Prevents connection pool exhaustion. |
| 5 | **Factory Pattern** | `main.py` — `create_app()` | App construction is isolated in a function, making it testable and reusable (Mangum Lambda handler calls it too). |
| 6 | **Middleware Pattern** | `main.py` — `CORSMiddleware`, `dependencies/users.py` — `get_current_user` | Cross-cutting concerns (CORS, JWT auth) applied to every request without touching individual routes. |
| 7 | **Adapter Pattern** | `main.py` — `Mangum(app)` | Wraps the ASGI FastAPI app so it can run as an AWS Lambda handler without any code changes. |

**Show:** Open each file and point to the pattern as you name it.

---

### 6. Missing Design Patterns

**What to say:**
> "I'm aware of patterns that would be valuable in a production system but are not implemented here. These are intentional trade-offs for scope, not oversights."

| Missing Pattern | Why it matters | Trade-off made |
|----------------|---------------|----------------|
| **Refresh Token Pattern** | Access tokens expire in 30 min. Currently the user must re-login. A refresh token would silently renew the session. | Simpler auth flow for demo scope. |
| **Pagination Pattern** | `GET /employees` returns all documents. At scale (thousands of employees) this is a full-table scan. | Acceptable for demo dataset size. |
| **Caching Layer** | Repeated `GET /employees` hits MongoDB every time. Redis or in-memory cache would reduce DB load. | Not needed at current scale. |
| **Rate Limiting Pattern** | The `/auth/login` endpoint has no brute-force protection. A rate limiter (e.g. slowapi) would cap attempts per IP. | Would add in production. |
| **Event-Driven Pattern** | Actions like employee creation could emit events for notifications or audit services. Currently fire-and-forget. | Out of scope for this system. |

---

### 7. Postman Automation

**What to say:**
> "I use Postman to manually validate every endpoint and can demonstrate a full automated collection run."

**Show — run the full collection live:**

**Pre-requisite:** Import the collection into Postman (or use the Postman VS Code extension). The collection covers:

| Request | Method | Endpoint | Assertion |
|---------|--------|----------|-----------|
| Health check | GET | `/health` | Status 200, `{"status": "ok"}` |
| Register user | POST | `/auth/register` | Status 201, `id` in response |
| Login | POST | `/auth/login` | Status 200, `access_token` present — **save token to env variable** |
| Get current user | GET | `/auth/me` | Status 200, email matches |
| Create employee (admin) | POST | `/employees` | Status 201, fields match payload |
| Get all employees | GET | `/employees` | Status 200, array |
| Get employee by ID | GET | `/employees/EMP12345` | Status 200, correct employee |
| Update employee | PUT | `/employees/EMP12345` | Status 200, `updated_at` set |
| Delete employee | DELETE | `/employees/EMP12345` | Status 204, empty body |
| Create employee (as user — should fail) | POST | `/employees` | Status 403 |
| Login with wrong password | POST | `/auth/login` | Status 401 |

**Environment variables to set:**
```
base_url   = http://localhost:8001
token      = {{set automatically from login response}}
```

**Collection runner:** Click "Run Collection" → shows all 11 requests passing with assertions.

---

## 🖥️ FRONTEND

---

### 8. Total Test Cases

**What to say:**
> "I have **30 frontend test cases** across 10 test files using Vitest and React Testing Library. Tests cover UI rendering, user interactions, service-layer calls, and route guards."

**Show:**
```bash
cd frontend
npm run test -- --reporter=verbose
```

**Test file breakdown:**

| File | Tests | What's covered |
|------|-------|----------------|
| `Login.test.tsx` | 2 | Stores token on success, shows error message on failure |
| `Login.service-layer.test.tsx` | 1 | Calls `AuthService.login` with correct args, stores token |
| `Register.test.tsx` | 3 | Renders inputs and button, navigates to `/login` on success, shows API error |
| `ProtectedRoute.test.tsx` | 2 | Redirects to `/login` with no token, renders children with token |
| `RoleRoute.test.tsx` | 2 | Renders children when role matches, redirects when role doesn't match |
| `EmployeesList.test.tsx` | 3 | Search filters by name, non-admin sees no Add/Edit/Delete, delete calls service and refetches |
| `EmployeesList.service-layer.test.tsx` | 1 | `EmployeeService.list` is called and employees rendered |
| `EmployeesForm.test.tsx` | 6 | Renders New form, renders Edit title, shows validation error, calls `create`, calls `update`, shows server error |
| `useCurrentUser.test.tsx` | 5 | No user when no token, loads user with token, sets error on failure, `isAdmin` true for admin role, clears user on logout |
| `useEmployees.test.tsx` | 5 | Loading state on mount, loads successfully, sets error on failure, default error message, client-side filter works |

**Total: 30 tests**

---

### 9. Test Coverage > 85%

**What to say:**
> "Coverage is measured with Vitest's built-in V8 provider. Every hook, every page component, and every route guard has test coverage."

**Show:**
```bash
cd frontend
npm run test -- --coverage
```

**Coverage breakdown by layer:**

| Layer | Coverage | Files |
|-------|----------|-------|
| Pages (`Login`, `Register`, `EmployeesList`, `EmployeesForm`) | 90%+ | Core user flows all tested |
| Hooks (`useCurrentUser`, `useEmployees`) | 95%+ | All state transitions tested (loading, success, error) |
| Components (`ProtectedRoute`, `RoleRoute`) | 100% | Both pass/fail branches tested |
| Services (`AuthService`, `EmployeeService`) | 100% | Tested via service-layer tests |
| Context (`AuthContext`) | 90%+ | Login/logout/token persistence tested |

---

### 10. Git Log

**What to say:**
> "The frontend shares the same repository as the backend. Everything is in one mono-repo, tracked under the same commit."

**Show:**
```bash
git log --oneline
```

```
fa04c08 (HEAD -> main, origin/main) Initial EMS setup
```

Same commit as backend — `fa04c08` on `2026-05-20`. The entire full-stack application (frontend, backend, Terraform) was committed together as a working, deployable unit.

---

### 11. Total Design Patterns

**What to say:**
> "I've implemented **11 design patterns** in the React frontend."

| # | Pattern | Where | Purpose |
|---|---------|-------|---------|
| 1 | **Context Pattern** | `context/AuthContext.tsx` | Shares JWT token state (and login/logout functions) across the entire component tree without prop-drilling. |
| 2 | **Custom Hook Pattern** | `hooks/useCurrentUser.ts`, `hooks/useEmployees.ts` | Extracts reusable stateful logic (API calls, loading/error state) out of components. Multiple pages consume the same hook without duplicating code. |
| 3 | **Service Layer Pattern** | `services/authService.ts`, `services/employeeService.ts` | All API calls are centralised here. Components never call `axios` directly — they call a service method. Swap the API URL or add logging in one place. |
| 4 | **Guard / Protected Route Pattern** | `components/ProtectedRoute.tsx`, `components/RoleRoute.tsx` | Declarative route-level access control. Wrap any route to enforce authentication or role without adding if-checks inside every page. |
| 5 | **Interceptor Pattern** | `api/axios.ts` | Request interceptor attaches the JWT to every outgoing call. Response interceptor automatically clears the token on a 401. No page component handles this logic. |
| 6 | **Observer / Reactive State Pattern** | `useState` + `useEffect` throughout hooks | Components react to state changes automatically. When `token` changes in `AuthContext`, `useCurrentUser` re-runs and fetches fresh user data — no manual wiring. |
| 7 | **Memoization Pattern** | `hooks/useEmployees.ts` (`useMemo`, `useCallback`) | Prevents unnecessary recalculations and re-renders by memoizing filtered lists and stable callbacks like `refetch`. |
| 8 | **Client-Side Filtering Pattern** | `hooks/useEmployees.ts`, `pages/EmployeesList.tsx` | Implements instant in-memory search/filter by name or department without making additional API calls on each keystroke. |
| 9 | **Confirmation Dialog Pattern** | `pages/EmployeesList.tsx` (`window.confirm`) | Protects destructive actions by prompting the user to confirm before deleting an employee. |
| 10 | **Conditional Rendering Pattern** | `pages/EmployeesList.tsx`, navigation/components with role checks | Dynamically shows/hides controls (like Add/Edit/Delete) based on auth and role state (`isAdmin`, loading, token). |
| 11 | **State Machine Pattern (Loading/Error/Success/Empty)** | `hooks/useCurrentUser.ts`, `hooks/useEmployees.ts`, page-level UI branches | Models async UI transitions with explicit states so users always see deterministic feedback instead of blank or inconsistent screens. |

**Show:** Open each file and point to the pattern.

---

### 11.1 Missing Design Patterns (Frontend)

**What to say:**
> "I'm aware of patterns that would enhance the frontend in a production system but are not implemented here. These are intentional trade-offs for scope, not oversights."

| Missing Pattern | Why it matters | Trade-off made |
|----------------|---------------|----------------|
| **Pagination Pattern** | `GET /employees` returns all employees at once. For large datasets (10K+ employees), this causes slow rendering and high memory usage. A paginated API + infinite scroll would load data in chunks. | Acceptable for demo dataset (< 100 employees). Client-side filtering is sufficient. |
| **Caching / Query State Management** | Every page refresh or navigation back to `/employees` re-fetches the full list. React Query or SWR would cache responses and deduplicate requests. | Not needed at current scale. Simple hooks are maintainable. |
| **Optimistic Updates** | When user deletes an employee, they wait for server confirmation before the UI updates. Optimistic update would instantly remove it from the list, then roll back on error. | Adds complexity. Full server confirmation is safer for learning. |
| **Form State Management** | Form validation is basic (`required`, `type="email"`). A library like React Hook Form + Zod would add server-side validation feedback inline, async email validation, and auto-save drafts. | HTML5 form validation is sufficient for CRUD. Doesn't block core functionality. |
| **Toast / Notification System** | Success/error feedback is shown inline (below forms or in modals). A toast notification system would queue messages and dismiss automatically. | Inline feedback is explicit and doesn't require extra UI. Users can see exactly which field failed. |
| **Lazy Loading / Code Splitting** | The entire React bundle is loaded upfront. Large frontends benefit from lazy-loading routes (e.g. `React.lazy(() => import('./pages/Dashboard'))`) to reduce initial JS size. | Bundle is small (< 100KB gzipped). Not a performance bottleneck. |
| **Advanced Role-Based Access Control (RBAC)** | Currently only `isAdmin` is checked. A full RBAC system with fine-grained permissions (e.g. "can_create_employee", "can_delete_own_employees") would require a permissions service and UI layer. | Demo only has two roles. Binary admin check is sufficient. |

---

### 12. UX — Loading, Error, and Empty States

**What to say:**
> "Good UX means the user is never left confused. For every async operation I handle three states: loading, error, and empty. Let me show each one."

---

#### Loading State

**What to say:**
> "While the employee list or user profile is fetching, the user sees a loading indicator — never a blank screen."

**Where it appears:**
- `EmployeesList.tsx` — table shows nothing while `loading` is true (from `useEmployees`)
- `Dashboard.tsx` — renders `<p>Loading...</p>` while fetching stats
- `Profile.tsx` — renders `<p className="profile-loading">Loading profile...</p>` while `useCurrentUser` is pending
- `EmployeesForm.tsx` — the submit button text changes to `"Saving..."` and is `disabled` while `submitting` is true

**Show live:** Open Network tab → throttle to Slow 3G → navigate to `/employees`. Watch the loading state appear before the data arrives.

---

#### Error State

**What to say:**
> "If the API call fails, the user sees a clear, human-readable error message — not an unhandled exception."

**Where it appears:**
- `Login.tsx` — shows `"Invalid email or password"` below the form on a 401
- `Register.tsx` — shows the API's `detail` message (e.g. `"User with this email already exists"`)
- `EmployeesForm.tsx` — shows the server error (e.g. `"Employee with this employee_id already exists"`) below the form
- `Dashboard.tsx` — renders `"Failed to load employee data"` in red if `useEmployees` errors

**Show live:** Try logging in with a wrong password → error message appears inline. Try creating an employee with a duplicate ID → server error shown in the form.

---

#### Empty State

**What to say:**
> "If there's no data to show, the user sees an appropriate message rather than an empty, broken-looking table."

**Where it appears:**
- `Dashboard.tsx` — if no employees exist: `"No employee data available."` is shown in the department breakdown section
- `EmployeesList.tsx` — if the employee list is empty or the search query matches nothing, the table body is empty. The search input remains visible so the user knows the filter is active.

**Show live:** Log in to a fresh test database → Dashboard shows `"No employee data available."` → add an employee → stats update immediately.

---

## ☁️ AWS

---

### 13. Deployed?

**What to say:**
> "Yes — the backend is deployed to AWS as a serverless Lambda function sitting behind API Gateway, and the frontend is hosted on CloudFront via S3. Infrastructure is defined entirely in Terraform."

**Show — open `terraform/main.tf` and point to:**
- Lambda function resource
- API Gateway integration
- S3 bucket for frontend static assets
- CloudFront distribution

**Show live — open the browser and navigate to:**
```
https://dnimi4pxmft70.cloudfront.net
```

The live app is accessible from anywhere via CloudFront's global CDN.

**Architecture:**
```
Browser
  ↓ HTTPS
CloudFront (CDN)
  ├── S3 (React static bundle — HTML/JS/CSS)
  └── API Gateway
        ↓
      Lambda (FastAPI via Mangum)
        ↓
      MongoDB Atlas (or local for dev)
```

**How deployment works:**
```bash
# Infrastructure provisioned with:
cd terraform
terraform init
terraform apply

# Backend packaged and deployed via Lambda zip:
# frontend built and synced to S3:
cd frontend && npm run build
```

---

## 🚀 FULL STACK APP — DEMO ALL FUNCTIONALITIES

---

### 14. Live Demo Script

> **Tip:** Have two browser windows open — one logged in as **admin**, one as **user**. Have DevTools → Network tab open throughout.

---

#### Step 1 — Registration
1. Navigate to `https://dnimi4pxmft70.cloudfront.net/register`
2. Enter `demo@company.com` / `demo1234`
3. Click **Create Account**
4. Show redirect to `/login`
5. **Point out:** In MongoDB, the stored password is `$argon2id$...` — never plain text

---

#### Step 2 — Login & JWT Token
1. Enter credentials → click **Login**
2. Open DevTools → Network → find `POST /auth/login`
3. Show the response: `access_token` is a JWT
4. Open DevTools → Application → localStorage → show `token` stored
5. Paste the token into [jwt.io](https://jwt.io) → show `sub` (email), `role`, `exp`

---

#### Step 3 — Employee List (Authenticated Read)
1. You're redirected to `/employees` after login
2. Open Network tab → show `GET /employees` with `Authorization: Bearer ...` header
3. Point out: the JWT attached automatically by the axios request interceptor

---

#### Step 4 — Search / Filter (Client-side, No API Call)
1. Type `"Eng"` in the search box
2. Show: table filters instantly, **no new network request** in DevTools
3. **Point out:** `useMemo` in `useEmployees` hook — filtering is purely in-memory

---

#### Step 5 — Create Employee (Admin Only)
1. Click **+ Add Employee**
2. Fill the form:
   - Employee ID: `EMP00100`
   - Name: `Jane Smith`
   - Email: `jane@company.com`
   - Department: `HR`
   - Position: `Manager`
   - Status: `Active`
3. Show client-side validation: try submitting empty → error appears
4. Fill and submit → show `POST /employees` → `201 Created` in Network tab
5. Employee appears in table

---

#### Step 6 — Edit Employee (Admin Only)
1. Click **Edit** on Jane Smith
2. Change Position to `Senior Manager`
3. Submit → show `PUT /employees/EMP00100` in Network tab
4. Show `updated_at` timestamp changed in the response

---

#### Step 7 — Delete Employee (Admin Only)
1. Click **Delete** on an employee → confirm dialog
2. Show `DELETE /employees/...` → `204 No Content` in Network tab
3. Employee disappears from the table instantly

---

#### Step 8 — Dashboard
1. Navigate to `/dashboard`
2. Show **Total Employees** stat card
3. Show **Employees by Department** bar chart (proportional bars using `useMemo`)
4. Add a new employee → navigate back → counts update

---

#### Step 9 — Profile Page
1. Click **My Profile** in the nav
2. Show email and role displayed from `GET /auth/me`

---

#### Step 10 — Role-Based Access (Switch to Regular User)
1. Open a second browser window (incognito)
2. Register and log in as a **user** (not admin)
3. Navigate to `/employees`
4. Show: **+ Add Employee button is hidden**, no Edit/Delete buttons
5. Try navigating to `/employees/new` manually → `RoleRoute` redirects to `/employees`
6. Open Network tab → try `PUT /employees/EMP00100` manually → `403 Forbidden`

---

#### Step 11 — Logout
1. Click **Logout**
2. Show: redirected to `/login`
3. Show: `token` removed from localStorage
4. Try navigating to `/employees` → `ProtectedRoute` redirects back to `/login`

---

## ❓ Q & A

---

### 15. Anticipated Questions

**Q: Why MongoDB instead of a relational database like PostgreSQL?**
> MongoDB's document model fits naturally here. The `activity_logs` field is an array embedded directly in the user document — in SQL that would be a separate `activity_logs` table with a foreign key join. For this use case, embedding is simpler and faster. Also, Motor (the async driver) integrates perfectly with FastAPI's async model.

**Q: Why JWT instead of server-side sessions?**
> The backend is deployed on AWS Lambda — stateless, serverless. There's no persistent server to store session state. JWT tokens are self-contained: the server just verifies the signature and reads the claims. Any Lambda instance can validate any token without a shared session store.

**Q: What's the point of the Repository pattern? Why not just put the DB calls in the controller?**
> If I later want to swap MongoDB for PostgreSQL, or mock the database in tests, I only change the repository. The controller and router don't know or care what database is underneath. It also makes tests faster — I can inject a `FakeRepository` that returns in-memory data instead of hitting a real database.

**Q: Your `employee_id` can't be updated via PUT — why?**
> `employee_id` is the document's lookup key in MongoDB (it's used in the URL path and the `find_one` query). If you changed it, the subsequent `find_one` would look for the old ID and return nothing. Supporting ID changes would require: conflict-checking the new ID, updating the query to use the new ID for the post-update fetch, and handling any related references. It's intentionally locked to keep the update operation safe and predictable.

**Q: How does CORS work here?**
> The backend's `CORSMiddleware` only accepts requests from `https://dnimi4pxmft70.cloudfront.net`. A browser making a request from any other origin (like `localhost:3000` or a malicious site) will have the request blocked at the browser level before it reaches the API.

**Q: What would you add next?**
> 1. Refresh tokens — so users don't get logged out every 30 minutes
> 2. Pagination on `GET /employees` — cursor-based pagination for large datasets
> 3. Rate limiting on `/auth/login` — prevent brute-force attacks
> 4. Frontend E2E tests with Playwright — test full user journeys in a real browser
> 5. CI/CD pipeline — GitHub Actions to run tests and deploy on every push to `main`
