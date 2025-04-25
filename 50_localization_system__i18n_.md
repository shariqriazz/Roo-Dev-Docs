# Chapter 50: Localization System (i18n)

Continuing from [Chapter 49: OAuth Helpers](49_oauth_helpers.md), which discussed authentication flows for external services, this chapter focuses on making Roo-Code accessible to a global audience by implementing internationalization (i18n) and localization (l10n): the **Localization System (i18n)**. This includes the tooling and guidelines necessary for translating the UI and managing translations effectively, referencing files like `.roo/rules-translate/`.

## Motivation: Supporting Multiple Languages

To reach a wider user base and provide a more inclusive experience, it's essential for applications like Roo-Code to display text in the user's preferred language. Hardcoding strings (button labels, descriptions, menu items, error messages, tooltips) directly into the codebase makes translation difficult, error-prone, and hard to maintain. Different languages also have different conventions for formatting, plurals, and tone.

A dedicated Localization System allows developers and translators to:

1.  **Externalize Strings:** Separate user-visible text from the source code into resource files (e.g., JSON files).
2.  **Translate Resources:** Create language-specific versions of these resource files (e.g., `en.json`, `de.json`, `ja.json`, `zh-cn.json`).
3.  **Load Dynamically:** Detect the user's preferred language (based on VS Code's locale settings) and load the appropriate translations at runtime for both the WebView UI and parts of the extension host.
4.  **Provide Fallbacks:** Define a default language (usually English) to use if a translation is missing for the user's locale.
5.  **Format Translations:** Handle plurals, interpolation (inserting dynamic values into strings), and potentially date/number formatting according to locale conventions.
6.  **Provide Guidelines:** Establish clear rules for translators (`.roo/rules-translate/`) covering tone (e.g., informal "du" in German), terminology (glossaries like in `instructions-zh-cn.md`), formatting, and workflow to ensure consistency and quality across languages.
7.  **Validate Translations:** Use helper scripts (`scripts/find-missing-translations.js`) to automatically detect missing translation keys across different languages and namespaces.

**Central Use Case:** Displaying the "Save" button in the Settings UI ([Chapter 35: Settings UI Components (WebView)](35_settings_ui_components__webview_.md)), ensuring it uses the informal tone in German as specified in the translation rules.

Without i18n:
```jsx
// Hardcoded string
<Button>Save</Button>
```

With i18n (WebView):
```typescript
// --- File: webview-ui/src/components/settings/SettingsView.tsx ---
import { useAppTranslation } from "@src/i18n/TranslationContext";
// ...
const { t } = useAppTranslation(); // Get translation function
// ...
<Button onClick={handleSubmit}>{t("settings:saveButton")}</Button> // Use key
```

1.  **Initialization:** The `i18next` instance is initialized (`webview-ui/src/i18n/setup.ts`) and provided via `TranslationProvider`. The user's VS Code language (e.g., "de") is detected and passed from the extension host via initial state or messages, stored in `ExtensionStateContext`. `i18next` loads the German translation files (`webview-ui/src/i18n/locales/de/*.json`).
2.  **Component Render:** The `SettingsView` component renders.
3.  **Hook Usage:** It calls `useAppTranslation()` to get the `t` function.
4.  **Translation Lookup:** It calls `t("settings:saveButton")`. `i18next` looks up the key `"saveButton"` within the `"settings"` namespace in the currently loaded language (`de`).
5.  **Resource File (`webview-ui/src/i18n/locales/de/settings.json`):** Based on guidelines in `.roo/rules-translate/instructions-de.md` (mandating "du" form).
    ```json
    {
      "saveButton": "Speichern" // Informal imperative, adheres to rule
    }
    ```
6.  **Result:** The `t` function returns `"Speichern"`.
7.  **Render:** The button renders with the German text "Speichern".

## Key Concepts

1.  **`i18next` & `react-i18next` (WebView):** The core libraries for managing translations and React integration in the WebView UI.
2.  **WebView Translation Files (`webview-ui/src/i18n/locales/<lang>/<namespace>.json`):** JSON resource files organized by language code (e.g., `en`, `de`, `ja`, `zh-CN`) and namespace (`common`, `settings`, `mcp`, etc.). These are imported directly during the build process (using `import.meta.glob`) in `webview-ui/src/i18n/setup.ts`, not loaded via HTTP.
3.  **Namespaces:** Logical grouping of strings within files (e.g., `settings.json`). `common` is the default.
4.  **WebView i18n Initialization (`webview-ui/src/i18n/setup.ts`):** Configures the `i18next` instance *for the WebView*.
    *   **Resource Loading:** Uses `import.meta.glob` to eagerly import all `.json` files from the `locales` directory during the build. The `loadTranslations` function adds these imported resources to the `i18next` instance.
    *   **Language Detection/Setting:** Primarily relies on the `language` setting passed from the host via `ExtensionStateContext` ([Chapter 12: ExtensionStateContext](12_extensionstatecontext.md)).
    *   **Fallback Language (`fallbackLng: 'en'`):** English is the default.
    *   **Namespaces:** Defines namespaces based on loaded files.
5.  **React Context Provider (`TranslationProvider` in `webview-ui/src/i18n/TranslationContext.tsx`):**
    *   Wraps the `App` component.
    *   Uses `I18nextProvider` to make the `i18next` instance available.
    *   `useEffect` hook syncs `i18nInstance.language` with the `language` from global state.
6.  **`useAppTranslation` Hook (`webview-ui/src/i18n/TranslationContext.tsx`):** Custom hook wrapping `react-i18next`'s `useTranslation`. Provides the `t` function.
7.  **`t` Function & `Trans` Component (WebView):** Used for string lookup (`t('ns:key')`), interpolation (`{{var}}`), pluralization, and rendering complex JSX translations.
8.  **Extension Host Localization (`src/i18n/`, `package.nls*.json`):** Uses a *separate* `i18next` instance configured in `src/i18n/setup.ts`.
    *   **Host Translation Files (`src/i18n/locales/<lang>/<namespace>.json`):** JSON files containing keys specifically for host-side messages (errors, info messages, logs).
    *   **Host Initialization (`src/i18n/setup.ts`):** Initializes a distinct `i18next` instance for the Node.js environment. Loads translations directly from the file system using Node.js `fs` within the `dist` directory.
    *   **Host `t` Function (`src/i18n/index.ts`):** Exports functions (`initializeI18n`, `t`) that wrap the host-side `i18next` instance. `initializeI18n` is called during extension activation with the VS Code language.
    *   **VS Code NLS:** For static contributions in `package.json` (commands, settings descriptions), standard VS Code localization (`%key%` syntax linked to `package.nls*.json` files) is used. These NLS files must be copied to `dist/` by the build system ([Chapter 56: Build System](56_build_system.md)).
9.  **Translation Guidelines (`.roo/rules-translate/`):** Crucial Markdown files defining rules for translators:
    *   **General (`001-general-rules.md`):** Overall tone, style, non-translatable terms, placeholder syntax, workflow, QA using validation script.
    *   **Language-Specific (`instructions-<lang>.md`):** Specific grammar (e.g., "du" form), terminology glossaries, formatting rules, checklists.
10. **Validation Scripts:**
    *   **`scripts/find-missing-translations.js`:** Checks if all keys present in the English files exist (and are non-empty) in all other language files for both `core` (host) and `webview` areas. Reports missing keys.
    *   **`scripts/find-missing-i18n-key.js`:** Scans the source code (`src/`, `webview-ui/src/`) for usages of `t()` or `i18nKey=` and checks if the used keys actually exist in the corresponding English locale files. Reports keys used in code but missing from translations.

## Using the Localization System

### WebView UI

1.  **Setup:** `<TranslationProvider>` wraps the app. Language synced from host state.
2.  **Usage:** `const { t } = useAppTranslation();`. Use `t('ns:key')` or `<Trans i18nKey="...">`.
3.  **Adding Strings:** Add key/value to `webview-ui/src/i18n/locales/en/<ns>.json`. Add translation to *all other* `webview-ui/src/i18n/locales/<lang>/<ns>.json`. Follow guidelines. Run validation scripts.

### Extension Host

1.  **Static (`package.json`):** Use `%key%`. Define in `package.nls.json` (en) and `package.nls.<lang>.json`. Ensure build copies NLS files to `dist/`.
2.  **Dynamic (Host Code):**
    *   Import `initializeI18n` and `t` from `src/i18n/index.ts`.
    *   Call `initializeI18n(vscode.env.language)` during extension activation.
    *   Use `t('ns:key')` for messages (e.g., `vscode.window.showInformationMessage(t("core:exportComplete"))`).
    *   Add key/value to `src/i18n/locales/en/<ns>.json` and translations to other `src/i18n/locales/<lang>/<ns>.json` files. Follow guidelines. Run validation scripts.

### Translators

1.  **Read Guidelines:** Read **all** files in `.roo/rules-translate/`.
2.  **Edit Files:** Edit JSON files in `webview-ui/src/i18n/locales/<lang>/` and `src/i18n/locales/<lang>/`. Also edit `package.nls.<lang>.json`.
3.  **Preserve Syntax:** Keep `{{placeholders}}` and `<1>...</1>` markers exact.
4.  **Terminology/Tone:** Use glossaries, follow tone guidelines (e.g., informal "du").
5.  **Validate:** Run `node scripts/find-missing-translations.js` and `node scripts/find-missing-i18n-key.js`. Fix reported issues.

## Code Walkthrough

### WebView: `i18n/setup.ts`

*(See code in Chapter 50 - Key Concepts - Updated)*
*   Uses `import.meta.glob` to import all `locales/**/*.json` files eagerly during build time.
*   `loadTranslations` function adds these imported resources to the `i18next` instance.
*   Initializes `i18next` minimally; language set later via `TranslationProvider`.

### WebView: `i18n/TranslationContext.tsx`

*(See code in Chapter 50 - Key Concepts)*
*   `TranslationProvider`: Gets `language` from `useExtensionState`. Uses `useEffect` to call `i18nInstance.changeLanguage(language)` when context language changes. Wraps children with `I18nextProvider`. Loads translations on mount via `loadTranslations()`.
*   `useAppTranslation`: Custom hook wrapping `useTranslation`.

### Host: `src/i18n/setup.ts`

*(See code provided in chapter context)*
*   Checks `isTestEnv` to avoid `fs` operations during tests.
*   Uses Node.js `fs` and `path` to find language directories and JSON files within `src/i18n/locales`.
*   Reads and parses each file, building the `translations` object `{ lang: { namespace: { key: value } } }`.
*   Initializes a separate `i18next` instance with these `resources`. Sets `lng: 'en'` initially.

### Host: `src/i18n/index.ts`

*(See code provided in chapter context)*
*   Exports `initializeI18n(language)` which calls `i18next.changeLanguage(language)` on the host instance.
*   Exports `getCurrentLanguage()` which returns `i18next.language`.
*   Exports `t(key, options)` which calls `i18next.t(key, options)` on the host instance.

### Translation Guidelines (`.roo/rules-translate/`)

*(See descriptions/excerpts in Key Concepts section and provided files)*
*   Markdown files outlining general and language-specific rules, glossaries, workflow, and QA steps.

### Validation Scripts (`scripts/find-missing-translations.js`, `scripts/find-missing-i18n-key.js`)

*(See code provided in chapter context)*
*   **`find-missing-translations.js`:** Compares non-English JSON files against English JSON files (for both `core` and `webview`) and reports missing or empty keys. Takes `--locale`, `--file`, `--area` arguments. Exits with code 1 if issues found.
*   **`find-missing-i18n-key.js`:** Scans `.ts`/`.tsx` files in `src/` and `webview-ui/src/` for `t("...")` or `i18nKey="..."` usages. Checks if the extracted key exists in the corresponding English locale file. Reports keys used in code but not found in locale files.

## Internal Implementation

*   **WebView (`i18next`):** Build process bundles locale JSON data via `import.meta.glob`. `TranslationProvider` syncs language with host state. `t()` performs lookup in in-memory resources.
*   **Extension Host (`i18next`):** `initializeI18n` sets language based on `vscode.env.language`. `setup.ts` reads JSON files from `dist/i18n/locales` using `fs` on activation. `t()` performs lookup in host memory.
*   **Extension Host (NLS):** VS Code loads `package.json`, finds matching `package.nls.<lang>.json` in `dist/`, substitutes `%key%`.

## Modification Guidance

Primarily involves adding strings, languages, namespaces, or updating guidelines/validation.

1.  **Adding a New String:**
    *   Determine if it's for WebView or Host.
    *   Add key/value to the appropriate `en/` namespace JSON file.
    *   Add key/translation to *all other* language JSON files in the same location.
    *   Use `t('ns:key')` in code (WebView or Host).
    *   Run validation scripts to check.

2.  **Adding a New Language:**
    *   **WebView:** Add code to `supportedLanguages` (optional, mainly for detector). Create `webview-ui/src/i18n/locales/<langCode>/`, copy `en` files, translate.
    *   **Host:** Create `src/i18n/locales/<langCode>/`, copy `en` files, translate. Create `package.nls.<langCode>.json`, translate keys from `package.nls.json`.
    *   **Guidelines:** Add `instructions-<langCode>.md` if needed.
    *   **Build:** Ensure build copies new NLS and locale files.
    *   **Test:** Set VS Code language and verify.

3.  **Adding a New Namespace:**
    *   **WebView (`config.ts` or `setup.ts`):** Add namespace name to `ns` array (if using `i18next-http-backend`, less relevant for `import.meta.glob`).
    *   **Files:** Create `<namespace>.json` in *all* `webview-ui/src/i18n/locales/<lang>/` folders. Add keys/translations.
    *   **Host:** Create `<namespace>.json` in *all* `src/i18n/locales/<lang>/` folders. Add keys/translations.
    *   **Usage:** Use `t('newNamespace:key')`.

4.  **Updating Validation Scripts:** Modify `.js` files in `scripts/` if file locations change, new key formats are introduced, or different checks are needed.

**Best Practices:**

*   **Keys:** Use semantic keys.
*   **Namespacing:** Organize logically.
*   **Fallback:** Maintain complete English translations.
*   **Sync Files:** Keep keys consistent. Use validation scripts (`find-missing-*`).
*   **`Trans` Component:** Use for JSX.
*   **Guidelines:** Maintain clear, specific `.roo/rules-translate/` docs.
*   **Separate Concerns:** Use correct system for WebView vs. Host.

**Potential Pitfalls:**

*   **Missing Keys/Translations:** Caught by validation scripts.
*   **JSON Errors.**
*   **Missing Assets:** Build script ([Chapter 56: Build System](56_build_system.md)) must copy `webview-ui/src/i18n/locales` (if using direct import like shown), `src/i18n/locales`, and `package.nls*.json` files to `dist/`.
*   **Language Sync Issues.**
*   **Namespace Errors.**
*   **Host vs. WebView Mix-up.**

## Conclusion

The Localization System enables Roo-Code to provide a user interface understandable to a global audience. By leveraging `i18next` and `react-i18next` in the WebView (loading resources directly via `import.meta.glob`), a separate `i18next` instance in the extension host (loading resources via `fs`), and VS Code's native NLS mechanism for static contributions, Roo-Code supports multiple languages effectively. Crucially, detailed Translation Guidelines stored in `.roo/rules-translate/`, along with validation scripts like `find-missing-translations.js`, provide essential instructions and quality control for translators, ensuring consistency in tone, terminology, and formatting across all supported languages.

Next, we revisit communication mechanisms, specifically focusing on the low-level Inter-Process Communication used for features like local MCP servers: [Chapter 51: IPC (Inter-Process Communication)](51_ipc__inter_process_communication_.md).
---
# Chapter 51: IPC (Inter-Process Communication)

Continuing from [Chapter 50: Localization System (i18n)](50_localization_system__i18n_.md), which covered adapting the UI for different languages, this chapter explores how different *processes* within or related to the Roo-Code ecosystem communicate with each other: **Inter-Process Communication (IPC)**.

## Motivation: Connecting Disparate Processes

While much of Roo-Code runs within the single VS Code extension host process and communicates with the WebView UI via a specific message protocol ([Chapter 3: Webview/Extension Message Protocol](03_webview_extension_message_protocol.md)), certain features involve or could involve separate, independent processes:

1.  **MCP Servers (Stdio):** Local MCP servers ([Chapter 19: McpHub / McpServerManager](19_mcphub___mcpservermanager.md)) are often started as child processes (e.g., Node.js scripts, Python scripts, compiled binaries). The main extension needs to send requests (like `tools/call`) to these processes and receive responses reliably over their standard input/output streams.
2.  **Background Services:** Potentially long-running analysis, indexing, or model management tasks might be offloaded to separate background processes (e.g., using Node.js `worker_threads` or `child_process.fork`) to avoid blocking the main extension host thread. Communication is needed to send tasks and receive results.
3.  **External Helper Tools:** Specific functionalities might be implemented as small, standalone command-line tools invoked by the extension (like `ripgrep` in [Chapter 16: Ripgrep Integration](16_ripgrep_integration.md)). Communication involves passing arguments and capturing `stdout`/`stderr`.

These separate processes cannot directly call functions or access memory in the main extension host (or vice-versa). They require a defined mechanism to exchange data and commands – this is Inter-Process Communication. Different scenarios call for different IPC mechanisms.

**Central Use Case:** Roo-Code (via `McpHub`) needs to call the `analyze_complexity` tool provided by a local MCP server (`project-analyzer`) running as a Node.js child process, communicating over standard streams (stdio).

1.  **Process Spawn:** When `McpHub` first connects ([Chapter 19: McpHub / McpServerManager](19_mcphub___mcpservermanager.md)), the `StdioClientTransport` (from `@modelcontextprotocol/sdk`) uses Node.js's `child_process.spawn` to start the server (e.g., `node /path/to/analyzer_server.js`). It keeps references to the child process's `stdin` and `stdout` streams.
2.  **Request:** `McpHub.callTool(...)` is called. The MCP `Client` formats the `tools/call` request as a JSON-RPC message string (e.g., `{"jsonrpc":"2.0","method":"tools/call","params":{...},"id":123}`).
3.  **Serialization & Framing:** The JSON-RPC message is serialized into a UTF-8 string. A newline character (`\n`) is appended as a message delimiter.
4.  **Send (Write to Stdin):** The `StdioClientTransport` writes the framed message (`json_string\n`) to the child process's `stdin` stream.
5.  **Server Receives & Processes:** The `project-analyzer` process has logic (likely using `readline` on `process.stdin`) to read data line by line. It receives the line, parses the JSON-RPC message, identifies the `tools/call` method, executes the `analyze_complexity` logic, and prepares a JSON-RPC response string (e.g., `{"jsonrpc":"2.0","result":{...},"id":123}`).
6.  **Server Sends (Write to Stdout):** The server serializes its response, appends `\n`, and writes it to its `stdout` stream.
7.  **Receive (Read from Stdout):** The `StdioClientTransport` listens to the child process's `stdout`. It buffers data and uses `readline` (or similar) to split it by newlines. When it receives the complete response line, it parses the JSON-RPC message.
8.  **Response Handling:** The transport matches the `id` (123) to the pending request and resolves the promise associated with the `client.request` call, providing the `result` payload back to `McpHub`.

This structured, message-based exchange over standard streams allows the two separate Node.js processes to communicate effectively.

## Key Concepts

1.  **Process Isolation:** Operating systems isolate processes, giving each its own memory space. IPC mechanisms bridge this isolation.
2.  **Communication Channels:** Methods for processes to exchange data. Roo-Code primarily uses:
    *   **Standard Streams (Stdio):** `stdin`, `stdout`, `stderr`. Text-based (often JSON) or binary data piped between parent and child. Requires message framing. Used for local MCP servers, potentially other helpers.
    *   **Network Protocols (HTTP, SSE, WebSockets):** Used for communicating with processes that expose network interfaces (even on `localhost`). SSE is used for remote MCP servers. Not OS-level IPC, but achieves inter-process communication.
    *   **Node.js `child_process`:** Spawns child processes (`spawn`, `exec`, `fork`). `spawn` provides direct stream access (used for Stdio IPC and tools like Ripgrep). `fork` is specific to Node.js child processes and establishes a dedicated IPC channel automatically (suitable for Node.js background workers).
    *   **VS Code Specific:**
        *   **WebView <-> Host:** Uses `postMessage` API brokered by VS Code ([Chapter 3: Webview/Extension Message Protocol](03_webview_extension_message_protocol.md)).
        *   **`vscode.authentication`:** Handles communication for auth flows internally ([Chapter 49: OAuth Helpers](49_oauth_helpers.md)).
3.  **Serialization:** Converting in-memory data (objects, etc.) into a transmissible format (e.g., JSON strings, Buffers). `JSON.stringify()` and `JSON.parse()` are standard for text-based IPC like stdio or HTTP. Schemas for specific IPC messages might be defined (e.g., in `src/schemas/ipc.ts`) using Zod ([Chapter 40: Schemas (Zod)](40_schemas__zod_.md)).
4.  **Message Framing/Delimiting:** Defining message boundaries in stream-based communication.
    *   **Newline Delimiter (`\n`):** Simple method where each complete message ends with a newline. Used by MCP SDK's stdio transport. Requires messages not to contain literal newlines internally (or requires escaping).
    *   **Length Prefixing:** Prepending the byte length of the message before the message itself. More robust for binary data or messages with internal newlines. (Not the primary method shown in Roo-Code's MCP usage).
5.  **Protocols (e.g., JSON-RPC):** Defines the structure of messages (request vs. response), method names, parameters, request IDs, and error formats. JSON-RPC is commonly used over stdio or sockets for RPC-style communication. MCP uses a JSON-RPC-like structure.
6.  **Request/Response Matching:** Using unique IDs (like JSON-RPC `id`) to correlate asynchronous responses with their original requests. The client stores pending requests (e.g., in a `Map`) keyed by ID and uses the ID in the response to find and resolve the correct promise.

## IPC Mechanisms Used in Roo-Code

1.  **Extension Host <-> WebView UI:** VS Code `postMessage` API (JSON serializable data). (See [Chapter 3: Webview/Extension Message Protocol](03_webview_extension_message_protocol.md)).
2.  **Extension Host <-> Local MCP Server (Stdio):** `child_process.spawn` + `stdin`/`stdout` streams + Newline-delimited JSON-RPC messages. Handled largely by `@modelcontextprotocol/sdk`'s `StdioClientTransport`. (See [Chapter 19: McpHub / McpServerManager](19_mcphub___mcpservermanager.md)).
3.  **Extension Host <-> Remote MCP Server (SSE):** HTTP + Server-Sent Events + JSON-RPC messages. Handled largely by `@modelcontextprotocol/sdk`'s `SSEClientTransport`. (See [Chapter 19: McpHub / McpServerManager](19_mcphub___mcpservermanager.md)).
4.  **Extension Host -> Ripgrep Process:** `child_process.spawn` + Command-line arguments (input) + `stdout` capture (output). (See [Chapter 16: Ripgrep Integration](16_ripgrep_integration.md)).
5.  **Extension Host <-> Other Child Processes (Potential):** Could use `child_process.fork` (for Node.js children, provides built-in IPC channel via `process.send`/`child.on('message')`) or `child_process.spawn` with stdio (for any executable, requiring manual message framing and parsing). Schemas for `fork` messages might be defined in `src/schemas/ipc.ts`.

## Code Walkthrough

We focus on the mechanisms explicitly visible or inferred.

### Stdio for Local MCP Servers (`StdioClientTransport` usage in `McpHub`)

*(See full code walkthrough in Chapter 19)*

*   **Spawning:** `new StdioClientTransport({ command, args, options })` configures the child process details.
*   **Starting:** `transport.start()` calls `child_process.spawn`. Crucially pipes `stderr` from the child to the extension host's logger.
*   **Sending:** `client.request(...)` -> `transport.send(jsonRpcString)` -> writes `jsonRpcString + '\n'` to child's `stdin`.
*   **Receiving:** Transport listens to child's `stdout`, uses `readline` (or similar) to buffer and split lines by `\n`, parses JSON-RPC, finds pending request by `id`, resolves promise.
*   **Error/Exit:** Transport listens to `stderr` and process `close`/`error` events to manage connection state and report errors.

### Child Process for Ripgrep (`execRipgrep` in `src/services/ripgrep/index.ts`)

*(See full code walkthrough in Chapter 16)*

*   **Spawning:** `childProcess.spawn(rgPath, args, { stdio: ['ignore', 'pipe', 'pipe'] })`. Input is via `args`.
*   **Output Capture:** Reads `stdout` line by line using `readline`. Captures `stderr` directly.
*   **Framing:** Relies on `rg --json` producing one JSON object per line (newline delimited), or captures raw lines.
*   **Completion:** Promise resolves with captured `stdout` or rejects based on `stderr` content or exit code.

### WebView <-> Host Messaging

*(See full code walkthrough in Chapter 3 and examples in UI chapters)*

*   **WebView -> Host:** `vscode.postMessage(jsonSerializableObject)` (using `acquireVsCodeApi` wrapper).
*   **Host -> WebView:** `panel.webview.postMessage(jsonSerializableObject)`.
*   **Receiving (WebView):** `window.addEventListener('message', (event) => { const data = event.data; ... })`.
*   **Receiving (Host):** `panel.webview.onDidReceiveMessage((message) => { ... })`.
*   **Framing/Serialization:** Handled internally by VS Code. Assumes JSON-serializable data.

### Node.js `fork` IPC (`src/schemas/ipc.ts` & Conceptual Usage)

```typescript
// --- File: src/schemas/ipc.ts --- (Conceptual Schema Definitions)
import { z } from "zod";

// Schema for messages FROM host TO worker
export const workerTaskSchema = z.object({
    id: z.number(), // Unique task ID
    type: z.literal("calculate"), // Example task type
    inputData: z.any(), // Define more strictly based on task
});
export type WorkerTask = z.infer<typeof workerTaskSchema>;

// Schema for messages FROM worker TO host
export const workerResponseSchema = z.union([
    z.object({
        id: z.number(),
        type: z.literal("result"),
        outputData: z.any(), // Define more strictly
    }),
    z.object({
        id: z.number(),
        type: z.literal("error"),
        errorMessage: z.string(),
    }),
    z.object({
        id: z.number().optional(), // Progress update might not have ID
        type: z.literal("progress"),
        message: z.string(),
    }),
]);
export type WorkerResponse = z.infer<typeof workerResponseSchema>;

// --- File: src/backgroundTasks/myWorker.ts --- (Conceptual Worker using Schemas)
import { workerTaskSchema, WorkerResponse } from '../schemas/ipc'; // Import schemas

process.on('message', (message) => {
    // Validate incoming message
    const taskResult = workerTaskSchema.safeParse(message);
    if (!taskResult.success) {
        console.error("[Worker] Received invalid task message:", taskResult.error);
        // Cannot easily send error back without task ID
        return;
    }
    const task = taskResult.data;
    console.log(`[Worker] Received task ${task.id}:`, task.type);

    try {
        // Example: Send progress update (validate outgoing message)
        const progressMsg: WorkerResponse = { type: 'progress', message: 'Starting calculation...', id: task.id };
        workerResponseSchema.parse(progressMsg); // Validate before sending
        process.send?.(progressMsg);

        const result = performHeavyCalculation(task.inputData); // Perform work

        // Send result back (validate outgoing message)
        const resultMsg: WorkerResponse = { type: 'result', id: task.id, outputData: result };
        workerResponseSchema.parse(resultMsg);
        process.send?.(resultMsg);

    } catch (error: any) {
        // Send error back (validate outgoing message)
        const errorMsg: WorkerResponse = { type: 'error', id: task.id, errorMessage: error.message };
        workerResponseSchema.parse(errorMsg);
        process.send?.(errorMsg);
    }
});
// ... (performHeavyCalculation) ...
console.log('[Worker] Ready for tasks.');

// --- File: src/services/backgroundService.ts --- (Conceptual Host using Schemas)
import * as childProcess from "child_process";
import * as path from "path";
import { logger } from "../utils/logging";
import { workerResponseSchema, WorkerResponse, WorkerTask } from '../schemas/ipc'; // Import schemas

const workerPath = path.join(__dirname, "../backgroundTasks/myWorker.js"); // Path to compiled worker script
let worker: childProcess.ChildProcess | null = null;
const pendingTasks = new Map<number, { resolve: (result: any) => void; reject: (reason?: any) => void }>();
let taskIdCounter = 0;

function ensureWorker(): childProcess.ChildProcess {
    if (!worker || worker.killed || !worker.connected) {
        logger.info("Forking new background worker...", { ctx: "IPC.Worker" });
        worker = childProcess.fork(workerPath);
        setupWorkerListeners(worker); // Attach listeners to the new worker
    }
    return worker;
}

function setupWorkerListeners(workerInstance: childProcess.ChildProcess) {
     workerInstance.on('message', (message) => {
        // Validate incoming message from worker
        const responseResult = workerResponseSchema.safeParse(message);
        if (!responseResult.success) {
            logger.error("Received invalid message from worker:", responseResult.error, { ctx: "IPC.Worker" });
            return;
        }
        const response = responseResult.data;
        logger.debug("Received validated message from worker:", response, { ctx: "IPC.Worker" });

        // Handle progress messages separately maybe
        if (response.type === 'progress') { /* Log progress or update UI */ return; }

        // Find pending task by ID (required for result/error)
        if (response.id === undefined) return;
        const task = pendingTasks.get(response.id);
        if (task) {
            pendingTasks.delete(response.id); // Remove completed task
            if (response.type === 'result') {
                task.resolve(response.outputData); // Resolve promise with data
            } else if (response.type === 'error') {
                task.reject(new Error(response.errorMessage || "Worker error")); // Reject promise with error
            }
        }
    });
    workerInstance.on('error', (err) => { /* ... reject all pending ... */ });
    workerInstance.on('exit', (code) => { /* ... reject all pending, mark worker null ... */ });
}

// Public function to run a task
export async function runTaskInBackground(input: any): Promise<any> {
    return new Promise((resolve, reject) => {
        const workerInstance = ensureWorker();
        const taskId = ++taskIdCounter;
        pendingTasks.set(taskId, { resolve, reject }); // Store promise resolvers

        // Construct and validate task message before sending
        const taskMsg: WorkerTask = { id: taskId, type: "calculate", inputData: input };
        try {
            workerTaskSchema.parse(taskMsg); // Validate outgoing message
            workerInstance.send(taskMsg); // Send validated task
            logger.debug(`Sent task ${taskId} to worker.`, { ctx: "IPC.Worker" });
        } catch (validationError: any) {
             logger.error("Failed to validate task message before sending:", validationError, { ctx: "IPC.Worker" });
             pendingTasks.delete(taskId);
             reject(new Error("Internal error: Invalid task structure."));
        }

        // Optional: Add timeout for task completion
        // setTimeout(() => { ... reject if task still pending ... }, 30000);
    });
}

export function shutdownWorker() { /* ... worker?.kill(); ... */ }
```

**Explanation:**

*   **Schemas (`ipc.ts`):** Defines Zod schemas (`workerTaskSchema`, `workerResponseSchema`) for messages passed between the host and a potential background worker using `child_process.fork`. This provides validation and type safety for the IPC messages themselves.
*   **Worker:** Uses `process.on('message')` to receive tasks. **Crucially, it uses `workerTaskSchema.safeParse` to validate the incoming message** before processing. It constructs responses conforming to `workerResponseSchema` (using `satisfies` for type checking and potentially `parse` for runtime validation before sending) and uses `process.send` to transmit them.
*   **Host:** The `message` handler in `setupWorkerListeners` **uses `workerResponseSchema.safeParse` to validate messages received from the worker** before processing them and resolving/rejecting the corresponding task promise stored in `pendingTasks`. The `runTaskInBackground` function also validates the outgoing task message using `workerTaskSchema.parse` before calling `workerInstance.send`.

## Internal Implementation

*   **Stdio:** Node.js `child_process.spawn` creates OS pipes. `stdin.write`, `stdout.on('data')`. `readline` for framing.
*   **SSE:** `SSEClientTransport` uses `ReconnectingEventSource` (HTTP GET for stream). Requests might use `axios` POST.
*   **`child_process.spawn` (General):** Creates OS process, Node streams interface with OS pipes.
*   **`child_process.fork`:** Specialized `spawn` for Node.js scripts. Automatically sets up an additional IPC channel (often pipes). `process.send`/`.on('message')` use this channel with built-in V8 serialization (handles more types than JSON, but only Node-to-Node).
*   **VS Code `postMessage`:** Internal VS Code mechanism for host <-> WebView communication. Handles serialization.

## Modification Guidance

Modifications typically involve adding communication with new types of processes or changing the protocol/framing for stdio/socket communication.

1.  **Adding Communication with a Python Script via Stdio (JSON-RPC):**
    *   **Python Script (`my_script.py`):** Implement JSON-RPC handling (read `sys.stdin` line, `json.loads`, process, `json.dumps` response, write to `sys.stdout`, `flush=True`).
    *   **Host Code:** Use `child_process.spawn('python', ['...'])`. Implement client-side JSON-RPC over stdio (manage request IDs, write `json.dumps(req)\n`, use `readline` on stdout, parse `json.loads(line)`, match `id`, resolve promise). Define Zod schemas for requests/responses in `src/schemas/ipc.ts`. Validate messages on both ends.

2.  **Changing Stdio Framing to Length Prefixing:**
    *   **Transport Logic:** Modify both client and server transports. Implement length calculation, writing length prefix, writing message buffer, reading length, reading exact bytes, parsing message. Handle partial reads.
    *   **Impact:** Requires changes on both ends. More complex but robust.

3.  **Using `child_process.fork` for a Complex Node.js Task:**
    *   Implement worker script (`myWorker.ts`) using `process.on('message')` / `process.send`.
    *   Implement host service (`backgroundService.ts`) using `child_process.fork`, `worker.send`, `worker.on('message')`, manage pending requests.
    *   Define message structures using Zod schemas in `src/schemas/ipc.ts`. **Validate messages** on both ends using `safeParse` or `parse`.

**Best Practices:**

*   **Choose Appropriate Mechanism:** Match IPC choice to process type and communication needs.
*   **Standardize Protocol:** Use JSON-RPC or define clear message schemas (Zod).
*   **Validate IPC Messages:** Use Zod schemas (`src/schemas/ipc.ts`) to validate messages passed between processes (`fork`, custom stdio) to catch structural or type errors early.
*   **Robust Framing:** Essential for stream-based IPC (stdio, sockets). Newlines are simple but fragile; length prefixing is better. `fork` handles it automatically.
*   **Serialization:** Use JSON unless binary performance is critical. Validate with Zod.
*   **Error Handling:** Handle connection, process exit, `stderr`, parsing, timeouts. Define error message formats.
*   **Resource Management:** Terminate processes, close streams/sockets/handles, remove listeners.
*   **Asynchronicity:** Use `async/await`.

**Potential Pitfalls:**

*   **Framing/Parsing Errors:** Corrupted messages in stream-based IPC.
*   **Process Management:** Zombie processes, spawn failures, unhandled exits.
*   **Deadlocks:** Waiting indefinitely for responses (need timeouts).
*   **Buffering Issues:** Data delays or fragmentation in streams. Stdio buffers can fill.
*   **Cross-Platform:** Path, environment, shell differences affecting child processes.
*   **Security:** Unvalidated data/commands passed between processes.
*   **Missing IPC Validation:** Forgetting to use Zod schemas to validate messages passed via `fork` or custom stdio can lead to runtime errors if message structures diverge.

## Conclusion

Inter-Process Communication is essential for enabling different parts of the Roo-Code ecosystem, particularly the extension host and external processes like local MCP servers or CLI tools, to collaborate effectively despite running in isolated memory spaces. Roo-Code utilizes appropriate mechanisms like the structured newline-delimited JSON-RPC over stdio provided by the MCP SDK, direct `child_process.spawn` with stdout/stderr capture for tools like Ripgrep, and the specialized `postMessage` API for WebView interaction. Defining message structures with Zod schemas (e.g., in `src/schemas/ipc.ts`) and validating messages at process boundaries adds crucial type safety and robustness to these communication channels. Understanding the chosen IPC mechanism, the importance of serialization and message framing/validation, and the need for robust error handling is key to debugging and extending Roo-Code's interactions with separate processes.

Next, we examine how Roo-Code collects and reports usage data and errors for monitoring and improvement: [Chapter 52: TelemetryService](52_telemetryservice.md).
---
# Chapter 52: TelemetryService

Continuing from [Chapter 51: IPC (Inter-Process Communication)](51_ipc__inter_process_communication_.md), which discussed how different processes communicate, this chapter focuses on how the Roo-Code extension gathers anonymous usage data and error information to help improve the product: the **TelemetryService**.

## Motivation: Understanding Usage and Improving Roo-Code

To make Roo-Code better, the development team needs insights into how it's being used and where problems occur. Gathering anonymous telemetry data helps answer questions like:

*   Which features, commands, or tools ([Chapter 8: Tools](08_tools.md)) are most popular?
*   Which LLM providers ([Chapter 5: ApiHandler](05_apihandler.md)) and models are commonly used?
*   How often do specific errors (API errors, validation failures ([Chapter 40: Schemas (Zod)](40_schemas__zod_.md)), tool errors) occur?
*   What is the typical duration or token usage ([Chapter 29: Cost Calculation Utilities](29_cost_calculation_utilities.md)) of tasks?
*   Are users encountering issues with specific configurations (e.g., shell integration issues detected via `no_shell_integration` event from [Chapter 15: Terminal Integration](15_terminal_integration.md))?
*   Are custom modes ([Chapter 10: CustomModesManager](10_custommodesmanager.md)) being utilized?

Collecting this data, while **strictly respecting user privacy and anonymity**, allows the team to prioritize development efforts, identify bugs, improve performance, and make data-driven decisions about the product's future direction. Developing without this feedback makes it hard to address real user needs and pain points effectively.

The `TelemetryService` provides a centralized mechanism for:
1.  **User Consent:** Explicitly requiring user opt-in for telemetry collection. No data is sent unless the user enables the setting. A clear notification or setup step should prompt the user initially.
2.  **Centralized Reporting API:** Offering simple methods (`captureEvent`, `captureError`, `captureException`, `captureSchemaValidationError`) for different parts of the codebase to easily record significant events or errors.
3.  **Anonymization:** Ensuring that collected data **never** includes personally identifiable information (PII) like file contents, specific user prompts/responses, API keys, full file paths, commit messages, etc. Focuses on aggregated counts, feature usage patterns, anonymized configuration types, and sanitized error information.
4.  **Context Enrichment:** Automatically adding relevant, non-sensitive context to events, such as the Roo-Code version, VS Code version, platform (OS), session ID, and potentially anonymized/generalized configuration details (e.g., provider *name* but not keys, mode *slug* but not custom instructions).
5.  **Backend Integration:** Sending the collected, anonymized data to a secure backend service (like PostHog) for aggregation and analysis. Handles initialization and shutdown of the backend client.

**Central Use Case:** A user runs a "Fix Code" action ([Chapter 30: CodeActionProvider](30_codeactionprovider.md)), and the underlying API call to the LLM results in a rate limit error. The user has opted **in** to telemetry.

1.  **User Opt-in:** During initial setup or via settings ([Chapter 35: Settings UI Components (WebView)](35_settings_ui_components__webview_.md)), the user enables the telemetry setting. The `TelemetryService` reads this setting (`telemetrySetting`) via `ContextProxy` ([Chapter 11: ContextProxy](11_contextproxy.md)) and sets its internal `enabled` flag to `true`.
2.  **Action Invoked:** The command handler for `roo-cline.fixCode` calls `telemetryService.captureEvent("codeaction.invoked", { command: "roo-cline.fixCode", mode: currentModeSlug })`.
3.  **API Error:** The `ApiHandler` ([Chapter 5: ApiHandler](05_apihandler.md)) catches the rate limit error from the SDK.
4.  **Error Reporting:** Error handling logic calls `telemetryService.captureError("api_error", { provider: "openai", modelId: "gpt-4", errorType: "rate_limit" })`.
5.  **`TelemetryService` Logic (`captureEvent`, `captureError`):**
    *   Checks `this.enabled` (true).
    *   Retrieves common properties (version, platform, session ID, anonymous `distinctId`). Gets provider context (e.g., `current_mode`).
    *   Merges common, provider, and event-specific properties.
    *   Calls `sanitizeProperties` to remove potential PII (e.g., ensures `modelId` isn't a sensitive custom name, removes hypothetical path properties).
    *   Calls the backend client's method (e.g., `this.posthogClient?.capture({ distinctId, event: "roo_codeaction.invoked", properties: sanitizedProps })` and `this.posthogClient?.capture({ distinctId, event: "roo_error_api_error", properties: sanitizedProps })`).
6.  **Backend:** The PostHog service receives and aggregates these anonymized events, allowing developers to see code action usage frequency and API error rates per provider/model type.

If the user had opted **out**, step 5 would check `this.enabled` (false) and immediately return without sending any data.

## Key Concepts

1.  **User Consent (`telemetrySetting`):** A dedicated setting (`"enabled"`, `"disabled"`, `"unset"`) stored via `ContextProxy` ([Chapter 11: ContextProxy](11_contextproxy.md)). The service respects this setting rigorously. Initial state (`"unset"`) triggers a prompt asking for consent via `vscode.window.showInformationMessage`. ([Chapter 35: Settings UI Components (WebView)](35_settings_ui_components__webview_.md) provides the persistent setting toggle).
2.  **Anonymity & PII Avoidance:** The absolute highest priority. **No sensitive or user-identifiable data should ever be sent.** This includes: user code, prompts/responses, file contents, API keys, full file paths, specific user-defined names (custom modes, API profiles - use slugs or types instead), commit messages, detailed error messages containing user data. Focus on event counts, feature flags, command IDs, provider *names*, generic model *types*, error *types*, durations, boolean flags, sanitized configuration values.
3.  **`TelemetryService` Class (`src/services/telemetry/TelemetryService.ts`):**
    *   **Singleton:** Instantiated once during extension activation.
    *   **Initialization (`init`):** Takes `ExtensionContext`, `ContextProxy`. Reads consent, initializes backend client (PostHog) if enabled and API key is available (via build-time environment variable `process.env.POSTHOG_PROJECT_API_KEY` set by [Chapter 56: Build System](56_build_system.md)). Generates session ID, gathers static context (versions, platform).
    *   **Consent Management (`updateConsent`):** Re-reads the setting and enables/disables the service, shutting down or initializing the PostHog client as needed. Triggered on init and by `onDidChangeConfiguration`.
    *   **Core Methods:** `captureEvent`, `captureError`, `captureException`, `captureSchemaValidationError`. Adds standard prefixes (`roo_`, `roo_error_`) to event names.
    *   **Context Enrichment:** The internal `_capture` method merges event `properties` with `commonProperties` (OS, versions, session ID) and dynamic `providerContext` (e.g., current mode slug from `ClineProvider`) before sanitization.
    *   **Sanitization (`sanitizeProperties`):** A crucial helper function (`src/services/telemetry/sanitize.ts`) that iterates through event properties and removes or masks potential PII based on key names, value length, or heuristics.
    *   **Backend Client (`posthog-node`):** Uses the `PostHog` client library. Configured with API key and host (from environment variables). Handles batching, retries, and asynchronous sending. `enableGeoip: false` is set for privacy.
    *   **Shutdown (`shutdown`):** Flushes buffered events using `posthogClient?.shutdown()`. Called during extension deactivation.
    *   **Provider Context (`setProvider`):** Allows linking to the active `ClineProvider` ([Chapter 2: ClineProvider](02_clineprovider.md)) to fetch non-sensitive context like the current mode slug. Uses `WeakRef` to avoid memory leaks.

4.  **Session/User IDs:**
    *   **Session ID:** `crypto.randomUUID()` generated per VS Code session. Groups events within a single run.
    *   **Distinct ID (User):** Uses `vscode.env.machineId` (a unique, anonymous ID per VS Code installation). Allows correlating events from the same installation over time without identifying the user. Handles the default test value `someValue.machineId`.

5.  **Event Structure:** Uses PostHog's format: `event` name (string, prefixed e.g., `roo_eventname`), `distinctId` (anonymous user ID), `properties` (key-value object with event data + context). Standard PostHog properties like `$os` are included in `commonProperties`.

6.  **Sanitization Logic (`sanitizeProperties`):** Needs careful implementation to remove known sensitive keys (`/key|secret|token|password|credential/i`), long strings (potential prompts/code/paths), and path-like strings, while allowing safe context values.

## Using the TelemetryService

The service is obtained as a singleton instance (`export const telemetryService = new TelemetryService();`) and its methods are called from various points in the codebase.

**1. Initialization & Consent (Extension Activation):**

```typescript
// --- File: src/extension.ts ---
import { TelemetryService } from "./services/telemetry/TelemetryService";
import { ContextProxy } from "./core/config/ContextProxy";
import * as vscode from "vscode";
import { ClineProvider } from "./core/webview/ClineProvider";

export const telemetryService = new TelemetryService(); // Create singleton

// Helper to prompt user for consent if needed
async function ensureTelemetryConsent(contextProxy: ContextProxy): Promise<void> {
    const telemetrySettingKey = "telemetrySetting"; // Use constant if available
    const currentSetting = contextProxy.getValue(telemetrySettingKey as any) ?? 'unset';
    if (currentSetting === 'unset') {
        const selection = await vscode.window.showInformationMessage(
            "Help improve Roo Code by sending anonymous usage data and error reports? No source code or personal data is ever sent. You can change this in Settings.",
            { modal: true },
            "Enable Telemetry", // User explicitly enables
            "Disable Telemetry" // User explicitly disables
        );
        // If user closed dialog without selecting, treat as disabled for this session, prompt again next time?
        // Or default to disabled. Let's default to disabled.
        const settingValue = selection === "Enable Telemetry" ? "enabled" : "disabled";
        await contextProxy.setValue(telemetrySettingKey as any, settingValue);
        // Capture the consent action itself
        telemetryService.captureEvent("consent.set", { consent_given: settingValue === "enabled", source: "initial_prompt" });
        // Update the service immediately after setting is saved
        await telemetryService.updateConsent(contextProxy);
    }
}

export async function activate(context: vscode.ExtensionContext) {
    // ... Create ContextProxy and initialize it ...
    const contextProxy = new ContextProxy(context);
    await contextProxy.initialize();

    // Initialize Telemetry AFTER Proxy is ready
    await telemetryService.init(context, contextProxy);

    // Create Provider and link it AFTER telemetry is initialized
    // (Provider constructor might capture init events)
    const provider = new ClineProvider(context, outputChannel);
    // Allow telemetry to get context like current mode from provider
    telemetryService.setProvider(provider);

    // Prompt for consent if needed (after telemetry service is initialized)
    // Run non-blockingly after activation completes
    ensureTelemetryConsent(contextProxy).catch(e => console.error("Consent prompt failed:", e));

    // Listen for setting changes to update consent dynamically
    context.subscriptions.push(vscode.workspace.onDidChangeConfiguration(async e => {
        // Use the actual setting key from package.json contribution
        const fullSettingKey = 'roo-code.telemetrySetting'; // Assuming this is the ID in package.json
        if (e.affectsConfiguration(fullSettingKey)) {
            const oldEnabled = telemetryService['enabled']; // Access internal state for comparison
            await telemetryService.updateConsent(contextProxy);
            // Capture event only if state actually changed
            if (oldEnabled !== telemetryService['enabled']) {
                 telemetryService.captureEvent("consent.updated", { consent_given: telemetryService['enabled'], source: "settings_change" });
            }
        }
    }));
    // ... Register commands, providers etc. ...
}

export async function deactivate(): Promise<void> {
    await telemetryService.shutdown(); // Flush events on exit
}
```
*Explanation:* Creates singleton. Calls `init` after `ContextProxy`. Links provider via `setProvider`. Calls `ensureTelemetryConsent` (which prompts if setting is `unset`). Listens for config changes to call `updateConsent` and log consent changes. Calls `shutdown` in `deactivate`.

**2. Capturing Events:**

```typescript
// --- File: src/activate/registerCodeActions.ts ---
import { telemetryService } from "../services/telemetry/TelemetryService";
// ... inside command handler ...
    vscode.commands.registerCommand(command, async (...args: any[]) => {
        // Get current mode if possible
        const currentMode = ClineProvider.getSidebarProviderInstance()?.contextProxy?.getValue('mode');
        telemetryService.captureEvent("codeaction.invoked", { command, mode: currentMode });
        // ...
    });

// --- File: src/core/tools/readFileTool.ts ---
    // ... after successful read ...
    telemetryService.captureEvent("tool.executed", { tool_name: "read_file", success: true });
    // ... after access denied by ignore ...
    telemetryService.captureEvent("tool.blocked", { tool_name: "read_file", reason: "rooignore" });
```
*Explanation:* Call `telemetryService.captureEvent` with name and non-sensitive properties.

**3. Capturing Errors:**

```typescript
// --- File: src/api/providers/anthropic.ts (Conceptual Error Handling) ---
    } catch (error: any) {
        const errorType = /* classify error (e.g., 'rate_limit') */;
        const statusCode = error?.status;
        telemetryService.captureError("api_error", {
            provider: "anthropic",
            modelId: this.getModel()?.id, // Should be sanitized by service
            errorType: errorType,
            statusCode: statusCode,
        });
        throw error;
    }

// --- File: src/core/config/ProviderSettingsManager.ts ---
    // ... inside load(), after safeParse ...
    if (!validationResult.success) {
        telemetryService.captureSchemaValidationError({
            schemaName: "ProviderProfiles", // Identify the schema
            error: validationResult.error // Pass the ZodError
        });
        // ... handle error ...
    }
```
*Explanation:* Use `captureError` for handled errors with categorization. Use `captureSchemaValidationError` for Zod errors. Use `captureException` in global handlers for uncaught exceptions.

## Code Walkthrough

### TelemetryService Class (`src/services/telemetry/TelemetryService.ts`)

*(See full code in Key Concepts section)*

**Key Implementation Details:**

*   **Constructor:** Sets up PostHog key/host from `process.env`. Generates `sessionId`. Sets `distinctId` to `vscode.env.machineId` (handling dev default).
*   **`init`:** Populates `commonProperties` (OS, versions). Calls `updateConsent`. Logs status.
*   **`updateConsent`:** Reads setting via `ContextProxy`. Shuts down/initializes `PostHog` client based on `enabled` state and key presence. Handles client init errors.
*   **`setProvider`/`getProviderContext`:** Uses `WeakRef` to link to `ClineProvider` and fetch non-sensitive context like `current_mode`.
*   **`_capture`:** Central internal method. Checks `enabled`. Merges common, provider, event properties. **Calls `sanitizeProperties`**. Calls `posthogClient.capture`. Includes error handling for capture process.
*   **Public Methods:** Add `roo_` prefixes. Prepare specific properties (limited stack for exceptions, paths/counts for schema errors). Call `_capture`.
*   **`shutdown`:** Calls `posthogClient.shutdown()`.

### Sanitization (`src/services/telemetry/sanitize.ts` - Conceptual)

*(See improved conceptual code in Key Concepts section)*

*   Iterates through properties object.
*   Applies rules based on key names (skip sensitive keywords like `key`, `secret`, `token`).
*   Applies rules based on value characteristics (truncate long strings, replace path-like strings with placeholders).
*   Allows safe types (number, boolean, null). Handles simple objects/arrays by size limit, replaces complex/unserializable ones.
*   **Must be carefully implemented and continuously reviewed.**

## Internal Implementation

1.  **Event Trigger:** Code calls `telemetryService.capture...()`.
2.  **Consent Check:** Service checks `this.enabled`. If false, returns.
3.  **Context/Sanitize:** `_capture` merges context, calls `sanitizeProperties`.
4.  **Client Call:** `_capture` calls `posthogClient.capture()`.
5.  **PostHog Client Buffering:** `posthog-node` library adds event to internal buffer.
6.  **Async Send:** Periodically, or on buffer limits, client sends batches via HTTPS to PostHog API.
7.  **Shutdown:** `telemetryService.shutdown()` -> `posthogClient.shutdown()` triggers final buffer flush.

**Sequence Diagram (Capture Event):**

*(See diagram in Chapter 52 - Internal Implementation)*

## Modification Guidance

Modifications involve adding events, changing context, or updating sanitization.

1.  **Adding a New Tracked Event:**
    *   **Identify:** Find code location.
    *   **Call:** Add `telemetryService.captureEvent("my_event", { property: safeValue })`.
    *   **Data:** Include only relevant, **non-sensitive** properties. Check against `sanitizeProperties`.
    *   **Sanitize:** Update `sanitizeProperties` if new property types need specific handling.

2.  **Adding More Context:**
    *   **Common Properties (`init`):** Add static, safe data.
    *   **Provider Context (`getProviderContext`):** Add logic to get more non-sensitive state from `ClineProvider`. Ensure safety.
    *   **Event Properties:** Pass more relevant, safe data directly via `captureEvent`.

3.  **Refining Sanitization (`sanitizeProperties`):**
    *   **Edit `sanitize.ts`:** Add more sensitive key patterns. Improve path/URL detection and anonymization (hashing?). Adjust length thresholds. Add recursive sanitization carefully.
    *   **CRITICAL:** Test rigorously to prevent PII leaks.

4.  **Changing Telemetry Backend:**
    *   Replace `posthog-node` library and client calls in `TelemetryService` with the new backend's SDK. Ensure consent and sanitization remain central.

**Best Practices:**

*   **Privacy First & Opt-In:** Non-negotiable. Rigorous sanitization, clear consent prompt (`ensureTelemetryConsent`). Default to disabled/unset.
*   **Centralized Service:** Use the singleton.
*   **Sanitize Rigorously:** Continuously review `sanitizeProperties`. Err on removing data.
*   **Meaningful Events:** Track actionable insights. Avoid noise.
*   **Safe Context:** Only add non-sensitive context.
*   **Graceful Failure:** Service failures shouldn't crash the extension.
*   **Shutdown:** Implement `shutdown` for data flushing.
*   **Transparency:** Document data collection practices clearly (README/Privacy Policy).

**Potential Pitfalls:**

*   **PII Leakage:** Insufficient sanitization is the primary risk.
*   **Consent Bypass:** Bugs sending data when disabled.
*   **Performance:** Excessive event capture (unlikely with batching).
*   **Network/Backend Issues:** Data loss if backend unavailable.
*   **Sanitization Complexity/Errors:** Bugs in `sanitizeProperties`. Over-sanitization losing useful anonymous data.
*   **Missing API Key:** Build process ([Chapter 56: Build System](56_build_system.md)) must correctly inject `POSTHOG_PROJECT_API_KEY` via `define` for telemetry to initialize.

## Conclusion

The `TelemetryService` provides a crucial, privacy-conscious mechanism for collecting anonymous usage data and error information from Roo-Code installations where users have explicitly opted in. By centralizing event capture, rigorously sanitizing data to remove PII via `sanitizeProperties`, enriching events with safe context, and integrating with a backend like PostHog, the service provides valuable insights for the development team to improve the extension based on real-world usage patterns and identify recurring problems. Respecting user consent (via the `telemetrySetting` and initial prompt) and prioritizing privacy through careful implementation (especially `sanitizeProperties`) and configuration are paramount.

Next, we explore the system designed for evaluating Roo-Code's performance on predefined tasks: [Chapter 53: Evals System](53_evals_system.md).
---
# Chapter 53: Evals System

Continuing from [Chapter 52: TelemetryService](52_telemetryservice.md), which focused on collecting anonymous usage data for product improvement, this chapter introduces a system designed for more structured and objective assessment of Roo-Code's capabilities: the **Evals System**.

## Motivation: Measuring and Improving AI Performance

While user feedback and telemetry provide valuable insights, objectively measuring the quality and performance of an AI assistant like Roo-Code requires a more systematic approach. We need a way to:

1.  **Define Test Cases:** Create a standardized set of realistic tasks (prompts, context, expected outcomes) that represent common user scenarios (e.g., "refactor this function", "explain this error", "write a unit test", "use tool X").
2.  **Run Automatically:** Execute Roo-Code against these test cases using different models, configurations, or code versions in a repeatable manner, independent of the interactive VS Code UI.
3.  **Capture Outputs:** Record the AI's full response, including generated text, code modifications requested via tools, actual tool usage sequences, token counts, and costs for each test case.
4.  **Evaluate Results:** Compare the captured outputs against predefined criteria (or using subsequent LLM-based evaluation) to assess performance metrics like correctness, helpfulness, efficiency (token usage, cost, time), safety (avoiding harmful actions), and adherence to instructions.
5.  **Track Progress:** Monitor how changes to prompts ([Chapter 7: SystemPrompt](07_systemprompt.md)), models, context strategies ([Chapter 23: Sliding Window Context Management](23_sliding_window_context_management.md)), or tool implementations ([Chapter 8: Tools](08_tools.md)) affect performance over time, preventing regressions and guiding improvements.

The **Evals System**, primarily located within the `evals/` directory, provides the framework and tooling to define, run, and analyze these evaluation tasks, enabling data-driven development and quality assurance for Roo-Code's core AI functionality.

**Central Use Case:** A developer modifies the main system prompt ([Chapter 7: SystemPrompt](07_systemprompt.md)) and wants to ensure it doesn't negatively impact code generation quality for Python.

1.  **Define Eval Suite:** An `evals/suites/python_codegen.yaml` file defines several test cases, each validated by `evalTaskSchema` ([Chapter 40: Schemas (Zod)](40_schemas__zod_.md)):
    *   `id`: `py_fib_recursive`
    *   `prompt`: "Write a Python function to calculate Fibonacci numbers recursively."
    *   `criteria`: "Function should be recursive, handle base cases (0, 1), and be syntactically correct Python."
    *   `config_overrides`: `{ "mode": "code" }`
2.  **Run Evals:** Developer runs `pnpm start --suite python_codegen --model gpt-4-turbo` from the `evals/` directory.
3.  **Eval Runner (`runEval.ts` / `evaluateTask.ts`):**
    *   Parses args (`yargs`). Finds `python_codegen.yaml` (`findYamlFiles`). Loads/validates tasks using `js-yaml` and `evalTaskSchema`.
    *   For task `py_fib_recursive` and model `gpt-4-turbo`:
        *   Calls `evaluateTask`.
        *   `evaluateTask` merges config, prepares context (none needed here). **Requires API key for `gpt-4-turbo` via environment variable.**
        *   Calls programmatic entry point `runRooCodeTaskForEval(prompt, context, config)`.
        *   `runRooCodeTaskForEval` (conceptual) instantiates `Cline` with mocks (for VS Code API) and specified config. Runs the task loop. Captures all emitted `ClineMessage`s. Returns transcript and metrics.
        *   `evaluateTask` receives transcript/metrics, calculates time, assembles `EvalOutput`.
        *   Runner saves `EvalOutput` to `evals/output/<ts>/python_codegen/gpt-4-turbo/py_fib_recursive.json`.
4.  **Review Results:** Developer uses the `evals-web` UI ([Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md)) to view the result. They examine the `transcript` (containing the generated Python code) and compare it against the `criteria`.
5.  **Analysis:** Assess if the code is correct. Compare with previous runs if available.

## Key Concepts

1.  **Evaluation Suites (`evals/suites/*.yaml`):** YAML files defining test cases (tasks). Each task specifies `id`, `prompt`, optional `context` (files, etc.), `criteria`, and `config_overrides`. Validated by `evalTaskSchema`.
2.  **Eval Runner (`evals/src/runEval.ts`, `evaluateTask.ts`):** Node.js scripts orchestrating the evaluation.
    *   **`runEval.ts`:** Parses CLI args (`yargs`), finds/loads/validates suite files, iterates through suites/models/tasks, calls `evaluateTask`, saves results/errors to structured JSON files in `evals/output/`. Uses script utilities ([Chapter 55: Script Utilities](55_script_utilities.md)).
    *   **`evaluateTask.ts`:** Executes a single task. Prepares context, merges config (including model and API keys from env vars), invokes core Roo-Code logic programmatically, captures the full `ClineMessage` transcript, calculates metrics (time, cost using [Chapter 29: Cost Calculation Utilities](29_cost_calculation_utilities.md)), cleans up context, and returns structured `EvalOutput`.
3.  **Programmatic Invocation (`runRooCodeTaskForEval` - Conceptual):** The critical link allowing the core `Cline` logic ([Chapter 4: Cline](04_cline.md)) to run outside VS Code. Requires careful dependency injection or mocking for VS Code APIs (UI prompts, workspace, terminal, secrets). Must capture the output stream/events (`ClineMessage[]`).
4.  **Output Format (`EvalOutput`):** Defined by Zod schema `evalOutputSchema` (`evals/schemas/`). Stores inputs and outputs (including full `transcript`) in JSON for analysis and reproducibility. Uses `clineMessageSchema`.
5.  **Evaluation/Analysis:** Primarily manual review against `criteria` using output JSON or the `evals-web` UI. Potential for automated metrics.
6.  **Web UI (`evals-web/`):** Separate Next.js/React application for browsing, viewing, comparing eval results. Uses standard web shadcn/ui primitives ([Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md)). Reads JSON outputs from `evals/output`.

## Using the Evals System

Primarily a developer tool for testing and regression analysis.

**Workflow:**

1.  **Define Suites:** Create/edit `evals/suites/*.yaml`.
2.  **Run Runner:** Execute `pnpm start --suite <name> --model <id>` from `evals/`. Ensure necessary API keys are available as environment variables (e.g., `OPENAI_API_KEY`).
3.  **Monitor:** Check console output.
4.  **Analyze:** Inspect JSON files in `evals/output/` or use the `evals-web` UI (`pnpm dev` in `evals-web/`). Compare transcripts against criteria and across runs.
5.  **Iterate:** Modify code/prompts, re-run evals, compare.

## Code Walkthrough

### Eval Suite Example (`evals/suites/example.yaml`)

*(See code in Key Concepts section)*

### Eval Runner (`evals/src/runEval.ts` - Conceptual)

*(See code in Key Concepts section)*
*   Uses `yargs`, `findYamlFiles`, `js-yaml`, `evalTaskSchema`. Loops through suites/models/tasks. Calls `evaluateTask`. Saves output.

### Task Evaluation (`evals/src/evaluateTask.ts` - Conceptual)

*(See code in Key Concepts section)*
*   Receives `EvalTask`, `modelIdToUse`. Merges config (needs API keys from `process.env`). Prepares/Cleans context. **Invokes core logic (complex part needing mocks/entry point).** Captures `transcript`. Calculates `metrics`. Assembles `EvalOutput`. Returns result.

### Schemas (`evals/schemas/`)

*   **`evalTask.ts` (`evalTaskSchema`):** Zod schema for YAML task definitions.
*   **`evalOutput.ts` (`evalOutputSchema`):** Zod schema for JSON output files. Includes `transcript: z.array(clineMessageSchema)`, `metrics`, etc.

### Web UI (`evals-web/`)

*   Separate Next.js/React project. Reads JSON outputs. Uses standard shadcn/ui primitives ([Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md)) for display.

## Internal Implementation

1.  **Runner:** Node.js script parses args, finds/loads/validates YAMLs.
2.  **Task Loop:** Calls `evaluateTask` for each task/model.
3.  **`evaluateTask`:** Sets up temp context, merges config, calls `runRooCodeTaskForEval`.
4.  **`runRooCodeTaskForEval`:** Instantiates core components (`ApiHandler`, mocked `ContextProxy`/VSCode APIs, `Cline`). Runs task loop. Captures `ClineMessage`s via listener/stream. Returns transcript/metrics.
5.  **`evaluateTask`:** Cleans up context, calculates metrics, returns `EvalOutput`.
6.  **Runner Saves Output:** Writes `EvalOutput` to JSON.
7.  **Web UI:** Reads JSONs, renders data using React & shadcn/ui.

**Sequence Diagram (Runner executing one task):**

*(See diagram in Chapter 53 - Internal Implementation)*

## Modification Guidance

Modifications typically involve adding suite features, runner options, evaluation metrics, or enhancing the analysis UI.

1.  **Adding New Context Type (e.g., Mock Terminal State):**
    *   **Schema (`evalTaskSchema`):** Add field to `context`.
    *   **YAML:** Add context to tasks.
    *   **`evaluateTask.ts`:** Pass mock state to core logic invocation.
    *   **Core Logic:** Ensure mocked terminal uses this state.

2.  **Adding Automated Metric (e.g., Code Linting):**
    *   **`evaluateTask.ts`:** Extract code from transcript, write temp file, run linter via `child_process`, record results in `metrics`.
    *   **Schema (`evalOutputSchema`):** Add metric field.
    *   **Analysis/UI:** Display/use metric. Consider security.

3.  **Supporting Parallel Execution:**
    *   **`runEval.ts`:** Use `p-limit` around `evaluateTask` calls.
    *   **`evaluateTask.ts`:** Ensure context setup/cleanup uses unique names. Be mindful of API rate limits.

4.  **Enhancing Web UI (`evals-web/`):**
    *   Modify Next.js pages/components. Use shadcn/ui primitives ([Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md)) to add filtering, sorting, diff views, charts, annotations. Read data from JSON outputs.

**Best Practices:**

*   **Clear Criteria:** Define specific, verifiable success criteria.
*   **Realistic Context:** Use representative prompts/context.
*   **Isolate Core Logic:** Create clean programmatic entry point minimizing VS Code dependencies.
*   **Robust Mocking:** Mock dependencies accurately or disable features gracefully.
*   **Structured Output:** Use schemas (`evalOutputSchema`). Store inputs+outputs.
*   **Version Control Suites:** Keep YAMLs in Git.
*   **Reproducibility:** Record config. Fix model versions if possible.
*   **Balance Automated/Manual:** Use automated metrics + manual review.
*   **Security:** Be cautious executing AI-generated code during evals.

**Potential Pitfalls:**

*   **Environment Differences:** Evals outside VS Code behaving differently due to inaccurate mocking.
*   **Mocking Complexity:** Mocking `Cline`'s environment is hard.
*   **Context Setup/Cleanup Errors.**
*   **API Keys/Costs:** Securely manage keys (env vars). Be mindful of costs.
*   **Flaky LLMs/Tests:** Non-determinism requires careful criteria or multiple runs.
*   **Eval Suite Maintenance:** Keeping suites relevant and updated.

## Conclusion

The Evals System provides an essential framework for systematically evaluating and improving the performance of Roo-Code's AI capabilities. By defining standardized test suites (YAML), using a runner script (`runEval.ts`, `evaluateTask.ts`) to execute tasks programmatically against different models/configurations, and capturing detailed output transcripts and metrics in a structured format (JSON), developers can objectively assess correctness, prevent regressions, and guide future improvements. While setting up the programmatic invocation of the core logic requires careful handling of dependencies and mocking, the benefits of automated, data-driven quality assessment are significant for developing a reliable AI assistant. The optional `evals-web` UI further enhances the analysis and comparison of results.

Next, we look at the UI primitives specifically used within the Evals web interface: [Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md).
---
# Chapter 54: Shadcn/UI Primitives (Evals Web)

Continuing from [Chapter 53: Evals System](53_evals_system.md), which detailed the framework for running automated evaluations of Roo-Code's AI, this chapter focuses on the user interface components used within the **Evals Web** application – the separate web interface designed for reviewing and analyzing evaluation results. Specifically, we look at the **Shadcn/UI Primitives** used in this context.

## Motivation: Building a Dedicated Web UI for Eval Analysis

While the main Roo-Code extension UI ([Chapter 1: WebView UI](01_webview_ui.md)) runs within VS Code and needs to match its theme precisely, the Evals Web UI is a standalone web application (likely built with Next.js or a similar framework like Vite/React) running in a standard browser. Its purpose is different: presenting potentially large amounts of structured evaluation data (results from `evals/output/`), enabling comparisons between different runs (e.g., different models, different code versions), filtering, sorting, and potentially annotating results.

This separate context allows for different UI choices. While consistency with VS Code is less critical, building a clean, modern, and functional web interface still requires a good set of UI components. `shadcn/ui` is again chosen for this purpose, but used in a more standard web configuration compared to its adaptation for the VS Code WebView ([Chapter 33: Shadcn/UI Primitives (WebView)](33_shadcn_ui_primitives__webview_.md)).

The reasons for using `shadcn/ui` here are similar to its use in the WebView, but with slightly different emphasis:

1.  **Rich Component Set:** Provides essential components for building data-rich web applications: `Table`, `Dialog`, `Select`, `Tabs`, `Card`, `Button`, `Badge`, `Tooltip`, `Accordion`, `Input`, `Checkbox`, `Separator`, `DropdownMenu`, `Popover`, etc., suitable for displaying complex evaluation results.
2.  **Composability & Customization:** Components are easily composed and fully customizable by having the source code directly in the `evals-web` project (`evals-web/components/ui/`). This allows tailoring the components specifically for the needs of the evaluation dashboard.
3.  **Modern Styling (Tailwind CSS):** Uses Tailwind CSS for styling, allowing for rapid development of a clean, modern aesthetic suitable for a standalone web application. Theming (light/dark mode) uses standard Tailwind theme configuration and CSS variables, independent of the VS Code environment.
4.  **Accessibility:** Built on Radix UI primitives, ensuring good accessibility out of the box.

**Central Use Case:** Displaying a table comparing the results of the same evaluation task (e.g., `py_fib_recursive` from [Chapter 53: Evals System](53_evals_system.md)) run against different LLM models (`gpt-4-turbo`, `claude-3-opus`).

1.  The Evals Web app (a Next.js page) fetches the relevant JSON output files generated by the eval runner (e.g., `evals/output/.../gpt-4-turbo/py_fib_recursive.json`, `evals/output/.../claude-3-opus/py_fib_recursive.json`).
2.  A React component within `evals-web` needs to render this comparison.
3.  It uses the `Table` components (`Table`, `TableHeader`, `TableRow`, `TableHead`, `TableBody`, `TableCell`) from `evals-web/components/ui/` (the standard shadcn/ui primitives copied into this project).
4.  The table has columns for "Model", "Status", "Cost", "Tokens", and potentially a snippet of the response or a link/button to view the full transcript.
5.  Data for each model's result (`EvalOutput`) is mapped to a `TableRow`. `TableCell` renders individual data points (model ID, error status, metrics).
6.  Components like `Badge` are used to display the status ("Success", "Error"). `Tooltip` might show full cost details on hover. `Button` (or a link styled as one) navigates to a detailed view showing the full transcript for that specific run.
7.  The table renders with clean styling defined by Tailwind CSS utility classes applied within the shadcn/ui component implementations, using the standard web theme configured in `evals-web/tailwind.config.js`.

## Key Concepts

1.  **Standalone Web Application (`evals-web/`):** Runs independently in a browser, built with a framework like Next.js or Vite+React. Its dependencies and styling are separate from the main VS Code extension and WebView UI.
2.  **Shadcn/UI Primitives (`evals-web/components/ui/`):** Contains the source code for components like `Button`, `Table`, `Dialog`, `Tabs`, `Select`, `Badge`, etc., copied and potentially slightly adapted from the main shadcn/ui library ([https://ui.shadcn.com/](https://ui.shadcn.com/)). These are standard React components.
3.  **Standard Web Styling:** Components are styled using Tailwind CSS with a standard web theme configuration (`evals-web/tailwind.config.js` and likely `globals.css`). It defines colors (e.g., `primary`, `background`, `destructive`), spacing, fonts, etc., using CSS variables for light/dark mode, but **not** VS Code CSS variables (`--vscode-*`).
4.  **Radix UI & Tailwind CSS:** Leverages Radix UI primitives for underlying behavior/accessibility and Tailwind CSS for applying utility-based styles.
5.  **Component Set:** Includes primitives well-suited for data dashboards and web applications: `Table`, `Card`, `Badge`, `Tabs`, `Select`, `DropdownMenu`, `Dialog`, `Popover`, `Tooltip`, `Button`, `Input`, `Checkbox`, `Separator`, `Accordion`, etc.
6.  **`cn` Utility:** Uses the standard `cn` utility (e.g., from `evals-web/lib/utils`) combining `clsx` and `tailwind-merge` for flexible class name composition within components.

## Using the Primitives in Evals Web

Usage follows standard React development patterns within the `evals-web` application. Components are imported from the local `components/ui` directory (often aliased as `@/components/ui`).

**Example: Rendering a Comparison Table (Conceptual)**

```typescript
// --- File: evals-web/components/results/ComparisonTable.tsx ---
import React from 'react';
// Import primitives from the Evals Web UI directory
import {
  Table, TableBody, TableCell, TableHead, TableHeader, TableRow,
} from '@/components/ui/table'; // Use path alias
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Tooltip, TooltipContent, TooltipProvider, TooltipTrigger } from '@/components/ui/tooltip';
import Link from 'next/link'; // Assuming Next.js for links
// Import shared schema type - assumes accessible path
// The path needs to be relative to this file's location or use path aliases
import { EvalOutput } from '../../schemas/evalOutput'; // Adjusted relative path

interface ComparisonTableProps {
  results: EvalOutput[]; // Array of results for the same task from different runs/models
  taskId: string; // ID of the task being compared
}

const ComparisonTable: React.FC<ComparisonTableProps> = ({ results, taskId }) => {
  // Helper functions for formatting
  const formatCost = (cost?: number) => cost !== undefined ? `$${cost.toFixed(6)}` : 'N/A';
  const formatTokens = (tokens?: number) => tokens?.toLocaleString() ?? 'N/A';

  return (
    // TooltipProvider is needed for Tooltip components to work
    <TooltipProvider>
      <Table>
        <TableHeader>
          <TableRow>
            <TableHead>Model</TableHead>
            <TableHead>Status</TableHead>
            <TableHead className="text-right">Cost</TableHead>
            <TableHead className="text-right">Input Tokens</TableHead>
            <TableHead className="text-right">Output Tokens</TableHead>
            <TableHead>Actions</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          {results.map((result) => {
            // Determine a unique key, modelId might not be unique if run multiple times
            const resultKey = `${result.config?.modelId}-${result.metrics?.executionTimeMs || Math.random()}`;
            const modelId = result.config?.modelId ?? 'Unknown Model';
            // Construct a unique ID for linking, perhaps combining suite/task/model if needed
            // Example: assumes output structure like output/<timestamp>/<suiteName>/<modelId>/<taskId>.json
            const suiteName = result.config?.suiteName || 'unknown_suite'; // Assume suiteName is in config
            const resultDetailId = `${suiteName}/${taskId}/${modelId}`; // Example identifier for URL

            return (
              <TableRow key={resultKey}>
                <TableCell className="font-medium">{modelId}</TableCell>
                <TableCell>
                  {/* Use Badge for status based on presence of error */}
                  <Badge variant={result.error ? 'destructive' : 'default'}>
                    {result.error ? 'Error' : 'Success'}
                  </Badge>
                </TableCell>
                <TableCell className="text-right">
                  {/* Use Tooltip for detailed cost breakdown */}
                  <Tooltip>
                    <TooltipTrigger asChild>
                       {/* Make trigger focusable for accessibility */}
                       <button type="button" className="cursor-default">{formatCost(result.metrics?.totalCost)}</button>
                    </TooltipTrigger>
                    <TooltipContent>
                      {/* TODO: Fetch prices from config object if available */}
                      <p>Input: {formatTokens(result.metrics?.totalTokensIn)} tokens</p>
                      <p>Output: {formatTokens(result.metrics?.totalTokensOut)} tokens</p>
                      {/* Add cache/other costs if available in metrics */}
                    </TooltipContent>
                  </Tooltip>
                </TableCell>
                <TableCell className="text-right">{formatTokens(result.metrics?.totalTokensIn)}</TableCell>
                <TableCell className="text-right">{formatTokens(result.metrics?.totalTokensOut)}</TableCell>
                <TableCell>
                  {/* Link to detailed view using Next.js Link */}
                  {/* Ensure URL structure matches page routing */}
                  <Link href={`/evals/${resultDetailId}`} passHref legacyBehavior>
                    <Button variant="outline" size="sm" asChild>
                      {/* Wrap anchor tag for correct styling and behavior */}
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

1.  **Imports:** Imports primitives (`Table`, `Badge`, `Button`, `Tooltip`) from `@/components/ui` within `evals-web`. Imports `EvalOutput` type from the shared `evals/schemas` directory using a relative path (adjust based on structure or aliases).
2.  **Composition:** Uses standard `Table` components. Uses `Badge` for status and `Tooltip` (requires wrapping parent in `TooltipProvider`) for cost details.
3.  **Styling:** Uses Tailwind utilities (`text-right`, `font-medium`). `Badge` uses `variant="destructive"`. Styles derived from the `evals-web` Tailwind theme.
4.  **Data Mapping:** Maps `results` array (`EvalOutput[]`). Accesses `result.config.modelId`, `result.error`, `result.metrics`. Formats cost/tokens.
5.  **Interaction:** Uses Next.js `Link` wrapping a `Button` (with `asChild`) to navigate to a detail page. The URL structure (`/evals/${suiteName}/${taskId}/${modelId}`) assumes how results might be organized and routed.

## Code Walkthrough

The primitive components in `evals-web/components/ui/` are standard shadcn/ui implementations, using Radix UI and styled with Tailwind CSS according to the web application's theme.

### Example Primitive (`evals-web/components/ui/table.tsx`)

*(Likely standard shadcn/ui table)*

```typescript
// --- File: evals-web/components/ui/table.tsx ---
import * as React from "react"
import { cn } from "@/lib/utils" // Path relative to evals-web project

// Basic wrappers around standard HTML table elements, applying Tailwind classes

const Table = React.forwardRef<HTMLTableElement, React.HTMLAttributes<HTMLTableElement>>(
  ({ className, ...props }, ref) => ( <div className="relative w-full overflow-auto"> <table ref={ref} className={cn("w-full caption-bottom text-sm", className)} {...props} /> </div> )
)
Table.displayName = "Table"

const TableHeader = React.forwardRef<HTMLTableSectionElement, React.HTMLAttributes<HTMLTableSectionElement>>(
  ({ className, ...props }, ref) => ( <thead ref={ref} className={cn("[&_tr]:border-b", className)} {...props} /> )
)
TableHeader.displayName = "TableHeader"

const TableBody = React.forwardRef<HTMLTableSectionElement, React.HTMLAttributes<HTMLTableSectionElement>>(
  ({ className, ...props }, ref) => ( <tbody ref={ref} className={cn("[&_tr:last-child]:border-0", className)} {...props} /> )
)
TableBody.displayName = "TableBody"

const TableFooter = React.forwardRef<HTMLTableSectionElement, React.HTMLAttributes<HTMLTableSectionElement>>(
  ({ className, ...props }, ref) => ( <tfoot ref={ref} className={cn("border-t bg-muted/50 font-medium [&>tr]:last:border-b-0", className)} {...props} /> )
)
TableFooter.displayName = "TableFooter"

const TableRow = React.forwardRef<HTMLTableRowElement, React.HTMLAttributes<HTMLTableRowElement>>(
  ({ className, ...props }, ref) => ( <tr ref={ref} className={cn("border-b transition-colors hover:bg-muted/50 data-[state=selected]:bg-muted", className)} {...props} /> )
)
TableRow.displayName = "TableRow"

const TableHead = React.forwardRef<HTMLTableCellElement, React.ThHTMLAttributes<HTMLTableCellElement>>(
  ({ className, ...props }, ref) => ( <th ref={ref} className={cn("h-12 px-4 text-left align-middle font-medium text-muted-foreground [&:has([role=checkbox])]:pr-0", className)} {...props} /> )
)
TableHead.displayName = "TableHead"

const TableCell = React.forwardRef<HTMLTableCellElement, React.TdHTMLAttributes<HTMLTableCellElement>>(
  ({ className, ...props }, ref) => ( <td ref={ref} className={cn("p-4 align-middle [&:has([role=checkbox])]:pr-0", className)} {...props} /> )
)
TableCell.displayName = "TableCell"

const TableCaption = React.forwardRef<HTMLTableCaptionElement, React.HTMLAttributes<HTMLTableCaptionElement>>(
  ({ className, ...props }, ref) => ( <caption ref={ref} className={cn("mt-4 text-sm text-muted-foreground", className)} {...props} /> )
)
TableCaption.displayName = "TableCaption"

export { Table, TableHeader, TableBody, TableFooter, TableHead, TableRow, TableCell, TableCaption }

```

**Explanation:**

*   **Structure:** Provides React components wrapping standard HTML table elements (`<table>`, `<thead>`, `<tbody>`, `<tr>`, `<th>`, `<td>`).
*   **Styling:** Uses `cn` to apply base Tailwind classes for structure, borders (`border-b`), hover effects (`hover:bg-muted/50`), padding (`p-4`, `px-4`), text alignment (`text-left`), and colors (`text-muted-foreground`, `bg-muted/50`). These classes (`muted`, `muted-foreground`, `border`) map to the standard web theme defined in the `evals-web` Tailwind config.

### Tailwind Configuration (`evals-web/tailwind.config.js` - Conceptual)

*(See code in Chapter 54 - Key Concepts)*
*   Defines a standard web color palette (light/dark mode) using CSS variables (e.g., `--background`, `--primary`) defined in `globals.css`. **Does not** use `--vscode-*` variables.

## Internal Implementation

Standard React rendering with Tailwind CSS:
1.  React component uses `<Table>`, `<TableCell>`, etc.
2.  These components render HTML elements (`<table>`, `<td>`) with Tailwind classes applied via `cn`.
3.  Tailwind build generates CSS rules based on `evals-web/tailwind.config.js` (e.g., `.p-4 { padding: 1rem; }`, `.hover\\:bg-muted\\/50:hover { background-color: hsl(var(--muted) / 0.5); }`).
4.  Global CSS defines the HSL values for `--muted`, etc.
5.  Browser applies the CSS rules, rendering a standard styled web table.

## Modification Guidance

Modifications involve customizing the theme or adapting/adding primitives within the `evals-web` project, following standard web development practices.

1.  **Changing the Evals Web Theme:**
    *   **CSS Variables:** Edit HSL values in `globals.css` for light/dark modes.
    *   **Tailwind Config:** Adjust semantic mappings or add new colors in `evals-web/tailwind.config.js`. Rebuild CSS.

2.  **Adapting/Adding a New Shadcn/UI Primitive:**
    *   Use the shadcn/ui CLI (`npx shadcn-ui@latest add ...`) within the `evals-web` directory or manually copy component source code into `evals-web/components/ui/`.
    *   The standard component should work directly with the existing Tailwind theme. No mapping to VS Code variables is needed.
    *   Update `tailwind.config.js` or global CSS if the component introduces new theme variables or requires plugins.

**Best Practices:**

*   **Standard Shadcn/UI:** Leverage the standard components and structure for consistency within the Evals Web app.
*   **Theme via CSS Variables:** Define the palette in global CSS using variables for easy theming.
*   **Tailwind Configuration:** Ensure `tailwind.config.js` is set up correctly for the `evals-web` project (content paths, theme definitions, plugins).
*   **Keep Separate from WebView UI:** Do not mix Evals Web UI primitives (standard web theme) with WebView UI primitives (VS Code theme). Maintain this separation clearly.

**Potential Pitfalls:**

*   **Incorrect Tailwind Setup:** Errors in config or CSS variables break styling.
*   **CSS Conflicts:** Other global styles interfering.
*   **Build Process:** Ensure Tailwind build is integrated correctly with Next.js/Vite.
*   **Dependency Mismatches:** Ensure Radix UI and other dependencies are compatible within `evals-web`.

## Conclusion

The shadcn/ui primitives provide a modern, flexible, and efficient way to build the user interface for the standalone **Evals Web** application (`evals-web/`). By adopting the standard shadcn/ui components and styling them with a conventional web-oriented Tailwind CSS theme (using CSS variables for light/dark modes), the `evals-web` project can create sophisticated data displays (like tables and cards) and interactions necessary for analyzing and comparing AI evaluation results. These components are intentionally styled differently from their counterparts in the main VS Code WebView, reflecting their different execution context and purpose, offering a familiar and productive development experience for building web-based dashboards.

Next, we look at utilities specifically created for scripting tasks within the Roo-Code project, often used during development, building, or evaluation processes: [Chapter 55: Script Utilities](55_script_utilities.md).
---
# Chapter 55: Script Utilities

Continuing from [Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md), which focused on UI components for the evaluation web interface, this chapter shifts focus to backend utilities used across various development and operational scripts within the Roo-Code project: the **Script Utilities**.

## Motivation: Common Helpers for Development and Build Scripts

Beyond the core extension runtime and UI code, a software project like Roo-Code involves numerous supporting scripts for tasks such as:

*   Building the extension ([Chapter 56: Build System](56_build_system.md)).
*   Running linters, formatters, or tests ([Chapter 57: Testing Framework](57_testing_framework.md)).
*   Generating documentation or schemas.
*   Running evaluation suites ([Chapter 53: Evals System](53_evals_system.md)).
*   Performing release tasks (e.g., updating versions, packaging).
*   Finding missing translation keys (e.g., `scripts/find-missing-translations.js` from [Chapter 50: Localization System (i18n)](50_localization_system__i18n_.md)).

These scripts, often written in TypeScript or JavaScript and executed using Node.js (e.g., via `pnpm run script-name`), frequently require common helper functionalities:

*   Parsing command-line arguments.
*   Interacting with the file system robustly (reading directories, finding files by pattern, checking paths) ([Chapter 42: File System Utilities](42_file_system_utilities.md), [Chapter 43: Path Utilities](43_path_utilities.md) might be reused or adapted).
*   Executing shell commands or other scripts reliably and capturing output.
*   Handling logging consistently across different scripts with clear formatting (e.g., colors).
*   Managing asynchronous operations, potentially with concurrency limits.

Implementing these helpers repeatedly in each script is inefficient and leads to inconsistency. The **Script Utilities** (conceptually located in directories like `scripts/lib/`, `evals/src/lib/`, or shared `src/utils/` if suitable and VS Code API independent) provide a collection of reusable functions tailored for these Node.js scripting contexts.

**Central Use Case:** The Evals runner script (`evals/src/runEval.ts`, [Chapter 53: Evals System](53_evals_system.md)) needs to find all `.yaml` suite files within a specified directory provided via a command-line argument `--suite`.

Without Utilities:
```typescript
// Conceptual script code without specific utils
import * as fs from "fs/promises";
import * as path from "path";
import yargs from "yargs";

async function run() {
    const argv = yargs(process.argv.slice(2)).option("suiteDir", { type: "string" }).parseSync();
    const suiteDir = path.resolve(argv.suiteDir || "evals/suites");
    let suiteFiles: string[] = [];

    try {
        // Complex logic to check if suiteDir is file or dir,
        // then recursively read dir, filter by .yaml, handle errors...
        // (Requires recursive readdir or external library like glob)
    } catch (error: any) { console.error(error.message); process.exit(1); }
    // ... use suiteFiles ...
}
```

With Utilities:
```typescript
// Conceptual script code using utils
import yargs from "yargs";
// Assuming utility exists in evals/src/lib for this specific need
import { findYamlFiles } from "../lib/fileUtils";
import { scriptLogger as logger } from "../lib/logger"; // Assuming logger utility

async function run() {
    const argv = yargs(process.argv.slice(2)).option("suitePath", { type: "string" }).parseSync();
    const suitePath = argv.suitePath || "evals/suites";

    try {
        // Single call handles file/dir check, recursion, filtering, errors
        const suiteFiles = await findYamlFiles(suitePath);
        if (suiteFiles.length === 0) {
             logger.error(`No .yaml files found at path: ${suitePath}`);
             process.exit(1);
        }
        logger.info(`Found ${suiteFiles.length} suite files.`);
        // ... use suiteFiles ...
    } catch (error: any) {
        logger.error(`Failed to find suite files: ${error.message}`, error);
        process.exit(1);
     }
}
```
The `findYamlFiles` utility encapsulates the file system interaction and error handling, making the main script cleaner and more focused on its primary task.

## Key Concepts

1.  **Target Environment:** Node.js execution context, typically run via `pnpm`, `npm`, or `yarn` scripts. Can leverage full Node.js API set and `devDependencies`. **Cannot** use `vscode` module APIs.
2.  **Common Functionality:**
    *   **Argument Parsing:** Using/wrapping `yargs`, `commander`.
    *   **File System:** Robust wrappers/specialized functions using `fs/promises`, `glob` (e.g., `findFiles`, `readFileSafe`, `writeFileSafe`, `copyDirRecursive`, `ensureDir`). May reuse `src/utils/fs.ts` helpers if VS Code API independent.
    *   **Shell Command Execution:** Helpers using `child_process` or `execa` for running commands, capturing output, checking exit codes.
    *   **Logging:** Console logger (`console`) enhanced with `chalk` for colors, providing consistent `info`, `warn`, `error`, `success` methods.
    *   **Path Manipulation:** Using/wrapping Node.js `path`. Reusing `src/utils/path.ts` helpers like `normalizePath`, `toPosix()`, `arePathsEqual` that are OS/Node based. **Cannot** use `getWorkspacePath` which relies on `vscode.workspace`. Base paths must come from `process.cwd()` or arguments.
    *   **Concurrency Control:** Using `p-limit` to manage parallel async tasks.
    *   **JSON/YAML Handling:** Safe reading/parsing/writing, potentially with Zod validation ([Chapter 40: Schemas (Zod)](40_schemas__zod_.md)).
3.  **Location:** Organized within `scripts/lib/` (general build/dev scripts) or `evals/src/lib/` (eval-specific). Shared, Node-compatible utilities might be in `src/utils/`.
4.  **Reusability:** Imported across different script files (`*.js`, `*.ts`). TypeScript scripts require execution via `ts-node` or prior compilation (like Roo-Code's build system produces).

## Using Script Utilities

Imported as standard functions/modules within script files.

**Example 1: Finding Files (`evals/src/lib/findSuites.ts` - Conceptual)**

*(See code in Key Concepts section)*
*   `findYamlFiles` utility uses `fs.stat` and `glob` library. Used by `runEval.ts`.

**Example 2: Executing a Shell Command (`scripts/lib/exec.ts` - Conceptual)**

*(See code in Key Concepts section)*
*   `runCommand` utility wraps `execa` (preferred) or `child_process.exec`. Provides logging, streaming (`stdio: 'inherit'`), and robust error handling. Used by build scripts.

**Example 3: Logging (`scripts/lib/logger.ts` - Conceptual)**

*(See code in Key Concepts section)*
*   `scriptLogger` object uses `chalk` for colored console output. Used by scripts for consistent status updates.

**Example 4: Reading/Validating JSON (`scripts/lib/fsUtils.ts` - Conceptual)**

*(See code in Key Concepts section)*
*   `readAndValidateJson` utility combines `safeReadFile`, `JSON.parse`, Zod `safeParse`, and `scriptLogger`. Used by scripts needing to load validated config files.

## Code Walkthrough

Examining utilities potentially used or defined within the project structure.

### Shared Utilities (`src/utils/`)

*   **`fs.ts` ([Chapter 42]):** `fileExistsAtPath`, `directoryExistsAtPath`, `safeWriteFile`, `createDirectoriesForFile`, `safeReadFile` are usable if absolute paths are provided.
*   **`path.ts` ([Chapter 43]):** `normalizePath`, `toPosix()`, `arePathsEqual`, `relativePath`/`toRelativePath` (require `cwd` argument), `getReadablePath` (requires `cwd` argument) are usable. `getWorkspacePath` **is NOT usable**.
*   **`logging.ts`:** The `logger` instance might need configuration for console output in scripts. Using a dedicated `scriptLogger` might be simpler.
*   **`text-normalization.ts` ([Chapter 45]):** Pure string functions (`normalizeLineEndings`, `normalizeString`, `unescapeHtmlEntities`) are directly usable.

### Evals Utilities (`evals/src/lib/`)

*   **`openai.ts` (from Ch 55):** `getCompletions` using `axios`. Network utility for OpenAI-compatible APIs.
*   **`findSuites.ts` (Conceptual `findYamlFiles`):** Uses `fs.stat`, `path.resolve`, `glob`. File system utility.
*   **`contextUtils.ts` (Conceptual):** Uses `fs/promises` (`mkdtemp`, `writeFile`, `rm`) and `path` for managing temporary eval context files.

### Build Script Utilities (`scripts/lib/`)

*   **`copyWasm.js` (Conceptual):** `esbuild` plugin logic using Node.js `fs` ([Chapter 56]). File system utility.
*   **`exec.ts` (Conceptual `runCommand`):** Uses `execa`. Process execution utility.
*   **`logger.ts` (Conceptual `scriptLogger`):** Uses `console` and `chalk`. Logging utility.
*   **(Other potential utils):** Reading/writing JSON (`fsUtils.ts`), Git interactions (`gitUtils.ts`), path finding, etc.

## Internal Implementation

*   **File System:** Node.js `fs/promises` API, `glob`.
*   **Command Execution:** Node.js `child_process` (`spawn`, `exec`, `fork`), `execa`.
*   **Argument Parsing:** `yargs`.
*   **Logging:** Node.js `console`, `chalk`.
*   **Path:** Node.js `path`.
*   **Concurrency:** `p-limit`.

## Modification Guidance

Modifications usually involve adding new helper functions for common script tasks or improving existing ones for robustness or flexibility.

1.  **Adding a `waitForFile` Utility:** (As shown in Ch 55) Implement using `fileExistsAtPath` and `setTimeout` promise.
2.  **Improving `runCommand` to Return Output (Instead of Streaming):**
    *   **Modify:** Change `runCommand` in `scripts/lib/exec.ts` to use `execa` *without* `stdio: 'inherit'`. Access `result.stdout` and `result.stderr` after the promise resolves.
        ```typescript
        import { execa, Options, ExecaReturnValue } from 'execa';
        import chalk from 'chalk';

        export async function runCommandGetOutput(
            command: string,
            args: string[] = [],
            options: Options = {}
        ): Promise<ExecaReturnValue> { // Returns full result object
            const commandString = `${command} ${args.join(" ")}`;
            console.log(chalk.blue(`$ ${commandString} ${options.cwd ? `(in ${options.cwd})`: ''}`));
            try {
                // Use default stdio (pipe) to capture output
                const result = await execa(command, args, { ...options });
                return result; // Includes stdout, stderr, exitCode etc.
            } catch (error: any) {
                console.error(chalk.red(`Command failed: ${commandString}`));
                // error object from execa already contains stdout, stderr, exitCode etc.
                throw error; // Re-throw execa error
            }
        }
        // Usage: const { stdout } = await runCommandGetOutput("git", ["status", "--short"]);
        ```
    *   **Use Case:** For commands where you need to capture and process the output string, rather than just see it live.

3.  **Adding a Git Helper (`scripts/lib/gitUtils.ts`):**
    *   **Define:** Create file. Import `runCommandGetOutput`.
    *   **Implement:**
        ```typescript
        import { runCommandGetOutput } from './exec';
        export async function checkGitDirty(cwd: string): Promise<boolean> {
            try {
                const { stdout } = await runCommandGetOutput("git", ["status", "--porcelain"], { cwd });
                return stdout.trim().length > 0; // Dirty if porcelain status has output
            } catch (error) {
                console.warn("Git status check failed (is git installed/in repo?)", error);
                return true; // Assume dirty or problematic if check fails
            }
        }
        export async function getCurrentBranch(cwd: string): Promise<string | undefined> { /* git rev-parse --abbrev-ref HEAD */ }
        ```
    *   **Usage:** Release scripts call `checkGitDirty` before proceeding.

**Best Practices:**

*   **Reusability:** Extract common logic into `scripts/lib/` or `evals/src/lib/`. Share with `src/utils/` only if fully Node-compatible (no `vscode` imports).
*   **Error Handling:** Scripts should fail fast. Utilities should throw/reject on critical errors. Log clearly using `scriptLogger`.
*   **Cross-Platform:** Use Node.js `path`, `os`. Use `execa` or `spawn` with arg arrays for commands. Use `cross-env`.
*   **Logging:** Use consistent, colored logging.
*   **Async:** Use `async/await`. Use concurrency limiters (`p-limit`) for parallel tasks.
*   **Configuration:** Read config via env vars (`process.env`), args (`yargs`), or JSON/YAML files (using FS utils + Zod). Avoid hardcoding.
*   **Dependencies:** Use `devDependencies`.

**Potential Pitfalls:**

*   **VS Code API Usage:** Trying to use `vscode` in standalone scripts. Requires careful separation or alternative implementations in `scripts/lib`.
*   **Path Resolution:** Incorrect CWD assumptions. Use `path.resolve`, `__dirname`.
*   **Shell Command Failures:** Need reliable error checking (exit codes, `stderr`). `execa` helps.
*   **Unhandled Rejections.**
*   **Environment Differences (CI vs. Local).**

## Conclusion

Script Utilities provide essential helper functions that streamline the development, build, testing, and evaluation processes for the Roo-Code project. By encapsulating common tasks like argument parsing, robust file system operations (often reusing core utilities), reliable command execution (using `execa`), and consistent logging (`scriptLogger` with `chalk`) into reusable modules tailored for the Node.js environment, they reduce code duplication, improve consistency, and make individual scripts cleaner and easier to maintain. While potentially sharing some low-level logic with runtime utilities, script utilities can directly leverage the full Node.js API and external CLI tools, distinct from the constraints of the VS Code extension runtime environment.

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
    *   **Host Build:** Calls `esbuild.build()` for `src/extension.ts` with production settings (minify, no sourcemap, `platform: 'node'`, `format: 'cjs'`, `external: ['vscode']`, `define` for env vars) outputting `dist/extension.js`.
    *   **WebView Build:** Executes `pnpm --filter webview-ui build` via `child_process.execSync`. Vite creates optimized assets in `webview-ui/build/`.
    *   **Asset Copying:** Uses script utilities (`copyRecursiveSync`, `fs.copyFileSync`, `ensureDirSync`) to copy `webview-ui/build/*` -> `dist/webview-ui/`, WASM files -> `dist/`, `public/locales/*` -> `dist/locales/`, `package.nls*.json` -> `dist/`.
5.  **Output:** `dist/` contains `extension.js`, `webview-ui/`, `locales/`, WASM files, NLS files.
6.  **Packaging:** Run `pnpm package`. `vsce` bundles `dist/` (respecting `.vscodeignore`) into `.vsix`.

## Key Concepts

1.  **`esbuild`:** Fast bundler for the **extension host** code (`src/` -> `dist/extension.js`). Configured in `esbuild.js`.
2.  **Vite:** Build tool for the **WebView UI** (`webview-ui/` -> `webview-ui/build/`). Configured in `webview-ui/vite.config.ts`. ([Chapter 1](01_webview_ui.md)).
3.  **Build Script (`esbuild.js`):** Node.js orchestrator using `esbuild` API, `child_process`, and `fs` utilities ([Chapter 55](55_script_utilities.md), [Chapter 42](42_file_system_utilities.md)). Handles modes, asset copying.
4.  **`package.json` Scripts:** Defines commands (`build:prod`, `watch:host`, `compile`, `vscode:prepublish`, `package`) using `pnpm`, `cross-env`, `node`, `tsc`, `rimraf`, `vsce`.
5.  **Entry Points & Outputs:** Host (`src/extension.ts` -> `dist/extension.js`), WebView (`webview-ui/src/index.tsx` -> `dist/webview-ui/assets/*`).
6.  **Host Configuration (`esbuild.js`):** `platform: 'node'`, `format: 'cjs'`, `target: 'node16'`, `external: ['vscode']`, `bundle: true`, `minify`, `sourcemap`, `define`.
7.  **Asset Handling:** Crucial step performed by `esbuild.js` or plugins. Copies WASM, locales, WebView build output, NLS files to `dist/`.
8.  **Development Mode (`watch:*`):** Separate watch processes: `esbuild --watch` for host, `vite dev` for WebView HMR.
9.  **Production Mode (`build:prod`):** Full sequence: clean, prepare, esbuild host, vite build webview, copy assets.
10. **Type Checking (`compile` script):** `tsc --noEmit` runs full TypeScript check.
11. **Packaging (`vsce`, `package` script):** `vsce package` bundles `dist/` based on `.vscodeignore`.
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
    "build:dev": "pnpm clean && pnpm prepare && cross-env NODE_ENV=development node ./esbuild.js", // Dev build (used by watch)
    "watch:host": "pnpm build:dev --watch", // Watch host code only
    "watch:webview": "pnpm --filter webview-ui dev", // Start Vite dev server
    "vscode:prepublish": "pnpm run compile && pnpm run build:prod", // Ensure compile + prod build before packaging
    "package": "vsce package --out ./releases/roo-code.vsix --yarn", // Create VSIX
    "publish": "vsce publish --yarn" // Publish to Marketplace
    // ... test, lint, etc.
  }
}
```

*   **Dev:** `pnpm install:all`, run `watch:host` & `watch:webview` concurrently, Debug (F5).
*   **Package:** `pnpm install:all`, `pnpm compile`, `pnpm build:prod`, `pnpm package`.

## Code Walkthrough

### Build Script (`esbuild.js`)

*(See refined code in Chapter 56 - Code Walkthrough)*

*   Uses `chalk`, `rimraf`, Node `fs`, `path`, `child_process`.
*   Detects `isWatch`, `isProduction`.
*   `copyAssets` function handles copying WASM, locales (WebView), NLS (Host), images to `dist/`.
*   `extensionConfig` defines `esbuild` options for the host build.
*   `build` function:
    1.  Cleans `dist` (`rimraf`).
    2.  Runs `esbuild.build` for host (with watch if `isWatch`).
    3.  Conditionally runs Vite build (`pnpm --filter webview-ui build`) via `execSync` if *not* host watch mode or if production.
    4.  Conditionally copies Vite output (`webview-ui/build/`) to `dist/webview-ui/`.
    5.  Calls `copyAssets`.

### Vite Configuration (`webview-ui/vite.config.ts`)

*(See code in Chapter 1)*
*   Configures Vite for React, Tailwind. Sets `build.outDir: "build"`.

### `.vscodeignore` (Conceptual)

*(See code in Chapter 56 - Key Concepts)*
*   Ignores source (`src/`, `webview-ui/src/`), `node_modules`, config files, tests, scripts, intermediate builds (`webview-ui/build`).
*   **Crucially does NOT ignore `dist/`** and ensures `package.json`, `README.md`, `LICENSE`, `CHANGELOG.md` are included.

## Internal Implementation

1.  **Trigger:** `pnpm build:prod` runs `esbuild.js` via Node.js.
2.  **Cleanup:** Script deletes `dist/`.
3.  **Host Build:** `esbuild.build()` bundles `src/` into `dist/extension.js`.
4.  **WebView Build:** `execSync` runs `vite build`. Vite bundles `webview-ui/` into `webview-ui/build/`.
5.  **Asset Copying:** Script copies `webview-ui/build/*` to `dist/webview-ui/`, WASM to `dist/`, locales to `dist/locales/`, NLS to `dist/`.
6.  **Packaging:** `vsce package` reads `package.json`, uses `.vscodeignore` to zip required files from `dist/` and root into `.vsix`.

## Modification Guidance

Modifications typically involve adding entry points, managing new assets, or adjusting build flags.

1.  **Adding a New Entry Point (e.g., Language Server):**
    *   **`esbuild.js`:** Define `languageServerConfig`, add `await esbuild.build(languageServerConfig)` call.
    *   **`package.json`:** Update if needed.

2.  **Adding a New Asset Type (e.g., Fonts):**
    *   **`esbuild.js`:** Modify `copyAssets` to copy source fonts to `dist/`.
    *   **Code:** Update references to use correct relative path within `dist/`.
    *   **`.vscodeignore`:** Ensure fonts in `dist/` are *not* ignored.

3.  **Changing esbuild Options (e.g., Target Node Version):**
    *   **`esbuild.js`:** Modify `target` in `extensionConfig`. Check VS Code compatibility.

4.  **Adding a Build-time Code Generation Step:**
    *   **Create Script:** Write a Node.js script (e.g., `scripts/generateTypes.js`) that performs the generation (e.g., Zod to TypeScript interfaces).
    *   **`package.json`:** Add the script execution to the `prepare` script: `"prepare": "node ./scripts/prepare.js && node ./scripts/generateTypes.js"`. This ensures it runs before `build:prod` or `build:dev`.

**Best Practices:**

*   **Use Correct Tools:** `esbuild` (host), Vite (WebView).
*   **Clean Output (`dist/`):** Keep structured. Clean before builds (`rimraf`).
*   **Separate Scripts:** Use `package.json` for orchestration.
*   **Environment Variables:** Use `NODE_ENV`, `define`. Handle secrets via build env vars.
*   **`external: ['vscode']`:** Essential for host.
*   **Asset Management:** Explicitly copy *all* runtime assets to `dist/`.
*   **`.vscodeignore`:** Maintain carefully.
*   **Type Checking:** Run `tsc --noEmit` separately (`compile` script).

**Potential Pitfalls:**

*   **Missing Assets in `dist/`:** Runtime errors. Check `copyAssets`.
*   **Incorrect `external`/`platform`.**
*   **Asset Path Issues:** Incorrect relative paths in code referencing assets in `dist/`.
*   **`.vscodeignore` Errors:** Ignoring needed files or including source/dev files.
*   **Build Environment Differences (CI vs. Local).**
*   **Vite/Esbuild Conflicts/Orchestration:** Ensure sequential execution in production build. Keep watch modes separate.

## Conclusion

The Build System, orchestrated via `esbuild.js` and `package.json` scripts, reliably transforms Roo-Code's source code into a functional and distributable VS Code extension. It leverages the speed of `esbuild` for the extension host and the rich features of Vite for the WebView UI. Key aspects include distinct configurations for host vs. web, handling development vs. production builds, injecting environment variables, and crucially, ensuring all necessary runtime assets (WASM, locales, WebView bundles, NLS files) are correctly copied to the `dist/` directory, ready for packaging with `vsce`. This automated system is fundamental for both efficient development and creating optimized, distributable extension builds.

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
3.  **Testing Environments (`jest.config.js` `projects`):**
    *   **Extension Host (`displayName: "Host"`, `testEnvironment: 'node'`):** For `src/` code. Mocks `vscode` API.
    *   **WebView UI (`displayName: "WebView"`, `testEnvironment: 'jsdom'`):** For `webview-ui/src/` code. Mocks `vscode` toolkit, `postMessage`.
4.  **Mocking Dependencies:** Crucial for isolation.
    *   **`vscode` API:** Mocked via `src/__mocks__/vscode.ts`.
    *   **`@vscode/webview-ui-toolkit/react`:** Mocked via `src/__mocks__/@vscode/webview-ui-toolkit/react.ts`.
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
    // ...
  }
}
```

## Code Walkthrough

### Jest Configuration (`jest.config.js`)

*(See code in Chapter 57 - Key Concepts)*
*   Defines "Host" (Node) and "WebView" (JSDOM) projects. Configures `ts-jest`, `moduleNameMapper`, `testEnvironment`, `roots` (for mocks), `collectCoverageFrom`.

### Mocking `vscode` (`src/__mocks__/vscode.ts`)

*(See code in Chapter 57 - Key Concepts)*
*   Provides `jest.fn()` mocks for APIs (`window.*`, `workspace.*`, `commands.*`, `env.*`, `authentication.*`), mock properties, classes (`Uri`, `Range`), and enums.

### Unit Test Example (Host Utility - `src/utils/path.test.ts`)

*(See code in Chapter 57 - Key Concepts)*
*   Uses `jest.mock('vscode', ...)` to control `workspace.workspaceFolders`. Uses `describe`/`it`, `beforeEach`, `expect`. Switches `process.platform` for platform-specific tests.

### React Component Test (WebView UI - `webview-ui/src/components/settings/About.test.tsx`)

*(See code in Chapter 57 - Key Concepts)*
*   Uses `@testing-library/react`. Mocks dependencies (`@/utils/vscode`, `@/i18n/TranslationContext`, toolkit components). Uses `screen`, `fireEvent`, `expect`. Asserts on DOM state and mock calls (`postMessage`, prop functions).

## Internal Implementation

1.  **Execution:** `pnpm test` runs `jest`.
2.  **Config/Projects:** Jest reads config, runs tests for each project.
3.  **Environment:** Sets up `node` or `jsdom`.
4.  **Mocking:** Intercepts imports, uses mocks from `__mocks__` or `jest.mock()`. Maps path aliases.
5.  **Test Run:** Executes test files (`describe`/`it`).
6.  **Rendering (UI):** RTL renders components into JSDOM using mocks.
7.  **Interaction/Assertion:** Tests simulate events, assert against JSDOM or mocks.
8.  **Coverage:** Instrumentation tracks execution paths.

## Modification Guidance

Involves adding/updating tests or mocks.

1.  **Adding Tests:** Create `*.test.ts(x)` files alongside source or in `__tests__`. Import, use `describe`/`it`, mock dependencies, use `expect`. Choose correct project (`test:host` or `test:ui`).
2.  **Updating `vscode` Mock:** Add `jest.fn()` or mock values to `src/__mocks__/vscode.ts` for newly used APIs. Use `mockReturnValue` etc. within tests for specific behavior.
3.  **Mocking New Modules:** Use `jest.mock('../path/to/module', factory)` at the top of test files.

**Best Practices:**

*   **Unit Focus:** Prioritize unit tests with effective mocking.
*   **Test Behavior:** Use RTL for UI tests focusing on user perspective.
*   **Mock Boundaries:** Mock external dependencies and logical internal boundaries.
*   **Coverage as Guide:** Use reports to find gaps, but prioritize testing critical/complex logic and edge cases.
*   **Maintainability:** Write clear, readable tests. Reset state (`clearMocks: true`).
*   **CI Integration:** Run `pnpm test` in CI pipeline.

**Potential Pitfalls:**

*   **Incomplete/Incorrect Mocks:** False positives/negatives. Requires maintenance.
*   **Over-Mocking:** Brittle tests tied to implementation.
*   **Testing Implementation:** Querying by CSS class, testing internal state.
*   **Environment Mismatch:** Bugs only in real VS Code. Requires some manual/E2E testing.
*   **Flaky Tests:** Async/timing issues, state leakage. Use `waitFor`, reset state.
*   **Slow Tests:** Complex setup, too many integration tests.

## Conclusion

The Testing Framework, primarily utilizing Jest and React Testing Library, is crucial for maintaining the quality and stability of the Roo-Code extension. By providing separate testing environments and configurations for the extension host (Node) and WebView UI (JSDOM), and leveraging comprehensive mocking for dependencies like the `vscode` API and UI toolkits, the framework enables automated verification of individual units and components. This automated testing catches regressions early, facilitates safer refactoring, and ultimately contributes to a more reliable and robust product for users. Running tests frequently, especially in CI pipelines, is a key practice for sustainable development.

Finally, we'll look at the documentation and community-related files that support the project: [Chapter 58: Documentation & Community Files](58_documentation___community_files.md).
---
# Chapter 58: Documentation & Community Files

Continuing from [Chapter 57: Testing Framework](57_testing_framework.md), which detailed how Roo-Code ensures code quality through testing, this final chapter looks at the essential supporting files that explain the project, guide contributors, and foster a healthy community: **Documentation & Community Files**.

## Motivation: Guiding Users and Contributors

A software project, especially an open-source one like Roo-Code, needs more than just functional code. To be successful, usable, and sustainable, it requires clear documentation and standard community health files to:

1.  **Explain Usage:** Guide end-users on installation, configuration, and features.
2.  **Onboard Developers:** Provide instructions for setup, architecture understanding (via this tutorial!), build ([Chapter 56](56_build_system.md)), test ([Chapter 57](57_testing_framework.md)), and contribution guidelines.
3.  **Document Architecture:** Explain design choices, components, and workflows.
4.  **Set Expectations:** Define codes of conduct, contribution processes, security policies.
5.  **Facilitate Collaboration:** Standardize bug reports, feature requests, PRs.
6.  **Meet Marketplace Requirements:** Provide `README`, `LICENSE`, `CHANGELOG`.
7.  **Define Project Standards:** Document coding/quality rules (`.roo/rules/rules.md`) and localization guidelines (`.roo/rules-translate/`).

These files, typically in the project root, `docs/`, `.github/`, or `.roo/`, are crucial for usability, maintainability, and community engagement.

## Key Concepts & Files

1.  **`README.md`:** Primary project entry point. Overview, features, install, quick start, links. Audience: Users & Contributors.
2.  **`LICENSE`:** Full open-source license text (e.g., MIT). Defines legal rights. Audience: Users, Contributors, Legal.
3.  **`CHANGELOG.md`:** Records changes per version (following Keep a Changelog format). Audience: Users.
4.  **`CONTRIBUTING.md`:** Guidelines for contributors (setup, build, test, code style, PR process). Audience: Contributors.
5.  **`CODE_OF_CONDUCT.md`:** Community behavior standards (often Contributor Covenant). Audience: All Members.
6.  **`SECURITY.md`:** Process for reporting security vulnerabilities responsibly. Audience: Security Researchers, Users.
7.  **`.github/` Directory:**
    *   **`ISSUE_TEMPLATE/`:** Markdown templates for bug reports, feature requests.
    *   **`PULL_REQUEST_TEMPLATE.md`:** Template for PR descriptions (checklist).
    *   **`workflows/`:** GitHub Actions YAML files for CI/CD (lint, test, build checks).
8.  **`docs/` Directory:** Detailed documentation: User Guides, Config Guides, **Architecture Docs (This Tutorial Series: `docs/architecture/*.md`)**.
9.  **`.roo/` Directory:** Project-specific rules:
    *   **`rules/rules.md`:** Code quality, architecture, testing standards ([Chapter 48](48_configuration_rules.md)). For Devs & AI Assistants.
    *   **`rules-translate/`:** Localization guidelines ([Chapter 50](50_localization_system__i18n_.md)). General (`001-general...`) and language-specific (`instructions-<lang>.md`) rules, glossaries. For Translators & AI Assistants.

## Using These Files

*   **Users:** `README.md`, `docs/`, `CHANGELOG.md`, `ISSUE_TEMPLATE`s.
*   **Contributors:** `README.md`, `CONTRIBUTING.md`, `docs/architecture/`, `CODE_OF_CONDUCT.md`, `.roo/rules/rules.md`, GitHub Templates, CI Workflows.
*   **Translators:** `.roo/rules-translate/` files, `CONTRIBUTING.md` (for workflow).
*   **Maintainers:** All files for management, reviews, community standards.
*   **AI Assistants:** `.roo/rules/` files as context for code generation/review/translation tasks.
*   **VS Code Marketplace:** Uses `README.md`, `CHANGELOG.md`, `LICENSE`.

## Code Walkthrough

These files are primarily Markdown, YAML, or JSON.

### `.roo/rules/rules.md` (Example Structure)

*(See code in Chapter 48)*
*   Defines project standards for testing (coverage goals), TypeScript usage (strict, avoid `any`), UI styling (themed Tailwind), state management conventions, error handling, dependencies. References relevant tutorial chapters.

### `.roo/rules-translate/` Files (Example Structure)

*(See code in Chapter 50)*
*   **`001-general-rules.md`:** General tone (informal), non-translatable terms, placeholder syntax, `Trans` component usage, QA script (`find-missing-translations.js`).
*   **`instructions-de.md`:** Mandates "du" form.
*   **`instructions-zh-cn.md`:** Detailed glossary, formatting rules.
*   **`instructions-zh-tw.md`:** Terminology differences, punctuation rules.

### GitHub Workflow Example (`.github/workflows/ci.yaml`)

*(See code in Chapter 57)*
*   Runs on push/PR to `main`. Sets up Node/pnpm. Installs deps. Runs `lint`, `compile` (type check), `test`, `build:prod`. Uses dummy env vars for build secrets if needed.

## Internal Implementation

Static documentation (Markdown) or configuration files (YAML, JSON) used by humans or external tools (GitHub, VS Code, Build/Test runners).

## Modification Guidance

Involves updating content for accuracy and clarity.

1.  **Updating `README.md` / `docs/`:** Add new features, update installation/usage, fix links.
2.  **Updating `CHANGELOG.md`:** Add entries for new releases.
3.  **Modifying `CONTRIBUTING.md`:** Update setup/build/test instructions. Refine workflow.
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

Documentation and Community Files, including project-specific rules like `rules.md` and `rules-translate/`, are vital supporting elements for the Roo-Code project. A clear `README.md` introduces the extension, `CONTRIBUTING.md` guides developers, `CHANGELOG.md` tracks progress, `LICENSE` defines legal terms, and other files like `CODE_OF_CONDUCT.md` and `SECURITY.md` foster a healthy community. GitHub-specific files streamline issue reporting, contributions, and CI/CD automation. Detailed architecture docs (like this tutorial) and specific rule files (`.roo/`) ensure consistency and quality. Maintaining these resources accurately is crucial for the project's success, usability, and collaborative potential.

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

