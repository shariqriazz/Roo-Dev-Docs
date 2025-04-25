# Chapter 45: Text Normalization Utilities

Continuing from [Chapter 44: Text Extraction Utilities](44_text_extraction_utilities.md), where we focused on reliably reading text content from files, this chapter examines utilities designed to clean and standardize text strings for consistent processing, comparison, or display: the **Text Normalization Utilities**.

## Motivation: Ensuring Consistency and Comparability of Text

Text data within Roo-Code comes from various sources: user input, file content read by utilities like `extractTextFromFile` ([Chapter 44: Text Extraction Utilities](44_text_extraction_utilities.md)), AI responses, diagnostic messages, terminal output, etc. This text can have inconsistencies that hinder processing or comparison:

*   **Whitespace:** Extra spaces, tabs, leading/trailing whitespace can affect parsing or comparisons.
*   **Line Endings:** Different operating systems use different line endings (CRLF `\r\n` on Windows, LF `\n` on Linux/macOS). AI models might generate text with either convention. Inconsistent line endings can break diffs ([Chapter 20: DiffViewProvider](20_diffviewprovider.md)), comparisons based on equality, line counting, and text processing logic that assumes a specific EOL character.
*   **Typographic Characters:** "Smart" quotes (curly quotes like `“ ” ‘ ’`), em dashes (`—`), ellipses (`…`), or non-breaking spaces (`&nbsp;` or U+00A0) might be introduced by users, editors, or AI models. These can cause issues in comparisons or when text is expected to use standard ASCII equivalents (straight quotes `"` `'`, hyphen `-`).
*   **HTML Entities:** AI responses, especially if sourced from web content or not properly constrained, might include HTML entities like `&lt;`, `&gt;`, `&amp;`. If this text is then processed expecting raw characters (e.g., parsing tool usage XML tags), these entities need to be converted back.
*   **Case Sensitivity:** Comparisons might need to be case-insensitive depending on the context.

Manually handling these variations everywhere text is processed is error-prone and repetitive. The Text Normalization Utilities provide a central set of functions to apply consistent cleaning and standardization rules to strings, making subsequent processing, comparison, and display more reliable.

**Central Use Case:** The [Chapter 20: DiffViewProvider](20_diffviewprovider.md) needs to compare the final content saved by the user (`editedContent`) with the content originally proposed by the AI (`newContent`) to detect if the user made any edits (`userEdits`). A simple string comparison (`editedContent === newContent`) might fail due to differences in line endings (e.g., user's editor saved with CRLF, AI generated with LF) or smart quotes, even if the actual textual content is identical.

Using Normalization:
1.  Before comparison, `DiffViewProvider.saveChanges` calls a normalization function like `normalizeString(editedContent)` and `normalizeString(newContent)`.
2.  The `normalizeString` utility (or specific functions it uses like `normalizeLineEndings`) converts all CRLF and CR line endings to LF (`\n`) and potentially replaces smart quotes with straight quotes.
3.  The comparison `normalizedEditedContent === normalizedNewContent` now accurately reflects whether the meaningful content differs, ignoring trivial variations.
4.  If they still differ, a diff patch representing `userEdits` is generated using the *normalized* strings to ensure the patch applies correctly regardless of original EOL differences.

## Key Concepts

1.  **Standardization Goal:** To transform text strings into a canonical format, often focusing on whitespace, line endings, and common typographic variations, enabling reliable processing and comparison.

2.  **`normalizeLineEndings(text)`:** Converts all line endings (CRLF `\r\n`, CR `\r`) in a string to a single LF (`\n`). This is crucial for comparing text from different sources or platforms, generating diffs, and ensuring consistent line counting. Uses simple, efficient string replacement (`replace(/\r\n/g, '\n').replace(/\r/g, '\n')`).

3.  **`normalizeString(text, options?)`:** A core utility that applies multiple normalizations based on options.
    *   Handles smart quotes (`“ ” ‘ ’` -> `" '`).
    *   Handles typographic characters (`…` -> `...`, `—`/`–` -> `-`, non-breaking space -> space).
    *   Handles extra whitespace (collapsing multiple spaces/tabs to one).
    *   Handles trimming (leading/trailing whitespace).
    *   Uses pre-defined `NORMALIZATION_MAPS` for character replacements.

4.  **`unescapeHtmlEntities(text)`:** Converts common HTML character entities (e.g., `&lt;`, `&gt;`, `&amp;`, `&quot;`, `&apos;`, `&#39;`) back into their literal characters (`<`, `>`, `&`, `"`, `'`). Essential when processing text received from sources (like AI responses) that might have encoded these characters, especially before parsing XML-like structures (e.g., tool calls).

5.  **Whitespace Trimming:** Standard `text.trim()` removes leading/trailing whitespace. Integrated into `normalizeString` via the `trim` option.

6.  **Case Normalization:** Not typically part of the primary normalization utilities (as case often matters in code/commands), but standard `text.toLowerCase()` or `text.toUpperCase()` can be used *after* other normalizations when specifically needed for case-insensitive comparisons.

7.  **Location (`src/utils/text-normalization.ts`):** These utility functions are grouped in a dedicated file for text standardization tasks.

## Using the Text Normalization Utilities

These are simple, pure functions imported and used where text consistency or canonical representation is required.

**Example 1: Line Ending & General Normalization (DiffViewProvider)**

```typescript
// --- File: src/integrations/editor/DiffViewProvider.ts ---
import { normalizeString, normalizeLineEndings } from "../../utils/text-normalization"; // Assuming location
import * as diff from "diff"; // Library for calculating diffs
import { formatResponse } from "../../core/prompts/responses";
// ...

// ... inside saveChanges ...
    const editedContent = updatedDocument.getText();

    // Compare final content with AI's proposal to detect user edits
    // Normalize EOL AND potentially other variations before comparison!
    // Using normalizeString might be better if smart quotes etc. could differ.
    // Or just normalize EOL if that's the only concern for comparison.
    const normalizedEditedContent = normalizeLineEndings(editedContent);
    const normalizedNewContent = normalizeLineEndings(this.newContent ?? "");

    let userEdits: string | undefined;
    if (normalizedEditedContent !== normalizedNewContent) {
        // Generate diff patch using normalized strings to ensure patch applies correctly
        userEdits = formatResponse.createPrettyPatch(
            this.relPath!.toPosix(), // Ensure path uses POSIX separators too
            normalizedNewContent, // Use normalized version for diff base
            normalizedEditedContent, // Use normalized version for diff target
        );
    }
    // ... return results ...
```
*Explanation:* `normalizeLineEndings` is applied to both the user's final content and the AI's proposed content before they are compared (`!==`) or used to generate a diff patch via `createPrettyPatch`. This ensures the comparison and diff are based on content changes, not just EOL differences. Using the broader `normalizeString` might be even safer if smart quotes or other typographic differences could arise.

**Example 2: Unescaping HTML Entities (Command Validation / Tool Input)**

```typescript
// --- File: src/core/tools/executeCommandTool.ts ---
import { unescapeHtmlEntities } from "../../utils/text-normalization"; // Import the utility
import { CommandValidator } from "../validation/commandValidation"; // Conceptual validator
import { formatResponse } from "../prompts/responses";
// ...

export async function executeCommandTool(...) {
    let command: string | undefined = block.params.command;
    // ... (handle partial, missing command) ...

    if (!command) { /* ... */ return; }

    // *** Unescape HTML entities before validation or execution ***
    // AI might output '<' as '&lt;' if not careful
    command = unescapeHtmlEntities(command);

    // --- Validate Command ---
    const validationError = await CommandValidator.validate(command, contextProxy);
    if (validationError) {
        // ... handle validation error ...
        return;
    }
    // --- Ask Approval ---
    const didApprove = await askApproval("command", command); // Pass unescaped command
    // ...
    // --- Execute Command ---
    const [userRejected, result] = await executeCommand(cline, command, customCwd); // Pass unescaped command
    // ...
}
```
*Explanation:* The command string received from the AI (`block.params.command`) might contain HTML entities (e.g., if the AI included `<` or `>` within an argument or description it generated). `unescapeHtmlEntities` converts `&lt;`, `&gt;`, `&amp;`, etc., back to `<`, `>`, `&` before the command is passed to `CommandValidator.validate` or sent to the terminal via `executeCommand`. This ensures the command is interpreted correctly.

**Example 3: Normalizing for Similarity Check (Diff Strategy)**

```typescript
// --- File: src/core/diff/strategies/multi-search-replace.ts ---
import { distance } from "fastest-levenshtein";
import { normalizeString } from "../../../utils/text-normalization"; // Import utility

function getSimilarity(original: string, search: string): number {
	if (search === "") return 0;

	// Use normalizeString to handle smart quotes, extra whitespace, etc.
	// before calculating similarity. Exclude line ending changes maybe?
	const normalizedOriginal = normalizeString(original, { trim: true, extraWhitespace: true, smartQuotes: true, typographicChars: true });
	const normalizedSearch = normalizeString(search, { trim: true, extraWhitespace: true, smartQuotes: true, typographicChars: true });

	if (normalizedOriginal === normalizedSearch) return 1;

	const dist = distance(normalizedOriginal, normalizedSearch); // Levenshtein distance
	const maxLength = Math.max(normalizedOriginal.length, normalizedSearch.length);
	return 1 - dist / maxLength; // Similarity ratio
}
```
*Explanation:* Before calculating the Levenshtein distance to find the best fuzzy match for a search block, `normalizeString` is used on both the original document chunk and the AI-provided search chunk. This makes the comparison robust against variations in whitespace, smart quotes, or other typographic differences that shouldn't affect the similarity score for the intended content.

## Code Walkthrough

### Text Normalization Utilities (`src/utils/text-normalization.ts`)

*(See full code in Key Concepts section)*

```typescript
// --- File: src/utils/text-normalization.ts ---

/** Character maps for normalization */
export const NORMALIZATION_MAPS = {
	SMART_QUOTES: { /* ... “ -> ", etc. ... */ },
	TYPOGRAPHIC: { /* ... … -> ..., — -> -, etc. ... */ },
};

/** Options for normalizeString */
export interface NormalizeOptions {
	smartQuotes?: boolean;
	typographicChars?: boolean;
	extraWhitespace?: boolean;
	trim?: boolean;
}
const DEFAULT_OPTIONS: NormalizeOptions = { /* ... defaults to true ... */ };

/** Normalizes a string based on options */
export function normalizeString(str: string, options: NormalizeOptions = DEFAULT_OPTIONS): string {
	const opts = { ...DEFAULT_OPTIONS, ...options };
	let normalized = str;
	if (!normalized) return ""; // Handle null/undefined/empty

	// Replace smart quotes
	if (opts.smartQuotes) {
		for (const [smart, regular] of Object.entries(NORMALIZATION_MAPS.SMART_QUOTES)) {
			normalized = normalized.replace(new RegExp(smart, "g"), regular);
		}
	}
	// Replace typographic characters
	if (opts.typographicChars) {
		for (const [typographic, regular] of Object.entries(NORMALIZATION_MAPS.TYPOGRAPHIC)) {
			normalized = normalized.replace(new RegExp(typographic, "g"), regular);
		}
	}
	// Normalize whitespace (multiple to single space)
	if (opts.extraWhitespace) {
		normalized = normalized.replace(/\s+/g, " ");
	}
	// Trim whitespace from start/end
	if (opts.trim) {
		normalized = normalized.trim();
	}
	return normalized;
}

/** Normalizes line endings to LF ('\n') */
export function normalizeLineEndings(text: string | undefined | null): string {
	if (!text) return "";
	return text.replace(/\r\n/g, '\n').replace(/\r/g, '\n');
}

/** Unescapes common HTML entities */
export function unescapeHtmlEntities(text: string | undefined | null): string {
	if (!text) return "";
	return text
		.replace(/&lt;/g, '<')
		.replace(/&gt;/g, '>')
		.replace(/&quot;/g, '"')
		.replace(/&#39;/g, "'") // Numeric entity for single quote
		.replace(/&apos;/g, "'") // Named entity for single quote
		.replace(/&amp;/g, '&'); // Ampersand last
}

// (Conceptual stripNonPrintableChars and others omitted for brevity,
// assume they use regex similar to Chapter 45 example if needed)
```

**Explanation:**

*   **`NORMALIZATION_MAPS`:** Defines constant objects mapping smart quotes and other typographic characters to their standard ASCII equivalents.
*   **`normalizeString`:** The main workhorse function. It takes the input string and options. It iteratively applies replacements based on the enabled options (`smartQuotes`, `typographicChars`, `extraWhitespace`, `trim`) using string `replace` with regex or direct lookups from `NORMALIZATION_MAPS`.
*   **`normalizeLineEndings`:** Simple chained `replace` for `\r\n` and `\r` to `\n`.
*   **`unescapeHtmlEntities`:** Chained `replace` calls for common named (`&lt;`, `&gt;`, etc.) and numeric (`&#39;`) HTML entities. `&amp;` is crucial to replace last.

## Internal Implementation

These utilities are pure functions operating directly on strings using built-in JavaScript string methods (`replace`, `trim`) and regular expressions. There are no complex internal states or dependencies beyond the string input and the defined maps/regex.

**Step-by-Step (`normalizeString` - default options):**

1.  Input: `str = "“Hello…\r\n World”  "`
2.  `opts = { smartQuotes: true, typographicChars: true, extraWhitespace: true, trim: true }`.
3.  `normalized = str`.
4.  **Smart Quotes:**
    *   `normalized = normalized.replace(/\u201C/g, '"')` -> `"Hello…\r\n World”  "`
    *   `normalized = normalized.replace(/\u201D/g, '"')` -> `"Hello…\r\n World"  "`
    *   (No single smart quotes to replace)
5.  **Typographic Chars:**
    *   `normalized = normalized.replace(/\u2026/g, '...')` -> `"Hello...\r\n World"  "`
    *   (No em/en dash, nbsp to replace)
6.  **Extra Whitespace:**
    *   `normalized = normalized.replace(/\s+/g, ' ')` -> `"Hello... World" "` (Replaces `\r\n` and multiple spaces with single space).
7.  **Trim:**
    *   `normalized = normalized.trim()` -> `"Hello... World"`
8.  Return `"Hello... World"`.

**Step-by-Step (`unescapeHtmlEntities`):**

1.  Input: `text = "Use &lt;tag&gt; &amp; check options."`
2.  `string1 = text.replace(/&lt;/g, '<')` -> `"Use <tag&gt; &amp; check options."`
3.  `string2 = string1.replace(/&gt;/g, '>')` -> `"Use <tag> &amp; check options."`
4.  `string3 = string2.replace(/&quot;/g, '"')` -> `"Use <tag> &amp; check options."` (No change)
5.  `string4 = string3.replace(/&#39;/g, "'")` -> `"Use <tag> &amp; check options."` (No change)
6.  `string5 = string4.replace(/&apos;/g, "'")` -> `"Use <tag> &amp; check options."` (No change)
7.  `string6 = string5.replace(/&amp;/g, '&')` -> `"Use <tag> & check options."`
8.  Return `"Use <tag> & check options."`.

## Modification Guidance

Modifications usually involve adding new normalization rules or adjusting the behavior of existing ones.

1.  **Adding a New Typographic Character Mapping:**
    *   **Locate:** Edit the `NORMALIZATION_MAPS.TYPOGRAPHIC` object in `src/utils/text-normalization.ts`.
    *   **Add:** Add a new key-value pair, e.g., `"\u2022": "*"` to replace bullet points with asterisks.
    *   **Effect:** `normalizeString` will automatically pick up and apply this new replacement when `typographicChars` option is true.

2.  **Making `normalizeString` Keep Line Breaks:**
    *   **Modify:** Change the regex in the `extraWhitespace` step of `normalizeString`. Instead of `/\s+/g` (which matches all whitespace including newlines), use a regex that only targets horizontal whitespace like spaces and tabs, e.g., `/[ \t]+/g`.
        ```typescript
        // Inside normalizeString
        if (opts.extraWhitespace) {
            // Replace multiple horizontal spaces/tabs with single space, keep newlines
            normalized = normalized.replace(/[ \t]+/g, " ");
        }
        // Apply EOL normalization separately if needed
        // normalized = normalizeLineEndings(normalized); // If desired alongside whitespace collapse
        if (opts.trim) {
            // Trim horizontal space from start/end of each line? More complex.
            // Or just trim overall start/end:
            normalized = normalized.trim();
        }
        ```
    *   **Considerations:** Decide if line ending normalization should happen before, after, or be excluded entirely when modifying whitespace handling. The `normalizeAndClean` example shows one way to combine specific steps.

3.  **Adding Diacritic Removal Utility:** (As shown in Ch45 thought process)
    *   Add `diacritics` library. Create `removeDiacritics` function using `import { remove } from 'diacritics';`.

**Best Practices:**

*   **Targeted Use:** Apply normalization functions deliberately where needed (comparison, specific processing) rather than globally on all text.
*   **Immutability:** Ensure functions return new strings.
*   **Performance:** Simple replacements are fast. Avoid very complex regex on large strings in hot paths.
*   **Order Matters:** The order of replacements (e.g., `&amp;` last in `unescapeHtmlEntities`) or normalizations (e.g., `trim` vs. `normalizeLineEndings`) can be important.
*   **Clear Naming:** Use descriptive function names.
*   **Options:** Use options objects (like in `normalizeString`) for flexibility when a function performs multiple types of normalization.

**Potential Pitfalls:**

*   **Over-Normalization:** Removing potentially significant whitespace (like code indentation) or characters (like typographic quotes within user-facing text) when not intended.
*   **Incorrect Regex:** Errors in regex leading to incorrect replacements.
*   **EOL Issues:** Failing to use `normalizeLineEndings` before line-sensitive operations (diffing, comparison, splitting by line).
*   **HTML Entity Complexity:** `unescapeHtmlEntities` only handles common entities. Full HTML decoding requires a library.
*   **Unicode:** Simple regex might not handle all Unicode whitespace or typographic characters correctly. More robust handling might require Unicode property escapes (`\p{...}`) in regex or dedicated libraries.

## Conclusion

The Text Normalization Utilities provide essential functions for ensuring text consistency and reliability within Roo-Code. By offering standard ways to handle line endings (`normalizeLineEndings`), common typographic variations and whitespace (`normalizeString`), and HTML entities (`unescapeHtmlEntities`), these utilities simplify comparison logic (e.g., in `DiffViewProvider`), improve data consistency for internal processing, and help prepare text received from potentially inconsistent sources (like AI responses) before further parsing or execution. They are fundamental building blocks applied in various parts of the extension that handle text comparison, diffing, or preparation for display, parsing, or execution.

Having covered various foundational utilities, we now look at how Roo-Code renders structured text, particularly Markdown, within its user interface: [Chapter 46: Markdown Rendering](46_markdown_rendering.md).


