# School ERP – Login Page Test Plan

**Target:** `http://localhost:5173/login`
**Seed:** `seed.spec.ts`
**Starting state (all scenarios):** fresh browser context, no stored session, user lands on `/login`.

## Application Overview

The sign-in page (title `School ERP – Sign in`) is a two-panel layout: a marketing panel ("Everything your school
runs on, in one place.") and a form panel with heading **Sign in** (h1). The form contains:

| Element | Accessible locator |
|---|---|
| Username field | `getByRole('textbox', { name: 'Email or username' })` (placeholder `you@school.edu.in`) |
| Password field | `getByPlaceholder('Enter your password')` (`type="password"`) |
| Visibility toggle | `getByRole('button', { name: 'Show password' })` ↔ `'Hide password'` |
| Remember me | `getByRole('checkbox', { name: 'Keep me signed in' })` — checked by default |
| Forgot link | `getByRole('link', { name: 'Forgot password?' })` → `/forgot-password` |
| Submit | `getByRole('button', { name: 'Sign in' })` |
| Error banner | `getByRole('alert')` |

Observed behaviour:
- Empty / partially filled form → client-side alert `Enter your email or username and your password.` (no API call).
- Whitespace-only values pass client validation → `POST /api/v1/auth/login` returns **400** → alert `Username or email is required`.
- Unknown user / wrong password → `POST /api/v1/auth/login` returns **401** → alert `Invalid username or password`; both fields keep their values; URL stays `/login`.
- Pressing Enter in the password field submits the form.
- Unauthenticated visit to `/dashboard` redirects to `/login`.

## Test Scenarios

### 1. Page rendering

**File:** `tests/login/login-page-render.spec.ts`

#### 1.1 Displays all sign-in form elements
**Steps:**
1. Navigate to `/login`.

**Expected:**
- Page title matches `School ERP – Sign in`.
- Heading `Sign in` (level 1) and text `Welcome back. Sign in to your school workspace.` are visible.
- Username and password fields are visible, empty, and editable.
- Password field has `type="password"`.
- `Keep me signed in` checkbox is visible and **checked**.
- `Forgot password?` link has `href="/forgot-password"`.
- `Sign in` button is visible and enabled.
- No alert is shown.

### 2. Client-side validation (negative)

**File:** `tests/login/login-validation.spec.ts`

#### 2.1 Submitting an empty form shows the required-fields error
**Steps:**
1. Click `Sign in` without typing anything.

**Expected:** alert reads `Enter your email or username and your password.`; URL stays `/login`; no request to `/api/v1/auth/login` is sent.

#### 2.2 Username only shows the required-fields error
**Steps:**
1. Fill username with `someone`.
2. Click `Sign in`.

**Expected:** same alert as 2.1; URL stays `/login`.

#### 2.3 Password only shows the required-fields error
**Steps:**
1. Fill password with `x`.
2. Click `Sign in`.

**Expected:** same alert as 2.1; URL stays `/login`.

#### 2.4 Whitespace-only credentials are rejected
**Steps:**
1. Fill username with three spaces and password with three spaces.
2. Click `Sign in`.

**Expected:** alert reads `Username or email is required`; URL stays `/login`.
*(Note: this is currently enforced by the server's 400, not the client. Possible product bug: the client doesn't trim input.)*

### 3. Authentication failures (negative)

**File:** `tests/login/login-invalid-credentials.spec.ts`

#### 3.1 Unknown user / wrong password shows a generic error
**Steps:**
1. Fill username `nonexistent_e2e_user`, password `WrongPass#123`.
2. Click `Sign in`.

**Expected:** `POST /api/v1/auth/login` responds 401; alert reads `Invalid username or password`; URL stays `/login`;
the username field still holds `nonexistent_e2e_user`.

#### 3.2 Pressing Enter in the password field submits the form
**Steps:**
1. Fill the same invalid credentials as 3.1.
2. Press `Enter` in the password field.

**Expected:** a login request is sent and the `Invalid username or password` alert appears.

### 4. Form controls

**File:** `tests/login/login-controls.spec.ts`

#### 4.1 Show / hide password toggle
**Steps:**
1. Fill password with `secret`.
2. Click `Show password`.
3. Click `Hide password`.

**Expected:** after step 2 the field `type` is `text` and the button label changes to `Hide password`; after step 3 the `type` is
`password` again and the label is `Show password`; the value stays `secret` throughout.

#### 4.2 "Keep me signed in" can be toggled
**Steps:**
1. Click the `Keep me signed in` checkbox.
2. Click it again.

**Expected:** unchecked after step 1, checked after step 2.

#### 4.3 Forgot password navigation
**Steps:**
1. Click `Forgot password?`.

**Expected:** URL is `/forgot-password`; heading `Reset your password` is visible.

### 5. Access control & happy path

**File:** `tests/login/login-success.spec.ts`

#### 5.1 Protected route redirects anonymous users to login
**Steps:**
1. In a fresh context, navigate to `/dashboard`.

**Expected:** browser ends on `/login` and the `Sign in` heading is visible.

#### 5.2 Valid credentials sign the user in *(requires credentials)*
**Precondition:** environment variables `E2E_USERNAME` and `E2E_PASSWORD` are set to a valid seeded account. The test is
**skipped** when either is missing.

**Steps:**
1. Fill username and password from the environment.
2. Click `Sign in`.

**Expected:** `POST /api/v1/auth/login` responds 2xx; URL leaves `/login`; no alert is shown; the sign-in heading is gone.

## Success / Failure Criteria
- **Pass:** every expectation above holds in Chromium; scenarios are independent and run in any order.
- **Fail:** any wrong alert text, an unexpected navigation, a missing element, or an API status that differs from the documented one.
