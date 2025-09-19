# Gemini CLI Technical Documentation

## Overview
Gemini CLI is a multi-package TypeScript monorepo that ships an interactive terminal client for Google’s Gemini models alongside shared core services, integration tooling, and editor companions.【F:package.json†L1-L58】【F:docs/index.md†L1-L38】The project targets Node.js 20+, bundles the CLI into a single executable entry point, and exposes build, test, lint, and release automation through workspace-aware npm scripts.【F:package.json†L4-L54】

## Repository structure
### Packages
- **`packages/cli`** – the user-facing CLI written with React Ink. It re-exports a `gemini` binary, depends on `@google/gemini-cli-core`, and pulls in Ink, yargs, Zod, and other UX dependencies.【F:packages/cli/package.json†L1-L60】Its main entry point (`src/gemini.tsx`) boots the Ink UI, loads configuration, handles sandbox relaunches, and mounts the application providers that drive the session UX.【F:packages/cli/src/gemini.tsx†L7-L200】
- **`packages/core`** – the backend orchestration layer that owns configuration, model access, tool execution, telemetry, and file/IDE services. It is published as `@google/gemini-cli-core` and exports numerous subsystems such as config, prompts, tools, telemetry, and IDE helpers.【F:packages/core/package.json†L1-L87】【F:packages/core/src/index.ts†L7-L96】
- **`packages/a2a-server`** – an auxiliary HTTP service that wraps the CLI core in an Agent-to-Agent (A2A) protocol via Express, supporting persistence backends and task management.【F:packages/a2a-server/package.json†L1-L33】【F:packages/a2a-server/src/http/app.ts†L7-L200】
- **`packages/vscode-ide-companion`** – a VS Code extension that exposes Gemini CLI commands, keybindings, and diff tooling to the editor, activated on startup.【F:packages/vscode-ide-companion/package.json†L1-L92】
- **`packages/test-utils`** – shared helpers for Vitest-based unit tests used across workspaces.【F:packages/test-utils/package.json†L1-L13】

### Supporting directories
- **`docs/`** – end-user and contributor documentation that mirrors the code structure (architecture, CLI, core, tools, deployment, telemetry, etc.).【F:docs/index.md†L5-L38】
- **`integration-tests/`** – cross-package Vitest suites that exercise CLI workflows, with a five-minute timeout, global setup/teardown to manage sandbox state, and retry logic for flaky environments.【F:integration-tests/vitest.config.ts†L7-L23】【F:integration-tests/globalSetup.ts†L7-L99】
- **`scripts/`** – automation for building, bundling, sandbox management, telemetry, and release preparation. The primary build script installs dependencies if needed, builds all workspaces, and optionally produces sandbox container images.【F:scripts/build.js†L28-L55】
- **`bundle/`** – generated output. The CLI binary published to npm resolves to `bundle/gemini.js` via the root `bin` map.【F:package.json†L56-L58】

## Build and distribution pipeline
The root scripts orchestrate workspace builds (`npm run build --workspaces`), bundling (esbuild + asset copy), linting, type-checking, and multi-environment integration tests, enabling reproducible release artifacts.【F:package.json†L26-L41】`npm run preflight` chains clean install, formatting, linting, build, type checks, and CI tests to enforce quality gates before submission.【F:package.json†L47-L54】【F:GEMINI.md†L1-L11】Development entry (`npm run start`) spawns `scripts/start.js`, which runs a build-status check, resolves sandbox commands, and launches the CLI workspace with debugging awareness.【F:package.json†L19-L23】【F:scripts/start.js†L20-L76】

## CLI runtime architecture
### Entry point and startup
`src/gemini.tsx` drives process lifecycle: it parses command-line flags, loads layered settings, optionally launches a sandbox, and starts either interactive Ink rendering or non-interactive execution depending on arguments.【F:packages/cli/src/gemini.tsx†L10-L200】It also manages memory-based relaunches, DNS resolution validation, update checks, and registers cleanup handlers for checkpoints and telemetry.【F:packages/cli/src/gemini.tsx†L62-L157】

### Configuration resolution
`parseArguments` composes the CLI’s option surface (models, sandbox toggles, telemetry targets, tool allowlists, MCP controls, output format, etc.) using yargs, while integrating with settings-driven defaults and extension commands.【F:packages/cli/src/config/config.ts†L7-L204】Settings are merged from system, user, and workspace scopes before being passed to the core `Config` object that will own tool registration and model routing.【F:packages/cli/src/config/config.ts†L20-L204】

### Initialization and IDE mode
Before rendering the UI, `initializeApp` authenticates the selected provider, validates themes, determines whether to prompt for login, and, when IDE mode is enabled, establishes a connection via the core IDE client and records the start event.【F:packages/cli/src/core/initializer.ts†L18-L56】

### Interactive session shell
`startInteractiveUI` wraps the Ink application in React context providers for settings, session statistics, Vim keybindings, and raw keypress handling, then renders `AppContainer` which manages the conversational UI and tool approvals.【F:packages/cli/src/gemini.tsx†L180-L200】It sets the terminal window title, enables kitty keyboard protocol detection, and ensures version metadata is available for the UI footer.【F:packages/cli/src/gemini.tsx†L187-L199】

### Non-interactive execution
`runNonInteractive` executes prompts provided via `-p`, STDIN, or `--prompt-interactive`. It streams model output (plain text or JSON), accumulates tool call requests, dispatches them through the core `executeToolCall`, and feeds tool responses back into the conversation loop until a final answer is produced.【F:packages/cli/src/nonInteractiveCli.ts†L30-L146】The routine patches console output to prevent interference, enforces session turn limits, surfaces tool errors, and gracefully shuts down telemetry at completion.【F:packages/cli/src/nonInteractiveCli.ts†L36-L154】

### Extensions, policy, and MCP configuration
CLI extensions are loaded from user and workspace scopes, hydrated with environment variables, and filtered by an enablement manager so that only active definitions contribute MCP servers, context files, and tool exclusions.【F:packages/cli/src/config/extension.ts†L36-L167】Policy configuration builds a prioritized rule set that controls whether tool invocations are auto-approved, require confirmation, or are denied, factoring in approval modes, MCP server trust, and explicit allow/deny lists.【F:packages/cli/src/config/policy.ts†L7-L182】

## Core runtime architecture
### Configuration orchestrator
The `Config` class in `@google/gemini-cli-core` centralizes runtime state: workspace paths, tool registries, prompt registries, telemetry settings, sandbox configuration, approval modes, MCP server definitions, and shell execution preferences.【F:packages/core/src/config/config.ts†L253-L449】Initialization wires the file discovery service, checkpoint-aware Git integration, prompt registry, tool registry, and Gemini client before any chat traffic occurs.【F:packages/core/src/config/config.ts†L452-L469】Telemetry is configurable per target (local file vs. GCP OTLP), can log prompts when enabled, and is initialized automatically when telemetry is turned on.【F:packages/core/src/config/config.ts†L366-L374】【F:packages/core/src/config/config.ts†L440-L445】Proxy support leverages Undici’s `ProxyAgent` to route outbound model requests.【F:packages/core/src/config/config.ts†L444-L445】

### Tool registry and discovery
`createToolRegistry` registers core tools (ls, read file, ripgrep/grep fallback, glob, smart edit or traditional edit, write file, web fetch, read-many-files, shell, memory, web search) while honoring allowlists, denylists, and smart-edit preferences.【F:packages/core/src/config/config.ts†L980-L1052】`ToolRegistry` manages declarative tool metadata, executes project-defined discovery commands, and integrates with MCP servers via `McpClientManager`, dynamically removing and reloading discovered tools as configurations change.【F:packages/core/src/tools/tool-registry.ts†L169-L299】It also replays tool discovery across MCP servers, supports per-server restarts, and namespaces discovered prompts through the `PromptRegistry`.【F:packages/core/src/tools/tool-registry.ts†L231-L291】【F:packages/core/src/prompts/prompt-registry.ts†L9-L74】

### File system and workspace context
`FileDiscoveryService` respects `.gitignore` and `.geminiignore` patterns when presenting files to the model or tools, and can return filtering reports for telemetry or UI feedback.【F:packages/core/src/services/fileDiscoveryService.ts†L14-L140】`WorkspaceContext` tracks multiple workspace roots, validates directories, and exposes helpers to check whether paths remain inside trusted workspace boundaries, supporting multi-root projects and include-directory overrides.【F:packages/core/src/utils/workspaceContext.ts†L15-L198】

### Content generation and authentication
`createContentGenerator` encapsulates authentication across OAuth, Gemini API keys, and Vertex AI credentials, sets user-agent headers, and wraps the Google GenAI client in a logging decorator that records requests for telemetry. Usage statistics can inject installation identifiers into requests when enabled.【F:packages/core/src/core/contentGenerator.ts†L23-L151】Helper `createContentGeneratorConfig` inspects environment variables to determine which auth strategy is valid for the current session.【F:packages/core/src/core/contentGenerator.ts†L58-L99】

### Conversation loop and model routing
`GeminiClient` maintains conversation history, enforces token limits, and performs chat compression when history exceeds configured thresholds, emitting events that the UI can surface.【F:packages/core/src/core/client.ts†L127-L200】【F:packages/core/src/core/client.ts†L443-L472】Its `sendMessageStream` generator streams model output, detects pending tool calls, injects IDE context when necessary, orchestrates loop detection, and cooperates with routing strategies to select models per turn.【F:packages/core/src/core/client.ts†L443-L520】History management includes IDE-aware context syncing and stripping thoughts when switching auth flows.【F:packages/core/src/core/client.ts†L163-L199】

### Tool execution flow
`CoreToolScheduler` tracks tool call lifecycles (validation, approval waiting, execution, cancellation, completion), formats function responses, coordinates modifiable tools (e.g., editor-backed edits), and enforces approval policies before executing file- or shell-affecting tools.【F:packages/core/src/core/coreToolScheduler.ts†L7-L143】Tool results are converted to Gemini function responses, with output truncation telemetry raised when necessary.【F:packages/core/src/core/coreToolScheduler.ts†L146-L200】

### Shell command execution
`ShellExecutionService` abstracts process spawning through `node-pty` (or falls back to child processes) to provide streaming ANSI-aware output, binary detection, abort handling, and consistent environment wiring across platforms.【F:packages/core/src/services/shellExecutionService.ts†L25-L149】Terminal buffers are serialized for UI presentation, and the service tracks active PTYs to support cancellation and cleanup.【F:packages/core/src/services/shellExecutionService.ts†L18-L98】【F:packages/core/src/services/shellExecutionService.ts†L117-L149】

### Telemetry and confirmation bus
Telemetry exports include OTLP exporters for logs, metrics, and traces, UI telemetry aggregation, activity detection, and high-water mark tracking, allowing sessions to log prompts, tool calls, and compression events.【F:packages/core/src/telemetry/index.ts†L7-L68】The `MessageBus` integrates the policy engine with the UI: tool confirmation requests are either auto-approved, denied, or routed to the client depending on policy decisions, and policy rejections emit structured events for logging.【F:packages/core/src/confirmation-bus/message-bus.ts†L7-L82】

## Tooling capabilities
Out of the box, Gemini CLI exposes file system exploration (`ls`, `read_file`, `read_many_files`), text search (`ripgrep` or `grep`, `glob`), editing (`edit`, `smart_edit`, `write_file`), shell execution, memory persistence, and web fetch/search tools. These tool builders are registered automatically during configuration, with optional ripgrep detection and smart-edit toggles, and can be extended via discovered MCP tools or custom discovery commands.【F:packages/core/src/config/config.ts†L1017-L1049】【F:packages/core/src/tools/tool-registry.ts†L231-L299】Prompt discovery mirrors tool discovery so that MCP servers can advertise prompt templates alongside tool schemas.【F:packages/core/src/prompts/prompt-registry.ts†L9-L52】

## Sandbox and environment management
Sandbox support is surfaced through CLI flags (`--sandbox`, `--sandbox-image`), environment variables, and settings, and the build pipeline can optionally build sandbox container images when the sandbox command is available.【F:packages/cli/src/config/config.ts†L89-L204】【F:scripts/build.js†L37-L52】The CLI’s startup script probes for sandbox availability to forward debugger flags appropriately, ensuring consistent behavior inside or outside containers.【F:scripts/start.js†L37-L55】Shell execution honors sandbox configuration through the core config’s shell execution settings and can disable PTY usage when running in constrained environments.【F:packages/core/src/config/config.ts†L410-L424】

## Integration ecosystem
### CLI extensions and MCP
Extensions can contribute MCP servers, context files, and tool policies. The loader supports git-based installs, GitHub releases, and local copies, persisting install metadata and migrating workspace extensions into user scope when necessary.【F:packages/cli/src/config/extension.ts†L36-L125】Enabled extensions are deduplicated per name and gated by trusted-folder checks, while MCP server declarations are merged with workspace-provided definitions before configuration initialization.【F:packages/cli/src/config/extension.ts†L138-L197】

### IDE integrations
Interactive mode can run as an IDE client, connecting to editor hosts via the core `IdeClient` on startup.【F:packages/cli/src/core/initializer.ts†L45-L48】For the Zed editor, `runZedIntegration` hosts an Agent Context Protocol bridge that reuses CLI configuration, negotiates authentication, mounts the remote file system service, and proxies tool calls and streaming responses back to the editor.【F:packages/cli/src/zed-integration/zedIntegration.ts†L7-L198】The VS Code companion extension activates on startup, contributes commands (accept/cancel diffs, run CLI, show notices), declares keybindings, and bundles an express-based local server to relay MCP traffic.【F:packages/vscode-ide-companion/package.json†L1-L92】

### A2A server
The A2A server exposes Gemini CLI as a networked agent with task creation, metadata retrieval, and optional Google Cloud Storage persistence. It configures task stores per deployment (in-memory vs. GCS), integrates the core executor, and surfaces REST endpoints for clients to create tasks and query metadata.【F:packages/a2a-server/src/http/app.ts†L7-L200】

## Testing and quality assurance
Each workspace ships Vitest unit suites, while the root scripts provide aggregated `npm run test --workspaces` and CI variants that chain integration tests under multiple sandbox backends.【F:package.json†L34-L41】Integration tests prepare isolated directories, manage memory files, and clean up after runs to avoid state leakage.【F:integration-tests/globalSetup.ts†L31-L99】Developers are expected to run `npm run preflight` before submitting changes to ensure formatting, linting, builds, type checks, and tests all succeed.【F:GEMINI.md†L1-L24】

## Documentation resources
The `docs/` tree mirrors the architecture, providing deep dives into CLI usage, configuration, tools, IDE integration, telemetry, releases, and deployment guidance, serving as the canonical reference for contributors and operators.【F:docs/index.md†L5-L38】Complementary contributor guidance (coding style, test practices, tooling) lives in the root `GEMINI.md` and `CONTRIBUTING.md`, reinforcing consistent development workflows.【F:GEMINI.md†L13-L136】

