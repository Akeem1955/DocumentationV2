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













It is 5:13 PM on Saturday here in Lagos. We have officially crushed the Phase 2 backend logic. 



***

# 🦉 Owlyn Phase 2 API Specs - Workspaces & Interviews

**Base URL:** `http://localhost:8080`
**Security:** All endpoints below require the `Authorization: Bearer <token>` header.

---

## 🏢 PART A: Workspace Settings (Admin Only)
*⚠️ FRONTEND NOTE: If the user's JWT decoded `role` is `"RECRUITER"`, hide the Workspace Settings page entirely. These endpoints will return `403 Forbidden` for recruiters.*

### 1. Get Workspace Details
Used to populate the top of the settings page.
*   **Method:** `GET /api/workspace`
*   **Success (200 OK):**
    ```json
    {
      "workspaceId": "uuid-1234-...",
      "name": "TechCorp's Workspace",
      "logoUrl": "https://example.com/logo.png",
      "memberCount": 3
    }
    ```

### 2. Update Workspace Profile
*   **Method:** `PUT /api/workspace`
*   **Body:**
    ```json
    {
      "name": "New TechCorp Name",
      "logoUrl": "https://new-url.png"
    }
    ```
*   **Success (200 OK):** Returns the updated Workspace object (same as GET above).

### 3. Get Team Members
Populates the data table of recruiters.
*   **Method:** `GET /api/workspace/members`
*   **Success (200 OK):**
    ```json[
      {
        "userId": "uuid-111",
        "fullName": "Alice Admin",
        "email": "alice@techcorp.com",
        "role": "ADMIN"
      },
      {
        "userId": "uuid-222",
        "fullName": "Amina Recruiter",
        "email": "amina@techcorp.com",
        "role": "RECRUITER"
      }
    ]
    ```

### 4. Invite a Recruiter (Hackathon Special)
*   **Method:** `POST /api/workspace/invite`
*   **Body:**
    ```json
    {
      "email": "amina@techcorp.com",
      "fullName": "Amina"
    }
    ```
*   **Success (200 OK):**
    ```json
    {
      "message": "Invite successful! Temporary password: a1b2c3d4"
    }
    ```
*   **⚠️ FRONTEND UX RULE:** You MUST display this exact message in a modal or alert. The admin needs to copy that temporary password and send it to the recruiter so they can log in!

### 5. Remove a Team Member
*   **Method:** `DELETE /api/workspace/members/{userId}` (Path Variable)
*   **Success (200 OK):** `{"message": "Member successfully removed."}`

---

## 🎙️ PART B: Interview Engine (Admin & Recruiter)

### 6. Generate AI Interview Questions (Agent 1 - Gemini 3.0 Flash)
Call this when the recruiter clicks "Generate Draft" in the setup form. It asks Gemini to write the questions.
*   **Method:** `POST /api/interviews/generate-questions`
*   **Body:**
    ```json
    {
      "jobTitle": "Senior Java Developer",
      "instructions": "Focus heavily on Spring Boot and AOP.",
      "questionCount": 5
    }
    ```
    *(Note: `instructions` and `questionCount` are optional).*
*   **Success (200 OK):**
    ```json
    {
      "draftedQuestions": "1. What is Spring AOP?\n2. How do you secure REST APIs?..."
    }
    ```
*   **⚠️ FRONTEND UX RULE:** Dump `draftedQuestions` into a large editable `<textarea>`. The recruiter must be able to read them, edit them, or type their own before final submission. If the API fails (Resilience4j fallback), it will return a friendly message asking them to type manually.

### 7. Finalize & Create Interview
Call this when they click "Create Interview". This generates the **6-digit access code** and saves it to the database.
*   **Method:** `POST /api/interviews`
*   **Body:**
    ```json
    {
      "title": "Senior Java Developer",
      "durationMinutes": 45,
      "toolsEnabled": {
        "codeEditor": true,
        "whiteboard": true,
        "notes": true
      },
      "aiInstructions": "Focus heavily on Spring Boot and AOP.",
      "generatedQuestions": "1. What is Spring AOP? 2. Explain Microservices..." 
    }
    ```
    *(Note: `generatedQuestions` is the final string from the textarea after they reviewed it).*
*   **Success (200 OK):**
    ```json
    {
      "interviewId": "uuid-9999-...",
      "title": "Senior Java Developer",
      "accessCode": "839201",
      "status": "UPCOMING"
    }
    ```
*   **⚠️ FRONTEND UX RULE:** Show a massive modal with the `accessCode` so the recruiter can copy it and send it to the candidate.

### 8. Get All Workspace Interviews (Dashboard)
Populates the main Interview Dashboard table.
*   **Method:** `GET /api/interviews`
*   **Success (200 OK):**
    ```json[
      {
        "interviewId": "uuid-9999-...",
        "title": "Senior Java Developer",
        "accessCode": "839201",
        "status": "UPCOMING"
      }
    ]
    ```

***


