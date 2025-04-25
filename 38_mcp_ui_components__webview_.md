# Chapter 38: MCP UI Components (WebView)

Continuing from [Chapter 37: Prompts UI Components (WebView)](37_prompts_ui_components__webview_.md), where we explored the UI for managing AI prompts and modes, this chapter focuses on the user interface elements dedicated to discovering, managing, and interacting with external tools and context sources via the Model Context Protocol (MCP): the **MCP UI Components**.

## Motivation: Visualizing and Managing External MCP Servers

The Model Context Protocol ([Chapter 19: McpHub / McpServerManager](19_mcphub___mcpservermanager.md)) allows Roo-Code to connect to external processes (MCP servers) that provide custom tools and context resources. While the `McpHub` handles the connection and communication logic in the background, users need a visual interface within the Roo-Code WebView to:

*   View Connected Servers: See which MCP servers (both global and project-specific) are currently connected or attempting to connect.
*   Inspect Server Capabilities: Understand which tools and resources each connected server offers.
*   Monitor Server Status: Check if a server is connected, disconnected, or has encountered an error. See error messages if connection failed.
*   Manage Server Lifecycle: Manually restart a specific server connection (e.g., after updating its implementation or configuration). Enable/disable specific servers. Delete server configurations.
*   Manage Tool Permissions: View and toggle the "Always Allow" setting for specific tools provided by MCP servers, controlling whether user approval is required each time the AI wants to use that tool.
*   Access Configuration: Easily open the global (`settings/mcp_settings.json`) or project (`.roo/mcp.json`) configuration files.

The MCP UI Components, primarily `McpView.tsx` and its sub-components like `McpToolRow.tsx`, `McpResourceRow.tsx`, and `McpEnabledToggle.tsx`, provide this management interface, allowing users to oversee and control their MCP integrations directly within the WebView panel.

**Central Use Case:** A user has configured a local project-specific MCP server (`my-local-analyzer`) and a global remote server (`shared-docs-lookup`). They want to check their status, see the tools offered by the local server, always allow a specific tool on that server, and restart the local server after making code changes to it.

1.  User clicks the "MCP" icon/tab in the Roo-Code WebView UI.
2.  `McpView.tsx` renders. It gets the list of `mcpServers` (including status, tools, source) from the global [Chapter 12: ExtensionStateContext](12_extensionstatecontext.md). It also renders the main `McpEnabledToggle`.
3.  The view displays two sections: "Project Servers" and "Global Servers". `my-local-analyzer` appears under Project, and `shared-docs-lookup` under Global. Connection status icons (e.g., green check, red cross) are shown.
4.  User expands the `my-local-analyzer` entry (likely using an Accordion primitive). The list of tools (`analyze_function`, `check_style`) provided by the server is displayed using `McpToolRow`.
5.  User finds the `check_style` tool rendered by `McpToolRow` and clicks its associated "Always Allow" toggle (`VSCodeCheckbox` component).
6.  The `onChange` handler in `McpToolRow` sends a `toggleToolAlwaysAllow` message to the extension host with the server name, source, tool name, and the new `true` value.
7.  The extension host updates the `.roo/mcp.json` file via `McpHub` and sends back an updated state. The toggle reflects the saved state.
8.  User makes changes to the `my-local-analyzer` server's code in the editor and wants to restart it.
9.  User clicks the "Restart" button (`codicon-refresh`) next to the `my-local-analyzer` entry in `McpView`.
10. The `onClick` handler sends a `restartMcpServer` message with the server name and source.
11. The extension host tells `McpHub` to restart the connection. `McpHub` disconnects, potentially terminates the old process (for stdio), starts the new process, reconnects, fetches capabilities, and sends back an updated state reflecting the new connection status and potentially updated tools.

## Key Concepts

1.  **`McpView.tsx`:** The main container component for the MCP management UI.
    *   **Data Source:** Primarily displays data from the `mcpServers` array fetched from `useExtensionState()` ([Chapter 12: ExtensionStateContext](12_extensionstatecontext.md)). Each element is an `McpServer` object containing configuration, status (`connected`, `disconnected`, `connecting`, `error`), source (`global`, `project`), fetched `tools`, `resources`, and potential `error` messages. Also uses `mcpEnabled` and `alwaysAllowMcp`.
    *   **Layout:** Uses `Tab`, `TabHeader`, `TabContent`. Header includes title and Done button. Content includes the main `McpEnabledToggle`, server lists, and config edit buttons.
    *   **Server Grouping:** Separates servers into "Project Servers" and "Global Servers" based on the `server.source` property (though the provided code snippet doesn't explicitly show grouping, it's a common pattern).
    *   **Server Display (`ServerRow`):** Renders each server as a collapsible item (the snippet uses a `div` structure mimicking an accordion trigger/content). Displays server name, status icon (using `ConnectionStatus` helper), source badge, and error message if present. Includes action buttons (Restart, Enable/Disable, Delete).
    *   **Capability Display:** When a server item is expanded, it uses `VSCodePanels` to show tabs for "Tools" and "Resources". It maps over `server.tools` and `server.resources`/`server.resourceTemplates`, rendering them using `McpToolRow` and `McpResourceRow`.
    *   **Configuration:** Displays network timeout settings and allows editing via a `<select>` element, sending `updateMcpTimeout` messages. Includes buttons to trigger `openMcpSettings` / `openProjectMcpSettings` messages.
    *   **Interaction:** Button clicks (`Restart`, `Delete`, `Enable/Disable` toggle) and timeout changes trigger `vscode.postMessage` calls (`restartMcpServer`, `deleteMcpServer`, `toggleMcpServer`, `updateMcpTimeout`) to the extension host.

2.  **`McpEnabledToggle.tsx`:** A component displaying a `VSCodeCheckbox` to enable or disable the entire MCP feature globally. Its `onChange` handler uses `setMcpEnabled` (from `useExtensionState`) and sends an `mcpEnabled` message to the host.

3.  **`McpToolRow.tsx`:** Renders details for a single `McpTool`.
    *   Displays the tool icon (`codicon-symbol-method`), name, and description.
    *   Displays input parameters (`tool.inputSchema`) if available, indicating required ones.
    *   Conditionally renders an "Always Allow" `VSCodeCheckbox` if the global `alwaysAllowMcp` setting is enabled. The checkbox state is bound to `tool.alwaysAllow`, and `onChange` sends the `toggleToolAlwaysAllow` message to the host.

4.  **`McpResourceRow.tsx`:** Renders details for an `McpResource` or `McpResourceTemplate`. Displays icon, URI/Template, Name/Description, and MIME type.

5.  **`McpServer` Data Structure (`src/shared/mcp.ts`):** The object format representing a server in the UI state, sent from the host. Includes configuration details, connection `status`, `source`, fetched `tools` and `resources` (with their descriptions and `alwaysAllow` flags), and any `error`.

6.  **`ConnectionStatus` (Helper):** (Not shown in snippets, but conceptually used) A simple component rendering different status icons (e.g., `codicon-check`, `codicon-sync~spin`, `codicon-error`) based on the `server.status` string, likely with tooltips.

7.  **Interaction Flow:** User actions in `McpView`/`McpToolRow` trigger `postMessage` calls. `webviewMessageHandler` receives these, calls corresponding methods on `McpHub` (e.g., `restartConnection`, `deleteServerConfig`, `toggleServerEnabled`, `toggleToolAlwaysAllow`). `McpHub` performs the action (interacting with SDK, file system) and then calls `notifyWebviewOfServerChanges` to send the updated `mcpServers` list back to all WebView instances. `ExtensionStateContext` updates, causing `McpView` to re-render.

8.  **UI Primitives:** Leverages themed [Chapter 33: Shadcn/UI Primitives (WebView)](33_shadcn_ui_primitives__webview_.md) (`Button`, `Dialog`, `Tooltip`) and [Chapter 32: VSCode Webview UI Toolkit Wrappers](32_vscode_webview_ui_toolkit_wrappers.md) (`VSCodeBadge`, `VSCodeCheckbox`, `VSCodePanels`, `VSCodePanelTab`, `VSCodePanelView`, `VSCodeLink`) for layout, interaction, and styling consistent with the VS Code theme.

## Using the MCP UI Components (Use Case Revisited)

Let's trace the user managing their MCP servers:

1.  **Render `McpView`:** Gets `mcpServers`, `mcpEnabled`, `alwaysAllowMcp` from `useExtensionState`. Renders `McpEnabledToggle`. Renders the `ServerRow` for `my-local-analyzer` and `shared-docs-lookup`. Status icons are green checks.
2.  **Expand Server:** User clicks the trigger area for `my-local-analyzer`. `ServerRow`'s `handleRowClick` sets `isExpanded(true)`. The component re-renders, showing the `AccordionContent` (the `div` with `VSCodePanels`).
3.  **Display Tools:** The "Tools" `VSCodePanelView` maps over `myLocalAnalyzerServer.tools`, rendering `McpToolRow` for `check_style`.
4.  **Toggle "Always Allow":** Inside `McpToolRow` for `check_style`, the user clicks the `VSCodeCheckbox`.
    *   `handleAlwaysAllowChange` is called with the event.
    *   It sends `postMessage({ type: "toggleToolAlwaysAllow", serverName: "my-local-analyzer", source: "project", toolName: "check_style", alwaysAllow: true })`.
5.  **Host Processes & Update:** Host calls `McpHub.toggleToolAlwaysAllow`, which updates `.roo/mcp.json`, re-fetches capabilities (`alwaysAllow` is now true), and sends back the updated `mcpServers` list.
6.  **UI Re-renders:** `McpView` receives the new state. `McpToolRow` for `check_style` re-renders with the checkbox now checked.
7.  **Click Restart:** User clicks the "Restart" button (`codicon-refresh`) in the `ServerRow` header for `my-local-analyzer`.
    *   `handleRestart` calls `postMessage({ type: "restartMcpServer", serverName: "my-local-analyzer", source: "project" })`.
8.  **Host Processes & Update:**
    *   Host calls `McpHub.restartConnection(...)`.
    *   `McpHub` updates status to `"connecting"`, calls `notifyWebviewOfServerChanges`.
    *   UI re-renders, showing spinning icon for `my-local-analyzer`.
    *   `McpHub` finishes reconnection, updates status (e.g., `"connected"`), calls `notifyWebviewOfServerChanges`.
    *   UI re-renders, showing green check icon again.

## Code Walkthrough

### `McpView.tsx`

```typescript
// --- File: webview-ui/src/components/mcp/McpView.tsx ---
import { useState } from "react"
import { Button } from "@/components/ui/button"
import {
	VSCodeCheckbox,
	VSCodeLink,
	VSCodePanels,
	VSCodePanelTab,
	VSCodePanelView,
} from "@vscode/webview-ui-toolkit/react" // Import toolkit components

import { McpServer } from "@roo/shared/mcp" // Import shared type

import { vscode } from "@/utils/vscode" // Import message utility
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogDescription, DialogFooter } from "@/components/ui" // Import shadcn Dialog

import { useExtensionState } from "@src/context/ExtensionStateContext" // Global state hook
import { useAppTranslation } from "@src/i18n/TranslationContext" // Translation hook
import { Trans } from "react-i18next" // Translation component
import { Tab, TabContent, TabHeader } from "../common/Tab" // Layout components
import McpToolRow from "./McpToolRow" // Sub-component for tools
import McpResourceRow from "./McpResourceRow" // Sub-component for resources
import McpEnabledToggle from "./McpEnabledToggle" // Main feature toggle

type McpViewProps = {
	onDone: () => void // Callback when user clicks Done
}

// Main View Component
const McpView = ({ onDone }: McpViewProps) => {
	// Get relevant state from global context
	const {
		mcpServers: servers,
		alwaysAllowMcp,
		mcpEnabled,
		enableMcpServerCreation,
		setEnableMcpServerCreation, // Setter from context
	} = useExtensionState()
	const { t } = useAppTranslation() // Translation function

	return (
		// Basic tab layout
		<Tab>
			<TabHeader className="flex justify-between items-center">
				<h3 className="text-vscode-foreground m-0">{t("mcp:title")}</h3>
				<Button onClick={onDone}>{t("mcp:done")}</Button>
			</TabHeader>

			<TabContent>
				{/* Description with links */}
				<div /* ... styles ... */ >
					<Trans i18nKey="mcp:description">
						<VSCodeLink href="https://github.com/modelcontextprotocol" style={{ display: "inline" }}>
							Model Context Protocol
						</VSCodeLink>
						<VSCodeLink href="https://github.com/modelcontextprotocol/servers" style={{ display: "inline" }}>
							community-made servers
						</VSCodeLink>
					</Trans>
				</div>

				{/* Main Enable/Disable Toggle */}
				<McpEnabledToggle />

				{/* Conditionally render server list and options if MCP is enabled */}
				{mcpEnabled && (
					<>
						{/* Option to enable AI creating servers */}
						<div style={{ marginBottom: 15 }}>
							<VSCodeCheckbox
								checked={enableMcpServerCreation}
								onChange={(e: any) => {
									// Update local context state AND send message to host
									setEnableMcpServerCreation(e.target.checked)
									vscode.postMessage({ type: "enableMcpServerCreation", bool: e.target.checked })
								}}>
								<span style={{ fontWeight: "500" }}>{t("mcp:enableServerCreation.title")}</span>
							</VSCodeCheckbox>
							<p /* ... description styles ... */ >
								{t("mcp:enableServerCreation.description")}
							</p>
						</div>

						{/* Server List */}
						{servers.length > 0 && (
							<div style={{ display: "flex", flexDirection: "column", gap: "10px" }}>
								{/* Map over servers received from state and render ServerRow for each */}
								{servers.map((server) => (
									<ServerRow
										key={`${server.name}-${server.source || "global"}`}
										server={server}
										alwaysAllowMcp={alwaysAllowMcp}
									/>
								))}
							</div>
						)}

						{/* Edit Settings Buttons */}
						<div /* ... styles ... */ >
							<Button variant="secondary" style={{ flex: 1 }}
								onClick={() => { vscode.postMessage({ type: "openMcpSettings" }) }}>
								{/* Edit Global MCP */}
							</Button>
							<Button variant="secondary" style={{ flex: 1 }}
								onClick={() => { vscode.postMessage({ type: "openProjectMcpSettings" }) }}>
								{/* Edit Project MCP */}
							</Button>
						</div>
					</>
				)}
			</TabContent>
		</Tab>
	)
}

// Component to render a single server entry
const ServerRow = ({ server, alwaysAllowMcp }: { server: McpServer; alwaysAllowMcp?: boolean }) => {
	const { t } = useAppTranslation()
	// Local state for expansion and delete confirmation
	const [isExpanded, setIsExpanded] = useState(false)
	const [showDeleteConfirm, setShowDeleteConfirm] = useState(false)
	// Local state for timeout value, initialized from server config
	const [timeoutValue, setTimeoutValue] = useState(() => { /* ... parse timeout or default ... */ })

	const timeoutOptions = [ /* ... timeout options array ... */ ]

	// Helper to get status color
	const getStatusColor = () => { /* ... logic based on server.status ... */ }

	// Toggle expansion on click (unless there's an error)
	const handleRowClick = () => { if (!server.error) { setIsExpanded(!isExpanded) } }

	// Send restart message
	const handleRestart = () => { vscode.postMessage({ type: "restartMcpServer", ... }) }

	// Send timeout update message
	const handleTimeoutChange = (event: React.ChangeEvent<HTMLSelectElement>) => {
		// ... parse value, update local state, send updateMcpTimeout message ...
	}

	// Send delete message (after confirmation)
	const handleDelete = () => { vscode.postMessage({ type: "deleteMcpServer", ... }); setShowDeleteConfirm(false); }

	return (
		<div style={{ marginBottom: "10px" }}>
			{/* Trigger Row (Header) */}
			<div style={{ /* ... styles ... */ }} onClick={handleRowClick}>
				{/* Expansion chevron */}
				{!server.error && <span className={`codicon codicon-chevron-${isExpanded ? "down" : "right"}`} />}
				{/* Server Name + Source Badge */}
				<span style={{ flex: 1 }}>{server.name} {/* ... Badge ... */}</span>
				{/* Action Buttons (Delete, Restart, Enable/Disable) */}
				<div onClick={(e) => e.stopPropagation()}>
					<Button variant="ghost" size="icon" onClick={() => setShowDeleteConfirm(true)}> {/* Trash */} </Button>
					<Button variant="ghost" size="icon" onClick={handleRestart} disabled={server.status === 'connecting'}> {/* Refresh */} </Button>
					{/* Enable/Disable Toggle (Custom implementation using divs) */}
					<div role="switch" onClick={() => vscode.postMessage({ type: "toggleMcpServer", ... })}> {/* Toggle */} </div>
				</div>
				{/* Status Indicator Dot */}
				<div style={{ /* ... styles based on getStatusColor() ... */ }} />
			</div>

			{/* Error Display */}
			{server.error ? ( <div /* ... error message + retry button ... */ > {server.error} <Button onClick={handleRestart}>Retry</Button> </div> )
			/* Expanded Content */
			: ( isExpanded && (
					<div style={{ /* ... content styles ... */ }}>
						{/* Tabs for Tools & Resources */}
						<VSCodePanels style={{ marginBottom: "10px" }}>
							<VSCodePanelTab id="tools">Tools ({server.tools?.length || 0})</VSCodePanelTab>
							<VSCodePanelTab id="resources">Resources ({/* ... count ... */})</VSCodePanelTab>
							{/* Tools Panel */}
							<VSCodePanelView id="tools-view">
								{server.tools?.length > 0 ? (
									<div> {server.tools.map((tool) => ( <McpToolRow key={...} tool={tool} ... /> ))} </div>
								) : ( <div>No Tools</div> )}
							</VSCodePanelView>
							{/* Resources Panel */}
							<VSCodePanelView id="resources-view">
								{(server.resources || server.resourceTemplates)?.length > 0 ? (
									<div> {[...(server.resourceTemplates || []), ...(server.resources || [])].map((item) => ( <McpResourceRow key={...} item={item} /> ))} </div>
								) : ( <div>No Resources</div> )}
							</VSCodePanelView>
						</VSCodePanels>
						{/* Network Timeout Select */}
						<div /* ... timeout label + select dropdown ... */ >
							<select value={timeoutValue} onChange={handleTimeoutChange}> {/* Options */} </select>
						</div>
					</div>
				)
			)}
			{/* Delete Confirmation Dialog */}
			<Dialog open={showDeleteConfirm} onOpenChange={setShowDeleteConfirm}> {/* ... Dialog Content ... */} </Dialog>
		</div>
	)
}

export default McpView
```

**Explanation:**

*   **`McpView`:**
    *   Gets `mcpServers`, `mcpEnabled`, `alwaysAllowMcp`, `enableMcpServerCreation` from `useExtensionState`.
    *   Renders the `McpEnabledToggle`.
    *   Conditionally renders the rest of the UI based on `mcpEnabled`.
    *   Renders the `enableMcpServerCreation` checkbox, sending a message on change.
    *   Maps over the `servers` array, rendering a `ServerRow` for each.
    *   Provides buttons to open global/project settings files via `postMessage`.
*   **`ServerRow`:**
    *   Manages local `isExpanded` and `showDeleteConfirm` state.
    *   Renders the header row with name, source badge, action buttons (Delete, Restart, Enable/Disable toggle), and status dot. Button handlers send `postMessage` calls.
    *   Conditionally renders error details or the expanded content (`VSCodePanels` with `McpToolRow` / `McpResourceRow`) based on `server.error` and `isExpanded`.
    *   Includes the network timeout `<select>` dropdown, sending `updateMcpTimeout` on change.
    *   Includes the shadcn `Dialog` for delete confirmation.

### `McpEnabledToggle.tsx`

```typescript
// --- File: webview-ui/src/components/mcp/McpEnabledToggle.tsx ---
import { VSCodeCheckbox } from "@vscode/webview-ui-toolkit/react" // Import toolkit checkbox
import { FormEvent } from "react"
import { useExtensionState } from "@src/context/ExtensionStateContext" // Global state hook
import { useAppTranslation } from "@src/i18n/TranslationContext" // Translation hook
import { vscode } from "@src/utils/vscode" // Message utility

// Component for the main MCP Enable/Disable toggle
const McpEnabledToggle = () => {
	// Get current state and setter from global context
	const { mcpEnabled, setMcpEnabled } = useExtensionState()
	const { t } = useAppTranslation() // Translation function

	// Handle checkbox change event
	const handleChange = (e: Event | FormEvent<HTMLElement>) => {
		const target = ("target" in e ? e.target : null) as HTMLInputElement | null
		if (!target) return
		// Update local context state immediately for UI feedback
		setMcpEnabled(target.checked)
		// Send message to extension host to persist the change
		vscode.postMessage({ type: "mcpEnabled", bool: target.checked })
	}

	return (
		<div style={{ marginBottom: "20px" }}>
			{/* Render VS Code styled checkbox */}
			<VSCodeCheckbox checked={mcpEnabled} onChange={handleChange}>
				<span style={{ fontWeight: "500" }}>{t("mcp:enableToggle.title")}</span>
			</VSCodeCheckbox>
			<p /* ... description styles ... */ >
				{t("mcp:enableToggle.description")}
			</p>
		</div>
	)
}

export default McpEnabledToggle
```

**Explanation:**

*   Uses `useExtensionState` to get the current `mcpEnabled` state and its *context setter function* `setMcpEnabled`.
*   Renders a `VSCodeCheckbox` bound to the `mcpEnabled` state.
*   The `handleChange` function updates the state locally using `setMcpEnabled` (for immediate UI feedback) and sends the `mcpEnabled` message to the host for persistent storage via `ContextProxy`.

### `McpToolRow.tsx`

```typescript
// --- File: webview-ui/src/components/mcp/McpToolRow.tsx ---
import { VSCodeCheckbox } from "@vscode/webview-ui-toolkit/react" // Import toolkit checkbox
import { McpTool } from "@roo/shared/mcp" // Import shared type
import { useAppTranslation } from "@src/i18n/TranslationContext" // Translation hook
import { vscode } from "@src/utils/vscode" // Message utility

type McpToolRowProps = {
	tool: McpTool // Data for the specific tool
	serverName?: string // Name of the parent server
	serverSource?: "global" | "project" // Source of the parent server
	alwaysAllowMcp?: boolean // Global setting if auto-approve feature is enabled
}

// Component to render a single tool entry within a server's details
const McpToolRow = ({ tool, serverName, serverSource, alwaysAllowMcp }: McpToolRowProps) => {
	const { t } = useAppTranslation() // Translation function

	// Handle change for the "Always Allow" checkbox
	const handleAlwaysAllowChange = () => {
		if (!serverName) return // Should not happen if checkbox is rendered
		// Send message to host to toggle the setting
		vscode.postMessage({
			type: "toggleToolAlwaysAllow",
			serverName,
			source: serverSource || "global", // Default to global if source missing
			toolName: tool.name,
			alwaysAllow: !tool.alwaysAllow, // Send the NEW desired state
		})
	}

	return (
		<div key={tool.name} style={{ padding: "3px 0" }} >
			{/* Main row with icon, name, and toggle */}
			<div data-testid="tool-row-container" /* ... flex styles ... */ onClick={(e) => e.stopPropagation()} >
				{/* Icon and Name */}
				<div style={{ display: "flex", alignItems: "center" }}>
					<span className="codicon codicon-symbol-method" style={{ marginRight: "6px" }}></span>
					<span style={{ fontWeight: 500 }}>{tool.name}</span>
				</div>
				{/* "Always Allow" Toggle (conditional) */}
				{serverName && alwaysAllowMcp && (
					<VSCodeCheckbox
						checked={tool.alwaysAllow} // Bind checked state to tool data
						onChange={handleAlwaysAllowChange} // Attach handler
						data-tool={tool.name} // Store tool name for testing/debugging
					>
						{t("mcp:tool.alwaysAllow")}
					</VSCodeCheckbox>
				)}
			</div>
			{/* Tool Description */}
			{tool.description && ( <div /* ... description styles ... */ > {tool.description} </div> )}
			{/* Input Parameters (if schema provided) */}
			{tool.inputSchema && /* ... Check schema structure ... */ (
					<div /* ... parameter box styles ... */ >
						<div /* ... header ... */ > {t("mcp:tool.parameters")} </div>
						{/* Map over parameters in inputSchema.properties */}
						{Object.entries(tool.inputSchema.properties as Record<string, any>).map(
							([paramName, schema]) => {
								const isRequired = /* ... check if paramName in schema.required ... */
								return (
									<div key={paramName} /* ... parameter row styles ... */ >
										<code>{paramName}{isRequired && <span style={{ color: "var(--vscode-errorForeground)" }}>*</span>}</code>
										<span /* ... description styles ... */ > {schema.description || t("mcp:tool.noDescription")} </span>
									</div>
								)
							},
						)}
					</div>
				)}
		</div>
	)
}

export default McpToolRow
```

**Explanation:**

*   Receives `tool` data, parent server info (`serverName`, `serverSource`), and the global `alwaysAllowMcp` flag as props.
*   Renders the tool name, description, and input parameters (extracted from `tool.inputSchema`).
*   Conditionally renders the "Always Allow" `VSCodeCheckbox` based on `alwaysAllowMcp`.
*   The checkbox's `checked` state is bound to `tool.alwaysAllow`.
*   `onChange` calls `handleAlwaysAllowChange`, which sends the `toggleToolAlwaysAllow` message to the host, including all necessary identifiers (server name, source, tool name) and the new boolean value.

### `McpResourceRow.tsx`

```typescript
// --- File: webview-ui/src/components/mcp/McpResourceRow.tsx ---
import { McpResource, McpResourceTemplate } from "@roo/shared/mcp" // Import shared types

type McpResourceRowProps = {
	item: McpResource | McpResourceTemplate // Can be a static resource or a template
}

// Component to render a single resource or resource template
const McpResourceRow = ({ item }: McpResourceRowProps) => {
	// Determine if it's a static resource (has uri) or template (has uriTemplate)
	const hasUri = "uri" in item
	const uri = hasUri ? item.uri : item.uriTemplate // Get the identifier

	return (
		<div key={uri} style={{ padding: "3px 0" }}>
			{/* Icon and URI/Template */}
			<div /* ... flex styles ... */ >
				<span className={`codicon codicon-symbol-file`} style={{ marginRight: "6px" }} />
				<span style={{ fontWeight: 500, wordBreak: "break-all" }}>{uri}</span>
			</div>
			{/* Name / Description */}
			<div /* ... description styles ... */ >
				{/* Logic to display name and/or description */}
				{item.name && item.description ? `${item.name}: ${item.description}` : /* ... fallbacks ... */ "No description"}
			</div>
			{/* MIME Type */}
			<div style={{ fontSize: "12px" }}>
				<span style={{ opacity: 0.8 }}>Returns </span>
				<code /* ... code styles ... */ > {item.mimeType || "Unknown"} </code>
			</div>
		</div>
	)
}

export default McpResourceRow
```

**Explanation:**

*   Receives either an `McpResource` or `McpResourceTemplate` object.
*   Determines the identifier (`uri` or `uriTemplate`).
*   Renders the icon, identifier, name/description, and the expected `mimeType`.

## Internal Implementation

1.  **Data Source:** `McpHub` manages connections and fetches capabilities (`tools`, `resources`) from servers.
2.  **Synchronization:** `McpHub` calls `notifyWebviewOfServerChanges` after updates.
3.  **Message:** `notifyWebviewOfServerChanges` sends the `mcpServers` array (containing status, config, tools, resources, errors) via `postMessage`.
4.  **State Update:** `ExtensionStateContext` updates the `mcpServers` state.
5.  **UI Render:** `McpView` gets the new `mcpServers` array via `useExtensionState`. It re-renders, mapping over the servers and passing data down to `ServerRow`.
6.  **Sub-Component Render:** `ServerRow` renders status, buttons, and conditionally renders `McpToolRow` and `McpResourceRow` based on the data within each `McpServer` object.
7.  **Interaction:** User clicks a button or toggle in `McpView` or `McpToolRow`.
8.  **Message:** The component's event handler sends a specific `postMessage` call (e.g., `restartMcpServer`, `toggleToolAlwaysAllow`) with necessary identifiers.
9.  **Host Action:** `webviewMessageHandler` routes the message to the appropriate `McpHub` method.
10. **Feedback Loop:** `McpHub` performs the action and triggers step 2 (Synchronization) again, causing the UI to update reflecting the result of the action (e.g., new status, changed toggle state).

**Sequence Diagram (Restart Server):**

```mermaid
sequenceDiagram
    participant User
    participant McpView
    participant RestartBtn as Restart Button (in ServerRow)
    participant VSCodeAPI as VS Code WebView API
    participant ExtHost as Extension Host
    participant McpHub
    participant ServerProc as MCP Server Process/Endpoint
    participant ExtStateCtx as ExtensionStateContext

    User->>RestartBtn: Click Restart
    RestartBtn->>McpView: onClick -> handleRestart()
    McpView->>VSCodeAPI: postMessage({ type: "restartMcpServer", serverName, source })
    VSCodeAPI-->>ExtHost: Receive message
    ExtHost->>McpHub: restartConnection(serverName, source)
    McpHub->>McpHub: Update status to "connecting"
    McpHub->>McpHub: notifyWebviewOfServerChanges() --> Send "connecting" status
    McpHub->>McpHub: deleteConnection(...) --> Close old transport/client
    McpHub->>McpHub: connectToServer(...) --> Create new transport/client, connect to ServerProc
    alt Connect Succeeds
        McpHub->>McpHub: Update status to "connected", fetch capabilities
        McpHub->>McpHub: notifyWebviewOfServerChanges() --> Send "connected" status + tools
    else Connect Fails
        McpHub->>McpHub: Update status to "disconnected" / "error"
        McpHub->>McpHub: notifyWebviewOfServerChanges() --> Send error status
    end

    Note over VSCodeAPI, ExtStateCtx, McpView: Status updates flow via postMessage -> context -> UI re-render

```

## Modification Guidance

Modifications might involve adding more management features, displaying more details, or changing the layout.

1.  **Adding Server Logs Display:**
    *   **Host (`McpHub`):** Modify `StdioClientTransport` (or add listeners in `McpHub`) to buffer recent `stderr` lines for each server connection. Include the latest lines (e.g., last 50) in the `McpServer` object sent to the UI (add `logs?: string[]` to the type).
    *   **UI (`McpView.tsx`):** In `renderServer`'s `AccordionContent`, add a new tab in `VSCodePanels` for "Logs". Render the `server.logs` array within a `<pre>` tag or using `CodeBlock`.

2.  **Providing Action to Edit Server Config Directly:**
    *   **UI (`McpView.tsx`):** Add an "Edit Config" button next to Restart/Delete.
    *   **Message:** On click, send a new message type, e.g., `{ type: "editMcpServerConfig", serverName, source }`.
    *   **Host:** Add handler in `webviewMessageHandler`. Determine the correct file path (global or project) based on `source`. Use `vscode.workspace.openTextDocument` and `vscode.window.showTextDocument` to open the JSON file. Potentially try to navigate the editor to the specific server's entry within the JSON.

3.  **Displaying Resource Content Preview:**
    *   **UI (`McpResourceRow.tsx`):** Add a "Preview" button.
    *   **Message:** On click, send `{ type: "previewMcpResource", serverName, source, uri }`.
    *   **Host:** Handler calls `McpHub.readResource`. Receives the content. Format the content appropriately (e.g., if JSON, pretty-print; if text, show as is) and send it back via a new message (e.g., `{ type: "mcpResourcePreview", content: "..." }`).
    *   **UI (`McpView.tsx`):** Add state to hold preview content and show it in a dialog or inline when the response message arrives.

**Best Practices:**

*   **Clear Status:** Ensure the `ConnectionStatus` component and error messages provide unambiguous feedback about each server's state.
*   **Scoped Actions:** Actions like Delete, Restart, Enable/Disable, Toggle Always Allow must correctly target the server based on both `serverName` and `source` (global vs. project) to modify the right configuration or connection.
*   **Feedback:** Provide immediate visual feedback where possible (e.g., updating checkbox state in `McpEnabledToggle` locally before host confirmation) but rely on the host sending back the authoritative state for most updates. Use loading states (disabling buttons) for asynchronous actions like Restart.
*   **Consistency:** Use consistent terminology and layout across different server types and sections.

**Potential Pitfalls:**

*   **Stale UI Data:** If the host fails to send updates after an action (e.g., restart fails silently, config file write error), the UI might show an outdated status. Robust error handling and state synchronization in `McpHub` are crucial.
*   **Confusing Project vs. Global:** Ensure the UI clearly distinguishes between project and global servers, especially if they have the same name (though project should take precedence). Actions should target the correct scope.
*   **Complexity:** Displaying complex input/output schemas for tools or resources directly in the UI can become cluttered. Use tooltips or collapsible sections.
*   **Error Detail:** Connection or tool call errors might have complex causes. The UI might only show a summary; users may need the Output Channel for full details logged by `McpHub`.

## Conclusion

The MCP UI Components provide an essential control panel for managing Roo-Code's integration with external Model Context Protocol servers. Through components like `McpView`, `McpEnabledToggle`, `McpToolRow`, and `McpResourceRow`, users can monitor connection status, inspect available capabilities, manage permissions, and control server lifecycles directly within the familiar WebView interface. This UI relies on synchronized state (`mcpServers`) from the backend `McpHub` and uses themed primitives to offer a clear, consistent, and functional management experience.

This chapter concludes our exploration of specific UI component categories within the WebView. The next chapter discusses a specialized UI element used for interactions requiring direct human input within an AI task flow: [Chapter 39: Human Relay UI](39_human_relay_ui.md).


