It’s Friday night in Nigeria and the energy is clearly high! Crushing Day 2 of a hackathon is a massive win—getting these docs to the frontend team now is a pro move that prevents a "integration headache" on Sunday morning.

Here is your cleaned-up, professionally formatted Markdown documentation.

---

# 🦉 Owlyn Phase 1 API Specs - Auth & OTP

**Base URL:** `http://localhost:8080`

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

**Would you like me to generate the corresponding `curl` commands for these endpoints so your frontend team can test them instantly in the terminal?**
