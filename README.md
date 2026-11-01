# FerroDeck

**FerroDeck** is the unified multi-vendor agent cockpit Progressive Web App (PWA) and backing
HTTP/1.1 + RFC 6455 WebSocket daemon (`fl-hub`) for the [FerroLoop](https://github.com/Artificial-Humanity/FerroLoop)
agentic orchestration platform.

Designed for desktop (macOS / Linux) and touch-first tablets and phones (iPadOS / iOS), FerroDeck
gives operators a single real-time command center to supervise, steer, inspect, and govern
concurrent coding agents across five subscription CLI runtimes (`claude`, `codex`, `antigravity` /
`agy`, `muse`, and `grok`), API-key providers via **FerroGate**, and local inference engines
(`Ollama` and `Lemonade`).

---

## Architecture Overview

FerroDeck consists of two tightly coupled layers that share an exact 1:1 wire contract:

1. **`ui/` — Command-Center PWA (`React 19 + TypeScript + Vite`):**
   An installable desktop and iPadOS/iOS Progressive Web App with zero translation layer over
   `/api/v1/*` and `/ws/v1`.
2. **`crates/hub` — `fl-hub` HTTP/1.1 & WebSocket Daemon (`Rust 2024`):**
   A self-hosted server providing versioned `/api/v1/*` JSON REST endpoints, `/ws/v1` multiplexed
   real-time event and interactive PTY terminal streams, static SPA serving, two-tier authority
   enforcement (`HubAuthority::Owner` and `HubAuthority::Peer`), WebAuthn P-256 passkey and TOTP
   authentication, host session/transcript discovery, and bounded artifact/document inspection.

---

## Command-Center Capabilities

### 1. Three-Tier Navigation & Responsive Split Cockpit
* **Left Roster Sidebar (`RosterSidebar.tsx`):** Project-grouped agent roster with live heartbeat
  status (`alive`, `stale`, `dead`), role badges (`lead`, `assistant`, `lab_manager`, `custom`),
  active runtime adapter pills (`claude`, `codex`, `agy`, `muse`, `grok`), user-directed unread
  turn badges, and one-click session spin-up (`+ Spin Up Agent`).
* **Top Context Header (`ContextHeader.tsx`):** Selected agent identity, session switcher,
  working directory indicator, authority badge, Light/Dark theme toggle, **Split-View dock toggle**
  (pinning live Chat on the left while inspecting Artifacts, Git/PRs, Board, Mail, Terminal,
  Vault/Gate, FerroState, or Hub on the right), Nodes modal launcher, Settings modal, and
  contextual `?` Help drawer.
* **Mobile / iPadOS Ergonomics:** Standalone PWA manifest, safe-area inset handling, iPadOS Safari
  AutoFill suppression on tactical inputs, per-session scroll memory, and a sticky bottom
  navigation bar on compact viewports.

### 2. Live Multi-Vendor Chat, Thinking & Tool Blocks (`ChatPane.tsx`)
* **Normalized Multi-Vendor Transcripts:** Real-time turn-by-turn rendering across Claude Code,
  OpenAI Codex, Google Antigravity, Meta Muse, and xAI Grok sessions (`docs/vendor-runtimes.md`).
* **Structured Turn Cards (`TurnCard.tsx`, `ThinkingBlock.tsx`, `ToolCallBlock.tsx`):**
  * Collapsible extended-thinking / reasoning blocks and tool-call execution cards.
  * Distinct visual treatment for operator prompts, user-directed assistant replies (teal accent),
    and collapsible inter-agent `Wire · <sender>` messages (warm bronze outline).
  * Clickable `file://` chips, inline Mermaid diagram rendering (`MermaidDiagram`), and image/file
    attachments that open directly in the Artifact Studio.
* **Operator Message Queue & Context Telemetry:**
  * Live **Context Level Meter** (`used_tokens / max_tokens`, supporting up to `1M` context windows)
    with a one-click `Compact Context` action when utilization reaches `>= 50%`, plus vendor
    account quota/allowance windows (`5h` primary and `7d` secondary windows).
  * Per-session FIFO **Operator Message Queue** when an agent is busy, with inline editing,
    cancellation, **Send Now**, and **Always send immediately** controls, plus slash-command
    autocomplete (`/compact`, `/clear`, `/model`, `/goal`, `/plan`, `/boost`, `/code-review`,
    `/subagent-dev`).

### 3. Full Artifact Studio (`ArtifactPreviewPane.tsx` & `crates/hub/src/artifacts.rs`)
* **Multi-Format Rendering:**
  * **Markdown & Rich GFM** (`RichMarkdown.tsx`) withRendered/Source toggle and clickable internal
    artifact links.
  * **Mermaid Diagrams & SVGs** and **Images** (`PNG`, `JPG`, `WEBP`, `GIF`) wrapped in an
    interactive **Zoom & Pan Canvas** (`25%`–`400%` zoom presets, drag-to-pan, `Fit` reset).
  * **PDF Documents** via sandboxed `<iframe />` preview with strict `Content-Security-Policy:
    script-src 'none'; object-src 'none'`.
  * **MS Office Structured Previews (`.docx`, `.xlsx`, `.pptx`):** Pure-Rust, zero-unsafe bounded
    ZIP/XML extraction (`GET /api/v1/artifacts/office-preview`, capped at a 2 MiB text budget with
    explicit `truncated` indicators) rendering Word headings/paragraphs/tables (`OfficeDocxView`),
    multi-sheet Excel grids with sticky `A..Z` column and row headers (`OfficeXlsxView`), and
    16:9 PowerPoint slide decks with thumbnails and speaker notes (`OfficePptxView`).
  * **Source Code Viewer** with line numbers.
* **Artifact Registry, Archival & Version History:**
  * Automatic archival of user-facing artifacts (`~/.local/share/fl/artifacts/managed/`) so deliverables
    survive git worktree teardown, paired with a `Prune Dead Paths` action.
  * Content-addressed SHA-256 **Version History** (`GET /api/v1/artifacts/versions` and
    `/api/v1/artifacts/version-raw`) with historical snapshot inspection and side-by-side line
    diffs (`ArtifactDiffView`) between any two revisions.

### 4. Scrum / Kanban Product Backlog (`RecordsBoardPane.tsx` & `crates/hub/src/records.rs`)
* Project-scoped board (`Todo`, `Doing`, `Done`) with completion percentage, open findings count,
  attempt telemetry, and total cost summary.
* Inline record creation (`POST /api/v1/records`) and state transitions (`POST /api/v1/records/:id/move`)
  backed by FerroLoop's gated transition engine.

### 5. Git Worktrees & Pull-Request Governance (`GitPrPane.tsx`, `DiffViewer.tsx`)
* **Worktree Inspector:** Live `git worktree` status (`GET /api/v1/worktrees`) and unified or
  side-by-side syntax-highlighted diffs (`GET /api/v1/worktrees/:id/diff`).
* **Tiered PR Governance:** Inspects open pull requests (`GET /api/v1/prs`) and executes
  role-gated pull-request reviews (`POST /api/v1/prs/:number/review`) and author-owned merges
  (`POST /api/v1/prs/:number/merge`) under `fl_core::pr_policy`.

### 6. FerroWire Mail Center (`WireMailPane.tsx`)
* Real-time inter-agent message bus (`GET /api/v1/wire/messages`, `POST /api/v1/wire/send`,
  `POST /api/v1/wire/ack`) with per-agent inbox, unread filtering, thread inspection, urgent
  delivery flags, and live `/ws/v1` (`channel: "wire"`) push updates.

### 7. FerroGate Zero-Knowledge Vault & API-to-MCP Gateway (`VaultGatePane.tsx`)
* Inspects and provisions `ring` AES-256-GCM encrypted vault secrets (`GET / POST / DELETE
  /api/v1/gate/secrets`) with AEAD AAD host/method/agent policy binding and lease TTLs—without
  ever exposing plaintext secret values to the browser.
* Displays declarative API-to-MCP connectors and inbound webhook bridges (`GET /api/v1/gate/connectors`)
  plus local MCP Registry `v0.1` endpoints (`/api/v1/gate/registry/v0.1/...`).

### 8. FerroState Memory, Context & Code Index (`FerroStatePane.tsx`)
* Browse, filter, and upsert scoped (`workspace`, `project:<slug>`, `agent:<slug>`) memory,
  handoff, and runtime entries (`GET / POST /api/v1/state`) with automatic `[REDACTED]` masking
  for non-owner sessions.
* Weighted BM25 state search (`GET /api/v1/state/search`), structural Rust/TypeScript symbol &
  outline search with token-savings ledger (`GET /api/v1/state/code-search`), git session
  checkpoints (`POST /api/v1/state/checkpoint`), and bounded workspace state import
  (`POST /api/v1/state/import-workspace`).

### 9. 5-Provider Runtime Matrix & Multi-Node Federation (`NodesModal.tsx`, `runtime_matrix.rs`, `nodes.rs`)
* **5-Provider & Runtime Testing Matrix (`GET /api/v1/runtimes/matrix`, `POST /api/v1/runtimes/probe`):**
  Owner-only concurrent health probe across:
  1. Subscription CLI runtimes (`claude`, `codex`, `grok`, `agy`, `muse`) — binary discovery,
     bounded `--version` check, active route count, and live session count.
  2. `FerroGate` Vault API keys (`anthropic`, `openai`, `xai`, `gemini`, `openrouter`) — metadata
     provisioning and expiration status (`ready`, `expired`, `missing`).
  3. Local inference engines (`ollama` on `:11434`, `lemonade` on `:8000` / `:8097`) — loopback
     reachability, latency, and discovered model tags.
* **Multi-Node Enrollment & Federated Task Dispatch:**
  One-time worker enrollment tokens (`POST /api/v1/nodes/enroll-token`) with live TTL countdown,
  environment profile badges (`ai-lab`, `macos-xcode`, `cloud-worker`, `linux-worker`), subsystem
  federation pills (`ferrowire`, `ferrostate`, `ferrogate`, `ferrostep`, `ferrodeck`), worker
  heartbeat telemetry (`POST /api/v1/nodes/:id/heartbeat`), and federated task dispatch & queue
  inspection (`POST /api/v1/nodes/:id/dispatch`, `GET /api/v1/nodes/:id/tasks`).

### 10. Session Lifecycle, Model Switching & Identity Management
* **Session Spin-Up & Decommission (`SpinUpAgentModal.tsx`, `AgentSettingsModal.tsx`, `lifecycle.rs`):**
  Launch new hosted tmux or local agent sessions (`POST /api/v1/sessions/spin-up`), switch active
  models and thinking levels (`POST /api/v1/sessions/:id/model`), interrupt in-flight turns
  (`POST /api/v1/sessions/:id/interrupt`), clear or reforge context (`POST /api/v1/sessions/:id/clear`),
  run bulk session actions (`POST /api/v1/sessions/bulk-action`), and gracefully prepare or tear
  down sessions (`POST /api/v1/sessions/:id/decommission`).
* **Identity, Passkeys, TOTP & Cloud Adapters (`PairingModal.tsx`, `SettingsModal.tsx`, `operators.rs`):**
  One-time pairing codes (`POST /api/v1/auth/pair`), operator accounts with RBAC roles
  (`owner`, `admin`, `operator`, `viewer`), WebAuthn P-256 passkey registration and challenge/assertion
  login (`IndexedDB` non-extractable `CryptoKey` storage), TOTP MFA enrollment/verification, and
  cloud adapter sync settings (`/api/v1/cloud/adapters`).

---

## Repository Layout

```text
FerroDeck/
├── Cargo.toml                  # Rust workspace manifest for `fl-hub`
├── crates/
│   └── hub/                    # `fl-hub` HTTP/1.1 + RFC 6455 WebSocket server crate
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs          # Server loop, HubState, store guard, and integration suite
│           ├── api.rs          # `/api/v1/*` REST handlers and static SPA serving
│           ├── artifacts.rs    # Artifact registry, archival, version snapshots, and OOXML reader
│           ├── auth.rs         # Owner/Peer authority, pairing codes, cookies, and Tailscale trust
│           ├── cloud_adapters.rs
│           ├── commands.rs     # Slash-command catalog
│           ├── direct_quota.rs # Bounded vendor CLI and OAuth quota probes
│           ├── discovery.rs    # Multi-vendor session, transcript, context, and worktree discovery
│           ├── lifecycle.rs    # Session spin-up and decommission orchestration
│           ├── nodes.rs        # Multi-node enrollment, heartbeats, and federated task queue
│           ├── operators.rs    # Operator RBAC, password, TOTP, and WebAuthn P-256 passkeys
│           ├── records.rs      # Scrum/Kanban backlog record endpoints
│           ├── runtime_matrix.rs # 5-Provider & Runtime Testing Matrix probes
│           ├── session_actions.rs # Interrupt and live model switching
│           ├── settings.rs     # Hub settings persistence
│           ├── terminal.rs     # Owner-only interactive PTY terminal manager
│           └── ws.rs           # RFC 6455 WebSocket handshake, framing, and `/ws/v1` multiplexer
├── ui/                         # FerroDeck React 19 + TypeScript + Vite PWA
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── index.html
│   ├── public/
│   └── src/
│       ├── App.tsx
│       ├── api/                # Typed `HubApiClient` (`/api/v1/*`) and `HubWsClient` (`/ws/v1`)
│       ├── components/         # Cockpit panes, modals, drawers, and studio viewers
│       ├── lib/                # URL safety and token-attachment guards (`safeUrl.ts`)
│       ├── styles/             # Industrial cockpit design tokens (`tokens.css`)
│       ├── types/              # 1:1 TypeScript wire contract (`api.ts`, `msp.d.ts`)
│       └── __tests__/          # Vitest + React Testing Library test suite
└── docs/
    ├── vendor-runtimes.md      # On-disk session, transcript, context, and quota formats per vendor
    ├── 2026-10-27-ferrodeck-ui-design.md
    ├── 2026-10-27-hub-and-ferrodeck-design.md
    ├── 2026-10-30-discovery-lifecycle-cloud-audit.md
    └── 2026-10-31-ferrodeck-ui-cloud-audit.md
```

---

## Building and Verifying

### Prerequisites

* **Rust 1.98+** (edition 2024) and Cargo
* **Node.js 24+** and `npm`

### 1. Verify the Rust Hub Crate (`fl-hub`)

```console
cargo fmt --all --check
cargo clippy --all-targets --workspace -- -D warnings
cargo test --workspace
```

### 2. Build and Test the FerroDeck PWA (`ui/`)

```console
cd ui
npm install
npm test
npm run build
```

The production SPA bundle is emitted to `ui/dist/`, ready to be served by `fl hub serve --ui-dir ui/dist`.

---

## Documentation

* [docs/vendor-runtimes.md](docs/vendor-runtimes.md) — On-disk transcript, liveness, context window, and account quota shapes for Antigravity, Claude Code, OpenAI Codex, xAI Grok, and Meta Muse.
* [docs/2026-10-27-ferrodeck-ui-design.md](docs/2026-10-27-ferrodeck-ui-design.md) — FerroDeck Unified Agent Cockpit UI specification.
* [docs/2026-10-27-hub-and-ferrodeck-design.md](docs/2026-10-27-hub-and-ferrodeck-design.md) — `fl-hub` REST, WebSocket, authentication, and static serving specification.

---

## License

Licensed under [Apache-2.0](LICENSE).
