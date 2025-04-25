# Chapter 53: Evals System

Continuing from [Chapter 52: TelemetryService](52_telemetryservice.md), which focused on collecting anonymous usage data for product improvement, this chapter introduces a distinct system designed for more structured and objective assessment of Roo-Code's capabilities: the **Evals System**, located primarily within the `evals/` monorepo directory.

## Motivation: Measuring and Improving AI Performance

While user feedback and telemetry provide valuable insights, objectively measuring the quality and performance of an AI assistant like Roo-Code requires a more systematic approach. We need a way to:

1.  **Define Test Cases:** Create a standardized set of realistic tasks (prompts, context, expected outcomes) that represent common user scenarios (e.g., "refactor this function", "explain this error", "write a unit test", "use tool X").
2.  **Run Automatically:** Execute Roo-Code against these test cases using different models, configurations, or code versions in a repeatable manner, independent of the interactive VS Code UI.
3.  **Capture Outputs:** Record the AI's full interaction transcript, including generated text, code modifications requested via tools, actual tool usage sequences, token counts, and costs for each test case.
4.  **Evaluate Results:** Compare the captured outputs against predefined criteria (or using subsequent LLM-based evaluation or manual review) to assess performance metrics like correctness, helpfulness, efficiency (token usage, cost, time), safety (avoiding harmful actions), and adherence to instructions.
5.  **Track Progress:** Monitor how changes to prompts ([Chapter 7: SystemPrompt](07_systemprompt.md)), models, context strategies ([Chapter 23: Sliding Window Context Management](23_sliding_window_context_management.md)), or tool implementations ([Chapter 8: Tools](08_tools.md)) affect performance over time, preventing regressions and guiding improvements.

The **Evals System** (`evals/` directory) provides the framework and tooling to define, run, and analyze these evaluation tasks, enabling data-driven development and quality assurance for Roo-Code's core AI functionality. It is a separate system composed of multiple packages and applications within the monorepo.

**Central Use Case:** A developer modifies the main system prompt ([Chapter 7: SystemPrompt](07_systemprompt.md)) and wants to ensure it doesn't negatively impact code generation quality for Python.

1.  **Define Eval Suite:** An `evals/suites/python_codegen.yaml` file defines several test cases, each validated by `evalTaskSchema` from `@evals/types`:
    *   `id`: `py_fib_recursive`
    *   `prompt`: "Write a Python function to calculate Fibonacci numbers recursively."
    *   `criteria`: "Function should be recursive, handle base cases (0, 1), and be syntactically correct Python."
    *   `config_overrides`: `{ "mode": "code" }`
2.  **Run Evals:** Developer runs `pnpm start --suite python_codegen --model gpt-4-turbo` from `evals/apps/cli/`.
3.  **Eval CLI Runner (`cli/src/index.ts` / `runExercise`):**
    *   Parses args (`gluegun`). Finds `python_codegen.yaml`. Creates run/task records in the database (`@evals/db`).
    *   Starts an IPC server (`IpcServer` from `@evals/ipc`) on a unique socket path.
    *   Launches a new VS Code instance (`execa code ...`) with the Roo-Code extension path, the exercise workspace (e.g., from `evals/exercises/`), and the `ROO_CODE_IPC_SOCKET_PATH` environment variable pointing to the IPC server's socket.
    *   Waits for the Roo-Code instance (acting as an IPC client) to connect back.
    *   Sends a `StartNewTask` command (IPC message) with prompt and config overrides. **Requires the necessary API key (e.g., `OPENAI_API_KEY`) to be available in the CLI runner's environment.**
    *   Receives a stream of `TaskEvent` messages (IPC) from Roo-Code, representing the conversation transcript and metrics. Updates the task record in the database.
    *   Optionally runs unit tests (`runUnitTest`) against generated code. Updates `passed` status in DB. Sends `Pass`/`Fail` event via IPC.
4.  **Review Results:** Developer uses the `evals-web` UI ([Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md)), which reads from the `@evals/db`, to view the task transcript, metrics, and pass/fail status, comparing the result against the `criteria`.
5.  **Analysis:** Assess if the system prompt change improved or degraded performance.

## Key Concepts

1.  **Monorepo Structure (`evals/`):** The system resides in its own directory, potentially structured as a monorepo with sub-packages:
    *   `apps/cli`: The command-line interface (CLI) application for running evaluations.
    *   `apps/web`: The Next.js/React web application for viewing results.
    *   `packages/db`: Database schema (using Drizzle ORM, likely SQLite or Turso) and query functions for storing run/task data.
    *   `packages/ipc`: Wrappers around `node-ipc` for communication between CLI and VS Code extension instance.
    *   `packages/types`: Shared Zod schemas ([Chapter 40: Schemas (Zod)](40_schemas__zod_.md)) and TypeScript types for tasks, outputs, configuration, and IPC messages (`evalTaskSchema`, `evalOutputSchema`, `ipcMessageSchema`, etc.).
    *   `suites/`: Directory containing YAML files defining evaluation tasks.
    *   `exercises/`: Directory containing workspace contexts (code files, project structures) used by evaluation tasks. Often a Git submodule like `Roo-Code-Benchmark`.
    *   `output/` (Optional): Directory where raw JSON outputs might be saved if not using a database exclusively.
    *   `lib/`: Shared script utilities ([Chapter 55: Script Utilities](55_script_utilities.md)) specific to the evals system.

2.  **Evaluation Suites (`evals/suites/*.yaml`):** Define tasks (`EvalTask` structure). Key fields: `id`, `prompt`, `context` (files to pre-load), `criteria` (success definition), `config_overrides` (temporary settings like model ID or mode). Loaded by `evals/apps/cli/src/exercises.ts`.

3.  **Eval CLI Runner (`evals/apps/cli/src/index.ts`):** The Node.js application that orchestrates the evaluation run.
    *   Uses `gluegun` or `yargs` for command-line arguments (`--suite`, `--model`, `--runId`, etc.).
    *   Interacts with `@evals/db` to create `Run` and `Task` records.
    *   Manages an `IpcServer` (`@evals/ipc`) to communicate with VS Code instances.
    *   Launches VS Code instances for each task using `execa`, passing the extension path, exercise workspace path, and IPC socket path (`ROO_CODE_IPC_SOCKET_PATH`).
    *   Connects an `IpcClient` to the launched VS Code instance.
    *   Sends `StartNewTask` command via IPC.
    *   Receives `TaskEvent` stream via IPC, updating the corresponding `Task` record in the database with the transcript, metrics, and status.
    *   Optionally runs unit tests (`runUnitTest`) using `execa` and updates the task's `passed` status.

4.  **IPC (`@evals/ipc`, `src/exports/ipc.ts`):** The communication backbone between the CLI runner and the Roo-Code extension instance.
    *   Uses `node-ipc` over Unix Domain Sockets or Windows Named Pipes.
    *   CLI acts as `IpcServer`, Extension acts as `IpcClient`.
    *   Uses custom JSON message types defined and validated by Zod schemas (`ipcMessageSchema`, `taskCommandSchema`, `taskEventSchema`) in `@evals/types`. (See [Chapter 51: IPC (Inter-Process Communication)](51_ipc__inter_process_communication_.md)).

5.  **Programmatic Invocation (Roo-Code Host):** The Roo-Code extension host needs logic (likely in `extension.ts` or `src/exports/`) to:
    *   Detect the `ROO_CODE_IPC_SOCKET_PATH` environment variable on activation.
    *   If detected, initialize an `IpcClient` and connect to the CLI server.
    *   Listen for `TaskCommand` messages (especially `StartNewTask`) from the CLI server.
    *   Trigger the core task execution logic (e.g., `API.startNewTask` -> `ClineProvider.initClineWithTask`) using the prompt and configuration received via IPC. **Crucially, dependencies on VS Code UI elements (like `askApproval` prompts) must be mocked or bypassed** (e.g., default to 'yes' or throw error). Access to workspace files should use paths relative to the workspace opened by the runner. `ContextProxy` might need to be initialized with overrides.
    *   Intercept internal events emitted by `Cline` (via `API.emit` override or similar) and forward them as `TaskEvent` messages back to the CLI server via the `IpcClient`.

6.  **Database (`@evals/db`):** Likely uses Prisma or Drizzle ORM with SQLite (for simplicity/portability) or Turso (for cloud synchronization). Defines schema (`schema.ts`) for `runs`, `tasks`, `taskMetrics`, `toolErrors`. Provides functions for CRUD operations. Stores results persistently.

7.  **Web UI (`evals-web/`):** A separate Next.js/React application.
    *   Fetches data from the database (`@evals/db`).
    *   Provides pages for browsing runs, suites, tasks, and comparing results.
    *   Renders transcripts (using Markdown rendering components, [Chapter 46: Markdown Rendering](46_markdown_rendering.md)), metrics, status, and configuration details.
    *   Uses standard web shadcn/ui primitives ([Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md)) for layout and components (`Table`, `Card`, `Badge`, etc.).

## Using the Evals System

Primarily a developer tool for automated testing and regression analysis.

**Workflow:**

1.  **Define Suites:** Create/edit `evals/suites/*.yaml` files with tasks relevant to the feature/change being tested. Ensure corresponding exercise files/projects exist in `evals/exercises/`.
2.  **Configure Runner:** Make sure API keys for the models specified in suites or via `--model` are available as environment variables (e.g., `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`). Configure database connection if needed (e.g., `TURSO_CONNECTION_URL`, `TURSO_AUTH_TOKEN` or `BENCHMARKS_DB_PATH`).
3.  **Run Runner:** From the `evals/apps/cli/` directory:
    ```bash
    # Example: Run specific exercises in the 'javascript' language group
    pnpm start --language javascript --exercise exercise-1 --exercise exercise-2 --model gpt-4-turbo

    # Example: Run all exercises for python
    pnpm start --language python --exercise all --model claude-3-sonnet

    # Example: Re-run specific tasks from a previous run
    pnpm start --runId 123 --taskId 45 --taskId 46
    ```
4.  **Monitor:** Observe the CLI output logging the progress of each task (launching VS Code, connecting, receiving events, running tests, completion status).
5.  **Analyze Results:** Start the web UI (`pnpm dev` within `evals-web/`) and navigate to the relevant run/task results page. Examine transcripts, metrics, pass/fail status, and compare against criteria or previous runs.
6.  **Iterate:** Modify Roo-Code source (prompts, core logic), rebuild the extension (`pnpm build:prod` in root), re-run the relevant evaluations, and analyze the new results.

## Code Walkthrough

### Eval CLI (`evals/apps/cli/src/index.ts`)

*(See code provided in Chapter 51)*
*   Uses `gluegun` for CLI commands (`run`).
*   Interacts heavily with `@evals/db` for state management (`findRun`, `createRun`, `createTask`, `updateTask`, etc.).
*   Creates `IpcServer` (`@evals/ipc`).
*   `runExercise` function orchestrates launching VS Code via `execa` (passing `ROO_CODE_IPC_SOCKET_PATH`), creating an `IpcClient` to connect to it, sending `StartNewTask`, listening for `TaskEvent`s (updating DB), optionally running `runUnitTest` (using `execa`, `ps-tree`), and cleaning up. Uses `pMap` for concurrency across tasks.

### IPC Library (`@evals/ipc`, `@evals/types`)

*(See code provided in Chapter 51)*
*   **Types (`ipc.ts`):** Defines Zod schemas and TS types for all IPC messages (`Ack`, `TaskCommand`, `TaskEvent`, `IpcMessage`). Crucial for type safety and validation.
*   **Client (`client.ts`):** `IpcClient` wraps `node-ipc`, connects to server, validates incoming messages, emits typed events.
*   **Server (`server.ts`):** `IpcServer` wraps `node-ipc`, listens, manages clients, validates incoming messages, emits typed events.

### Roo-Code Host IPC Client (`src/exports/api.ts` / `src/exports/ipc.ts`)

*(See corrected conceptual code in Chapter 51)*
*   **Initialization (`extension.ts`):** Checks `process.env.ROO_CODE_IPC_SOCKET_PATH`, creates `IpcClient`.
*   **Command Handling:** Listens for `TaskCommand` (`StartNewTask`), validates, triggers `API.startNewTask`. **Requires applying config overrides** from the message data to the task being started.
*   **Event Forwarding:** `API.emit` override intercepts core events, formats/validates as `TaskEvent`, sends back to CLI server via `ipcClient.sendMessage`.

### Database (`@evals/db`)

*   **Schema (`schema.ts`):** Defines tables (`runs`, `tasks`, `taskMetrics`, `toolErrors`) using Drizzle ORM syntax. Includes types (`Run`, `Task`, etc.), relations, and insert schemas derived using `drizzle-zod`. Uses Zod schemas (`rooCodeSettingsSchema`, `toolUsageSchema`) from `@evals/types` for JSON blob columns.
*   **Client (`db.ts`):** Configures the Drizzle client connection using environment variables for Turso (`TURSO_CONNECTION_URL`, `TURSO_AUTH_TOKEN`) or a local SQLite file (`BENCHMARKS_DB_PATH`).
*   **Queries (`index.ts`):** Exports functions (`createRun`, `getTasks`, `updateTask`, etc.) that use the Drizzle client (`db`) to perform specific database operations.

### Web UI (`evals-web/`)

*   Separate Next.js app.
*   Uses pages (e.g., `/runs/[runId]/tasks/[taskId]`) to fetch data from `@evals/db` (likely in `getServerSideProps`).
*   Renders data using React components and shadcn/ui primitives ([Chapter 54](54_shadcn_ui_primitives__evals_web_.md)) (`Table`, `Card`, `Badge`). Includes components for displaying transcripts with Markdown/code rendering.

## Internal Implementation

1.  **CLI Runner:** Starts IPC server, launches VS Code instance with env var pointing to server socket.
2.  **VS Code Host:** Detects env var, starts IPC client, connects to CLI server, receives Ack.
3.  **CLI Runner:** Sends `StartNewTask` command (IPC).
4.  **VS Code Host:** Receives command, validates, applies overrides, calls `API.startNewTask` -> `ClineProvider.initClineWithTask` (using **mocked** dependencies where necessary, e.g., `askApproval` might always return true).
5.  **`Cline` Execution:** Runs task loop. Makes API calls (using keys from env vars provided to runner). Emits events.
6.  **VS Code Host (`API.emit`):** Intercepts events, validates payload against `taskEventSchema`, sends `TaskEvent` (IPC) to CLI server.
7.  **CLI Runner:** Receives `TaskEvent`s, validates, updates database record for the task (status, transcript fragments, metrics).
8.  **Completion:** `TaskCompleted` event received. CLI updates final status in DB. Runs `runUnitTest`. Updates `passed` status in DB. Sends `Pass`/`Fail` event (IPC).
9.  **Cleanup:** CLI sends `CloseTask`, disconnects client.
10. **Web UI:** Reads database, renders results.

**Sequence Diagram (Eval Task Execution with IPC):**

*(See refined diagram in Chapter 53 - Internal Implementation)*

## Modification Guidance

Modifications typically involve adding new suite features, runner options, evaluation metrics, or enhancing the analysis UI or core logic invocation.

1.  **Adding New Task Context (e.g., Specific VS Code Setting):**
    *   **Schema (`evalTaskSchema`):** Add `vscode_settings?: { [key: string]: any }` to `context`.
    *   **YAML:** Add `context: { vscode_settings: { "editor.tabSize": 2 } }` to tasks.
    *   **`evaluateTask.ts` / Core Logic Invocation:** Modify context preparation to write these settings to a temporary workspace settings file (`.vscode/settings.json`) before launching VS Code for that task. Ensure cleanup removes this file.
    *   **Core Logic:** Roo-Code running in the VS Code instance will pick up these temporary workspace settings.

2.  **Adding a New Automated Metric (e.g., Hallucination Check):**
    *   **`evaluateTask.ts`:** After capturing transcript, implement logic (potentially using another LLM call via `getCompletions`) to check the AI's response for factual inaccuracies based on the provided context.
    *   Add `hallucination_check: { passed: boolean, details?: string }` to metrics stored in DB.
    *   **Schema/DB:** Add field to `taskMetrics` schema/table.
    *   **Analysis/UI:** Display/use the new metric.

3.  **Improving Programmatic Invocation/Mocking:**
    *   Refine the `runRooCodeTaskForEval` entry point or the mocking strategy used when instantiating `Cline` directly.
    *   Improve mocks for VS Code APIs (`window`, `workspace`, `commands`) to more accurately simulate behavior needed by evals (e.g., mock `showQuickPick` to return a predefined value for tool testing).
    *   Ensure configuration overrides (model, mode, API keys, other settings) are correctly applied within the mocked/programmatic environment.

**Best Practices:**

*   **Clear Criteria:** Define specific, verifiable success criteria in suites.
*   **Realistic Context:** Use representative prompts/context. Manage via temp files/dirs.
*   **Isolate Core Logic:** Refine the programmatic entry point (`runRooCodeTaskForEval`) to minimize direct VS Code API dependencies needed during evals.
*   **Robust Mocking/Bypass:** Mock VS Code dependencies accurately or bypass UI-dependent logic (like `askApproval`) predictably.
*   **Structured Output:** Use database (`@evals/db`) and schemas (`evalOutputSchema`, DB schemas) for consistent results. Store inputs+outputs.
*   **Version Control Suites:** Store YAMLs in Git.
*   **Reproducibility:** Record config, model ID. Fix model versions if possible.
*   **Balance Automated/Manual:** Use automated metrics (`runUnitTest`) + manual review via Web UI.
*   **Secure API Keys:** Use environment variables (`process.env`) for the CLI runner.

**Potential Pitfalls:**

*   **Environment Differences:** Evals outside VS Code behaving differently due to inaccurate mocking. **This is a significant challenge.**
*   **Mocking Complexity:** Mocking `Cline`'s full environment is hard.
*   **Context Setup/Cleanup Errors.**
*   **API Keys/Costs:** Secure management. Cost of runs.
*   **Flaky LLMs/Tests:** Non-determinism.
*   **IPC Failures:** Connection issues, validation errors, timeouts.
*   **Eval Suite Maintenance.**
*   **Database Schema Evolution:** Requires migrations for `@evals/db` if structure changes.

## Conclusion

The Evals System provides an essential framework for systematically evaluating and improving the performance of Roo-Code's AI capabilities. By defining structured test suites (YAML), using a CLI runner script (`runEval.ts`, `evaluateTask.ts`) to execute tasks programmatically (launching VS Code and communicating via IPC using `@evals/ipc`), storing detailed results in a database (`@evals/db`), and providing a web UI (`evals-web/`) for analysis, developers can objectively measure correctness, prevent regressions, and guide future improvements. While setting up the programmatic invocation and IPC requires careful handling of dependencies, mocking, and error states, the benefits of automated, data-driven quality assessment are significant for developing a reliable AI assistant.

Next, we look at the UI primitives specifically used within the Evals web interface: [Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md).
---
# Chapter 54: Shadcn/UI Primitives (Evals Web)

Continuing from [Chapter 53: Evals System](53_evals_system.md), which detailed the framework for running automated evaluations of Roo-Code's AI, this chapter focuses on the user interface components used within the **Evals Web** application – the separate web interface designed for reviewing and analyzing evaluation results. Specifically, we look at the **Shadcn/UI Primitives** used in this context.

## Motivation: Building a Dedicated Web UI for Eval Analysis

While the main Roo-Code extension UI ([Chapter 1: WebView UI](01_webview_ui.md)) runs within VS Code and needs to match its theme precisely, the Evals Web UI (`evals-web/`) is a standalone web application (built with Next.js/React) running in a standard browser. Its purpose is different: presenting potentially large amounts of structured evaluation data (results from the `@evals/db` database), enabling comparisons between different runs, filtering, sorting, and potentially annotating results.

This separate context allows for different UI choices. Building a clean, modern, and functional web interface requires a good set of UI components. `shadcn/ui` is chosen for this purpose, used in a standard web configuration.

The reasons for using `shadcn/ui` here are:
1.  **Rich Component Set:** Provides essential components for data-rich web apps: `Table`, `Dialog`, `Select`, `Tabs`, `Card`, `Button`, `Badge`, `Tooltip`, `Accordion`, `Input`, `Checkbox`, `Separator`, `DropdownMenu`, `Popover`, etc.
2.  **Composability & Customization:** Components are easily composed and fully customizable via source code in `evals-web/components/ui/`.
3.  **Modern Styling (Tailwind CSS):** Uses Tailwind CSS for styling with a standard web theme (light/dark modes via CSS variables defined in `globals.css`), independent of VS Code theming.
4.  **Accessibility:** Built on Radix UI primitives.

**Central Use Case:** Displaying a table comparing results for the same task across different models.

1.  Evals Web app fetches results (`TaskWithRun[]`) from `@evals/db`.
2.  A React component renders the comparison using `Table` components from `@/components/ui`.
3.  Columns show Model, Status, Cost, Tokens, Actions.
4.  Data mapped to `TableRow`, rendered using `TableCell`.
5.  `Badge` displays status. `Tooltip` shows cost details. `Button` + Next.js `Link` navigate to detail view.
6.  Table uses standard web theme styling from Tailwind config.

## Key Concepts

1.  **Standalone Web Application (`evals-web/`):** Next.js/React app. Reads data from `@evals/db`. Separate styling.
2.  **Shadcn/UI Primitives (`evals-web/components/ui/`):** Standard shadcn/ui component source code ([https://ui.shadcn.com/](https://ui.shadcn.com/)).
3.  **Standard Web Styling:** Uses Tailwind CSS + standard web theme (CSS variables in `globals.css`). **Not** VS Code variables.
4.  **Radix UI & Tailwind CSS:** Foundation for behavior/accessibility and styling.
5.  **Component Set:** `Table`, `Card`, `Badge`, `Tabs`, `Select`, `Dialog`, `Tooltip`, `Button`, `Input`, etc.
6.  **`cn` Utility:** Standard `clsx` + `tailwind-merge` helper.

## Using the Primitives in Evals Web

Standard React development. Import from `@/components/ui`.

**Example: Rendering a Comparison Table (Conceptual)**

```typescript
// --- File: evals-web/components/results/ComparisonTable.tsx ---
import React from 'react';
import {
  Table, TableBody, TableCell, TableHead, TableHeader, TableRow,
} from '@/components/ui/table'; // Import Table
import { Badge } from '@/components/ui/badge'; // Import Badge
import { Button } from '@/components/ui/button'; // Import Button
import { Tooltip, TooltipContent, TooltipProvider, TooltipTrigger } from '@/components/ui/tooltip'; // Import Tooltip
import Link from 'next/link'; // Import Next.js Link
import type { TaskWithRun } from '@evals/db'; // Import DB type

interface ComparisonTableProps { tasks: TaskWithRun[]; }

const ComparisonTable: React.FC<ComparisonTableProps> = ({ tasks }) => {
  // Formatters...
  const formatCost = (cost?: number | null) => /* ... */;
  const formatTokens = (tokens?: number | null) => /* ... */;
  const getStatusVariant = (task: TaskWithRun): "default" | "destructive" | "secondary" | "outline" => { /* ... */ };
  const getStatusText = (task: TaskWithRun): string => { /* ... */ };

  return (
    <TooltipProvider>
      <Table>
        <TableHeader> {/* ... Headers ... */} </TableHeader>
        <TableBody>
          {tasks.map((task) => {
            const metrics = task.metrics?.[0];
            const modelId = task.run?.model ?? 'Unknown';
            const resultKey = task.id;
            const resultDetailPath = `/runs/${task.runId}/tasks/${task.id}`;

            return (
              <TableRow key={resultKey}>
                <TableCell className="font-medium">{modelId}</TableCell>
                <TableCell> <Badge variant={getStatusVariant(task)}>{getStatusText(task)}</Badge> </TableCell>
                <TableCell className="text-right">
                  <Tooltip>
                    <TooltipTrigger asChild><button type="button" className="cursor-default">{formatCost(metrics?.cost)}</button></TooltipTrigger>
                    <TooltipContent>{/* Cost details */}</TooltipContent>
                  </Tooltip>
                </TableCell>
                <TableCell className="text-right">{formatTokens(metrics?.tokensIn)}</TableCell>
                <TableCell className="text-right">{formatTokens(metrics?.tokensOut)}</TableCell>
                <TableCell className="text-right">{metrics?.duration ? `${(metrics.duration / 1000).toFixed(1)}s` : 'N/A'}</TableCell>
                <TableCell>
                  <Link href={resultDetailPath} passHref legacyBehavior>
                    <Button variant="outline" size="sm" asChild><a>View Details</a></Button>
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

1.  **Imports:** Imports primitives from `@/components/ui` within `evals-web`. Imports DB type from `@evals/db`.
2.  **Composition:** Uses `Table`, `Badge`, `Tooltip`, `Button`.
3.  **Styling:** Uses Tailwind utilities + `Badge` variants based on web theme.
4.  **Data Mapping:** Maps `tasks` array (`TaskWithRun[]`). Accesses data from `task`, `task.run`, `task.metrics`. Formats values.
5.  **Interaction:** Uses Next.js `Link` + `Button` for navigation.

## Code Walkthrough

Components in `evals-web/components/ui/` are standard shadcn/ui implementations.

### Example Primitive (`evals-web/components/ui/badge.tsx`)

*(See code in Chapter 54 - Code Walkthrough)*
*   Standard shadcn/ui structure. Uses semantic Tailwind classes (`bg-primary`, etc.) mapping to the web theme.

### Tailwind Configuration (`evals-web/tailwind.config.js` - Conceptual)

*(See code in Chapter 54 - Key Concepts)*
*   Defines standard web color palette using CSS variables defined in `globals.css`. **Not** VS Code variables.

## Internal Implementation

Standard React rendering with Tailwind CSS: Component -> CVA/`cn` -> HTML with classes -> Tailwind generates CSS -> Browser uses CSS variables from `globals.css` -> Renders standard web UI element.

## Modification Guidance

Follow standard web dev practices for shadcn/ui and Tailwind within `evals-web`.

1.  **Changing Evals Web Theme:** Edit HSL values in `globals.css` / semantic mappings in `evals-web/tailwind.config.js`. Rebuild CSS.
2.  **Adding/Adapting Primitives:** Use `npx shadcn-ui@latest add ...` or copy source into `evals-web/components/ui/`. Standard components work directly with the web theme. Update Tailwind config if needed.

**Best Practices:**

*   **Standard Shadcn/UI:** Use components as documented.
*   **Theme via CSS Variables:** Define palette in `globals.css`.
*   **Tailwind Configuration:** Ensure correct setup for `evals-web`.
*   **Keep Separate:** Maintain distinction from WebView UI components/theme.

**Potential Pitfalls:**

*   **Incorrect Tailwind Setup.**
*   **CSS Conflicts.**
*   **Build Process Integration** (Tailwind with Next.js/Vite).
*   **Dependency Mismatches** within `evals-web`.

## Conclusion

The shadcn/ui primitives provide a modern, flexible, and efficient way to build the user interface for the standalone **Evals Web** application (`evals-web/`). By adopting the standard shadcn/ui components and styling them with a conventional web-oriented Tailwind CSS theme, the `evals-web` project creates sophisticated data displays and interactions necessary for analyzing and comparing AI evaluation results retrieved from the `@evals/db`. These components are intentionally styled differently from their counterparts in the main VS Code WebView, reflecting their different execution context and purpose.

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
// Assuming utility exists in evals/src/lib for this specific need
import { findYamlFiles } from "../lib/fileUtils";
import { scriptLogger as logger } from "../lib/logger"; // Assuming logger utility

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
    *   **Shell Execution:** `child_process`, `execa` (preferred).
    *   **Logging:** Console logger (`console`) enhanced with `chalk`.
    *   **Path Manipulation:** Node.js `path`. Reuses Node-compatible `src/utils/path.ts` helpers ([Chapter 43](43_path_utilities.md)). Base paths from `process.cwd()` or args.
    *   **Concurrency:** `p-limit`.
    *   **JSON/YAML:** `JSON`, `js-yaml`, Zod validation ([Chapter 40](40_schemas__zod_.md)).
3.  **Location:** `scripts/lib/` (build/dev), `evals/src/lib/` (eval-specific), shared `src/utils/` (only if Node-compatible).
4.  **Reusability:** Imported across script files (`*.js`, `*.ts`). TS scripts run via `ts-node` or compiled JS.

## Using Script Utilities

Imported as standard functions/modules within script files.

*   **Argument Parsing (`evals/src/runEval.ts`):** Uses `yargs`. (Ch 53)
*   **Finding Files (`findYamlFiles` utility):** Uses `fs.stat`, `glob`. (Ch 55 Concepts)
*   **Executing Commands (`runCommand` utility):** Uses `execa`. (Ch 55 Concepts)
*   **Logging (`scriptLogger` utility):** Uses `chalk`. (Ch 55 Concepts)
*   **Reading/Validating JSON (`readAndValidateJson` utility):** Uses `safeReadFile`, `JSON.parse`, Zod `safeParse`. (Ch 55 Concepts)

## Code Walkthrough

Examining utilities potentially used or defined within the project structure.

### Shared Utilities (`src/utils/`)

*   **`fs.ts` ([Chapter 42]):** `fileExistsAtPath`, `directoryExistsAtPath`, `safeWriteFile`, `createDirectoriesForFile`, `safeReadFile` are usable (require absolute paths).
*   **`path.ts` ([Chapter 43]):** `normalizePath`, `toPosix()`, `arePathsEqual`, `relativePath`/`toRelativePath` (require `cwd`), `getReadablePath` (require `cwd`) are usable. `getWorkspacePath` **is NOT usable**.
*   **`logging.ts`:** `logger` needs configuration for console output in scripts, or use dedicated `scriptLogger`.
*   **`text-normalization.ts` ([Chapter 45]):** Pure string functions are usable.

### Evals Utilities (`evals/src/lib/`)

*   **`openai.ts`:** `getCompletions` using `axios`. Network utility.
*   **`findSuites.ts` (Conceptual `findYamlFiles`):** Uses `fs.stat`, `path.resolve`, `glob`. File system utility.
*   **`contextUtils.ts` (Conceptual):** Uses `fs/promises`, `path` for temp eval files.

### Build Script Utilities (`scripts/lib/`)

*   **`copyWasm.js` (Conceptual):** `esbuild` plugin logic using Node.js `fs` ([Chapter 56]).
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
*   `build` function: Cleans `dist`, builds host (`esbuild`), conditionally builds WebView (`vite build` via `execSync`), conditionally copies Vite output, calls `copyAssets`. Handles `isWatch` flag.

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

