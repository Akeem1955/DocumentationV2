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

# OWLYN — Updated Hackathon Execution Plan (LiveKit + Python Pivot)

> **Deadline**: Mar 16, 2026  
> **Core Stack**: Electron (Frontend), Java/Spring Boot (Control Plane), Python Worker (LiveKit AI Data Plane), Gemini 3.1 Pro & 3.0 Flash  
> **Architecture Pivot**: We have abandoned custom WebSockets. We are now using **LiveKit** (True WebRTC) for flawless, ultra-low latency audio/video, and a Python worker to orchestrate the Live Gemini Agents.

---

## Architecture Overview — The Microservice Split

### The Roles
| Component | Role | Description |
|-----------|------|-------------|
| **Java Spring Boot** | *The Command Center* | Handles Auth, JWTs, Database (Postgres), AI Copilot (`/api/copilot`), generates LiveKit Room Tokens, and uses Agent 4 (Gemini 3.1 Pro) to generate final JSON reports. |
| **LiveKit Cloud** | *The WebRTC Router* | Replaces our WebSockets. Handles ultra-low latency routing of the candidate's audio and screen-share tracks. |
| **Python Worker** | *The AI Data Plane* | Connects to the LiveKit room. Uses `livekit-agents` and Google GenAI SDK to run Agent 2 (Voice) and Agent 3 (The Dual Sentinels). |
| **Electron App** | *The Senses* | Uses `@livekit/components-react` to publish the microphone and a **Unified Screen-Share** (recording the entire app window containing the face + code). |

---

## PHASE 3 — Candidate Experience & Pre-Interview

### Frontend Tasks (Updated for LiveKit)
**F3.1 — Candidate Code Entry Screen**
*   Input field for 6-digit code.
*   On submit: call `POST /api/interviews/validate-code` with `{code}`.
*   If valid → backend returns a Guest JWT **AND a LiveKit Access Token**.

**F3.2 — Practice Interview Entry**
*   Bypasses code validation. Calls `POST /api/public/sessions/practice`.
*   Backend generates a mock interview and returns the LiveKit token.

**F3.3 — Pre-Flight Lobby**
*   Check Camera & Mic.
*   Network Check: Ensure connection to LiveKit Cloud is stable.

**F3.4 — Lockdown Execution**
*   Fullscreen, kiosk mode, block `Alt+Tab`. OS-Level DRM `setContentProtection(true)` to block OBS/screen recorders.

**F3.5 — Connect to LiveKit (NO MORE WEBSOCKETS)**
*   Instead of opening a WSS to Java, use the `@livekit/components-react` SDK to connect to the LiveKit Room using the token received in F3.1.

**F3.6 — Unified Media Capture**
*   **CRITICAL CHANGE:** Do not capture the webcam and code separately. Use Electron's `desktopCapturer` to capture the **entire Owlyn app window** (Face on the left, code on the right).
*   Publish this video track (1fps) and the microphone audio track to the LiveKit room natively. 

### Backend Tasks (Java — Control Plane)
**B3.1 — LiveKit Token Generation**
*   Update `validate-code` endpoint. Use the `livekit-server-sdk-java` to generate a secure Room Token for the candidate. Return it alongside the Guest JWT.

**B3.2 — Status Lockdown**
*   `PUT /api/interviews/{code}/status/active` to lock the room in Postgres.

---

## PHASE 4 — Interview Workspace UI

### Frontend Tasks
**F4.1 — Workspace Layout**
*   Header Bar (Timer), Main Area (Monaco Editor, Whiteboard), Sidebar (LiveKit Audio Visualizer).

**F4.2 — Monaco Editor Setup & Copilot**
*   Install `monaco-editor`.
*   Implement `registerInlineCompletionsProvider`: pause typing for 1.5s → call Java backend `POST /api/copilot` → display ghost text.

**F4.3 — AI Voice Playback**
*   Handled entirely by LiveKit's `<AudioTrack>` component! No more manual base64 PCM queuing!

**F4.4 — LiveKit Data Channels (UI Commands)**
*   Listen to the LiveKit DataChannel. If the Python worker sends `{"type": "PROCTOR_WARNING", "message": "..."}`, show the red banner. If it sends `{"type": "TOOL_HIGHLIGHT", "line": 14}`, highlight the code.

---

## PHASE 5 — The Python AI Worker (The Live Intelligence)

> **CRITICAL CLARIFICATION FOR FRONTEND:** The AI does **NOT** run or execute the candidate's code in a sandbox. The AI acts as a **"Visual Compiler"**. It physically reads the 1fps screen-share image and uses its massive LLM reasoning to mentally trace the logic and find bugs.

### Backend Tasks (Python Worker)
**B5.1 — The LiveKit Agent Connects**
*   A Python script running `livekit-agents` connects to the room when the candidate joins.

**B5.2 — Agent 2 (The Voice / Master Interviewer)**
*   Runs Gemini Live API in BIDI mode. 
*   Prompt: *"You are Owlyn, the interviewer. Ask the pre-approved questions."*
*   Receives the candidate's audio track natively via LiveKit and speaks back.

**B5.3 — Agent 3 (The Dual Sentinels - Background Tasks)**
*   While Agent 2 talks, Python runs two `async` background loops inspecting the LiveKit video track (the screen-share):
    *   **Sentinel A (Proctor):** Looks at the left side of the image. *"Is there a phone? Are they looking away?"*
    *   **Sentinel B (Smart Workspace):** Looks at the code on the right side. *"Is there an infinite loop or syntax error?"* (It acts as a Visual Compiler).

**B5.4 — The Yield (Agent Injection)**
*   If Sentinel B sees a missing semicolon on line 14, it injects a system message into Agent 2's brain.
*   Agent 2 interrupts the candidate and speaks: *"David, check line 14, you missed a semicolon."*
*   Simultaneously, Python sends a DataChannel message to the frontend to highlight line 14 in red.

**B5.5 — The Handback (Agent 4 Assessor)**
*   When the LiveKit room closes, Python packages the entire transcript and POSTs it to the Java Backend: `POST /api/internal/reports/trigger`.
*   Java takes the transcript, calls Agent 4 (Gemini 3.1 Pro), generates the JSON scorecard, and saves it to Postgres.

---

## PHASE 6 — Full Integration & Recruiter God-View

**I6.1 — The Recruiter Monitor (Zero Backend Effort!)**
*   Because we use LiveKit, the Recruiter Dashboard doesn't need a custom WebSocket relay.
*   When Amina clicks "Watch Live", the Java backend generates a **LiveKit Token with hidden/subscriber privileges**. Amina's frontend connects to the LiveKit room and simply watches the candidate's screen-share track and listens to the audio natively!

**I6.2 — The Pitch Script (Updated)**
> *"We built an enterprise-grade, distributed AI architecture. Our Control Plane is Java Spring Boot, handling zero-trust security and structured JSON grading via Gemini 3.1 Pro. Our Data Plane leverages LiveKit WebRTC and a Python Worker to orchestrate a Concurrent Multi-Agent system. Instead of hacking together slow code execution sandboxes, we use Gemini 3.0 Flash as a 'Visual Compiler'. Two background AI Sentinels silently analyze 1fps desktop screen-shares for cheating and logical bugs, whispering their findings into the ear of our Master Voice AI, which guides the candidate in real-time with sub-second latency."*

---

## PHASE 7 — Stretch Goals (Tutor Mode)

**Tutor Mode Architecture:**
*   Frontend: Don't lock down the OS. Use Electron `desktopCapturer` to share the user's entire desktop (so they can use VS Code). Publish to LiveKit.
*   Java Backend: Flags the room as `TUTOR`. 
*   Python Worker: Reads the flag. **Turns OFF the Proctor Sentinel.** Changes Agent 2's prompt to: *"You are a friendly, patient human tutor looking at my screen."*
*   The Visual Compiler (Sentinel B) remains ON to catch bugs in the user's IDE.

***
