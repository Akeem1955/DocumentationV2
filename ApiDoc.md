It’s Friday night in Nigeria and the energy is clearly high! Crushing Day 2 of a hackathon is a massive win—getting these docs to the frontend team now is a pro move that prevents a "integration headache" on Sunday morning.

Here is your cleaned-up, professionally formatted Markdown documentation.

---

# 🦉 Owlyn Phase 1 API Specs - Auth & OTP

**Base URL:** `https://deb9-102-90-98-27.ngrok-free.app `

**Security:** All endpoints are **PUBLIC** except for `/api/auth/me`.

> 🚨 **FOR TESTING / DEMO PURPOSES**
> Use **`owlyn.admin@gmail.com`** for signup and login.
> The system will bypass standard email generation and the OTP will **ALWAYS** be **`123456`**.

---

### 1. Initiate Signup

Sends a 6-digit OTP to the user's email.

* **Method:** `POST /api/auth/signup`
* **Body:**
```json
{
  "email": "user@company.com",
  "password": "StrongPassword123",
  "fullName": "John Doe"
}

```


* **Success (200 OK):** `"OTP Sent Successfully"` (Plain text)
* **Error (409 Conflict):** `{"error": "User with email... already exists"}`

---

### 2. Verify Signup

*Creates Workspace & Generates JWT.*

* **Method:** `POST /api/auth/verify-signup?otp=123456&email=user@company.com`
* **Note:** Send parameters as **URL query parameters**.
* **Success (200 OK):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "user": {
    "id": "uuid-here",
    "email": "user@company.com",
    "fullName": "John Doe",
    "role": "ADMIN"
  }
}

```


* **Error (401 Unauthorized):** `{"error": "Invalid or expired OTP"}`

---

### 3. Initiate Login

Sends a 6-digit OTP to the user's email if credentials match.

* **Method:** `POST /api/auth/login`
* **Body:**
```json
{
  "email": "user@company.com",
  "password": "StrongPassword123"
}

```


* **Success (200 OK):** `"OTP Sent Successfully"` (Plain text)
* **Error (401 Unauthorized):** `{"error": "Invalid email or password"}`

---

### 4. Verify Login

* **Method:** `POST /api/auth/verify-login?otp=123456&email=user@company.com`
* **Note:** Send parameters as **URL query parameters**.
* **Success (200 OK):** Identical to *Verify Signup*. Returns `{token, user}`.
* **Error (401 Unauthorized):** `{"error": "Invalid or expired OTP"}`

---

### 5. Get Current User (Auth Required)

Validates the JWT and returns the user's data. Run this every time the Electron app opens to ensure the token isn't expired or revoked.

* **Method:** `GET /api/auth/me`
* **Headers:** `Authorization: Bearer <your_jwt_here>`
* **Success (200 OK):** Returns the User object.
* **Error (401 / 403):** Token missing, tampered with, or expired.
* **Frontend Action:** Clear local storage and redirect to login screen.

---

Hey! The backend for Phase 1 is officially done and secure. Everything aligns with the specs for F1.2 and F1.3 (JWT Bearer tokens, /api/auth/me returning 401 if expired, etc.).
However, I upgraded our security to use a 2-Factor OTP flow via Redis. This means your Login and Signup UI needs a slight tweak:
1. When they submit the Email+Password form, call /api/auth/signup (or login).
2. Show them a 6-digit OTP input modal.
3. Send that OTP to /api/auth/verify-signup (or verify-login) to get the actual JWT.
Also, for the 'Candidate Entry' screen, just build the UI for now. Don't worry about the API for candidates yet; we will link that up
