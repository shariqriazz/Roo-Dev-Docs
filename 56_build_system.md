# Chapter 56: Build System

Continuing from [Chapter 55: Script Utilities](55_script_utilities.md), which covered helpers used in various project scripts, this chapter focuses on the specific processes and tools used to compile, bundle, and package the Roo-Code extension for distribution and development: the **Build System**.

## Motivation: Transforming Source Code into a Deployable Extension

The Roo-Code source code, written primarily in TypeScript (`src/`) with React for the WebView UI (`webview-ui/`) and including assets like images (`images/`), locale files ([Chapter 50: Localization System (i18n)](50_localization_system__i18n_.md)) (`public/locales`, `src/i18n/locales`), NLS files (`package.nls.*.json`), and WASM modules ([Chapter 17: Tree-sitter Integration](17_tree_sitter_integration.md)) (`node_modules/`), needs to be transformed into a format that VS Code can load and execute. This isn't just simple compilation; it involves several critical steps:

1.  **TypeScript Compilation:** Converting TypeScript code (for both the extension host `src/` and the WebView UI `webview-ui/src/`) into executable JavaScript.
2.  **Bundling:** Combining potentially hundreds of source files and their dependencies from `node_modules` into a small number of optimized bundles. This improves the extension's startup time and reduces its overall size.
3.  **Asset Handling:** Ensuring all necessary non-code assets (locale JSON files, NLS files for `package.json`, WASM parser files, images, the compiled WebView UI) are copied to the correct locations within the final distribution directory (`dist/`) so the extension can find them at runtime.
4.  **Environment Configuration:** Injecting build-time variables, such as setting `NODE_ENV` to differentiate between development and production builds, or embedding API keys for services like telemetry ([Chapter 52: TelemetryService](52_telemetryservice.md)) if sourced from the build environment.
5.  **Development vs. Production Builds:** Creating optimized, minified builds for production release versus development builds with source maps for easier debugging and potentially faster incremental rebuilds using watch modes.
6.  **WebView UI Build Integration:** Orchestrating the separate build process for the React-based WebView UI, which uses Vite ([Chapter 1: WebView UI](01_webview_ui.md)), and incorporating its output into the main extension package.
7.  **Packaging Preparation:** Producing a final `dist/` directory with a structure that the `vsce` (Visual Studio Code Extensions) command-line tool can correctly package into a `.vsix` file for installation or publishing to the Marketplace.

Manually performing these steps is complex, time-consuming, and highly error-prone. A dedicated, automated **Build System** is essential for consistency, efficiency, and correctness. Roo-Code achieves this by combining the strengths of two modern build tools: `esbuild` for its exceptional speed in bundling the Node.js-based extension host code, and Vite for its rich feature set and development experience for the React-based WebView UI. These tools are orchestrated via custom Node.js scripts (primarily `esbuild.js`) and managed through `pnpm` scripts defined in `package.json`.

**Central Use Case:** A developer needs to create an optimized, production-ready `.vsix` package for release.

1.  **Command:** The developer runs `pnpm run package` (or potentially `pnpm run vscode:prepublish` followed by `vsce package`).
2.  **Pre-publish Hook:** The `vscode:prepublish` script triggers, first running `pnpm run compile` (`tsc --noEmit` for type checking) and then `pnpm run build:prod`.
3.  **`build:prod` Execution:** This executes `pnpm clean && pnpm prepare && cross-env NODE_ENV=production node ./esbuild.js`.
4.  **`clean`:** `rimraf dist webview-ui/build` removes previous build artifacts.
5.  **`prepare`:** `node ./scripts/prepare.js` runs any pre-build code generation (e.g., schema types).
6.  **`esbuild.js` (Production Mode):**
    *   Detects `NODE_ENV=production`.
    *   **Host Build:** Configures and runs `esbuild.build()` for `src/extension.ts` with `minify: true`, `sourcemap: false`, `platform: 'node'`, `format: 'cjs'`, `external: ['vscode']`, and production `define` values, outputting `dist/extension.js`.
    *   **WebView Build:** Executes `pnpm --filter webview-ui build` using `child_process.execSync`. This invokes Vite in production mode (`vite build`), creating optimized JS/CSS chunks and assets in `webview-ui/build/`.
    *   **Asset Copying (`copyAssets`):** Uses Node.js `fs` utilities ([Chapter 42: File System Utilities](42_file_system_utilities.md), [Chapter 55: Script Utilities](55_script_utilities.md)) to meticulously copy:
        *   `webview-ui/build/*` -> `dist/webview-ui/`
        *   WASM files (`tree-sitter-*.wasm`, `tree-sitter.wasm`) -> `dist/`
        *   WebView locale files (`public/locales/*`) -> `dist/locales/`
        *   Host locale files (`src/i18n/locales/*`) -> `dist/i18n/locales/`
        *   Host NLS files (`package.nls*.json`) -> `dist/`
        *   Images (`images/*`) -> `dist/images/`
7.  **`vsce package`:** The `pnpm package` script runs `vsce package ...`. The `vsce` tool reads `package.json`, uses `.vscodeignore` to determine which files from the project (primarily the contents of `dist/`, plus root files like `README.md`, `LICENSE`, `CHANGELOG.md`) should be included, and creates the final `roo-code.vsix` file in the `releases/` directory.

## Key Concepts

1.  **`esbuild`:** Extremely fast JavaScript/TypeScript bundler and minifier written in Go. Used via its Node.js API (`require('esbuild')`) within `esbuild.js` to bundle the **extension host** code. Key benefits are speed and simplicity for Node.js/CommonJS targets.
2.  **Vite:** A modern frontend build tool providing a fast development server (with HMR) and optimized production builds (using Rollup internally). Used for the **WebView UI** ([Chapter 1: WebView UI](01_webview_ui.md)). Configured via `webview-ui/vite.config.ts`.
3.  **Build Script (`esbuild.js`):** The primary Node.js orchestrator script.
    *   Detects build mode (production/development, watch) via `process.env.NODE_ENV` and `process.argv`.
    *   Configures and invokes `esbuild.build()` for the extension host.
    *   Invokes the separate Vite build process for the WebView UI using `child_process.execSync`.
    *   Contains or calls utility functions (`copyAssets`) using Node.js `fs` or script utilities ([Chapter 55](55_script_utilities.md)) to handle copying all necessary assets into the `dist` directory.
4.  **`package.json` Scripts:** Central location defining how build tasks are run. Uses `pnpm` workspace features (`--filter`) and tools like `cross-env` (for setting environment variables cross-platform) and `rimraf` (for cross-platform directory deletion). Defines lifecycle scripts like `prepare` and `vscode:prepublish`.
5.  **Entry Points & Outputs:**
    *   Host: `src/extension.ts` is the entry point bundled by `esbuild` into `dist/extension.js`.
    *   WebView: `webview-ui/src/index.tsx` is the entry point processed by Vite, producing assets in `webview-ui/build/` which are then copied to `dist/webview-ui/`.
6.  **Host Configuration (`esbuild.js`):** Specific `esbuild` options for the Node.js environment required by VS Code extensions:
    *   `platform: 'node'`: Target Node.js runtime.
    *   `format: 'cjs'`: Output CommonJS module format.
    *   `target: 'node16'`: Target a Node.js version compatible with VS Code's runtime.
    *   `external: ['vscode']`: **Crucial.** Tells `esbuild` *not* to bundle the `vscode` module, as it's provided by the VS Code environment at runtime.
    *   `bundle: true`: Enable bundling of dependencies.
    *   `minify: isProduction`: Enable code minification for production builds.
    *   `sourcemap: !isProduction`: Generate source maps (usually `'inline'`) for development builds to aid debugging.
    *   `define`: Replace global identifiers (like `process.env.NODE_ENV`) with static values at build time. Used for injecting build mode and potentially configuration like telemetry keys.
    *   `outfile`: Specifies the single output bundle file.
7.  **Asset Handling (`copyAssets` in `esbuild.js`):** A critical step manually performed by the build script *after* `esbuild` and Vite complete. Ensures all non-code files needed at runtime are present in the `dist` directory with the correct structure. This includes:
    *   WASM files for Tree-sitter ([Chapter 17: Tree-sitter Integration](17_tree_sitter_integration.md)) copied to `dist/`.
    *   WebView locale files from `public/locales` copied to `dist/locales/`.
    *   Extension host locale files from `src/i18n/locales` copied to `dist/i18n/locales/`.
    *   NLS files (`package.nls.*.json`) from project root copied to `dist/`.
    *   The entire contents of the Vite build output (`webview-ui/build/`) copied to `dist/webview-ui/`.
    *   Other static assets like images (`images/`) copied to `dist/images/`.
8.  **Development Mode (`watch:*` scripts):** Facilitates faster development cycles.
    *   `watch:host` (`esbuild.js --watch`): `esbuild` monitors `src/` files and quickly rebuilds `dist/extension.js` on changes.
    *   `watch:webview` (`vite dev`): Vite serves the WebView UI from memory with Hot Module Replacement (HMR), allowing UI changes to appear instantly without full rebuilds or extension reloads.
    *   These are typically run in separate terminals.
9.  **Production Mode (`build:prod` script):** Creates the final optimized build for distribution. Disables source maps, enables minification, runs Vite's production build (`vite build`), and copies all assets.
10. **Type Checking (`compile` script):** `tsc --noEmit` leverages the TypeScript compiler (`tsconfig.json`) to perform a thorough static type check across the entire codebase. This catches type errors that `esbuild`'s faster transpilation might miss. Essential for CI and pre-publish checks.
11. **Packaging (`vsce`, `package` script):** Uses the `vsce` tool. `vsce package` reads `package.json`, respects rules in `.vscodeignore` to exclude source code, tests, dev dependencies (`node_modules`), configuration files, etc., and bundles only the necessary runtime files (primarily the contents of `dist/` plus root metadata like `README.md`, `LICENSE`) into the `.vsix` archive.
12. **Preparation Script (`scripts/prepare.js`):** Executes before the main build steps. Used for tasks like code generation (e.g., generating types from Zod schemas if needed).

## Executing the Build

Managed via `pnpm` scripts defined in `package.json`.

```json
// --- File: package.json (Excerpt with Key Scripts) ---
{
  "scripts": {
    "install:all": "pnpm install && pnpm --filter webview-ui install", // Installs everywhere
    "prepare": "node ./scripts/prepare.js", // Runs pre-build tasks
    "compile": "tsc -p ./ --noEmit", // Type check only
    "clean": "rimraf dist webview-ui/build", // Clean output dirs
    "build:prod": "pnpm clean && pnpm prepare && cross-env NODE_ENV=production node ./esbuild.js", // Full production build
    "build:dev": "pnpm clean && pnpm prepare && cross-env NODE_ENV=development node ./esbuild.js", // Dev build (often used by watch)
    "watch:host": "pnpm build:dev --watch", // Watch host code
    "watch:webview": "pnpm --filter webview-ui dev", // Run Vite dev server for UI
    "vscode:prepublish": "pnpm run compile && pnpm run build:prod", // Hook for vsce, ensures quality
    "package": "vsce package --out ./releases/roo-code.vsix --yarn", // Create .vsix package
    "publish": "vsce publish --yarn" // Publish to Marketplace
  }
}
```

*   **Development:**
    1.  Run `pnpm install:all`.
    2.  Open two terminals.
    3.  Terminal 1: `pnpm watch:host` (rebuilds `dist/extension.js` on `src/` changes).
    4.  Terminal 2: `pnpm watch:webview` (starts Vite dev server for UI HMR).
    5.  VS Code: Press F5 to start debugging (uses `dist/extension.js`, connects to Vite server for UI).
*   **Packaging for Release:**
    1.  Run `pnpm install:all`.
    2.  Run `pnpm compile` (catch type errors).
    3.  Run `pnpm test` (catch functional errors - [Chapter 57](57_testing_framework.md)).
    4.  Run `pnpm build:prod` (create optimized `dist/`).
    5.  Run `pnpm package` (create `releases/roo-code.vsix` from `dist/`).

## Code Walkthrough

### Build Script (`esbuild.js`)

*(See refined code in Chapter 56 - Code Walkthrough)*

*   **Orchestration:** Defines and executes the build steps in sequence.
*   **Mode Handling:** Uses `isWatch` and `isProduction` flags to adjust `esbuild` config (sourcemap, minify) and conditionally run the Vite build.
*   **`esbuild.build()`:** Invokes the fast bundler for the extension host code (`src/extension.ts`). Key options: `platform: 'node'`, `format: 'cjs'`, `external: ['vscode']`.
*   **`child_process.execSync`:** Synchronously executes `pnpm --filter webview-ui build` to trigger Vite's production build. Ensures this completes before asset copying.
*   **`copyAssets()`:** Explicitly copies various asset types (WASM, locales for host & webview, NLS, images, Vite build output) from their source locations into the final `dist/` directory structure using Node.js `fs` functions.

### Vite Configuration (`webview-ui/vite.config.ts`)

*(See code in Chapter 1)*
*   Configures Vite for the React application (`webview-ui/`).
*   Sets `build.outDir` to `"build"`, meaning Vite outputs its production assets into `webview-ui/build/`.
*   Includes React and Tailwind plugins.
*   Defines path aliases.
*   Configures the development server for `pnpm watch:webview`.

### `.vscodeignore` (Conceptual)

*(See conceptual code in Chapter 56)*
*   **Purpose:** Tells `vsce` which files/folders *not* to include in the `.vsix` package.
*   **Crucial Exclusions:** `node_modules/`, `src/`, `webview-ui/src/`, `webview-ui/node_modules/`, `webview-ui/build/`, test files, config files (`.env`, `tsconfig.*`, `jest.*`), `scripts/`, `evals/`.
*   **Crucial Inclusions:** The build script ensures everything needed at runtime is in `dist/`. Therefore, `.vscodeignore` should generally **not** ignore `dist/**`. It should also include essential root files like `package.json`, `README.md`, `CHANGELOG.md`, `LICENSE`.

## Internal Implementation

1.  **Trigger:** `pnpm build:prod` initiates the sequence defined in `package.json`.
2.  **Clean:** `rimraf` deletes `dist/` and `webview-ui/build/`.
3.  **Prepare:** `node scripts/prepare.js` runs.
4.  **Build Script (`esbuild.js`):**
    *   Detects `NODE_ENV=production`.
    *   Calls `esbuild.build()` for host -> `dist/extension.js` created (minified).
    *   Calls `execSync` for `pnpm --filter webview-ui build`.
        *   Vite runs, bundles React app -> `webview-ui/build/` created.
    *   Calls `copyRecursiveSync` -> `webview-ui/build/*` copied to `dist/webview-ui/`.
    *   Calls `copyAssets` -> WASM, locales (`public/locales`, `src/i18n/locales`), NLS files are copied into `dist/`.
5.  **Packaging:** `pnpm package` runs `vsce package`. `vsce` reads `package.json`, uses `.vscodeignore` to select files (mostly from `dist/`), zips them into `.vsix`.

## Modification Guidance

Modifications usually involve adjusting build targets, adding new entry points/packages, configuring plugins, or managing new types of assets.

1.  **Adding a New Asset Type (e.g., Python Scripts for a Tool):**
    *   **Source Location:** Place the scripts in a logical source location (e.g., `resources/python_scripts/`).
    *   **`esbuild.js`:** Modify the `copyAssets` function. Add logic to find the source scripts and copy them to the desired runtime location within `dist/` (e.g., `dist/python_scripts/`).
        ```javascript
        const pySrc = path.join(__dirname, "resources", "python_scripts");
        const pyDest = path.join(outDir, "python_scripts");
        if (fs.existsSync(pySrc)) { copyRecursiveSync(pySrc, pyDest); log.info("Copied Python scripts."); }
        ```
    *   **Code Reference:** Update any host code that needs to execute these scripts to use paths relative to the extension's runtime directory (e.g., using `context.extensionPath` combined with `dist/python_scripts/...`).
    *   **`.vscodeignore`:** Ensure the new directory within `dist/` (e.g., `dist/python_scripts/`) is **not** ignored.

2.  **Changing Build Target (e.g., Newer Node Version):**
    *   **`esbuild.js`:** Update `target: 'node16'` to `target: 'node18'` (or newer) in `extensionConfig`.
    *   **VS Code `engines`:** Update the `engines.vscode` and potentially `engines.node` fields in the root `package.json` to reflect the new minimum requirements for the extension to run.

3.  **Adding a Web Worker to the WebView UI:**
    *   **Vite Config:** Modify `webview-ui/vite.config.ts`. Use Vite's Web Worker features (e.g., `import MyWorker from './worker?worker'`) and potentially configure Rollup options within `build.rollupOptions` for worker output if needed. Vite will handle bundling the worker script.
    *   **`esbuild.js`:** No changes usually needed here, as Vite handles the worker bundling. The `copyRecursiveSync` for the Vite output will include the generated worker file(s).

**Best Practices:**

*   **Tool Choice:** Use fast, modern tools like `esbuild` and Vite.
*   **Clean Builds:** Always clean the output directory (`dist/`) before starting a build to avoid stale artifacts.
*   **Separate Processes:** Keep host and WebView builds separate (esbuild vs. Vite) as they target different environments. Orchestrate them in the main build script.
*   **Explicit Asset Copying:** Clearly define and implement the copying of all necessary runtime assets in the build script or dedicated plugins. Don't rely on implicit behavior.
*   **Environment Variables:** Use `NODE_ENV` consistently. Inject build-time constants via `define`. Manage secrets via build environment variables (from CI/local `.env` NOT committed).
*   **`.vscodeignore`:** Keep this file accurate and minimal to reduce package size. Test packaging locally (`vsce package`) to verify contents.
*   **Type Checking:** Integrate `tsc --noEmit` (`pnpm compile`) into the workflow (especially `vscode:prepublish` and CI) for comprehensive type safety.

**Potential Pitfalls:**

*   **Missing Assets in `dist/`:** Runtime "File not found" errors. Double-check `copyAssets` logic and source/destination paths. Verify WASM files, NLS files, host/web locales, and the entire `webview-ui` build output are copied correctly.
*   **Incorrect Build Mode:** Accidentally creating dev builds (with source maps, larger size) for production/publishing due to incorrect `NODE_ENV`.
*   **`external: ['vscode']`:** Forgetting this for the host build.
*   **Asset Path Issues:** Code referencing assets using paths that are incorrect relative to the final `dist/` structure or the extension's runtime location (`context.extensionPath`).
*   **`.vscodeignore` Errors:** Excluding necessary files from `dist/` or failing to exclude large source/dev files.
*   **Build Tool Conflicts:** Issues arising from interactions between `esbuild`, `vite`, `tsc`, `pnpm` workspaces if not configured correctly.
*   **Slow Asset Copying:** Copying huge numbers of small files can be slow; check efficiency of copy utilities.

## Conclusion

The Build System, orchestrated via `esbuild.js` and `package.json` scripts, reliably transforms Roo-Code's source code into a functional and distributable VS Code extension. It leverages the speed of `esbuild` for the extension host and the rich features of Vite for the WebView UI. Key aspects include distinct configurations for host vs. web, handling development vs. production builds, injecting environment variables, and crucially, ensuring all necessary runtime assets (WASM, host/web locales, WebView bundles, NLS files) are correctly copied to the `dist/` directory, ready for packaging with `vsce`. This automated system is fundamental for both efficient development and creating optimized, distributable extension builds.

Next, we will look at how the project ensures code quality and functionality through automated tests: [Chapter 57: Testing Framework](57_testing_framework.md).
---
# Chapter 57: Testing Framework

Continuing from [Chapter 56: Build System](56_build_system.md), which focused on compiling and packaging Roo-Code, this chapter delves into the strategies and tools used to ensure the quality, correctness, and stability of the codebase: the **Testing Framework**.

## Motivation: Ensuring Quality and Preventing Regressions

As Roo-Code grows in complexity, with interactions between the extension host, WebView UI, LLM APIs, file system, terminal, and various utilities, the risk of introducing bugs or regressions increases significantly. Manually testing every feature and edge case after each change is impractical and unreliable.

A robust testing framework provides an automated way to:

1.  **Verify Correctness:** Ensure individual functions, components, and modules behave as expected according to their specifications.
2.  **Prevent Regressions:** Automatically detect when changes inadvertently break existing functionality.
3.  **Improve Design:** Writing testable code often encourages better design practices (modularity, pure functions, dependency injection).
4.  **Facilitate Refactoring:** Provide confidence that code can be refactored safely.
5.  **Document Behavior:** Tests act as executable documentation, illustrating how components are intended to be used and what their expected outputs are.

Roo-Code employs a combination of testing strategies, primarily using **Jest** as the test runner and framework, along with **React Testing Library** for testing WebView UI components, running in appropriate environments (Node.js for host, JSDOM for UI) with extensive mocking.

**Central Use Case:** A developer refactors the `normalizeLineEndings` utility ([Chapter 45: Text Normalization Utilities](45_text_normalization_utilities.md)) and needs to ensure it still works correctly.

1.  **Write/Update Test:** Ensures `src/utils/text-normalization.test.ts` has comprehensive test cases covering CRLF, CR, LF, mixed, empty, null inputs.
    ```typescript
    // Example test case in src/utils/text-normalization.test.ts
    import { normalizeLineEndings } from './text-normalization';

    describe('normalizeLineEndings', () => {
      it('should replace CRLF with LF', () => {
        expect(normalizeLineEndings("Hello\r\nWorld")).toBe("Hello\nWorld");
      });
      it('should replace CR with LF', () => {
        expect(normalizeLineEndings("Hello\rWorld")).toBe("Hello\nWorld");
      });
      // ... other test cases ...
    });
    ```
2.  **Run Tests:** Runs `pnpm test` or `pnpm test:host`.
3.  **Jest Execution:** Jest finds and runs the test file using the 'Host' project configuration (`testEnvironment: 'node'`). It executes the assertions.
4.  **Results:** Jest reports pass/fail status. Any regressions introduced by the refactoring cause test failures, alerting the developer immediately.

## Key Concepts

1.  **Jest:** The core testing framework ([https://jestjs.io/](https://jestjs.io/)). Provides test runner, assertion library (`expect`), powerful mocking capabilities (`jest.fn`, `jest.mock`, `jest.spyOn`), snapshot testing, and configuration (`jest.config.js`).
2.  **React Testing Library (`@testing-library/react`, `@testing-library/jest-dom`):** Standard library for testing React components focusing on user interaction and accessibility (`render`, `screen.getByRole`, `fireEvent`/`userEvent`). Used for WebView UI tests. ([https://testing-library.com/](https://testing-library.com/))
3.  **Testing Environments (`jest.config.js` `projects`):** Crucial for Roo-Code due to the separate runtimes:
    *   **Extension Host (`displayName: "Host"`, `testEnvironment: 'node'`):** For code in `src/`. Runs tests in a Node.js environment. **Requires mocking the `vscode` API**.
    *   **WebView UI (`displayName: "WebView"`, `testEnvironment: 'jsdom'`):** For code in `webview-ui/src/`. Runs tests in a simulated browser DOM (JSDOM). **Requires mocking VS Code toolkit components** ([Chapter 32](32_vscode_webview_ui_toolkit_wrappers.md)) and the **`postMessage` communication mechanism** ([Chapter 3](03_webview_extension_message_protocol.md)).
4.  **Mocking Dependencies:** Essential for isolating units under test and enabling execution outside the actual VS Code environment.
    *   **`vscode` API:** Mocked via `src/__mocks__/vscode.ts`. Provides `jest.fn()` stand-ins for API calls, allowing tests to assert interactions or provide controlled responses.
    *   **`@vscode/webview-ui-toolkit/react`:** Mocked via `src/__mocks__/@vscode/webview-ui-toolkit/react.ts`. Renders simple HTML equivalents, enabling UI component logic tests in JSDOM.
    *   **`postMessage`:** The `webview-ui/src/utils/vscode.ts` wrapper containing the `acquireVsCodeApi().postMessage` call is mocked in UI tests (`jest.mock('@/utils/vscode', ...)`). This allows tests to assert that `vscode.postMessage` was called with specific message objects.
    *   **Network Calls (`ApiHandler`, `axios`):** Modules making external HTTP requests are mocked using `jest.mock()` to prevent actual network calls and provide controlled responses during tests.
    *   **File System (`fs`):** For tests involving file operations (e.g., persistence, config loading), the `fs/promises` module can be mocked using `jest.mock('fs/promises')` or libraries like `memfs` to simulate file interactions in memory.
5.  **Test Structure (`describe`, `it`/`test`, Arrange-Act-Assert):** Standard Jest structure for organizing tests (`describe`) and defining individual test cases (`it` or `test`). The Arrange-Act-Assert pattern is commonly used within test cases. Setup/teardown hooks (`beforeEach`, `afterEach`, `beforeAll`, `afterAll`) manage shared state or resources for tests within a `describe` block.
6.  **Code Coverage:** Generated using `jest --coverage`. Measures the percentage of code lines, branches, and functions executed by the test suite. Useful for identifying untested areas. Configured via `collectCoverageFrom` in `jest.config.js`.

## Executing Tests

Tests are run via scripts defined in `package.json`.

```json
// --- File: package.json (Excerpt) ---
{
  "scripts": {
    "test": "jest", // Run all tests defined in Jest projects
    "test:watch": "jest --watch", // Run in interactive watch mode
    "test:coverage": "jest --coverage", // Run tests and generate coverage report
    // Use Jest's --selectProjects flag with project displayNames
    "test:host": "jest --selectProjects Host",
    "test:ui": "jest --selectProjects WebView",
    // Specific file test example
    "test:path": "jest src/utils/path.test.ts"
    // ... other scripts ...
  }
}
```

*   `pnpm test`: Runs all tests defined across Jest projects ("Host" and "WebView").
*   `pnpm test:watch`: Interactive watch mode, re-runs tests on file changes.
*   `pnpm test:coverage`: Runs tests and outputs coverage summary/reports.
*   `pnpm test:host` / `pnpm test:ui`: Runs only tests from the specified project configuration.
*   `pnpm test <path/to/file>`: Runs only tests in a specific file or directory.

## Code Walkthrough

### Jest Configuration (`jest.config.js`)

*(See full code in Chapter 57 - Key Concepts)*

*   **`projects` Array:** Defines the "Host" and "WebView" projects with distinct `displayName`, `testEnvironment`, `rootDir`, `testMatch`, and `moduleNameMapper` settings.
*   **`ts-jest` Preset:** Enables TypeScript support.
*   **`moduleNameMapper`:** Handles path aliases (`@/`, `@roo/`, `@src/`) and CSS module mocking (`identity-obj-proxy`).
*   **`roots` (in WebView project):** `roots: ["<rootDir>/.."]` is crucial. It tells Jest (when running the WebView project with `rootDir: "webview-ui"`) to look one level up (`..`) for mock directories, allowing it to find `src/__mocks__`.
*   **Coverage Config:** `collectCoverageFrom` specifies which source files should be included in coverage calculations, excluding tests, mocks, schemas, types, build outputs, etc.

### Mocking `vscode` (`src/__mocks__/vscode.ts`)

*(See full code in Chapter 57 - Key Concepts)*

*   Provides `jest.fn()` implementations for essential VS Code API functions used by Roo-Code (`window.showInformationMessage`, `workspace.getConfiguration().get`, `commands.executeCommand`, `secrets.store/get`, `languages.getDiagnostics`, `Uri.file`, etc.).
*   Allows tests to assert calls (`expect(...).toHaveBeenCalledWith(...)`) or provide mock return values (`mockReturnValue`, `mockResolvedValue`).

### Mocking Toolkit (`src/__mocks__/@vscode/webview-ui-toolkit/react.ts`)

*(See code in Chapter 32)*

*   Exports simple React functional components that render basic HTML elements (`<button>`, `<input>`, `<select>`) corresponding to toolkit components (`VSCodeButton`, `VSCodeTextField`, `VSCodeDropdown`).
*   Adapts event handlers (`onChange`, `onInput`) slightly to match the structure expected by components using the real toolkit.
*   Allows React Testing Library to render and interact with components using the toolkit in the JSDOM environment.

### Unit Test Example (Host Utility - `src/utils/path.test.ts`)

*(See code in Chapter 57 - Key Concepts)*

*   Uses `jest.mock('vscode', ...)` locally to control `workspace.workspaceFolders` for specific test cases.
*   Uses `Object.defineProperty(process, 'platform', ...)` to simulate different OS environments for testing platform-specific logic (`arePathsEqual`).
*   Employs standard Jest assertions (`expect().toBe()`).

### React Component Test (WebView UI - `webview-ui/src/components/settings/About.test.tsx`)

*(See code in Chapter 57 - Key Concepts)*

*   Uses `@testing-library/react`'s `render`, `screen`, `fireEvent`.
*   Explicitly mocks local dependencies (`@/utils/vscode`, `@/i18n/TranslationContext`). Relies on Jest config (`roots`) for auto-mocking the toolkit via `src/__mocks__`.
*   Uses user-centric queries (`screen.getByLabelText`, `screen.getByRole`).
*   Simulates user interaction (`fireEvent.click`).
*   Asserts on DOM state (`.toBeInTheDocument()`, `.checked`) using `@testing-library/jest-dom` and on mock function calls (`vscode.postMessage`, `mockSetTelemetry`) using Jest matchers.

## Internal Implementation

1.  **Test Execution:** `pnpm test` invokes `jest`.
2.  **Configuration/Projects:** Jest reads `jest.config.js`, identifies "Host" and "WebView" projects. It runs tests for each project sequentially or in parallel based on config/flags.
3.  **Environment Setup:** For each project, Jest sets up the specified `testEnvironment` (`node` or `jsdom`). JSDOM creates a virtual DOM for UI tests.
4.  **Mock Resolution:** Jest's module loader intercepts `import` statements.
    *   It checks for corresponding files in `__mocks__` directories (considering the project's `roots` config).
    *   It checks for explicit `jest.mock()` calls in the test file.
    *   It applies `moduleNameMapper` rules.
    *   If a mock is found, it's used; otherwise, the real module is loaded.
5.  **Test File Execution:** Jest executes the test code (`describe`/`it` blocks).
6.  **Rendering (UI):** `@testing-library/react`'s `render` renders components into JSDOM using the (mocked) toolkit components.
7.  **Interaction/Assertion:** Tests use Testing Library utils to interact with the JSDOM and Jest's `expect` to assert outcomes or mock function calls.
8.  **Code Coverage:** If requested (`--coverage`), code execution is tracked, and reports are generated.

## Modification Guidance

Involves adding tests for new code, updating tests for changed code, or improving the mocking infrastructure.

1.  **Adding Tests for a New Utility Function (Host):**
    *   Create `<filename>.test.ts` in `src/utils/`.
    *   Import the function. Use `describe`/`it`, `expect`. Mock dependencies (e.g., `fs`, other utils) using `jest.mock()`. Run with `pnpm test:host`.

2.  **Adding Tests for a New React Component (WebView):**
    *   Create `<ComponentName>.test.tsx` in `webview-ui/src/components/`.
    *   Import React, RTL, component. Mock dependencies (`vscode` util, context hooks, toolkit components if needed). Use `render`, `screen`, `fireEvent`/`userEvent`, `expect`. Run with `pnpm test:ui`.

3.  **Updating the `vscode` Mock (`src/__mocks__/vscode.ts`):**
    *   Add `jest.fn()` mocks or mock values for newly used VS Code APIs.
    *   Refine existing mocks if needed (e.g., improve mock return values for `getConfiguration`).

4.  **Improving Test Robustness:**
    *   Use `@testing-library/user-event` instead of `fireEvent` for more realistic interaction simulation.
    *   Ensure mocks accurately reflect essential dependency interfaces.
    *   Add tests for error conditions and edge cases.

**Best Practices:**

*   **Unit Focus:** Prioritize unit tests with effective mocking for speed and isolation.
*   **Test Behavior:** Use RTL for UI tests, focusing on user interaction and observable outcomes. Avoid testing implementation details.
*   **Mock Boundaries:** Mock external dependencies (`vscode`, network, toolkit) and potentially major internal modules/services, but avoid mocking simple internal helpers.
*   **Coverage as Guide:** Use coverage to find untested areas, but focus on testing critical logic paths.
*   **Maintainability:** Write clear, readable tests. Reset mocks (`clearMocks: true`).
*   **CI Integration:** Run `pnpm test` in the CI pipeline (e.g., GitHub Actions workflow defined in `.github/workflows/`) to catch regressions automatically.

**Potential Pitfalls:**

*   **Incomplete/Incorrect Mocks:** Tests passing locally but failing in reality, or vice versa. Mocks need maintenance.
*   **Over-Mocking:** Brittle tests breaking on minor refactors.
*   **Testing Implementation Details:** Fragile tests.
*   **Testing Environment Mismatch:** Bugs only reproducible in the actual VS Code environment (Node host or WebView). Requires some manual or end-to-end testing.
*   **Flaky Tests:** Async issues, state leakage. Use `waitFor`, `act`, reset mocks/state carefully.
*   **Slow Test Suite:** Can slow down development feedback. Optimize tests, use parallel execution (Jest default), mock effectively.

## Conclusion

The Testing Framework, primarily utilizing Jest and React Testing Library, is crucial for maintaining the quality and stability of the Roo-Code extension. By providing separate testing environments and configurations for the extension host (Node) and WebView UI (JSDOM), and leveraging comprehensive mocking for dependencies like the `vscode` API and UI toolkits, the framework enables automated verification of individual units and components. This automated testing catches regressions early, facilitates safer refactoring, and ultimately contributes to a more reliable and robust product for users. Running tests frequently, especially in CI pipelines, is a key practice for sustainable development.

Finally, we'll look at the documentation and community-related files that support the project: [Chapter 58: Documentation & Community Files](58_documentation___community_files.md).
---
# Chapter 58: Documentation & Community Files

Continuing from [Chapter 57: Testing Framework](57_testing_framework.md), which detailed how Roo-Code ensures code quality through testing, this final chapter looks at the essential supporting files that explain the project, guide contributors, and foster a healthy community: **Documentation & Community Files**.

## Motivation: Guiding Users and Contributors

A software project, especially an open-source one like Roo-Code, needs more than just functional code. To be successful, usable, and sustainable, it requires clear documentation and standard community health files to:

1.  **Explain Usage:** Guide end-users on how to install, configure, and effectively use the extension's features ([Chapter 1: WebView UI](01_webview_ui.md), [Chapter 35: Settings UI Components (WebView)](35_settings_ui_components__webview_.md)).
2.  **Onboard Developers:** Provide instructions for contributors on how to set up the development environment, understand the architecture (like this tutorial!), run builds ([Chapter 56: Build System](56_build_system.md)), run tests ([Chapter 57: Testing Framework](57_testing_framework.md)), and follow contribution guidelines.
3.  **Document Architecture:** Explain the design choices, core components (e.g., `ClineProvider`, `ApiHandler`, `Cline`), data flow, and workflows within the codebase (as this tutorial series aims to do).
4.  **Set Expectations:** Define codes of conduct, contribution processes, and security policies for the community.
5.  **Facilitate Collaboration:** Make it easier for users to report bugs, request features, and for contributors to participate effectively.
6.  **Meet Marketplace Requirements:** Provide necessary information (`README`, `LICENSE`, `CHANGELOG`) for publishing to the VS Code Marketplace.
7.  **Define Project Standards:** Document coding conventions, code quality rules (`.roo/rules/rules.md`), and specific guidelines like those for localization (`.roo/rules-translate/`).

These files, typically in the project root, `docs/`, `.github/`, or `.roo/`, are crucial for usability, maintainability, and community engagement.

## Key Concepts & Files

1.  **`README.md`:** The primary entry point for the project repository. Contains a high-level overview, key features, installation instructions, quick start examples, screenshots/GIFs, links to detailed documentation, contribution pointers, and license information. Vital for both users and potential contributors.
2.  **`LICENSE` (or `LICENSE.md`):** The full open-source license text (e.g., MIT License). Legally defines usage and distribution rights. Essential for clarity and compliance.
3.  **`CHANGELOG.md`:** Records user-facing changes (Added, Changed, Fixed, Removed) for each version, typically following the [Keep a Changelog](https://keepachangelog.com/) format. Helps users understand updates.
4.  **`CONTRIBUTING.md`:** Guidelines for developers wanting to contribute. Includes development setup (linking to build/test docs), coding standards (linking to `rules.md`), Git workflow (branching, commits), pull request process (linking to template), and where to find tasks.
5.  **`CODE_OF_CONDUCT.md`:** Establishes community behavior standards, fostering a positive and inclusive environment. Often based on the Contributor Covenant.
6.  **`SECURITY.md`:** Outlines the private, responsible disclosure process for reporting security vulnerabilities (e.g., via a dedicated email address).
7.  **`.github/` Directory:** Standard location for GitHub repository configuration:
    *   **`ISSUE_TEMPLATE/`:** Markdown templates (e.g., `bug_report.md`, `feature_request.md`) to structure issue reporting.
    *   **`PULL_REQUEST_TEMPLATE.md`:** Markdown template automatically added to new PR descriptions, often including a checklist for contributors (e.g., "Tests added/passed", "Docs updated").
    *   **`workflows/`:** GitHub Actions YAML files defining CI/CD pipelines (e.g., `ci.yaml` to run lint, compile, test, build on pushes/PRs).
8.  **`docs/` Directory:** Contains more extensive documentation:
    *   **User Guides:** Detailed explanations of features and configuration.
    *   **Architecture Documentation:** This tutorial series (`docs/architecture/*.md`), potentially diagrams, explaining the internal design and components.
9.  **`.roo/` Directory:** Contains project-specific rules and guidelines, potentially used by humans and AI assistants involved in the project:
    *   **`rules/rules.md`:** Project-specific coding standards, architectural patterns, testing requirements, UI guidelines ([Chapter 48: Configuration Rules](48_configuration_rules.md)).
    *   **`rules-translate/`:** Localization guidelines ([Chapter 50: Localization System (i18n)](50_localization_system__i18n_.md)). Includes `001-general-rules.md` and language-specific files like `instructions-de.md`, `instructions-zh-cn.md` with glossaries, tone guidance, etc.

## Using These Files

*   **Users:** Start with `README.md`. Refer to `docs/` for details. Check `CHANGELOG.md`. Use `ISSUE_TEMPLATE`s.
*   **Contributors:** Start with `README.md`, then `CONTRIBUTING.md`. Refer to `docs/architecture/`, `CODE_OF_CONDUCT.md`, `.roo/rules/rules.md`. Use GitHub Templates, check CI results.
*   **Translators:** Focus on files within `.roo/rules-translate/`. Refer to `CONTRIBUTING.md` for workflow.
*   **Maintainers:** Use all files for project management, reviews, community standards. Use CI workflows.
*   **AI Assistants:** `.roo/rules/` files can be provided as context to AI tools used within the project for tasks like code review, generation, or translation, ensuring AI output adheres to project standards.
*   **VS Code Marketplace:** Reads `README.md`, `CHANGELOG.md`, `LICENSE` for the extension's listing.

## Code Walkthrough

These files are primarily Markdown, YAML, or JSON, defining text content or configuration.

### `.roo/rules/rules.md` (Example Structure)

*(See code in Chapter 48)*
*   Defines standards for testing, TypeScript, UI styling, state management, error handling, dependencies. References relevant tutorial chapters.

### `.roo/rules-translate/` Files (Example Structure)

*(See code in Chapter 50)*
*   `001-general-rules.md`: General tone, non-translatable terms, placeholders, `Trans` usage, QA script (`find-missing-translations.js`).
*   Language-specific files (`instructions-<lang>.md`): Specific rules (e.g., "du" form), glossaries, formatting.

### GitHub Workflow Example (`.github/workflows/ci.yaml`)

*(See code in Chapter 57)*
*   Defines CI pipeline triggered on push/PR. Sets up Node/pnpm. Runs install, lint, compile, test, build.

## Internal Implementation

These are static documentation or configuration files. Their "implementation" lies in their usage by humans, platforms like GitHub/VS Code Marketplace, and automated tools (CI runners, linters, build scripts).

## Modification Guidance

Involves updating content to reflect project changes, improve clarity, or add new guidelines.

1.  **Updating `README.md` / `docs/`:** Add/modify feature descriptions, update setup/configuration steps, fix broken links, add new architecture diagrams or tutorial chapters.
2.  **Updating `CHANGELOG.md`:** Add a new section for the upcoming release at the top. List changes under `Added`, `Changed`, `Fixed`, `Removed`. Follow date and version formatting.
3.  **Modifying `CONTRIBUTING.md`:** Update commands if the build ([Chapter 56](56_build_system.md)) or test ([Chapter 57](57_testing_framework.md)) process changes. Clarify branching strategy or code style requirements (linking to `rules.md`).
4.  **Updating Code Quality Rules (`.roo/rules/rules.md`):** Add/modify rules based on team decisions, new best practices, or architectural changes. Ensure alignment with automated checks (linters, `tsconfig.json`).
5.  **Updating Translation Guidelines (`.roo/rules-translate/`):** Add new terms to glossaries. Clarify tone or formatting rules based on feedback. Update QA process or validation script usage.

**Best Practices:**

*   **Keep Updated:** Essential for trustworthiness and usability. Assign ownership or establish a process for regular review (e.g., alongside releases).
*   **Clarity & Conciseness:** Write clearly for the intended audience. Use formatting (headings, lists, code blocks) effectively.
*   **Standard Formats:** Follow established conventions (Keep a Changelog, Contributor Covenant). Use standard GitHub file locations (`.github/`, root files).
*   **Automation:** Use CI (`workflows/`) to enforce checks mentioned in guidelines (lint, test, build). Use GitHub templates to structure contributions.
*   **Discoverability:** Link related documents (e.g., README -> CONTRIBUTING -> docs -> rules).
*   **Accuracy:** Ensure technical instructions (setup, build, test) are correct and verified.

**Potential Pitfalls:**

*   **Outdated Documentation:** The most common problem, leading to user frustration and contributor difficulties.
*   **Inaccurate Instructions:** Build/setup steps that no longer work or are incomplete.
*   **Missing Files:** Forgetting essential files like `LICENSE` or `CHANGELOG` required for publishing or open-source community standards.
*   **Broken Links:** Links within documentation becoming invalid over time.
*   **Unclear Contribution Process:** Vague guidelines deterring potential contributors.
*   **Inconsistent Rules:** Contradictions between different documents or between documentation and actual practice/automated checks.
*   **Guidelines Ignored:** Rules existing but not being followed or enforced during reviews.

## Conclusion

Documentation and Community Files, including project-specific rules like `rules.md` and `rules-translate/`, are vital supporting elements for the Roo-Code project's health, usability, and collaborative success. A clear `README.md` introduces the extension, `CONTRIBUTING.md` guides developers, `CHANGELOG.md` tracks progress, `LICENSE` defines legal terms, and other files like `CODE_OF_CONDUCT.md` and `SECURITY.md` foster a healthy community. GitHub-specific files streamline issue reporting, contributions, and CI/CD automation. Detailed architecture docs (like this tutorial series) and specific rule files (`.roo/`) ensure consistency and quality. Maintaining these resources accurately and keeping them up-to-date is an ongoing but critical task for ensuring a positive experience for everyone involved with Roo-Code.

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

