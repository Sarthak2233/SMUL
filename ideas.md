# SMUL GPSE — General Purpose Simulation Engine (Serverless Edition)


> A **serverless‑first**, web‑based simulation IDE that turns natural language, code, and gestures into real‑time physics, AI environments, and interactive 3D visualizations – without managing a single server.

---


## Vision

A browser-based IDE where users can:

- Describe a scenario in **natural language** → the system derives equations and launches a simulation
- Write or edit **Python simulation code** in an embedded Monaco editor
- Visualize results in **real-time 2D/3D** via Three.js
- Manipulate objects live using **hand gestures** (MediaPipe, no extra hardware)
- Iterate with an embedded **LLM assistant** (free one, pluggable)
- Save, share, and version **projects** with full user accounts and persistence

This is a hybrid of Wolfram Mathematica, MATLAB, Unity, and an LLM-powered assistant — built entirely for the web.

---

## Core Philosophy

- **Everything reduces to mathematics** — physics, biology, economics, and AI behavior are all mathematical models
- **LLM as orchestrator, not controller** — suggests and derives; the human approves before execution
- **Modular architecture** — simulation, rendering, AI, vision, and LLM are independent plug-and-play engines
- **Multimodal-first** — code, natural language, and gesture are all first-class inputs
- **Sandbox safety** — all LLM-generated code is validated, sandboxed, and execution-time-limited before running

---

## IDE Layout (UI Spec)

```
+---------------------------------------------------------------------------------+
| Menu Bar:  Project | Simulation | AI | View | Tools | Settings                 |
+---------------------+-------------------------------+--------------------------+
| File Explorer       |       3D / 2D VIEWPORT        |   LLM CHAT PANEL        |
|---------------------|                               |-------------------------|
| /scenarios          |   (Three.js canvas)           | > Prompt input          |
| /scripts            |   Camera: Orbit / FP / Top    | > Suggestions           |
| /assets             |   Gesture overlay (skeleton)  | > Generated code preview|
| /models             |   Object selection handles    | > [Run] [Edit] buttons  |
+---------------------+-------------------------------+--------------------------+
| Monaco Code Editor  |  Inspector (Properties)       | Console / Logs          |
|---------------------|-------------------------------|-------------------------|
| Python syntax       |  Mass / Velocity / Friction   | Runtime output          |
| LLM inline hints    |  Position / Rotation / Scale  | Errors + LLM explain    |
| Run / Debug toolbar |  Material / Shader options    | Simulation timeline     |
+---------------------------------------------------------------------------------+
```

## Updated Vision (Serverless & Open)

- **Fully serverless** – no VMs, no Kubernetes, no manual scaling.  
  Frontend: S3/CloudFront. Backend: API Gateway + Lambda + Fargate (for long‑running sims).  
  Data: DynamoDB + S3 + ElastiCache Serverless (Redis).  
- **Simulation as a function** – stateless physics steps triggered by events or timer.
- **Real‑time updates** via WebSocket (API Gateway) with state cached in Redis.
- **Long‑running or continuous sims** offloaded to **AWS Fargate** (serverless containers) – auto‑scale to zero when idle.
- **Multi‑cloud ready** – design works on AWS, Google Cloud Run, or Azure Container Apps.
- **True pay‑per‑use** – costs stay near zero when nobody is simulating.

**Trade‑offs made explicit:**  
- Continuous 60 fps simulation → Fargate (not Lambda).  
- Occasional user prompts & code generation → Lambda.  
- Gesture & vision → frontend (MediaPipe) + Lambda for any backend processing.  
- AI agent training → async, using Step Functions + spot Fargate tasks.

---

## High‑Level Serverless Architecture

```
[Browser] 
   │
   ├─ Static assets (S3 + CloudFront)
   ├─ WebSocket (API Gateway) ──► Lambda (connection manager) → Redis (state)
   ├─ REST (API Gateway) ──► Lambda (auth, projects, LLM, orchestration)
   └─ Gesture (MediaPipe in‑browser) → local mapping → WebSocket commands

                            │
                            ▼
           +────────────────────────────────────+
           |  Simulation Orchestrator (Lambda)   |
           |  - Starts/stops Fargate tasks       |
           |  - Routes messages via SQS          |
           +──────────────┬─────────────────────+
                          │
           ┌──────────────┴──────────────┐
           │                             │
           ▼                             ▼
   [Fargate Task]                 [DynamoDB / S3]
   (Physics, Agents)               (Projects, Snapshots)
     │
     └──► Redis (ElastiCache Serverless) for live state
```

**Key serverless services mapped:**

| Component | Serverless Service | Why |
|-----------|-------------------|-----|
| Frontend hosting | S3 + CloudFront | Zero‑ops, global CDN |
| REST + WebSocket | API Gateway | Scales automatically, pay per request |
| Auth, projects, LLM | AWS Lambda (Python) | Short‑lived, <15 min execution |
| Simulation engine runtime | AWS Fargate | Containerised, long‑running, scales to zero |
| Real‑time state cache | ElastiCache Serverless (Redis) | Sub‑ms latency, auto‑scaling |
| Persistent data | DynamoDB + S3 | No‑SQL + blob storage, serverless |
| Asynchronous tasks | SQS + Step Functions | orchestrate training, batch jobs |
| Task scheduling | EventBridge (cron) | For periodic simulations / cleanup |

> **Alternative providers:** Google Cloud Run (Fargate equivalent) + Firestore + Memorystore; Azure Container Apps + Cosmos DB.

---

## Technology Stack (Serverless Edition)

| Layer | Technology |
|-------|-------------|
| **Frontend** | React 18 + TypeScript + Vite + Tailwind (static build) |
| **Hosting** | AWS S3 + CloudFront (or Netlify / Vercel) |
| **Code Editor** | Monaco Editor |
| **3D Renderer** | Three.js (client‑side) |
| **Gesture Input** | MediaPipe Hands JS (in‑browser) |
| **API Gateway** | AWS API Gateway (REST + WebSocket) |
| **Backend compute** | Python 3.11 Lambda functions + Fargate containers (simulation engine) |
| **LLM** | LangChain + Groq (free tier) / OpenAI (pluggable, called from Lambda) |
| **Physics / Math** | SciPy, PyBullet, NumPy, SymPy (inside Fargate) |
| **AI Agents** | PyTorch (Fargate training tasks) |
| **State storage** | DynamoDB (user metadata, projects) + Redis (live simulation state) |
| **File storage** | S3 (simulation snapshots, user models, logs) |
| **Auth** | Amazon Cognito (or JWT + Lambda authorizer) |
| **Infrastructure as code** | AWS CDK / Terraform |

---
# Before coding harness first

I've read the Martin Fowler article, and it's a great find. The concept of "Harness Engineering" can be the key to turning your ambitious serverless simulation engine from a good idea into a reliable, production-ready platform. The article's core insight is that **reliability isn't just in the AI model, but in the engineering controls you build around it**.

Let me break down what it is and, more importantly, how you can build it right into your SMUL GPSE project.

### 🤔 What is "Harness Engineering"?

In short, the article proposes that a successful AI Agent is not just the model. It's a system described by a simple, powerful equation:

> **Agent = Model + Harness**

The **Harness** is everything you, as the engineer, can control. It's not about improving the model itself, but about creating a structured, reliable environment around it.

The goal of a "Harness" is to reduce the "review toil"—the time you spend checking and fixing AI outputs. It provides a "steering loop" that catches and corrects issues before they ever reach the user.

To build this harness, the article outlines two main categories of controls:

*   **Feedforward Guides (Anticipate & Prevent)**: These are rules and constraints that guide the model *before* it takes action, increasing the chance it gets it right on the first try.
*   **Feedback Sensors (Observe & Correct)**: These tools check the model's output *after* it acts, allowing it to self-correct.

Crucially, these controls can be either "computational" (fast, cheap, and deterministic, like a linter) or "inferential" (slower, more expensive, and powered by another AI model, like a code reviewer). A well-designed harness uses a mix of both.

### 🔧 How to Apply Harness Engineering to SMUL GPSE

This framework is a perfect fit for your architecture. Here's a practical guide to building the harness, moving from the safest (computational) to the most powerful (inferential) controls, with examples you can implement.

#### ✅ 1. Computational Guides (Feedforward)

These are the first line of defense, acting before code is ever generated.

*   **Action**: Provide the LLM with clear rules **before** generating code.
*   **Feature to Build**: A `SIMULATION_RULES.md` or `AGENTS.md` file in your project root. This file is your "constitution for simulation code."
*   **What to Write**: Use these **exact examples** as a template:

    ```markdown
    ## Physics Simulation Rules
    - Use `scipy.integrate.solve_ivp` for ODEs, never Euler method.
    - Define all physical parameters (mass, length, gravity) as constants at the top.
    - When calculating energy, output total (kinetic + potential) as a field named "total_energy".
    - The simulation state MUST be a dictionary with keys: "objects", "time", "energy".

    ## Serverless API Rules
    - Every exposed endpoint must have a corresponding OpenAPI schema in `schemas/`.
    - All simulation state updates must be published to the Redis channel `sim:{id}:state`.
    - Fargate tasks must register a shutdown hook to save final state to S3.

*   **How to Integrate**: When your backend calls the LLM to generate code, prepend the content of your `AGENTS.md` file as part of the system prompt. This directly influences the model's behavior from the very first step.

#### ✅ 2. Computational Sensors (Feedback)

These are your fast, automated sanity checks that run immediately after code is generated.

*   **Action**: Create a validation pipeline that runs on all LLM-generated code before execution.
*   **Feature to Build**: A `SimulationCodeValidator` service.
*   **Implementation Idea**:
    *   **Static Analysis (Linting)**: Use a tool like `pylint` to check for rule violations (e.g., "Is the LLM using `scipy.integrate.solve_ivp`?"). This is a perfect **computational sensor**.
    *   **Type Checking**: Use `mypy` to ensure the generated code adheres to your project's data schemas.
    *   **Structural Validation**: Write a small script that checks if the generated code defines the required functions (e.g., `run_simulation`, `get_state`) and returns the required dictionary structure.

This pipeline acts as a gatekeeper, rejecting clearly invalid code instantly and cheaply.

#### 🧠 3. Inferential Guides (Feedforward)

When computational rules are too complex, you can use an AI to guide another AI.

*   **Action**: Use a "reviewer" AI model to analyze a "generator" AI's plan before execution.
*   **Feature to Build**: A `CodeReviewAgent`.
*   **How it Works**:
    1.  Your main LLM (the generator) produces an initial "plan" or "pseudocode" for the simulation.
    2.  A second, more powerful (or differently-tuned) LLM (the reviewer) is prompted: `"You are a senior simulation architect. Review this plan for adherence to our physics rules, performance considerations in a serverless environment, and security best practices. Provide a score (1-10) and a pass/fail decision."`
    3.  If the plan fails, the reviewer provides specific feedback. This feedback is then fed back to the generator for a second, revised attempt.

This uses the unique strength of LLMs (semantic understanding) to enforce complex, nuanced guidelines that would be impossible to codify as simple rules.

#### 🧠 4. Inferential Sensors (Feedback)

This is the most powerful but also the most expensive control. It's an AI reviewing the final output of another AI.

*   **Action**: After the code is written, have an AI judge check it.
*   **Feature to Build**: The same `CodeReviewAgent` (or a separate `CodeJudgeAgent`) can be used post-generation.
*   **How it Works**: Feed the final, generated code to an LLM with a prompt like:
    > `"Does this simulation code correctly implement a double pendulum with air resistance? Does it calculate energy? Does it pass the rawdog test?"`

This is the kind of "sensor" that could identify if the LLM secretly cheated and used a simplified model instead of the complex one you requested—a type of error no linter could ever catch.

#### 🏗️ Architectural Guardrails

Finally, the structure of your codebase itself can act as an implicit harness.

*   **Action**: Enforce strict, modular patterns.
*   **Feature to Build**: Your `ai/templates` folder (`component.tsx`, `api_route.py`, `service.py`) is a perfect start. But you can take it further.
*   **How it Works**: The very way you architect your codebase can act as a harness.
    *   **Mandatory Dependency Injection**: Structure your code so that the simulation engine *must* receive its dependencies (e.g., a `SensorInterface`, a `Logger`) rather than creating them. This forces a modular design.
    *   **Standardized Interfaces**: Enforce that every plugin must conform to a `SimulationPlugin` abstract base class. This creates a known, testable boundary for all AI-generated components.

### 📝 Harness Implementation Plan for SMUL GPSE

You can integrate these harness components into your existing development plan seamlessly. Here is a consolidated **Harness Implementation Table** for you to reference:

| Module | Control Type | Key Features / Implementation                                                                                                                                                               |
| :----- | :----------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **📬 Input** | Feedforward  | Validate all API inputs against strict JSON Schemas; reject malformed requests at the Lambda/API Gateway level.                                                                            |
| **🧠 LLM** | Feedforward  | 1) Prepend `AGENTS.md` rules to system prompt for context. 2) Use a secondary LLM to review the initial plan (inferential guide) before code generation.                                 |
| **💻 Code** | Feedback     | 1) Run **computational sensors** (`pylint`, `mypy`) to validate structure. 2) Post-code review by a **CodeJudgeAgent** (inferential sensor) to catch nuanced errors.                 |
| **🏗️ Architecture** | Feedforward  | **Architectural guardrails**: Mandatory dependency injection patterns, standardized `SimulationPlugin` interfaces, and modular contracts that force AI to "plug in" correctly.             |
| **🏃 Runtime** | Feedback     | **Historical pattern memory**: Log successful and failed code patterns into `harness/knowledge/`. Use this as few-shot examples or RAG context to guide future generations.                |

This framework, inspired by the article, turns your project from a passive platform into an active, resilient system—one that learns and improves every time it's used.

This is a powerful new way of thinking about AI integration. Does the breakdown of the different control types (computational vs. inferential) make sense? If you have questions about how to implement a specific one, like the `CodeReviewAgent`, just let me know.
```


## Repository Structure (Serverless Optimised)

```
SMUL-GPSE/
│
├── frontend/                       # React static app
│   ├── src/
│   ├── public/
│   └── package.json
│
├── backend/
│   ├── functions/                  # 🔥 Lambda handlers (one per endpoint)
│   │   ├── auth/
│   │   ├── projects/
│   │   ├── llm/
│   │   ├── simulation/
│   │   │   ├── start.py           # starts Fargate task
│   │   │   ├── stop.py
│   │   │   └── status.py
│   │   ├── websocket/
│   │   │   ├── connect.py
│   │   │   ├── disconnect.py
│   │   │   └── send.py
│   │   └── shared/                # common DB & Redis clients
│   │
│   ├── containers/                # 🔥 Fargate tasks (long‑running)
│   │   ├── simulation_engine/     # physics + agent loop
│   │   │   ├── Dockerfile
│   │   │   ├── main.py            # connects to Redis, consumes SQS
│   │   │   └── engine/
│   │   ├── trainer/               # AI agent training (spot instances)
│   │   └── plugin_runner/
│   │
│   ├── infrastructure/            # CDK / Terraform
│   │   ├── stacks/
│   │   │   ├── api.ts
│   │   │   ├── database.ts
│   │   │   ├── compute.ts
│   │   │   └── frontend.ts
│   │   └── bin/
│   │
│   ├── shared/                    # Python packages (Lambda layers)
│   │   ├── models/
│   │   ├── utils/
│   │   └── requirements.txt
│   │
│   └── tests/
│
├── ai/                            # LLM prompts & rules (unchanged)
├── docs/
├── scripts/                       # deployment helpers
└── .github/workflows/             # CI/CD (deploy to AWS)
```

---

## Updated Simulation Data Flow (Serverless)

### Scenario: user types “simulate double pendulum with damping”

1. **Frontend** → `POST /llm/prompt` (API Gateway → Lambda).
2. **Lambda (LLM)** → calls Groq, returns code & scenario config.
3. User approves → frontend calls `POST /simulation/start` with config.
4. **Lambda (simulation/start)**:
   - Creates entry in DynamoDB (`simulation_id`, status=`starting`).
   - Pushes config to SQS queue (`simulation-queue.fifo`).
   - Triggers Fargate task (via `RunTask`). Task pulls from SQS.
5. **Fargate task** (simulation engine):
   - Reads config, loads physics engine.
   - Registers with Redis (key `sim:{id}:state`, TTL=10min).
   - Opens WebSocket connection (via API Gateway URL + connectionId from DB).
   - Loop: `physics.step(dt)`, `agent.act()`, push state to Redis, broadcast via WebSocket.
   - On user gesture: receives command from WebSocket → applies force.
   - On idle (no interactions for 30s) → gracefully shuts down (task stops).
6. **Frontend** (Three.js) listens to WebSocket – updates scene at ~60 fps.
7. **Snapshot** → frontend calls `POST /simulation/{id}/snapshot` → Lambda stores simulation blob in S3 + metadata in DynamoDB.

> **Cost efficiency:** Fargate task runs only while user interacts (or continuous simulation is active). Idle timeout kills the task – starting new simulation takes ~10-15 sec (cold start). For sub‑second startup, provisioned concurrency can be used (higher cost).

---

## Core APIs (Serverless Style)

### REST Endpoints (API Gateway + Lambda)

| Method | Endpoint | Lambda Handler |
|--------|----------|----------------|
| `POST` | `/auth/register` | auth.register |
| `POST` | `/auth/login` | auth.login |
| `GET` | `/projects` | projects.list |
| `POST` | `/projects` | projects.create |
| `POST` | `/simulation/start` | simulation.start |
| `POST` | `/simulation/{id}/stop` | simulation.stop |
| `GET` | `/simulation/{id}/status` | simulation.status |
| `POST` | `/simulation/{id}/snapshot` | simulation.snapshot |
| `POST` | `/llm/prompt` | llm.parse |
| `POST` | `/llm/explain` | llm.explain |

### WebSocket Events (API Gateway WebSocket)

| Action (message type) | Direction | Payload | Handling |
|-----------------------|-----------|---------|----------|
| `sim:start` | client → | `{simulationId, config}` | Lambda stores connection mapping, triggers Fargate |
| `sim:command` | client → | `{type:"force", objectId, forceVec}` | Lambda pushes to SQS for that simulation task |
| `sim:state` | server → | `{objects, energy, time}` | Fargate task -> API Gateway -> client |
| `sim:error` | server → | `{code, message}` | Fargate task → client |

### Internal (Fargate ↔ SQS / Redis)

- **Control queue** (SQS FIFO):  
  Messages from outside (gestures, LLM actions) delivered to the simulation task.
- **State cache** (Redis):  
  Key `sim:{id}:state` – JSON blob updated at each physics step.  
  Key `sim:{id}:connections` – set of WebSocket connectionIds for broadcasting.

---

## Threading & Concurrency (Serverless mind‑set)

| Component | Concurrency Model |
|-----------|-------------------|
| **Lambda functions** | Each invocation = isolated process. No shared memory. Use DynamoDB + Redis for coordination. |
| **Fargate simulation task** | Single‑threaded asyncio event loop. `physics.step()` + `websocket.send()` + `sqs.poll()` in one loop. |
| **Message ordering** | SQS FIFO queue + `MessageGroupId=simulationId` ensures commands arrive in order. |
| **State consistency** | Redis with optimistic locking (WATCH) for critical updates (e.g., grabbing an object). |

**No more double buffering or shared threads** – everything is stateless across invocations. The simulation task runs continuously, but it can be stopped/restarted without losing state because Redis persists the latest snapshot.

---

## Gesture Control (same as original – in‑browser)

No change: MediaPipe runs entirely on client. Gesture → command → WebSocket (`sim:command`) → SQS → Fargate task.  
**Serverless advantage:** The Fargate task can be idle and still receive commands via SQS, waking up if needed (though typically task is already alive while simulation is active).

---

## AI Agent Module (Serverless)

- **Training** – triggered by user: `POST /agent/train` → Lambda starts a **Fargate spot task** (cheap) that runs for hours, streaming metrics back via WebSocket.
- **Inference** – during simulation: agent runs inside the same Fargate simulation task (real‑time).
- **Model storage** – S3 (versioned). Lambda can download model weights on task start.

---

## Plugin System (Serverless)

Plugins are now **container images** registered in ECR.  
When user selects a plugin scenario:
1. API Gateway → Lambda → launches a new Fargate task using that plugin’s container.
2. Plugin code can be written in any language (container interface defined via stdin/stdout + Redis).
3. Plugin marketplace: users can upload their own container images (subject to validation).

---

## Security (Serverless Hardened)

| Concern | Mitigation |
|---------|-------------|
| **LLM code injection** | Sandbox in Fargate task: run inside a locked‑down container (no host network, read‑only root, seccomp). |
| **Execution time abuse** | Fargate task max runtime = 24h (configurable). Idle timeout kills it. |
| **Auth bypass** | Cognito user pools + Lambda authorizers on every API endpoint. |
| **Data leakage** | DynamoDB encrypted at rest, S3 server‑side encryption, HTTPS only. |
| **Cold start latency** | For low‑latency requirements (e.g., real‑time demo) → use Fargate provisioned concurrency ($$) or keep‑warm scheduling. |
| **WebSocket connection limits** | API Gateway supports 500k concurrent connections – enough for most. |

---

## Development Phases (Serverless Adjusted)

### Phase 0 – Infrastructure & CI/CD (2 days)
- Setup AWS CDK / Terraform project.
- Deploy: S3 bucket + CloudFront for static hosting.
- Deploy: DynamoDB table for `Simulations` + `Projects`.
- Deploy: ElastiCache Serverless (Redis).
- Deploy: API Gateway (REST + WebSocket) with mock integrations.
- **Milestone:** `cdk deploy` creates all resources; no backend code yet.

### Phase 1 – Lambda Auth & Projects (3 days)
- Write Lambda functions: `auth.register`, `auth.login`, `projects.list`, `projects.create`.
- Use JWT (or Cognito) – Lambda authorizer.
- Frontend calls endpoints → works.
- **Milestone:** User can register, login, create project via API.

### Phase 2 – Simulation Orchestration (5 days)
- Lambda `simulation.start`:
  - Validate config.
  - Store `simulation` record in DynamoDB (status = PENDING).
  - Push message to SQS.
  - Call ECS `RunTask` for Fargate simulation task.
- Fargate container: simple loop that reads config from SQS, writes to Redis, sends dummy state via WebSocket.
- WebSocket handlers: `connect`, `disconnect`, `send`.
- **Milestone:** POST /simulation/start → Fargate task starts, WebSocket receives `sim:state` every second.

### Phase 3 – Real Physics Engine (4 days)
- Integrate PyBullet + SciPy into Fargate container.
- 60 Hz simulation loop, push state to Redis → WebSocket.
- Frontend Three.js consumes WebSocket and animates.
- **Milestone:** Double pendulum animates smoothly.

### Phase 4 – LLM Integration (3 days)
- Lambda `llm.parse` (LangChain + Groq).
- Frontend LLM chat panel → code preview → approve → calls `simulation/start` with generated config.
- **Milestone:** “simulate a double pendulum” → code approved → simulation runs.

### Phase 5 – Gesture & Real‑time Commands (3 days)
- MediaPipe frontend → classify gesture → WebSocket `sim:command`.
- Fargate task polls SQS, applies force to physics engine.
- **Milestone:** Pinch gesture selects pendulum bob, drag applies impulse.

### Phase 6 – AI Agents & Training (5 days)
- Fargate simulation task includes PyTorch policy.
- Separate Fargate training task (spot) that saves models to S3.
- Frontend training dashboard (Chart.js) via WebSocket stream.
- **Milestone:** Cart‑pole training shows improving reward curve.

### Phase 7 – Snapshots, Plugins & Polish (4 days)
- Snapshot endpoint: Lambda stores Redis state to S3.
- Plugin system: custom container images.
- Idle timeout: Fargate task stops after 30s of no WebSocket messages.
- **Milestone:** Production‑like, cost‑efficient serverless simulation IDE.

---

## Example Pipeline (Serverless Execution)

**User types:** *“Show me a chaotic double pendulum and visualize energy loss”*

1. Frontend → `POST /llm/prompt` → Lambda → Groq → returns Python code & config.
2. User approves → Frontend → `POST /simulation/start` with config.
3. Lambda `simulation.start`:
   - Creates DynamoDB entry.
   - Sends config to SQS FIFO queue.
   - Runs Fargate task (container: `simulation_engine`).
4. Fargate task:
   - Reads config from SQS.
   - Starts PyBullet + ODE solver.
   - Opens WebSocket connection via API Gateway URL (connectionId from DynamoDB).
   - Loop: compute physics, update Redis, broadcast state.
5. Three.js frontend renders, shows energy trail.
6. User pinches pendulum bob → Frontend sends `sim:command` via WebSocket.
   - API Gateway → Lambda (websocket/send) → SQS → Fargate task applies force.
7. User closes tab → API Gateway `$disconnect` → Lambda → stops Fargate task (or marks idle).
8. Next day, user returns → `GET /simulation/{id}/status` → Fargate task auto‑stops, but last state is in Redis. User can `POST /simulation/{id}/resume` to restart a new task with same state.

---

## Open Possibilities (Serverless Unlocks)

- **Massive parallel simulations** – Run 1000 different parameter sweeps concurrently using Fargate `RunTask` (each cheap, isolated).
- **Real‑time collaboration** – Multiple users join same WebSocket room (API Gateway supports 500k connections). Fargate task broadcasts state to everyone.
- **Simulation as a Service (SaaS)** – Users expose simulation endpoints via API Gateway + Lambda (serverless backend for their own apps).
- **Serverless rendering** – Offload heavy Three.js renders to AWS Lambda + headless Chrome (e.g., generate videos of simulations).
- **Pay‑per‑simulation** – Bill users per second of Fargate compute + API calls. No upfront infrastructure cost.
- **Cross‑region replication** – DynamoDB global tables + CloudFront → simulations run close to users.

---

## Migration from Original Design – Key Changes Summary

| Original (Container) | Serverless Edition |
|----------------------|--------------------|
| Docker Compose for everything | AWS CDK + Fargate + Lambda |
| PostgreSQL (stateful) | DynamoDB + S3 (stateless) |
| Redis inside same network | ElastiCache Serverless (managed) |
| FastAPI long‑running | API Gateway + Lambda functions |
| WebSocket handled by FastAPI | API Gateway WebSocket + Lambda |
| Simulation thread inside backend | Separate Fargate task (auto‑scales to zero) |
| Nginx + Gunicorn | CloudFront + Lambda (no servers) |
| Manual scaling of containers | No scaling config (pay per use) |

**Trade‑off caution:** Cold starts for Fargate tasks can be 10‑15 seconds. For an instant “live” feel, keep‑warm strategies (scheduled ping) or use provisioned concurrency at extra cost.

---

## Conclusion

The serverless architecture transforms SMUL GPSE into a **cost‑efficient, elastically scalable, and operationally invisible** platform. Users get the full power of a simulation IDE (physics, LLM, gestures, AI agents) without provisioning a single server. The hybrid approach (Lambda for control plane, Fargate for continuous simulation) balances real‑time requirements with serverless benefits.

All original features (3D rendering, gesture control, LLM assistant, AI agents, plugin system) are preserved and often enhanced by serverless capabilities—parallel runs, collaboration, and pure pay‑per‑use billing.

---

## Future Extensions

- **Digital twins** — connect to real sensor data streams
- **Scientific research platform** — export simulation data as CSV/JSON for papers
- **Cloud simulation clusters** — offload heavy workloads to remote compute
- **Networked / multiplayer simulation** — multiple users in same scene via shared WebSocket room
- **Custom ML gesture classifier** — train PyTorch model on user-specific gestures (Phase G5)
- **ECS (Entity Component System)** — replace object dict with proper ECS for large-scale scenes
- **`.simproj` project file format** — portable project bundle (scene + scripts + assets)
