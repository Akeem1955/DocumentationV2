# OWLYN — 19-Day Hackathon Execution Plan

> **Start**: Feb 26, 2026 → **Deadline**: Mar 16, 2026  
> **Core Stack**: Electron (Frontend), Java/Spring Boot (Cloud Backend & ADK), Gemini 2.5 Multimodal Live API, Google Cloud Platform (GCP)  
> **Rule**: Each phase has a checkpoint. Phase is BLOCKED until all items pass.

---

## Architecture Overview — Secure Cloud-Controlled Design

### The Security Reality

This is an anti-cheat proctoring system. If the Gemini API keys or system prompts lived on the candidate's local machine, a smart candidate could steal the API keys, read the hidden proctoring instructions, or intercept and rewrite the AI's final scorecard. The **Java Cloud Server** is the secure fortress. It holds the ADK, the API keys, and the secret instructions. The candidate cannot touch it.

### The Two Roles

| Component | Location | Role | Analogy |
|-----------|----------|------|---------|
| **Java Spring Boot** | Cloud Server | **The Brain** — Controls the ADK, opens WSS to Gemini, holds API keys & system prompts, generates reports, writes to Cloud SQL | Decision-maker |
| **Electron App** | Candidate's Machine | **The Senses** — Captures webcam + microphone, streams raw media up to Java via WSS, renders UI, hosts Monaco code editor | Dumb camera/mic pipe |

### Data Flow

```
[ CANDIDATE'S LOCAL MACHINE ]
Electron App (Captures A/V)  ─── WSS ──→  Java Cloud Server

[ GOOGLE CLOUD PLATFORM ]
Java Cloud Server (Spring Boot / ADK)  ─── WSS ──→  Gemini 2.5 Live API
Java Cloud Server (Spring Boot / ADK)  ─── SQL ──→  Google Cloud SQL
Java Cloud Server (Spring Boot / ADK)  ─── REST ──→ Sandbox API (Judge0)
```

### The Configurable Workspace Tools

Recruiters configure the workspace per interview (e.g., Algorithms vs. System Design). The tools available to the candidate are:

| Tool | Type | Description |
|------|------|-------------|
| **Code Editor + Runner** | Optional | Monaco editor integrated with a Sandbox execution API (Judge0) for real code execution |
| **Whiteboard** | Optional | HTML5 Canvas for architecture diagrams, parsed via Gemini Flash Vision |
| **Notes** | Optional | Plain text scratchpad |
| **Camera/Mic** | Mandatory | Always-on for proctoring |
| **AI Interviewer** | Mandatory | Voice interface (Gemini Live) |

### The Interview Loop

1. Electron captures user's camera (1fps JPEG) + mic (16kHz PCM)
2. Electron streams raw media up to **Java Cloud Server** via secure WebSocket
3. Java pipes the media into **Gemini 2.5 Live API** via the ADK
4. Gemini responds with voice → Java sends audio back down to Electron → candidate hears the AI
5. Candidate clicks **"Run / Review Workspace"** → Java sends code to an isolated **Sandbox API** (e.g., Judge0) for execution and parses Whiteboard via **Vision API** → Java injects the FACTUAL execution results into the Gemini Live stream → Gemini speaks feedback based on verified facts, preventing hallucinations
6. Gemini dictates the final report → Java writes it directly to **Cloud SQL**

### The 4-Agent System

| Agent | Role | API Used | When |
|-------|------|----------|------|
| **Agent 1: Recruiter Assistant** | Auto-generates custom technical questions from job title | Standard Gemini 2.5 Flash (one-shot `generateContent`) | Interview creation (`POST /api/interviews`) |
| **Agent 2: Interviewer & Proctor** | The "Face". Conducts the conversational interview via voice and strictly monitors the webcam video for proctoring. It does NOT process UI interactions directly. | Gemini 2.5 Flash **Multimodal Live API** | Continuous (WSS Audio/Video only) |
| **Agent 3: Smart Workspace Agent** | The "Engine". Owns all UI actions. Handles AI Copilot typing, executes code via Judge0, and parses Whiteboard diagrams via Vision. It silently informs Agent 2 of the results. | Standard Gemini 2.5 Flash (REST) + Judge0 API | Triggered by UI events (typing, clicking Run) |
| **Agent 4: Assessor** | Takes full transcript + final code, generates structured JSON evaluation | Standard Gemini 2.5 **Pro** API with Structured Output (JSON Schema) | After interview ends (one-shot) |

---

## PHASE 1 — Project Foundation & Auth System (Days 1–3: Feb 26–28)

**Checkpoint Deadline: Feb 28 EOD**

### Frontend Tasks

#### F1.1 — Electron Project Scaffold
- Initialize Electron app with `npm init` + `electron` dependency
- Set up project structure: `main.js` (main process), `preload.js` (context bridge), `renderer/` (pages, styles, scripts)
- Configure IPC bridge via `contextBridge.exposeInMainWorld` for auth channels (login, signup, getToken, logout, onTokenExpired)

#### F1.2 — Login & Registration UI
- **Admin/Recruiter** signup: Email + Password form
- **Recruiter** login: Email + Password form
- **Candidate** entry screen with two buttons: `Enter Interview Code` | `Practice Interview`
- Store JWT in Electron's `safeStorage` (encrypted OS keychain)
- On every app launch, call `GET /api/auth/me` with the stored token — let the **backend** definitively confirm validity. Do NOT rely on frontend-only decode checks. If backend returns `401` → clear token, show login

#### F1.3 — JWT Handling in Electron
- Install `jsonwebtoken` for token decode (read-only, verification happens server-side)
- On login success: store token via IPC to main process → `safeStorage.encryptString(token)`
- Attach token as `Authorization: Bearer <token>` header on every HTTP request
- If any API returns `401`, clear token, redirect to login screen

---

### Backend Tasks (Spring Boot — Cloud)

#### B1.1 — Java Project Scaffold
- Java 17+ with Gradle or Maven
- Dependencies: `spring-boot-starter-web`, `spring-boot-starter-security`, `jjwt`, `spring-boot-starter-data-jpa`, `postgresql` driver, `com.google.adk:google-adk:0.5.0`, `spring-boot-starter-websocket`
- Structure: `config/`, `controller/`, `service/`, `model/`, `repository/`, `dto/`, `security/`, `gemini/` (ADK integration)

#### B1.2 — Database Schema (Cloud SQL PostgreSQL)

**Users table** — all roles share this. Roles: `ADMIN`, `RECRUITER`, `CANDIDATE`. Constraint: `CHECK (role IN ('ADMIN', 'RECRUITER', 'CANDIDATE'))`

**Workspaces table** — every account exists inside a Workspace. A lone recruiter is simply an ADMIN of a single-member Workspace. Fields: `id`, `name`, `logo_url`, `owner_id` (FK to users)

**Workspace members table** — links users to workspaces. Composite PK: `(workspace_id, user_id)`. Default role: `RECRUITER`

**Interviews table** — Fields: `id`, `workspace_id` (FK), `created_by` (FK), `title`, `access_code` (VARCHAR 6, unique), `duration_minutes` (default 45), `tools_enabled` (JSONB), `ai_instructions` (TEXT), `generated_questions` (TEXT — auto-generated by Agent 1), `status` (CHECK: `UPCOMING`, `ACTIVE`, `COMPLETED`)

**Interview reports table** — Fields: `id`, `interview_id` (FK), `candidate_email`, `score`, `behavioral_notes`, `code_output`, `behavior_flags` (JSONB), `human_feedback`

#### B1.3 — Auth REST Endpoints

| Method | Path | Body | Returns |
|--------|------|------|---------|
| POST | `/api/auth/signup` | `{email, password, role, fullName}` | `{token, user}` |
| POST | `/api/auth/login` | `{email, password}` | `{token, user}` |
| GET | `/api/auth/me` | – (Bearer token) | `{user}` |

- JWT payload: `{sub: userId, email, role, workspaceId, iat, exp}`. Expiry: 24 hours
- Password hashing: BCrypt with strength 12. JWT secret: env `JWT_SECRET`
- When a user signs up: automatically create a Workspace, assign them as ADMIN + owner

#### B1.4 — JWT Security Filter
- Implement `JwtAuthenticationFilter extends OncePerRequestFilter`
- Extract token from `Authorization` header, validate signature + expiry, set `SecurityContext`
- Public endpoints: `/api/auth/signup`, `/api/auth/login`, `/api/health`

---

### ✅ Phase 1 Checkpoint

| # | Check | Pass? |
|---|-------|-------|
| 1 | Electron app starts, shows login screen | ☐ |
| 2 | Recruiter can sign up with email + password | ☐ |
| 3 | Login returns JWT, app navigates to dashboard | ☐ |
| 4 | Opening app without valid token stays on login | ☐ |
| 5 | Backend rejects requests without valid JWT (401) | ☐ |
| 6 | Candidate screen shows "Enter Code" and "Practice" buttons | ☐ |

---

## PHASE 2 — Staff Dashboards & Interview Setup (Days 4–6: Mar 1–3)

**Checkpoint Deadline: Mar 3 EOD**

### The Workspace Concept (Lone Recruiter vs. Team)

Every account exists inside a **Workspace**.

- **Lone Recruiter**: Signs up, becomes ADMIN of a single-member Workspace. Has access to everything — Workspace Settings + Interview Dashboard.
- **Team**: ADMIN creates Workspace and invites multiple RECRUITER users. Recruiters only see the Interview Dashboard, not team management settings.

The backend handles both seamlessly — no separate "freelancer" features. An ADMIN is simply a Recruiter who also has access to Workspace Settings.

### Dashboard Routing Logic

```
[ STAFF LOGS IN ] → [ CHECK JWT ROLE ]
  → IF ADMIN → [ WORKSPACE SETTINGS & TEAM MANAGEMENT ] + [ INTERVIEW DASHBOARD ]
  → IF RECRUITER → [ INTERVIEW DASHBOARD only ]

[ INTERVIEW DASHBOARD ] → [ CREATE NEW INTERVIEW ] → [ GENERATE 6-DIGIT CODE ]
```

---

### Frontend Tasks

#### F2.1 — Dashboard Role Router
- After login, read `role` from the JWT payload
- If `ADMIN`: show sidebar with "Workspace Settings" + "Interviews"
- If `RECRUITER`: show sidebar with "Interviews" only
- Both roles land on the Interview Dashboard by default

#### F2.2 — Workspace Settings Page (Admin Only)
- **Company Profile**: form for Company Name, Logo upload. Action: `PUT /api/workspace`
- **Invite Team Member**: email input. Action: `POST /api/workspace/invite`. Show "Invitation sent" on success
- **Manage Team**: fetch list via `GET /api/workspace/members`. Display each with "Revoke Access" button (`DELETE /api/workspace/members/:userId`)

#### F2.3 — Interview Dashboard (Admin & Recruiter)
- Fetch **all** interviews for the Workspace via `GET /api/interviews`
- Table columns: Title, Access Code, Status, Duration, Created By, Date
- Filter tabs: **Upcoming** | **Active** | **Completed**
- Poll every 10 seconds. Each row clickable → interview detail / monitoring view

#### F2.4 — Create Interview Panel
- Input fields: Interview Title (required), Duration dropdown (30/45/60/90 min, required), Allowed Tools checkboxes (Code Editor, Drawing Board, Notes), AI Instructions textarea (optional)
- Action: `POST /api/interviews` → receive 6-digit access code + auto-generated questions → display code in modal with Copy button

#### F2.5 — Interview Monitoring View (Placeholder)
- Page skeleton for: interview title + status badge, candidate indicator, live AI feed area, warnings/flags area, AI audio player area
- Wire up WebSocket placeholder

---

### Backend Tasks (Spring Boot — Cloud)

#### B2.1 — Workspace API (Admin Only)
All endpoints require JWT role = `ADMIN`. Return `403 Forbidden` for RECRUITER.

| Method | Path | Body | Returns |
|--------|------|------|---------|
| GET | `/api/workspace` | – | `{workspace, memberCount}` |
| PUT | `/api/workspace` | `{name, logoUrl}` | `{workspace}` |
| POST | `/api/workspace/invite` | `{email}` | `{success, message}` |
| GET | `/api/workspace/members` | – | `[{user}]` |
| DELETE | `/api/workspace/members/:userId` | – | `{success}` |

**Invite Logic**: Verify ADMIN role → check email doesn't exist → create RECRUITER user linked to workspace → generate password-setup token → trigger email with setup link

#### B2.2 — Interviews API (Admin & Recruiter)

| Method | Path | Body | Returns |
|--------|------|------|---------|
| GET | `/api/interviews` | – | `[{interview}]` (all for workspace) |
| GET | `/api/interviews/:id` | – | `{interview, report?}` |
| POST | `/api/interviews` | `{title, duration, tools, aiInstructions}` | `{interview, accessCode, generatedQuestions}` |
| PUT | `/api/interviews/{code}/status` | `{status}` | `{interview}` |
| POST | `/api/interviews/validate-code` | `{code}` | `{interviewId, valid, config}` |

**Code Generation**: Random 6-digit numeric code via `SecureRandom`. Check DB for collisions with active interviews. Regenerate if collision.

**Interview Fetching**: Scope by `workspaceId` from JWT. Return ALL workspace interviews for team collaboration.

**Agent 1 — Recruiter Assistant**: When `POST /api/interviews` is called, Java uses a standard Gemini 2.5 Flash `generateContent` API call to auto-generate custom technical questions based on the job title and any provided `aiInstructions`. The generated questions are saved to the `generated_questions` field in the DB and included in the system instructions when the Live interview session starts.

#### B2.3 — Interview Report Endpoints

| Method | Path | Body | Returns |
|--------|------|------|---------|
| GET | `/api/reports/:interviewId` | – | `{report}` |
| POST | `/api/reports/:interviewId/feedback` | `{humanFeedback, approved}` | `{report}` |

---

### ✅ Phase 2 Checkpoint

| # | Check | Pass? |
|---|-------|-------|
| 1 | Admin sees Workspace Settings + Interviews; Recruiter sees Interviews only | ☐ |
| 2 | Admin can update workspace name/logo | ☐ |
| 3 | Admin invites a Recruiter by email → new account created | ☐ |
| 4 | Admin can revoke a Recruiter's access | ☐ |
| 5 | Both roles can create an interview and receive a 6-digit access code | ☐ |
| 6 | Interview creation auto-generates technical questions via Gemini (Agent 1) | ☐ |
| 7 | Interview list shows ALL workspace interviews (team-wide visibility) | ☐ |
| 8 | Filter tabs (Upcoming/Active/Completed) work correctly | ☐ |
| 9 | 6-digit code is unique — no collisions with active interviews | ☐ |
| 10 | Monitoring page skeleton loads (placeholders OK) | ☐ |

---

## PHASE 3 — Candidate Experience & Pre-Interview (Days 7–9: Mar 4–6)

**Checkpoint Deadline: Mar 6 EOD**

> Whether the candidate enters a 6-digit code for a real interview, or launches Practice Mode, the execution logic is exactly the same. The only difference is where the final report goes.

### Interview Flow

```
[ ENTER 6-DIGIT CODE ] (or "Practice Mode")
         ↓
[ PRE-FLIGHT LOBBY ] → Check Camera, Mic, Network
         ↓
[ INITIATE LOCKDOWN ] → Kiosk Mode, Block Shortcuts, DRM Content Protection
         ↓
[ CONNECT TO CLOUD ] → Electron opens WSS to Java Cloud Server
         ↓
[ LIVE INTERVIEW ] → AI talks, User codes in Monaco, AI monitors
```

---

### Frontend Tasks

#### F3.1 — Candidate Code Entry Screen
- Input field for 6-digit code (numeric only)
- On submit: call `POST /api/interviews/validate-code` with `{code}`
- If invalid → show error "Invalid access code"
- If valid → navigate to Pre-Flight Lobby

#### F3.2 — Practice Interview Entry
- "Practice Interview" bypasses code validation
- Same workspace UI, same AI interview, but with "PRACTICE MODE" banner
- No report saved at end

#### F3.3 — Pre-Flight Lobby
- **Camera preview**: display live `<video>` so candidate can adjust lighting/position
- **System checks** — run sequentially with real-time status indicators (✅ / ❌):
  - ✅ Camera Working — request `getUserMedia({ video: true })`, show preview
  - ✅ Microphone Working — request `getUserMedia({ audio: true })`, verify audio level above threshold
  - ✅ Internet Stable — `fetch('/api/health')`, verify latency < 2 seconds
- All checks pass → enable **"I am ready — Start Interview"** button
- Do NOT auto-start. Candidate must click manually.

#### F3.4 — Lockdown Execution
On "Start Interview" click:
- Set `BrowserWindow` to fullscreen, kiosk, alwaysOnTop (screen-saver level), non-closable, non-minimizable
- Register `globalShortcut` to intercept: Ctrl+Tab, Alt+Tab, Escape, Alt+F4, Ctrl+W, Ctrl+Q, Meta+Tab
- **OS-Level Screen Capture Blocking**: trigger `win.setContentProtection(true)` — this uses the OS's native DRM (Display Affinity) to force any screen recording software (OBS, Discord, WebRTC) to capture a completely black screen. Zero process scanning required.
- **Environment breach detection**: listen for `blur` event on the window. If app loses focus, log an "ENVIRONMENT_BREACH" event, force window back to focus, and send breach warning to the cloud server

#### F3.5 — Connect to Cloud & Stream
After lockdown is active:
1. **Open WSS connection** to Java Cloud Server (e.g., `wss://api.yourdomain.com/stream`) and begin streaming media
2. Authenticate the WSS connection using the JWT token

#### F3.6 — Media Capture & Streaming
- Capture webcam at 1fps as Base64 JPEG (640×480, quality 0.7)
- Capture microphone at 16kHz mono PCM, convert Float32 → Int16
- Stream both continuously over the WSS connection to the **Java Cloud Server**
- Listen on the same WSS for incoming audio from Gemini (via Java) and play it through speakers at 24kHz

#### F3.7 — Downstream Event Handler
- When the Java server sends Gemini's voice audio → play it through speakers
- When the Java server sends text → append to transcript sidebar
- When the Java server sends proctor alerts → show full-width red warning banner with shake animation, log in behavior log

---

### Backend Tasks (Spring Boot — Cloud)

#### B3.1 — Interview State Tracking
- `PUT /api/interviews/{code}/status` — when candidate clicks "Start", Electron calls this to set status from `UPCOMING` to `ACTIVE`. This prevents the 6-digit code from being used on another machine.

#### B3.2 — WebSocket Endpoint for Electron Media Stream
- Expose a secure WebSocket endpoint (e.g., `/stream`) that accepts incoming media from Electron
- Authenticate the connection using the JWT token (passed as query param or first message)
- Receive Base64 JPEG frames and PCM audio chunks
- Forward them directly into the Gemini 2.5 Live API session via the ADK (Phase 5 wires this up)

#### B3.3 — Health Check Endpoint
- `GET /api/health` → returns `{status: "ok", timestamp}` (no auth required)

---

### ✅ Phase 3 Checkpoint

| # | Check | Pass? |
|---|-------|-------|
| 1 | Candidate enters valid code → proceeds to Pre-Flight Lobby | ☐ |
| 2 | Invalid code shows error, blocks entry | ☐ |
| 3 | Camera preview shows in lobby, all 3 system checks run | ☐ |
| 4 | Failed check disables "Start Interview" button | ☐ |
| 5 | Kiosk mode activates: fullscreen, shortcuts blocked, always on top | ☐ |
| 6 | Focus loss triggers environment breach warning | ☐ |
| 7 | Electron opens WSS to Java cloud and begins streaming media | ☐ |
| 8 | Spring Boot marks interview as ACTIVE when started | ☐ |
| 9 | Practice mode works without a code, same workflow | ☐ |

---

## PHASE 4 — Interview Workspace UI (Days 10–12: Mar 7–9)

**Checkpoint Deadline: Mar 9 EOD**

### Frontend Tasks

#### F4.1 — Interview Workspace Layout
Full-screen workspace with panels:
- **Header Bar**: Timer (countdown from session duration, warns at 5 min and 1 min), Status badge, "End Interview" button
- **Main Area**: Tabbed interface — Code (Monaco Editor), Whiteboard (HTML5 Canvas with pen/eraser/colors), Notes (auto-saving textarea)
- **Right Sidebar**: Small camera preview (160×120, bottom-right), AI Voice Indicator (pulsing dot when AI speaks), Transcript area

#### F4.2 — Monaco Editor Setup
- Install `monaco-editor` npm package
- Default language: Java. Support switching to Python, JavaScript
- The candidate writes code here
- Add a **"Run / Review Workspace"** button. On click, Electron packages the current Monaco editor text, the Notes text, and the Whiteboard (as a Base64 image) and sends it to the Java Cloud Server via WSS
- Implement `registerInlineCompletionsProvider` for AI-assisted code completion: when the user stops typing for 1.5 seconds, call `POST /api/copilot` to fetch "ghost text" predictions and display them inline

#### F4.3 — AI Voice Playback
- Queue incoming PCM audio chunks to avoid overlap
- Show pulsing animation when AI is speaking
- Show transcript of AI speech in sidebar

#### F4.4 — Proctor Warning UI
- When Java sends proctor alerts via WSS: display full-width red banner ("⚠️ Warning: Please focus on your interview"), brief shake animation, log the warning in a hidden behavior log

---

### Backend Tasks (Spring Boot — Cloud)

#### B4.1 — Copilot Endpoint
- `POST /api/copilot` — receives `{code, language, cursorPosition}`, uses a standard Gemini 2.5 Flash `generateContent` call to generate inline code completion suggestions, returns `{suggestion}`

---

### ✅ Phase 4 Checkpoint

| # | Check | Pass? |
|---|-------|-------|
| 1 | Workspace UI renders with Code, Whiteboard, Notes tabs | ☐ |
| 2 | Camera preview shows in corner | ☐ |
| 3 | Monaco editor loads and accepts code input | ☐ |
| 4 | "Run / Review Workspace" button packages code + notes + whiteboard and sends to Java | ☐ |
| 5 | Copilot ghost text appears after 1.5s typing pause | ☐ |
| 6 | Timer counts down correctly, warns at 5 min and 1 min | ☐ |
| 7 | AI audio playback works with pulsing indicator | ☐ |
| 8 | Proctor warning banner displays when alert received | ☐ |

---

## PHASE 5 — Gemini Live API & AI Intelligence (Days 13–15: Mar 10–12)

**Checkpoint Deadline: Mar 12 EOD**

> This is the core of the product. The Java Cloud Server becomes the AI orchestrator using a 4-Agent system.

### Backend Tasks (Spring Boot — Cloud, Primary Focus)

#### B5.1 — Gemini Live API Connection via ADK (Agent 2: Interviewer & Proctor)
- Use the `google-adk` Java SDK (`com.google.adk:google-adk:0.5.0`) to open a secure bidirectional WebSocket to Gemini 2.5 Flash Live API
- Authenticate using `GOOGLE_API_KEY` from server environment variables — **never exposed to the client**
- Configure the `SessionConfig` with system instructions and tool declarations (`write_cloud_sql_report`)

#### B5.2 — System Instructions (Secret — Cloud Only)
Define the AI interviewer personality. This is stored **only on the Java server** and never sent to clients:
- Greet the candidate warmly, introduce yourself as Owlyn
- Ask technical questions based on the `ai_instructions` and `generated_questions` fields from the interview config
- **CRITICAL PROCTORING RULE**: You are receiving a live 1fps video feed of the candidate's webcam. If you visually detect a smartphone, another person in the room, or the candidate looking away from the screen for more than 10 seconds, immediately pause the technical interview and strictly warn them.
- When the interview is done or time runs out, use `write_cloud_sql_report` to save the evaluation
- Tone: Professional but encouraging. Like a senior engineer.

#### B5.3 — RunConfig Setup
Based on **Level 4 Codelab Section 5**:
- `streamingMode`: BIDI
- `responseModalities`: AUDIO, TEXT
- `inputAudioTranscription`: enabled
- `outputAudioTranscription`: enabled
- `sessionResumption`: enabled (in case of session timeout — Gemini Live has duration caps)
- `proactiveAudio`: **true** — this is critical for proctoring. Without it, Gemini waits for the candidate to speak. With it, Gemini can proactively warn about phone use or looking away.

#### B5.4 — Media Routing Loop
The core routing logic inside the Java server:
- **Upstream**: receive Base64 JPEG frames and PCM audio from Electron's WSS → decode → feed into Gemini via the ADK's `LiveRequestQueue`
- **Downstream**: receive Gemini's audio/text responses → forward them to Electron via WSS as JSON events
- **Tool calls**: receive `FunctionCall` events from Gemini → route to the appropriate handler

#### B5.5 — Smart Workspace Handler (Agent 3)
Agent 3 owns all UI events coming from Electron.

1. **Auto-Smart (Copilot):** When the candidate types, Agent 3 silently fetches ghost-text via REST and sends it back to the UI. Agent 2 is not involved.
2. **Workspace Review:** When the candidate clicks "Run / Review Workspace", Agent 3 executes the code in **Judge0** and parses the Whiteboard via **Gemini Flash Vision**.
3. **The Handoff:** Agent 3 compiles these facts and injects a hidden text prompt into Agent 2's Live stream: `{"text": "SYSTEM ALERT FROM WORKSPACE AGENT: The candidate just ran their code. Execution output: [STDOUT]. Whiteboard: [VISION SUMMARY]. Acknowledge this and ask them about the time complexity."}`
4. Agent 2 reads this hidden message and speaks to the candidate out loud.

This guarantees strict **separation of concerns**: Agent 2 only handles voice + webcam, Agent 3 owns the entire workspace UI. If you swap Monaco for a different editor later, Agent 2 is untouched.

#### B5.6 — Post-Interview Assessment (Agent 4: Assessor)
When the interview ends (candidate says "I'm done" or timer runs out):
1. Java collects the full transcript and final code from the session
2. Java sends **one prompt** to the standard **Gemini 2.5 Pro API** using **Structured Output (JSON Schema)** to generate: `{score: int, behavioral_notes: string, code_quality: string, communication_rating: string}`
3. Java writes this JSON directly to Cloud SQL `interview_reports` table
4. Java sends the report down to Electron to display the "Interview Complete" summary

#### B5.7 — Tool Call Dispatch
When Gemini triggers `write_cloud_sql_report`: extract `candidate_email`, `score`, `behavioral_notes` → write directly to Cloud SQL interview_reports table → send confirmation back to Gemini

---

### Frontend Tasks

#### F5.1 — Wire Up Media-to-Cloud Pipeline
- Ensure the media capture from Phase 3 (F3.6) correctly streams to the Java Cloud Server
- Ensure downstream events from Java are correctly parsed and routed to the UI (audio playback, transcript, proctor alerts)

---

### ✅ Phase 5 Checkpoint

| # | Check | Pass? |
|---|-------|-------|
| 1 | Java connects to Gemini Live API via ADK WSS | ☐ |
| 2 | Sending video frame + audio from Electron → Java → Gemini → Gemini responds with voice | ☐ |
| 3 | Gemini's audio plays through Electron speakers | ☐ |
| 4 | Proctoring works natively: Gemini warns when phone detected or candidate looks away | ☐ |
| 5 | Proctor alert appears as red banner in Electron UI | ☐ |
| 6 | "Run / Review Workspace" → Agent 3 executes code in Sandbox + parses Whiteboard → Gemini gives factual feedback | ☐ |
| 7 | Post-interview: Assessor generates structured JSON report via Gemini Pro | ☐ |

---

## PHASE 6 — Full Integration, Testing & Demo Prep (Days 16–19: Mar 13–16)

**Checkpoint Deadline: Mar 16 EOD (FINAL)**

### Full Team Tasks

#### I6.1 — End-to-End Integration Loop
Test this exact sequence. Every step must work without manual intervention:

1. **Admin/Recruiter** creates interview → Agent 1 auto-generates questions → gets code `492104`
2. **Candidate** opens Electron → enters code → passes system check
3. Kiosk activates → workspace loads
4. Electron streams A/V to Java Cloud → Java pipes to Gemini
5. Gemini greets: *"Hello, I'm Owlyn. Let's begin your technical interview."*
6. Gemini asks a coding question (from auto-generated questions)
7. Candidate writes code in Monaco editor
8. Candidate clicks **"Run / Review Workspace"** → Agent 3 executes code in Sandbox API and processes Whiteboard → Injects facts into stream → Gemini speaks feedback with 100% factual confidence
9. Gemini says: *"Your code compiles and passes. Well done."*
10. **Proctor test**: hold up a phone → Gemini warns within 10 seconds
11. Candidate says: *"I'm done"*
12. Agent 4 (Assessor) generates structured JSON report via Gemini Pro → written to Cloud SQL
13. Workspace unlocks → candidate sees "Interview Complete"
14. **Recruiter** views dashboard → report appears with score + notes

#### I6.2 — Specific Test Scenarios

| Test | Action | Expected Result |
|------|--------|-----------------|
| Access Control | Open app without JWT | Stays on login screen |
| Invalid Code | Enter wrong 6-digit code | Error: "Invalid code" |
| Proctoring | Look at phone for 6s | Gemini says: "Please put your phone away" |
| Code Execution | Write code and click "Run / Review Workspace" | Sandbox executes code, Gemini speaks factual feedback |
| Whiteboard Vision | Draw a system diagram and click "Run / Review Workspace" | Gemini describes the diagram and gives design feedback |
| DB Logging | Say "I am done" | Cloud SQL shows new structured JSON report row |
| Lockdown | Press Alt+Tab during interview | Nothing happens (blocked) |
| DRM | Try to screen-record with OBS | OBS captures black screen |
| Focus Breach | System popup steals focus | App regains focus, breach logged |
| Practice Mode | Use "Practice Interview" | Works without code, no report saved |

#### I6.3 — Recruiter/Admin Monitoring Dashboard (Wire Up)
- Replace Phase 2 placeholders with real data
- WebSocket connection to backend for real-time interview status
- Display behavior flags as they come in
- Show final report when interview completes

#### I6.4 — Recruiter Flow (Wire Up)
- Recruiters login → see all Workspace interviews
- Monitor view: candidate camera feed (relayed), code view, flags
- Add human feedback form on the report page
- Submit button finalizes the report

#### I6.5 — Demo Preparation
- Prepare a clean demo candidate account
- Pre-write a correct + intentionally buggy code solution
- Test phone-detection proctoring 3 times for reliability
- Open Google Cloud Console → Cloud SQL → Prepare query to show inserted report
- Rehearse the full demo flow at least twice

#### I6.6 — Pitch Script

**The Hook:**
*"We built a fully isolated, hardware-accelerated technical interview environment that utilizes Gemini 2.5 Multimodal Live API as a native, real-time proctor and conversational interviewer."*

**The Google Architecture Flex:**
*"We didn't hack together random APIs. This is a pure Google ecosystem showcase. We use JWT for secure entry. We use the google-adk Java SDK in the cloud to stream raw 1fps video and PCM audio directly to Gemini so it can see and hear the candidate natively. There are no clunky computer vision scripts here — Gemini is the vision."*

**The 4-Agent Orchestration Flex:**
*"Instead of building a clunky virtual machine sandbox, we treated Gemini 2.5 like a real Senior Staff Engineer. We built a 4-Agent Orchestrated system. We use a single Multimodal Live API stream as our real-time Interviewer AND Proctor, eliminating token waste. To prevent code hallucinations, we decoupled the conversation from the evaluation using our Smart Assist Agent. It executes the candidate's code in an isolated Sandbox API and processes visual whiteboard diagrams, feeding undeniable, factual results directly into the Live API so the AI speaks with 100% confidence. Finally, an asynchronous Gemini Pro Assessor agent generates a structured JSON evaluation directly into Google Cloud SQL."*

**The Live Demo Flow (5 Steps):**
1. Launch Electron, authenticate via JWT.
2. Show the locked workspace.
3. Pull out a phone on stage — let the judges hear Gemini native-voice scold you in real-time.
4. Click "Run / Review Workspace" — show the Sandbox API executing code and Gemini giving factual feedback.
5. End the interview, open Google Cloud Console, and show the freshly generated structured JSON report in Cloud SQL.

---

### ✅ Phase 6 FINAL Checkpoint

| # | Check | Pass? |
|---|-------|-------|
| 1 | Full loop (recruiter create → candidate interview → report) works E2E | ☐ |
| 2 | JWT blocks unauthorized access | ☐ |
| 3 | Proctoring detects phone and warns verbally (native Gemini) | ☐ |
| 4 | Smart Assist (Agent 3) executes code in Sandbox + parses Whiteboard → factual feedback | ☐ |
| 5 | Structured JSON report saved in Cloud SQL via Agent 4 (Gemini Pro) | ☐ |
| 6 | Kiosk mode cannot be bypassed | ☐ |
| 7 | DRM content protection blocks screen recording | ☐ |
| 8 | Practice mode works independently | ☐ |
| 9 | Admin/Recruiter dashboard shows live interview data | ☐ |
| 10 | Recruiter can add feedback and approve report | ☐ |
| 11 | Demo rehearsed successfully at least twice | ☐ |

---

## Quick Reference: Key Technical Resources

### 1. The Core Architecture Blueprint (Conceptual Python to Java Translation)

**URL:** https://github.com/google/adk-samples/tree/main/python/agents/bidi-demo

* **Target Audience:** Java Backend Devs
* **Why you need it:** Even though this example is written in Python (FastAPI), it contains the **exact architectural logic** your Java team needs to build. It demonstrates how to initialize the `RunConfig` with `StreamingMode.BIDI`, set up the `LiveRequestQueue`, and handle concurrent upstream (receiving from client) and downstream (sending to client) WebSocket tasks. Translate this logic directly into Spring Boot.

### 2. The Media Streaming Guide (Eyes & Ears)

**URL:** https://codelabs.developers.google.com/way-back-home-level-3/instructions#0

* **Target Audience:** Frontend (Electron) & Java Backend Devs
* **Why you need it:**
  * **Frontend:** Provides the exact JavaScript approach needed to capture video frames at 1 FPS, encode them to Base64, capture PCM audio, and format the JSON payloads to send over WebSockets (see the "Implement the WebSocket Hook" section).
  * **Backend:** Shows exactly how to parse those incoming JSON payloads and dump the `types.Blob` data into the ADK `LiveRequestQueue`.

### 3. The Orchestration & Proctoring Guide (The Brain)

**URL:** https://codelabs.developers.google.com/way-back-home-level-4/instructions#0

* **Target Audience:** Java Backend Devs
* **Why you need it:**
  * **Proctoring / Barge-in:** Shows how to configure `proactive_audio: true` inside the `RunConfig` so Gemini can interrupt the candidate naturally if it detects cheating.
  * **Code Injection:** Demonstrates how the `LiveRequestQueue` handles injected text (`ClientContent`). This is the exact mechanism you will use for the "Run / Review Workspace" feature — taking the Monaco editor string and pushing it upstream so Gemini can verbally analyze it.
  * **Downstream Parsing:** Details the exact JSON shape of the `serverContent.modelTurn.parts[]` so your backend knows how to extract Gemini's voice chunks and send them to Electron.

### 4. Structured Output for the Database (The Assessor)

**URL:** https://ai.google.dev/gemini-api/docs/structured-output

* **Target Audience:** Java Backend Devs
* **Why you need it:** After the Live WebSocket closes, Agent 4 (Gemini Pro) takes the transcript and generates the final evaluation. This documentation shows how to pass a JSON Schema into the API call to guarantee Gemini returns a perfectly formatted JSON object (`score`, `behavioral_notes`, `code_quality`) that maps exactly to your Google Cloud SQL `interview_reports` table.

### 5. Sandbox Code Execution (The Hands)

**URL:** https://judge0.com/ (or https://github.com/engineer-man/piston)

* **Target Audience:** Java Backend Devs
* **Why you need it:** Agent 3 (Smart Assist) needs a secure, isolated environment to execute the candidate's code. Judge0 provides a REST API that accepts source code + language, runs it in a sandboxed container, and returns literal `stdout`, `stderr`, and `exitCode`. This guarantees zero hallucinations — Gemini speaks feedback based on real execution results.
