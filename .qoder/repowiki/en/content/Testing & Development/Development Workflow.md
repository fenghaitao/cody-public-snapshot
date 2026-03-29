# Development Workflow

<cite>
**Referenced Files in This Document**
- [pnpm-workspace.yaml](file://pnpm-workspace.yaml)
- [package.json](file://package.json)
- [README.md](file://README.md)
- [AGENT.md](file://AGENT.md)
- [ARCHITECTURE.md](file://ARCHITECTURE.md)
- [TESTING.md](file://TESTING.md)
- [tsconfig.json](file://tsconfig.json)
- [vitest.config.ts](file://vitest.config.ts)
- [biome.jsonc](file://biome.jsonc)
- [.stylelintrc.json](file://.stylelintrc.json)
- [vscode/package.json](file://vscode/package.json)
- [agent/package.json](file://agent/package.json)
- [vscode/CONTRIBUTING.md](file://vscode/CONTRIBUTING.md)
- [jetbrains/CONTRIBUTING.md](file://jetbrains/CONTRIBUTING.md)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Project Structure](#project-structure)
3. [Core Components](#core-components)
4. [Architecture Overview](#architecture-overview)
5. [Detailed Component Analysis](#detailed-component-analysis)
6. [Dependency Analysis](#dependency-analysis)
7. [Performance Considerations](#performance-considerations)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Contribution Guidelines](#contribution-guidelines)
10. [Continuous Integration and Release Management](#continuous-integration-and-release-management)
11. [Conclusion](#conclusion)

## Introduction
This document describes Cody’s development workflow and tooling across a monorepo powered by pnpm workspaces. It covers package management, dependency resolution, build orchestration, development environments for VS Code and JetBrains, the build system (TypeScript compilation, asset bundling, platform-specific builds), debugging techniques, development server and hot reloading, testing and CI practices, and contribution workflows.

## Project Structure
Cody is organized as a pnpm workspace with multiple packages:
- Root workspace configuration defines packages for agent, cli, lib/*, vscode, and web.
- The root package orchestrates scripts for building, testing, formatting, and release tasks.
- Each package (agent, vscode, web) has its own build and test configuration.

```mermaid
graph TB
A["Root (cody)"] --> B["agent"]
A --> C["cli"]
A --> D["lib/*"]
A --> E["vscode"]
A --> F["web"]
subgraph "Root Scripts"
S1["build/watch/check/test scripts"]
end
A --- S1
```

**Diagram sources**
- [pnpm-workspace.yaml:1-8](file://pnpm-workspace.yaml#L1-L8)
- [package.json:18-39](file://package.json#L18-L39)

**Section sources**
- [pnpm-workspace.yaml:1-8](file://pnpm-workspace.yaml#L1-L8)
- [package.json:1-99](file://package.json#L1-L99)

## Core Components
- Monorepo tooling: pnpm workspaces and root scripts coordinate builds and tests across packages.
- TypeScript configuration enables composite builds and references across packages.
- Vitest config centralizes global setup for tests.
- Formatting and linting via Biome and Stylelint.
- Package-specific build scripts for desktop and web targets.

Key responsibilities:
- Root package: orchestration, formatting, linting, and cross-package scripts.
- agent: CLI and agent runtime with platform-specific bundling.
- vscode: VS Code extension host and webviews, esbuild and vite builds.
- web: Web UI assets and build pipeline.

**Section sources**
- [package.json:18-39](file://package.json#L18-L39)
- [tsconfig.json:27-35](file://tsconfig.json#L27-L35)
- [vitest.config.ts:1-8](file://vitest.config.ts#L1-L8)
- [biome.jsonc:1-149](file://biome.jsonc#L1-L149)
- [.stylelintrc.json:1-28](file://.stylelintrc.json#L1-L28)
- [agent/package.json:13-26](file://agent/package.json#L13-L26)
- [vscode/package.json:11-56](file://vscode/package.json#L11-L56)

## Architecture Overview
The development architecture integrates:
- Composite TypeScript builds across packages.
- esbuild for Node and browser bundles in the VS Code extension.
- Vite for webviews and web UI assets.
- Agent runtime with JSON-RPC protocol and optional remote debugging.
- Cross-platform builds for Windows/macOS/Linux.

```mermaid
graph TB
subgraph "Root"
RPKG["Root package.json scripts"]
TSCFG["tsconfig.json references"]
end
subgraph "VS Code Extension"
VP["vscode/package.json"]
ES["esbuild (node/web)"]
VITEW["vite (webviews)"]
end
subgraph "Agent"
APKG["agent/package.json"]
BIN["CLI binary"]
end
subgraph "Web"
WP["web package"]
WVITE["vite (web)"]
end
RPKG --> VP
RPKG --> APKG
TSCFG --> VP
TSCFG --> APKG
VP --> ES
VP --> VITEW
APKG --> BIN
WP --> WVITE
```

**Diagram sources**
- [package.json:18-39](file://package.json#L18-L39)
- [tsconfig.json:27-35](file://tsconfig.json#L27-L35)
- [vscode/package.json:34-37](file://vscode/package.json#L34-L37)
- [agent/package.json:15-18](file://agent/package.json#L15-L18)
- [vscode/package.json:11-56](file://vscode/package.json#L11-L56)

## Detailed Component Analysis

### Monorepo and Build Orchestration
- Workspace definition lists packages and glob patterns for tests.
- Root scripts delegate to sub-packages for build, test, and release tasks.
- Composite TypeScript builds enable incremental compilation across packages.

```mermaid
flowchart TD
Start(["Developer runs script"]) --> RootScripts["Root package.json scripts"]
RootScripts --> AgentBuild["agent build"]
RootScripts --> VSCodeBuild["vscode build"]
RootScripts --> WebBuild["web build"]
AgentBuild --> DistAgent["agent dist outputs"]
VSCodeBuild --> DistVS["vscode dist outputs"]
WebBuild --> DistWeb["web dist outputs"]
DistAgent --> End(["Ready for testing/deployment"])
DistVS --> End
DistWeb --> End
```

**Diagram sources**
- [pnpm-workspace.yaml:1-8](file://pnpm-workspace.yaml#L1-L8)
- [package.json:18-39](file://package.json#L18-L39)
- [agent/package.json:15-18](file://agent/package.json#L15-L18)
- [vscode/package.json:22-32](file://vscode/package.json#L22-L32)

**Section sources**
- [pnpm-workspace.yaml:1-8](file://pnpm-workspace.yaml#L1-L8)
- [package.json:18-39](file://package.json#L18-L39)
- [tsconfig.json:27-35](file://tsconfig.json#L27-L35)

### TypeScript Compilation and References
- Composite builds configured with references to agent, lib, vscode, and web packages.
- Watch options optimized for file system events.
- Strict compiler options and source maps enabled.

```mermaid
flowchart TD
CFG["tsconfig.json"] --> COMPILE["tsc --build"]
COMPILE --> DIST["dist outputs per package"]
CFG --> WATCH["tsc --watch"]
WATCH --> DEV["Development feedback loop"]
```

**Diagram sources**
- [tsconfig.json:1-37](file://tsconfig.json#L1-L37)

**Section sources**
- [tsconfig.json:1-37](file://tsconfig.json#L1-L37)

### Asset Bundling and Platform Builds
- VS Code extension uses esbuild for Node and browser bundles with aliases and external modules.
- Vite builds webviews and web UI assets.
- Agent builds a CLI binary and related assets.

```mermaid
sequenceDiagram
participant Dev as "Developer"
participant VS as "vscode/package.json"
participant ES as "esbuild"
participant VT as "vite"
participant OUT as "dist/"
Dev->>VS : Run build/watch scripts
VS->>ES : Bundle Node extension
VS->>ES : Bundle Web extension
VS->>VT : Build webviews
ES-->>OUT : dist/extension.node.js
ES-->>OUT : dist/extension.web.js
VT-->>OUT : webviews build artifacts
```

**Diagram sources**
- [vscode/package.json:34-37](file://vscode/package.json#L34-L37)
- [vscode/package.json:22-32](file://vscode/package.json#L22-L32)

**Section sources**
- [vscode/package.json:11-56](file://vscode/package.json#L11-L56)
- [agent/package.json:15-26](file://agent/package.json#L15-L26)

### Testing and Test Orchestration
- Vitest is configured centrally with a global setup file.
- Root scripts run unit, integration, and E2E tests across packages.
- VS Code extension provides Playwright-based E2E and integration tests.

```mermaid
flowchart TD
VCFG["vitest.config.ts"] --> SETUP["Global setup"]
ROOT["Root scripts"] --> UNIT["Unit tests"]
ROOT --> INTEG["Integration tests"]
ROOT --> E2E["E2E tests"]
VS["vscode/package.json"] --> PW["Playwright tests"]
```

**Diagram sources**
- [vitest.config.ts:1-8](file://vitest.config.ts#L1-L8)
- [package.json:27-31](file://package.json#L27-L31)
- [vscode/package.json:44-51](file://vscode/package.json#L44-L51)

**Section sources**
- [vitest.config.ts:1-8](file://vitest.config.ts#L1-L8)
- [package.json:27-31](file://package.json#L27-L31)
- [vscode/package.json:44-51](file://vscode/package.json#L44-L51)
- [TESTING.md:1-317](file://TESTING.md#L1-L317)

### Formatting, Linting, and Style
- Biome enforces imports, style, correctness, and complexity rules; formatter settings match style guide.
- Stylelint enforces CSS rules and severity.
- Root scripts expose format and check commands.

```mermaid
flowchart TD
BIOME["biome.jsonc rules"] --> APPLY["Apply/fix on save"]
STYLE["stylelint config"] --> CHECK["CSS checks"]
ROOT["Root scripts"] --> RUN["pnpm format / check"]
```

**Diagram sources**
- [biome.jsonc:1-149](file://biome.jsonc#L1-L149)
- [.stylelintrc.json:1-28](file://.stylelintrc.json#L1-L28)
- [package.json:23-25](file://package.json#L23-L25)

**Section sources**
- [biome.jsonc:1-149](file://biome.jsonc#L1-L149)
- [.stylelintrc.json:1-28](file://.stylelintrc.json#L1-L28)
- [package.json:23-25](file://package.json#L23-L25)

### Development Servers, Hot Reloading, and Proxying
- VS Code development supports desktop and web targets with concurrent watchers.
- JetBrains plugin supports running with fresh agent builds and optional split mode.
- Network traffic can be captured via proxy environment variables.

```mermaid
sequenceDiagram
participant Dev as "Developer"
participant VS as "vscode/package.json"
participant Watch as "Watchers"
participant Proxy as "Proxy Env"
Dev->>VS : pnpm run dev : web / watch : dev : web
VS->>Watch : Concurrent esbuild + vite watchers
Dev->>Proxy : Set http_proxy / https_proxy
Watch-->>Dev : Live reload feedback
```

**Diagram sources**
- [vscode/package.json:19-32](file://vscode/package.json#L19-L32)
- [jetbrains/CONTRIBUTING.md:72-78](file://jetbrains/CONTRIBUTING.md#L72-L78)

**Section sources**
- [vscode/package.json:19-32](file://vscode/package.json#L19-L32)
- [jetbrains/CONTRIBUTING.md:72-78](file://jetbrains/CONTRIBUTING.md#L72-L78)

### Debugging Tools and Techniques
- VS Code extension: Chrome DevTools integration via inspect-extensions flag; dedicated Node DevTools; autocomplete trace view; export logs and heap dumps.
- JetBrains plugin: JCEF webview debugging; agent debugging via two modes (Cody spawns or IntelliJ spawns); Chrome tracing for performance.
- Agent runtime: optional remote debugging and tracing to file.

```mermaid
sequenceDiagram
participant Dev as "Developer"
participant VS as "VS Code"
participant JB as "JetBrains"
participant Agent as "Agent Runtime"
Dev->>VS : Launch extension host with inspect flag
VS-->>Dev : chrome : //inspect connection
Dev->>JB : Run with CODY_AGENT_DEBUG_INSPECT
JB->>Agent : Spawn with --inspect
Agent-->>Dev : Remote debugging on port
Dev->>Agent : Enable tracing to file
```

**Diagram sources**
- [vscode/CONTRIBUTING.md:88-106](file://vscode/CONTRIBUTING.md#L88-L106)
- [jetbrains/CONTRIBUTING.md:228-356](file://jetbrains/CONTRIBUTING.md#L228-L356)
- [agent/package.json:21-21](file://agent/package.json#L21-L21)

**Section sources**
- [vscode/CONTRIBUTING.md:88-123](file://vscode/CONTRIBUTING.md#L88-L123)
- [jetbrains/CONTRIBUTING.md:211-356](file://jetbrains/CONTRIBUTING.md#L211-L356)
- [agent/package.json:21-21](file://agent/package.json#L21-L21)

### Contribution Guidelines
- Code style: indentation, quotes, line width, type safety, imports, documentation, async patterns, telemetry naming, and error handling.
- Testing checklist: commands, chat UX, autocomplete, and telemetry coverage.
- VS Code and JetBrains development tips, WASM modules, and release dry-run flows.

```mermaid
flowchart TD
STYLE["AGENT.md style guidelines"] --> CODE["Code contributions"]
TEST["TESTING.md checklist"] --> QA["Testing requirements"]
DOC["vscode/CONTRIBUTING.md"] --> DEV["VS Code dev tips"]
JBDOC["jetbrains/CONTRIBUTING.md"] --> JBDEV["JetBrains dev tips"]
CODE --> Submit["Submit PR"]
QA --> Submit
DEV --> Submit
JBDEV --> Submit
```

**Diagram sources**
- [AGENT.md:12-26](file://AGENT.md#L12-L26)
- [TESTING.md:1-317](file://TESTING.md#L1-L317)
- [vscode/CONTRIBUTING.md:67-87](file://vscode/CONTRIBUTING.md#L67-L87)
- [jetbrains/CONTRIBUTING.md:168-227](file://jetbrains/CONTRIBUTING.md#L168-L227)

**Section sources**
- [AGENT.md:12-26](file://AGENT.md#L12-L26)
- [TESTING.md:1-317](file://TESTING.md#L1-L317)
- [vscode/CONTRIBUTING.md:67-87](file://vscode/CONTRIBUTING.md#L67-L87)
- [jetbrains/CONTRIBUTING.md:168-227](file://jetbrains/CONTRIBUTING.md#L168-L227)

## Dependency Analysis
- Root overrides and patches ensure consistent dependency versions and compatibility.
- Root engines constrain Node and pnpm versions.
- Workspace references tie packages together for composite builds.

```mermaid
graph LR
Root["Root package.json"] --> Overrides["pnpm.overrides/patches"]
Root --> Engines["Engines constraints"]
Root --> Refs["tsconfig references"]
Refs --> Agent["agent"]
Refs --> VS["vscode"]
Refs --> Web["web"]
```

**Diagram sources**
- [package.json:84-97](file://package.json#L84-L97)
- [tsconfig.json:27-35](file://tsconfig.json#L27-L35)

**Section sources**
- [package.json:84-97](file://package.json#L84-L97)
- [tsconfig.json:27-35](file://tsconfig.json#L27-L35)

## Performance Considerations
- Use composite builds to speed up incremental compilation.
- Prefer esbuild for rapid bundling during development.
- Leverage watch mode and concurrent watchers for hot reloading.
- Use Chrome tracing to profile Agent and extension performance.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
- VS Code: enable verbose debug logging, use autocomplete trace view, export logs and heap dumps.
- JetBrains: enable JCEF debugging, adjust registry settings, and use Chrome tracing.
- Network diagnostics: configure proxy environment variables to capture extension traffic.
- Version mismatches: ensure Node and pnpm versions match engine constraints; restart Gradle daemons if needed.

**Section sources**
- [vscode/CONTRIBUTING.md:28-37](file://vscode/CONTRIBUTING.md#L28-L37)
- [jetbrains/CONTRIBUTING.md:90-103](file://jetbrains/CONTRIBUTING.md#L90-L103)
- [jetbrains/CONTRIBUTING.md:211-227](file://jetbrains/CONTRIBUTING.md#L211-L227)

## Continuous Integration and Release Management
- Root scripts provide commands for building, testing, and releasing.
- VS Code release dry-run builds a packaged extension for local verification.
- JetBrains plugin supports experimental and stable release tagging and publishing workflows.

```mermaid
flowchart TD
CI["CI triggers"] --> BUILD["Root build scripts"]
BUILD --> TESTS["Unit/Integration/E2E"]
TESTS --> ARTIFACTS["Package artifacts"]
ARTIFACTS --> RELEASE["Release dry-run / tagging"]
```

**Diagram sources**
- [package.json:18-39](file://package.json#L18-L39)
- [vscode/package.json:43-43](file://vscode/package.json#L43-L43)
- [jetbrains/CONTRIBUTING.md:168-204](file://jetbrains/CONTRIBUTING.md#L168-L204)

**Section sources**
- [package.json:18-39](file://package.json#L18-L39)
- [vscode/package.json:43-43](file://vscode/package.json#L43-L43)
- [jetbrains/CONTRIBUTING.md:168-204](file://jetbrains/CONTRIBUTING.md#L168-L204)

## Conclusion
Cody’s development workflow leverages a pnpm workspace to coordinate builds and tests across agent, VS Code, and web packages. Composite TypeScript builds, esbuild, and Vite streamline development and deployment. Robust debugging and testing tooling, combined with clear style and testing guidelines, support efficient collaboration and high-quality releases.