# Feature Flags & Experiments

<cite>
**Referenced Files in This Document**
- [FeatureFlagProvider.ts](file://lib/shared/src/experimentation/FeatureFlagProvider.ts)
- [client.ts](file://lib/shared/src/sourcegraph-api/graphql/client.ts)
- [configuration.ts](file://vscode/src/configuration.ts)
- [configuration-keys.ts](file://vscode/src/configuration-keys.ts)
- [package.json](file://vscode/package.json)
- [features.json5 (VS Code)](file://vscode/features.json5)
- [features.json5 (JetBrains)](file://jetbrains/features.json5)
- [create-autoedits-provider.ts](file://vscode/src/autoedits/create-autoedits-provider.ts)
- [autoedit-completions.test.ts](file://vscode/src/autoedits/adapters/sourcegraph-completions.test.ts)
- [userProductSubscription.ts](file://lib/shared/src/sourcegraph-api/userProductSubscription.ts)
- [FeatureFlagProvider.test.ts](file://lib/shared/src/experimentation/FeatureFlagProvider.test.ts)
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
This document explains the feature flag management and experimental features system in the Cody platform. It covers:
- The centralized feature flag provider and its evaluation lifecycle
- How experimental features are surfaced in configuration and UI
- A/B testing and exposure tracking via the backend
- Enterprise controls and user permission relationships
- Practical scenarios: enabling beta features for specific users, gradual rollouts, testing, and deprecations
- Validation, rollback, and performance considerations
- The features.json5 configuration system and how it integrates with the broader configuration architecture

## Project Structure
The feature flag system spans shared libraries, VS Code extension code, and JetBrains plugin code. The VS Code extension reads user-facing configuration keys and maps them to internal feature flags. The shared library evaluates flags against the Sourcegraph backend and caches results for efficient consumption.

```mermaid
graph TB
subgraph "VS Code Extension"
CFG["configuration.ts<br/>Builds ClientConfiguration"]
KEYS["configuration-keys.ts<br/>Maps package.json keys"]
PKG["package.json<br/>Contributes commands/settings"]
FEAT_JSON_VS["features.json5 (VS Code)"]
end
subgraph "Shared Library"
FF_PROVIDER["FeatureFlagProvider.ts<br/>evaluatedFeatureFlag()"]
API_CLIENT["client.ts<br/>GraphQL API calls"]
SUBS["userProductSubscription.ts<br/>Enterprise detection"]
end
subgraph "JetBrains Plugin"
FEAT_JSON_JB["features.json5 (JetBrains)"]
end
CFG --> KEYS
CFG --> FF_PROVIDER
PKG --> CFG
FEAT_JSON_VS --> CFG
FEAT_JSON_JB --> CFG
FF_PROVIDER --> API_CLIENT
FF_PROVIDER --> SUBS
```

**Diagram sources**
- [configuration.ts:1-200](file://vscode/src/configuration.ts#L1-L200)
- [configuration-keys.ts:1-55](file://vscode/src/configuration-keys.ts#L1-L55)
- [package.json:1-800](file://vscode/package.json#L1-L800)
- [features.json5 (VS Code):1-91](file://vscode/features.json5#L1-L91)
- [features.json5 (JetBrains):1-69](file://jetbrains/features.json5#L1-L69)
- [FeatureFlagProvider.ts:182-358](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L182-L358)
- [client.ts:1564-1588](file://lib/shared/src/sourcegraph-api/graphql/client.ts#L1564-L1588)
- [userProductSubscription.ts:89-104](file://lib/shared/src/sourcegraph-api/userProductSubscription.ts#L89-L104)

**Section sources**
- [configuration.ts:1-200](file://vscode/src/configuration.ts#L1-L200)
- [configuration-keys.ts:1-55](file://vscode/src/configuration-keys.ts#L1-L55)
- [package.json:1-800](file://vscode/package.json#L1-L800)
- [features.json5 (VS Code):1-91](file://vscode/features.json5#L1-L91)
- [features.json5 (JetBrains):1-69](file://jetbrains/features.json5#L1-L69)
- [FeatureFlagProvider.ts:182-358](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L182-L358)
- [client.ts:1564-1588](file://lib/shared/src/sourcegraph-api/graphql/client.ts#L1564-L1588)
- [userProductSubscription.ts:89-104](file://lib/shared/src/sourcegraph-api/userProductSubscription.ts#L89-L104)

## Core Components
- FeatureFlagProvider: Centralized evaluator for feature flags with caching, exposure tracking, and reactive updates.
- GraphQL API client: Fetches evaluated flags and exposes them to the provider.
- VS Code configuration: Bridges user-facing settings to internal feature flags and experimental toggles.
- features.json5: Declares feature metadata and statuses per editor, used for documentation and visibility.

Key responsibilities:
- Evaluate flags reactively and cache results for performance
- Expose “exposed experiments” for immediate synchronous access
- Support forced refresh for rapidly changing flags
- Integrate with enterprise user detection for capability gating

**Section sources**
- [FeatureFlagProvider.ts:22-178](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L22-L178)
- [FeatureFlagProvider.ts:182-358](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L182-L358)
- [client.ts:1564-1588](file://lib/shared/src/sourcegraph-api/graphql/client.ts#L1564-L1588)
- [configuration.ts:144-159](file://vscode/src/configuration.ts#L144-L159)
- [features.json5 (VS Code):1-91](file://vscode/features.json5#L1-L91)

## Architecture Overview
The feature flag pipeline evaluates flags reactively, caches them, and exposes them to consumers. Enterprise users may have different capabilities gated by subscription checks.

```mermaid
sequenceDiagram
participant VS as "VS Code Extension"
participant CFG as "configuration.ts"
participant FF as "FeatureFlagProvider.ts"
participant API as "client.ts (GraphQL)"
participant BE as "Sourcegraph Backend"
VS->>CFG : Build ClientConfiguration (reads settings)
CFG-->>VS : ClientConfiguration with experimental flags
VS->>FF : observedFeatureFlag(flag)
FF->>FF : Check cache for endpoint
alt Cache hit
FF-->>VS : Emit cached value
else Cache miss
FF->>API : evaluateFeatureFlag(name)
API->>BE : GraphQL evaluateFeatureFlag
BE-->>API : { flag : boolean }
API-->>FF : Result
FF->>FF : Update cache
FF-->>VS : Emit updated value
end
```

**Diagram sources**
- [configuration.ts:144-159](file://vscode/src/configuration.ts#L144-L159)
- [FeatureFlagProvider.ts:288-335](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L288-L335)
- [client.ts:1564-1588](file://lib/shared/src/sourcegraph-api/graphql/client.ts#L1564-L1588)

**Section sources**
- [FeatureFlagProvider.ts:207-358](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L207-L358)
- [client.ts:1564-1588](file://lib/shared/src/sourcegraph-api/graphql/client.ts#L1564-L1588)

## Detailed Component Analysis

### Feature Flag Provider and Evaluation Lifecycle
- Reactive evaluation: Consumers subscribe to a flag via an observable to receive updates without reloading.
- Caching: Results are cached per server endpoint to avoid redundant network calls.
- Exposure tracking: Exposed experiments are synchronized for immediate access.
- Forced refresh: Supports refreshing values for flags that change frequently.
- No-op mode: When disabled via environment variable, all evaluations return false.

```mermaid
classDiagram
class FeatureFlagProvider {
+evaluatedFeatureFlag(flag) : Observable<boolean>
+evaluateFeatureFlagEphemerally(flag) : Promise<boolean>
+getExposedExperiments(endpoint) : Record<string, boolean>
+refresh() : void
+dispose() : void
}
class FeatureFlagProviderImpl {
-cache : Record<string, Record<string, boolean>>
-refreshRequests : Subject<void>
-evaluatedFeatureFlagCache : Partial<Record<FeatureFlag, StoredLastValue<boolean>>>
+evaluatedFeatureFlag(flag, forceRefresh) : Observable<boolean>
+getExposedExperiments(endpoint) : Record<string, boolean>
+refresh() : void
+dispose() : void
}
class FeatureFlag {
<<enum>>
+CodyAutocompleteTracing
+CodyAutoEditExperimentEnabledFeatureFlag
+CodyAutoEditHotStreak
+CodyAutoEditUseWebSocketForFireworksConnections
+CodyExperimentalOneBoxDebug
+CodyExperimentalPromptEditor
+... many others
}
FeatureFlagProvider <|.. FeatureFlagProviderImpl
FeatureFlagProviderImpl --> FeatureFlag : "consumes"
```

**Diagram sources**
- [FeatureFlagProvider.ts:182-358](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L182-L358)
- [FeatureFlagProvider.ts:22-178](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L22-L178)

**Section sources**
- [FeatureFlagProvider.ts:182-358](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L182-L358)
- [FeatureFlagProvider.test.ts:1-47](file://lib/shared/src/experimentation/FeatureFlagProvider.test.ts#L1-L47)

### Experimental Features in Configuration and UI
- VS Code configuration maps user-facing keys to internal feature flags. Examples include:
  - experimentalTracing: Controls tracing for autocomplete
  - experimentalSupercompletions: Enables supercompletions
  - experimentalAutoEditEnabled: Enables auto-edit suggestions
  - experimentalAutoEditConfigOverride: Allows provider/websocket overrides
- These settings influence UI commands and feature availability (e.g., commands gated by config conditions).

```mermaid
flowchart TD
Start(["User sets cody.experimental.*"]) --> ReadCfg["configuration.ts reads settings"]
ReadCfg --> MapFlags["Map to internal feature flags"]
MapFlags --> UI["package.json contributes commands/settings"]
UI --> Consumer["Feature consumers (e.g., autoedits, supercompletions)"]
Consumer --> Effect["Feature behavior changes reactively"]
```

**Diagram sources**
- [configuration.ts:144-159](file://vscode/src/configuration.ts#L144-L159)
- [package.json:268-276](file://vscode/package.json#L268-L276)
- [package.json:529-532](file://vscode/package.json#L529-L532)

**Section sources**
- [configuration.ts:144-159](file://vscode/src/configuration.ts#L144-L159)
- [package.json:268-276](file://vscode/package.json#L268-L276)
- [package.json:529-532](file://vscode/package.json#L529-L532)

### Auto-Edit and Tracing Integration
- Auto-edit eligibility depends on:
  - Feature flag for auto-edit experiments
  - User subscription tier
  - Editor environment (e.g., agent vs desktop)
- Tracing can be toggled via configuration and also via a dedicated feature flag for autocomplete tracing.

```mermaid
sequenceDiagram
participant User as "User"
participant VS as "VS Code"
participant CFG as "configuration.ts"
participant Provider as "FeatureFlagProvider.ts"
participant Auto as "create-autoedits-provider.ts"
User->>VS : Enable experimental auto-edit
VS->>CFG : Read cody.experimental.autoedit.*
CFG-->>Auto : experimentalAutoEditEnabled = true
Auto->>Provider : evaluatedFeatureFlag(CodyAutoEditExperimentEnabledFeatureFlag)
Provider-->>Auto : boolean
Auto->>Auto : Compute user eligibility (subscription, agent?)
Auto-->>User : Register provider and commands
```

**Diagram sources**
- [configuration.ts:148-153](file://vscode/src/configuration.ts#L148-L153)
- [create-autoedits-provider.ts:83-122](file://vscode/src/autoedits/create-autoedits-provider.ts#L83-L122)
- [FeatureFlagProvider.ts:66-69](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L66-L69)

**Section sources**
- [create-autoedits-provider.ts:83-122](file://vscode/src/autoedits/create-autoedits-provider.ts#L83-L122)
- [configuration.ts:148-153](file://vscode/src/configuration.ts#L148-L153)
- [autoedit-completions.test.ts:43-69](file://vscode/src/autoedits/adapters/sourcegraph-completions.test.ts#L43-L69)

### Enterprise Permissions and Capability Gating
- Enterprise detection helps gate advanced features based on subscription status.
- The provider supports exposing experiments per endpoint and caching per endpoint to avoid cross-instance contamination.

```mermaid
flowchart TD
Auth["Auth Status"] --> Enterprise{"Enterprise User?"}
Enterprise --> |Yes| Allow["Allow enterprise-only features"]
Enterprise --> |No| Deny["Restrict enterprise-only features"]
Allow --> Eligibility["Compute feature eligibility"]
Deny --> Eligibility
Eligibility --> Provider["FeatureFlagProvider"]
```

**Diagram sources**
- [userProductSubscription.ts:89-104](file://lib/shared/src/sourcegraph-api/userProductSubscription.ts#L89-L104)
- [FeatureFlagProvider.ts:207-264](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L207-L264)

**Section sources**
- [userProductSubscription.ts:89-104](file://lib/shared/src/sourcegraph-api/userProductSubscription.ts#L89-L104)
- [FeatureFlagProvider.ts:207-264](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L207-L264)

### features.json5 Configuration System
- Declares feature metadata and statuses per editor (VS Code/JetBrains).
- Used for documentation and visibility; does not directly control runtime behavior.
- Integrates with the broader configuration architecture by providing human-readable feature catalogs.

```mermaid
graph LR
VS["features.json5 (VS Code)"] --> Docs["Documentation & Visibility"]
JB["features.json5 (JetBrains)"] --> Docs
Docs --> UI["VS Code package.json contributes"]
UI --> Users["Users see feature status"]
```

**Diagram sources**
- [features.json5 (VS Code):1-91](file://vscode/features.json5#L1-L91)
- [features.json5 (JetBrains):1-69](file://jetbrains/features.json5#L1-L69)
- [package.json:1-800](file://vscode/package.json#L1-L800)

**Section sources**
- [features.json5 (VS Code):1-91](file://vscode/features.json5#L1-L91)
- [features.json5 (JetBrains):1-69](file://jetbrains/features.json5#L1-L69)
- [package.json:1-800](file://vscode/package.json#L1-L800)

## Dependency Analysis
- FeatureFlagProvider depends on:
  - GraphQL client for evaluating flags
  - Auth status and endpoint information
  - Caching and reactive streams for performance and updates
- VS Code configuration depends on:
  - package.json contributions for command enablement
  - configuration-keys.ts for type-safe key mapping
- Enterprise gating depends on:
  - Subscription resolution and user product information

```mermaid
graph TB
FF["FeatureFlagProvider.ts"] --> API["client.ts"]
FF --> AUTH["Auth Status (via resolver)"]
CFG["configuration.ts"] --> FF
CFG --> PKG["package.json"]
CFG --> KEYS["configuration-keys.ts"]
SUBS["userProductSubscription.ts"] --> CFG
```

**Diagram sources**
- [FeatureFlagProvider.ts:182-358](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L182-L358)
- [client.ts:1564-1588](file://lib/shared/src/sourcegraph-api/graphql/client.ts#L1564-L1588)
- [configuration.ts:1-200](file://vscode/src/configuration.ts#L1-L200)
- [configuration-keys.ts:1-55](file://vscode/src/configuration-keys.ts#L1-L55)
- [package.json:1-800](file://vscode/package.json#L1-L800)
- [userProductSubscription.ts:89-104](file://lib/shared/src/sourcegraph-api/userProductSubscription.ts#L89-L104)

**Section sources**
- [FeatureFlagProvider.ts:182-358](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L182-L358)
- [client.ts:1564-1588](file://lib/shared/src/sourcegraph-api/graphql/client.ts#L1564-L1588)
- [configuration.ts:1-200](file://vscode/src/configuration.ts#L1-L200)
- [configuration-keys.ts:1-55](file://vscode/src/configuration-keys.ts#L1-L55)
- [package.json:1-800](file://vscode/package.json#L1-L800)
- [userProductSubscription.ts:89-104](file://lib/shared/src/sourcegraph-api/userProductSubscription.ts#L89-L104)

## Performance Considerations
- Caching: Per-endpoint caching avoids repeated network calls and ensures fast reads.
- Reactive updates: Subscriptions emit changes without requiring reloads.
- Refresh cadence: Periodic refresh keeps values fresh while minimizing overhead.
- Forced refresh: For rapidly changing flags, consumers can request immediate updates.
- No-op mode: Disabling feature flags globally avoids network calls and simplifies testing.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and resolutions:
- Flags not updating: Trigger a refresh on the provider to pull latest values.
- Confusion about feature availability: Check both configuration keys and feature flags; ensure subscription allows the feature.
- Auto-edit not appearing: Verify experimental auto-edit is enabled in configuration and the auto-edit experiment flag is true.
- Tracing not visible: Confirm experimental tracing is enabled and the autocomplete tracing flag is set.

Operational tips:
- Use the provider’s refresh method when toggling flags in the backend.
- Prefer reactive subscriptions over ephemeral evaluations to avoid stale behavior.
- Validate enterprise eligibility using subscription utilities.

**Section sources**
- [FeatureFlagProvider.ts:337-346](file://lib/shared/src/experimentation/FeatureFlagProvider.ts#L337-L346)
- [configuration.ts:144-159](file://vscode/src/configuration.ts#L144-L159)
- [create-autoedits-provider.ts:83-122](file://vscode/src/autoedits/create-autoedits-provider.ts#L83-L122)

## Conclusion
Cody’s feature flag system combines a robust provider with reactive evaluation, caching, and exposure tracking. Experimental features are surfaced through VS Code configuration and UI, while enterprise users benefit from capability gating. The features.json5 files provide documentation and visibility. By leveraging reactive subscriptions, periodic refresh, and careful validation, teams can safely roll out, test, and deprecate features with minimal disruption.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### Example Scenarios and How-To
- Enable beta auto-edit for specific users:
  - Set the experimental auto-edit configuration to true.
  - Ensure the auto-edit experiment flag is true for those users.
  - Optionally force-refresh flags if toggled mid-session.
- Test new autocomplete tracing:
  - Enable experimental tracing in configuration.
  - Toggle the autocomplete tracing feature flag.
  - Observe trace views and telemetry.
- Manage feature deprecations:
  - Keep the feature flag for a grace period.
  - Redirect behavior to a new implementation behind a new flag.
  - Announce deprecation and guide users to updated settings.

[No sources needed since this section provides general guidance]