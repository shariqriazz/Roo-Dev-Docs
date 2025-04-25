# Chapter 52: TelemetryService

Continuing from [Chapter 51: IPC (Inter-Process Communication)](51_ipc__inter_process_communication_.md), which discussed how different processes communicate, this chapter focuses on how the Roo-Code extension gathers anonymous usage data and error information to help improve the product: the **TelemetryService**.

## Motivation: Understanding Usage and Improving Roo-Code

To make Roo-Code better, the development team needs insights into how it's being used and where problems occur. Gathering anonymous telemetry data helps answer questions like:

*   Which features, commands, or tools ([Chapter 8: Tools](08_tools.md)) are most popular?
*   Which LLM providers ([Chapter 5: ApiHandler](05_apihandler.md)) and models are commonly used (by type/name, not specific user keys)?
*   How often do specific errors (API errors, validation failures ([Chapter 40: Schemas (Zod)](40_schemas__zod_.md)), tool errors) occur?
*   What is the typical duration or token usage ([Chapter 29: Cost Calculation Utilities](29_cost_calculation_utilities.md)) of tasks?
*   Are users encountering issues with specific configurations (e.g., shell integration issues detected via `no_shell_integration` event from [Chapter 15: Terminal Integration](15_terminal_integration.md))?
*   Are custom modes ([Chapter 10: CustomModesManager](10_custommodesmanager.md)) being utilized (by count, not content)?

Collecting this data, while **strictly respecting user privacy and anonymity** (see `PRIVACY.md`), allows the team to prioritize development efforts, identify bugs, improve performance, and make data-driven decisions about the product's future direction.

The `TelemetryService` provides a centralized, privacy-conscious mechanism for this data collection within the extension host.

**Central Use Case:** A user runs a "Fix Code" action ([Chapter 30: CodeActionProvider](30_codeactionprovider.md)), and the underlying API call to the LLM results in a rate limit error. The user has opted **in** to telemetry.

1.  **User Opt-in:** During initial setup (potentially via `TelemetryBanner.tsx` in the UI triggering a setting change) or via settings ([Chapter 35: Settings UI Components (WebView)](35_settings_ui_components__webview_.md)), the user enables the telemetry setting (`telemetrySetting` stored via `ContextProxy`).
2.  **Service Initialization:** On extension activation, `TelemetryService.initialize()` runs, which instantiates `PostHogClient`. `PostHogClient.updateTelemetryState` reads the consent setting and enables the underlying PostHog client.
3.  **Action Invoked:** The command handler for `roo-cline.fixCode` calls `telemetryService.captureCodeActionUsed("roo-cline.fixCode")`.
4.  **API Error:** The `ApiHandler` ([Chapter 5: ApiHandler](05_apihandler.md)) catches the rate limit error.
5.  **Error Reporting:** Error handling logic might eventually call a generic error capture method on `telemetryService`, e.g., `telemetryService.captureError("api_error", { provider: "openai", errorType: "rate_limit" })`.
6.  **`TelemetryService`/`PostHogClient` Logic:**
    *   The `capture...` methods in `TelemetryService` delegate to the underlying `PostHogClient` instance.
    *   `PostHogClient.capture` checks if telemetry is enabled (`this.telemetryEnabled`). Assume yes.
    *   It retrieves common properties (version, platform, session ID, anonymous `distinctId`). It fetches dynamic context via `provider.getTelemetryProperties()` (e.g., current mode).
    *   It merges common, provider, and event-specific properties.
    *   **(Implicit) Sanitization:** Properties passed should already be sanitized, or `PostHogClient.capture` or a helper like `sanitizeProperties` (conceptual) must filter/mask potential PII before sending.
    *   It calls the `posthog-node` client's `capture` method with the event name (e.g., `Task Code Action Used`, `Error api_error`), distinct ID, and sanitized properties.
7.  **Backend:** The PostHog service receives and aggregates these anonymized events.

If the user had opted **out**, the `this.telemetryEnabled` check in `PostHogClient.capture` would prevent any data from being sent.

## Key Concepts

1.  **User Consent (`telemetrySetting`):** A dedicated setting (`"enabled" | "disabled" | "unset"`) stored via `ContextProxy` ([Chapter 11: ContextProxy](11_contextproxy.md)). Governs whether any data is sent. An initial prompt (`TelemetryBanner.tsx` leading to setting change, or host-side `ensureTelemetryConsent`) handles the `"unset"` state. VS Code's global telemetry setting (`telemetryLevel`) is also respected by `PostHogClient.updateTelemetryState`.
2.  **Anonymity & PII Avoidance:** **Critical.** No user code, prompts, responses, keys, full file paths, specific user-defined names (use slugs/types), sensitive error details. Focus on event counts, feature flags, command IDs, provider *names*, generic model *types*, error *types*, durations, boolean flags, sanitized config values. Handled by careful property selection and potentially a `sanitizeProperties` utility ([Chapter 52: TelemetryService](52_telemetryservice.md)).
3.  **`TelemetryService` Class (`src/services/telemetry/TelemetryService.ts`):**
    *   **Facade/Wrapper:** Acts as a simple, stable public API facade. It defers initialization and delegates calls to the actual `PostHogClient`.
    *   **Initialization (`initialize`):** Creates the singleton `PostHogClient` instance *after* environment variables (like PostHog key) are loaded.
    *   **Core Methods:** Provides convenient, named methods (`captureTaskCreated`, `captureToolUsage`, `captureError`, `captureSchemaValidationError`, etc.) which internally call `PostHogClient.capture` with predefined event names (from `PostHogClient.EVENTS`) and structured properties.
    *   **Consent Update:** Provides `updateTelemetryState` to pass consent status down to the `PostHogClient`.
    *   **Provider Link (`setProvider`):** Passes the `ClineProvider` reference to `PostHogClient` for context enrichment.
    *   **Shutdown (`shutdown`):** Delegates to `PostHogClient.shutdown`.
4.  **`PostHogClient` Class (`src/services/telemetry/PostHogClient.ts`):**
    *   **Singleton:** Instantiated by `TelemetryService.initialize`.
    *   **Backend Integration:** Contains the actual `PostHog` client instance (`posthog-node`). Configured using API key/host from `process.env` (injected by [Chapter 56: Build System](56_build_system.md)).
    *   **Consent Logic (`updateTelemetryState`):** Reads the user's opt-in status *and* VS Code's global `telemetry.telemetryLevel`. Only enables internal `telemetryEnabled` flag if *both* allow telemetry (`level === "all"` and `didUserOptIn === true`). Calls `client.optIn()` or `client.optOut()`.
    *   **Core `capture` Method:** The central point for sending events. Checks `telemetryEnabled`. Fetches common properties (OS, versions, session) and dynamic properties via `provider.getTelemetryProperties()`. Merges properties. **Performs or calls sanitization logic.** Calls `this.client.capture()`.
    *   **Context Enrichment (`getTelemetryProperties` on Provider):** Relies on the linked `ClineProvider` instance ([Chapter 2: ClineProvider](02_clineprovider.md)) to provide relevant, non-sensitive dynamic context (like `mode`, `apiProvider`). `WeakRef` is used to avoid cycles.
    *   **Shutdown (`shutdown`):** Calls `this.client.shutdown()` to flush the event buffer.
5.  **Session/User IDs:**
    *   **Session ID:** Not explicitly shown in `PostHogClient`, but potentially added to common properties if needed.
    *   **Distinct ID (`distinctId`):** Uses `vscode.env.machineId` (anonymous installation ID).
6.  **Event Structure:** Events sent via `PostHogClient.capture` follow PostHog format (`distinctId`, `event`, `properties`). Event names are defined in `PostHogClient.EVENTS`. Properties are merged (common + provider + event-specific) and sanitized.
7.  **Sanitization (`sanitizeProperties` - Conceptual):** A crucial (though not explicitly shown in provided code snippets) function responsible for removing/masking PII from the `properties` object before it's sent to PostHog. See conceptual implementation in Chapter 52 Key Concepts.
8.  **WebView Telemetry (`TelemetryClient.ts`, `TelemetryBanner.tsx`):** A *separate* mechanism potentially exists for the WebView UI itself (using `posthog-js`). `TelemetryBanner` prompts for initial consent, sending the choice (`telemetrySetting` message) to the host. `TelemetryClient` would initialize `posthog-js` based on state received from the host (`TelemetrySetting`, API key, distinctId) and capture UI-specific events (e.g., button clicks within the WebView). Communication between host and WebView telemetry settings is managed via `ExtensionStateContext`.

## Using the TelemetryService

The host-side `telemetryService` singleton is imported and used.

**1. Initialization & Consent (Extension Activation):**

```typescript
// --- File: src/extension.ts ---
import { TelemetryService, telemetryService } from "./services/telemetry/TelemetryService"; // Import singleton
import { ContextProxy } from "./core/config/ContextProxy";
import * as vscode from "vscode";
import { ClineProvider } from "./core/webview/ClineProvider";
import { formatLanguage } from "./shared/language"; // Needed for i18n init

// Helper to prompt user for consent if needed
async function ensureTelemetryConsent(contextProxy: ContextProxy): Promise<void> {
    const telemetrySettingKey = "telemetrySetting";
    const currentSetting = contextProxy.getValue(telemetrySettingKey as any) ?? 'unset';
    if (currentSetting === 'unset') {
        const selection = await vscode.window.showInformationMessage(/* ... prompt text ... */);
        const settingValue = selection === "Enable Telemetry" ? "enabled" : "disabled";
        await contextProxy.setValue(telemetrySettingKey as any, settingValue);
        // Service should pick up change via onDidChangeConfiguration,
        // but capture initial consent action explicitly IF service is already enabled.
        // Need careful timing here. Let's assume updateConsent handles the PostHog client init.
        // We capture the event *after* potentially enabling the service.
        if (telemetryService.isTelemetryEnabled()) { // Check AFTER consent potentially set
             telemetryService.captureEvent("consent.set", { consent_given: settingValue === "enabled", source: "initial_prompt" });
        }
    }
}

export async function activate(context: vscode.ExtensionContext) {
    // Load .env variables FIRST (for PostHog key)
    // ... dotenvx.config() ...

    // Initialize i18n needed by other components
    initializeI18n(context.globalState.get("language") ?? formatLanguage(vscode.env.language))

    // Initialize ContextProxy
    const contextProxy = new ContextProxy(context);
    await contextProxy.initialize();

    // Initialize TelemetryService - reads consent via proxy, inits PostHog client
    telemetryService.initialize(); // Now simplified, uses internal access if needed or relies on updateConsent
    // Or pass proxy if needed: await telemetryService.init(context, contextProxy);

    // Initialize other services like TerminalRegistry, McpServerManager
    // ...

    // Create Provider AFTER telemetry service is ready
    const provider = new ClineProvider(context, outputChannel);
    // Link provider so telemetry can get context from it
    telemetryService.setProvider(provider);

    // Prompt for consent only after everything is set up
    // (Run async, don't block activation)
    ensureTelemetryConsent(contextProxy).catch(e => console.error("Consent prompt failed:", e));

    // Listen for config changes to update consent
    context.subscriptions.push(vscode.workspace.onDidChangeConfiguration(async e => {
        const fullSettingKey = 'roo-code.telemetrySetting'; // Use actual setting ID
        if (e.affectsConfiguration(fullSettingKey)) {
            const oldEnabled = telemetryService.isTelemetryEnabled();
            // Re-read setting and update internal state/PostHog client
            telemetryService.updateTelemetryState(contextProxy.getValue(telemetrySettingKey as any) === "enabled");
            if (oldEnabled !== telemetryService.isTelemetryEnabled()) {
                 telemetryService.captureEvent("consent.updated", { consent_given: telemetryService.isTelemetryEnabled(), source: "settings_change" });
            }
        }
    }));

    // ... Register commands, providers, URI handler ...
    // ... Final activation steps ...

    // Capture activation event
    telemetryService.captureEvent("extension.activated");
}

export async function deactivate(): Promise<void> {
    await telemetryService.shutdown(); // Flush events on exit
}
```
*Explanation:* Initializes `TelemetryService` early after env vars/proxy are ready. Links the `ClineProvider` using `setProvider`. Prompts for initial consent if needed. Listens for configuration changes to update consent status and capture change events. Calls `shutdown` on deactivation. Captures an activation event.

**2. Capturing Events:**

```typescript
// Example: Tool usage
telemetryService.captureToolUsage("my_task_id", "read_file");

// Example: Mode switch
telemetryService.captureModeSwitch("my_task_id", "new_mode_slug");

// Example: Generic event
telemetryService.captureEvent("ui.button.clicked", { button_id: "save_settings" });
```
*Explanation:* Use the specific helper methods (`captureToolUsage`, `captureModeSwitch`) or the generic `captureEvent` with non-sensitive properties.

**3. Capturing Errors:**

```typescript
// Zod Validation Error in CustomModesManager
if (!result.success) {
    telemetryService.captureSchemaValidationError({
        schemaName: "CustomModesSettings", error: result.error
    });
    // ...
}

// Uncaught Exception Handler (conceptual)
process.on('uncaughtException', async (error) => {
    console.error('Uncaught Exception:', error);
    telemetryService.captureException(error); // Capture limited info
    // Ensure telemetry flushes before exiting
    await telemetryService.shutdown();
    process.exit(1); // Exit uncleanly
});
```
*Explanation:* Use `captureSchemaValidationError` for Zod errors. Use `captureException` for uncaught errors, capturing minimal stack info.

## Code Walkthrough

### TelemetryService (`src/services/telemetry/TelemetryService.ts`)

*(See full code in chapter context)*
*   Acts as a facade. `initialize` creates the `PostHogClient` singleton.
*   Public methods (`captureEvent`, `captureTaskCreated`, `captureSchemaValidationError`, etc.) mostly delegate to the `client` instance, often formatting the event name or properties slightly.
*   `updateTelemetryState` passes the user's consent choice down to the `PostHogClient`.
*   `setProvider` links the `ClineProvider` for context fetching by `PostHogClient`.
*   `isTelemetryEnabled` checks the client's status.
*   `shutdown` delegates to the client.

### PostHogClient (`src/services/telemetry/PostHogClient.ts`)

*(See full code in chapter context)*
*   Singleton pattern (`getInstance`).
*   Constructor initializes `posthog-node` client using env vars (`POSTHOG_API_KEY`, `POSTHOG_HOST`).
*   `updateTelemetryState`: **Combines user opt-in (`didUserOptIn`) with VS Code global setting (`telemetry.telemetryLevel`)**. Only sets `this.telemetryEnabled = true` if *both* allow it. Calls `this.client.optIn()` or `this.client.optOut()`.
*   `setProvider`: Stores `WeakRef<ClineProviderInterface>`.
*   `capture`:
    *   Checks `this.telemetryEnabled`.
    *   Calls `provider.getTelemetryProperties()` to get dynamic context (e.g., mode). Handles errors.
    *   Merges common, provider, and event properties.
    *   **(MISSING STEP - CRITICAL): Explicitly calls `sanitizeProperties(mergedProperties)` before sending.** The current code seems to assume properties are already safe or relies on PostHog filtering, which is risky. This step MUST be added.
    *   Calls `this.client.capture(...)` with `distinctId`, event name, and **sanitized** properties.
*   `isTelemetryEnabled`: Returns internal flag.
*   `shutdown`: Calls `this.client.shutdown()`.

### Sanitization (`src/services/telemetry/sanitize.ts` - Conceptual)

*(See refined conceptual code in Key Concepts section of Chapter 52)*
*   Function `sanitizeProperties` takes `properties: Record<string, any>`.
*   Iterates through keys/values.
*   Removes properties matching sensitive key patterns.
*   Replaces long strings or path-like strings with placeholders.
*   Allows safe primitives (number, boolean, null).
*   Handles simple objects/arrays by size limit (or skips/masks complex ones).
*   Returns the sanitized properties object.

## Internal Implementation

1.  **Event Trigger:** Code calls `telemetryService.capture...()`.
2.  **Delegation:** `TelemetryService` calls `postHogClient.capture(...)`.
3.  **Consent Check:** `PostHogClient` checks `this.telemetryEnabled` (which considers both user opt-in and VS Code global setting).
4.  **Context/Sanitize:** `PostHogClient` merges context properties, **calls `sanitizeProperties` (needs explicit call)**.
5.  **Client Call:** `PostHogClient` calls `this.client.capture()` (from `posthog-node`).
6.  **PostHog Client Lib:** Buffers event, sends batches asynchronously via HTTPS.
7.  **Shutdown:** `telemetryService.shutdown()` -> `postHogClient.shutdown()` flushes buffer.

## Modification Guidance

Involves adding events, context, or refining sanitization/consent.

1.  **Adding New Event:** Identify trigger, call appropriate `telemetryService.capture...` method with clear name and **safe** properties. Update `sanitizeProperties` if needed.
2.  **Adding Context:** Add static data to `PostHogClient` common properties (if truly static). Add safe dynamic data retrieval to `ClineProvider.getTelemetryProperties()`. Pass more safe data directly via event properties.
3.  **Refining Sanitization (`sanitizeProperties`):** **Critical.** Add sensitive key patterns. Improve detection/anonymization of paths, URLs, long strings. Test rigorously.
4.  **Improving Consent Flow:** Enhance the `ensureTelemetryConsent` prompt, potentially link to `PRIVACY.md`. Ensure `updateConsent` is called correctly from all relevant places (init, settings change).

**Best Practices:**

*   **Privacy First & Opt-In:** Non-negotiable. Rigorous sanitization, clear consent prompt, respect global VS Code setting.
*   **Centralized Service:** Use singleton facade (`TelemetryService`).
*   **Sanitize Rigorously:** Continuously review `sanitizeProperties`. **Ensure it's called before sending data.**
*   **Meaningful Events:** Track actionable insights. Use consistent naming (`feature.action`, `error.type`).
*   **Safe Context:** Only add non-sensitive, generalized context.
*   **Graceful Failure:** Service/client init/capture failures shouldn't crash extension.
*   **Shutdown:** Implement `shutdown`.
*   **Transparency:** Maintain `PRIVACY.md`.

**Potential Pitfalls:**

*   **PII Leakage:** Insufficient sanitization or forgetting to call `sanitizeProperties`. **Highest risk.**
*   **Consent Bypass:** Not correctly checking both user setting AND global VS Code setting (`telemetryLevel`). Sending data when disabled.
*   **Missing API Key:** Build process fails to inject `POSTHOG_PROJECT_API_KEY`. Telemetry client won't initialize.
*   **Network/Backend Issues:** Data loss (less critical).
*   **Sanitization Errors:** Bugs in `sanitizeProperties`.

## Conclusion

The `TelemetryService`, acting as a facade for the `PostHogClient`, provides a crucial, privacy-conscious mechanism for collecting anonymous usage data and error information in Roo-Code. It strictly adheres to user consent (both explicit opt-in and VS Code's global setting) and employs sanitization routines to prevent the transmission of PII. By capturing standardized events related to feature usage, errors, and performance, and enriching them with safe context, the service provides valuable insights for the development team while prioritizing user privacy. Proper initialization, consent handling, rigorous sanitization, and graceful shutdown are key elements of its implementation.

Next, we explore the system designed for evaluating Roo-Code's performance on predefined tasks: [Chapter 53: Evals System](53_evals_system.md).
---
# Chapter 53: Evals System

Continuing from [Chapter 52: TelemetryService](52_telemetryservice.md), which focused on collecting anonymous usage data for product improvement, this chapter introduces a system designed for more structured and objective assessment of Roo-Code's capabilities: the **Evals System**. This system allows developers to define benchmark tasks and automatically run Roo-Code against them to measure performance, prevent regressions, and compare different models or configurations.

## Motivation: Measuring and Improving AI Performance

While user feedback and telemetry provide valuable insights, objectively measuring the quality and performance of an AI assistant like Roo-Code requires a more systematic approach. We need a way to:

1.  **Define Test Cases:** Create standardized tasks (prompts, context, expected outcomes) representing common scenarios. Roo-Code uses YAML files in `evals/suites/`.
2.  **Run Automatically:** Execute Roo-Code against these cases using different models/configs repeatably, outside the interactive UI. Handled by the runner script in `evals/apps/cli/src/index.ts`.
3.  **Capture Outputs:** Record the full interaction transcript (AI text, tool usage), token counts, and costs. Results stored via `@evals/db`.
4.  **Evaluate Results:** Compare outputs against predefined criteria (defined in YAML) or use subsequent LLM/manual review. Facilitated by the `evals-web/` UI.
5.  **Track Progress:** Monitor how changes (prompts [Ch 7], models, context [Ch 23], tools [Ch 8]) affect performance over time.

The **Evals System** (`evals/` directory including CLI, IPC, types, schemas, DB, and `evals-web/` UI) provides this framework for data-driven development and quality assurance.

**Central Use Case:** Developer modifies system prompt ([Chapter 7](07_systemprompt.md)), tests impact on Python code generation.

1.  **Define Suite:** `evals/suites/python_codegen.yaml` defines task `py_fib_recursive` (prompt, criteria, `config_overrides: { mode: "code" }`). Validated by `evalTaskSchema` ([Chapter 40](40_schemas__zod_.md)).
2.  **Run Evals:** Developer runs `pnpm start --suite python_codegen --model gpt-4-turbo` from `evals/apps/cli/`.
3.  **Eval CLI Runner (`cli/src/index.ts` / `runExercise`):**
    *   Parses args (`gluegun`). Finds YAML (`getExercises`). Creates DB records (`@evals/db`).
    *   Starts IPC server (`IpcServer` from `@evals/ipc`) on socket path.
    *   Launches VS Code (`execa code ...`) with extension path, exercise workspace, and `ROO_CODE_IPC_SOCKET_PATH` env var.
    *   Waits for Roo-Code (IPC client) to connect.
    *   Sends `StartNewTask` command (IPC) with prompt/config. **Requires API key for model via env vars.**
    *   Receives `TaskEvent` stream (IPC) from Roo-Code (transcript, metrics). Updates DB.
    *   Optionally runs unit tests (`runUnitTest`). Updates DB `passed` status. Sends `Pass`/`Fail` event (IPC).
4.  **Review Results:** Developer uses `evals-web` UI ([Chapter 54](54_shadcn_ui_primitives__evals_web_.md)) (reads DB) to compare transcript against `criteria` and previous runs.
5.  **Analysis:** Assess impact of the change.

## Key Concepts

1.  **Evaluation Suites (`evals/suites/*.yaml`):** Define tasks (`EvalTask`: `id`, `prompt`, `context`, `criteria`, `config_overrides`). Validated by `evalTaskSchema`.
2.  **Eval CLI Runner (`evals/apps/cli/src/index.ts`):** Node.js app (`gluegun`). Parses args, loads/validates suites, manages DB records (`@evals/db`), orchestrates VS Code launch and task execution via IPC. Uses script utilities ([Chapter 55](55_script_utilities.md)).
3.  **Database (`@evals/db`):** Stores run/task metadata, status, metrics, possibly transcripts. Allows querying results.
4.  **IPC (`@evals/ipc`, `src/exports/ipc.ts`):** Uses `node-ipc` over sockets/pipes for CLI (server) <-> Extension (client) communication. Defines/validates messages (`ipcMessageSchema`, `taskCommandSchema`, `taskEventSchema`). See [Chapter 51](51_ipc__inter_process_communication_.md).
5.  **Programmatic Invocation:** Roo-Code host listens for IPC `StartNewTask`. Triggers core logic (`API.startNewTask` -> `ClineProvider`). **Requires mocking VS Code dependencies.** `API.emit` override sends `TaskEvent`s back via IPC.
6.  **Output/Transcript:** Sequence of `TaskEvent` payloads received via IPC, stored in DB.
7.  **Evaluation/Analysis:** Manual review against `criteria` via `evals-web` UI, or automated metrics (`runUnitTest` results, token/cost).
8.  **Web UI (`evals-web/`):** Separate Next.js app reading from `@evals/db`. Uses standard web shadcn/ui primitives ([Chapter 54](54_shadcn_ui_primitives__evals_web_.md)).

## Using the Evals System

Primarily a developer tool.

**Workflow:**

1.  **Define Suites:** Create/edit `evals/suites/*.yaml`.
2.  **Setup Exercises:** Ensure code exists in `evals/exercises/`.
3.  **Configure Runner:** Set API keys as environment variables.
4.  **Run Runner:** `pnpm start --language <lang> --exercise <name> [--model <id>]` from `evals/apps/cli/`.
5.  **Monitor:** Check console output.
6.  **Analyze:** Use `evals-web` UI (`pnpm dev` in `evals-web/`).
7.  **Iterate:** Modify code/prompts, re-run, compare.

## Code Walkthrough

### Eval CLI (`evals/apps/cli/src/index.ts`)

*(See code provided in Chapter 51)*
*   Uses `gluegun`. Interacts with `@evals/db`. Creates `IpcServer`.
*   `runExercise` function launches VS Code (`execa`), creates `IpcClient`, waits for connection, listens for `TaskEvent`s (updates DB), sends `StartNewTask`, waits for completion, runs `runUnitTest` (updates DB `passed`), cleans up. Uses `pMap` for concurrency.

### IPC Library (`@evals/ipc`, `@evals/types`)

*(See code provided in Chapter 51)*
*   **Types (`ipc.ts`):** Zod schemas (`ackSchema`, `taskCommandSchema`, `taskEventSchema`, `ipcMessageSchema`) + TS types. Enums for names/types/origins.
*   **Client (`client.ts`):** `IpcClient` wraps `node-ipc`. Connects, validates messages, emits typed events. Sends validated messages.
*   **Server (`server.ts`):** `IpcServer` wraps `node-ipc`. Listens, manages clients, validates messages, emits typed events. Sends/broadcasts messages.

### Roo-Code Host IPC Client (`src/exports/api.ts` or `src/exports/ipc.ts`)

*(See corrected conceptual code in Chapter 51)*
*   **Init:** Checks `process.env.ROO_CODE_IPC_SOCKET_PATH`, creates `IpcClient`.
*   **Command Handling:** Listens for `TaskCommand`, validates, triggers `API`/`ClineProvider` actions, **applying config overrides**.
*   **Event Forwarding:** `API.emit` override intercepts events, formats as `TaskEvent` (validates), sends via `ipcClient.sendMessage`.

### Schemas (`evals/schemas/` or `@evals/types`)

*   **`evalTaskSchema`:** Zod schema for YAML task definitions (`id`, `prompt`, `context`, `criteria`, `config_overrides`).
*   **`evalOutputSchema` (Conceptual):** Zod schema for output (JSON or DB structure). Includes `transcript: z.array(clineMessageSchema)`, `metrics`.

### Web UI (`evals-web/`)

*   Separate Next.js/React project. Reads from `@evals/db`. Uses standard web shadcn/ui primitives ([Chapter 54](54_shadcn_ui_primitives__evals_web_.md)). Renders transcripts using `MarkdownBlock` concepts ([Chapter 46](46_markdown_rendering.md)).

## Internal Implementation

1.  **Runner:** Starts IPC server, launches VS Code (with IPC path).
2.  **VS Code Host:** Starts IPC client, connects.
3.  **Runner:** Sends `StartNewTask` (IPC).
4.  **VS Code Host:** Receives command, calls `API.startNewTask` -> `ClineProvider.initClineWithTask` (with overrides, **mocked VS Code deps**).
5.  **`Cline`:** Runs task loop. Emits events.
6.  **VS Code Host (`API.emit`):** Intercepts events, formats as `TaskEvent`, sends back (IPC).
7.  **Runner:** Receives `TaskEvent`s, logs, updates DB.
8.  **Completion:** Runner gets `TaskCompleted`, updates DB. Runs `runUnitTest`. Updates DB `passed`. Sends `Pass`/`Fail` event (IPC).
9.  **Cleanup:** Runner sends `CloseTask`, disconnects client.
10. **Web UI:** Reads DB, displays results.

**Sequence Diagram (Eval Task Execution):**

*(See diagram in Chapter 53 - Internal Implementation)*

## Modification Guidance

Involves adding suite features, runner options, metrics, or enhancing analysis UI.

1.  **Adding New Context Type (e.g., Mock Terminal State):**
    *   **Schema (`evalTaskSchema`):** Add field to `context`.
    *   **YAML:** Add context to tasks.
    *   **`evaluateTask.ts`/Invocation:** Pass mock state to mocked `TerminalRegistry`.
    *   **Core Logic:** Ensure mocked terminal uses state.

2.  **Adding Automated Metric (e.g., Code Linting):**
    *   **`evaluateTask.ts`:** Extract code, run linter via `execa`, record results in DB metrics.
    *   **Schema/DB:** Add metric field.
    *   **Analysis/UI:** Display/use metric. Consider security.

3.  **Supporting Parallel Execution:**
    *   **`runEval.ts`:** Use `p-limit` around `runExercise`. Generate unique socket paths. Manage multiple VS Code instances/IPC connections.
    *   **Considerations:** Resource usage, API rate limits.

4.  **Enhancing Web UI (`evals-web/`):**
    *   Modify Next.js pages/components. Use shadcn/ui primitives ([Chapter 54](54_shadcn_ui_primitives__evals_web_.md)) for filtering, sorting, diff views, charts. Read data from `@evals/db`.

**Best Practices:**

*   **Clear Criteria:** Define specific, verifiable criteria.
*   **Realistic Context:** Use representative prompts/context.
*   **Isolate Core Logic:** Clean programmatic entry point minimizing VS Code dependencies.
*   **Robust Mocking:** Mock VS Code dependencies accurately. Mock tool interactions predictably.
*   **Structured Output:** Use database (`@evals/db`) or schemas (`evalOutputSchema`) for JSON. Store inputs+outputs.
*   **Version Control Suites:** Keep YAMLs in Git.
*   **Reproducibility:** Record config. Fix model versions.
*   **Balance Automated/Manual:** Use automated metrics + manual review via Web UI.
*   **Secure API Keys:** Use environment variables (`process.env`).

**Potential Pitfalls:**

*   **Environment Differences:** Evals outside VS Code behaving differently due to mocking.
*   **Mocking Complexity:** Hard to accurately mock full VS Code environment for `Cline`.
*   **Context Setup/Cleanup Errors.**
*   **API Keys/Costs:** Secure management. Cost of runs.
*   **Flaky LLMs/Tests:** Non-determinism.
*   **IPC Failures:** Connection issues, validation errors.
*   **Eval Suite Maintenance.**

## Conclusion

The Evals System provides an essential framework for systematically evaluating and improving the performance of Roo-Code's AI capabilities. By defining structured test suites (YAML), using a CLI runner script (`runEval.ts`, `evaluateTask.ts`) to execute tasks programmatically (launching VS Code and communicating via IPC), storing detailed results in a database (`@evals/db`), and providing a web UI (`evals-web/`) for analysis, developers can objectively measure correctness, prevent regressions, and guide future improvements. While setting up the programmatic invocation and IPC requires careful handling of dependencies, mocking, and error states, the benefits of automated, data-driven quality assessment are significant for developing a reliable AI assistant.

Next, we look at the UI primitives specifically used within the Evals web interface: [Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md).
---
# Chapter 54: Shadcn/UI Primitives (Evals Web)

Continuing from [Chapter 53: Evals System](53_evals_system.md), which detailed the framework for running automated evaluations of Roo-Code's AI, this chapter focuses on the user interface components used within the **Evals Web** application – the separate web interface designed for reviewing and analyzing evaluation results. Specifically, we look at the **Shadcn/UI Primitives** used in this context.

## Motivation: Building a Dedicated Web UI for Eval Analysis

While the main Roo-Code extension UI ([Chapter 1: WebView UI](01_webview_ui.md)) runs within VS Code and needs to match its theme precisely, the Evals Web UI is a standalone web application (likely built with Next.js or a similar framework like Vite/React) running in a standard browser. Its purpose is different: presenting potentially large amounts of structured evaluation data (results from the `evals` database (`@evals/db`) or `evals/output/` JSON files), enabling comparisons between different runs (e.g., different models, different code versions), filtering, sorting, and potentially annotating results.

This separate context allows for different UI choices. While consistency with VS Code is less critical, building a clean, modern, and functional web interface still requires a good set of UI components. `shadcn/ui` is again chosen for this purpose, but used in a more standard web configuration compared to its adaptation for the VS Code WebView ([Chapter 33: Shadcn/UI Primitives (WebView)](33_shadcn_ui_primitives__webview_.md)).

The reasons for using `shadcn/ui` here are similar to its use in the WebView, but with slightly different emphasis:

1.  **Rich Component Set:** Provides essential components for building data-rich web applications: `Table`, `Dialog`, `Select`, `Tabs`, `Card`, `Button`, `Badge`, `Tooltip`, `Accordion`, `Input`, `Checkbox`, `Separator`, `DropdownMenu`, `Popover`, etc., suitable for displaying complex evaluation results.
2.  **Composability & Customization:** Components are easily composed and fully customizable by having the source code directly in the `evals-web` project (`evals-web/components/ui/`). This allows tailoring the components specifically for the needs of the evaluation dashboard.
3.  **Modern Styling (Tailwind CSS):** Uses Tailwind CSS for styling, allowing for rapid development of a clean, modern aesthetic suitable for a standalone web application. Theming (light/dark mode) uses standard Tailwind theme configuration and CSS variables, independent of the VS Code environment.
4.  **Accessibility:** Built on Radix UI primitives, ensuring good accessibility out of the box.

**Central Use Case:** Displaying a table comparing the results of the same evaluation task (e.g., `py_fib_recursive` from [Chapter 53: Evals System](53_evals_system.md)) run against different LLM models (`gpt-4-turbo`, `claude-3-opus`).

1.  The Evals Web app (a Next.js page) fetches the relevant evaluation results (`TaskWithRun` records) for the task across different models from the `@evals/db` package.
2.  A React component within `evals-web` needs to render this comparison.
3.  It uses the `Table` components (`Table`, `TableHeader`, `TableRow`, `TableHead`, `TableBody`, `TableCell`) from `evals-web/components/ui/` (the standard shadcn/ui primitives copied into this project).
4.  The table has columns for "Model", "Status", "Cost", "Tokens", and potentially a link/button to view the full transcript.
5.  Data for each model's result is mapped to a `TableRow`. `TableCell` renders individual data points.
6.  Components like `Badge` display the status ("Success", "Error", "Passed"/"Failed"). `Tooltip` might show cost details. `Button` (or a link) navigates to a detailed view.
7.  The table renders with clean styling defined by Tailwind CSS utility classes, using the standard web theme configured in `evals-web/tailwind.config.js`.

## Key Concepts

1.  **Standalone Web Application (`evals-web/`):** Runs independently in a browser, built with Next.js/React. Separate dependencies and styling from the VS Code extension. Reads data primarily from the `@evals/db` package.
2.  **Shadcn/UI Primitives (`evals-web/components/ui/`):** Source code for standard shadcn/ui components (e.g., `Button`, `Table`, `Dialog`, `Badge`) copied into the project for customization. ([https://ui.shadcn.com/](https://ui.shadcn.com/))
3.  **Standard Web Styling:** Uses Tailwind CSS with a standard web theme (`evals-web/tailwind.config.js` + `globals.css`). Defines colors (`primary`, `background`) using CSS variables for light/dark mode, **not** VS Code variables (`--vscode-*`).
4.  **Radix UI & Tailwind CSS:** Leverages Radix UI for behavior/accessibility and Tailwind for styling.
5.  **Component Set:** Includes `Table`, `Card`, `Badge`, `Tabs`, `Select`, `DropdownMenu`, `Dialog`, `Popover`, `Tooltip`, `Button`, `Input`, `Checkbox`, `Separator`, `Accordion`, etc.
6.  **`cn` Utility:** Standard utility (`clsx` + `tailwind-merge`) for class name composition.

## Using the Primitives in Evals Web

Usage follows standard React development patterns within the `evals-web` application. Components are imported from the local `components/ui` directory (aliased as `@/components/ui`).

**Example: Rendering a Comparison Table (Conceptual)**

```typescript
// --- File: evals-web/components/results/ComparisonTable.tsx ---
import React from 'react';
import {
  Table, TableBody, TableCell, TableHead, TableHeader, TableRow,
} from '@/components/ui/table'; // Import Table components
import { Badge } from '@/components/ui/badge'; // Import Badge
import { Button } from '@/components/ui/button'; // Import Button
import { Tooltip, TooltipContent, TooltipProvider, TooltipTrigger } from '@/components/ui/tooltip'; // Import Tooltip
import Link from 'next/link'; // Import Next.js Link
// Import type from the database package
import type { TaskWithRun } from '@evals/db'; // Assuming a type combining Task and Run info

interface ComparisonTableProps {
  tasks: TaskWithRun[]; // Array of task results for the same exercise from different runs/models
}

const ComparisonTable: React.FC<ComparisonTableProps> = ({ tasks }) => {
  // Formatters
  const formatCost = (cost?: number | null) => cost !== undefined && cost !== null ? `$${cost.toFixed(6)}` : 'N/A';
  const formatTokens = (tokens?: number | null) => tokens?.toLocaleString() ?? 'N/A';
  const getStatusVariant = (task: TaskWithRun): "default" | "destructive" | "secondary" | "outline" => {
    if (task.error) return "destructive";
    if (task.passed === true) return "default"; // Success/Passed
    if (task.passed === false) return "secondary"; // Failed unit test
    if (!task.finishedAt) return "outline"; // In progress or unknown
    return "default"; // Completed but no test result yet
  };
   const getStatusText = (task: TaskWithRun): string => {
    if (task.error) return "Error";
    if (task.passed === true) return "Passed";
    if (task.passed === false) return "Failed";
    if (!task.finishedAt) return "Running";
    return "Completed";
  };

  return (
    <TooltipProvider> {/* Required for Tooltips */}
      <Table>
        <TableHeader>
          <TableRow>
            <TableHead>Run/Model</TableHead>
            <TableHead>Status</TableHead>
            <TableHead className="text-right">Cost</TableHead>
            <TableHead className="text-right">Tokens In</TableHead>
            <TableHead className="text-right">Tokens Out</TableHead>
            <TableHead className="text-right">Duration</TableHead>
            <TableHead>Actions</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          {tasks.map((task) => {
            const metrics = task.metrics?.[0]; // Assuming one metrics entry per task
            const modelId = task.run?.model ?? 'Unknown Model';
            const resultKey = task.id;
            const resultDetailPath = `/runs/${task.runId}/tasks/${task.id}`; // Link based on DB IDs

            return (
              <TableRow key={resultKey}>
                <TableCell className="font-medium">{modelId}</TableCell>
                <TableCell>
                  <Badge variant={getStatusVariant(task)}>
                    {getStatusText(task)}
                  </Badge>
                </TableCell>
                <TableCell className="text-right">
                  <Tooltip>
                    <TooltipTrigger asChild>
                       <button type="button" className="cursor-default">{formatCost(metrics?.cost)}</button>
                    </TooltipTrigger>
                    <TooltipContent>
                      <p>Input: {formatTokens(metrics?.tokensIn)}</p>
                      <p>Output: {formatTokens(metrics?.tokensOut)}</p>
                      <p>Cache Reads: {formatTokens(metrics?.cacheReads)}</p>
                    </TooltipContent>
                  </Tooltip>
                </TableCell>
                <TableCell className="text-right">{formatTokens(metrics?.tokensIn)}</TableCell>
                <TableCell className="text-right">{formatTokens(metrics?.tokensOut)}</TableCell>
                <TableCell className="text-right">
                    {metrics?.duration ? `${(metrics.duration / 1000).toFixed(1)}s` : 'N/A'}
                </TableCell>
                <TableCell>
                  <Link href={resultDetailPath} passHref legacyBehavior>
                    <Button variant="outline" size="sm" asChild>
                      <a>View Details</a>
                    </Button>
                  </Link>
                </TableCell>
              </TableRow>
            );
          })}
        </TableBody>
      </Table>
    </TooltipProvider>
  );
};

export default ComparisonTable;
```

**Explanation:**

1.  **Imports:** Imports primitives from `@/components/ui` within `evals-web`. Imports DB type `TaskWithRun` from `@evals/db`.
2.  **Composition:** Uses `Table`, `Badge`, `Tooltip` (with `TooltipProvider`), `Button`.
3.  **Styling:** Uses Tailwind utilities (`text-right`, `font-medium`). `Badge` uses variants based on `task.error` and `task.passed`. Styles derived from the web theme.
4.  **Data Mapping:** Maps `tasks` array (`TaskWithRun[]`). Accesses data from `task` and nested `task.run` / `task.metrics`. Formats data using helpers.
5.  **Interaction:** Uses Next.js `Link` + `Button` with `asChild` for navigation.

## Code Walkthrough

Components in `evals-web/components/ui/` are standard shadcn/ui implementations, styled via Tailwind and web theme CSS variables defined in `globals.css`.

### Example Primitive (`evals-web/components/ui/table.tsx`)

*(See code in Chapter 54 - Code Walkthrough)*
*   Wraps standard HTML table elements.
*   Applies base Tailwind classes using semantic web theme colors (`muted`, `muted-foreground`).

### Tailwind Configuration (`evals-web/tailwind.config.js` - Conceptual)

*(See code in Chapter 54 - Key Concepts)*
*   Defines standard web color palette (light/dark) using CSS variables defined in `globals.css`. **Not** VS Code variables.

## Internal Implementation

Standard React rendering flow with Tailwind CSS:
1.  Component uses `<Badge variant="destructive">`.
2.  CVA/`cn` generate classes: `... bg-destructive ...`.
3.  React renders `<div class="...">`.
4.  Tailwind build generates CSS: `.bg-destructive { background-color: hsl(var(--destructive)); } ...`.
5.  Global CSS (`globals.css`) defines `--destructive` HSL value for web theme.
6.  Browser applies styles, rendering standard red web badge.

## Modification Guidance

Follow standard web development practices for shadcn/ui and Tailwind within the `evals-web` project.

1.  **Changing Evals Web Theme:** Edit HSL values in `globals.css` and/or semantic mappings in `evals-web/tailwind.config.js`. Rebuild CSS.
2.  **Adding/Adapting Primitives:** Use `npx shadcn-ui@latest add ...` or copy source into `evals-web/components/ui/`. Standard components work directly with the web theme. No VS Code variable mapping needed. Update Tailwind config if necessary.

**Best Practices:**

*   **Standard Shadcn/UI:** Use components as documented.
*   **Theme via CSS Variables:** Define palette in global CSS.
*   **Tailwind Configuration:** Ensure config is correct for `evals-web`.
*   **Keep Separate:** Maintain distinction from WebView UI components/theme.

**Potential Pitfalls:**

*   **Incorrect Tailwind Setup:** Broken styles.
*   **CSS Conflicts:** Other global styles interfering.
*   **Build Process Integration:** Ensure Tailwind build works with Next.js/Vite.
*   **Dependency Mismatches:** Radix UI, etc. compatibility within `evals-web`.

## Conclusion

The shadcn/ui primitives provide a modern, flexible, and efficient way to build the user interface for the standalone **Evals Web** application (`evals-web/`). By adopting the standard shadcn/ui components and styling them with a conventional web-oriented Tailwind CSS theme, the `evals-web` project creates sophisticated data displays (like tables and cards) and interactions necessary for analyzing and comparing AI evaluation results retrieved from the `@evals/db`. These components are intentionally styled differently from their counterparts in the main VS Code WebView, reflecting their different execution context and purpose.

Next, we look at utilities specifically created for scripting tasks within the Roo-Code project: [Chapter 55: Script Utilities](55_script_utilities.md).
---
# Chapter 55: Script Utilities

Continuing from [Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md), which focused on UI components for the evaluation web interface, this chapter shifts focus to backend utilities used across various development and operational scripts within the Roo-Code project: the **Script Utilities**.

## Motivation: Common Helpers for Development and Build Scripts

Beyond the core extension runtime and UI code, a software project like Roo-Code involves numerous supporting scripts for tasks such as:

*   Building the extension ([Chapter 56: Build System](56_build_system.md)).
*   Running linters, formatters, or tests ([Chapter 57: Testing Framework](57_testing_framework.md)).
*   Generating documentation or schemas (e.g., `scripts/prepare.js`).
*   Running evaluation suites ([Chapter 53: Evals System](53_evals_system.md)).
*   Performing release tasks (e.g., updating versions, packaging).
*   Finding missing translation keys (e.g., `scripts/find-missing-translations.js` from [Chapter 50: Localization System (i18n)](50_localization_system__i18n_.md)).

These scripts, often written in TypeScript or JavaScript and executed using Node.js (e.g., via `pnpm run script-name`), frequently require common helper functionalities:

*   Parsing command-line arguments (`yargs`).
*   Interacting with the file system robustly (reading directories, finding files (`glob`), checking paths). ([Chapter 42](42_file_system_utilities.md) & [Chapter 43](43_path_utilities.md) utilities reused/adapted).
*   Executing shell commands reliably and capturing output (`execa`).
*   Handling logging consistently with clear formatting (`chalk`).
*   Managing asynchronous operations, potentially with concurrency limits (`p-limit`).
*   Handling JSON/YAML parsing and writing (`js-yaml`).

Implementing these helpers repeatedly is inefficient and inconsistent. The **Script Utilities** (located in `scripts/lib/`, `evals/src/lib/`, or shared `src/utils/` if Node-compatible) provide reusable functions tailored for these Node.js scripting contexts.

**Central Use Case:** The Evals runner script (`evals/apps/cli/src/index.ts`, [Chapter 53](53_evals_system.md)) needs to find all `.yaml` suite files specified via `--suite`.

Using Utilities:
```typescript
// Conceptual script code using utils
import yargs from "yargs";
import { findYamlFiles } from "../lib/fileUtils"; // Utility using glob
import { scriptLogger as logger } from "../lib/logger"; // Logger utility using chalk

async function run() {
    const argv = yargs(process.argv.slice(2)).option("suitePath", { type: "string" }).parseSync();
    const suitePath = argv.suitePath || "evals/suites";

    try {
        // Utility handles file/dir check, recursion, filtering, errors
        const suiteFiles = await findYamlFiles(suitePath);
        if (suiteFiles.length === 0) { /* Log error, exit */ }
        logger.info(`Found ${suiteFiles.length} suite files.`);
        // ... use suiteFiles ...
    } catch (error: any) { /* Log error, exit */ }
}
```
The `findYamlFiles` utility simplifies the main script significantly.

## Key Concepts

1.  **Target Environment:** Node.js (run via `pnpm`, `npm`, `yarn`). Can use full Node.js API and `devDependencies`. **Cannot use `vscode` API.**
2.  **Common Functionality:**
    *   **Argument Parsing:** `yargs`, `commander`.
    *   **File System:** `fs/promises`, `glob` (e.g., `findFiles`, `readFileSafe`, `writeFileSafe`, `copyDirRecursive`, `ensureDir`). Reuses/adapts `src/utils/fs.ts` ([Chapter 42](42_file_system_utilities.md)).
    *   **Shell Execution:** `child_process`, `execa` (preferred for better error handling/promises).
    *   **Logging:** Console logger (`console`) enhanced with `chalk` for colors (`info`, `warn`, `error`, `success`).
    *   **Path Manipulation:** Node.js `path`. Reuses `src/utils/path.ts` helpers ([Chapter 43](43_path_utilities.md)) that are Node-compatible (e.g., `normalizePath`, `toPosix()`, `arePathsEqual`). **Cannot use workspace-dependent helpers.** Base paths from `process.cwd()` or args.
    *   **Concurrency:** `p-limit`.
    *   **JSON/YAML:** `JSON.parse/stringify`, `js-yaml`, potentially Zod validation ([Chapter 40](40_schemas__zod_.md)).
3.  **Location:** `scripts/lib/` (build/dev), `evals/src/lib/` (eval-specific), shared `src/utils/` (only if fully Node-compatible).
4.  **Reusability:** Imported across script files (`*.js`, `*.ts`). TS scripts run via `ts-node` or compiled JS.

## Using Script Utilities

Imported as standard functions/modules within script files.

*   **Argument Parsing (`evals/src/runEval.ts`):** Uses `yargs`. (Ch 53)
*   **Finding Files (`findYamlFiles` utility):** Uses `fs.stat`, `glob`. (Ch 55 Concepts)
*   **Executing Commands (`runCommand` utility):** Uses `execa`. (Ch 55 Concepts)
*   **Logging (`scriptLogger` utility):** Uses `chalk`. (Ch 55 Concepts)
*   **Reading/Validating JSON (`readAndValidateJson` utility):** Uses `safeReadFile`, `JSON.parse`, Zod `safeParse`. (Ch 55 Concepts)

## Code Walkthrough

Examining utilities relevant to scripts.

### Shared Utilities (`src/utils/`)

*   **`fs.ts` ([Chapter 42]):** `fileExistsAtPath`, `directoryExistsAtPath`, `safeWriteFile`, `createDirectoriesForFile`, `safeReadFile` are usable (require absolute paths).
*   **`path.ts` ([Chapter 43]):** `normalizePath`, `toPosix()`, `arePathsEqual`, `relativePath`/`toRelativePath` (require `cwd`), `getReadablePath` (require `cwd`) are usable. `getWorkspacePath` **is NOT usable**.
*   **`logging.ts`:** `logger` needs configuration for console output, or use dedicated `scriptLogger`.
*   **`text-normalization.ts` ([Chapter 45]):** Pure string functions are usable.

### Evals Utilities (`evals/src/lib/`)

*   **`openai.ts`:** `getCompletions` using `axios`. Network utility.
*   **`findSuites.ts` (Conceptual `findYamlFiles`):** Uses `fs.stat`, `path.resolve`, `glob`. File system utility.
*   **`contextUtils.ts` (Conceptual):** Uses `fs/promises`, `path` for temp eval files.

### Build Script Utilities (`scripts/lib/`)

*   **`copyWasm.js` (Conceptual):** `esbuild` plugin using Node.js `fs` ([Chapter 56]).
*   **`exec.ts` (Conceptual `runCommand`):** Uses `execa`. Process execution utility.
*   **`logger.ts` (Conceptual `scriptLogger`):** Uses `console`, `chalk`. Logging utility.
*   **`fsUtils.ts` (Conceptual `readAndValidateJson`):** FS + JSON + Zod utility.
*   **`find-missing-translations.js` / `find-missing-i18n-key.js`:** Specific scripts using `fs`, `path`, `glob`, `console`.

## Internal Implementation

*   **File System:** Node.js `fs/promises` API, `glob`.
*   **Command Execution:** Node.js `child_process`, `execa`.
*   **Argument Parsing:** `yargs`.
*   **Logging:** Node.js `console`, `chalk`.
*   **Path:** Node.js `path`.
*   **Concurrency:** `p-limit`.

## Modification Guidance

Involves adding new helpers or refining existing ones for script tasks.

1.  **Adding a `copyDirRecursive` Utility:** Implement using `fs.cp` (Node 16.7+) or manual recursion. Use `scriptLogger`.
2.  **Improving `runCommand` to Return Output:** Modify to use `execa` without `stdio: 'inherit'`, return `result` object (`stdout`/`stderr`).
3.  **Adding a Git Helper (`scripts/lib/gitUtils.ts`):** Implement `checkGitDirty` using `runCommandGetOutput("git", ["status", "--porcelain"])`.

**Best Practices:**

*   **Reusability:** Extract common script logic into libs. Share carefully with `src/utils`.
*   **Error Handling:** Fail fast. Utilities throw/reject critical errors. Log clearly (`scriptLogger`).
*   **Cross-Platform:** Use Node.js `path`. Use `execa`/`spawn` with arg arrays. Use `cross-env`.
*   **Logging:** Use consistent, colored logging.
*   **Async:** Use `async/await`. Use concurrency limiters (`p-limit`).
*   **Configuration:** Read via env vars, args, or files. Avoid hardcoding.
*   **Dependencies:** Use `devDependencies`.

**Potential Pitfalls:**

*   **VS Code API Usage:** Cannot use `vscode` in standalone scripts.
*   **Path Resolution:** CWD issues. Use `path.resolve`, `__dirname`.
*   **Shell Command Failures:** Need reliable error checking. `execa` helps.
*   **Unhandled Rejections.**
*   **Environment Differences (CI vs. Local).**

## Conclusion

Script Utilities provide essential helper functions that streamline the development, build, testing, and evaluation processes for the Roo-Code project. By encapsulating common tasks like argument parsing, robust file system operations, reliable command execution (using `execa`), and consistent logging (`scriptLogger` with `chalk`) into reusable modules tailored for the Node.js environment, they reduce code duplication, improve consistency, and make individual scripts cleaner and easier to maintain. While potentially sharing some low-level logic with runtime utilities, script utilities can directly leverage the full Node.js API and external CLI tools, distinct from the constraints of the VS Code extension runtime environment.

Next, we examine the specific system used to build the Roo-Code extension itself: [Chapter 56: Build System](56_build_system.md).
---
# Chapter 56: Build System

Continuing from [Chapter 55: Script Utilities](55_script_utilities.md), which covered helpers used in various project scripts, this chapter focuses on the specific processes and tools used to compile, bundle, and package the Roo-Code extension for distribution and development: the **Build System**.

## Motivation: Transforming Source Code into a Deployable Extension

The Roo-Code source code, written primarily in TypeScript (`src/`) with React for the WebView UI (`webview-ui/`) and including assets like images (`images/`), locale files (`public/locales`), NLS files (`package.nls.*.json`), and WASM modules (`node_modules/`), needs to be transformed into a format that VS Code can load and execute. This involves several steps:

1.  **TypeScript Compilation:** Converting TypeScript code (host `src/` and WebView `webview-ui/src/`) into JavaScript.
2.  **Bundling:** Combining multiple JavaScript/TypeScript files and their dependencies into fewer, optimized bundles.
3.  **Asset Handling:** Copying necessary assets (locales, images, WASM, NLS, WebView build output) to the final distribution directory (`dist/`).
4.  **Environment Configuration:** Injecting build-time environment variables (e.g., `NODE_ENV`, telemetry keys).
5.  **Development vs. Production Builds:** Creating different outputs (source maps vs. minification).
6.  **WebView UI Build:** Handling the separate Vite build process for the React UI and integrating its output.
7.  **Packaging Preparation:** Ensuring `dist/` has the correct structure for `vsce`.

Manually performing these steps is tedious and error-prone. A dedicated **Build System** automates this. Roo-Code utilizes `esbuild` for the extension host bundling and Vite for the WebView UI build, orchestrated via Node.js scripts (`esbuild.js`, `scripts/prepare.js`) and managed through `pnpm` scripts in `package.json`.

**Central Use Case:** Creating a production build for the VS Code Marketplace.

1.  Run `pnpm run build:prod`.
2.  Executes `pnpm clean`, `pnpm prepare`, then `cross-env NODE_ENV=production node ./esbuild.js`.
3.  **`scripts/prepare.js`:** Performs pre-build tasks (e.g., schema type generation).
4.  **`esbuild.js`:**
    *   Sets `NODE_ENV=production`. Cleans `dist/` (`rimraf`).
    *   **Host Build:** Calls `esbuild.build()` for `src/extension.ts` with production settings (minify, no sourcemap, `platform: 'node'`, `format: 'cjs'`, `external: ['vscode']`, `define`) outputting `dist/extension.js`.
    *   **WebView Build:** Executes `pnpm --filter webview-ui build` via `child_process.execSync`. Vite creates optimized assets in `webview-ui/build/`.
    *   **Asset Copying:** Uses script utilities ([Chapter 55](55_script_utilities.md), [Chapter 42](42_file_system_utilities.md)) to copy `webview-ui/build/*` -> `dist/webview-ui/`, WASM files -> `dist/`, `public/locales/*` -> `dist/locales/`, `package.nls*.json` -> `dist/`.
5.  **Output:** `dist/` contains `extension.js`, `webview-ui/`, `locales/`, WASM files, NLS files.
6.  **Packaging:** Run `pnpm package`. `vsce` bundles `dist/` (respecting `.vscodeignore`) into `.vsix`.

## Key Concepts

1.  **`esbuild`:** Fast bundler for the **extension host** (`src/` -> `dist/extension.js`). Configured in `esbuild.js`.
2.  **Vite:** Build tool for the **WebView UI** (`webview-ui/` -> `webview-ui/build/`). Configured in `webview-ui/vite.config.ts`. ([Chapter 1](01_webview_ui.md)).
3.  **Build Script (`esbuild.js`):** Node.js orchestrator using `esbuild` API, `child_process`, and `fs` utilities ([Chapter 55](55_script_utilities.md), [Chapter 42](42_file_system_utilities.md)). Handles modes, asset copying.
4.  **`package.json` Scripts:** Defines commands (`build:prod`, `watch:host`, `compile`, `vscode:prepublish`, `package`) using `pnpm`, `cross-env`, `node`, `tsc`, `rimraf`, `vsce`.
5.  **Entry Points & Outputs:** Host (`src/extension.ts` -> `dist/extension.js`), WebView (`webview-ui/src/index.tsx` -> `dist/webview-ui/assets/*`).
6.  **Host Configuration (`esbuild.js`):** `platform: 'node'`, `format: 'cjs'`, `target: 'node16'`, `external: ['vscode']`, `bundle: true`, `minify`, `sourcemap`, `define`.
7.  **Asset Handling:** Critical build step performed by `esbuild.js` or plugins. Copies WASM, locales, WebView build output, NLS files to `dist/`.
8.  **Development Mode (`watch:*`):** Separate watch processes: `esbuild --watch` for host, `vite dev` for WebView HMR.
9.  **Production Mode (`build:prod`):** Full sequence: clean, prepare, esbuild host, vite build webview, copy assets.
10. **Type Checking (`compile` script):** `tsc --noEmit` provides full TS check. Crucial for CI.
11. **Packaging (`vsce`, `package` script):** `vsce package` bundles `dist/` based on `.vscodeignore`. `.vscodeignore` excludes source, dev deps, build artifacts.
12. **Preparation Script (`scripts/prepare.js`):** Optional pre-build steps (e.g., code generation).

## Executing the Build

Via `pnpm` scripts defined in `package.json`.

```json
// --- File: package.json (Excerpt with Key Scripts) ---
{
  "scripts": {
    "install:all": "pnpm install && pnpm --filter webview-ui install",
    "prepare": "node ./scripts/prepare.js", // Pre-build steps
    "compile": "tsc -p ./ --noEmit", // Type checking
    "clean": "rimraf dist webview-ui/build", // Clean outputs
    "build:prod": "pnpm clean && pnpm prepare && cross-env NODE_ENV=production node ./esbuild.js", // Full production build
    "build:dev": "pnpm clean && pnpm prepare && cross-env NODE_ENV=development node ./esbuild.js", // Dev build
    "watch:host": "pnpm build:dev --watch", // Watch host code only
    "watch:webview": "pnpm --filter webview-ui dev", // Start Vite dev server
    "vscode:prepublish": "pnpm run compile && pnpm run build:prod", // Ensure compile + prod build before packaging
    "package": "vsce package --out ./releases/roo-code.vsix --yarn", // Create VSIX
    "publish": "vsce publish --yarn" // Publish
    // ... test, lint, etc.
  }
}
```

*   **Dev:** `pnpm install:all`, run `watch:host` & `watch:webview` concurrently, Debug (F5).
*   **Package:** `pnpm install:all`, `pnpm compile`, `pnpm build:prod`, `pnpm package`.

## Code Walkthrough

### Build Script (`esbuild.js`)

*(Based on refined version from Chapter 56)*

```javascript
// --- File: esbuild.js ---
const esbuild = require("esbuild");
const fs = require("node:fs");
const path = require("node:path");
const childProcess = require("node:child_process");
const os = require("node:os");
const chalk = require("chalk"); // Dev dependency
const rimraf = require("rimraf"); // Dev dependency

// Check modes
const isWatch = process.argv.includes("--watch");
const isProduction = process.env.NODE_ENV === "production";
const outDir = path.resolve(__dirname, "dist");

// --- Logging Helper ---
const log = { /* ... info, warn, error, success using chalk ... */ };

// --- Asset Copying Logic ---
function copyRecursiveSync(src, dest) { /* ... fs.cpSync or manual ... */ }
function ensureDirSync(dirPath) { fs.mkdirSync(dirPath, { recursive: true }); }

function copyAssets() {
    log.info("Copying assets...");
    ensureDirSync(outDir);

    // Copy WASM files -> dist/
    // ... (Logic as in Chapter 56) ...
    log.info("Copied WASM files.");

    // Copy Locales for WebView -> dist/locales
    const localeWebViewSrc = path.join(__dirname, "public", "locales");
    const localeWebViewDest = path.join(outDir, "locales");
    if (fs.existsSync(localeWebViewSrc)) { copyRecursiveSync(localeWebViewSrc, localeWebViewDest); log.info("Copied WebView locale files."); }

    // Copy NLS files for Host -> dist/
    const nlsFiles = fs.readdirSync(__dirname).filter(f => f.startsWith('package.nls') && f.endsWith('.json'));
    nlsFiles.forEach(f => fs.copyFileSync(path.join(__dirname, f), path.join(outDir, f)));
    log.info("Copied NLS files.");

    // Copy other assets (e.g., images -> dist/images)
    const imagesSrc = path.join(__dirname, "images");
    if (fs.existsSync(imagesSrc)) { /* ... copy ... */ log.info("Copied images."); }

    log.info("Asset copying finished.");
}

// --- esbuild Common Config ---
const commonOptions = {
	bundle: true,
	sourcemap: !isProduction ? 'inline' : false,
	minify: isProduction,
	external: ["vscode"], // Externalize vscode for host
	define: { /* ... NODE_ENV, POSTHOG_*, IS_TEST_ENVIRONMENT ... */ },
	logLevel: "info",
    absWorkingDir: __dirname,
};

// --- Extension Host Config ---
const extensionConfig = {
	...commonOptions,
	platform: "node",
	target: "node16",
	format: "cjs",
	entryPoints: ["src/extension.ts"],
	outfile: path.join(outDir, "extension.js"),
};

// --- Build Function ---
async function build() {
	try {
        log.info(`[Build] Starting ${isProduction ? 'production' : 'development'} build...`);
        // Clean output directory using rimraf
        log.info(`Cleaning output directory: ${outDir}`);
        await rimraf(outDir); // Asynchronous rimraf
        ensureDirSync(outDir);

		// 1. Build Extension Host with esbuild
        log.info("Building extension host...");
		await esbuild.build({
			...extensionConfig,
			watch: isWatch ? { onRebuild(error) { /* log rebuild status */ } } : undefined,
		});
        log.info("Extension host build complete.");

        // 2. Build WebView UI using Vite (only for production or non-host-watch builds)
        if (!isWatch || isProduction) {
            log.info("Building webview UI via Vite...");
            try {
                const webviewBuildCommand = isProduction ? 'build' : 'build --mode development';
                // Execute `pnpm build` within the `webview-ui` package
                childProcess.execSync(`pnpm ${webviewBuildCommand}`, {
                    stdio: 'inherit', cwd: path.join(__dirname, "webview-ui")
                });
                log.info("Webview UI build complete.");

                // 3. Copy WebView build output to dist
                const webviewBuildSrc = path.join(__dirname, "webview-ui", "build");
                const webviewBuildDest = path.join(outDir, "webview-ui");
                if (fs.existsSync(webviewBuildSrc)) { /* ... copy ... */ }
                else { log.warn(`Webview build output not found: ${webviewBuildSrc}`); }

            } catch (error) { /* ... handle error ... */ throw error; }
        } else { log.info("Skipping webview build in host watch mode."); }

        // 4. Copy other static assets
        copyAssets(); // Copy locales, WASM, NLS, images etc.

		log.success(`Build finished! Output: ${outDir}`);
        if (isWatch) log.info("Watching for extension host changes...");

	} catch (error) { /* ... handle error, process.exit(1) ... */ }
}

// --- Run Build ---
build();
```

**Explanation:**

*   Uses `chalk`, `rimraf`, Node `fs`, `path`, `child_process`, `os`.
*   `copyAssets` function copies WASM, WebView locales (`public/locales`), Host NLS files (`package.nls.*.json`), and images to `dist/`.
*   `build` function: Cleans `dist`, builds host (`esbuild`), conditionally builds WebView (`vite build` via `execSync`), conditionally copies Vite output, calls `copyAssets`. Handles `isWatch` flag for host build and skipping WebView build.

### Vite Configuration (`webview-ui/vite.config.ts`)

*(See code in Chapter 1)*
*   Configures Vite for React, Tailwind. Sets `build.outDir: "build"`.

### `.vscodeignore` (Conceptual)

*(See code in Chapter 56 - Key Concepts)*
*   Excludes source, `node_modules`, config files, tests, scripts, `webview-ui/build`.
*   **Includes `dist/`** and root metadata files (README, LICENSE, CHANGELOG).

## Internal Implementation

1.  **Trigger:** `pnpm build:prod` runs `esbuild.js` (`NODE_ENV=production`).
2.  **Cleanup:** Script deletes `dist/`.
3.  **Host Build:** `esbuild.build()` bundles `src/` -> `dist/extension.js` (minified).
4.  **WebView Build:** `execSync` runs `vite build`. Vite bundles `webview-ui/` -> `webview-ui/build/`.
5.  **Asset Copying:** Script copies `webview-ui/build/*` -> `dist/webview-ui/`, WASM -> `dist/`, locales -> `dist/locales/`, NLS -> `dist/`.
6.  **Packaging:** `vsce package` zips required files from `dist/` and root based on `.vscodeignore`.

## Modification Guidance

Involves changing targets, entry points, plugins, or asset handling.

1.  **Changing Target Node Version:** Modify `target` in `esbuild.js`. Check VS Code compatibility.
2.  **Adding New Asset Type (e.g., Fonts):** Modify `copyAssets` in `esbuild.js` to copy source fonts to `dist/`. Update code references. Ensure `.vscodeignore` doesn't exclude them from `dist/`.
3.  **Adding an esbuild Plugin:** Install, import, add to `plugins` array in `esbuild.js`.
4.  **Configuring Vite Build:** Modify `webview-ui/vite.config.ts` (`build` options).
5.  **Adding Pre-build Step:** Add command to `prepare` script in `package.json`.

**Best Practices:**

*   **Use Correct Tools:** `esbuild` (host), Vite (WebView).
*   **Clean Output (`dist/`):** Keep structured. Clean before builds (`rimraf`).
*   **Separate Scripts:** Use `package.json` for orchestration.
*   **Environment Variables:** Use `NODE_ENV`. Use `define` for build constants. Handle build secrets via CI/build env vars.
*   **`external: ['vscode']`:** Essential for host.
*   **Asset Management:** Explicitly copy *all* runtime assets to `dist/`. Verify paths.
*   **`.vscodeignore`:** Maintain carefully.
*   **Type Checking (`tsc --noEmit`):** Run separately (`compile` script).

**Potential Pitfalls:**

*   **Missing Assets in `dist/`:** Runtime errors. Check `copyAssets`.
*   **Incorrect `external`/`platform`.**
*   **Asset Path Issues:** Incorrect relative paths in code referencing assets in `dist/`.
*   **`.vscodeignore` Errors:** Ignoring needed files in `dist/` or including source.
*   **Build Environment Differences.**
*   **Vite/Esbuild Conflicts/Orchestration.**

## Conclusion

The Build System, orchestrated via `esbuild.js` and `package.json` scripts, reliably transforms Roo-Code's source code into a functional and distributable VS Code extension. It leverages the speed of `esbuild` for the extension host and the rich features of Vite for the WebView UI. Key aspects include distinct configurations, handling development vs. production builds, injecting environment variables, and crucially, ensuring all necessary runtime assets (WASM, locales, WebView bundles, NLS files) are correctly copied to the `dist/` directory, ready for packaging with `vsce`. This automated system is fundamental for both efficient development and creating optimized, distributable extension builds.

Next, we will look at how the project ensures code quality and functionality through automated tests: [Chapter 57: Testing Framework](57_testing_framework.md).
---
# Chapter 57: Testing Framework

Continuing from [Chapter 56: Build System](56_build_system.md), which focused on compiling and packaging Roo-Code, this chapter delves into the strategies and tools used to ensure the quality, correctness, and stability of the codebase: the **Testing Framework**.

## Motivation: Ensuring Quality and Preventing Regressions

As Roo-Code grows in complexity, with interactions between the extension host, WebView UI, LLM APIs, file system, terminal, and various utilities, the risk of introducing bugs or regressions increases significantly. Manually testing every feature and edge case after each change is impractical and unreliable.

A robust testing framework provides an automated way to:

1.  **Verify Correctness:** Ensure individual functions, components, and modules behave as expected.
2.  **Prevent Regressions:** Automatically detect when changes inadvertently break existing functionality.
3.  **Improve Design:** Writing testable code often encourages better design (modularity, pure functions, dependency injection).
4.  **Facilitate Refactoring:** Provide confidence that code can be refactored safely.
5.  **Document Behavior:** Tests act as executable documentation.

Roo-Code employs a combination of testing strategies, primarily using **Jest** as the test runner and framework, along with **React Testing Library** for testing WebView UI components, running in appropriate environments (Node.js for host, JSDOM for UI) with extensive mocking.

**Central Use Case:** A developer refactors the `normalizeLineEndings` utility ([Chapter 45: Text Normalization Utilities](45_text_normalization_utilities.md)) and needs to ensure it still works correctly.

1.  **Write/Update Test:** Ensures `src/utils/text-normalization.test.ts` has comprehensive cases covering CRLF, CR, LF, mixed, empty, null inputs.
    ```typescript
    // Example test case
    it('should replace CRLF with LF', () => {
      expect(normalizeLineEndings("Hello\r\nWorld")).toBe("Hello\nWorld");
    });
    ```
2.  **Run Tests:** Runs `pnpm test` or `pnpm test:host`.
3.  **Jest Execution:** Jest finds and runs the test file in the 'Host' project's Node.js environment. It executes the assertions.
4.  **Results:** Jest reports pass/fail status. Any regressions introduced by the refactoring cause test failures, alerting the developer immediately.

## Key Concepts

1.  **Jest:** The core testing framework. Provides test runner, assertions (`expect`), mocking (`jest.fn`, `jest.mock`), snapshot testing, and configuration (`jest.config.js`).
2.  **React Testing Library (`@testing-library/react`, `@testing-library/jest-dom`):** For testing WebView UI components (`webview-ui/`) focusing on user interaction and accessibility (`render`, `screen.getByRole`, `fireEvent`/`userEvent`).
3.  **Testing Environments (`jest.config.js` `projects`):** Crucial for Roo-Code.
    *   **Extension Host (`displayName: "Host"`, `testEnvironment: 'node'`):** For `src/` code. Mocks `vscode` API.
    *   **WebView UI (`displayName: "WebView"`, `testEnvironment: 'jsdom'`):** For `webview-ui/src/` code. Mocks `vscode` toolkit, `postMessage`.
4.  **Mocking Dependencies:** Essential for isolation.
    *   **`vscode` API:** Mocked via `src/__mocks__/vscode.ts`.
    *   **`@vscode/webview-ui-toolkit/react`:** Mocked via `src/__mocks__/@vscode/webview-ui-toolkit/react.ts` ([Chapter 32](32_vscode_webview_ui_toolkit_wrappers.md)).
    *   **`postMessage`:** `webview-ui/src/utils/vscode.ts` wrapper is mocked in UI tests.
    *   **Network Calls (`ApiHandler`, `axios`):** Mocked using `jest.mock()`.
    *   **File System (`fs`):** Mocked using `jest.mock('fs/promises')` or `memfs`.
5.  **Test Structure (`describe`, `it`/`test`, Arrange-Act-Assert):** Standard Jest structure. Setup/teardown hooks (`beforeEach`, etc.).
6.  **Code Coverage:** Generated using `jest --coverage`. Identifies untested code. Configured in `jest.config.js`.

## Executing Tests

Via `package.json` scripts:

```json
// --- File: package.json (Excerpt) ---
{
  "scripts": {
    "test": "jest", // All tests
    "test:watch": "jest --watch", // Watch mode
    "test:coverage": "jest --coverage", // With coverage
    "test:host": "jest --selectProjects Host", // Host only
    "test:ui": "jest --selectProjects WebView", // WebView only
    // ... other scripts ...
  }
}
```

## Code Walkthrough

### Jest Configuration (`jest.config.js`)

*(See code in Chapter 57 - Key Concepts)*
*   Defines "Host" (Node) and "WebView" (JSDOM) **projects**.
*   Configures `ts-jest`, `moduleNameMapper` (aliases, CSS mocks), `testEnvironment`.
*   WebView project sets `rootDir`, `roots` (to find top-level `__mocks__`), `testMatch`.
*   Includes coverage config (`collectCoverageFrom`).

### Mocking `vscode` (`src/__mocks__/vscode.ts`)

*(See code in Chapter 57 - Key Concepts)*
*   Provides `jest.fn()` mocks for APIs (`window.*`, `workspace.*`, `commands.*`, `env.*`, `authentication.*`).
*   Mocks properties, classes (`Uri`), enums. Allows testing host code and asserting API interactions.

### Unit Test Example (Host Utility - `src/utils/path.test.ts`)

*(See code in Chapter 57 - Key Concepts)*
*   Uses `jest.mock('vscode', ...)` to control `workspace.workspaceFolders`. Uses `describe`/`it`, `beforeEach`, `expect`. Switches `process.platform`.

### React Component Test (WebView UI - `webview-ui/src/components/settings/About.test.tsx`)

*(See code in Chapter 57 - Key Concepts)*
*   Uses `@testing-library/react`. Mocks dependencies (`@/utils/vscode`, `@/i18n/TranslationContext`, toolkit components via `__mocks__` or explicit `jest.mock`). Uses `screen`, `fireEvent`, `expect` with `jest-dom` matchers. Asserts on DOM and mock calls.

## Internal Implementation

1.  **Execution:** `pnpm test` runs `jest`.
2.  **Config/Projects:** Jest reads config, runs tests per project.
3.  **Environment:** Sets up `node` or `jsdom`.
4.  **Mocking:** Intercepts `import`, uses mocks (`__mocks__`, `jest.mock`). Maps aliases.
5.  **Test Run:** Executes `*.test.ts(x)`.
6.  **Rendering (UI):** RTL renders components into JSDOM using mocks.
7.  **Interaction/Assertion:** Tests simulate events, assert against JSDOM/mocks.
8.  **Coverage:** Instrumentation tracks execution.

## Modification Guidance

Involves adding/updating tests or mocks.

1.  **Adding Tests:** Create `*.test.ts(x)`. Import, `describe`/`it`, mock deps, `expect`. Choose correct project (`test:host` or `test:ui`).
2.  **Updating `vscode` Mock:** Add `jest.fn()` or values to `src/__mocks__/vscode.ts` for newly used APIs. Use `mockReturnValue` etc. within tests for specific behavior.
3.  **Mocking New Modules:** Use `jest.mock('../path/to/module', factory)` at top of test file. Import mock for assertions.

**Best Practices:**

*   **Unit Focus:** Prioritize unit tests with effective mocking.
*   **Test Behavior:** Use RTL for UI tests (user perspective).
*   **Mock Boundaries:** Mock external dependencies and logical internal boundaries.
*   **Coverage as Guide:** Find gaps, but prioritize testing critical logic/edges.
*   **Maintainability:** Write clear tests. Reset state (`clearMocks: true`). Keep mocks simple.
*   **CI Integration:** Run `pnpm test` in CI pipeline ([Chapter 58](58_documentation___community_files.md)).

**Potential Pitfalls:**

*   **Incomplete/Incorrect Mocks:** False positives/negatives. Requires maintenance.
*   **Over-Mocking:** Brittle tests.
*   **Testing Implementation:** Querying by CSS class, testing internal state.
*   **Testing Environment Mismatch:** Bugs only in real VS Code. Requires some manual/E2E testing.
*   **Flaky Tests:** Async/timing issues, state leakage. Use `waitFor`, reset state.
*   **Slow Tests:** Complex setup, inefficient mocks.

## Conclusion

The Testing Framework, primarily utilizing Jest and React Testing Library, is crucial for maintaining the quality and stability of the Roo-Code extension. By providing separate testing environments and configurations for the extension host (Node) and WebView UI (JSDOM), and leveraging comprehensive mocking for dependencies like the `vscode` API and UI toolkits, the framework enables automated verification of individual units and components. This automated testing catches regressions early, facilitates safer refactoring, and ultimately contributes to a more reliable and robust product for users. Running tests frequently, especially in CI pipelines, is a key practice for sustainable development.

Finally, we'll look at the documentation and community-related files that support the project: [Chapter 58: Documentation & Community Files](58_documentation___community_files.md).
---
# Chapter 58: Documentation & Community Files

Continuing from [Chapter 57: Testing Framework](57_testing_framework.md), which detailed how Roo-Code ensures code quality through testing, this final chapter looks at the essential supporting files that explain the project, guide contributors, and foster a healthy community: **Documentation & Community Files**.

## Motivation: Guiding Users and Contributors

A software project, especially an open-source one like Roo-Code, needs more than just functional code. To be successful, usable, and sustainable, it requires clear documentation and standard community health files to:

1.  **Explain Usage:** Guide end-users on installation, configuration, and features.
2.  **Onboard Developers:** Provide instructions for contributors on setup, architecture (this tutorial!), build ([Chapter 56](56_build_system.md)), test ([Chapter 57](57_testing_framework.md)), and contribution guidelines.
3.  **Document Architecture:** Explain design choices, components, and workflows.
4.  **Set Expectations:** Define codes of conduct, contribution processes, security policies.
5.  **Facilitate Collaboration:** Standardize bug reports, feature requests, PRs.
6.  **Meet Marketplace Requirements:** Provide `README`, `LICENSE`, `CHANGELOG`.
7.  **Define Project Standards:** Document coding rules (`.roo/rules/rules.md`) and localization guidelines (`.roo/rules-translate/`).

These files, typically in the project root, `docs/`, `.github/`, or `.roo/`, are crucial for usability, maintainability, and community engagement.

## Key Concepts & Files

1.  **`README.md`:** Primary repository entry point. Overview, features, install, quick start, links. Audience: Users & Contributors.
2.  **`LICENSE`:** Full open-source license text (e.g., MIT). Defines legal rights. Audience: Users, Contributors, Legal.
3.  **`CHANGELOG.md`:** Records notable changes per version (following Keep a Changelog). Audience: Users.
4.  **`CONTRIBUTING.md`:** Guidelines for contributors (setup, build, test, code style, PR process). Audience: Contributors.
5.  **`CODE_OF_CONDUCT.md`:** Community behavior standards (often Contributor Covenant). Audience: All Members.
6.  **`SECURITY.md`:** Process for responsibly reporting security vulnerabilities. Audience: Security Researchers, Users.
7.  **`.github/` Directory:** GitHub-specific files:
    *   **`ISSUE_TEMPLATE/`:** Markdown templates for bug reports, feature requests.
    *   **`PULL_REQUEST_TEMPLATE.md`:** Template for PR descriptions (checklist).
    *   **`workflows/`:** GitHub Actions YAML files for CI/CD (lint, test, build checks).
8.  **`docs/` Directory:** Detailed documentation: User Guides, Config Guides, **Architecture Docs (This Tutorial Series: `docs/architecture/*.md`)**.
9.  **`.roo/` Directory:** Project-specific rules:
    *   **`rules/rules.md`:** Code quality, architecture, testing standards ([Chapter 48](48_configuration_rules.md)). For Devs & AI Assistants.
    *   **`rules-translate/`:** Localization guidelines ([Chapter 50](50_localization_system__i18n_.md)). General (`001-general...`) and language-specific (`instructions-<lang>.md`) rules, glossaries. For Translators & AI Assistants.

## Using These Files

*   **Users:** Start with `README.md`. Refer to `docs/` for details. Check `CHANGELOG.md`. Use `ISSUE_TEMPLATE`s.
*   **Contributors:** Start with `README.md`, then `CONTRIBUTING.md`. Refer to `docs/architecture/`, `CODE_OF_CONDUCT.md`, `.roo/rules/rules.md`. Use GitHub Templates, check CI results.
*   **Translators:** Focus on `.roo/rules-translate/`. Refer to `CONTRIBUTING.md` for workflow.
*   **Maintainers:** Use all files for management, reviews, community standards. Use CI workflows.
*   **AI Assistants:** `.roo/rules/` files as context for development/translation tasks.
*   **VS Code Marketplace:** Uses `README.md`, `CHANGELOG.md`, `LICENSE`.

## Code Walkthrough

These files are primarily Markdown, YAML, or JSON.

### `.roo/rules/rules.md` (Example Structure)

*(See code in Chapter 48)*
*   Defines standards for testing (coverage goals), TypeScript usage (strict, avoid `any`), UI styling (themed Tailwind), state management conventions, error handling, dependencies. References relevant tutorial chapters.

### `.roo/rules-translate/` Files (Example Structure)

*(See code in Chapter 50)*
*   **`001-general-rules.md`:** General tone (informal), non-translatable terms, placeholder syntax, `Trans` component usage, QA script (`find-missing-translations.js`).
*   **`instructions-de.md`:** Mandates "du" form.
*   **`instructions-zh-cn.md`:** Detailed glossary, formatting rules.
*   **`instructions-zh-tw.md`:** Terminology differences, punctuation.

### GitHub Workflow Example (`.github/workflows/ci.yaml`)

*(See code in Chapter 57)*
*   Runs on push/PR to `main`. Sets up Node/pnpm. Installs deps. Runs `lint`, `compile` (type check), `test`, `build:prod`. Uses dummy env vars for build secrets if needed.

## Internal Implementation

Static documentation (Markdown) or configuration files (YAML, JSON) used by humans or external tools (GitHub, VS Code, Build/Test runners).

## Modification Guidance

Involves updating content for accuracy and clarity.

1.  **Updating `README.md` / `docs/`:** Add features, update instructions, fix links.
2.  **Updating `CHANGELOG.md`:** Add entries for new releases.
3.  **Modifying `CONTRIBUTING.md`:** Update setup/build/test instructions ([Chapter 56](56_build_system.md), [Chapter 57](57_testing_framework.md)). Refine workflow.
4.  **Updating Code Quality Rules (`.roo/rules/rules.md`):** Add/modify rules. Align with linters/CI.
5.  **Updating Translation Guidelines (`.roo/rules-translate/`):** Add terms, clarify rules, update workflow/checklist.

**Best Practices:**

*   **Keep Updated:** Essential. Assign ownership or periodic review.
*   **Clarity & Conciseness:** Write for the target audience. Use formatting.
*   **Standard Formats:** Follow conventions (Keep a Changelog, Contributor Covenant). Use standard GitHub locations.
*   **Automation:** Use CI (`workflows/`) to enforce checks. Use GitHub templates.
*   **Discoverability:** Link related documents.
*   **Accuracy:** Ensure technical instructions are correct.

**Potential Pitfalls:**

*   **Outdated Documentation.**
*   **Inaccurate Instructions.**
*   **Missing Files** (LICENSE, CHANGELOG).
*   **Broken Links.**
*   **Unclear Contribution Process.**
*   **Inconsistent Rules.**
*   **Guidelines Ignored/Unenforced.**

## Conclusion

Documentation and Community Files, including project-specific rules like `rules.md` and `rules-translate/`, are vital supporting elements for the Roo-Code project. A clear `README.md` introduces the extension, `CONTRIBUTING.md` guides developers, `CHANGELOG.md` tracks progress, `LICENSE` defines legal terms, and other files like `CODE_OF_CONDUCT.md` and `SECURITY.md` foster a healthy community. GitHub-specific files streamline issue reporting, contributions, and CI/CD automation. Detailed architecture docs (like this tutorial series) and specific rule files (`.roo/`) ensure consistency and quality. Maintaining these resources accurately is crucial for the project's success, usability, and collaborative potential.

This chapter concludes the Roo-Code tutorial series, having covered the architecture from the high-level UI and provider interactions down to core services, utilities, build processes, testing, and supporting documentation.
---
# Chapter 59: Cache Strategy (Bedrock)

This chapter number duplicates the content intended for [Chapter 28: Bedrock Cache Strategy](28_bedrock_cache_strategy.md). There appears to be a mix-up in the chapter list provided.

To avoid redundancy, please refer to **[Chapter 28: Bedrock Cache Strategy](28_bedrock_cache_strategy.md)** for the detailed explanation of how Roo-Code implements strategies to leverage AWS Bedrock's prompt caching feature.

**Chapter 28 covers:**
*   The motivation for using Bedrock Prompt Caching (reducing latency and cost).
*   Key concepts like cache points (`<cache_point/>`), relevant `ModelInfo` properties (`supportsPromptCache`, `maxCachePoints`, etc.), the `BaseStrategy` interface, and the `MultiPointStrategy` implementation.
*   How the strategy determines cache point placement, including handling previous placements (N-1 rule).
*   How `AwsBedrockHandler` integrates the strategy.
*   Code walkthroughs and sequence diagrams illustrating the process.
*   Guidance on modifying or extending the caching strategies.

All information regarding Bedrock caching strategies can be found in **[Chapter 28: Bedrock Cache Strategy](28_bedrock_cache_strategy.md)**.
```

