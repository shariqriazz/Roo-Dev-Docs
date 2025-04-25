# Chapter 55: Script Utilities

Continuing from [Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md), which focused on UI components for the evaluation web interface, this chapter shifts focus to backend utilities used across various development and operational scripts within the Roo-Code project: the **Script Utilities**.

## Motivation: Common Helpers for Development and Build Scripts

Beyond the core extension runtime and UI code, a software project like Roo-Code involves numerous supporting scripts located in the `scripts/` directory (and potentially within the `evals/` structure) for tasks such as:

*   Building the extension ([Chapter 56: Build System](56_build_system.md)).
*   Running linters, formatters, or tests ([Chapter 57: Testing Framework](57_testing_framework.md)).
*   Generating documentation or schemas (e.g., `scripts/prepare.js`).
*   Running evaluation suites ([Chapter 53: Evals System](53_evals_system.md)).
*   Performing release tasks (e.g., updating versions, packaging).
*   Finding missing translation keys (`scripts/find-missing-translations.js`, `scripts/find-missing-i18n-key.js` - see [Chapter 50: Localization System (i18n)](50_localization_system__i18n_.md)).
*   Updating the contributors list in README files (`scripts/update-contributors.js`).

These scripts, often written in TypeScript or JavaScript and executed using Node.js (e.g., via `pnpm run script-name`), frequently require common helper functionalities:

*   Parsing command-line arguments.
*   Interacting with the file system robustly (reading directories, finding files by pattern, checking paths) ([Chapter 42: File System Utilities](42_file_system_utilities.md), [Chapter 43: Path Utilities](43_path_utilities.md) might be reused or adapted).
*   Executing shell commands or other scripts reliably and capturing output.
*   Handling logging consistently across different scripts with clear formatting (e.g., colors).
*   Managing asynchronous operations, potentially with concurrency limits.
*   Handling JSON/YAML parsing and writing.

Implementing these helpers repeatedly in each script is inefficient and leads to inconsistency. The **Script Utilities** (conceptually located in directories like `scripts/lib/`, `evals/src/lib/`, or shared `src/utils/` if suitable and VS Code API independent) provide a collection of reusable functions tailored for these Node.js scripting contexts.

**Central Use Case:** The `find-missing-translations.js` script needs to read multiple JSON locale files from different directories (`src/i18n/locales/` and `webview-ui/src/i18n/locales/`), parse them, compare keys, and log warnings to the console.

Without Utilities (Partial):
```javascript
// Conceptual code within find-missing-translations.js without specific utils
const fs = require('fs');
const path = require('path');

function checkArea(areaPath) {
    // ... Complex logic to find 'en' files ...
    const enDir = path.join(areaPath, 'en');
    // ... Read and parse all en/*.json files, collect keys ...
    // ... Find other language directories ...
    // ... For each lang/file, read/parse ...
    // ... Compare keys, log differences to console ...
    // ... Error handling for fs and JSON.parse ...
}
```

With Utilities:
```javascript
// Conceptual code using utils
import { findJsonFiles, readAndValidateJson } from './lib/fsUtils'; // Assuming utilities exist
import { scriptLogger as logger } from './lib/logger'; // Assuming logger utility
import { translationKeySchema } from '../schemas/i18n'; // Assuming Zod schema

async function checkArea(areaPath) {
    const defaultLang = 'en';
    const defaultLangPath = path.join(areaPath, defaultLang);
    const defaultFiles = await findJsonFiles(defaultLangPath);
    const defaultKeys = new Map(); // Map<namespace, Set<key>>

    // Load default keys, validating with Zod
    for (const file of defaultFiles) {
        const namespace = path.basename(file, '.json');
        const data = await readAndValidateJson(file, z.record(z.string())); // Basic schema
        if (data) {
            defaultKeys.set(namespace, new Set(Object.keys(data)));
        } else {
            logger.warn(`Skipping invalid default file: ${file}`);
        }
    }

    // ... Find other language dirs ...
    for (const lang of otherLangs) {
        // ... For each namespace ...
            const langFilePath = path.join(areaPath, lang, `${namespace}.json`);
            const langData = await readAndValidateJson(langFilePath, z.record(z.string()).optional());
            defaultKeys.get(namespace)?.forEach(key => {
                if (!langData || !langData[key]) {
                    logger.warn(`Missing ${lang}/${namespace}:${key}`);
                    // ... track missing ...
                }
            });
        // ...
    }
    // ... report results ...
}
```
Utilities like `findJsonFiles`, `readAndValidateJson`, and `scriptLogger` handle common file system interactions, validation, and logging, making the main script logic focus on the comparison task itself.

## Key Concepts

1.  **Target Environment:** Node.js execution context (run via `pnpm`, etc.). Can use full Node.js API and `devDependencies`. **Cannot use `vscode` module APIs.**
2.  **Common Functionality:** Focus on recurring script tasks:
    *   **Argument Parsing:** `yargs`, `commander`.
    *   **File System:** `fs/promises`, `glob` (e.g., `findFiles`, `readFileSafe`, `writeFileSafe`, `copyDirRecursive`, `ensureDir`). Reuses/adapts Node-compatible utils from `src/utils/fs.ts` ([Chapter 42]).
    *   **Shell Execution:** `child_process`, `execa` (preferred).
    *   **Logging:** Console logger (`console`) enhanced with `chalk`.
    *   **Path Manipulation:** Node.js `path`. Reuses Node-compatible `src/utils/path.ts` helpers ([Chapter 43]). Base paths from `process.cwd()` or args.
    *   **Concurrency:** `p-limit`.
    *   **JSON/YAML:** `JSON`, `js-yaml`, Zod validation ([Chapter 40]).
3.  **Location:** Primarily within `scripts/lib/`. Some eval-specific helpers might be in `evals/src/lib/`. Shared, Node-compatible utilities might be in `src/utils/`.
4.  **Reusability:** Imported across script files (`*.js`, `*.ts`).
5.  **Specific Scripts Provided:**
    *   `scripts/find-missing-translations.js`: Validates translation files across languages ([Chapter 50]). Uses FS, path, console.
    *   `scripts/find-missing-i18n-key.js`: Scans source code for `t()` usages and checks if keys exist in locale files. Uses FS, path, regex, console.
    *   `scripts/update-contributors.js`: Fetches contributor data from GitHub API (`https`) and updates `README.md` and localized versions (`locales/**/README.md`) using FS and string manipulation.
    *   `scripts/prepare.js`: Placeholder for potential pre-build code generation or setup tasks.

## Using Script Utilities

Imported as standard functions/modules within script files located in `scripts/`, `evals/apps/cli/src/`, etc.

*   **Argument Parsing:** Scripts like `find-missing-translations.js` parse `process.argv` manually or use `yargs` (as in `evals/apps/cli`).
*   **Finding Files:** `find-missing-translations.js` uses `fs.readdirSync`, `fs.statSync`, `path.join`. `find-missing-i18n-key.js` uses recursive `fs.readdirSync`. A utility like `findYamlFiles` (using `glob`) is used by the eval runner.
*   **Executing Commands:** The build script `esbuild.js` uses `child_process.execSync` to run the Vite build. A utility like `runCommand` (using `execa`) would be ideal for cleaner execution.
*   **Logging:** `find-missing-translations.js` uses `console.log/warn/error`. A dedicated `scriptLogger` with `chalk` (Conceptual Example) provides colored, formatted output.
*   **Reading/Validating Files:** `find-missing-translations.js` uses `fs.readFileSync`, `JSON.parse`. A utility like `readAndValidateJson` (Conceptual Example) adds robustness with Zod schema validation.

## Code Walkthrough

Examining the actual scripts provided and relevant shared utilities.

### `scripts/find-missing-translations.js`

*(See code provided in chapter context)*
*   **Purpose:** Validates completeness of translation JSON files.
*   **Usage:** `node scripts/find-missing-translations.js [--locale=<lang>] [--file=<ns.json>] [--area=<core|webview|both>]`
*   **Logic:**
    *   Parses CLI arguments for filtering.
    *   Defines paths to `core` (`src/i18n/locales`) and `webview` (`webview-ui/src/i18n/locales`) locale directories.
    *   `findKeys` recursively gets all dot-separated keys from a JSON object.
    *   `getValueAtPath` retrieves a value from an object using a dot-separated path.
    *   `checkAreaTranslations`:
        *   Finds non-English language directories.
        *   Loads all keys from English namespace files (`en/*.json`).
        *   For each other language and each English namespace file:
            *   Loads the corresponding language file.
            *   Checks if each English key exists and has a non-empty value in the language file using `getValueAtPath`.
            *   Records missing keys.
    *   `outputResults`: Prints a formatted report of missing keys per language/file to the console.
    *   Main function `findMissingTranslations` calls `checkAreaTranslations` for requested areas and `process.exit(1)` if any keys are missing (for CI integration).
*   **Utilities Used:** Node.js `fs` (sync versions), `path`, `process`.

### `scripts/find-missing-i18n-key.js`

*(See code provided in chapter context)*
*   **Purpose:** Finds `t()` or `i18nKey` usages in code that don't have corresponding entries in the English locale files.
*   **Usage:** `node scripts/find-missing-i18n-key.js [--locale=<lang>] [--file=<ns.json>]` (Locale/file args seem less relevant here, primarily checks against English).
*   **Logic:**
    *   Defines paths to check (`components` in webview, `src`).
    *   Defines regex patterns (`i18nPatterns`) to extract keys from `t("ns:key")`, `{t("ns:key")}`, `i18nKey="ns:key"`.
    *   `checkKeyInLocales`: Checks if a given key (`ns:key`) exists in the *English* locale file for the relevant area (core/webview). *Correction: This script focuses on existence in English source files, not all locales.*
    *   `findMissingI18nKeys`:
        *   Recursively walks (`walk` function) through specified source directories (`components`, `src`).
        *   Reads `.ts`/`.tsx`/`.js`/`.jsx` files.
        *   Applies `i18nPatterns` regex to find key usages.
        *   For each found key, calls `checkKeyInLocales` to see if it exists in the corresponding English locale file.
        *   Collects results where keys are used but not found.
    *   Main function `main` calls `findMissingI18nKeys`, prints report of missing keys and the file using them, exits with 1 if missing keys are found.
*   **Utilities Used:** Node.js `fs` (sync versions), `path`, `process`.

### `scripts/update-contributors.js`

*(See code provided in chapter context)*
*   **Purpose:** Automatically updates the contributors list in the main `README.md` and localized READMEs by fetching data from the GitHub API.
*   **Usage:** `node scripts/update-contributors.js` (requires `GITHUB_TOKEN` env var for higher rate limits).
*   **Logic:**
    *   Defines GitHub API URL, README paths, and sentinel markers (`<!-- START CONTRIBUTORS SECTION -->`, `<!-- END ... -->`).
    *   `httpGet`: Helper using Node.js `https` module to perform GET requests (used for GitHub API).
    *   `parseLinkHeader`: Parses GitHub's pagination `Link` header.
    *   `fetchContributorsPage`: Fetches one page of contributors.
    *   `fetchContributors`: Fetches *all* contributors by following pagination links.
    *   `formatContributorsSection`: Takes the contributor list, filters bots, and generates a Markdown table displaying contributor avatars and links using a fixed number of columns.
    *   `readReadme`/`writeReadme`: Read/write the main README.
    *   `updateReadme`: Reads the README, finds the section between the sentinels, replaces it with the newly generated section, and writes the file back.
    *   `findLocalizedReadmes`: Finds `README.md` files within `locales/<lang>/` directories.
    *   `updateLocalizedReadme`: Reads, finds sentinels, replaces section, writes back for a specific localized README.
    *   `main` orchestrates fetching, formatting, and updating the main and all localized READMEs. Includes error handling.
*   **Utilities Used:** Node.js `https`, `fs` (async via `promisify`), `path`, `process`.

## Internal Implementation

*   **File System:** Node.js `fs`/`fs/promises` API, `glob`.
*   **Command Execution:** Node.js `child_process` (`spawn`, `exec`, `execSync`), `execa`.
*   **Argument Parsing:** `yargs`, `gluegun`, manual `process.argv`.
*   **Logging:** Node.js `console`, `chalk`.
*   **Path:** Node.js `path`.
*   **Network:** Node.js `https`, `axios`.
*   **YAML/JSON:** `js-yaml`, `JSON`.

## Modification Guidance

Involves adding new scripts/helpers or refining existing ones.

1.  **Adding a Release Script (`scripts/release.ts`):**
    *   **Purpose:** Automate version bumping, changelog generation, tagging, potentially publishing.
    *   **Utilities Needed:**
        *   Argument parsing (`yargs`) for version type (patch, minor, major).
        *   Git helpers (`gitUtils.ts`): Check for dirty state, get current branch, add/commit, tag, push.
        *   FS helpers: Read/write `package.json` to update version.
        *   Command execution (`runCommand`): Run `pnpm changelog` (if using conventional commits/changelog tool), `pnpm build:prod`, `pnpm package`, `pnpm publish`.
    *   **Workflow:** Check clean repo -> Get version bump type -> Update `package.json` -> Run `pnpm changelog` -> Commit changes -> Create Git tag -> Run `build:prod` -> Run `package` -> Run `publish`. Include confirmation steps or dry-run options.

2.  **Improving `find-missing-i18n-key.js`:**
    *   **Refine Regex:** Make `i18nPatterns` more robust to handle different ways `t()` might be called (e.g., with options object).
    *   **Use AST Parser:** Instead of regex, use `@typescript-eslint/parser` or Babel to parse TS/TSX code into an Abstract Syntax Tree. Traverse the AST to find `CallExpression` nodes for `t()` or `JSXOpeningElement` for `Trans` to extract keys more reliably. This is much more complex but less prone to regex errors.

3.  **Adding a Utility for Zod Schema to TypeScript Type Generation:**
    *   **Dependency:** Add `zod-to-ts` (`pnpm add -D zod-to-ts`).
    *   **Script (`scripts/generateSchemaTypes.ts`):**
        *   Use `glob` to find all files exporting Zod schemas (e.g., `src/schemas/*.ts`).
        *   For each schema file, import it dynamically or parse its exports.
        *   Use `zod-to-ts` library's functions to generate TypeScript `interface` or `type` definitions from the Zod schemas.
        *   Write the generated types to corresponding `.d.ts` files or a central types file.
    *   **`package.json`:** Add this script to the `prepare` step: `"prepare": "node ./scripts/prepare.js && node ./scripts/generateSchemaTypes.js"`. *(Note: This adds complexity compared to just using `z.infer` directly in code, but might be useful for specific scenarios).*

**Best Practices:**

*   **Reusability:** Extract common logic into `scripts/lib/`.
*   **Error Handling:** Scripts should fail clearly on error (`process.exit(1)`). Utilities should throw/reject critical errors. Use `scriptLogger`.
*   **Cross-Platform:** Use Node.js `path`. Use `execa`/`spawn` with arg arrays. Use `cross-env`. Use `rimraf` for deletion.
*   **Logging:** Use consistent, colored logging.
*   **Async:** Use `async/await`.
*   **Configuration:** Read via env vars, args, or files.
*   **Dependencies:** Use `devDependencies`.
*   **Clarity:** Write clear, commented scripts and utilities.

**Potential Pitfalls:**

*   **VS Code API Usage:** Cannot use `vscode` in standalone scripts.
*   **Path Resolution:** CWD issues. Use `path.resolve`, `__dirname`.
*   **Shell Command Failures:** Need reliable error checking (`execa` helps).
*   **Unhandled Rejections.**
*   **Environment Differences.**
*   **Dependencies:** Ensure `devDependencies` are installed where scripts run (e.g., in CI).

## Conclusion

Script Utilities provide essential helper functions that streamline the development, build, testing, evaluation, and maintenance processes for the Roo-Code project. By encapsulating common tasks like argument parsing, robust file system operations (`glob`, safe read/write, validation), reliable command execution (`execa`), translation validation, contributor updates (`https`), and consistent logging (`chalk`) into reusable modules (`scripts/lib/`, `evals/src/lib/`) tailored for the Node.js environment, they reduce code duplication, improve consistency, and make individual scripts cleaner and easier to maintain. They are crucial for automating repetitive developer tasks and ensuring project health.

Next, we examine the specific system used to build the Roo-Code extension itself: [Chapter 56: Build System](56_build_system.md).
---
# Chapter 56: Build System

Continuing from [Chapter 55: Script Utilities](55_script_utilities.md), which covered helpers used in various project scripts, this chapter focuses on the specific processes and tools used to compile, bundle, and package the Roo-Code extension for distribution and development: the **Build System**.

## Motivation: Transforming Source Code into a Deployable Extension

The Roo-Code source code, written primarily in TypeScript (`src/`) with React for the WebView UI (`webview-ui/`) and including assets like images (`images/`), locale files (`public/locales`, `src/i18n/locales`), NLS files (`package.nls.*.json`), and WASM modules (`node_modules/`), needs to be transformed into a format that VS Code can load and execute. This involves several steps:

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
    *   **Asset Copying:** Uses script utilities ([Chapter 55](55_script_utilities.md), [Chapter 42](42_file_system_utilities.md)) to copy `webview-ui/build/*` -> `dist/webview-ui/`, WASM files -> `dist/`, host locales (`src/i18n/locales/*`) -> `dist/i18n/locales/`, NLS files (`package.nls*.json`) -> `dist/`.
5.  **Output:** `dist/` contains `extension.js`, `webview-ui/`, host `i18n/locales/`, WASM files, NLS files.
6.  **Packaging:** Run `pnpm package`. `vsce` bundles `dist/` (respecting `.vscodeignore`) into `.vsix`.

## Key Concepts

1.  **`esbuild`:** Fast bundler for the **extension host** (`src/` -> `dist/extension.js`). Configured in `esbuild.js`.
2.  **Vite:** Build tool for the **WebView UI** (`webview-ui/` -> `webview-ui/build/`). Configured in `webview-ui/vite.config.ts`. ([Chapter 1](01_webview_ui.md)).
3.  **Build Script (`esbuild.js`):** Node.js orchestrator using `esbuild` API, `child_process`, and `fs` utilities ([Chapter 55](55_script_utilities.md), [Chapter 42](42_file_system_utilities.md)). Handles modes, asset copying.
4.  **`package.json` Scripts:** Defines commands (`build:prod`, `watch:host`, `compile`, `vscode:prepublish`, `package`) using `pnpm`, `cross-env`, `node`, `tsc`, `rimraf`, `vsce`.
5.  **Entry Points & Outputs:** Host (`src/extension.ts` -> `dist/extension.js`), WebView (`webview-ui/src/index.tsx` -> `dist/webview-ui/assets/*`).
6.  **Host Configuration (`esbuild.js`):** `platform: 'node'`, `format: 'cjs'`, `target: 'node16'`, `external: ['vscode']`, `bundle: true`, `minify`, `sourcemap`, `define`.
7.  **Asset Handling:** Critical build step performed by `esbuild.js` or plugins. Copies WASM, WebView build output, host locales (`src/i18n/locales`), NLS files to `dist/`. Note the distinction between WebView locales (from `public/locales`) and host locales (from `src/i18n/locales`).
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

    // Copy Locales for WebView UI -> dist/locales
    // NOTE: Previous chapters indicated webview locales might be bundled differently by Vite
    // If Vite bundles them, this copy might be unnecessary or need adjustment.
    // Assuming for now they might still be static assets in public/.
    const localeWebViewSrc = path.join(__dirname, "public", "locales");
    const localeWebViewDest = path.join(outDir, "locales");
    if (fs.existsSync(localeWebViewSrc)) { copyRecursiveSync(localeWebViewSrc, localeWebViewDest); log.info("Copied WebView locale files."); }

    // Copy Locales for Extension Host -> dist/i18n/locales
    const localeHostSrc = path.join(__dirname, "src", "i18n", "locales");
    const localeHostDest = path.join(outDir, "i18n", "locales");
    if (fs.existsSync(localeHostSrc)) {
        ensureDirSync(path.dirname(localeHostDest)); // Ensure parent 'i18n' exists
        copyRecursiveSync(localeHostSrc, localeHostDest);
        log.info("Copied Extension Host locale files.");
    } else { log.warn(`Host locales not found: ${localeHostSrc}`); }

    // Copy NLS files for Extension Host -> dist/
    const nlsFiles = fs.readdirSync(__dirname).filter(f => f.startsWith('package.nls') && f.endsWith('.json'));
    nlsFiles.forEach(f => fs.copyFileSync(path.join(__dirname, f), path.join(outDir, f)));
    log.info("Copied NLS files.");

    // Copy other assets (e.g., images -> dist/images)
    // ...

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
    // Add plugins here if used (e.g., for WASM copy)
    // plugins: [copyWasmFilesPlugin], // Example
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
                if (fs.existsSync(webviewBuildSrc)) {
                    ensureDirSync(webviewBuildDest);
                    copyRecursiveSync(webviewBuildSrc, webviewBuildDest);
                    log.info(`Copied webview build to ${webviewBuildDest}...`);
                } else { log.warn(`Webview build output not found: ${webviewBuildSrc}`); }

            } catch (error) { /* ... handle error ... */ throw error; }
        } else { log.info("Skipping webview build in host watch mode."); }

        // 4. Copy other static assets (if not handled by plugins)
        copyAssets(); // Copy locales, WASM (if needed), NLS, images etc.

		log.success(`Build finished! Output: ${outDir}`);
        if (isWatch) log.info("Watching for extension host changes...");

	} catch (error) { /* ... handle error, process.exit(1) ... */ }
}

// --- Run Build ---
build();
```

**Explanation:**

*   **Asset Copying:** `copyAssets` function now explicitly handles copying host locales (`src/i18n/locales`) to `dist/i18n/locales/` and NLS files (`package.nls*.json`) to `dist/`. It assumes WebView locales might still be in `public/locales` (adjust if Vite bundles them differently).
*   **Build Function:** Orchestrates cleaning, host build (`esbuild`), conditional WebView build (`vite`), conditional copying of Vite output, and final asset copying (`copyAssets`).

### Vite Configuration (`webview-ui/vite.config.ts`)

*(See code in Chapter 1)*
*   Configures Vite for React, Tailwind. Sets `build.outDir: "build"`. Crucially, it likely bundles the WebView's locale files (from `webview-ui/src/i18n/locales`) using `import.meta.glob` as shown in Chapter 50, meaning they don't need separate copying if they originate from within `webview-ui/src`. The copy step in `esbuild.js` for `public/locales` might be for other static assets or needs review based on the actual locale file location.

### `.vscodeignore` (Conceptual)

*(See code in Chapter 56 - Key Concepts)*
*   Excludes source, `node_modules`, config, tests, scripts, `webview-ui/build`.
*   **Includes `dist/`** (which contains `extension.js`, `webview-ui/`, `locales/`, `i18n/locales/`, WASM, NLS) and root metadata files.

## Internal Implementation

1.  **Trigger:** `pnpm build:prod`.
2.  **Cleanup:** `rimraf dist`.
3.  **Prepare:** `node scripts/prepare.js`.
4.  **Host Build:** `esbuild.build()` -> `dist/extension.js`.
5.  **WebView Build:** `execSync('pnpm --filter webview-ui build')` -> `vite build` -> `webview-ui/build/`.
6.  **Asset Copying:** `esbuild.js` copies `webview-ui/build/*` -> `dist/webview-ui/`, WASM -> `dist/`, `src/i18n/locales/*` -> `dist/i18n/locales/`, `package.nls*.json` -> `dist/`.
7.  **Packaging:** `vsce package` zips `dist/` + root files based on `.vscodeignore`.

## Modification Guidance

Involves changing targets, entry points, plugins, or asset handling.

1.  **Changing Target Node Version:** Modify `target` in `esbuild.js`.
2.  **Adding New Asset Type:** Modify `copyAssets` in `esbuild.js`. Update code references. Check `.vscodeignore`.
3.  **Adding esbuild Plugin:** Install, import, add to `plugins` in `esbuild.js`.
4.  **Configuring Vite Build:** Modify `webview-ui/vite.config.ts`.
5.  **Adding Pre-build Step:** Add command to `prepare` script in `package.json`.

**Best Practices:**

*   **Use Correct Tools:** `esbuild` (host), Vite (WebView).
*   **Clean Output (`dist/`):** Keep structured. Clean before builds.
*   **Separate Scripts:** Use `package.json` for orchestration.
*   **Environment Variables:** Use `NODE_ENV`, `define`. Handle build secrets via CI env vars.
*   **`external: ['vscode']`:** Essential for host.
*   **Asset Management:** Explicitly copy *all* runtime assets to `dist/`. Verify paths (WASM, Host Locales, WebView Locales if static, NLS, WebView Build).
*   **`.vscodeignore`:** Maintain carefully.
*   **Type Checking (`tsc --noEmit`):** Run separately (`compile` script).

**Potential Pitfalls:**

*   **Missing Assets in `dist/`:** Runtime errors. Double-check all asset types (WASM, Host Locales, WebView Build, NLS) are copied by `copyAssets`. Verify source paths.
*   **Incorrect `external`/`platform`.**
*   **Asset Path Issues:** Code referencing assets using incorrect relative paths post-build.
*   **`.vscodeignore` Errors.**
*   **Build Environment Differences.**
*   **Vite/Esbuild Conflicts.**

## Conclusion

The Build System, orchestrated via `esbuild.js` and `package.json` scripts, reliably transforms Roo-Code's source code into a functional and distributable VS Code extension. It leverages the speed of `esbuild` for the extension host and the rich features of Vite for the WebView UI. Key aspects include distinct configurations, handling development vs. production builds, injecting environment variables, and crucially, ensuring all necessary runtime assets (WASM, Host/WebView locales, WebView bundles, NLS files) are correctly copied to the `dist/` directory, ready for packaging with `vsce`. This automated system is fundamental for both efficient development and creating optimized, distributable extension builds.

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

