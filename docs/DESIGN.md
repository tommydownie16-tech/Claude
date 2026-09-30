# Mac AI Assistant — Architecture Proposal (for approval)

Status: PROPOSAL, awaiting approval. Nothing beyond this document has been built.

Working codename: **Helm** (the app is the "helm" of the Mac). This is only a repository/target name and is trivially renamed; the assistant's spoken name is set by you in onboarding.

---

## 0. Findings from inspecting the development environment

| Fact | Consequence |
|---|---|
| This Claude session runs in an Ubuntu 24.04 Linux container (x86_64), not on your Mac | SwiftUI, AppKit, AVFoundation, ScreenCaptureKit, Vision, Accessibility and Keychain code **cannot be compiled or run here** |
| No Swift toolchain is installed, and the network policy denies `download.swift.org` | Even pure Swift cannot be compiled here today |
| The repository `tommydownie16-tech/Claude` is empty (no commits) | We start from a clean slate |
| GitHub is reachable | GitHub Actions **macOS runners** can build the app, run unit tests and report back; I can read the CI logs from here |
| Node 22, Python 3.11, clang 18, cmake are installed | Useful only for scripts and tooling, not the app |

### What this means for "Build → Test → Fix → Commit" at every phase

The requirement that the repository is never left broken can only be honoured if every push is compiled and tested on macOS. Proposed verification pipeline:

1. **GitHub Actions `macos-15` runner** (Xcode 16.x): `xcodegen generate` → `xcodebuild build test` for the app + all packages, on every push. This is the authoritative build. macOS minutes are billed at 10× on private repos (200 effective minutes/month on the free tier); a public repo has unlimited minutes. Your call (see §11).
2. **Linux core tests (optional, faster feedback)**: if you add `download.swift.org` to the environment's allowed domains, I can install Swift 6 here and run the platform-independent packages' tests locally before pushing. Without it I rely solely on CI.
3. **Hardware-dependent behaviour** (camera, microphone, screen capture, Accessibility control, Keychain prompts, Touch ID) cannot be exercised on any CI runner. Each phase will ship a **manual verification checklist** for you to run on the Mac mini, and diagnostics inside the Developer panel to make that quick. I will not claim these paths are verified until you confirm them.

Architecture decisions below are driven by this: as much logic as possible lives in **platform-independent Swift packages** (testable on Linux and CI), and the macOS-only layer is a thin set of adapters.

---

## 1. Guiding principles

1. **Observe → Understand → Plan → Ask if necessary → Act → Verify → Report.** The model proposes; the app disposes.
2. **The LLM never touches the OS.** It can only emit structured tool calls. A tool call is data until the Permission Engine issues a single-use ticket for it.
3. **Every input that did not come from your keyboard or microphone is untrusted content** — files, screen text, web pages, emails, OCR results, tool output — and is tainted as such through the whole pipeline.
4. **Nothing is hidden.** Sensor state, data egress, memory writes and every tool action are visible and logged.
5. **No fake features.** A capability is either implemented and verified, implemented-but-unverified-on-hardware (labelled), or absent. Buttons that do nothing do not ship.
6. **Least privilege, defence in depth.** Untrusted parsing happens in a sandboxed XPC service. Sensitive actions require macOS authentication. Policies default to ASK.

---

## 2. Technology choices

| Concern | Choice | Why / alternatives considered |
|---|---|---|
| Language | **Swift 6**, strict concurrency (actors) | Native, single language across UI/services; `Sendable` checking helps with the many concurrent subsystems |
| Minimum OS | **macOS 15 (Sequoia)**; macOS 26 features behind `#available` | ScreenCaptureKit screenshot API, MenuBarExtra, SwiftUI shaders all available; SpeechAnalyzer/Foundation Models used opportunistically on 26 |
| UI | **SwiftUI**, AppKit where SwiftUI is weak (`NSPanel` for the palette, `NSStatusItem`/`MenuBarExtra`, global hotkeys, `NSOpenPanel`) | Native look, low latency, Metal-backed orb via SwiftUI `.layerEffect` shaders |
| Project definition | **SwiftPM packages** for all logic + **XcodeGen `project.yml`** for the `.app` and XPC targets | `.xcodeproj` files cannot be authored from Linux; XcodeGen makes the project text-based and reproducible on CI |
| App Sandbox | **Not sandboxed** main app (Developer ID + Hardened Runtime). **Sandboxed XPC** for document parsing | Accessibility control of other apps, arbitrary shell execution and AppleScript are incompatible with App Sandbox. Privilege is instead confined by process separation + policy |
| Local DB | **SQLite** via **GRDB.swift** (MIT) with **FTS5** | Mature, migrations, `Codable` records, full-text search built in. Vector search: brute-force cosine over BLOB embeddings with Accelerate in v1 (fine to ~500k chunks); `sqlite-vec` later if needed |
| Secrets | **Keychain** (`SecItem`, access group scoped to the app) | API keys, face-template encryption key, integration tokens |
| Sensitive data at rest | **AES-GCM (CryptoKit)** with a Keychain-held key for face templates, audit log bodies and memory export | SQLCipher considered; column-level encryption keeps GRDB stock and lets FTS work on non-sensitive columns |
| Camera / audio I/O | **AVFoundation** (`AVCaptureSession`, `AVAudioEngine`) | Standard |
| Speech-to-text | Provider protocol. Default **Apple Speech** on-device (`SFSpeechRecognizer` with `requiresOnDeviceRecognition`; `SpeechAnalyzer` on macOS 26). Optional **whisper.cpp** (SwiftPM, Metal) and hosted STT | Apple is zero-dependency and streams; whisper.cpp is the quality fallback; hosted for accents/noise if you choose |
| Voice activity detection | Energy + spectral-flatness VAD in v1; **Silero VAD** (CoreML) as an upgrade | Apple offers no public VAD. Silero is small (~2 MB) and accurate; conversion is a Phase 3 task if the simple VAD proves inadequate |
| Wake phrase | Continuous **on-device** STT with phrase matching; push-to-talk and open-conversation modes | No public wake-word API. Porcupine/openWakeWord are options but add licences/ONNX; on-device STT keeps audio local at higher CPU cost |
| Text-to-speech | **`AVSpeechSynthesizer`** (on-device, Siri/Premium voices, instant stop for barge-in). Optional hosted TTS (OpenAI, ElevenLabs) | Interruption is a hard requirement; local synthesis has ~zero latency to first audio |
| Computer vision (local) | **Vision framework**: text (`VNRecognizeTextRequest`), barcodes/QR, document segmentation, face detection/landmarks, image classification, feature prints | All on-device, no models to ship |
| Scene / object understanding | Vision-capable LLM through the router (Anthropic, OpenAI-compatible, Ollama `qwen2.5-vl`/`llava`/`gemma3`) | Conversational multi-frame reasoning needs an LLM; local models supported for privacy |
| Face recognition (enrolled users) | `FaceRecognizer` protocol. v1: Vision face detection + landmark alignment + `VNGenerateImageFeaturePrint` similarity with a conservative threshold. Upgrade: ArcFace/FaceNet CoreML model | Presence/personalisation only, never authentication. Feature prints are generic image similarity — v1 will be labelled "presence heuristic" and must pass an enrolment self-test before it is enabled |
| Screen awareness | **ScreenCaptureKit** (`SCScreenshotManager`, `SCStream`, `SCContentSharingPicker` for user-chosen windows) + **Accessibility tree** text (`AXUIElement`) for cheap, exact text of the frontmost window | AX text avoids OCR for terminals/editors; screenshots for everything else |
| Computer control | **Accessibility API** (`AXUIElement`) for inspect/click/focus; **`CGEvent`** for typing/shortcuts; `NSWorkspace` for apps/files/URLs; **AppleScript** only for allow-listed scripts (Mail compose, Finder reveal); **Shortcuts** via `shortcuts run` | Structured, verifiable, no screen-coordinate guessing unless AX fails |
| Shell | Foundation `Process`, argv-only (no `sh -c` unless explicitly a "shell" tool with higher risk), sanitised env, cwd confined to project roots, timeouts, PTY optional for dev servers | Command policy engine classifies risk (see §5) |
| LLM providers | Thin in-house clients: **Anthropic Messages API** (tools, vision, streaming) and **OpenAI-compatible** (OpenAI, Ollama, LM Studio, OpenRouter, vLLM). Gemini later | Two protocols cover nearly everything, including local. Avoids heavy SDK dependencies |
| Embeddings | Router: Ollama `nomic-embed-text`/`bge-m3` (local), hosted (OpenAI/Voyage), fallback `NLContextualEmbedding` (on-device, lower quality) | Local by default |
| Local LLM | Ollama / LM Studio over `localhost` (OpenAI-compatible); macOS 26 Foundation Models for tiny on-device classification | No inference engine to embed in v1; keeps the app small |
| Global hotkeys | **KeyboardShortcuts** (sindresorhus, MIT) | Proven Carbon wrapper + SwiftUI recorder UI for settings |
| Secure IPC | **XPC** with `NSXPCConnection.setCodeSigningRequirement` | Only same-team-signed binaries can talk to the services. **No localhost HTTP server** in the app: any process on the Mac could otherwise drive the tool layer |
| Notifications | `UNUserNotificationCenter` | Proactive alerts with quiet hours and rate limiting |
| Tests | Swift Testing + XCTest; XCUITest smoke test for launch/onboarding | Runs on the macOS CI runner |
| Lint/format | `swift-format` (ships with toolchain), SwiftLint on CI | Consistency |
| Signing | Developer ID + notarization for a real build; **an Apple Development certificate at minimum** | TCC (Accessibility, Screen Recording, Camera…) grants are tied to code signature. Ad-hoc signing changes per build and would re-prompt every rebuild |

### Required macOS permissions, entitlements and external dependencies (nothing is faked)

| Capability | Mechanism | User-visible requirement |
|---|---|---|
| Camera | `NSCameraUsageDescription` | TCC prompt |
| Microphone | `NSMicrophoneUsageDescription` | TCC prompt |
| Speech recognition (Apple) | `NSSpeechRecognitionUsageDescription` | TCC prompt (on-device mode; no audio leaves the Mac) |
| Screen | ScreenCaptureKit | Screen Recording in System Settings → Privacy |
| Computer control | Accessibility API, CGEvent | Accessibility in System Settings → Privacy |
| AppleScript to Mail/Finder/etc. | `NSAppleEventsUsageDescription` | Automation prompt per target app |
| Contacts ("email John") | `NSContactsUsageDescription` | TCC prompt |
| Calendar | `NSCalendarsFullAccessUsageDescription` | TCC prompt |
| Files | User picks folders in `NSOpenPanel`; roots stored as security-scoped bookmarks; access outside chosen roots refused by policy even though the app is unsandboxed | Explicit folder choice; Full Disk Access **not** requested |
| Touch ID / password for RED actions | `LocalAuthentication` `.deviceOwnerAuthentication` | Touch ID needs a Magic Keyboard with Touch ID on a Mac mini; otherwise password (Apple Watch unlock also works) |
| Hosted models | API keys entered in Settings → stored in Keychain | Internet; egress indicator shown |
| Local models | Ollama or LM Studio installed by you | None; detected automatically |
| Notifications | `UNUserNotificationCenter` | Prompt |

---

## 3. System architecture

### 3.1 Processes

```
┌────────────────────────────────────────────────────────────────────────┐
│  Helm.app (SwiftUI, unsandboxed, hardened runtime)                     │
│                                                                        │
│  UI layer            Command Centre · Orb · Palette (NSPanel)          │
│                      Menu bar · Onboarding · Permission prompts        │
│  ───────────────────────────────────────────────────────────────────── │
│  Orchestrator        AssistantSession (actor) — the O-U-P-A-A-V-R loop │
│  ───────────────────────────────────────────────────────────────────── │
│  Services (actors)   Voice · Vision · Screen · Control · Projects      │
│                      Memory · Tasks · Automation · Proactive · Audit   │
│  ───────────────────────────────────────────────────────────────────── │
│  Gate                ToolBroker → RiskClassifier → PermissionEngine    │
│                      → Ticket → ToolExecutor → Verifier → AuditLog     │
│  ───────────────────────────────────────────────────────────────────── │
│  Infrastructure      ModelRouter · Providers · Storage(GRDB) · Keychain│
└───────────────┬───────────────────────────────────┬────────────────────┘
                │ XPC (code-signing requirement)     │ XPC (Phase 11)
┌───────────────▼───────────────┐      ┌────────────▼───────────────────┐
│ HelmIndexer.xpc (SANDBOXED)   │      │ HelmToolRunner.xpc (optional)   │
│ parses PDFs/docs/code, chunks │      │ shell + AppleScript execution   │
│ read-only bookmarks to roots  │      │ isolated from UI/model process  │
└───────────────────────────────┘      └────────────────────────────────┘
                │ localhost HTTP (outbound only)
        ┌───────▼────────┐        ┌──────────────────────┐
        │ Ollama/LM Studio│        │ Hosted APIs (opt-in) │
        └────────────────┘        └──────────────────────┘
```

* The main app runs the UI, sensors and orchestration. Sensors must be in the app because TCC attributes camera/mic/screen to the responsible app bundle.
* `HelmIndexer.xpc` is **sandboxed** and parses untrusted documents. A malicious PDF cannot reach the Keychain, network or shell.
* `HelmToolRunner.xpc` (Phase 11) moves shell/AppleScript execution out of the process that holds the model context, giving crash isolation and a second enforcement point that re-validates the ticket.
* No inbound network listener of any kind.

### 3.2 Packages (SwiftPM) and the dependency rule

Dependencies point downward only. Upper layers never import lower layers' platform frameworks; lower layers never import SwiftUI.

```
HelmApp (Xcode target, macOS only)
  └── HelmMac (macOS adapters: AVFoundation, Vision, SCK, AX, Keychain, NSWorkspace…)
        └── HelmCore (platform-independent, Foundation only — Linux-testable)
              ├── HelmProviders (Anthropic / OpenAI-compatible / Ollama clients)
              ├── HelmStorage  (GRDB schema, migrations, repositories)
              ├── HelmIndexing (chunking, tokenisation, ranking, hybrid search)
              └── HelmPolicy   (risk tables, shell command policy, path policy,
                                injection heuristics, taint tracking)
```

`HelmCore` defines every protocol the macOS layer implements (`CameraSource`, `SpeechRecognizer`, `ScreenCapturer`, `UIController`, `ProcessRunner`, `SecretStore`, …), so the orchestration, task engine and permission logic are tested with fakes on Linux/CI, independent of hardware.

---

## 4. Repository structure

```
Helm/
├── Package.swift                     # HelmCore, HelmPolicy, HelmProviders, HelmStorage, HelmIndexing, HelmMac
├── Sources/
│   ├── HelmCore/
│   │   ├── Domain/                   # Conversation, Message, Attachment, Project, Memory, Task, AuditEntry…
│   │   ├── Tools/                    # ToolSpec, ToolCall, ToolRequest, ToolResult, ToolRegistry, schemas
│   │   ├── Gate/                     # ToolBroker, RiskClassifier, PermissionEngine, Ticket, Verifier
│   │   ├── Orchestration/            # AssistantSession, TurnRunner, ContextBuilder, Taint
│   │   ├── Tasks/                    # TaskEngine, TaskPlan, TaskStep, state machine
│   │   ├── Memory/                   # MemoryLayers, MemoryStore protocol, retrieval
│   │   ├── Routing/                  # ModelRouter, ModelRole, Capability matrix, EgressPolicy
│   │   ├── Ports/                    # protocols implemented by HelmMac (sensors, control, secrets)
│   │   └── Support/                  # Clock, Logger, IDs, Result helpers
│   ├── HelmPolicy/
│   │   ├── RiskTable.swift           # static tool → GREEN/AMBER/RED
│   │   ├── ShellPolicy.swift         # allow/deny/patterns, destructive detection
│   │   ├── PathPolicy.swift          # roots, exclusions, symlink resolution
│   │   ├── InjectionHeuristics.swift # untrusted content flags
│   │   └── Taint.swift
│   ├── HelmProviders/
│   │   ├── Anthropic/                # Messages API, SSE streaming, tool use, vision
│   │   ├── OpenAICompatible/         # chat/completions, embeddings, streaming; Ollama/LM Studio/OpenAI
│   │   ├── Ollama/                   # /api/tags, /api/embed, model discovery
│   │   └── Common/                   # HTTP client, retries, token accounting
│   ├── HelmStorage/
│   │   ├── Database.swift            # GRDB setup, encryption helpers
│   │   ├── Migrations/
│   │   └── Repositories/             # ConversationRepo, ProjectRepo, MemoryRepo, AuditRepo, IndexRepo, PolicyRepo
│   ├── HelmIndexing/
│   │   ├── Chunkers/                 # code-aware, markdown, plain text
│   │   ├── Parsers/                  # text/markdown/source (PDF parsing lives in the XPC via PDFKit)
│   │   ├── Search/                   # FTS5 + vector hybrid ranking (RRF)
│   │   └── Watch/                    # change detection model (FSEvents adapter in HelmMac)
│   └── HelmMac/                      # macOS-only adapters (compiled on macOS only)
│       ├── Voice/                    # AudioEngine, VAD, AppleSpeech, WhisperSpeech, AVSpeechTTS
│       ├── Vision/                   # CameraSession, VisionAnalyzer, FaceRecognizer, FrameStore
│       ├── Screen/                   # ScreenCaptureService, AXTextReader, WindowInspector
│       ├── Control/                  # AppControl, FileControl, AXControl, KeyboardControl, ShellRunner, AppleScriptRunner, ProcessManager
│       ├── Secrets/                  # KeychainStore
│       ├── System/                   # LocalAuth, Notifications, FSEvents, PowerAssertions
│       └── Indexer/                  # XPC client for HelmIndexer
├── App/
│   ├── project.yml                   # XcodeGen: Helm.app, HelmIndexer.xpc, HelmToolRunner.xpc, tests
│   ├── Helm/
│   │   ├── HelmApp.swift             # @main, scenes, lifecycle
│   │   ├── CommandCentre/            # main window: sidebar, conversation, status rail
│   │   ├── Orb/                      # OrbView, OrbState, shaders
│   │   ├── Conversation/             # message cells: text, transcript, screenshot, camera, file, action, permission, task
│   │   ├── Palette/                  # floating NSPanel + SwiftUI
│   │   ├── MenuBar/                  # MenuBarExtra, Privacy Mode
│   │   ├── Onboarding/               # 11-step wizard
│   │   ├── Screens/                  # Projects, Memory, Permissions, Audit, Automations, Developer, Settings
│   │   ├── Permissions/              # confirmation sheets, RED auth flow
│   │   ├── DesignSystem/             # colours, typography, glass panels, status chips
│   │   ├── Resources/                # Assets.xcassets, Info.plist, Helm.entitlements, Localizable
│   │   └── Support/
│   ├── HelmIndexer/                  # sandboxed XPC service target (PDFKit, NaturalLanguage)
│   ├── HelmToolRunner/               # Phase 11 XPC service target
│   ├── HelmTests/                    # macOS-only tests (adapters, storage, integration)
│   └── HelmUITests/                  # launch + onboarding smoke
├── Tests/                            # SwiftPM tests for HelmCore/Policy/Providers/Indexing (Linux + macOS)
├── docs/
│   ├── DESIGN.md                     # this document
│   ├── SECURITY.md                   # threat model, policies (Phase 11 expands)
│   ├── TOOLS.md                      # tool catalogue with schemas and risk
│   ├── PERMISSIONS.md
│   └── VERIFICATION/                 # per-phase manual checklists for the Mac
├── scripts/                          # bootstrap.sh (brew xcodegen, swiftlint), ci helpers
├── .github/workflows/ci.yml          # macos-15: xcodegen → build → test; ubuntu: core tests (if toolchain allowed)
├── .swift-format · .swiftlint.yml · .gitignore · README.md · LICENSE
```

---

## 5. Security model

### 5.1 Threat model (what we defend against)

| Threat | Defence |
|---|---|
| Prompt injection via files, screen text, web pages, emails, OCR, tool output | Provenance tagging, taint tracking, escalation of risk for tainted turns, injection heuristics, confirmation cards that highlight content-derived arguments |
| Model hallucinating or mis-planning a destructive action | Structured tools only; risk classification; RED requires macOS auth; post-condition verification; dry-run previews for file moves/deletes |
| Another process on the Mac abusing the assistant's privileges | No localhost server; XPC restricted by code-signing requirement; tickets are single-use, request-hash-bound, short-lived |
| Malicious document exploiting a parser | Parsing in sandboxed XPC with read-only bookmarks |
| Secret leakage (API keys, tokens) | Keychain only; redaction filter on everything sent to models and written to logs; `reveal_secret` is RED |
| Biometric data misuse | Templates encrypted (AES-GCM, Keychain key), never leave the Mac, deletable in one click, presence-only semantics |
| Data egress without knowledge | Egress policy per data type (text, files, screen, camera); status rail shows LEAVING MAC indicator; local-only mode |
| Runaway automation | Emergency stop; per-task step budget; process manager kills children; rate limits on proactive notifications |

### 5.2 The Gate (every tool call passes through this, without exception)

```
model emits ToolCall {name, args(JSON)}
   │
   ▼
ToolBroker.validate      → decode against the tool's Codable schema; reject unknown fields, oversize args,
   │                        paths outside roots (after symlink resolution), malformed URLs
   ▼
RiskClassifier.classify  → base risk from RiskTable
   │                        + escalators: taint, path scope, shell pattern, external recipient, size of change
   ▼
PermissionEngine.decide  → policy (DENY / ASK / ALLOW) resolved from: global category setting
   │                        → per-project override → session grants ("allow for this task") → hard rules
   │                        (RED is never ALLOW-able without confirmation; Privacy Mode/E-Stop deny everything)
   ├── ALLOW  → Ticket
   ├── ASK    → UI confirmation card (AMBER) / confirmation + LocalAuthentication (RED) → Ticket or Denied
   └── DENY   → ToolResult(denied) returned to model with reason
   ▼
ToolExecutor.run(ticket) → executes ONLY with a valid ticket; ticket hash must match the request
   ▼
Verifier.check           → tool-specific post-condition (app running? file exists? exit code? git state?)
   ▼
AuditLog.record          → timestamp, request, decision, parameters, result, verification, latency
   ▼
ToolResult → wrapped as untrusted content → back to model
```

### 5.3 Action risk classification

| Level | Default policy | Tools (initial) | Escalators |
|---|---|---|---|
| **GREEN** | ALLOW (visible in conversation + audit) | `read_file`, `search_files`, `semantic_search`, `list_directory`, `capture_screen`, `inspect_window`, `read_window_text`, `capture_camera`, `identify_object`, `open_application`, `open_folder`, `open_project`, `git_status`, `git_log`, `project_info`, `recall_memory`, `list_processes` | Tainted turn → still GREEN if strictly read-only and inside current scope; otherwise AMBER |
| **AMBER** | ASK (remember per-project allowed) | `create_file`, `write_file`, `move_file`, `run_command` (policy-allowed), `run_project_script` (dev/build/test), `install_package`, `edit_project_config`, `type_text`, `click_element`, `press_shortcut`, `open_url`, `compose_email` (draft), `remember` (write memory), `start_dev_server`, `stop_process`, `run_routine` | Tainted turn → always ASK; outside project roots → RED |
| **RED** | ASK + macOS authentication, never auto | `delete_path`, `run_command` matching destructive patterns (`rm -rf`, `sudo`, `curl … \| sh`, `git push --force`, `dd`, `mkfs`, `kill -9 -1`, …), `send_email`, `send_message`, `reveal_secret`, `change_permission_policy`, `install_privileged`, `publish`/deploy, `run_shell_raw` | None; cannot be lowered by policy |

Rules that cannot be overridden by settings: RED always confirms; Privacy Mode and Emergency Stop deny everything; `reveal_secret` never returns a secret into model context (it is shown to you in the UI only).

### 5.4 Untrusted content and prompt injection

1. **Provenance envelope.** Every non-user input is rendered to the model as
   `[BEGIN UNTRUSTED CONTENT source="file:///…/notes.md" kind="file"] … [END UNTRUSTED CONTENT]`
   with a system-prompt rule that such content is data and may contain adversarial instructions.
2. **Taint tracking.** A turn becomes *tainted* when untrusted content enters context. Tool calls in a tainted turn are escalated (GREEN read-only stays GREEN inside scope; everything else becomes ASK). Taint persists for the conversation until you explicitly clear it or start a new conversation.
3. **Argument highlighting.** Confirmation cards highlight any argument substring that appears verbatim in untrusted content ("this URL came from the web page").
4. **Injection heuristics.** Ingested text is scanned for instruction-like patterns; matches add a warning chip to the content card and force ASK for the turn. Heuristics inform; they never silently drop content.
5. **No instruction parsing in code.** No code path ever treats tool output as commands.
6. **Secrets redaction** on every outbound model request and log write (Keychain values, patterns for API keys/tokens).

### 5.5 Data at rest

* `~/Library/Application Support/Helm/helm.sqlite` (0600), WAL mode.
* Encrypted columns: face templates, audit parameter bodies containing file contents, memory exports.
* Index contents respect the exclusion list (see §6) and never include Keychain, browser profile databases, `~/Library` (except explicitly chosen sub-folders), `.env` files by default, SSH keys, or anything matching your exclusion patterns.

### 5.6 Emergency Stop and Privacy Mode

* **Emergency Stop** (global hotkey, proposed `⌃⌥⌘⎋`, configurable): cancels the current turn, drains the task queue, stops audio engine, camera session and screen streams, sets `ControlTools.disabled`, kills managed child processes, shows a full-window **ASSISTANT PAUSED** overlay. Reactivation is a deliberate click in the app, never automatic.
* **Privacy Mode** (menu bar): stops camera, microphone, screen capture and all background observers immediately; conversation by typing still works; tools that need sensors return `denied(privacy_mode)`.

---

## 6. Tool protocol

Provider-agnostic tool definitions are declared once in Swift and rendered to each provider's schema format.

```swift
public struct ToolSpec: Sendable {
    public let name: String                 // snake_case, stable
    public let description: String          // written for the model
    public let parameters: JSONSchema       // generated from the Codable args type
    public let baseRisk: RiskLevel          // GREEN / AMBER / RED
    public let category: PermissionCategory // camera, microphone, screen, accessibility, files, terminal, applications, internet, email, calendar, integrations, memory
    public let sideEffects: Bool
}

public protocol Tool: Sendable {
    associatedtype Args: Codable & Sendable
    associatedtype Output: Codable & Sendable
    static var spec: ToolSpec { get }
    func execute(_ args: Args, context: ToolContext) async throws -> ToolOutcome<Output>
    func verify(_ args: Args, outcome: ToolOutcome<Output>, context: ToolContext) async -> Verification
}
```

Message envelope between model and gate (internal, JSON):

```json
{ "id": "toolu_01…", "name": "open_project", "args": { "project": "Electron" } }
→ { "id": "toolu_01…", "status": "ok|denied|failed|needs_confirmation",
    "risk": "GREEN", "decision": "allow(policy:project=Electron)",
    "output": { "path": "/Users/thomas/Dev/electron", "opened_in": ["Visual Studio Code"] },
    "verification": { "ok": true, "checks": ["VS Code running", "window title contains 'electron'"] },
    "untrusted": false, "duration_ms": 412 }
```

Initial tool catalogue (each with typed args, risk, verification) — full schemas will live in `docs/TOOLS.md`:

`open_application`, `quit_application`, `switch_to_application`, `open_file`, `open_folder`, `open_url`, `open_project`, `search_files`, `semantic_search`, `read_file`, `list_directory`, `create_file`, `write_file`, `move_file`, `delete_path`, `run_command`, `run_project_script`, `start_dev_server`, `stop_process`, `list_processes`, `git_status`, `git_log`, `git_diff`, `capture_screen`, `inspect_window`, `read_window_text`, `capture_camera`, `identify_object`, `read_text_in_view`, `scan_code`, `type_text`, `press_shortcut`, `click_element`, `find_ui_element`, `compose_email`, `send_email`, `lookup_contact`, `calendar_today`, `remember`, `recall_memory`, `forget_memory`, `run_routine`, `run_shortcut`, `ask_user` (structured question), `report_task_progress`.

---

## 7. Permission model

Categories: Camera · Microphone · Screen · Accessibility · Files · Terminal · Applications · Internet (egress) · Email · Calendar · Memory · External integrations.

Each category has a policy `DENY | ASK | ALLOW`, resolved in this order (first match wins):

1. Hard rules (E-Stop, Privacy Mode, RED tools) → cannot be relaxed.
2. Session grant ("Allow for this task" / "Allow for 1 hour") → time-boxed, shown in status rail, revocable.
3. Per-project override (e.g. Terminal = ALLOW inside `Electron` for `npm test` only).
4. Per-application override (e.g. Accessibility control of Terminal = ASK, of 1Password = DENY).
5. Global category policy (defaults: Camera ASK, Microphone ASK, Screen ASK, Accessibility ASK, Files ALLOW-within-roots, Terminal ASK, Applications ALLOW, Internet ASK-per-provider, Email ASK, Calendar ASK, Memory ALLOW-visible, Integrations ASK).

Egress sub-policy (Internet): per data kind — **text**, **files**, **screen captures**, **camera frames** — each `LOCAL_ONLY | ASK | ALLOW` per provider. The status rail shows **LEAVING MAC → Anthropic** whenever a request goes off-device.

Permission Centre also mirrors macOS TCC state (granted / denied / not determined) with a button to the right System Settings pane; the app never pretends a TCC permission exists.

---

## 8. Data model (SQLite)

| Table | Purpose / key columns |
|---|---|
| `conversations` | id, title, project_id?, started_at, taint_state |
| `messages` | id, conversation_id, role (user/assistant/system/tool), kind (text, transcript, screenshot, camera, file, action, permission, task_result), content, provenance(json), model_used, tokens_in/out, latency_ms, created_at |
| `attachments` | id, message_id, type, path/blob ref, sha256, untrusted(bool), ocr_text? |
| `tool_invocations` | id, message_id, tool, args(json), risk, decision, ticket_id, status, output(json), verification(json), started/finished |
| `audit_log` | id, ts, request_text, ai_decision, tool, params(json, encrypted where sensitive), permission_decision, result, success, error, task_id? — FTS5 over text columns |
| `projects` | id, name, description, folder, git_remote, website, dev_cmd, build_cmd, test_cmd, related_apps(json), notes, ai_context, aliases(json), last_opened |
| `project_documents` | project_id, path, label |
| `memories` | id, layer (conversation/working/project/preference/system), project_id?, key, content, source (user/assistant/tool), pinned, disabled, embedding(blob), created/updated, last_used |
| `preferences` | key, value (non-sensitive) |
| `policies` | category, scope (global/project/app), scope_id, policy, constraints(json) |
| `grants` | id, category/tool, scope, expires_at, task_id? |
| `index_roots` | id, path, bookmark(blob), include/exclude patterns, last_scan |
| `index_files` | id, root_id, path, mtime, size, sha256, language, status |
| `index_chunks` | id, file_id, ordinal, text, start_line, end_line, embedding(blob), fts (FTS5 external content) |
| `tasks` | id, title, request, plan(json), state, created/finished |
| `task_steps` | id, task_id, ordinal, description, tool?, status, result(json), verification(json) |
| `routines` | id, name, trigger(phrase/manual), steps(json), project_id? |
| `managed_processes` | id, project_id, command, pid, started_at, exited_at, exit_code, log_path |
| `users` | id, name, enrolled_at, enabled |
| `face_templates` | id, user_id, template(encrypted blob), quality, created_at |
| `providers` | id, kind (anthropic/openai_compatible/ollama), base_url, label, models(json), enabled (keys in Keychain by provider id) |
| `model_roles` | role (default/fast/reasoning/vision/coding/local/transcription/tts/embeddings), provider_id, model |

---

## 9. UI structure

**Command Centre (main window)**
* Left sidebar: Chat · Tasks · Projects · Memory · Automations · Permissions · Audit · Developer · Settings.
* Centre: the **Orb** header (state-driven animation: Idle, Listening, Thinking, Looking, Acting, Speaking, Waiting for permission, Error) above the **Conversation** (message kinds: text, voice transcript with waveform, screenshot card, camera observation card, file card, action card with risk colour + verification tick, permission request card with Approve/Deny/Allow-for-task, task card with live steps and PAUSE/CANCEL/APPROVE/RETRY).
* Bottom composer: text field, push-to-talk button, attach (file / screenshot / camera frame), model chip, project chip.
* Right **Status rail** (always visible, never hidden): MICROPHONE, CAMERA, SCREEN, VOICE, MODEL, MEMORY, plus LEAVING MAC and PRIVACY MODE / PAUSED banners; current task; active session grants.

**Command Palette** — `⌥ Space` (configurable) opens a non-activating floating `NSPanel`; type a request, see the answer/task inline; `⌘↩` to hand off to the full window. Works when the main window is closed.

**Menu bar** — status glyph (mirrors orb state), toggles for microphone/camera/screen, current task line, Quick command, Pause assistant, **PRIVACY MODE**.

**Onboarding** — the 11 steps you specified; each permission step performs the real request and shows the real TCC result; the final step runs a live round-trip (model ping, STT of a spoken phrase, TTS reply, optional camera frame) and only then shows **YOUR ASSISTANT IS READY**.

**Developer panel** — model per role, context size, streaming token counts, latency per stage (VAD → STT → model → tool → TTS), tool call/result log, memory retrieval hits, vision requests with thumbnails, speech state machine, background processes, audio level meters.

**Design system** — dark glass panels on near-black, one accent colour, monospaced status chips in capitals, subtle motion; light mode supported. Premium but restrained; no decorative controls.

---

## 10. Implementation roadmap

Each phase ends with: CI green on the macOS runner → manual checklist for the Mac mini (`docs/VERIFICATION/phase-N.md`) → commit. A phase is "done" only when you confirm its hardware checklist.

| Phase | Deliverables | What I can verify from here | What needs your Mac |
|---|---|---|---|
| **0 — Scaffold & CI** | SwiftPM packages, XcodeGen project, entitlements/Info.plist, design system, GitHub Actions macOS workflow, lint/format, README | CI build + unit tests | Xcode opens the project; app launches |
| **1 — Desktop shell + Command Centre** | Main window, sidebar, Orb (all 8 states, driven by a state machine), conversation list with all message kinds, status rail, menu bar extra with Privacy Mode + Pause, command palette panel + global hotkey, Emergency Stop hotkey + PAUSED overlay, onboarding wizard frame (steps 1, 2, 11 functional; permission steps wired in later phases) | Compilation, state-machine tests, UI smoke test | Hotkeys, panel behaviour, animation feel |
| **2 — AI conversation** | Anthropic + OpenAI-compatible + Ollama providers (streaming, tool-use, vision message types), Model Router with roles and egress policy, Keychain store, Settings → Providers, conversation persistence (GRDB), ContextBuilder with provenance envelopes and taint, `ask_user` tool, Developer panel basics | Provider clients against recorded fixtures; router tests; taint tests | Live API round-trip; Ollama detection |
| **3 — Voice** | AudioEngine, VAD, Apple on-device STT (streaming), AVSpeech TTS with barge-in, push-to-talk / wake phrase / open-conversation modes, transcript cards, mic state everywhere. Optional whisper.cpp provider | State machine + VAD unit tests on synthetic audio | Latency, interruption, wake-phrase accuracy |
| **4 — Webcam vision** | CameraSession, FrameStore (recent frames, full-res on demand), Vision analyzers (text, barcode/QR, document, classification), `capture_camera`/`identify_object`/`read_text_in_view`/`scan_code` tools, conversational follow-ups ("which model?") using retained frames and crops, optional enrolment + presence recognition (encrypted templates, self-test gate) | Analyzer pipeline with fixture images (Vision only runs on macOS CI) | Real objects; enrolment; presence accuracy |
| **5 — Screen awareness** | ScreenCaptureService (display / window / app / manual / temporary live), AX text reader, `capture_screen`/`inspect_window`/`read_window_text`, SCContentSharingPicker consent flow, screenshot cards, redaction of secure text fields | Tool schemas, policy | Screen Recording TCC; terminal-error reading |
| **6 — Project & file intelligence** | Project Registry (CRUD UI, aliases, commands), indexer XPC (sandboxed), chunkers, FTS5 + embeddings hybrid search, exclusion rules, incremental FSEvents updates, `semantic_search`/`search_files`/`read_file`/`open_project`/`git_*` tools, "where did we leave off" summariser | Chunking/ranking tests; index repo tests | XPC + bookmarks; indexing your real folders |
| **7 — Tool execution & Task Engine** | Full Gate (broker, classifier, permission engine, tickets, verifier, audit), Permission Centre UI, Audit Log UI with search, Task Engine with plan cards, PAUSE/CANCEL/APPROVE/RETRY, step verification, `run_project_script`, `run_command` with ShellPolicy, ProcessManager for dev servers | Extensive: gate, policy, task state machine, shell policy fixtures | Real commands; confirmation flows; RED auth |
| **8 — Computer control** | AppControl, FileControl, AX inspect/find/click, KeyboardControl (type/shortcut), AppleScript allow-list (Mail compose, Finder), `compose_email`/`send_email` (RED), Contacts lookup, per-app Accessibility overrides | Arg validation, policy | Accessibility TCC; real UI interaction |
| **9 — Memory** | Five layers, `remember`/`recall_memory`/`forget_memory`, retrieval in ContextBuilder, Memory screen (search/view/edit/delete/pin/disable), global disable, export/erase | Retrieval and scoping tests | UX |
| **10 — Automation & proactive** | Routines (named step lists, run via palette/voice), Automations screen, watchers (git status, managed process exit, build failure, TODO scan, certificate expiry for registered docs), notification policy with quiet hours, suggestion cards | Watcher logic with fakes | Notifications; real routines |
| **11 — Security hardening** | HelmToolRunner XPC, code-signing requirements, secrets redaction audit, injection heuristics tuning, path-policy fuzz tests, threat-model doc, RED auth coverage review, privacy-mode kill-switch audit, dependency pinning | Fuzz/property tests | Penetration checklist on the Mac |
| **12 — Testing & packaging** | Full test pass, XCUITest onboarding, signing + notarization script, Sparkle-free manual update path, release checklist, user guide | CI | Notarized build installs and keeps TCC grants across updates |

Estimated shape: Phases 0–2 are the foundation and go first without exception; 3–5 (senses) and 6–8 (hands) are independent of each other after Phase 2 and can be reordered if you want, e.g., project awareness before voice.

---

## 11. Decisions I need from you

1. **Approve the stack**: Swift 6 / SwiftUI, macOS 15 minimum, unsandboxed Developer-ID app with sandboxed XPC parser, SwiftPM + XcodeGen, GRDB. (Recommended: yes.)
2. **Build verification**: use **GitHub Actions macOS runners** as the authoritative build. If the repo stays private, the free tier gives ~200 macOS minutes/month; making it public removes the limit. Optionally add `download.swift.org` to this environment's allowed domains so I can compile and test the core packages locally as well. (Recommended: macOS runners + allow the domain.)
3. **Tell me about the Mac mini**: macOS version, Xcode version, Apple Silicon generation, whether you have an Apple Developer account (or at least an Apple ID signed into Xcode for a development certificate), and whether Ollama or LM Studio is installed.
4. **First providers**: Anthropic + OpenAI-compatible (covers OpenAI and local Ollama/LM Studio). Gemini and others later. (Recommended.)
5. **Name**: keep `Helm` as the repository/target codename, or give me another.
6. **Phase order**: the 12-phase order above, or pull project/file intelligence (Phase 6) ahead of voice/vision.

On approval I begin Phase 0 and Phase 1 immediately.
