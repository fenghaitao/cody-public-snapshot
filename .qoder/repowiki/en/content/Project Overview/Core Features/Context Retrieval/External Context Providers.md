# External Context Providers

<cite>
**Referenced Files in This Document**
- [openctx.ts](file://vscode/src/context/openctx.ts)
- [openctx.test.ts](file://vscode/src/context/openctx.test.ts)
- [codeSearch.ts](file://vscode/src/context/openctx/codeSearch.ts)
- [git.ts](file://vscode/src/context/openctx/git.ts)
- [web.ts](file://vscode/src/context/openctx/web.ts)
- [remoteRepositorySearch.ts](file://vscode/src/context/openctx/remoteRepositorySearch.ts)
- [remoteDirectorySearch.ts](file://vscode/src/context/openctx/remoteDirectorySearch.ts)
- [remoteFileSearch.ts](file://vscode/src/context/openctx/remoteFileSearch.ts)
- [rules.ts](file://vscode/src/context/openctx/rules.ts)
- [linear-issues.ts](file://vscode/src/context/openctx/linear-issues.ts)
- [get-repository-mentions.ts](file://vscode/src/context/openctx/common/get-repository-mentions.ts)
- [context.ts](file://lib/shared/src/context/openctx/context.ts)
- [api.ts](file://lib/shared/src/context/openctx/api.ts)
- [types.ts](file://vscode/src/context/openctx/types.ts)
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
9. [Conclusion](#conclusion)
10. [Appendices](#appendices)

## Introduction
This document explains how external context providers integrate with the OpenCtx protocol in the application. It covers the OpenCtx integration architecture, provider registration, context fetching mechanisms, and the built-in providers for code search, Git repository integration, web search, and remote directory access. It also documents provider configuration, authentication handling, rate limiting considerations, practical usage examples, and the relationship between external providers and local indexing for comprehensive codebase search.

## Project Structure
The OpenCtx integration is implemented primarily under the OpenCtx module in the VS Code extension, with shared APIs and context orchestration in the shared library. The key areas are:
- Provider registration and controller lifecycle
- Built-in providers (web, code search, Git, remote repository/file/directory, rules, Linear issues)
- Shared OpenCtx client controller and context retrieval utilities
- Common repository mention utilities

```mermaid
graph TB
subgraph "VS Code Extension"
OC["openctx.ts<br/>Controller & Registration"]
WEB["web.ts<br/>Web Provider"]
CS["codeSearch.ts<br/>Code Search Provider"]
GIT["git.ts<br/>Git Provider"]
RR["remoteRepositorySearch.ts<br/>Remote Repo Provider"]
RD["remoteDirectorySearch.ts<br/>Remote Directory Provider"]
RF["remoteFileSearch.ts<br/>Remote File Provider"]
RL["rules.ts<br/>Rules Provider"]
LI["linear-issues.ts<br/>Linear Issues Provider"]
CRM["get-repository-mentions.ts<br/>Repo Mentions Utility"]
end
subgraph "Shared Library"
API["api.ts<br/>OpenCtx Controller API"]
CTX["context.ts<br/>Context Retrieval for Chat"]
end
OC --> WEB
OC --> CS
OC --> GIT
OC --> RR
OC --> RD
OC --> RF
OC --> RL
OC --> LI
RR --> CRM
RD --> CRM
RF --> CRM
CTX --> API
OC --> API
```

**Diagram sources**
- [openctx.ts:46-105](file://vscode/src/context/openctx.ts#L46-L105)
- [web.ts:8-38](file://vscode/src/context/openctx/web.ts#L8-L38)
- [codeSearch.ts:35-65](file://vscode/src/context/openctx/codeSearch.ts#L35-L65)
- [git.ts:25-130](file://vscode/src/context/openctx/git.ts#L25-L130)
- [remoteRepositorySearch.ts:14-56](file://vscode/src/context/openctx/remoteRepositorySearch.ts#L14-L56)
- [remoteDirectorySearch.ts:12-44](file://vscode/src/context/openctx/remoteDirectorySearch.ts#L12-L44)
- [remoteFileSearch.ts:17-49](file://vscode/src/context/openctx/remoteFileSearch.ts#L17-L49)
- [rules.ts:20-79](file://vscode/src/context/openctx/rules.ts#L20-L79)
- [linear-issues.ts:4-10](file://vscode/src/context/openctx/linear-issues.ts#L4-L10)
- [get-repository-mentions.ts:35-90](file://vscode/src/context/openctx/common/get-repository-mentions.ts#L35-L90)
- [api.ts:6-41](file://lib/shared/src/context/openctx/api.ts#L6-L41)
- [context.ts:6-75](file://lib/shared/src/context/openctx/context.ts#L6-L75)

**Section sources**
- [openctx.ts:46-105](file://vscode/src/context/openctx.ts#L46-L105)
- [api.ts:6-41](file://lib/shared/src/context/openctx/api.ts#L6-L41)
- [context.ts:6-75](file://lib/shared/src/context/openctx/context.ts#L6-L75)

## Core Components
- OpenCtx controller lifecycle and provider registration
- Built-in providers and their capabilities
- Shared OpenCtx controller API and context retrieval pipeline
- Common repository mention utilities

Key responsibilities:
- Register providers based on configuration, authentication status, and feature flags
- Provide context items to the chat pipeline via the shared controller
- Fetch and transform data from GraphQL and external systems

**Section sources**
- [openctx.ts:109-207](file://vscode/src/context/openctx.ts#L109-L207)
- [openctx.ts:209-255](file://vscode/src/context/openctx.ts#L209-L255)
- [api.ts:6-41](file://lib/shared/src/context/openctx/api.ts#L6-L41)
- [context.ts:6-75](file://lib/shared/src/context/openctx/context.ts#L6-L75)

## Architecture Overview
The OpenCtx integration centers on a controller that manages provider lifecycles and exposes meta, mentions, and items APIs. Providers are registered conditionally based on:
- Authentication status and endpoint type (dotCom vs enterprise)
- Client configuration (e.g., omni box enabled)
- Feature flags (e.g., Git mentions)
- Site version compatibility for advanced features

```mermaid
sequenceDiagram
participant Ext as "Extension"
participant Reg as "Provider Registry<br/>openctx.ts"
participant Ctrl as "OpenCtx Controller<br/>api.ts"
participant Prov as "Providers"
participant Chat as "Chat Pipeline<br/>context.ts"
Ext->>Reg : Initialize with createController
Reg->>Ctrl : Build providers list
Ctrl-->>Reg : Controller instance
Reg-->>Ext : Observable controller
Chat->>Ctrl : meta({}) to discover providers
Chat->>Ctrl : items({message}) for matching providers
Ctrl->>Prov : Dispatch to provider items(...)
Prov-->>Ctrl : Items with AI content
Ctrl-->>Chat : Flat list of context items
```

**Diagram sources**
- [openctx.ts:50-105](file://vscode/src/context/openctx.ts#L50-L105)
- [api.ts:6-41](file://lib/shared/src/context/openctx/api.ts#L6-L41)
- [context.ts:7-75](file://lib/shared/src/context/openctx/context.ts#L7-L75)

## Detailed Component Analysis

### OpenCtx Controller and Provider Registration
- Creates the OpenCtx controller with provider configurations
- Merges user-provided configuration from the Sourcegraph instance when enabled
- Registers providers differently for VS Code and Cody Web environments
- Conditionally enables providers based on auth, site version, and feature flags

```mermaid
flowchart TD
Start(["Initialize"]) --> CheckNoodle["experimentalNoodle enabled?"]
CheckNoodle --> |Yes| MergeCfg["mergeConfiguration from viewerSettings"]
CheckNoodle --> |No| SkipMerge["Use local providers only"]
MergeCfg --> BuildProviders["Build providers list"]
SkipMerge --> BuildProviders
BuildProviders --> VSCode{"VS Code?"}
VSCode --> |Yes| VSProviders["Web + Rules + Remote + GitMentions + CodeSearch"]
VSCode --> |No| WebProviders["Web + Rules + Remote + CodeSearch"]
VSProviders --> Controller["Create Controller"]
WebProviders --> Controller
Controller --> Ready(["Controller Ready"])
```

**Diagram sources**
- [openctx.ts:50-105](file://vscode/src/context/openctx.ts#L50-L105)
- [openctx.ts:109-207](file://vscode/src/context/openctx.ts#L109-L207)
- [openctx.ts:209-255](file://vscode/src/context/openctx.ts#L209-L255)
- [openctx.ts:257-297](file://vscode/src/context/openctx.ts#L257-L297)

**Section sources**
- [openctx.ts:50-105](file://vscode/src/context/openctx.ts#L50-L105)
- [openctx.ts:109-207](file://vscode/src/context/openctx.ts#L109-L207)
- [openctx.ts:209-255](file://vscode/src/context/openctx.ts#L209-L255)
- [openctx.ts:257-297](file://vscode/src/context/openctx.ts#L257-L297)

### Built-in Providers

#### Web Provider
- Fetches content from a URL and returns it as an AI-readable item
- Supports proxy mode via GraphQL client and direct fetch mode with sanitization
- Provides mention suggestions for typed URLs

```mermaid
sequenceDiagram
participant Chat as "Chat"
participant Ctrl as "Controller"
participant Web as "Web Provider"
participant GQL as "GraphQL Client"
Chat->>Ctrl : mentions({query})
Ctrl->>Web : mentions({query})
Web->>GQL : getURLContent(url)
GQL-->>Web : {title, content}
Web-->>Ctrl : [{title, url, ai.content}]
Ctrl-->>Chat : Items
Chat->>Ctrl : items({message})
Ctrl->>Web : items({message})
Web->>GQL : getURLContent(url)
GQL-->>Web : {title, content}
Web-->>Ctrl : [{title, url, ai.content}]
Ctrl-->>Chat : Items
```

**Diagram sources**
- [web.ts:8-38](file://vscode/src/context/openctx/web.ts#L8-L38)
- [web.ts:40-94](file://vscode/src/context/openctx/web.ts#L40-L94)

**Section sources**
- [web.ts:8-38](file://vscode/src/context/openctx/web.ts#L8-L38)
- [web.ts:40-94](file://vscode/src/context/openctx/web.ts#L40-L94)

#### Code Search Provider
- Converts code search results into context items
- Fetches file content from GraphQL and constructs AI-readable items
- Exposes a context item creator for embedding search results

```mermaid
sequenceDiagram
participant Chat as "Chat"
participant Ctrl as "Controller"
participant CS as "Code Search Provider"
participant GQL as "GraphQL Client"
Chat->>Ctrl : items({mention.data})
Ctrl->>CS : items({mention})
CS->>CS : Validate mention.data
CS->>GQL : getFileContents(repo, path, rev)
GQL-->>CS : {repository.commit.file}
CS-->>Ctrl : [{title, url, ai.content}]
Ctrl-->>Chat : Items
```

**Diagram sources**
- [codeSearch.ts:35-65](file://vscode/src/context/openctx/codeSearch.ts#L35-L65)
- [codeSearch.ts:67-91](file://vscode/src/context/openctx/codeSearch.ts#L67-L91)

**Section sources**
- [codeSearch.ts:35-65](file://vscode/src/context/openctx/codeSearch.ts#L35-L65)
- [codeSearch.ts:67-91](file://vscode/src/context/openctx/codeSearch.ts#L67-L91)
- [codeSearch.ts:101-127](file://vscode/src/context/openctx/codeSearch.ts#L101-L127)

#### Git Provider
- Generates mentions for Git repository diffs and uncommitted changes
- Executes Git commands to produce diffs and commit logs
- Parses custom mention URIs to route to appropriate handlers

```mermaid
sequenceDiagram
participant Chat as "Chat"
participant Ctrl as "Controller"
participant Git as "Git Provider"
participant OS as "OS Shell"
Chat->>Ctrl : mentions({query})
Ctrl->>Git : mentions({query})
Git->>Git : getGitInfoForMentions()
Git->>OS : git symbolic-ref, diff, log
OS-->>Git : stdout/stderr
Git-->>Ctrl : [{title, uri, description}]
Ctrl-->>Chat : Mentions
Chat->>Ctrl : items({mention})
Ctrl->>Git : items({mention})
Git->>Git : parseMentionURI()
Git->>OS : git diff/log
OS-->>Git : stdout
Git-->>Ctrl : [{title, ai.content}]
Ctrl-->>Chat : Items
```

**Diagram sources**
- [git.ts:25-130](file://vscode/src/context/openctx/git.ts#L25-L130)
- [git.ts:147-175](file://vscode/src/context/openctx/git.ts#L147-L175)
- [git.ts:183-225](file://vscode/src/context/openctx/git.ts#L183-L225)

**Section sources**
- [git.ts:25-130](file://vscode/src/context/openctx/git.ts#L25-L130)
- [git.ts:147-175](file://vscode/src/context/openctx/git.ts#L147-L175)
- [git.ts:183-225](file://vscode/src/context/openctx/git.ts#L183-L225)

#### Remote Repository Provider
- Suggests repositories and performs context search within a selected repository
- Uses repository mention utilities and GraphQL context search

```mermaid
sequenceDiagram
participant Chat as "Chat"
participant Ctrl as "Controller"
participant Repo as "Remote Repo Provider"
participant GQL as "GraphQL Client"
participant CRM as "Repo Mentions"
Chat->>Ctrl : mentions({query})
Ctrl->>Repo : mentions({query})
Repo->>CRM : getRepositoryMentions(query)
CRM-->>Repo : [{title, data.repoId, uri}]
Repo-->>Ctrl : Mentions
Ctrl-->>Chat : Mentions
Chat->>Ctrl : items({mention})
Ctrl->>Repo : items({mention})
Repo->>GQL : contextSearch(repoIDs, query)
GQL-->>Repo : [nodes]
Repo-->>Ctrl : [{title, url, ai.content}]
Ctrl-->>Chat : Items
```

**Diagram sources**
- [remoteRepositorySearch.ts:14-56](file://vscode/src/context/openctx/remoteRepositorySearch.ts#L14-L56)
- [get-repository-mentions.ts:35-90](file://vscode/src/context/openctx/common/get-repository-mentions.ts#L35-L90)

**Section sources**
- [remoteRepositorySearch.ts:14-56](file://vscode/src/context/openctx/remoteRepositorySearch.ts#L14-L56)
- [get-repository-mentions.ts:35-90](file://vscode/src/context/openctx/common/get-repository-mentions.ts#L35-L90)

#### Remote Directory Provider
- Suggests directories within repositories and retrieves directory contents
- Builds file patterns for context search and returns AI-readable items

```mermaid
sequenceDiagram
participant Chat as "Chat"
participant Ctrl as "Controller"
participant Dir as "Remote Directory Provider"
participant GQL as "GraphQL Client"
Chat->>Ctrl : mentions({query})
Ctrl->>Dir : mentions({query})
Dir->>GQL : searchFileMatches("repo : ... file : ... select : file.directory")
GQL-->>Dir : Results
Dir-->>Ctrl : Mentions with repoID, directoryPath
Ctrl-->>Chat : Mentions
Chat->>Ctrl : items({mention})
Ctrl->>Dir : items({mention})
Dir->>GQL : contextSearch(repoIDs, query, filePatterns)
GQL-->>Dir : [nodes]
Dir-->>Ctrl : [{title, url, ai.content}]
Ctrl-->>Chat : Items
```

**Diagram sources**
- [remoteDirectorySearch.ts:12-44](file://vscode/src/context/openctx/remoteDirectorySearch.ts#L12-L44)
- [remoteDirectorySearch.ts:84-105](file://vscode/src/context/openctx/remoteDirectorySearch.ts#L84-L105)

**Section sources**
- [remoteDirectorySearch.ts:12-44](file://vscode/src/context/openctx/remoteDirectorySearch.ts#L12-L44)
- [remoteDirectorySearch.ts:84-105](file://vscode/src/context/openctx/remoteDirectorySearch.ts#L84-L105)

#### Remote File Provider
- Suggests files within repositories and fetches file content
- Escapes regex patterns for precise matching

```mermaid
sequenceDiagram
participant Chat as "Chat"
participant Ctrl as "Controller"
participant File as "Remote File Provider"
participant GQL as "GraphQL Client"
Chat->>Ctrl : mentions({query})
Ctrl->>File : mentions({query})
File->>GQL : searchFileMatches("repo : ... file : ...")
GQL-->>File : Results
File-->>Ctrl : Mentions with repoName, filePath, rev
Ctrl-->>Chat : Mentions
Chat->>Ctrl : items({mention})
Ctrl->>File : items({mention})
File->>GQL : getFileContents(repoName, filePath, rev)
GQL-->>File : {repository.commit.file}
File-->>Ctrl : [{title, url, ai.content}]
Ctrl-->>Chat : Items
```

**Diagram sources**
- [remoteFileSearch.ts:17-49](file://vscode/src/context/openctx/remoteFileSearch.ts#L17-L49)
- [remoteFileSearch.ts:88-112](file://vscode/src/context/openctx/remoteFileSearch.ts#L88-L112)

**Section sources**
- [remoteFileSearch.ts:17-49](file://vscode/src/context/openctx/remoteFileSearch.ts#L17-L49)
- [remoteFileSearch.ts:88-112](file://vscode/src/context/openctx/remoteFileSearch.ts#L88-L112)

#### Rules Provider
- Automatically includes applicable rules for the current file or workspace root
- Transforms rules into context items with instructions

```mermaid
sequenceDiagram
participant Chat as "Chat"
participant Ctrl as "Controller"
participant Rules as "Rules Provider"
participant RS as "Rule Service"
Chat->>Ctrl : mentions({autoInclude, uri})
Ctrl->>Rules : mentions({autoInclude, uri})
Rules->>RS : rulesForPaths([fileOrWorkspaceRoot])
RS-->>Rules : Rules[]
Rules-->>Ctrl : [{title, data.rules}]
Ctrl-->>Chat : Mentions
Chat->>Ctrl : items({mention})
Ctrl->>Rules : items({mention})
Rules-->>Ctrl : [{title, url, ai.content}]
Ctrl-->>Chat : Items
```

**Diagram sources**
- [rules.ts:20-79](file://vscode/src/context/openctx/rules.ts#L20-L79)

**Section sources**
- [rules.ts:20-79](file://vscode/src/context/openctx/rules.ts#L20-L79)

#### Linear Issues Provider
- Integrates with the external Linear issues OpenCtx provider
- Acts as a thin wrapper exposing the provider with a dedicated provider URI

**Section sources**
- [linear-issues.ts:4-10](file://vscode/src/context/openctx/linear-issues.ts#L4-L10)

### Context Retrieval Pipeline
The shared context retrieval pipeline:
- Discovers providers via controller meta
- Filters providers by message selectors
- Collects items from matching providers
- Normalizes items into context items with AI content

```mermaid
flowchart TD
A["getContextForChatMessage(message)"] --> B["controller.meta()"]
B --> C["Filter providers by messageSelectors"]
C --> D["controller.items({message}) for each provider"]
D --> E["Collect and flatten items"]
E --> F["Filter items with ai.content"]
F --> G["Map to ContextItemOpenCtx"]
G --> H["Return context items"]
```

**Diagram sources**
- [context.ts:7-75](file://lib/shared/src/context/openctx/context.ts#L7-L75)

**Section sources**
- [context.ts:7-75](file://lib/shared/src/context/openctx/context.ts#L7-L75)

## Dependency Analysis
- Provider registration depends on:
  - Authentication status and endpoint type
  - Client configuration (omni box enabled)
  - Feature flags (e.g., Git mentions)
  - Site version compatibility for advanced features
- Providers depend on:
  - GraphQL client for remote data
  - Shared utilities for repository mentions and fuzzy matching
  - OS shell for Git operations (in Git provider)
- Context retrieval depends on:
  - Global OpenCtx controller observable
  - Provider message selectors

```mermaid
graph LR
Auth["Auth Status"] --> Reg["Provider Registry"]
Config["Client Config"] --> Reg
Flags["Feature Flags"] --> Reg
Version["Site Version"] --> Reg
Reg --> Providers["Built-in Providers"]
Providers --> GQL["GraphQL Client"]
Providers --> OS["OS Shell (Git)"]
Providers --> Utils["Common Utilities"]
CtrlObs["OpenCtx Controller Observable"] --> CTX["Context Retrieval"]
CTX --> Providers
```

**Diagram sources**
- [openctx.ts:109-207](file://vscode/src/context/openctx.ts#L109-L207)
- [openctx.ts:209-255](file://vscode/src/context/openctx.ts#L209-L255)
- [context.ts:7-75](file://lib/shared/src/context/openctx/context.ts#L7-L75)

**Section sources**
- [openctx.ts:109-207](file://vscode/src/context/openctx.ts#L109-L207)
- [openctx.ts:209-255](file://vscode/src/context/openctx.ts#L209-L255)
- [context.ts:7-75](file://lib/shared/src/context/openctx/context.ts#L7-L75)

## Performance Considerations
- Web provider content truncation: The web provider truncates fetched content to avoid overwhelming the context window for the LLM.
- Repository fuzzy matching: Repository suggestions use fuzzy matching with tiebreakers to prioritize relevant results efficiently.
- Conditional provider loading: Providers are only registered when relevant, reducing overhead.
- Abort handling: The context retrieval pipeline respects abort signals to cancel long-running operations gracefully.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
- OpenCtx extension conflict: The integration warns and directs users to disable the external OpenCtx extension when both are present.
- Provider initialization failures: Initialization errors are logged and surfaced to aid debugging.
- Mention URI parsing errors: Git provider validates and parses mention URIs, logging invalid URIs for inspection.
- Rate limiting and timeouts: While explicit rate limiting is not implemented in the providers, the web provider uses timeouts for direct fetch mode and relies on GraphQL client error handling for proxy mode.

**Section sources**
- [openctx.ts:299-309](file://vscode/src/context/openctx.ts#L299-L309)
- [git.ts:191-214](file://vscode/src/context/openctx/git.ts#L191-L214)
- [web.ts:40-94](file://vscode/src/context/openctx/web.ts#L40-L94)

## Conclusion
The OpenCtx integration provides a flexible, extensible framework for bringing external context into the chat pipeline. Providers are registered dynamically based on configuration and environment, and the shared controller orchestrates discovery, mentions, and item retrieval. Built-in providers cover web content, code search, Git repository insights, and remote repository/file/directory access, with optional rules and Linear integration. The design supports graceful error handling, abort signals, and performance-conscious content shaping.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### Provider Configuration and Authentication
- Provider registration is driven by:
  - Authentication status and endpoint type
  - Client configuration toggles (e.g., omni box)
  - Feature flags (e.g., Git mentions)
  - Site version checks for advanced features
- Authentication affects availability of enterprise-only providers and site version-dependent features.

**Section sources**
- [openctx.ts:109-207](file://vscode/src/context/openctx.ts#L109-L207)
- [openctx.ts:209-255](file://vscode/src/context/openctx.ts#L209-L255)

### Practical Usage Examples
- Web URL context: Type or paste a URL; the Web provider suggests and fetches content for the LLM.
- Code search context: Trigger code search results; the Code Search provider converts results into context items.
- Git diff context: Use Git provider mentions to include diffs vs. default branch or uncommitted changes.
- Remote repository/file/directory context: Select a repository or directory to search and include matching files as context.

**Section sources**
- [web.ts:8-38](file://vscode/src/context/openctx/web.ts#L8-L38)
- [codeSearch.ts:35-65](file://vscode/src/context/openctx/codeSearch.ts#L35-L65)
- [git.ts:25-130](file://vscode/src/context/openctx/git.ts#L25-L130)
- [remoteRepositorySearch.ts:14-56](file://vscode/src/context/openctx/remoteRepositorySearch.ts#L14-L56)
- [remoteDirectorySearch.ts:12-44](file://vscode/src/context/openctx/remoteDirectorySearch.ts#L12-L44)
- [remoteFileSearch.ts:17-49](file://vscode/src/context/openctx/remoteFileSearch.ts#L17-L49)

### Context Filtering, Relevance Scoring, and Result Merging
- Filtering: Providers are filtered by message selectors matched against the chat message.
- Relevance scoring: Repository suggestions use fuzzy matching with tiebreakers to prioritize results.
- Result merging: Items from multiple providers are flattened and normalized into context items with AI content.

**Section sources**
- [context.ts:7-75](file://lib/shared/src/context/openctx/context.ts#L7-L75)
- [get-repository-mentions.ts:20-24](file://vscode/src/context/openctx/common/get-repository-mentions.ts#L20-L24)

### Relationship Between External Providers and Local Indexing
- External providers augment context with remote data (web pages, repositories, files, Git diffs).
- Local indexing complements external providers by enabling fast, offline search within the workspace.
- Together, they provide comprehensive codebase search and contextual awareness across local and remote sources.

[No sources needed since this section provides general guidance]