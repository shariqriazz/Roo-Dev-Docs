# Chapter 8: Tools

In [Chapter 7: SystemPrompt](07_systemprompt.md), we learned how Roo-Code constructs detailed instructions to guide the Language Model (LLM), including defining the actions it can perform. This chapter focuses specifically on those actions: the **Tools**.

## Motivation: Empowering the AI to Act

An LLM's primary function is text generation. However, to be a truly effective coding assistant, the AI needs to interact with the user's development environment. It needs the ability to read and write files, execute terminal commands, apply code changes, browse the web for information, interact with external services, or even ask clarifying questions. Simply generating text describing these actions isn't enough; the AI needs a mechanism to *request* that these actions be performed.

**Tools** are this mechanism. They represent the concrete actions the AI agent can request Roo-Code to execute within the user's environment.

**Central Use Case:** A user asks Roo-Code to "Read the `package.json` file and tell me the current version of React."
1.  The LLM, guided by its [Chapter 7: SystemPrompt](07_systemprompt.md), understands it needs to read a file.
2.  It generates a response containing an XML instruction like `<read_file><path>package.json</path></read_file>`.
3.  Roo-Code parses this instruction and recognizes it as a request to use the `read_file` tool.
4.  Roo-Code validates if the `read_file` tool is allowed in the current mode.
5.  It asks the user for permission ("Roo wants to read `package.json`. Allow?").
6.  If approved, Roo-Code executes the underlying logic for the `read_file` tool (reading the file content from the workspace).
7.  Roo-Code formats the file content (or an error message) as a tool result.
8.  This result is sent back to the LLM in the next turn of the conversation.
9.  The LLM can now process the file content and answer the user's original question about the React version.

Without the `read_file` tool, the LLM would be unable to fulfill this simple request. Tools bridge the gap between the AI's text-based reasoning and the practical actions required within a developer's workflow.

## Key Concepts

1.  **Action Representation:** Tools represent specific, executable actions like `read_file`, `write_to_file`, `execute_command`, `apply_diff`, `browser_action`, `use_mcp_tool`, `ask_followup_question`, `attempt_completion`, etc.

2.  **Declaration (System Prompt):** Tools are declared to the LLM within the [Chapter 7: SystemPrompt](07_systemprompt.md) using a specific XML format. Each declaration includes:
    *   `<tool_name>`: The unique identifier for the tool.
    *   `<summary>`: A brief description of what the tool does.
    *   `<parameters>`: Definitions of the required and optional parameters the tool accepts (e.g., `<path>`, `<command>`, `<diff>`).
    *   `<usage>`: An example of how the LLM should format its request to use the tool.
    Tool descriptions are dynamically generated by `getToolDescriptionsForMode` ([Chapter 7: SystemPrompt](07_systemprompt.md)) based on the allowed tools for the current `Mode`. Find tool description generation logic in `src/core/prompts/tools/`.

3.  **Implementation (Tool Functions):** The actual logic for each tool is implemented as an `async` function within the `src/core/tools/` directory (e.g., `readFileTool.ts`, `executeCommandTool.ts`). These functions adhere to a standard signature:
    ```typescript
    async function toolNameTool(
        cline: Cline, // The current Cline instance
        block: ToolUse, // Parsed tool parameters from AI
        askApproval: AskApproval, // Function to ask user approval
        handleError: HandleError, // Function to handle unexpected errors
        pushToolResult: PushToolResult, // Function to send result back to AI
        removeClosingTag: RemoveClosingTag, // Utility for partial parsing
        // ... potentially other arguments like toolDescription, askFinishSubTaskApproval
    ): Promise<void>;
    ```

4.  **Parsing (AI Response):** When the LLM wants to use a tool, it includes the corresponding XML block in its response. The `parseAssistantMessage` function ([Chapter 25: Assistant Message Parsing](25_assistant_message_parsing.md)) identifies these blocks and converts them into structured `ToolUse` objects (defined in `src/shared/tools.ts`).

5.  **Orchestration (`Cline.presentAssistantMessage`):** The `presentAssistantMessage` method within the [Chapter 4: Cline](04_cline.md) class is the central orchestrator for tool execution. When it encounters a `ToolUse` block during stream processing:
    *   It identifies the `toolName`.
    *   It calls `validateToolUse` to ensure the tool is permitted in the current mode.
    *   It typically calls `askApproval` to get explicit user consent via the UI ([Chapter 1: WebView UI](01_webview_ui.md), [Chapter 39: Human Relay UI](39_human_relay_ui.md)). Certain global settings (`alwaysAllow...`) can bypass this step for specific tool types.
    *   If validation and approval succeed, it calls the corresponding tool implementation function from `src/core/tools/`.
    *   It passes necessary callbacks (`askApproval`, `handleError`, `pushToolResult`) to the tool function.

6.  **Validation (`validateToolUse`):** Located in `src/core/mode-validator.ts`, this function checks if a given `toolName` is allowed for the current `mode`. It relies on `isToolAllowedForMode` (from `src/shared/modes.ts`), which consults the mode's configuration (tool `groups`, specific `tools` overrides) defined by the [Chapter 10: CustomModesManager](10_custommodesmanager.md) or built-in mode definitions. If the tool isn't allowed, an error is thrown, preventing execution and informing the AI.

7.  **Approval (`askApproval`):** This callback, passed to tool functions, internally calls `Cline.ask('tool', ...)`. This triggers a message to the WebView UI, presenting the intended tool action (often with parameters like filename or command) and prompting the user for confirmation (Yes/No buttons). The tool's execution pauses until the user responds.

8.  **Execution (Tool Functions):** The specific tool function (e.g., `readFileTool`) performs the action. This might involve:
    *   Reading/validating parameters from the `block` argument.
    *   Interacting with VS Code APIs (e.g., workspace filesystem via `src/utils/fs.ts` ([Chapter 42: File System Utilities](42_file_system_utilities.md))).
    *   Running external processes (e.g., `executeCommandTool` using [Chapter 15: Terminal Integration](15_terminal_integration.md)).
    *   Interacting with the browser ([Chapter 18: Browser Interaction](18_browser_interaction.md)).
    *   Communicating with MCP servers ([Chapter 19: McpHub / McpServerManager](19_mcphub___mcpservermanager_.md)).
    *   Showing diff views ([Chapter 20: DiffViewProvider](20_diffviewprovider.md)).

9.  **Result Handling (`pushToolResult`):** Once the tool function completes its action (or encounters an expected error like "file not found"), it calls the `pushToolResult` callback. This function formats the result (success message, file content, command output, error description) into a standard XML structure (e.g., `<read_file><path>...</path><content>...</content></read_file>`). This formatted result is then added to the `userMessageContent` within the `Cline` instance, ready to be sent back to the LLM in the next turn of the agentic loop.

10. **Error Handling (`handleError`):** If an *unexpected* error occurs during tool execution (e.g., a filesystem permission error, network issue), the tool function should call the `handleError` callback. This typically logs the error and might push a generic error message back to the AI using `pushToolResult`.

11. **Mode Dependency:** Tool availability is a core aspect of `Modes`. Different modes expose different sets of tools, tailoring the AI's capabilities to the specific task context (e.g., a "Chat" mode might disable file writing, while a "Code" mode enables it).

12. **Shared Types (`src/shared/tools.ts`):** This file centralizes key types related to tools, including:
    *   `ToolName`: Union of all valid tool names.
    *   `ToolUse`: Interface representing the parsed tool request from the AI.
    *   `ToolGroup`: Categories used by modes to enable sets of tools.
    *   `TOOL_GROUPS`: Mapping of group names to the tools they contain.
    *   `ALWAYS_AVAILABLE_TOOLS`: Tools enabled regardless of mode config.
    *   Callback types: `AskApproval`, `HandleError`, `PushToolResult`, `RemoveClosingTag`.
    *   `DiffStrategy`: Interface for applying diffs, relevant for `apply_diff`.

## Using Tools: The File Reading Example

Let's trace the central use case ("Read `package.json`...") focusing on the tool interaction:

1.  **AI Decision:** LLM processes the user request and its System Prompt (which includes the `<read_file>` tool description). It decides to use the tool.
2.  **AI Response:** LLM generates output including: `<read_file><path>package.json</path></read_file>`
3.  **Parsing (`Cline`):** The `Cline` instance receives the stream. `parseAssistantMessage` ([Chapter 25: Assistant Message Parsing](25_assistant_message_parsing.md)) identifies this block and creates a `ToolUse` object: `{ type: 'tool_use', name: 'read_file', params: { path: 'package.json' }, partial: false }`.
4.  **Orchestration (`Cline.presentAssistantMessage`):**
    *   The method receives the `ToolUse` block.
    *   It enters the `switch (block.name)` statement and hits the `case 'read_file':` block.
5.  **Validation:** It calls `validateToolUse('read_file', this.mode, ...)` using the function from `src/core/mode-validator.ts`. Assuming `read_file` is allowed in the current mode, this passes.
6.  **Approval:** It prepares the `askApproval` callback. The `case 'read_file':` block will eventually call it implicitly or explicitly within the `readFileTool` function. Let's assume `readFileTool` calls `askApproval('tool', JSON.stringify({ tool: 'readFile', path: 'package.json', ... }))`.
    *   `askApproval` calls `this.ask('tool', ...)`.
    *   `Cline` emits a `message` event with the `ask` payload.
    *   [Chapter 2: ClineProvider](02_clineprovider.md) sends an `ExtensionMessage` to the [Chapter 1: WebView UI](01_webview_ui.md).
    *   The UI displays the approval request: "Roo wants to read package.json. Allow? [Yes] [No]".
    *   User clicks "Yes". UI sends a `WebviewMessage` (`askResponse`).
    *   `ClineProvider` receives it and calls `cline.handleWebviewAskResponse('yesButtonClicked')`.
    *   The `pWaitFor` inside `Cline.ask` resolves, and thus the `askApproval` promise resolves successfully.
7.  **Execution:** `presentAssistantMessage` calls `readFileTool(this, block, askApproval, handleError, pushToolResult, removeClosingTag)`.
8.  **Tool Logic (`readFileTool`):**
    *   Validates the `path` parameter.
    *   Constructs the absolute path: `/workspace/package.json`.
    *   Uses filesystem utilities (e.g., `fs.readFile` via helpers in `src/utils/fs.ts` or `src/integrations/misc/extract-text.ts`) to read the content.
    *   Handles potential file reading errors.
    *   Formats the content (potentially adding line numbers, handling truncation based on settings).
    *   Calls `pushToolResult(`<file><path>package.json</path>\n<content lines="1-50">{\n  "name": "roo-code",\n  "version": "1.0.0",\n...\n}</content>\n</file>`);`.
9.  **Result Handling (`pushToolResult`):**
    *   This callback adds the formatted XML result string to `cline.userMessageContent`.
    *   It sets `cline.didAlreadyUseTool = true`.
    *   It eventually sets `cline.userMessageContentReady = true`.
10. **Loop Continuation:** The `pWaitFor(() => this.userMessageContentReady)` in `recursivelyMakeClineRequests` resolves. The loop prepares the next API call, including the `<file>...</file>` result from the tool in the 'user' message content sent back to the LLM.
11. **AI Answers:** The LLM receives the file content and can now extract the React version to answer the user's original question.

## Code Walkthrough

### Shared Types (`src/shared/tools.ts`)

```typescript
// --- File: src/shared/tools.ts ---
import { Anthropic } from "@anthropic-ai/sdk"
import { ClineAsk, ToolProgressStatus, ToolGroup, ToolName } from "../schemas" // Assumes schemas define these base types

// Type for the result a tool function should provide via pushToolResult
export type ToolResponse = string | Array<Anthropic.TextBlockParam | Anthropic.ImageBlockParam>

// Callback type for requesting user approval
export type AskApproval = (
	type: ClineAsk, // Type of ask (e.g., 'tool', 'command', 'browser_action_launch')
	partialMessage?: string, // JSON stringified payload with tool details
	progressStatus?: ToolProgressStatus // Optional progress info for streaming tools
) => Promise<boolean> // Resolves true if approved, false otherwise

// Callback type for handling unexpected errors
export type HandleError = (action: string, error: Error) => Promise<void>

// Callback type for pushing the tool's result back to the conversation history
export type PushToolResult = (content: ToolResponse) => void

// Callback type for stripping the closing XML tag (used during partial parsing)
export type RemoveClosingTag = (tag: ToolParamName, content?: string) => string

// Callback type for asking approval to finish a sub-task (used by attempt_completion)
export type AskFinishSubTaskApproval = () => Promise<boolean>

// Callback type for getting the tool's description string
export type ToolDescription = () => string

// Represents the parsed structure of a tool usage request from the AI
export interface ToolUse {
	type: "tool_use"
	name: ToolName // The name of the tool being requested
	// Parsed parameters provided by the AI
	params: Partial<Record<ToolParamName, string>>
	partial: boolean // True if this block is from a partial stream chunk
}

// List of all possible parameter names used across different tools
export const toolParamNames = [
	"command", "path", "content", "line_count", "regex", "file_pattern", "recursive",
	"action", "url", "coordinate", "text", "server_name", "tool_name", "arguments",
	"uri", "question", "result", "diff", "start_line", "end_line", "mode_slug",
	"reason", "operations", "line", "mode", "message", "cwd", "follow_up", "task",
	"size", "search", "replace", "use_regex", "ignore_case",
] as const
export type ToolParamName = (typeof toolParamNames)[number]

// Maps group names to the tools they contain
export const TOOL_GROUPS: Record<ToolGroup, { tools: readonly string[], alwaysAvailable?: boolean }> = {
	read: { tools: ["read_file", /* ... */] },
	edit: { tools: ["apply_diff", "write_to_file", /* ... */] },
	browser: { tools: ["browser_action"] },
	command: { tools: ["execute_command"] },
	mcp: { tools: ["use_mcp_tool", "access_mcp_resource"] },
	modes: { tools: ["switch_mode", "new_task"], alwaysAvailable: true },
}

// Tools available regardless of mode configuration
export const ALWAYS_AVAILABLE_TOOLS: ToolName[] = [
	"ask_followup_question", "attempt_completion", "switch_mode", "new_task",
] as const

// Interface for diff strategies (used by apply_diff tool)
export interface DiffStrategy {
	getName(): string
	getToolDescription(args: { cwd: string; toolOptions?: { [key: string]: string } }): string
	applyDiff(originalContent: string, diffContent: string, startLine?: number, endLine?: number): Promise<DiffResult>
	// ... potentially other methods like getProgressStatus ...
}

export type DiffResult = /* ... definition ... */;
```

**Explanation:**

*   Defines the core `ToolUse` interface representing a parsed tool request.
*   Defines the signatures for the essential callback functions (`AskApproval`, `HandleError`, `PushToolResult`, etc.) that orchestrate the tool lifecycle.
*   Lists all possible parameter names (`ToolParamName`).
*   Defines the `TOOL_GROUPS` mapping used by modes to enable sets of tools.
*   Lists `ALWAYS_AVAILABLE_TOOLS`.
*   Includes the `DiffStrategy` interface, relevant for the `apply_diff` tool.

### Tool Orchestration (`src/core/Cline.ts` - `presentAssistantMessage`)

```typescript
// --- File: src/core/Cline.ts ---
// (Simplified excerpt focusing on tool handling within presentAssistantMessage)

async presentAssistantMessage() {
    // ... loop processing content blocks (this.assistantMessageContent) ...
    const block = this.assistantMessageContent[this.currentStreamingContentIndex];

    if (block.type === "tool_use") {
        // Check if tool interaction already happened or was rejected
        if (this.didRejectTool || this.didAlreadyUseTool) {
            this.userMessageContentReady = true; // Signal loop can continue
            return;
        }

        // --- Tool Handling Logic ---
        const toolName = block.name as ToolName;
        this.activeTool = toolName; // Mark the tool as active

        // Define helper callbacks for the tool function
        const askApproval: AskApproval = async (type, partialMessage, progressStatus) => {
            const { response } = await this.ask(type, partialMessage, block.partial, progressStatus);
            const approved = response === "yesButtonClicked";
            if (!approved) this.didRejectTool = true;
            return approved;
        };
        const handleError: HandleError = async (action, error) => {
             // Log error, potentially inform user via say('error', ...)
             console.error(`Error ${action}:`, error);
             // Push a generic error result back to AI
             this.pushToolResult(formatResponse.toolError(`An unexpected error occurred while ${action}.`));
        };
        const pushToolResult: PushToolResult = (content) => {
             // Format result and add to userMessageContent for next AI turn
             this.pushToolResult(toolName, content);
        };
        const removeClosingTag: RemoveClosingTag = (tag, content) => { /* ... utility ... */ };
        const toolDescription = () => { /* ... get tool description string ... */ };
        const askFinishSubTaskApproval: AskFinishSubTaskApproval = async () => { /* ... logic ... */ };

        try {
            // 1. Validate if tool is allowed in current mode
            validateToolUse(toolName, this.mode, this.customModes, undefined, block.params);

            // 2. Execute the specific tool function based on name
            switch (toolName) {
                case "read_file":
                    await readFileTool(this, block, askApproval, handleError, pushToolResult, removeClosingTag);
                    break;
                case "write_to_file":
                    await writeToFileTool(this, block, askApproval, handleError, pushToolResult, removeClosingTag);
                    break;
                case "execute_command":
                    await executeCommandTool(this, block, askApproval, handleError, pushToolResult, removeClosingTag);
                    break;
                case "apply_diff":
                     await applyDiffTool(this, block, askApproval, handleError, pushToolResult, removeClosingTag);
                     break;
                // ... cases for ALL other tools ...
                case "attempt_completion":
                     await attemptCompletionTool(this, block, askApproval, handleError, pushToolResult, removeClosingTag, toolDescription, askFinishSubTaskApproval);
                     break;
                default:
                    console.warn(`Unknown tool requested: ${toolName}`);
                    pushToolResult(formatResponse.toolError(`Unknown tool: ${toolName}`));
            }
        } catch (error) { // Catch errors from validateToolUse or tool function itself
            console.error(`Tool execution failed for ${toolName}:`, error);
            this.recordToolError(toolName, error.message);
            // Inform user and AI about the validation/execution error
            await this.say("error", error.message);
            pushToolResult(formatResponse.toolError(error.message));
        } finally {
            this.activeTool = undefined; // Mark tool as inactive
            // Signal that tool interaction is done (or skipped) for this turn
            // Crucially allows the pWaitFor in recursivelyMakeClineRequests to resolve
            this.userMessageContentReady = true;
        }
    }
    // ... handle 'text' blocks ...

    // Move to next content block if stream isn't finished
    if (!this.didCompleteReadingStream && !block.partial) {
        this.currentStreamingContentIndex++;
        if (this.currentStreamingContentIndex < this.assistantMessageContent.length) {
            await this.presentAssistantMessage(); // Recurse for next block
        }
    }
}

// Helper to add tool result to userMessageContent
private pushToolResult(toolName: ToolName, content: ToolResponse) {
    const toolDescription = () => { /* ... get tool description string ... */ };
    const textResult = `${toolDescription()} Result:\n`;

    this.userMessageContent.push({ type: "text", text: textResult });
    if (typeof content === "string") {
        this.userMessageContent.push({ type: "text", text: content });
    } else if (Array.isArray(content)) {
        this.userMessageContent.push(...content);
    }
    this.didAlreadyUseTool = true; // Mark that a tool was used this turn
    this.recordToolUsage(toolName); // Record usage for analytics/history
}
```

**Explanation:**

*   **Entry Point:** This method handles `ToolUse` blocks parsed from the AI's response stream.
*   **State Checks:** It first checks `didRejectTool` and `didAlreadyUseTool` to prevent redundant tool execution within a single AI turn.
*   **Callback Definition:** It defines the standard callback functions (`askApproval`, `handleError`, `pushToolResult`, etc.) that will be passed to the specific tool implementation. `askApproval` wraps `Cline.ask` and sets `didRejectTool` if the user denies. `pushToolResult` calls the internal `this.pushToolResult` helper.
*   **Validation:** It calls `validateToolUse` to check mode restrictions.
*   **Execution Switch:** A `switch` statement routes execution to the correct tool function (e.g., `readFileTool`, `writeToFileTool`) based on `block.name`.
*   **Error Handling:** A `try...catch` block surrounds the validation and execution call to handle errors gracefully, informing both the user and the AI.
*   **State Update (`finally`):** Critically, the `finally` block sets `this.userMessageContentReady = true`. This unblocks the main agentic loop (`recursivelyMakeClineRequests`) which waits on this flag after the stream processing finishes, ensuring the loop only continues after tool execution (or failure/rejection) is complete.
*   **`pushToolResult` Helper:** This internal method formats the tool's result (provided by the tool function via the callback) and adds it to `this.userMessageContent` for the next AI turn. It also sets `didAlreadyUseTool`.

### Example Tool Implementation (`src/core/tools/listFilesTool.ts`)

```typescript
// --- File: src/core/tools/listFilesTool.ts ---
import * as path from "path"
import { Cline } from "../Cline"
import { ClineSayTool } from "../../shared/ExtensionMessage"
import { formatResponse } from "../prompts/responses"
import { listFiles } from "../../services/glob/list-files" // Uses glob library
import { getReadablePath } from "../../utils/path"
import { ToolUse, AskApproval, HandleError, PushToolResult, RemoveClosingTag } from "../../shared/tools"

export async function listFilesTool(
	cline: Cline,
	block: ToolUse,
	askApproval: AskApproval,
	handleError: HandleError,
	pushToolResult: PushToolResult,
	removeClosingTag: RemoveClosingTag,
) {
	const relDirPath: string | undefined = block.params.path; // Get path param
	const recursiveRaw: string | undefined = block.params.recursive; // Get recursive param
	const recursive = recursiveRaw?.toLowerCase() === "true";

	// Prepare data for UI approval message
	const sharedMessageProps: ClineSayTool = {
		tool: !recursive ? "listFilesTopLevel" : "listFilesRecursive",
		path: getReadablePath(cline.cwd, removeClosingTag("path", relDirPath)),
	};

	try {
		// Handle partial stream case (update UI but don't execute yet)
		if (block.partial) {
			const partialMessage = JSON.stringify({ ...sharedMessageProps, content: "" });
			await cline.ask("tool", partialMessage, block.partial).catch(() => {});
			return;
		}

		// === Execution logic (only runs when block.partial is false) ===

		// 1. Validate parameters
		if (!relDirPath) {
			cline.consecutiveMistakeCount++;
			cline.recordToolError("list_files");
			// Use helper to generate standard error and push it as result
			pushToolResult(await cline.sayAndCreateMissingParamError("list_files", "path"));
			return;
		}

		cline.consecutiveMistakeCount = 0; // Reset counter on valid request

		// 2. Ask for user approval
		const absolutePath = path.resolve(cline.cwd, relDirPath);
		// Generate dummy result for approval message content (actual listing done after approval)
		const dummyResult = `Listing files in ${absolutePath}...`;
		const completeMessage = JSON.stringify({ ...sharedMessageProps, content: dummyResult });
		const didApprove = await askApproval("tool", completeMessage); // Pauses here

		if (!didApprove) {
			// User rejected, push rejection message and return
            pushToolResult(formatResponse.toolError("User rejected the request to list files."));
			return; // Exits the tool function
		}

		// 3. Perform the action (User approved)
		// Call the service that uses 'glob' to list files, respecting .rooignore
		const [files, didHitLimit] = await listFiles(absolutePath, recursive, 200, cline.rooIgnoreController);
        const { showRooIgnoredFiles = true } = (await cline.providerRef.deref()?.getState()) ?? {};

        // 4. Format the result
		const result = formatResponse.formatFilesList(
			absolutePath, files, didHitLimit, cline.rooIgnoreController, showRooIgnoredFiles
		);

		// 5. Push the final result
		pushToolResult(result);

	} catch (error) {
		// 6. Handle unexpected errors
		await handleError("listing files", error);
	}
}
```

**Explanation:**

*   **Standard Signature:** Follows the required async function signature, accepting `cline`, `block`, and the standard callbacks.
*   **Parameter Extraction:** Retrieves `path` and `recursive` parameters from `block.params`.
*   **Partial Handling:** If `block.partial` is true (meaning the tool request is still streaming in), it only updates the UI via `cline.ask` and returns, deferring execution.
*   **Validation:** Checks if the required `path` parameter is present. If not, it uses `cline.sayAndCreateMissingParamError` to generate a standard error message and pushes it using `pushToolResult`.
*   **Approval:** It calls `askApproval` with details about the intended action. The function pauses execution here until the user responds in the UI. If the user rejects (`didApprove` is false), it pushes a rejection message and returns.
*   **Action:** If approved, it calls the actual file listing logic (`listFiles` from `src/services/glob/list-files.ts`, which likely uses the `glob` library and respects `.rooignore` rules via `cline.rooIgnoreController` ([Chapter 21: RooIgnoreController](21_rooignorecontroller.md))).
*   **Formatting:** It formats the list of files using `formatResponse.formatFilesList` into a string suitable for the LLM.
*   **Result Pushing:** It calls `pushToolResult` with the formatted file list.
*   **Error Handling:** A `try...catch` block wraps the main logic to call `handleError` for any unexpected exceptions during file listing.

## Internal Implementation

The process of invoking and executing a tool involves several components working together:

**Step-by-Step Flow:**

1.  **AI Generates Tool XML:** The LLM includes `<tool_name>...</tool_name>` in its response stream.
2.  **Stream Received (`Cline`):** `Cline` receives stream chunks via the [Chapter 6: ApiStream](06_apistream.md) from the [Chapter 5: ApiHandler](05_apihandler.md).
3.  **Parsing (`parseAssistantMessage`):** As text accumulates, `parseAssistantMessage` ([Chapter 25: Assistant Message Parsing](25_assistant_message_parsing.md)) identifies the tool XML and parses it into a `ToolUse` object, potentially marking it as `partial`.
4.  **Orchestration (`presentAssistantMessage`):** This method in `Cline` receives the `ToolUse` object(s).
5.  **Partial Handling:** If `partial` is true, `presentAssistantMessage` typically updates the UI via `ask` (showing the tool being typed out) but defers full execution.
6.  **Full Block Handling:** When `partial` is false:
    *   It checks state flags (`didRejectTool`, `didAlreadyUseTool`).
    *   It calls `validateToolUse(toolName, mode, ...)` ([Chapter core/mode-validator.ts](TODO: Link not in list, potentially part of Chapter 10 or other)).
    *   It enters the `switch` statement for the specific `toolName`.
    *   It defines the standard callbacks (`askApproval`, `handleError`, `pushToolResult`).
    *   It calls the specific tool function (e.g., `readFileTool(...)`).
7.  **Approval (`askApproval` inside Tool Function):**
    *   The tool function calls `askApproval`.
    *   `askApproval` calls `Cline.ask`.
    *   `Cline` emits `message` event.
    *   `ClineProvider` sends message to UI.
    *   **PAUSE:** Execution pauses, awaiting user response.
    *   User responds.
    *   `ClineProvider` calls `Cline.handleWebviewAskResponse`.
    *   `Cline.ask`'s internal `pWaitFor` resolves.
    *   `askApproval` promise resolves (true/false).
8.  **Execution (Tool Function):**
    *   If approved, the tool function performs its core action (read file, run command, etc.).
    *   It interacts with necessary VS Code APIs, filesystem, or external services.
9.  **Result Formatting (Tool Function):**
    *   The tool function formats the outcome (data or error) into a string (often XML).
10. **Result Pushing (Tool Function):**
    *   It calls `pushToolResult(formattedResult)`.
11. **State Update (`pushToolResult` callback in `Cline`):**
    *   The callback adds the `formattedResult` to `cline.userMessageContent`.
    *   It sets `cline.didAlreadyUseTool = true`.
12. **Loop Synchronization (`presentAssistantMessage`):**
    *   The `finally` block in `presentAssistantMessage` sets `cline.userMessageContentReady = true`.
13. **Agent Loop Continues (`recursivelyMakeClineRequests`):**
    *   The `pWaitFor(() => this.userMessageContentReady)` resolves.
    *   The loop prepares the next API request, including the tool result in the 'user' message.

**Sequence Diagram:**

```mermaid
sequenceDiagram
    participant AI as LLM
    participant Cline
    participant Parser as parseAssistantMessage
    participant Orchestrator as presentAssistantMessage
    participant Validator as validateToolUse
    participant ToolFn as Specific Tool Function (e.g., readFileTool)
    participant UI as WebView UI
    participant Provider as ClineProvider
    participant ActionSvc as Action Service (FS, Terminal, etc.)

    AI-->>Cline: Stream chunk with <tool>...</tool>
    Cline->>Parser: Process stream chunk
    Parser-->>Cline: ToolUse object (partial or full)
    Cline->>Orchestrator: presentAssistantMessage()
    alt Full ToolUse Block
        Orchestrator->>Validator: validateToolUse(name, mode)
        Validator-->>Orchestrator: OK (or throws error)
        Orchestrator->>ToolFn: readFileTool(cline, block, askApproval, ...)
        ToolFn->>Orchestrator: askApproval('tool', details)
        Orchestrator->>Cline: ask('tool', details)
        Cline->>Provider: emit('message', askPayload)
        Provider->>UI: postMessage(askPayload)
        UI->>User: Show Approval Request
        User->>UI: Click Approve
        UI->>Provider: postMessage(askResponse)
        Provider->>Cline: handleWebviewAskResponse(...)
        Note over Cline, Orchestrator, ToolFn: ask/askApproval resolves true
        ToolFn->>ActionSvc: Perform Action (e.g., read file)
        ActionSvc-->>ToolFn: Action Result (e.g., file content)
        ToolFn->>ToolFn: Format Result (XML String)
        ToolFn->>Orchestrator: pushToolResult(formattedResult)
        Orchestrator->>Cline: Add result to userMessageContent, set didAlreadyUseTool=true
        Orchestrator->>Orchestrator: Set userMessageContentReady=true
    end
    Note over Cline: Main loop continues with tool result
```

## Modification Guidance

Tools are a primary area for extending Roo-Code's capabilities.

**Common Modifications:**

1.  **Adding a New Tool:** (Covered in Chapter 7 guidance, summarized here)
    *   **Implement Function:** Create `src/core/tools/myNewTool.ts` with the standard async signature. Implement validation, action logic, use `askApproval`, and call `pushToolResult` or `handleError`.
    *   **Define Description:** Create `src/core/prompts/tools/my-new-tool.ts` exporting `getMyNewToolDescription` which returns the XML definition.
    *   **Register:**
        *   Add the tool name to `ToolName` schema.
        *   Add the function to `toolDescriptionMap` in `src/core/prompts/tools/index.ts`.
        *   Add a `case 'my_new_tool':` to the `switch` in `Cline.presentAssistantMessage` calling your function.
    *   **Mode Configuration:** Add the tool to relevant `TOOL_GROUPS` in `src/shared/tools.ts` or configure it specifically in mode definitions (`src/shared/modes.ts` or custom modes). Update `isToolAllowedForMode` if needed.
    *   **UI (Optional):** If the tool involves specific UI interactions beyond simple approval (like a custom dialog), implement that UI and the corresponding message protocol updates.

2.  **Modifying Tool Logic:**
    *   **Locate:** Open the specific tool file (e.g., `src/core/tools/executeCommandTool.ts`).
    *   **Edit:** Change how parameters are handled, how external services/APIs are called, or how results are formatted before calling `pushToolResult`.
    *   **Test:** Ensure the changes work correctly and handle edge cases and potential errors.

3.  **Changing Tool Parameters:**
    *   **Description:** Update the `<parameters>` section in the tool's description function (`src/core/prompts/tools/*.ts`).
    *   **Implementation:** Modify the tool function (`src/core/tools/*.ts`) to read the new/changed parameters from `block.params` and use them in its logic. Add validation for new required parameters.
    *   **Types:** Update `ToolParamName` in `src/shared/tools.ts` if adding a completely new parameter name.

4.  **Adjusting Tool Availability:**
    *   Modify the `groups` or `tools` properties in the `ModeConfig` for the relevant modes (built-in or custom) as described in [Chapter 7: SystemPrompt](07_systemprompt.md) modification guidance.

**Best Practices:**

*   **Single Responsibility:** Each tool function should focus on performing one well-defined action.
*   **Parameter Validation:** Robustly validate all required parameters within the tool function before proceeding. Use `cline.sayAndCreateMissingParamError` for standardized error reporting.
*   **Clear Approval Messages:** Provide concise but informative details to the user when calling `askApproval`. Use JSON stringification of a structured object (`ClineSayTool` or similar) for consistency.
*   **Informative Results:** Format results clearly for the LLM, including context (like file paths) and explicit success/error messages within the `pushToolResult` call.
*   **Idempotency (where possible):** Design tools so that accidental duplicate execution (if possible) doesn't cause harm.
*   **Security:** Be extremely cautious with tools that execute commands (`execute_command`) or modify files (`write_to_file`, `apply_diff`). Rely on user approval and potentially add further safeguards or configuration options. Validate paths and commands strictly.

**Potential Pitfalls:**

*   **Missing Parameter Validation:** Leads to errors or unexpected behavior if the AI provides incomplete requests.
*   **Unhandled Errors:** Tool functions crashing without calling `handleError` can break the agent loop.
*   **Security Vulnerabilities:** Poorly validated commands or file paths in tools like `execute_command` or `write_to_file` could allow malicious use.
*   **Vague Approval Prompts:** Users might approve actions without fully understanding the implications.
*   **Incorrect Result Formatting:** Results not formatted correctly (especially errors) can confuse the LLM in the next turn.
*   **Blocking Operations:** Long-running synchronous operations within a tool function will block the entire extension host. Use `async/await` and asynchronous APIs.

## Conclusion

Tools are the hands and voice of the Roo-Code AI agent, allowing it to interact meaningfully with the developer's environment. Defined in the System Prompt, parsed from AI responses, orchestrated by `Cline`, and implemented as distinct functions, they form a robust and extensible system for action execution. The integration of mode-based validation and user approval provides crucial control and safety mechanisms. Understanding how tools are defined, validated, approved, executed, and how their results are fed back into the conversation is essential for comprehending Roo-Code's agentic capabilities.

With the core AI interaction loop (`Cline`, `ApiHandler`, `SystemPrompt`, `Tools`) covered, we now move towards configuration management. The next chapter explores how Roo-Code manages different sets of API keys and provider settings: [Chapter 9: ProviderSettingsManager](09_providersettingsmanager.md).

