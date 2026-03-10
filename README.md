
let's go win this hackathon!

***

# 🦉 OWLYN — The Master Architecture & Frontend Bible (LiveKit Edition)

> **Deadline**: Mar 16, 2026  
> **Core Stack**: Electron (Frontend), Java/Spring Boot (Control Plane), Python Worker (LiveKit AI Data Plane), Gemini 3.1 Pro & 3.0 Flash  
> **🚨 MAJOR ARCHITECTURE PIVOT:** We have completely abandoned custom WebSockets. We are now using **LiveKit** (True WebRTC) for flawless, ultra-low latency audio/video, and a Python worker to orchestrate the AI. 

---

## 🏛️ Architecture Overview — The Microservice Split

### The Roles
| Component | Role | Description |
|-----------|------|-------------|
| **Java Spring Boot** | *The Command Center* | Handles Auth, JWTs, Database (Postgres), AI Copilot (`/api/copilot`), generates LiveKit Room Tokens, and uses Agent 4 (Gemini 3.1 Pro) to generate final JSON ATS reports. |
| **LiveKit Cloud** | *The WebRTC Router* | Replaces our custom WebSockets. Handles ultra-low latency routing of the candidate's audio, video, screen-share tracks, and JSON data payloads. |
| **Python Worker** | *The AI Data Plane* | Connects silently to the LiveKit room. Uses `livekit-agents` to run Agent 2 (Voice) and Agent 3 (The Dual Sentinels). |
| **Electron App** | *The Senses* | Uses `@livekit/components-react` to publish the candidate's microphone, a webcam track, and a screen-share track. |

---

## 💡 CRITICAL: The "Visual Compiler" (How the AI actually works)
**FRONTEND TEAM, PLEASE READ THIS CAREFULLY:** 
The AI does **NOT** run or execute the candidate's Java/Python code in a backend sandbox. Do not build remote execution environments. 

The AI acts as a **"Visual Compiler"**. 
Every second, our Python worker takes the 1FPS screen-share video track you published via LiveKit. It feeds that image to Gemini 3.0 Flash Vision. Gemini physically *reads the code off the image* and uses its massive LLM reasoning to mentally trace the logic and find bugs. Simultaneously, a separate Sentinel watches the Webcam track to catch cheating.

---

## ⚡ The LiveKit Frontend Cheat Sheet
*Because we dropped WebSockets, here is exactly how you handle real-time data in React/Electron.*

**1. Connecting to the Room**
When you call `POST /api/interviews/validate-code`, Java will return a `livekitToken`. Use the `@livekit/components-react` package to wrap your interview workspace in a `<LiveKitRoom token={livekitToken} serverUrl="wss://your-livekit-url">`.

**2. Separated Media Capture (Video, Screen & Audio)**
You must publish **THREE** separate tracks to LiveKit simultaneously. Do not combine them!
*   **AudioTrack:** The candidate's microphone.
*   **CameraTrack:** A standard webcam feed (Face only). Configure this to **1 Frame Per Second (1 FPS)** to save bandwidth. The AI uses this for Proctoring.
*   **ScreenShareTrack:** Capture the Monaco editor/whiteboard (or their whole desktop in Tutor mode). Configure this to **1 FPS** as well. The AI uses this to read the code.

**3. Sending Commands to the AI (DataChannels)**
To instantly wake up the AI when the user clicks "Run Code", do not use REST APIs. Use LiveKit's `useLocalParticipant()` to publish a message to the Data Channel:
*   *Frontend Sends:* `{"event": "RUN_CODE"}` (Encode as byte array/string). The Python worker will catch this and instantly analyze the code on screen.

**4. Receiving Commands from the AI (DataChannels)**
Listen to the room's Data Channel (`RoomEvent.DataReceived`). The Python AI will send JSON commands to control your UI:
*   *If Payload is:* `{"type": "PROCTOR_WARNING", "message": "Put your phone away"}` ➡️ **Action:** Shake the UI red and show a toast.
*   *If Payload is:* `{"type": "TOOL_HIGHLIGHT", "line": 14}` ➡️ **Action:** Tell Monaco Editor to highlight line 14.

**5. AI Voice Playback**
You don't have to queue base64 audio chunks anymore! Just render LiveKit's `<AudioTrack>` or `<RoomAudioRenderer />` component, and when the AI speaks, LiveKit plays it natively with zero latency.

---

## 🚀 PHASE-BY-PHASE IMPLEMENTATION GUIDE

### PHASE 3 — Candidate Gateway & Lockdown
*   **F3.1 — Candidate Code Entry:** Input 6-digit code → Call `POST /api/interviews/validate-code` → Save `token` (Guest JWT) and `livekitToken`.
*   **F3.3 — Pre-Flight Lobby:** Test Mic, Camera, and Network.
*   **F3.4 — Lockdown:** On "Start Interview", trigger `win.setContentProtection(true)` to block OBS/Screen Recorders. Block `Alt+Tab`. Go Fullscreen Kiosk. Call Java `PUT /api/interviews/{code}/status/active` to lock the DB.
*   **F3.5 — Connect:** Join the LiveKit Room and publish the 3 tracks.

### PHASE 4 — Interview Workspace UI
*   **F4.1 — Layout:** Timer, Monaco Editor, Whiteboard, LiveKit Audio Visualizer (to show when AI is speaking).
*   **F4.2 — AI Copilot:** When the user stops typing for 1.5 seconds, grab the code text and cursor position. Call `POST /api/copilot` (using the Guest JWT). Display the returned `suggestion` as ghost text.

### PHASE 5 — The Python AI Worker (Backend Only)
*(Handled entirely by the Backend Team. The Python worker connects to LiveKit, runs the Dual Sentinels, injects warnings into the Voice AI, and sends the final transcript to Java when the room closes).*

### PHASE 6 — ATS Updates & Recruiter God-View
*   **I6.1 — The Recruiter Monitor:** When Amina clicks "Watch Live", call `GET /api/interviews/{id}/monitor-token`. Connect to LiveKit using this token. It grants "Subscriber Only" access so the recruiter can watch the candidate's screen-share and hear the audio natively without being seen!
*   **I6.2 — Top Performer Card:** Call `GET /api/reports/top` to get the highest scoring candidate instantly.
*   **I6.3 — HIRE/DECLINE Toggle:** Call `POST /api/reports/{id}/feedback` and pass `{"decision": "HIRE"}` to sync state with the database.

### PHASE 7 — Stretch Goals (Tutor Mode & Practice)
*   **Tutor Mode Difference:** Call `POST /api/public/sessions/tutor`. **DO NOT lock down the OS.** Minimize Owlyn to a small widget. Use Electron's `desktopCapturer` to stream the candidate's *entire monitor* so the AI can see their personal VS Code or PDF. The Python AI automatically turns off the Proctor and becomes a friendly tutor.
*   **Ephemeral Reports:** For Practice/Tutor modes, call `GET /api/public/reports/{id}` after the LiveKit room closes. Save the JSON response to `localStorage` because the backend will delete it after 15 minutes!

---

## 📚 THE MASTER REST API REFERENCE

### 🔐 Auth & Security (Public)
*   `POST /api/auth/signup` & `POST /api/auth/login` ➡️ Send OTP.
*   `POST /api/auth/verify-login?otp=...&email=...` ➡️ Returns `{ "token": "..." }`.
*   `GET /api/auth/me` ➡️ Session check.

### 🏢 Workspaces & ATS Dashboards (Admin/Recruiter)
*   `GET /api/workspace/members` ➡️ List all recruiters.
*   `POST /api/workspace/invite` ➡️ Returns a temporary password for the new recruiter in the JSON response!
*   `GET /api/reports` ➡️ The Talent Pool. Returns all completed interview reports.
*   `GET /api/reports/top` ➡️ Returns the single highest-scoring report.
*   `POST /api/reports/{id}/feedback` ➡️ Body: `{"humanFeedback": "...", "decision": "HIRE"}`. (Decision can be HIRE, DECLINE, or PENDING).

### 🤖 AI Personas (Admin/Recruiter)
*   `GET /api/personas` ➡️ List saved personas.
*   `DELETE /api/personas/{id}` ➡️ Delete persona. (Will return 400 Bad Request if attached to an interview).
*   `POST /api/personas` ➡️ Use `FormData`. Append JSON to `"persona"` and file (PDF/DOCX) to `"file"`.

### 🎙️ The Interview Setup (Admin/Recruiter)
*   `POST /api/interviews/generate-questions` ➡️ Ask Gemini to draft questions.
*   `POST /api/interviews` ➡️ Body: `{"personaId": "...", "generatedQuestions": "..."}`. Returns the 6-digit `accessCode`.
*   `GET /api/interviews/{id}/monitor-token` ➡️ Returns the `livekitToken` so the Recruiter can watch the Live God-View.

### 🚪 Candidate Gateway (Public)
*   `POST /api/interviews/validate-code` ➡️ Returns `token` (Guest JWT) and `livekitToken` (WebRTC).
*   `PUT /api/interviews/{code}/status/active` ➡️ Locks the room.
*   `POST /api/copilot` ➡️ Generates Monaco autocomplete ghost text.

### 🎓 B2C Educational Modes (Public)
*   `POST /api/public/sessions/practice` ➡️ Auto-generates mock interview, returns LiveKit tokens.
*   `POST /api/public/sessions/tutor` ➡️ Starts Homework Helper mode, returns LiveKit tokens.
*   `GET /api/public/reports/{id}` ➡️ Fetches Ephemeral JSON Scorecard (Store in localStorage!).























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
Electron App (A/V + Workspace UI) ─── WSS ──→ Java Cloud Server

[ GOOGLE CLOUD PLATFORM (The Multi-Agent Hub) ]
Java Cloud Server (ADK) ─── WSS (Stream A) ──→ Agent 2: Gemini Live API (Proctor/Interviewer)
Java Cloud Server (ADK) ─── WSS (Stream B) ──→ Agent 3: Gemini Live API (Workspace/Compiler)
Java Cloud Server (ADK) ─── SQL ──→ Google Cloud SQL
```

*Note: Agent 3 handles all code compilation using Gemini's Native Code Execution tool. Agent 3 directly passes its evaluation results to Agent 2's context queue inside the Java Server via the ADK.*

### The Configurable Workspace Tools

Recruiters configure the workspace per interview (e.g., Algorithms vs. System Design). The tools available to the candidate are:

| Tool | Type | Description |
|------|------|-------------|
| **Code Editor + Runner** | Optional | Monaco editor integrated with Gemini's Native Code Execution tool (via Agent 3's Live API stream) for real code compilation and evaluation |
| **Whiteboard** | Optional | HTML5 Canvas for architecture diagrams, parsed via Gemini Flash Vision |
| **Notes** | Optional | Plain text scratchpad |
| **Camera/Mic** | Mandatory | Always-on for proctoring |
| **AI Interviewer** | Mandatory | Voice interface (Gemini Live) |

### The Interview Loop

1. Electron captures user's camera (1fps JPEG) + mic (16kHz PCM)
2. Electron streams raw media up to **Java Cloud Server** via secure WebSocket
3. Java pipes the media into **Gemini 2.5 Live API** via the ADK
4. Gemini responds with voice → Java sends audio back down to Electron → candidate hears the AI
5. Candidate clicks **"Run / Review Workspace"** → Java pushes code to **Agent 3's Live API stream** → Agent 3 natively executes the code via Gemini's Code Execution tool and parses Whiteboard via Vision → Java catches Agent 3's output and injects it into **Agent 2's Live stream** via the ADK → Agent 2 speaks feedback based on verified facts, preventing hallucinations
6. Gemini dictates the final report → Java writes it directly to **Cloud SQL**

### The 4-Agent System

| Agent | Role | API Used | When |
|-------|------|----------|------|
| **Agent 1: Recruiter Assistant** | Auto-generates custom technical questions from job title | Standard Gemini 2.5 Flash (one-shot `generateContent`) | Interview creation (`POST /api/interviews`) |
| **Agent 2: Interviewer & Proctor** | The "Face". Conducts the conversational interview via voice and strictly monitors the webcam video for proctoring. It does NOT process UI interactions directly. | Gemini 2.5 Flash **Multimodal Live API** | Continuous (WSS Audio/Video only) |
| **Agent 3: Smart Workspace Agent** | The "Engine". Owns the Code and Whiteboard. It runs concurrently with Agent 2. When the candidate codes, Agent 3 uses Gemini's Native Code Execution Tool to compile and evaluate the logic. It uses Vision for the whiteboard. It then communicates its findings directly to Agent 2 via the ADK. | Gemini 2.5 Flash **Multimodal Live API** (with Code Execution tool enabled) | Continuous (WSS UI/Workspace stream) |
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







It is Monday evening, and you have exactly one week left. Pivoting to LiveKit + Python for the live session is the ultimate strategic move because it completely eliminates WebSocket latency issues and lets you use Google's native Python streaming patterns. 

To clear up the confusion with your frontend developer, we need to completely rewrite the execution plan to reflect the **Microservice Pivot (Java Control Plane + Python/LiveKit Data Plane)** and the **Visual Compiler Pivot (No execution sandboxes, just AI vision)**.

Here is the officially updated, copy-pasteable execution plan. Send this directly to your team so everyone is on the exact same page for the final 7 days.

***
