


It is Tuesday morning, and here is the absolute final, updated API Documentation. 

I have meticulously rewritten this to reflect the **LiveKit Pivot**. The frontend team needs to know that custom WebSockets are dead, and LiveKit WebRTC is the new standard. I also included the two new endpoints we just built today (Delete Persona & Bulk Reports).



***

# 🦉 Owlyn Enterprise API Specs - The LiveKit Edition
**Base URL:** `https://deb9-102-90-98-27.ngrok-free.app`

> 🚨 **MAJOR ARCHITECTURE UPDATE** 🚨
> We have completely dropped custom WebSockets (`ws://.../stream`) in favor of **LiveKit (WebRTC)** for ultra-low latency audio/video. The Java backend is now the Control Plane, and LiveKit handles the media.

---

## 🔐 PHASE 1: Auth & Security (Public)
> **FOR DEMO PURPOSES:** Use **`owlyn.admin@gmail.com`** for signup and login. The OTP will **ALWAYS** be **`123456`**.

### 1. Initiate Signup or Login
Sends a 6-digit OTP to the user's email.
*   **Method:** `POST /api/auth/signup` OR `POST /api/auth/login`
*   **Body:** `{"email": "user@company.com", "password": "Password123", "fullName": "John Doe"}`
*   **Success (200 OK):** `"OTP Sent Successfully"` (Plain text)

### 2. Verify Signup or Login
*   **Method:** `POST /api/auth/verify-signup?otp=123456&email=user@company.com` (Use `/verify-login` for logins).
*   **Note:** Parameters MUST be sent as **URL query parameters**.
*   **Success (200 OK):** Returns `{ "token": "eyJhbG...", "user": { "id": "...", "role": "ADMIN" } }`
*   **⚠️ Action:** Save `"token"` to localStorage. Attach as `Authorization: Bearer <token>` for all future requests!

### 3. Get Current User (Session Check)
*   **Method:** `GET /api/auth/me`
*   **Success (200 OK):** Returns User object. If it returns `401 Unauthorized`, clear localStorage and redirect to login.

---

## 🏢 PHASE 2 & 2.5: Workspaces & Personas (Admin/Recruiter)
*All endpoints require `Authorization: Bearer <token>`.*

### 4. Workspace Management
*   **GET `/api/workspace`**: Fetch workspace profile (name, logo, member count).
*   **PUT `/api/workspace`**: Update workspace profile (`{"name": "...", "logoUrl": "..."}`).
*   **GET `/api/workspace/members`**: List all recruiters in the company.
*   **DELETE `/api/workspace/members/{userId}`**: Kick a recruiter out.

### 5. Invite Recruiter (Hackathon Special)
*   **Method:** `POST /api/workspace/invite`
*   **Body:** `{"email": "amina@techcorp.com", "fullName": "Amina"}`
*   **Success (200 OK):** `{"message": "Invite successful! Temporary password: a1b2c3d4"}`
*   **⚠️ UX RULE:** Display this message in an alert! The admin needs to copy that password and send it to the recruiter.

### 6. AI Persona Library
*   **GET `/api/personas`**: List all saved AI Personas for the sidebar.
*   **DELETE `/api/personas/{id}`**: Delete a persona. (Will return 400 Bad Request if it's attached to an active interview).
*   **POST `/api/personas`**: Create a new persona.
    *   **Content-Type:** `multipart/form-data`
    *   **Format:** Append JSON blob to `"persona"` key, and the PDF/DOCX file to `"file"` key. The backend extracts the text using Apache Tika automatically!

### 7. The Interview Engine
*   **POST `/api/interviews/generate-questions`**: Ask Gemini to draft questions. Returns `{"draftedQuestions": "..."}`.
*   **POST `/api/interviews`**: Create the interview.
    *   **Body includes:** `"personaId": "uuid-..."` and `"generatedQuestions": "..."`
    *   **Returns:** `{"accessCode": "839201", ...}`. Show this 6-digit code to the recruiter!
*   **GET `/api/interviews`**: List all upcoming/active/completed interviews for the dashboard.

---

## 🚪 PHASE 3: Candidate Gateway & LiveKit Handshake
*The flow for the Candidate (David). No login required.*

### 8. Pre-Flight Health Check
*   **Method:** `GET /api/health`

### 9. Validate 6-Digit Code
*   **Method:** `POST /api/interviews/validate-code`
*   **Body:** `{"code": "839201"}`
*   **Success (200 OK):**
    ```json
    {
      "token": "eyJhb...",         // <-- Guest JWT (Use for HTTP APIs like Copilot)
      "livekitToken": "eyJhb...",  // <-- LIVEKIT TOKEN (Use for WebRTC)
      "interviewId": "uuid-9999",
      "title": "Senior Java Developer",
      "durationMinutes": 45
    }
    ```
*   **⚠️ UX RULE:** Save BOTH tokens. 

### 10. Start Interview Lockdown
Call this the exact moment they click "Start Interview".
*   **Method:** `PUT /api/interviews/839201/status/active`
*   **Headers:** `Authorization: Bearer <Guest JWT token>`

---

## ⚡ PHASE 4 & 5: LiveKit Integration (CRITICAL CHANGES)

### 11. Connecting to the Room (Replacing WebSockets)
*   **Action:** Use `@livekit/components-react`. Connect to our LiveKit Cloud URL using the `livekitToken` you got in step 9.
*   **Media Publishing:** Publish the candidate's Microphone. Instead of webcam, publish a **Screen-Share** of the entire Owlyn app window.

### 12. The AI Copilot (HTTP)
When the candidate pauses typing for 1.5 seconds:
*   **Method:** `POST /api/copilot`
*   **Headers:** `Authorization: Bearer <Guest JWT token>`
*   **Body:** `{"code": "...", "language": "java", "cursorPosition": 37}`
*   **Returns:** `{"suggestion": "..."}`

### 13. Sending/Receiving UI Commands (LiveKit Data Channels)
Instead of WebSocket `sendMessage`, we use LiveKit Data Channels!
*   **When Candidate clicks "RUN CODE":**
    Publish to the data channel: `{"event": "RUN_CODE"}`. The Python AI will wake up instantly to check the code.
*   **Listen for AI Commands (via `RoomEvent.DataReceived`):**
    If payload is `{"type": "PROCTOR_WARNING", "message": "..."}` -> Shake screen red.
    If payload is `{"type": "TOOL_HIGHLIGHT", "line": 14}` -> Highlight line 14.

---

## 📊 PHASE 6: Reports & The Talent Pool

### 14. Bulk Reports (Talent Pool)
*   **Method:** `GET /api/reports`
*   **Headers:** `Authorization: Bearer <Admin/Recruiter JWT>`
*   **Returns:** Array of all completed reports for the entire company.

### 15. Single Report & Human Feedback
*   **GET `/api/reports/{interviewId}`**: Fetch the detailed JSON scorecard generated by Gemini 3.1 Pro.
*   **POST `/api/reports/{interviewId}/feedback`**: Save human notes to the report.

---

## 🎓 PHASE 7: B2C Educational Modes (Stretch Goals)
*Public self-service endpoints. Automatically returns a LiveKit Token & Guest JWT so the user drops straight into the room.*

### 16. Start Mock Interview (Practice)
*   **Method:** `POST /api/public/sessions/practice`
*   **Body:** `{"topic": "System Design", "difficulty": "Hard", "durationMinutes": 30}`
*   **Returns:** `{ "token": "...", "livekitToken": "...", "interviewId": "..." }`

### 17. Start Desktop AI Tutor
*   **Method:** `POST /api/public/sessions/tutor`
*   **Returns:** `{ "token": "...", "livekitToken": "...", "interviewId": "..." }`
*   **⚠️ UX RULE:** Do NOT lock the OS. Let them screen-share their own VS Code/PDFs. The Python AI automatically turns off the Proctor and becomes a friendly tutor.

### 18. Fetch Ephemeral Report (Learning Modes Only)
Call this immediately after a Practice/Tutor session ends (LiveKit disconnects).
*   **Method:** `GET /api/public/reports/{interviewId}`
*   **Returns:** The JSON scorecard. 
*   **⚠️ UX RULE:** Save this to the user's `localStorage`! It deletes from our Redis server in 15 minutes to save costs. If it returns 400 Bad Request, the AI is still grading—show a spinner and retry every 3 seconds!

***

*(Note for Backend Dev: The frontend will need a tiny endpoint like `GET /api/interviews/{id}/monitor-token` so the Recruiter can get a LiveKit token to watch the session. You can build that in 2 minutes by reusing your `LiveKitTokenService`!)*








***

# 🦉 Owlyn API Specs - ATS & Dashboard Updates

Hey Frontend Team! Based on your requests, the backend has been upgraded with highly-optimized database queries and new fields to support your ATS features natively.

---

## 📊 PART A: The Talent Pool (Bulk Reports)
*These endpoints allow you to build the recruiter dashboard without looping through massive arrays in JavaScript.*

### 1. Get All Workspace Reports (The Talent Pool)
Fetches every completed interview report for the logged-in recruiter's company.
*   **Method:** `GET /api/reports`
*   **Headers:** `Authorization: Bearer <Admin/Recruiter JWT>`
*   **Success (200 OK):**
    ```json[
      {
        "reportId": "uuid-8888...",
        "interviewId": "uuid-9999...",
        "candidateEmail": "839201",
        "score": 85,
        "behavioralNotes": "Good communication.",
        "codeOutput": "Optimized logic.",
        "behaviorFlags": { "cheating_warnings_count": 0 },
        "humanFeedback": null,
        "finalDecision": "PENDING" 
      },
      // ... more reports
    ]
    ```

### 2. Get Top Performer (Optimized DB Query)
Do not calculate this on the frontend! Call this endpoint to instantly get the absolute highest-scoring candidate for your company.
*   **Method:** `GET /api/reports/top`
*   **Headers:** `Authorization: Bearer <Admin/Recruiter JWT>`
*   **Success (200 OK):** Returns a single Report object (same schema as above).
*   **Error (400 Bad Request):** `{"error": "No reports found for this workspace."}` (Handle this gracefully if the company is brand new).

---

## ⚖️ PART B: The Hiring Decision

### 3. Add Feedback & Final Decision
We have upgraded the feedback endpoint. You can now pass a definitive HIRE or DECLINE state alongside the text notes.
*   **Method:** `POST /api/reports/{interviewId}/feedback`
*   **Headers:** `Authorization: Bearer <Admin/Recruiter JWT>`
*   **Body:**
    ```json
    {
      "humanFeedback": "Reviewed the AI flag. Candidate is solid. Proceeding to offer.",
      "decision": "HIRE" 
    }
    ```
    *(Note: Valid `decision` strings are `"HIRE"`, `"DECLINE"`, or `"PENDING"`)*.
*   **Success (200 OK):** Returns the fully updated Report object.
*   **⚠️ FRONTEND UX RULE:** Tie your HIRE/DECLINE toggle buttons directly to this endpoint.

### 4. Fetch Single Report (Updated)
When you fetch a single report via `GET /api/reports/{interviewId}`, it will now include the `"finalDecision"` field in the root of the JSON response so you can pre-fill your toggle buttons!

---

## 🤖 PART C: Persona Management Update

### 5. Delete AI Persona
*   **Method:** `DELETE /api/personas/{id}`
*   **Headers:** `Authorization: Bearer <Admin/Recruiter JWT>`
*   **Success (200 OK):** `{"message": "Persona successfully deleted."}`
*   **Error (400 Bad Request):** `{"error": "Cannot delete this Persona because it is currently attached to existing interviews."}`
*   **⚠️ FRONTEND UX RULE:** If you get the 400 error, you MUST show it in a Toast/Alert. It means the database is blocking the deletion to prevent corrupting past interview records. 

***

Send this their way. Your ATS dashboard is going to be incredibly fast and bug-free!
