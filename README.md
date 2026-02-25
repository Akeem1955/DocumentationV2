








# 📂 SYSTEM ARCHITECTURE & DEVELOPMENT ROADMAP
**Project:** Live Multimodal Interview & Proctoring Platform  
**Methodology:** Waterfall (Requirements ➔ Design ➔ Implementation ➔ Testing ➔ Deployment)  
**Core Stack:** Electron (Frontend), Python (Backend/MCP), Gemini 2.0 Multimodal Live API, Google Cloud Platform (GCP)

---

## PHASE 1: System Requirements Specification (SRS)
*Team, this phase freezes our project scope. We are not adding any new features beyond this point so we can guarantee we actually finish this hackathon on time.*

### 1.1 Core Objectives
*   **Identity & Access:** Candidates will authenticate via **Firebase Authentication** (Email Link / Magic Link) to prevent unauthorized access.
*   **Controlled Workspace (Frontend):** We are building an **Electron** desktop application that enforces a lock-down (kiosk mode) and launches a lightweight **QEMU** virtual machine (running Alpine Linux) for candidate tasks.
*   **The Brain & Senses (AI Intelligence):** **Gemini 2.0 Multimodal Live API** acts as our proctor and interviewer. It natively processes continuous video and audio streams. *Remember: No local computer vision scripts. We are letting Gemini do the heavy lifting.*
*   **The Hands (Action Execution):** **Model Context Protocol (MCP)** via Python. Our MCP servers will act as secure bridges allowing Gemini to execute code inside the QEMU VM and write reports to **Google Cloud SQL**.

### 1.2 Strict Constraints
*   **Ecosystem:** Exclusively Google (GCP, Firebase, Gemini). Do not deviate.
*   **Backend:** Python 3.10+ using the official `google-genai` SDK and Python MCP SDK.
*   **Frontend:** Node.js/Electron.

---

## PHASE 2: System Architecture & Design
*This is our precise blueprint for component interactions. Frontend and Backend devs, I need you to adhere strictly to these interfaces.*

### 2.1 Component Architecture Diagram

```text
[ GOOGLE CLOUD PLATFORM (GCP) ]
  +------------------+       +-------------------+       +-----------------------+
  | Firebase Auth    |       | Google Cloud SQL  |       | Gemini 2.0 Flash      |
  | (Access Control) |       | (PostgreSQL Db)   |       | Multimodal Live API   |
  +--------^---------+       +--------^----------+       +-----------^-----------+
           |                          |                              |
           | HTTP                     | MCP Tool Call                | Secure WebSocket (WSS)
           |                          |                              |
[ CANDIDATE LOCAL MACHINE (HOST OS) ] |                              |
  +--------v--------------------------v------------------------------v-----------+
  |                          PYTHON BACKEND PROCESS (Local)                      |
  |  - Manages `google-genai` Live API connection                                |
  |  - Hosts standard Python MCP Client                                          |
  |  - Routes Tool Calls (e.g., `write_report` -> GCP, `run_code` -> VM)         |
  +--------^---------------------------------------------------------^-----------+
           | IPC / Local WebSocket Bridge                            |
  +--------v---------------------------------------------------------v-----------+
  |                          ELECTRON FRONTEND (App Wrapper)                     |
  |  - Kiosk Mode / Keyboard Hooking (Blocks Win Key, Alt+Tab)                   |
  |  - Media Capture (`getUserMedia` -> 1fps Video + 16kHz Audio)                |
  |  - Renders QEMU VM interface                                                 |
  +-----------------------------------v------------------------------------------+
                                      | Spawns & Embeds
  [ ISOLATED WORKSPACE (GUEST OS) ]   |
  +-----------------------------------v------------------------------------------+
  |  QEMU (Minimal Alpine Linux Image)                                           |
  |  +------------------------------------------------------------------------+  |
  |  | Local Python MCP Server                                                |  |
  |  | - Exposes `execute_bash`, `read_file`, `write_file` to Host Python     |  |
  |  +------------------------------------------------------------------------+  |
  +------------------------------------------------------------------------------+
```

### 2.2 Sub-System Breakdown for the Team

**Frontend (Electron):**
*   **Your Role:** You are building the shell. It verifies identity via Firebase, locks the screen, captures the user's webcam/mic, and pipes that raw media data down to the local Python Backend. You will also embed the QEMU display.
*   **Input/Output:** Send Base64 video frames and PCM audio to Python. Receive PCM audio (Gemini's voice) from Python to play through the speakers.

**Backend (Python - The Orchestrator):**
*   **Your Role:** You are the central router. Use the `google-genai` Python SDK to open a bidirectional WebSocket to Gemini. Pipe the media from Electron up to Google. 
*   **Tool Handling:** When Gemini decides to call a tool (e.g., the user says "run my code"), the Python backend receives the `FunctionCall` from the WebSocket. Use the Python MCP SDK to pass this to the specific MCP Server.

**MCP Servers (Python):**
1.  **VM MCP Server:** Runs inside QEMU. Executes candidate code and returns terminal output.
2.  **GCP MCP Server:** Runs alongside the backend. Holds Google Cloud SQL credentials and writes the final interview evaluation.

---

## PHASE 3: Implementation Plan (Developer Instructions)
*Here is our step-by-step build order. You guys can work on Frontend and Backend in parallel based on these specs.*

### Step 1: Frontend - Access & Lockdown (Hours 1-6)
**Assigned to: Frontend / UI Devs**
1.  **Firebase Auth:** Initialize the Firebase JS SDK in Electron. Implement Email Link authentication. Do not allow the app to proceed without a valid token.
2.  **Kiosk Mode:** Configure the Electron `BrowserWindow` with `fullscreen: true`, `kiosk: true`, `alwaysOnTop: true`. Use `globalShortcut` to intercept and disable 'CommandOrControl+Tab', 'Alt+Tab', 'Esc'.
3.  **Media Capture:** Use standard `navigator.mediaDevices.getUserMedia({ video: true, audio: true })`.
    *   *Video Constraint:* Downsample to 1 frame per second, convert to Base64 JPEG.
    *   *Audio Constraint:* 16kHz, single channel (mono) PCM.
4.  **Local Transport:** Create a local WebSocket (`ws://localhost:8080`) to stream this media to the Python Backend.

### Step 2: Backend - Gemini Native Intelligence (Hours 1-8)
**Assigned to: Python Backend Devs**
1.  **Setup Google SDK:** Install `google-genai`. Authenticate using a GCP Service Account (`GOOGLE_API_KEY`).
2.  **Live WebSocket Connection:** Write a Python script using `asyncio` to connect to the Gemini Live API.
3.  **System Instruction Payload:** I need you to configure the session exactly like this:
    ```python
    config = {
        "system_instruction": {
            "parts": [{"text": "You are a technical interviewer and proctor. Receive live video and audio. 1. Verbally ask the candidate to write a Python function to reverse a string. 2. Monitor them. If they look away from the screen for 5 seconds or pick up a phone, verbally warn them immediately. 3. When they ask to run code, use the execute_bash tool. 4. At the end, use write_cloud_sql_report."}]
        },
        "tools": [
            {"function_declarations": [
                {"name": "execute_bash", "description": "Runs code in the VM terminal", "parameters": {...}},
                {"name": "write_cloud_sql_report", "description": "Saves the interview result to Google Cloud SQL", "parameters": {...}}
            ]}
        ]
    }
    ```
4.  **Routing Loop:** Accept A/V from the Electron frontend, forward to Gemini. Accept Audio from Gemini, forward to Electron to play.

### Step 3: Action Layer - Python MCP Setup (Hours 8-16)
**Assigned to: Python Backend / Systems Devs**
1.  **QEMU Setup:** Download Alpine Linux (~150MB). Configure a shared folder or SSH local bridge between the Host OS and Guest OS.
2.  **Code Execution MCP (Inside VM):** 
    *   Write a tiny Python MCP server that exposes `execute_bash`.
    *   *Security Note:* It only runs inside the QEMU environment, so if the candidate writes `rm -rf /`, it only destroys the disposable VM.
3.  **Database MCP (Host OS):**
    *   Provision a Google Cloud SQL (PostgreSQL) instance.
    *   Write a Python MCP server using `asyncpg` or `SQLAlchemy` that securely connects to GCP using Workload Identity or Service Account keys.
    *   Expose `write_cloud_sql_report(candidate_email, score, behavioral_notes)`.

### Step 4: Integration & Synchronization (Hours 16-22)
**Assigned to: Full Team**
1.  Connect the loop: Electron captures User "Hello" -> Python routes to Gemini -> Gemini replies "Let's begin" -> Python routes audio to Electron -> User hears it.
2.  User writes code in VM and says "Run it" -> Gemini triggers `execute_bash` -> Python Backend receives Function Call -> Python MCP Client sends to VM MCP Server -> VM executes and returns stdout -> Gemini reads stdout and says "Your code failed on line 4."

---

## PHASE 4: Testing Plan
*We need to run these specific scenarios to guarantee our demo is completely stable.*

*   **Test 1: Impersonation / Access.** Try to open the Electron app without clicking a Firebase Magic Link. *Expected:* App remains locked on login screen.
*   **Test 2: Behavioral Proctoring (Native Gemini).** Start the interview. Look down at a cell phone for 6 seconds. *Expected:* Gemini detects the phone in the video frames and instantly generates audio: *"Please put your phone away."*
*   **Test 3: MCP Code Execution.** Write `print(1/0)` in the VM editor. Ask Gemini to test it. *Expected:* MCP executes it, returns the `ZeroDivisionError` to Gemini, and Gemini explains the math error verbally.
*   **Test 4: MCP Database Logging.** Say "I am done with the interview." *Expected:* Gemini triggers `write_cloud_sql_report`. Verify in Google Cloud Console that the row was inserted successfully.

---

## PHASE 5: Demo & Pitch Strategy
*Here is how we are going to present and pitch this to the Google Hackathon Judges. Let's make sure we hit these points.*

**1. The Hook:**
"We built a fully isolated, hardware-accelerated technical interview environment that utilizes **Gemini 2.0 Multimodal Live API** as a native, real-time proctor and conversational interviewer."

**2. The Google Architecture Flex:**
"We didn't hack together random APIs. This is a pure Google ecosystem showcase. We use **Firebase Auth** for secure entry. We use the **`google-genai` Python SDK** to stream raw 1fps video and PCM audio directly to Gemini so it can see and hear the candidate natively. There are no clunky computer vision scripts here—Gemini *is* the vision."

**3. The MCP Innovation:**
"For security and modularity, we implemented the **Model Context Protocol (MCP)** using Python. Gemini doesn't have raw access to our system. Instead, it securely calls our Python MCP servers to execute the candidate's code safely inside a QEMU VM, and it uses a separate MCP server to log the final structured evaluations directly into **Google Cloud SQL**."

**4. The Live Demo Flow:**
*   Launch Electron, authenticate via Firebase.
*   Show the locked workspace.
*   Pull out a phone on stage—let the judges hear Gemini native-voice scold you in real-time.
*   Ask Gemini to run some code, proving the MCP bridge to QEMU works.
*   End the interview, open Google Cloud Console, and show the freshly generated structured report in Cloud SQL.







Team, here is the **Developer Resource Kit (Appendix)** we are attaching to our documentation. I have reviewed all the URLs we collected and translated their contents into plain, simple English for us. 

This section tells you exactly *what* each link is, *why* we need it for this hackathon, and *who* on the team should be reading it. Let's execute this.

---

# 📚 APPENDIX: DEVELOPER RESOURCE KIT & REFERENCE URLs
*Listen up: these are mandatory reading materials for the team to prevent us from wasting time on trial and error. Google has already provided the exact blueprints for what we are building, so let's use them.*

### 1. The Core Live-Streaming Blueprint (Eyes & Ears)
**URL:** [Way Back Home - Level 3: Building an ADK Bi-Directional Streaming Agent](https://codelabs.developers.google.com/way-back-home-level-3/instructions#0)
* **Target Audience:** Our Frontend (Electron/UI) & Backend (Python) Devs
* **What it is in simple terms:** This is Google’s official tutorial on how to capture a user's webcam and microphone in the browser (or Electron) and stream it directly to Gemini using WebSockets. 
* **Why it is helpful for us:** 
  * **Frontend Team:** It gives you the exact JavaScript code needed to capture video frames at 2 FPS, encode them to Base64, and send them over WebSockets (see the *“Implement the WebSocket Hook”* section). You don't need to guess how to format the data.
  * **Backend Team:** It shows exactly how you need to set up our Python FastAPI server to receive those audio/video chunks and feed them into Google’s Agent Development Kit (ADK) `LiveRequestQueue`.

### 2. The Multi-Agent & Streaming Tools Guide (The Orchestrator)
**URL:** [Way Back Home - Level 4: Live Bidirectional Multi-Agent system](https://codelabs.developers.google.com/way-back-home-level-4/instructions#0)
* **Target Audience:** Our Python Backend / Systems Devs
* **What it is in simple terms:** This codelab teaches us how to make one "Main Agent" talk to other "Sub-Agents" (like an Orchestrator talking to a Database Agent) while handling a live video feed.
* **Why it is helpful for us:**
  * It explains how to build a **Streaming Tool**. For our proctoring use case, if we want the AI to constantly monitor the video feed for a phone or a candidate looking away without blocking the conversation, the code for a background "Sentinel" or monitoring loop is right here.
  * It shows us exactly how to format the `RunConfig` payload for a bidirectional session (handling text, audio, and tools simultaneously).

### 3. The File Upload & Sequential Agent Guide (The Pipeline)
**URL:** [Build a Multimodal AI Agent with Graph RAG, ADK & Memory Bank](https://codelabs.developers.google.com/codelabs/survivor-network/instructions#0)
* **Target Audience:** Our Python Backend Devs
* **What it is in simple terms:** A massive tutorial on building complex AI pipelines. You guys can ignore the "Graph RAG" and "Spanner" database stuff for our project. **Focus strictly on Section 9 & 10: Multimodal Pipeline.**
* **Why it is helpful for us:** 
  * If the candidate needs to upload a resume or if our agent needs to read a static code file, this shows you how to build a **Sequential Agent**. 
  * It provides the exact Python code for a `multimedia_agent.py` that handles file uploads, extracts data from the file using Gemini Vision, and saves the output.

### 4. The "Hands" Standard: Model Context Protocol (MCP) in ADK
**URL:** [Agent Development Kit (ADK) - Model Context Protocol (MCP)](https://google.github.io/adk-docs/mcp/)
* **Target Audience:** Our Backend & VM (QEMU) Tooling Devs
* **What it is in simple terms:** The official documentation on how Google’s ADK interacts with MCP. MCP is the universal plug-and-play standard we are using to let Gemini securely run code in the QEMU VM and write to the database.
* **Why it is helpful for us:**
  * It shows you how to use `FastMCP` (a Python library) to turn a simple Python function (like `execute_bash()`) into an official MCP tool with just a single `@mcp.tool()` decorator.
  * It prevents you from writing complex API wrappers from scratch. You just write the Python function, wrap it in FastMCP, and Google's ADK knows exactly how to let Gemini call it.

---











As the backend developer/orchestrator for this hackathon, your primary challenges are **managing the live bidirectional WebSocket stream**, **implementing proactive video proctoring**, and **connecting to Python MCP servers**. 

Google's "Way Back Home" Codelabs are basically a cheat sheet for your exact tech stack. However, you are on a hackathon clock, so **do not read all of them**. 

Here is your exact reading list and a detailed breakdown of how each Codelab maps directly to your project requirements.

---

### 🚨 MUST READ: The Core Foundation
#### [Level 3: Building an ADK Bi-Directional Streaming Agent](https://codelabs.developers.google.com/way-back-home-level-3/instructions#0) [3]
**This is your Holy Grail.** It contains the exact Python backend blueprint for your Electron-to-Gemini bridge. If you only read one Codelab, read this.

*   **What you will extract from it:**
    *   **The `LiveRequestQueue` Logic (Section 5):** It shows you exactly how to solve the "impedance mismatch" between raw WebSocket data from Electron and Gemini's native ingestion. 
    *   **The Upstream Task:** You will literally copy-paste the `upstream_task()` logic that receives Base64 video frames (at 1fps/2fps) and raw 16kHz PCM audio from the frontend and pushes them into `types.Blob` objects for Gemini.
    *   **The Downstream Task:** It shows you how to use `runner.run_live()` to listen for Gemini's voice and Tool Calls, format them into JSON, and shoot them back down the WebSocket to Electron.
    *   **Live Agent Prompting (Section 4):** It teaches you how to write "Control Loop Scripts" instead of standard prompts, which you will need to force Gemini to prioritize calling the `execute_bash` tool rather than just talking.

#### [Level 4: Live Bidirectional Multi-Agent system](https://codelabs.developers.google.com/way-back-home-level-4/instructions#0)[4]
Read this specifically for the **Proctoring** requirement. Standard LLMs wait for the user to speak. Your AI proctor needs to actively interrupt the user if they pull out a phone. 

*   **What you will extract from it:**
    *   **Streaming Tools / Background Monitors (Section 4):** This is the secret sauce for your proctoring requirement. It shows you how to write an `async def` function (like `monitor_for_hazard`) that runs in the background. It continuously intercepts the `LiveRequestQueue` video frames, runs a lightweight vision check, and `yields` a warning if a rule is broken. 
    *   **Agent-as-a-Tool:** It teaches how to let a main Bi-Directional Agent communicate with other tools without breaking the live audio stream. You will use this to wrap your database MCP server.

---

### 📙 HIGHLY RECOMMENDED: The Action Layer
####[Level 1: Pinpoint Location](https://codelabs.developers.google.com/way-back-home-level-1/instructions?hl=en#0) [1]
Skip the first half about generating crash evidence. Go straight to **Section 4: Build the Custom MCP Server** and **Section 6: Build the MCP Tool Connections** [1].

*   **What you will extract from it:**
    *   **FastMCP (Section 4):** This shows you how to use the `FastMCP` Python library. You will use this exact syntax (`@mcp.tool()`) to build the tiny server that runs inside your Alpine Linux QEMU VM to execute candidate code.
    *   **MCP Toolset (Section 6):** It provides the exact boilerplate for connecting Google's Agent Development Kit (ADK) to an external MCP server using `StreamableHTTPConnectionParams`. You will use this to connect your main Python backend to the Google Cloud SQL database.

---

### 🛑 DO NOT READ: Skip to Save Time
*   **[Level 0: Identify Yourself](https://codelabs.developers.google.com/way-back-home-level-0/instructions?hl=en#0):** This is about text-to-image generation (making avatars) [2]. You are doing live video streaming. Skip it entirely [2].
*   **[Level 5: Event-Driven Architecture with Kafka](https://codelabs.developers.google.com/way-back-home-level-5/instructions?hl=en#0):** This teaches how to decouple agents using Apache Kafka [5]. While cool, setting up a Kafka cluster during a 24/48-hour hackathon will destroy your timeline. Stick to direct WebSockets and HTTP/SSE for your MCP servers as defined in your strict constraints.

### 💡 Your Backend Build Path (Action Plan)
1.  **Hours 1-2:** Open **Level 3**. Copy the FastAPI boilerplate, WebSocket endpoints, and `LiveRequestQueue` setup. Use their "Loopback Test" script to ensure Electron can send you video/audio.
2.  **Hours 3-5:** Go to **Level 1 (Section 4)**. Build your two `FastMCP` servers (one for QEMU Bash execution, one for Cloud SQL).
3.  **Hours 6-8:** Open **Level 4 (Section 4)**. Write your `async def proctor_candidate()` Streaming Tool so Gemini can constantly watch the video stream for cell phones. Hook everything together into the `RunConfig`.
