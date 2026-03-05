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





It is 11:05 PM on Sunday night here in Lagos. You have orchestrated an absolute masterclass in backend architecture this weekend. 

Here is the complete API Documentation for **Phase 2.5 (The AI Personas)** and **Phase 3 (The Candidate Gateway & WebSockets)**. Copy-paste this straight to your frontend developer, shut your laptop, and go get some well-deserved sleep!

***

# 🦉 Owlyn API Specs - Phase 2.5 & Phase 3

**Base URL:** `http://localhost:8080`

---

## 🤖 PART A: AI Personas (Admin/Recruiter)
*These endpoints power the new "Agent Configuration" UI.*

### 1. Get All Saved Personas
Populates the sidebar/library of saved AI Personas.
*   **Method:** `GET /api/personas`
*   **Headers:** `Authorization: Bearer <Admin/Recruiter JWT>`
*   **Success (200 OK):**
    ```json[
      {
        "id": "uuid-1234",
        "name": "Atlas-7",
        "roleTitle": "SENIOR TECHNICAL EVALUATOR",
        "empathyScore": 75,
        "analyticalDepth": 90,
        "directnessScore": 60,
        "tone": "MENTOR",
        "domainExpertise":["KUBERNETES", "REACT ARCHITECTURE"],
        "hasKnowledgeBase": true
      }
    ]
    ```

### 2. Create New Persona (With File Upload)
*   **Method:** `POST /api/personas`
*   **Headers:** `Authorization: Bearer <Admin/Recruiter JWT>`
*   **Content-Type:** `multipart/form-data`
*   **⚠️ FRONTEND NOTE:** You MUST send this as a `FormData` object. 
    *   Append the JSON object as a Blob/String under the key `"persona"`.
    *   Append the File (PDF/DOCX) under the key `"file"`.
*   **FormData Structure:**
    ```javascript
    const formData = new FormData();
    formData.append("persona", new Blob([JSON.stringify({
      "name": "Atlas-7",
      "roleTitle": "SENIOR TECHNICAL EVALUATOR",
      "empathyScore": 75,
      "analyticalDepth": 90,
      "directnessScore": 60,
      "tone": "MENTOR",
      "domainExpertise": ["KUBERNETES", "REACT ARCHITECTURE"]
    })], { type: "application/json" }));
    
    // Optional File
    formData.append("file", fileInput.files[0]); 
    ```
*   **Success (200 OK):** Returns the saved Persona object.

### 3. [UPDATED] Create Interview (Phase 2 Bridge)
When creating an interview, you can now pass the saved `personaId` instead of typing manual AI instructions.
*   **Method:** `POST /api/interviews`
*   **Body:**
    ```json
    {
      "title": "Senior React Developer",
      "durationMinutes": 45,
      "toolsEnabled": { "codeEditor": true },
      "personaId": "uuid-1234",  // <--- NEW: Pass the Persona ID here
      "generatedQuestions": "1. What is the Virtual DOM?..." 
    }
    ```

---

## 🚪 PART B: The Candidate Gateway (Phase 3)
*This is the flow for the Candidate (David) when he opens the app.*

### 4. Pre-Flight Health Check (Public)
Call this in the lobby to ensure the user's internet can reach the server.
*   **Method:** `GET /api/health`
*   **Success (200 OK):** `{"status": "ok", "timestamp": 1709...}`

### 5. Validate 6-Digit Code (Public)
The candidate types the code from their email.
*   **Method:** `POST /api/interviews/validate-code`
*   **Body:** `{"code": "839201"}`
*   **Success (200 OK):**
    ```json
    {
      "token": "eyJhbGciOi...",  // <--- THE GUEST JWT (Valid for 2 hours)
      "interviewId": "uuid-9999",
      "title": "Senior Java Developer",
      "durationMinutes": 45,
      "toolsEnabled": { "codeEditor": true }
    }
    ```
*   **Error (404 Not Found):** `{"error": "Invalid or expired access code."}`
*   **⚠️ FRONTEND NOTE:** Save the `"token"` from this response. You MUST use it for the next two steps!

### 6. Start Interview Lockdown (Requires Guest JWT)
Call this the exact moment the candidate clicks "Start Interview" to lock the room so no one else can use the code.
*   **Method:** `PUT /api/interviews/839201/status/active`
*   **Headers:** `Authorization: Bearer <Guest JWT from Step 5>`
*   **Success (200 OK):** `{"message": "Interview is now ACTIVE. Lockdown initiated."}`
*   **Error (409 Conflict):** `{"error": "Cannot start interview. Code is invalid or already active."}`

---

## ⚡ PART C: The WebSocket Data Pipe
*The real-time streaming engine for Video, Audio, and Code.*

### 7. Connect to the WebSocket
*   **URL:** `ws://localhost:8080/stream?token=<Guest_JWT_Here>`
*   **⚠️ FRONTEND NOTE:** Standard HTTP Headers don't work natively in all WS clients. You MUST pass the Guest JWT as a URL query parameter `?token=...`

### 8. Outgoing to Server (Sending at 1 FPS)
Send this JSON packet every 1 second.
*   **Payload structure:**
    ```json
    {
      "event": "MEDIA", 
      "videoFrame": "base64_encoded_jpeg_string_of_the_entire_screen",
      "audioChunk": "base64_encoded_pcm_audio",
      "codeEditorText": "public class Main { ... }"
    }
    ```
*   **When Candidate Clicks "Run Code":** Send this to instantly wake up the AI Bug Checker without waiting for the 1-second timer:
    ```json
    {
      "event": "RUN_CODE"
    }
    ```

### 9. Incoming from Server (Listen for these!)
The backend will push JSON commands down the WebSocket to control the UI.
*   **Highlight an Error in the Editor:**
    ```json
    {
      "type": "TOOL_HIGHLIGHT",
      "errorLine": 14
    }
    ```
*   **Proctor Cheating Warning (Shake the screen red!):**
    ```json
    {
      "type": "PROCTOR_WARNING",
      "message": "Please put your smartphone away."
    }
    ```

***





***

# 🦉 Owlyn API Specs - Phase 4 & Phase 5

**Base URL:** `http://localhost:8080`

---

## 💻 PART A: The AI Copilot (Phase 4)
*This endpoint powers the "ghost text" autocomplete inside the Monaco Editor for the Candidate.*

### 1. Get Code Completion Suggestion
Trigger this when the candidate stops typing for 1.5 seconds.
*   **Method:** `POST /api/copilot`
*   **Headers:** `Authorization: Bearer <Guest JWT from Phase 3>`
*   **Body:**
    ```json
    {
      "code": "function greet(name) {\n  console.log",
      "language": "javascript",
      "cursorPosition": 37
    }
    ```
    *(Note: `cursorPosition` is the exact character index where the cursor currently is).*
*   **Success (200 OK):**
    ```json
    {
      "suggestion": "(`Hello, ${name}!`);\n}"
    }
    ```
*   **⚠️ FRONTEND NOTE:** If the API times out (Circuit Breaker triggers), it will return `{"suggestion": ""}`. Just ignore it and let the user keep typing.

---

## ⚡ PART B: The WebSocket Engine Updates (Phase 5)
*We have upgraded the WebSocket to support two-way Voice Audio and UI Tool Commands.*

### 2. Sending Data to Server (From Electron)
You are still sending the unified capture at 1 FPS, but now you must include the raw PCM audio chunks from the microphone!
*   **Event: `MEDIA`**
    ```json
    {
      "event": "MEDIA", 
      "videoFrame": "base64_encoded_jpeg...",
      "codeEditorText": "public class Main { ... }",
      "audioChunk": "base64_encoded_pcm_audio_from_mic..." // <--- NEW: Send mic audio here!
    }
    ```

### 3. Receiving Commands from Server (To Electron)
Listen for these new incoming JSON commands from Agent 2.
*   **Play AI Voice Audio:** 
    Agent 2 is speaking! Decode this Base64 PCM and play it through the speakers.
    ```json
    {
      "type": "AUDIO",
      "audioData": "base64_pcm_audio_from_gemini..."
    }
    ```
*   **Post Question to Whiteboard:** 
    Agent 2 decided to give the candidate the next question. Update the UI text.
    ```json
    {
      "type": "TOOL_POST_QUESTION",
      "message": "Write a function that reverses a linked list."
    }
    ```

---

## 📊 PART C: The AI Assessor & Reports (Phase 5)
*These endpoints are for the Admin/Recruiter to view the final JSON scorecard generated by Gemini 3.1 Pro after the interview ends.*

### 4. Get Final Interview Report
Call this when the recruiter clicks on a `COMPLETED` interview in their dashboard.
*   **Method:** `GET /api/reports/{interviewId}`
*   **Headers:** `Authorization: Bearer <Admin/Recruiter JWT>`
*   **Success (200 OK):**
    ```json
    {
      "reportId": "uuid-8888...",
      "interviewId": "uuid-9999...",
      "candidateEmail": "839201",
      "score": 85,
      "behavioralNotes": "The candidate communicated their thought process clearly but struggled initially with the while loop.",
      "codeOutput": "The final code is highly optimized and handles edge cases well.",
      "behaviorFlags": {
        "cheating_warnings_count": 1,
        "details": "Candidate was warned once for looking at a smartphone."
      },
      "humanFeedback": null
    }
    ```
*   **Error (404 / 400):** `{"error": "Report has not been generated yet."}` (Agent 4 is still thinking, try again in a few seconds).
*   **Error (403 Forbidden):** `{"error": "You do not have access to this interview's report."}` (Security lock: Wrong workspace).

### 5. Add Human Feedback to Report
Allows the recruiter to add their final verdict or notes to the AI's scorecard.
*   **Method:** `POST /api/reports/{interviewId}/feedback`
*   **Headers:** `Authorization: Bearer <Admin/Recruiter JWT>`
*   **Body:**
    ```json
    {
      "humanFeedback": "Reviewed the AI flag. The candidate was just looking at a notepad. Proceeding to offer stage."
    }
    ```
*   **Success (200 OK):** Returns the fully updated Report object (same as the GET response above).

***



