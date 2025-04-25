# Chapter 49: OAuth Helpers

Continuing from [Chapter 48: Configuration Rules](48_configuration_rules.md), which discussed enforcing constraints within settings and project standards, this chapter focuses on a specific mechanism used for integrating with external services that require user authentication: **OAuth Helpers**.

## Motivation: Securely Connecting to External Services

Roo-Code might need to integrate with external services that require user authentication via OAuth 2.0. Examples could include:

*   Connecting to a code hosting platform (like GitHub, GitLab) for specific features (e.g., creating issues, fetching private repository information).
*   Integrating with project management tools (e.g., Jira, Asana).
*   Authenticating with certain cloud providers or APIs that use OAuth for user-delegated permissions.

The OAuth 2.0 Authorization Code flow (often enhanced with PKCE) is the standard for third-party applications like VS Code extensions. This flow typically involves:
1.  Redirecting the user to the service's authorization page in their browser.
2.  User logs in and grants permission.
3.  The service redirects the user back to a specific `redirect_uri` registered by the application, providing an authorization `code`.
4.  The application needs to capture this `code` and securely exchange it (along with a client secret or PKCE verifier) for an access token and potentially a refresh token by making a backend request to the service's token endpoint.
5.  The received tokens need to be stored securely (e.g., using `vscode.ExtensionContext.secrets`) for future API calls.

Handling this flow within a VS Code extension presents challenges:
*   **Redirect URI:** Extensions don't typically run a web server to receive redirects. VS Code provides mechanisms like the `vscode.authentication` API or custom URI handlers (`vscode.window.registerUriHandler`).
*   **State Management:** A `state` parameter is needed to prevent Cross-Site Request Forgery (CSRF) attacks and correlate the callback with the initial request.
*   **Token Exchange:** The exchange for tokens usually requires a client secret (which **cannot** be safely stored in the extension) or PKCE. This step often needs to be proxied through a secure backend server controlled by the extension publisher, especially if a client secret is involved.
*   **Secure Token Storage:** Access and refresh tokens are highly sensitive and must be stored using VS Code's secure `secrets` API.
*   **Token Refresh:** Access tokens expire. Logic is needed to automatically use the refresh token to obtain a new access token without requiring the user to re-authenticate constantly.

The **OAuth Helpers** (conceptually located in `src/services/oauth/` or `src/utils/oauth/`) provide utilities and potentially manage the state needed to facilitate these OAuth flows within the VS Code extension environment, **strongly preferring the built-in `vscode.authentication` API** whenever possible and providing guidance or minimal structure for custom flows otherwise.

*Note: Based on the provided project structure, explicit OAuth helpers beyond potentially using the built-in `vscode.authentication` API might not be extensively implemented yet for custom providers. The examples for Glama, OpenRouter, Requesty in `webview-ui/src/oauth/urls.ts` and `src/activate/handleUri.ts` seem to use a simpler callback mechanism, potentially a variation of the custom flow or a provider-specific one, which might not fully represent a standard OAuth 2.0 PKCE flow. This chapter focuses on the general best practices and the preferred VS Code approach.*

**Central Use Case (Hypothetical - Using `vscode.authentication` for GitHub):** Roo-Code needs to authenticate with GitHub to access private repository data.

1.  User clicks "Connect GitHub Account" in Roo-Code Settings ([Chapter 35: Settings UI Components (WebView)](35_settings_ui_components__webview_.md)).
2.  The UI triggers a command (e.g., `roo-code.connectGitHub`).
3.  The command handler executes:
    ```typescript
    import * as vscode from 'vscode';
    import { telemetryService } from '../services/telemetry/TelemetryService'; // Assuming singleton telemetry
    import { ContextProxy } from '../core/config/ContextProxy'; // Assuming access to ContextProxy
    import { SECRET_STATE_KEYS } from '../schemas'; // Key definitions
    import { ClineProvider } from '../core/webview/ClineProvider'; // For state refresh

    async function connectGitHub(contextProxy: ContextProxy) {
        telemetryService.captureEvent('auth.github.start');
        try {
            const scopes = ['repo', 'read:user', 'gist']; // Define necessary scopes
            // Use VS Code's built-in authentication provider
            const session = await vscode.authentication.getSession('github', scopes, { createIfNone: true });

            if (session) {
                const token = session.accessToken;
                const tokenKey = 'githubAccessToken';
                // Ensure key is defined in SECRET_STATE_KEYS before storing
                if (!(SECRET_STATE_KEYS as readonly string[]).includes(tokenKey)) {
                     throw new Error(`Configuration error: Secret key ${tokenKey} not defined.`);
                }
                await contextProxy.setValue(tokenKey, token);
                vscode.window.showInformationMessage(`Successfully connected to GitHub as ${session.account.label}!`);
                telemetryService.captureEvent('auth.github.success');
                // Trigger UI state update
                await ClineProvider.postStateToAllWindows(); // Conceptual static method
            } else {
                 vscode.window.showWarningMessage('GitHub authentication cancelled.');
                 telemetryService.captureEvent('auth.github.cancelled');
            }
        } catch (error: any) {
            vscode.window.showErrorMessage(`GitHub authentication failed: ${error.message}`);
            telemetryService.captureError('auth.github.failed', { error_message: error.message });
        }
    }
    ```
4.  **VS Code Magic:** `vscode.authentication.getSession('github', ...)` handles the entire flow securely: initiates redirect, intercepts callback, exchanges code (using VS Code's client secret), stores/refreshes tokens internally.
5.  **Token Storage:** The handler receives the `accessToken` and stores it using `ContextProxy` ([Chapter 11: ContextProxy](11_contextproxy.md)) via a predefined secret key.
6.  **Completion:** The UI is updated via `postStateToAllWindows` to reflect the connected state.

## Key Concepts

1.  **OAuth 2.0 Flows:** Primarily Authorization Code Grant with PKCE is recommended for public clients like extensions. Implicit Grant is discouraged. Flows requiring client secrets directly in the extension are insecure.
2.  **`vscode.authentication` API (Preferred):**
    *   **`vscode.authentication.getSession(providerId, scopes, options)`:** The core function. `providerId` must match a built-in provider (e.g., `'github'`, `'microsoft'`) or one registered by another extension.
    *   **Benefits:** Handles the entire flow securely, including UI prompts, browser redirects, callback interception, state/PKCE management, secure token exchange (using secrets held by VS Code or the auth provider extension), secure token storage, and automatic background token refresh. **This is the most secure and recommended approach.**
3.  **Custom Flow (using `registerUriHandler` - Use with Extreme Caution):** Only necessary for providers *not* supported by `vscode.authentication`.
    *   **`vscode.window.registerUriHandler`:** Registers a handler for a custom URI scheme (e.g., `vscode://YourPublisher.YourExtension/authCallback`). The `redirect_uri` sent to the OAuth provider *must* exactly match this.
    *   **`vscode.env.openExternal`:** Opens the provider's authorization URL in the user's browser.
    *   **PKCE Implementation:** **Mandatory** for security. Requires manual generation (`crypto` module) and handling of `code_verifier` and `code_challenge`.
    *   **State Management:** Requires manual generation, temporary storage (e.g., `Map` keyed by state), and verification of the `state` parameter to prevent CSRF.
    *   **Token Exchange:** Requires making an HTTP POST request (e.g., `axios`) from the extension to the provider's token endpoint using the `code` and `code_verifier`. **Never include a client secret directly in the extension.**
    *   **Backend Proxy for Secrets:** If the provider *absolutely requires* a client secret for token exchange, this step **MUST** be proxied through a secure backend server controlled by the extension publisher. The extension sends the `code` to the proxy; the proxy adds the secret and performs the exchange.
    *   **Secure Token Storage:** Requires explicitly using `vscode.ExtensionContext.secrets.store` via `ContextProxy`.
    *   **Token Refresh:** Requires manually implementing the entire refresh token flow (detecting expired tokens, making refresh requests, storing new tokens). This is complex and error-prone.
4.  **PKCE (Proof Key for Code Exchange):** Security mechanism for Authorization Code Grant. Generate `code_verifier`, create `code_challenge` (SHA256 hash, base64url encoded). Send challenge in auth request, send verifier in token request.
5.  **Secure Token Storage:** Use `vscode.ExtensionContext.secrets` via `ContextProxy` ([Chapter 11: ContextProxy](11_contextproxy.md)). Define keys in `SECRET_STATE_KEYS` ([Chapter 40: Schemas (Zod)](40_schemas__zod_.md)).
6.  **`webview-ui/src/oauth/urls.ts`:** Contains helper functions (`getGlamaAuthUrl`, `getOpenRouterAuthUrl`, `getRequestyAuthUrl`) to construct the authorization URLs for specific integrated providers, including encoding the callback URL (`getCallbackUrl`) which uses the VS Code URI scheme (`vscode://<publisher>.<extensionName>/<provider>`). This implies these integrations likely use the **Custom Flow** with `registerUriHandler`.
7.  **`src/activate/handleUri.ts`:** Contains the `handleUri` function, likely registered as the `UriHandler` for the extension's custom scheme. It parses the callback URI path and query parameters (`code`), determines the provider based on the path, and calls provider-specific handler functions within `ClineProvider` (e.g., `visibleProvider.handleGlamaCallback(code)`). These `handle...Callback` methods would contain the logic for the token exchange (hopefully using PKCE and not client secrets).

## Using OAuth Helpers

The primary patterns involve either using `vscode.authentication` or orchestrating the custom flow.

**Pattern 1: Using `vscode.authentication` (Preferred)**

1.  Define a command (e.g., `roo-code.connectGitHub`).
2.  Implement the command handler (see Central Use Case code).
3.  Call `vscode.authentication.getSession` with provider ID and scopes.
4.  Handle the returned `session` (store `accessToken` via `ContextProxy` into secrets) or cancellation/error.
5.  Update UI state.

**Pattern 2: Using Custom Flow (e.g., Glama, OpenRouter, Requesty via `handleUri`)**

1.  **UI Trigger:** A button/link in the WebView (`ApiOptions.tsx`) constructs the auth URL using helpers from `webview-ui/src/oauth/urls.ts` (e.g., `getGlamaAuthUrl(uriScheme)`) and opens it (likely via an `<a>` tag or `window.open`).
    ```typescript
    // In ApiOptions.tsx (or similar)
    <VSCodeButtonLink href={getGlamaAuthUrl(uriScheme)} ...>Connect Glama</VSCodeButtonLink>
    ```
2.  **External Auth:** User authenticates with the service (Glama, OpenRouter, Requesty).
3.  **Redirect:** Service redirects to `vscode://rooveterinaryinc.roo-cline/<provider>?code=...`.
4.  **VS Code Routing:** VS Code activates the extension (if needed) and routes the URI to the handler registered in `extension.ts`.
    ```typescript
    // In extension.ts activate function
    context.subscriptions.push(vscode.window.registerUriHandler({ handleUri }));
    ```
5.  **`handleUri` (`src/activate/handleUri.ts`):**
    *   Parses the URI path (`/glama`, `/openrouter`) and query (`code`).
    *   Identifies the active `ClineProvider`.
    *   Calls the provider-specific callback method, e.g., `visibleProvider.handleGlamaCallback(code)`.
6.  **`ClineProvider.handle<Provider>Callback` (Conceptual):**
    *   This method performs the **token exchange**. It needs to make an HTTP POST request to the provider's token endpoint.
    *   **Crucially, it should use PKCE if supported.** It would need access to the `code_verifier` stored temporarily when the flow was initiated (this initiation part is missing from the provided structure but is necessary for PKCE/state).
    *   **If PKCE is not used and a client secret is required, this step MUST be proxied through a secure backend.**
    *   On successful exchange, it receives tokens.
    *   It stores the tokens securely using `contextProxy.setValue(...)` (keys must be in `SECRET_STATE_KEYS`).
    *   It updates the relevant API configuration profile ([Chapter 9: ProviderSettingsManager](09_providersettingsmanager.md)).
    *   It calls `postStateToAllWindows()` to update the UI.
    *   Handles errors during the exchange.

## Code Walkthrough

### Custom Flow URL Generation (WebView)

```typescript
// --- File: webview-ui/src/oauth/urls.ts ---
export function getCallbackUrl(provider: string, uriScheme?: string): string {
    // Use the extension's unique ID for the callback scheme/authority
	const callbackUrl = `${uriScheme || "vscode"}://rooveterinaryinc.roo-cline/${provider}`
	// URL-encode the callback URL for use as a query parameter
	return encodeURIComponent(callbackUrl)
}

// Construct auth URLs for specific providers, passing the encoded callback URL
export function getGlamaAuthUrl(uriScheme?: string): string {
	return `https://glama.ai/oauth/authorize?callback_url=${getCallbackUrl("glama", uriScheme)}`
}

export function getOpenRouterAuthUrl(uriScheme?: string): string {
	// Note: OpenRouter uses 'callback_url' parameter name
	return `https://openrouter.ai/auth?callback_url=${getCallbackUrl("openrouter", uriScheme)}`
}

export function getRequestyAuthUrl(uriScheme?: string): string {
	return `https://app.requesty.ai/oauth/authorize?callback_url=${getCallbackUrl("requesty", uriScheme)}`
}
```

**Explanation:**

*   `getCallbackUrl`: Constructs the VS Code custom URI callback URL (e.g., `vscode://rooveterinaryinc.roo-cline/glama`) and URL-encodes it. The `rooveterinaryinc.roo-cline` part is the unique extension ID from `package.json`.
*   `get<Provider>AuthUrl`: Creates the full authorization URL for each specific service, appending the encoded callback URL as a query parameter (`callback_url`). These URLs are used in the Settings UI to initiate the respective flows.

### Custom Flow URI Handling (Extension Host)

```typescript
// --- File: src/activate/handleUri.ts ---
import * as vscode from "vscode";
import { ClineProvider } from "../core/webview/ClineProvider";
import { logger } from "../utils/logging";

/**
 * Handles incoming URIs registered for the extension's custom scheme.
 * Parses the path and query parameters to route OAuth callbacks.
 */
export const handleUri = async (uri: vscode.Uri) => {
	logger.info(`Handling incoming URI: ${uri.toString(true)}`, { ctx: "URIHandler" }); // true hides query
	const path = uri.path; // e.g., "/glama", "/openrouter"
	// Use URLSearchParams for robust query parsing, handle '+' -> space if needed
	const query = new URLSearchParams(uri.query); // No need for replace with URLSearchParams
	const code = query.get("code");
    // TODO: Add state parameter handling for CSRF protection!
    // const state = query.get("state");
    // const error = query.get("error");

    // Find the currently visible/active ClineProvider instance
    // This assumes the user initiated the flow from a visible panel.
	const visibleProvider = ClineProvider.getVisibleInstance(); // Conceptual static accessor
	if (!visibleProvider) {
        logger.warn("Received OAuth callback URI but no visible Roo Code panel found.", { ctx: "URIHandler" });
        vscode.window.showErrorMessage("Could not complete authentication. Please ensure the Roo Code panel is visible.");
		return;
	}

    // TODO: Verify received 'state' parameter against stored value for this provider

    if (!code) {
        const error = query.get("error");
        const errorDesc = query.get("error_description");
        logger.error(`OAuth callback missing code. Error: ${error}, Desc: ${errorDesc}`, { ctx: "URIHandler" });
        vscode.window.showErrorMessage(`Authentication failed: ${errorDesc || error || "Authorization code missing."}`);
        return;
    }

	// Route based on the path segment corresponding to the provider
	switch (path) {
		case "/glama": {
			logger.info(`Routing Glama callback with code...`, { ctx: "URIHandler" });
			await visibleProvider.handleGlamaCallback(code); // Delegate to provider method
			break;
		}
		case "/openrouter": {
			logger.info(`Routing OpenRouter callback with code...`, { ctx: "URIHandler" });
			await visibleProvider.handleOpenRouterCallback(code); // Delegate to provider method
			break;
		}
		case "/requesty": {
			logger.info(`Routing Requesty callback with code...`, { ctx: "URIHandler" });
			await visibleProvider.handleRequestyCallback(code); // Delegate to provider method
			break;
		}
		default:
            logger.warn(`Received callback for unknown path: ${path}`, { ctx: "URIHandler" });
			break;
	}
};

// --- File: src/extension.ts --- (Registration Excerpt)
// import { handleUri } from "./activate/handleUri";
//
// export async function activate(context: vscode.ExtensionContext) {
//     // ... other setup ...
//     // Register the URI handler
//     context.subscriptions.push(vscode.window.registerUriHandler({ handleUri }));
//     // ...
// }
```

**Explanation:**

*   `handleUri`: This function is registered via `vscode.window.registerUriHandler` during activation.
*   It parses the incoming `uri`'s path and query (`code`, `state`, `error`).
*   **Crucially, it needs to validate the `state` parameter (TODO added).**
*   It finds the active `ClineProvider` instance (needed to access `ContextProxy`, `ProviderSettingsManager`, and trigger UI updates).
*   It uses a `switch` on the `uri.path` to determine which provider the callback is for.
*   It calls a corresponding method on the `ClineProvider` instance (e.g., `handleGlamaCallback`), passing the extracted authorization `code`.

### Token Exchange (`ClineProvider` - Conceptual)

```typescript
// --- File: src/core/webview/ClineProvider.ts --- (Conceptual Callbacks)
import { ContextProxy } from "../config/ContextProxy";
import { ProviderSettingsManager } from "../config/ProviderSettingsManager";
import { logger } from "../../utils/logging";
import axios from "axios"; // For making HTTP requests

export class ClineProvider /* ... */ {
    // ... constructor, properties (contextProxy, providerSettingsManager) ...

    // Example for Glama - NEEDS ACTUAL ENDPOINT AND PARAMETERS
    async handleGlamaCallback(code: string): Promise<void> {
        logger.info("Exchanging Glama authorization code for token...", { ctx: "OAuth.Glama" });
        const tokenUrl = "https://glama.ai/oauth/token"; // Placeholder URL
        // TODO: Implement PKCE - Requires storing verifier associated with initial request
        const pkceVerifier = /* Retrieve stored verifier */;
        const clientId = /* Glama Client ID from config/env */;

        try {
            const response = await axios.post(tokenUrl, new URLSearchParams({
                grant_type: "authorization_code",
                code: code,
                redirect_uri: `vscode://rooveterinaryinc.roo-cline/glama`,
                client_id: clientId,
                // code_verifier: pkceVerifier, // Include if using PKCE
            }).toString(), {
                headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
            });

            if (response.data?.access_token) {
                const accessToken = response.data.access_token;
                const refreshToken = response.data.refresh_token; // Optional
                logger.info("Glama token exchange successful.", { ctx: "OAuth.Glama" });

                // Store securely via ProviderSettingsManager/ContextProxy
                // Find the Glama profile, update its key, save it
                const profileName = /* Find relevant Glama profile name */;
                const currentSettings = await this.providerSettingsManager.getConfigByName(profileName); // Needs method
                if (currentSettings) {
                    currentSettings.glamaApiKey = accessToken; // Assuming key stored here
                    // Store refresh token if applicable
                    // currentSettings.glamaRefreshToken = refreshToken;
                    await this.providerSettingsManager.saveConfig(profileName, currentSettings);
                } else { throw new Error("Glama profile not found to store token."); }

                vscode.window.showInformationMessage("Successfully connected to Glama!");
                await ClineProvider.postStateToAllWindows(); // Update UI
            } else {
                throw new Error("Token exchange response did not contain access_token.");
            }
        } catch (error: any) {
            logger.error("Glama token exchange failed", { error: error.response?.data || error.message, ctx: "OAuth.Glama" });
            vscode.window.showErrorMessage(`Glama connection failed: ${error.response?.data?.error_description || error.message}`);
        }
    }

    // Implement similar handleOpenRouterCallback, handleRequestyCallback methods...
    // Each needs the correct token endpoint URL and parameters.
    // Ensure PKCE is used if supported, and handle client secrets via backend proxy if required.
}
```

**Explanation:**

*   These conceptual methods within `ClineProvider` are called by `handleUri`.
*   They contain the provider-specific logic for the **token exchange**.
*   They construct the request payload (using `URLSearchParams`) including `grant_type`, `code`, `redirect_uri`, `client_id`, and crucially the PKCE `code_verifier` (which needs to be retrieved from temporary storage linked to the initial request's `state`).
*   They use `axios.post` to call the provider's token endpoint.
*   **Security:** This example assumes PKCE is sufficient. If a client secret is needed, this `axios.post` call **must be replaced** with a call to a secure backend proxy service.
*   On success, they extract tokens, find the relevant API profile using `ProviderSettingsManager`, update the stored token using `saveConfig` (which persists to `secrets`), show a success message, and refresh the UI state.
*   Error handling reports failures to the user and logs details.

## Internal Implementation

*   **`vscode.authentication`:** Relies on VS Code's core implementation and auth provider extensions. Securely manages the entire flow.
*   **Custom Flow:** Manual orchestration using VS Code APIs (`openExternal`, `registerUriHandler`), Node.js modules (`crypto`, `axios`), and temporary state management (`Map`) linked by the `state` parameter. Token exchange happens via HTTP POST. Secure storage uses `context.secrets`.

**Sequence Diagrams:**

*   **`vscode.authentication.getSession`:** *(See diagram in Chapter 49 - Internal Implementation)*
*   **Custom URI Handler Flow:** *(See diagram in Chapter 49 - Internal Implementation)*

## Modification Guidance

Modifications typically involve adding support for a new OAuth provider.

1.  **Integrating New Service via `vscode.authentication` (Preferred):**
    *   **Check ID:** Verify VS Code or trusted extension provides an ID (e.g., `'google'`).
    *   **Scopes:** Find required OAuth scopes.
    *   **Command:** Implement handler calling `vscode.authentication.getSession` with ID/scopes.
    *   **Schema:** Add secret key (e.g., `googleAccessToken`) to `SECRET_STATE_KEYS`/`SecretState`.
    *   **Store:** Use `contextProxy.setValue` after validating key.
    *   **Use:** Retrieve via `contextProxy.getValue`.

2.  **Integrating New Service via Custom Flow (Only if PKCE-only):**
    *   **Verify PKCE:** Ensure service supports Authorization Code Grant + PKCE **without client secret**. If secret required -> **Backend Proxy Needed**.
    *   **URLs/Params:** Get auth URL, token URL, required params/scopes from provider docs.
    *   **Config:** Add necessary config (Client ID) to settings/schemas.
    *   **URL Helper (`oauth/urls.ts`):** Add `getNewServiceAuthUrl` function.
    *   **URI Handler (`handleUri.ts`):** Add `case "/newservice":` to route to a new provider callback.
    *   **Provider Callback (`ClineProvider.ts`):** Implement `handleNewServiceCallback(code)`:
        *   Retrieve stored PKCE verifier (requires storing it during initiation - modify `startAuthFlow` concept).
        *   Implement `exchangeNewServiceCodeForToken` using `axios` POST with code, client ID, redirect URI, PKCE verifier.
        *   Define secret key in schemas (`newServiceAccessToken`).
        *   Store token using `contextProxy.setValue` after key validation.
        *   Update API profile via `ProviderSettingsManager`.
        *   Refresh UI state.
    *   **Initiation:** Add UI button/link using `getNewServiceAuthUrl`. Add command/logic to handle initiation if needed (e.g., storing PKCE verifier/state).

**Best Practices:**

*   **Prioritize `vscode.authentication`:** Simplest and most secure.
*   **PKCE for Custom Flows:** Non-negotiable if client secret isn't used (and secret shouldn't be in extension).
*   **Proxy Secret-Based Exchanges:** Mandatory if client secret required.
*   **Secure Token Storage (`secrets`):** Use `ContextProxy`, define keys in schemas.
*   **Minimal Scopes:** Request only needed permissions.
*   **State Parameter:** Use and validate rigorously in custom flows.
*   **Clear User Feedback:** Guide user, report status clearly. Handle errors/timeouts.

**Potential Pitfalls:**

*   **Security Risks:** Client secrets in extension, insecure token storage, missing `state` validation, incorrect PKCE.
*   **Custom Flow Complexity:** Errors in state, token exchange, refresh logic.
*   **Redirect URI Issues:** Mismatches.
*   **Token Refresh Failures (Custom Flow).**
*   **Browser/Network Issues.**

## Conclusion

OAuth Helpers are essential for securely connecting Roo-Code to external services requiring user authentication. The built-in `vscode.authentication` API is the strongly preferred and recommended approach for supported providers like GitHub, handling the complexities securely and automatically. For unsupported services, Roo-Code utilizes a custom flow leveraging `vscode.window.registerUriHandler` and `vscode.env.openExternal`, as seen with integrations like Glama, OpenRouter, and Requesty. This requires careful manual implementation of state validation, PKCE (preferred, essential if no backend proxy), token exchange (potentially via a backend proxy if client secrets are unavoidable), and secure token storage via `context.secrets`. These helpers enable Roo-Code to securely leverage user-delegated permissions from a variety of external platforms.

Next, we examine the system Roo-Code uses for internationalization (i18n), allowing the UI to be presented in different languages: [Chapter 50: Localization System (i18n)](50_localization_system__i18n_.md).
---
# Chapter 50: Localization System (i18n)

Continuing from [Chapter 49: OAuth Helpers](49_oauth_helpers.md), which discussed authentication flows, this chapter focuses on making Roo-Code accessible to a global audience by implementing internationalization (i18n) and localization (l10n): the **Localization System (i18n)**. This includes specific guidelines found in `.roo/rules-translate/`.

## Motivation: Supporting Multiple Languages

To reach a wider user base and provide a more inclusive experience, it's essential for applications like Roo-Code to display text in the user's preferred language. Hardcoding strings (button labels, descriptions, menu items, error messages, tooltips) directly into the codebase makes translation difficult, error-prone, and hard to maintain.

A dedicated Localization System allows developers to:

1.  **Externalize Strings:** Separate user-visible text from the source code into resource files (e.g., JSON files).
2.  **Translate Resources:** Create language-specific versions of these resource files (e.g., `en.json`, `fr.json`, `ja.json`, `de.json`, `zh-cn.json`).
3.  **Load Dynamically:** Detect the user's preferred language (based on VS Code's locale settings) and load the appropriate translations at runtime.
4.  **Provide Fallbacks:** Define a default language (usually English) to use if a translation is missing for the user's locale.
5.  **Format Translations:** Handle plurals, interpolation (inserting dynamic values into strings), and potentially date/number formatting according to locale conventions.
6.  **Provide Guidelines:** Establish clear rules for translators (`.roo/rules-translate/`) covering tone, terminology, formatting, and workflow to ensure consistency and quality across languages.

Roo-Code implements such a system using the popular `i18next` library and its companion `react-i18next` for integration with the React-based WebView UI, along with leveraging VS Code's built-in localization mechanisms for extension-level strings (like command titles defined in `package.json`).

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

1.  **Initialization:** `i18next` is initialized (`i18n/config.ts`) and provided via `TranslationProvider`. The user's VS Code language (e.g., "de") is detected and passed from the extension host via initial state or messages, stored in `ExtensionStateContext`. `i18next` loads the German translation files (`public/locales/de/*.json`).
2.  **Component Render:** The `SettingsView` component renders.
3.  **Hook Usage:** It calls `useAppTranslation()` to get the `t` function.
4.  **Translation Lookup:** It calls `t("settings:saveButton")`. `i18next` looks up the key `"saveButton"` within the `"settings"` namespace in the currently loaded language (`de`).
5.  **Resource File (`public/locales/de/settings.json`):** Based on guidelines in `.roo/rules-translate/instructions-de.md` (mandating "du" form).
    ```json
    {
      "saveButton": "Speichern" // Informal imperative, adheres to rule
    }
    ```
6.  **Result:** The `t` function returns `"Speichern"`.
7.  **Render:** The button renders with the German text "Speichern".

## Key Concepts

1.  **`i18next` & `react-i18next`:** Core libraries for managing translations and React integration in the WebView UI.
2.  **Translation Files (`public/locales/<lang>/<namespace>.json`):** JSON resource files organized by language (ISO 639-1 code like `en`, `de`, `ja`, `zh-CN`) and namespace (`common`, `settings`, `mcp`, `tools`, `welcome`, `history`, `prompts`). Must be copied to `dist/locales` by the build system ([Chapter 56: Build System](56_build_system.md)).
3.  **Namespaces:** Logical grouping of strings within files (e.g., `settings.json`). Allows loading subsets if needed. `common` is the default namespace.
4.  **WebView i18n Initialization (`webview-ui/src/i18n/config.ts`):** Configures `i18next` instance:
    *   Uses `i18next-http-backend` to load files from `/locales/{{lng}}/{{ns}}.json` via HTTP.
    *   Relies on `language` setting from host state ([Chapter 12: ExtensionStateContext](12_extensionstatecontext.md)) reflecting `vscode.env.language`. Uses `LanguageDetector` as fallback.
    *   Sets `fallbackLng: 'en'`.
    *   Defines `ns` array and `defaultNS`.
5.  **React Context Provider (`TranslationProvider`):** Wraps the app, provides `i18next` instance, and includes `useEffect` to sync `i18nInstance.language` with the `language` from global state.
6.  **`useAppTranslation` Hook:** Custom hook wrapping `react-i18next`'s `useTranslation`. Provides the `t` function.
7.  **`t` Function & `Trans` Component:** Used for string lookup (`t('ns:key')`), interpolation (`{{var}}`), pluralization (`{ count: n }`), and rendering complex JSX translations.
8.  **Extension Host Localization (NLS):** Uses VS Code's native mechanism for static contributions in `package.json`.
    *   `package.json`: Uses `%key%` syntax.
    *   `package.nls.json`: English defaults.
    *   `package.nls.<lang>.json`: Translations. Copied to `dist/` by build system. VS Code handles loading.
    *   Dynamic strings use `vscode.l10n` (requires build setup) or alternative (`tHost`).
9.  **Translation Guidelines (`.roo/rules-translate/`):**
    *   **General (`001-general-rules.md`):** Defines overall tone (informal), style, technical term handling, placeholder syntax, `Trans` usage, QA process (validation script).
    *   **Language-Specific (`instructions-<lang>.md`):** Provides specific rules (e.g., "du" form in German `instructions-de.md`), terminology glossaries (e.g., `instructions-zh-cn.md`), formatting rules. Includes checklists.
10. **Validation Script (`scripts/find-missing-translations.js` - Conceptual):** A helper script ([Chapter 55: Script Utilities](55_script_utilities.md)) to compare keys across language files and report missing translations, aiding the QA process defined in the guidelines.

## Using the Localization System

### WebView UI

1.  **Setup:** Ensure `<TranslationProvider>` wraps app. Ensure `language` from host updates `i18nInstance.language` via `useEffect` in provider.
2.  **Usage:** Import `useAppTranslation`. Use `t('ns:key')` or `<Trans i18nKey="...">`.
3.  **Adding Strings:** Add key/value to `public/locales/en/<ns>.json`. Add key/translation to *all other* `public/locales/<lang>/<ns>.json` files, following guidelines in `.roo/rules-translate/`. Run validation script (`node scripts/find-missing-translations.js`).

### Extension Host

1.  **Static:** Use `%key%` in `package.json`. Define in `package.nls.json` (en) and `package.nls.<lang>.json`. Ensure build copies NLS files.
2.  **Dynamic:** Use `vscode.l10n.t()` (needs build config) or `tHost`.

### Translators

1.  **Read Guidelines:** Thoroughly read **all** files in `.roo/rules-translate/`, especially general rules and the target language's specific instructions.
2.  **Edit Files:** Use an editor for JSON files in `public/locales/<lang>/` and potentially `package.nls.<lang>.json`.
3.  **Preserve Syntax:** Keep `{{placeholders}}` and `<1>...</1>` markers exactly as in the English source.
4.  **Terminology:** Use the provided glossaries consistently.
5.  **Tone & Formatting:** Adhere to specified tone (e.g., informal "du") and formatting rules.
6.  **Validate:** Run `node scripts/find-missing-translations.js` to check for missing keys.

## Code Walkthrough

### i18n Configuration (`webview-ui/src/i18n/config.ts`)

*(See code in Chapter 50 - Key Concepts)*
*   Configures `i18next` with `HttpBackend` (loads from `/locales/`), detector fallback, `initReactI18next`. Sets `loadPath`, `fallbackLng`, namespaces.

### Context Provider & Hook (`webview-ui/src/i18n/TranslationContext.tsx`)

*(See code in Chapter 50 - Key Concepts)*
*   `TranslationProvider`: Syncs `i18nInstance.language` with `language` from `useExtensionState`.
*   `useAppTranslation`: Provides `t` function via `useTranslation`.

### Translation Guidelines (`.roo/rules-translate/`)

*(See descriptive examples in Key Concepts section)*
*   Markdown files providing rules, glossaries, and checklists for human translators.

### Validation Script (`scripts/find-missing-translations.js` - Conceptual)

*(See code in Chapter 50 - Key Concepts)*
*   Node.js script using `fs` and `glob` to find language files.
*   Loads default language keys (`en`).
*   Iterates through other languages, comparing keys against the default set.
*   Logs warnings for missing or empty keys.

### Extension Host: `package.json` / NLS Files

*(See examples in Key Concepts section)*
*   `package.json` uses `%key%`.
*   `package.nls.json` defines English strings.
*   `package.nls.<lang>.json` provides translations. Build must copy these to `dist/`.

## Internal Implementation

*   **WebView (`i18next`):** Language detection/sync -> HTTP Backend fetches JSON -> Resources stored -> `t()` lookup -> Fallback -> String returned. Language change triggers fetch + re-render.
*   **Extension Host (NLS):** VS Code detects language -> Loads matching `package.nls.<lang>.json` -> Substitutes `%key%` placeholders in `package.json`. Dynamic strings use `vscode.l10n` or alternative.

**Sequence Diagram (WebView Language Change & Translation):**

*(See diagram in Chapter 50 - Internal Implementation)*

## Modification Guidance

Modifications primarily involve adding strings, namespaces, or languages, or updating guidelines.

1.  **Adding a New String:** Follow steps in "Using the Localization System". Remember both WebView (`public/locales`) and potentially Host (`package.nls*`). Update all language files. Run validation script.
2.  **Adding a New Language:** Follow steps in "Using the Localization System". Requires creating new files/folders and translating all existing keys. Add language-specific guidelines to `.roo/rules-translate/`.
3.  **Adding a New Namespace:** Follow steps in "Using the Localization System". Requires updating `config.ts` (`ns` array) and creating new JSON files in all language folders.
4.  **Updating Translation Guidelines:** Edit Markdown files in `.roo/rules-translate/`. Communicate changes to translators. Ensure validation script aligns with any workflow changes.

**Best Practices:**

*   **Keys, Not Values:** Use semantic keys.
*   **Namespacing:** Organize logically.
*   **Fallback:** Maintain complete English translations.
*   **Sync Files:** Keep keys consistent. Use validation scripts (`find-missing-translations.js`).
*   **`Trans` Component:** Use for JSX embedding.
*   **Context/Formatting:** Use i18n features for interpolation/plurals.
*   **Separate Concerns:** Use appropriate system for WebView (`i18next`) vs. Host (NLS/%key%, `vscode.l10n`).
*   **Guidelines:** Maintain clear, specific, and up-to-date guidelines (`.roo/rules-translate/`) including glossaries and checklists.

**Potential Pitfalls:**

*   **Missing Keys/Translations:** Shows keys or fallback. Validation script helps.
*   **JSON Errors:** Malformed translation files break loading.
*   **Incorrect Paths (`loadPath`) / Missing Assets:** Backend fails to load files. Check build script ([Chapter 56: Build System](56_build_system.md)) ensures `public/locales` and `package.nls*` are copied to `dist/`.
*   **Language Sync Issues:** Discrepancy between VS Code language and `i18next` language. `useEffect` in `TranslationProvider` aims to fix this.
*   **Namespace Errors.**
*   **Host vs. WebView Mix-up.**
*   **`Trans` Component Errors:** Mismatched placeholders.
*   **Outdated/Ignored Guidelines:** Leads to inconsistent or poor-quality translations.

## Conclusion

The Localization System enables Roo-Code to provide a user interface understandable to a global audience. By leveraging `i18next` and `react-i18next` in the WebView for dynamic translation loading based on VS Code's locale, and using VS Code's native NLS mechanism for static extension contributions, Roo-Code supports multiple languages effectively. Crucially, the detailed Translation Guidelines stored in `.roo/rules-translate/`, along with validation scripts, provide essential instructions and quality control for translators, ensuring consistency in tone, terminology, and formatting across all supported languages. This comprehensive approach is vital for delivering a high-quality, inclusive user experience worldwide.

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

*   **Spawning:** `child_process.spawn(rgPath, args, { stdio: ['ignore', 'pipe', 'pipe'] })`. Input is via `args`.
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
        workerResponseSchema.parse(progressMsg); // Validate before sending (optional but good)
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
*   **SSE:** `SSEClientTransport` uses `ReconnectingEventSource`. Requests might use `axios` POSTs.
*   **`child_process.spawn` (General):** Creates OS process, Node streams interface with OS pipes.
*   **`child_process.fork`:** Specialized `spawn` for Node.js scripts. Automatically sets up IPC channel (often pipes). `process.send`/`.on('message')` use this channel with built-in V8 serialization (handles more types than JSON, but only Node-to-Node).
*   **VS Code `postMessage`:** Internal VS Code mechanism for host <-> WebView communication.

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
*   **Robust Framing:** Essential for stream-based IPC. Newlines are simple; length prefixing is more robust. `fork` handles it automatically.
*   **Serialization:** Use JSON unless binary performance is critical. Validate with Zod.
*   **Error Handling:** Handle connection, process exit, `stderr`, parsing, timeouts. Define error message formats.
*   **Resource Management:** Terminate processes, close streams/sockets/handles, remove listeners.
*   **Asynchronicity:** Use `async/await`.

**Potential Pitfalls:**

*   **Framing/Parsing Errors:** Common in stream-based IPC.
*   **Process Management:** Zombies, spawn failures, unhandled exits.
*   **Deadlocks:** Waiting indefinitely for responses (need timeouts).
*   **Buffering Issues:** Data delays/fragmentation in streams. Stdio buffers can fill.
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

6.  **Sanitization Logic (`sanitizeProperties`):** Needs careful implementation to remove known sensitive keys (`/key|secret|token|password|credential/i`), long strings (potential prompts/code/paths), while allowing safe context values.

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
    const currentSetting = contextProxy.getValue("telemetrySetting" as any) ?? 'unset'; // Default to 'unset'
    if (currentSetting === 'unset') {
        const selection = await vscode.window.showInformationMessage(
            "Help improve Roo Code by sending anonymous usage data and error reports? No source code or personal data is ever sent. You can change this in Settings.",
            { modal: true }, // Block until choice
            "Enable Telemetry", // User explicitly enables
            "Disable Telemetry" // User explicitly disables
        );
        // If user closed dialog without selecting, treat as disabled for now
        const settingValue = selection === "Enable Telemetry" ? "enabled" : "disabled";
        await contextProxy.setValue("telemetrySetting" as any, settingValue);
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
    const provider = new ClineProvider(context, outputChannel);
    // Allow telemetry to get context like current mode from provider
    telemetryService.setProvider(provider);

    // Prompt for consent if needed (after telemetry service is initialized)
    // Run non-blockingly after activation completes
    ensureTelemetryConsent(contextProxy).catch(e => console.error("Consent prompt failed:", e));

    // Listen for setting changes to update consent dynamically
    context.subscriptions.push(vscode.workspace.onDidChangeConfiguration(async e => {
        // Use the actual setting key from package.json contribution
        const fullSettingKey = 'roo-code.telemetrySetting';
        if (e.affectsConfiguration(fullSettingKey)) {
            const oldEnabled = telemetryService['enabled']; // Access internal state
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
*Explanation:* Creates the singleton. Calls `init` after `ContextProxy`. Links the provider using `setProvider`. Calls `ensureTelemetryConsent` asynchronously to prompt the user if the setting is `unset`. Listens for configuration changes to update consent dynamically via `updateConsent` and captures the consent change event. Calls `shutdown` in `deactivate`.

**2. Capturing Events:**

```typescript
// --- File: src/activate/registerCodeActions.ts ---
import { telemetryService } from "../services/telemetry/TelemetryService";
// ... inside command handler ...
    vscode.commands.registerCommand(command, async (...args: any[]) => {
        // Get current mode if possible (needs access to provider instance)
        const currentMode = ClineProvider.getSidebarProviderInstance()?.contextProxy?.getValue('mode');
        telemetryService.captureEvent("codeaction.invoked", { command, mode: currentMode }); // Include mode
        // ... rest of handler ...
    });

// --- File: src/core/tools/executeCommandTool.ts ---
import { telemetryService } from "../../services/telemetry/TelemetryService";
// ... after successful execution ...
    telemetryService.captureEvent("tool.executed", {
        tool_name: "execute_command",
        success: exitCode === 0,
        duration_ms: Date.now() - startTime,
    });
```
*Explanation:* Call `telemetryService.captureEvent` with a clear name (e.g., `feature.subfeature.action`) and relevant, non-sensitive properties (like `command` ID, `mode` slug, `tool_name`, `success` flag, `duration`).

**3. Capturing Errors:**

```typescript
// --- File: src/api/providers/openai.ts (Conceptual Error Handling) ---
    } catch (error: any) {
        const errorType = classifyApiError(error); // Generic error type
        const statusCode = error?.status || error?.response?.status;
        telemetryService.captureError("api_error", {
            provider: "openai",
            modelId: this.getModel().id, // Already sanitized? Assume yes.
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
*Explanation:* Use `captureError` for handled errors, providing categorization (provider, error type, status code). Use `captureSchemaValidationError` for Zod errors; the service extracts safe information (paths, counts) automatically. Use `captureException` in global error handlers.

## Code Walkthrough

### TelemetryService Class (`src/services/telemetry/TelemetryService.ts`)

*(See full code in Key Concepts section)*

**Key Implementation Details:**

*   **Constructor:** Sets up PostHog key/host from `process.env` (injected by build system). Generates `sessionId`. Sets `distinctId` to `vscode.env.machineId`, handling the dev default value.
*   **`init`:** Populates `commonProperties` (OS, versions). Calls `updateConsent`. Logs status.
*   **`setProvider`/`getProviderContext`:** Uses `WeakRef` to link to `ClineProvider` and fetch non-sensitive context like `current_mode`.
*   **`updateConsent`:** Reads setting via `ContextProxy`. Shuts down/initializes `PostHog` client based on `enabled` state and key presence. Includes error handling for client init.
*   **`_capture`:** Central internal method. Checks `enabled`. Merges common, provider, and event properties. **Calls `sanitizeProperties`**. Calls `posthogClient.capture`. Includes error handling.
*   **Public Methods:** Add `roo_` prefixes. Prepare specific properties (limited stack for exceptions, paths/counts for schema errors). Call `_capture`.
*   **`shutdown`:** Calls `posthogClient.shutdown()`.

### Sanitization (`src/services/telemetry/sanitize.ts` - Conceptual)

```typescript
// --- File: src/services/telemetry/sanitize.ts --- (Conceptual)
import { logger } from "../../utils/logging";

const MAX_STRING_LENGTH = 250; // Max length for property values to avoid PII
const SENSITIVE_KEY_PATTERNS = [/key/i, /secret/i, /token/i, /password/i, /credential/i]; // Regex for sensitive keys
const PATH_LIKE_REGEX = /(\/|\\)/; // Simple check for path separators

/**
 * Sanitizes properties object to remove or mask potential PII.
 * - Removes keys matching sensitive patterns.
 * - Truncates long strings.
 * - Removes strings that look like file paths.
 * - TODO: Add more robust checks (e.g., email, specific formats) if needed.
 * - TODO: Consider recursive sanitization for nested objects.
 */
export function sanitizeProperties(properties: Record<string, any>): Record<string, any> {
    if (!properties) return {};

    const sanitized: Record<string, any> = {};
    for (const key in properties) {
        if (!Object.prototype.hasOwnProperty.call(properties, key)) continue;

        const value = properties[key];

        // 1. Check key name for sensitive patterns
        if (SENSITIVE_KEY_PATTERNS.some(pattern => pattern.test(key))) {
            logger.debug(`Sanitizing sensitive key: ${key}`, { ctx: "Telemetry.Sanitize" });
            continue; // Skip sensitive keys entirely
        }

        // 2. Handle different value types
        if (typeof value === 'string') {
            // 2a. Check for path-like strings (simple heuristic)
            if (value.length > 10 && PATH_LIKE_REGEX.test(value)) {
                 logger.debug(`Sanitizing path-like value for key: ${key}`, { ctx: "Telemetry.Sanitize" });
                 sanitized[key] = '<path_omitted>'; // Replace potential paths
                 continue;
            }
            // 2b. Check for excessive length (potential code/prompt/PII)
            if (value.length > MAX_STRING_LENGTH) {
                 logger.debug(`Sanitizing long string value for key: ${key}`, { ctx: "Telemetry.Sanitize" });
                 sanitized[key] = `${value.substring(0, MAX_STRING_LENGTH)}...<truncated>`; // Truncate
                 continue;
            }
            // If string is okay, include it
            sanitized[key] = value;

        } else if (typeof value === 'number' || typeof value === 'boolean' || value === null) {
            // Allow numbers, booleans, null
            sanitized[key] = value;
        } else if (typeof value === 'object' && value !== null) {
            // Handle objects - currently skips complex objects/arrays
            // TODO: Implement recursive sanitization if needed, or flatten specific known objects.
            // For now, let's stringify simple objects/arrays if they aren't too large
            try {
                const jsonString = JSON.stringify(value);
                if (jsonString.length <= MAX_STRING_LENGTH) {
                    sanitized[key] = value; // Keep small objects/arrays as is
                } else {
                    logger.debug(`Sanitizing large object/array value for key: ${key}`, { ctx: "Telemetry.Sanitize" });
                    sanitized[key] = '<complex_object_omitted>';
                }
            } catch {
                 logger.debug(`Sanitizing unserializable object value for key: ${key}`, { ctx: "Telemetry.Sanitize" });
                 sanitized[key] = '<unserializable_object_omitted>';
            }
        } else {
            // Skip other types (undefined, function, symbol)
             logger.debug(`Skipping unsupported type value for key: ${key} (${typeof value})`, { ctx: "Telemetry.Sanitize" });
        }
    }
    return sanitized;
}
```
*Explanation:* Provides a more concrete sanitization strategy. It checks key names against patterns, omits keys matching sensitive patterns. For string values, it checks for path-like characters (replacing with placeholder) and length (truncating). It allows numbers, booleans, and nulls. It attempts to keep small objects/arrays but replaces large or complex ones. Includes TODOs for recursion.

## Internal Implementation

1.  **Event Trigger:** Code calls `telemetryService.capture...()`.
2.  **Consent Check:** Service checks `this.enabled`. If false, returns.
3.  **Context/Sanitize:** `_capture` merges context, calls `sanitizeProperties` which filters/masks data.
4.  **Client Call:** `_capture` calls `posthogClient.capture()`.
5.  **PostHog Client Buffering & Async Send:** `posthog-node` library batches events and sends them asynchronously via HTTPS.
6.  **Shutdown:** `telemetryService.shutdown()` calls `posthogClient.shutdown()` to flush buffer.

**Sequence Diagram (Capture Event):**

*(See diagram in Chapter 52 - Internal Implementation)*

## Modification Guidance

Modifications involve adding events, changing context, or updating sanitization.

1.  **Adding a New Tracked Event:**
    *   **Identify:** Find code location.
    *   **Call:** Add `telemetryService.captureEvent("my_event", { property: safeValue })`.
    *   **Data:** Include only relevant, **non-sensitive** properties.
    *   **Sanitize:** Verify `sanitizeProperties` handles new properties or update it.

2.  **Adding More Context:**
    *   **Common Properties (`init`):** Add static, safe data (e.g., `vscode.env.appHost`).
    *   **Provider Context (`getProviderContext`):** Add logic to get more non-sensitive state from `ClineProvider` (e.g., count of configured profiles, NOT the names/keys). Ensure safety.
    *   **Event Properties:** Pass more relevant, safe data directly via `captureEvent`.

3.  **Refining Sanitization (`sanitizeProperties`):**
    *   **Edit `sanitize.ts`:** Add more sensitive key patterns. Improve path/URL detection and anonymization (e.g., hashing). Adjust length thresholds. Implement recursive sanitization if complex objects need partial reporting.
    *   **CRITICAL:** Test thoroughly to prevent PII leaks.

4.  **Changing Telemetry Backend:**
    *   Replace `posthog-node` library and client calls in `TelemetryService` (`init`, `_capture`, `shutdown`) with the new backend's SDK. Ensure consent and sanitization remain central.

**Best Practices:**

*   **Privacy First & Opt-In:** Non-negotiable. Rigorous sanitization, clear consent prompt ([`ensureTelemetryConsent`]). Default to disabled/unset.
*   **Centralized Service:** Use the singleton.
*   **Sanitize Rigorously:** Continuously review `sanitizeProperties`.
*   **Meaningful Events:** Track actionable insights.
*   **Safe Context:** Only add non-sensitive context.
*   **Graceful Failure:** Service failures shouldn't crash the extension.
*   **Shutdown:** Implement `shutdown` for data flushing.
*   **Transparency:** Document data collection practices (README/Privacy Policy).

**Potential Pitfalls:**

*   **PII Leakage:** Insufficient sanitization is the primary risk.
*   **Consent Bypass:** Bugs sending data when disabled.
*   **Performance:** Excessive event capture.
*   **Network/Backend Issues:** Data loss if backend unavailable.
*   **Sanitization Complexity/Errors:** Bugs in `sanitizeProperties`. Over-sanitization losing useful anonymous data.

## Conclusion

The `TelemetryService` provides a crucial, privacy-conscious mechanism for collecting anonymous usage data and error information from Roo-Code installations where users have explicitly opted in. By centralizing event capture, rigorously sanitizing data to remove PII via `sanitizeProperties`, enriching events with safe context, and integrating with a backend like PostHog, the service provides valuable insights for the development team to improve the extension. Respecting user consent (via the `telemetrySetting` and initial prompt) and prioritizing privacy through careful implementation are paramount.

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

The **Evals System** provides the framework and tooling to define, run, and potentially analyze these evaluation tasks, enabling data-driven development and quality assurance for Roo-Code's core AI functionality. It involves defining test suites (YAML), a runner script (`runEval.ts`) to execute them using a programmatic entry point to the core logic, and a dedicated web interface (`evals-web/`) for reviewing results.

**Central Use Case:** A developer modifies the main system prompt ([Chapter 7: SystemPrompt](07_systemprompt.md)) and wants to ensure it doesn't negatively impact code generation quality for Python.

1.  **Define Eval Suite:** An `evals/suites/python_codegen.yaml` file defines several test cases, each validated by `evalTaskSchema` ([Chapter 40: Schemas (Zod)](40_schemas__zod_.md)):
    *   `id`: `py_fib_recursive`
    *   `prompt`: "Write a Python function to calculate Fibonacci numbers recursively."
    *   `criteria`: "Function should be recursive, handle base cases (0, 1), and be syntactically correct Python."
    *   `config_overrides`: `{ "mode": "code" }`
2.  **Run Evals:** Developer runs `pnpm start --suite python_codegen --model gpt-4-turbo` from `evals/`.
3.  **Eval Runner (`runEval.ts` / `evaluateTask.ts`):**
    *   Parses args (`yargs`). Finds `python_codegen.yaml` (`findYamlFiles`). Loads/validates tasks using `js-yaml` and `evalTaskSchema`.
    *   For task `py_fib_recursive` and model `gpt-4-turbo`:
        *   Calls `evaluateTask`.
        *   `evaluateTask` merges config, prepares context (none needed here).
        *   Calls programmatic entry point `runRooCodeTaskForEval(prompt, context, config)`.
        *   `runRooCodeTaskForEval` (conceptual) instantiates `Cline` with mocks (for VS Code API) and specified config/API key (from env vars). Runs the task loop. Captures all emitted `ClineMessage`s. Returns transcript and metrics.
        *   `evaluateTask` receives transcript/metrics, calculates time, assembles `EvalOutput`.
        *   Runner saves `EvalOutput` to `evals/output/<ts>/python_codegen/gpt-4-turbo/py_fib_recursive.json`.
4.  **Review Results:** Developer uses the `evals-web` UI ([Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md)) to view the result. They examine the `transcript` (containing the generated Python code) and compare it against the `criteria`.
5.  **Analysis:** Assess if the code is correct and meets expectations. Compare with previous runs if available.

## Key Concepts

1.  **Evaluation Suites (`evals/suites/*.yaml`):** YAML files defining test cases (tasks). Each task specifies `id`, `prompt`, optional `context` (files, etc.), `criteria`, and `config_overrides`.
2.  **Test Case Structure (`EvalTask`):** Defined by Zod schema `evalTaskSchema` (`evals/schemas/evalTask.ts`). Ensures consistency of suite files.
3.  **Eval Runner (`evals/src/runEval.ts`, `evaluateTask.ts`):** Node.js scripts orchestrating the evaluation.
    *   **`runEval.ts`:** Parses CLI args (`yargs`), finds/loads/validates suite files, iterates through suites/models/tasks, calls `evaluateTask`, saves results/errors to structured JSON files in `evals/output/`. Uses script utilities ([Chapter 55: Script Utilities](55_script_utilities.md)).
    *   **`evaluateTask.ts`:** Executes a single task. Prepares context, merges config (including model and API keys from env vars), invokes core Roo-Code logic programmatically, captures the full `ClineMessage` transcript, calculates metrics (time, cost using [Chapter 29: Cost Calculation Utilities](29_cost_calculation_utilities.md)), cleans up context, and returns structured `EvalOutput`.
4.  **Programmatic Invocation (`runRooCodeTaskForEval` - Conceptual):** The critical link. An entry point allowing the core `Cline` logic ([Chapter 4: Cline](04_cline.md)) to run outside VS Code. Requires careful dependency injection or mocking for VS Code APIs (UI prompts, workspace, terminal, secrets). Must capture the output stream/events (`ClineMessage[]`).
5.  **Output Format (`EvalOutput`):** Defined by Zod schema `evalOutputSchema` (`evals/schemas/`). Stores inputs (`taskId`, `prompt`, `context`, `config`) and outputs (`transcript`, `metrics`, `error`) in JSON for analysis. Uses `clineMessageSchema`.
6.  **Evaluation/Analysis:** Primarily manual review against `criteria` using output JSON or the `evals-web` UI. Potential for automated metrics (code execution checks, linting, LLM-based grading).
7.  **Web UI (`evals-web/`):** Separate Next.js/React application for browsing, displaying, comparing eval results from the `evals/output/` directory. Uses standard web shadcn/ui primitives ([Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md)).

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
*   Uses `yargs`, `findYamlFiles`, `js-yaml`, `evalTaskSchema`.
*   Main loop iterates suites -> models -> tasks.
*   Calls `evaluateTask`. Saves output using `evalOutputSchema` structure.

### Task Evaluation (`evals/src/evaluateTask.ts` - Conceptual)

*(See code in Key Concepts section)*
*   Receives `EvalTask`, `modelIdToUse`.
*   Merges config (needs API keys from `process.env`).
*   Prepares/Cleans context (temp files).
*   **Invokes core logic (complex part needing mocks/entry point).** Captures `transcript`.
*   Calculates `metrics`. Assembles `EvalOutput`. Returns result.

### Schemas (`evals/schemas/`)

*   **`evalTask.ts` (`evalTaskSchema`):** Zod schema for YAML task definitions. Includes nested `context` schema.
*   **`evalOutput.ts` (`evalOutputSchema`):** Zod schema for JSON output files. Includes `transcript: z.array(clineMessageSchema)`, `metrics`, etc. Reuses `clineMessageSchema` from `src/schemas/messages.ts`.

### Web UI (`evals-web/`)

*   Separate Next.js/React project.
*   Uses file system routing or API routes to read JSON data from `evals/output`.
*   Uses standard shadcn/ui primitives (`Table`, `Card`, `Badge`, etc.) for display ([Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md)).
*   Includes components for rendering transcripts (likely using `MarkdownBlock` concepts from [Chapter 46: Markdown Rendering](46_markdown_rendering.md), adapted for standard web).

## Internal Implementation

1.  **Runner:** Node.js script parses args, finds/loads/validates YAMLs.
2.  **Task Loop:** For each task/model combo, calls `evaluateTask`.
3.  **`evaluateTask`:**
    *   Sets up temp context (if needed).
    *   Merges config, gets API key from `process.env`.
    *   Calls programmatic entry point `runRooCodeTaskForEval`.
    *   `runRooCodeTaskForEval` instantiates core components (`ApiHandler`, mocked `ContextProxy`, `Cline`) disabling/mocking VS Code dependencies (UI prompts always approve/follow default, FS uses temp dir, terminal might be disabled or mocked).
    *   `Cline` executes the task loop. `ClineMessage`s are captured via listener/stream.
    *   Entry point returns transcript/metrics.
    *   `evaluateTask` cleans up context, calculates final metrics, returns `EvalOutput`.
4.  **Runner Saves Output:** Runner writes `EvalOutput` to JSON file.
5.  **Web UI:** Reads JSON files, renders data using React and shadcn/ui components.

**Sequence Diagram (Runner executing one task):**

*(See diagram in Chapter 53 - Internal Implementation)*

## Modification Guidance

Modifications involve adding suite features, runner options, evaluation metrics, or enhancing the analysis UI.

1.  **Adding New Context Type (e.g., Mock Terminal State):**
    *   **Schema (`evalTaskSchema`):** Add `mock_terminal_output?: { ... }` to `context`.
    *   **YAML:** Add `context: { mock_terminal_output: { ... } }` to tasks.
    *   **`evaluateTask.ts`:** Modify `prepareEvalContext` / core invocation to pass this mock state to the mocked `TerminalRegistry` or `Cline`.
    *   **Core Logic:** Ensure mocked terminal logic uses this state.

2.  **Adding Automated Metric (e.g., Code Linting):**
    *   **`evaluateTask.ts`:** After capturing transcript, extract code, write to temp file, run linter via `child_process`, record pass/fail/errors in `metrics`.
    *   **Schema (`evalOutputSchema`):** Add new field to `evalMetricsSchema`.
    *   **Analysis/UI:** Display/use the new metric. Consider security implications.

3.  **Supporting Parallel Execution:**
    *   **`runEval.ts`:** Use `p-limit` or `Promise.all` with concurrency control around the `evaluateTask` calls in the loops.
    *   **`evaluateTask.ts`:** Ensure context setup/cleanup uses unique names to avoid conflicts. Be mindful of API rate limits.

4.  **Enhancing Web UI (`evals-web/`):**
    *   Modify Next.js pages/components. Use shadcn/ui primitives ([Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md)) to add filtering, sorting, diff views, charts, annotation features. Read data from JSON outputs.

**Best Practices:**

*   **Clear Criteria:** Define specific, verifiable `criteria`.
*   **Realistic Context:** Use representative prompts/context.
*   **Isolate Core Logic:** Create a clean programmatic entry point (`runRooCodeTaskForEval`) minimizing VS Code dependencies.
*   **Robust Mocking:** Mock VS Code dependencies accurately or disable features gracefully.
*   **Structured Output:** Use schemas (`evalOutputSchema`) for JSON outputs. Store inputs+outputs.
*   **Version Control Suites:** Store YAMLs in Git.
*   **Reproducibility:** Record config. Fix model versions if possible.
*   **Balance Automated/Manual:** Use automated metrics + manual review.
*   **Security:** Be cautious executing AI-generated code during evals.

**Potential Pitfalls:**

*   **Environment Differences:** Evals outside VS Code behaving differently due to inaccurate mocking.
*   **Mocking Complexity:** Mocking `Cline`'s environment is hard.
*   **Context Setup/Cleanup Errors.**
*   **API Keys/Costs:** Securely manage keys (use env vars). Be mindful of costs.
*   **Flaky LLMs/Tests:** Non-determinism needs careful criteria or multiple runs.
*   **Eval Suite Maintenance:** Keeping suites relevant and updated.

## Conclusion

The Evals System provides an essential framework for systematically measuring and improving the performance of Roo-Code's AI capabilities. By defining structured test suites (YAML), using a runner script (`runEval.ts`, `evaluateTask.ts`) to execute tasks programmatically against different models/configurations, and capturing detailed output transcripts and metrics in a structured format (JSON), developers can objectively assess correctness, prevent regressions, and guide future improvements. While setting up the programmatic invocation of the core logic requires careful handling of dependencies and mocking, the benefits of automated, data-driven quality assessment are significant for developing a reliable AI assistant. The optional `evals-web` UI further enhances the analysis and comparison of results.

Next, we look at the UI primitives specifically used within the Evals web interface: [Chapter 54: Shadcn/UI Primitives (Evals Web)](54_shadcn_ui_primitives__evals_web_.md).
---
# Chapter 54: Shadcn/UI Primitives (Evals Web)

Continuing from [Chapter 53: Evals System](53_evals_system.md), which detailed the framework for running automated evaluations of Roo-Code's AI, this chapter focuses on the user interface components used within the **Evals Web** application – the separate web interface designed for reviewing and analyzing evaluation results. Specifically, we look at the **Shadcn/UI Primitives** used in this context.

## Motivation: Building a Dedicated Web UI for Eval Analysis

While the main Roo-Code extension UI ([Chapter 1: WebView UI](01_webview_ui.md)) runs within VS Code and needs to match its theme precisely, the Evals Web UI is a standalone web application (likely built with Next.js or a similar framework like Vite/React) running in a standard browser. Its purpose is different: presenting potentially large amounts of structured evaluation data (results from `evals/output/`), enabling comparisons between different runs (e.g., different models, different code versions), filtering, sorting, and potentially annotating results.

This separate context allows for different UI choices. While consistency with VS Code is less critical, building a clean, modern, and functional web interface still requires a good set of UI components. `shadcn/ui` is again chosen for this purpose, but used in a more standard web configuration compared to its adaptation for the VS Code WebView ([Chapter 33: Shadcn/UI Primitives (WebView)](33_shadcn_ui_primitives__webview_.md)).

The reasons for using `shadcn/ui` here are similar to its use in the WebView, but with slightly different emphasis:

1.  **Rich Component Set:** Provides essential components for building data-rich web applications: `Table`, `Dialog`, `Select`, `Tabs`, `Card`, `Button`, `Badge`, `Tooltip`, etc., suitable for displaying complex evaluation results.
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
5.  **Component Set:** Includes primitives well-suited for data dashboards and web applications: `Table`, `Card`, `Badge`, `Tabs`, `Select`, `DropdownMenu`, `Dialog`, `Popover`, `Tooltip`, `Button`, `Input`, `Checkbox`, `Separator`, etc.
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
import { EvalOutput } from '../../schemas/evalOutput'; // Adjusted path

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
            const resultDetailId = `${result.config?.suiteName || 'suite'}/${taskId}/${modelId}`; // Example

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
                  <Link href={`/results/${resultDetailId}`} passHref legacyBehavior>
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

1.  **Imports:** Imports primitives from `@/components/ui` within `evals-web`. Imports `EvalOutput` type from the shared `evals/schemas` directory using a relative path (adjust as needed based on actual file structure or if path aliases are configured for `evals-web`).
2.  **Composition:** Uses standard `Table` components. Uses `Badge` for status and `Tooltip` (wrapped in `TooltipProvider`) for cost details.
3.  **Styling:** Uses Tailwind utilities (`text-right`, `font-medium`). `Badge` uses `variant="destructive"` for errors.
4.  **Data Mapping:** Maps `results` array (`EvalOutput[]`). Accesses `result.config.modelId`, `result.error`, `result.metrics`. Formats cost/tokens.
5.  **Interaction:** Uses Next.js `Link` wrapping a `Button` (with `asChild`) to navigate to a detail page (URL structure assumes pages exist under `/results/`).

## Code Walkthrough

The primitive components in `evals-web/components/ui/` are standard shadcn/ui implementations, visually distinct from the VS Code-themed versions used in the main extension's WebView.

### Example Primitive (`evals-web/components/ui/badge.tsx`)

*(Likely standard shadcn/ui badge, identical or similar to Chapter 54 - Code Walkthrough)*

```typescript
// --- File: evals-web/components/ui/badge.tsx ---
import * as React from "react"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils" // Path relative to evals-web project

// Standard shadcn/ui badge variants using Tailwind theme colors
const badgeVariants = cva(
  "inline-flex items-center rounded-full border px-2.5 py-0.5 text-xs font-semibold transition-colors focus:outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2",
  {
    variants: {
      variant: {
        // These classes (primary, secondary, destructive, foreground)
        // map to the web theme defined in evals-web/tailwind.config.js
        default: "border-transparent bg-primary text-primary-foreground hover:bg-primary/80",
        secondary: "border-transparent bg-secondary text-secondary-foreground hover:bg-secondary/80",
        destructive: "border-transparent bg-destructive text-destructive-foreground hover:bg-destructive/80",
        outline: "text-foreground",
      },
    },
    defaultVariants: { variant: "default" },
  }
)

export interface BadgeProps extends React.HTMLAttributes<HTMLDivElement>, VariantProps<typeof badgeVariants> {}

function Badge({ className, variant, ...props }: BadgeProps) {
  return (
    <div className={cn(badgeVariants({ variant }), className)} {...props} />
  )
}

export { Badge, badgeVariants }
```

**Explanation:**

*   **Structure:** Identical to the base shadcn/ui badge component.
*   **Styling:** Uses semantic Tailwind classes (`bg-primary`, etc.) which map to the standard web theme defined in `evals-web/tailwind.config.js` and `globals.css`, **not** to VS Code CSS variables.

### Tailwind Configuration (`evals-web/tailwind.config.js` - Conceptual)

*(See code in Chapter 54 - Key Concepts)*
*   Defines a standard web color palette (light/dark mode) using HSL values mapped to CSS variables (`--background`, `--primary`, `--border`, etc.) in a global CSS file (e.g., `globals.css`).
*   **Crucially distinct** from the WebView's Tailwind config; it does not reference `--vscode-*` variables.

## Internal Implementation

Standard React rendering flow using Tailwind CSS:
1.  React component uses `<Badge variant="destructive">`.
2.  CVA/`cn` generate classes: `... bg-destructive text-destructive-foreground ...`.
3.  React renders `<div class="...">`.
4.  Tailwind build generates CSS: `.bg-destructive { background-color: hsl(var(--destructive)); } ...`.
5.  Global CSS defines `--destructive` HSL value.
6.  Browser applies styles, rendering a standard red web badge.

## Modification Guidance

Modifications involve customizing the theme or adapting/adding primitives within the `evals-web` project, following standard web development practices with shadcn/ui and Tailwind.

1.  **Changing the Evals Web Theme:**
    *   **CSS Variables:** Edit HSL values in `globals.css` for light/dark modes.
    *   **Tailwind Config:** Adjust semantic mappings or add new colors in `evals-web/tailwind.config.js`. Rebuild CSS.

2.  **Adapting/Adding a New Shadcn/UI Primitive:**
    *   Use the shadcn/ui CLI (`npx shadcn-ui@latest add ...`) within the `evals-web` directory or manually copy component source code into `evals-web/components/ui/`.
    *   The standard component should work directly with the existing Tailwind theme. No mapping to VS Code variables is needed.
    *   Update `tailwind.config.js` or global CSS if the component introduces new theme variables or requires plugins.

**Best Practices:**

*   **Standard Shadcn/UI:** Leverage the standard components and structure.
*   **Theme via CSS Variables:** Define the palette in global CSS.
*   **Tailwind Configuration:** Ensure `tailwind.config.js` is set up correctly for `evals-web`.
*   **Keep Separate:** Maintain the distinction between Evals Web components (standard web theme) and WebView UI components (VS Code theme).

**Potential Pitfalls:**

*   **Incorrect Tailwind Setup:** Broken styles.
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
*   Finding missing translation keys (as mentioned in [Chapter 50: Localization System (i18n)](50_localization_system__i18n_.md)).

These scripts, often written in TypeScript or JavaScript and executed using Node.js (e.g., via `pnpm run script-name`), frequently require common helper functionalities:

*   Parsing command-line arguments.
*   Interacting with the file system robustly (reading directories, finding files by pattern, checking paths) ([Chapter 42: File System Utilities](42_file_system_utilities.md), [Chapter 43: Path Utilities](43_path_utilities.md) might be reused or adapted).
*   Executing shell commands or other scripts reliably and capturing output.
*   Handling logging consistently across different scripts with clear formatting (e.g., colors).
*   Managing asynchronous operations, potentially with concurrency limits.

Implementing these helpers repeatedly in each script is inefficient and leads to inconsistency. The **Script Utilities** (conceptually located in directories like `scripts/lib/`, `evals/src/lib/`, or shared `src/utils/` if suitable) provide a collection of reusable functions tailored for these Node.js scripting contexts.

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
        // (Code becomes more complex for robust recursive search without libraries)
    } catch (error) { /* handle errors */ }
    // ... use suiteFiles ...
}
```

With Utilities:
```typescript
// Conceptual script code using utils
import yargs from "yargs";
// Assuming utility exists in evals/src/lib for this specific need
import { findYamlFiles } from "../lib/fileUtils";

async function run() {
    const argv = yargs(process.argv.slice(2)).option("suitePath", { type: "string" }).parseSync();
    const suitePath = argv.suitePath || "evals/suites";

    try {
        // Single call handles directory traversal and filtering robustly
        const suiteFiles = await findYamlFiles(suitePath);
        if (suiteFiles.length === 0) {
             console.error(`No .yaml files found at path: ${suitePath}`); return;
        }
        // ... use suiteFiles ...
    } catch (error: any) {
        console.error(`Failed to find suite files: ${error.message}`);
        process.exit(1);
     }
}
```
The `findYamlFiles` utility encapsulates the logic for checking if the input path is a file or directory, recursively searching directories using reliable methods (like the `glob` library), filtering for `.yaml` extensions, resolving paths, and handling errors, making the main script cleaner.

## Key Concepts

1.  **Target Environment:** Node.js execution context, typically run via `pnpm`, `npm`, or `yarn` scripts defined in `package.json`. Can leverage the full Node.js API set (`fs`, `path`, `child_process`, `process`, `os`, etc.) and external npm packages (usually `devDependencies`).
2.  **Common Functionality:** Focus on recurring script tasks:
    *   **Argument Parsing:** Using or wrapping libraries like `yargs` or `commander`.
    *   **File System Operations:** Robust wrappers or specialized functions using `fs/promises` and potentially `glob` for pattern matching/directory traversal (e.g., `findFiles`, `readFileSafe`, `writeFileSafe`, `copyDir`, `ensureDir`). These might reuse or adapt utilities from `src/utils/fs.ts` ([Chapter 42: File System Utilities](42_file_system_utilities.md)) if those utilities don't depend on VS Code APIs.
    *   **Shell Command Execution:** Helpers for running external commands (e.g., `tsc`, `eslint`, `git`, `docker`) using `child_process` or libraries like `execa`, capturing output, and handling exit codes reliably.
    *   **Logging:** A simple console logger (`console.log`, `console.error`) perhaps enhanced with libraries like `chalk` for colored output, providing consistent formatting for status and error reporting in scripts.
    *   **Path Manipulation:** Using or wrapping Node.js `path` module functions for resolving, joining, normalizing paths (reusing utilities from `src/utils/path.ts` ([Chapter 43: Path Utilities](43_path_utilities.md)) that don't depend on VS Code APIs).
    *   **Concurrency Control:** Using libraries like `p-limit` to manage parallelism when running many asynchronous tasks (e.g., processing multiple files, running multiple eval tasks).
    *   **JSON/YAML Handling:** Safe reading/parsing/writing of JSON or YAML files, potentially using Zod ([Chapter 40: Schemas (Zod)](40_schemas__zod_.md)) for validation.
3.  **Location:** Organized within dedicated directories like `scripts/lib/` (for general build/dev scripts) or `evals/src/lib/` (for eval-specific helpers). Some highly generic utilities (e.g., string manipulation, basic path normalization without workspace context) might reside in the shared `src/utils/` directory if they are usable by both the extension runtime and scripts without relying on VS Code APIs.
4.  **Reusability:** Designed to be imported and used across different `.js` or `.ts` script files within the project. Scripts written in TypeScript (`.ts`) need a way to be executed, typically via `ts-node` (e.g., `node -r ts-node/register scripts/myScript.ts`) or by being compiled to JavaScript first as part of a pre-build step (as Roo-Code does with `esbuild`).

## Using Script Utilities

These are typically imported as standard functions or modules within script files (`*.js` or `*.ts`).

**Example 1: Parsing Arguments (`evals/src/runEval.ts`)**

*(See code in Chapter 53)*
*   Directly uses the `yargs` library. Could be wrapped in a utility if argument definitions become complex or shared across scripts (e.g., a function `parseEvalArgs()` exported from `evals/src/lib/args.ts`).

**Example 2: Finding Files (`evals/src/lib/findSuites.ts` - Conceptual)**

```typescript
// --- File: evals/src/lib/findSuites.ts --- (Conceptual Utility)
import * as fs from "fs/promises";
import * as path from "path";
import { glob } from "glob"; // Use glob library for pattern matching

/**
 * Finds YAML files based on a path (file or directory).
 * @param suitePathOrDir Path to a specific .yaml file or a directory containing .yaml files.
 * @returns Array of absolute paths to found .yaml files.
 * @throws Error if path does not exist or is not a file/directory.
 */
export async function findYamlFiles(suitePathOrDir: string): Promise<string[]> {
    const absolutePath = path.resolve(suitePathOrDir); // Ensure absolute path
    let stats: fs.Stats;
    try {
        stats = await fs.stat(absolutePath);
    } catch (error: any) {
        if (error.code === 'ENOENT') {
            throw new Error(`Path not found: ${absolutePath}`);
        }
        throw error; // Re-throw other errors
    }

    if (stats.isFile()) {
        // If it's a file, check if it's YAML and return it
        return absolutePath.endsWith(".yaml") || absolutePath.endsWith(".yml")
            ? [absolutePath]
            : [];
    } else if (stats.isDirectory()) {
        // If it's a directory, use glob to find YAML files recursively
        const pattern = "**/*.@(yaml|yml)"; // Glob pattern for .yaml or .yml
        // Use { posix: true } for consistent glob patterns across OS
        const files = await glob(pattern, { cwd: absolutePath, absolute: true, nodir: true, posix: true });
        return files;
    } else {
        throw new Error(`Path is not a file or directory: ${absolutePath}`);
    }
}

// --- Usage in evals/src/runEval.ts ---
// import { findYamlFiles } from "./lib/findSuites";
// try {
//   const suitePaths = await findYamlFiles(argv.suite);
//   // ... process files
// } catch (error) { console.error(error.message); process.exit(1); }
```
*Explanation:* A utility function `findYamlFiles` uses `fs.stat` and the `glob` library to robustly handle file/directory input and find matching files. It throws errors on failure for the calling script to handle.

**Example 3: Executing a Shell Command (`scripts/lib/exec.ts` - Conceptual)**

```typescript
// --- File: scripts/lib/exec.ts --- (Conceptual Utility using execa)
import { execa, Options, ExecaReturnValue } from 'execa'; // Use execa
import chalk from 'chalk'; // For colored output

/** Executes a shell command using execa and streams output */
export async function runCommand(
    command: string,
    args: string[] = [],
    options: Options = {}
): Promise<ExecaReturnValue> {
    const commandString = `${command} ${args.join(" ")}`;
    console.log(chalk.blue(`$ ${commandString} ${options.cwd ? `(in ${options.cwd})`: ''}`));
    try {
        // Use execa, pipe output to current process's streams
        const subprocess = execa(command, args, { stdio: 'inherit', ...options });
        return await subprocess; // Wait for completion
    } catch (error: any) {
        // execa throws detailed error object on non-zero exit
        console.error(chalk.red(`Command failed: ${commandString}`));
        console.error(chalk.red(`Exit Code: ${error.exitCode}`));
        if (error.stderr) console.error(chalk.red(`Stderr: ${error.stderr}`));
        if (error.stdout) console.error(chalk.red(`Stdout: ${error.stdout}`)); // Might be useful context
        throw error; // Re-throw for the calling script to handle/exit
    }
}

// --- Usage in scripts/build.ts ---
// import { runCommand } from "./lib/exec";
// try {
//   await runCommand("pnpm", ["--filter", "webview-ui", "build"], { cwd: path.resolve(__dirname, '..') });
//   await runCommand("node", ["./esbuild.js"]);
// } catch(e) { process.exit(1); }
```
*Explanation:* A utility `runCommand` wraps `execa`. It logs the command, executes it with `stdio: 'inherit'` (to show live output), and throws a detailed error if the command fails (non-zero exit code). The build script uses this to run build steps sequentially, exiting if any step fails.

**Example 4: Logging (`scripts/lib/logger.ts` - Conceptual)**

*(See code in Key Concepts section)*
*   `scriptLogger` object uses `chalk` for colored console output (`info`, `warn`, `error`, `success`). Used by scripts for consistent status updates.

## Code Walkthrough

The provided code focuses heavily on the Evals system and shared runtime utilities. We examine relevant utilities.

### Shared Utilities (`src/utils/`)

*   **`fs.ts` ([Chapter 42]):** `fileExistsAtPath`, `directoryExistsAtPath`, `safeWriteFile`, `createDirectoriesForFile` are directly usable and essential for scripts managing files (e.g., eval outputs, build artifacts). `safeReadFile` is also useful.
*   **`path.ts` ([Chapter 43]):** `normalizePath`, `toPosix()`, `arePathsEqual`, `relativePath`/`toRelativePath` are essential for script path manipulation. **Note:** `getWorkspacePath` relies on `vscode.workspace` and **cannot** be used directly. Scripts must determine the base path from `process.cwd()` or arguments and pass it explicitly to functions like `relativePath`.
*   **`logging.ts`:** The `logger` instance. Scripts might need to configure its transport to use `console` instead of/in addition to the VS Code Output Channel, or use a simpler dedicated script logger (like the `chalk` example).
*   **`text-normalization.ts` ([Chapter 45]):** Pure string functions like `normalizeLineEndings` are directly usable.

### Evals Utilities (`evals/src/lib/`)

*   **`openai.ts` (from Ch 55):** Contains `getCompletions` using `axios`. A specific network utility for evals needing OpenAI-compatible endpoints.
*   **`findSuites.ts` (Conceptual `findYamlFiles`):** Uses `fs.stat`, `path.resolve`, `glob`. File system utility specific to finding YAMLs for the eval runner.
*   **`contextUtils.ts` (Conceptual):** Would use `fs/promises` (`mkdtemp`, `writeFile`, `rm`) and `path` for managing temporary files/directories needed for eval contexts.

### Build Script Utilities (`scripts/lib/`)

*   **`copyWasm.js` (Conceptual):** Contains `esbuild` plugin logic using `fs` ([Chapter 56: Build System](56_build_system.md)).
*   **`exec.ts` (Conceptual `runCommand`):** Uses `child_process` or `execa` for executing shell commands.
*   **`logger.ts` (Conceptual `scriptLogger`):** Uses `console` and `chalk` for script logging.
*   **(Other potential utils):** Functions for reading/writing JSON, checking dependencies, interacting with Git CLI, etc.

## Internal Implementation

*   **File System:** Relies on Node.js `fs`/`fs/promises` API. Libraries like `glob` provide pattern matching.
*   **Command Execution:** Uses Node.js `child_process` (`spawn`, `exec`, `fork`) or libraries like `execa`.
*   **Argument Parsing:** Uses libraries like `yargs` which parse `process.argv`.
*   **Logging:** Uses Node.js `console` module, potentially enhanced with `chalk`.
*   **Path:** Uses Node.js `path` module.

## Modification Guidance

Modifications usually involve adding new helper functions for common script tasks or improving existing ones for robustness or flexibility.

1.  **Adding a JSON Reading/Validation Utility:**
    *   **Define:** Add `async function readValidateJson<T>(filePath: string, schema: z.ZodSchema<T>): Promise<T>` to `scripts/lib/fsUtils.ts`.
    *   **Implement:** Use `safeReadFile` ([Chapter 42: File System Utilities](42_file_system_utilities.md)), `JSON.parse`, and `schema.parse()` (using `parse` to throw on error, suitable for scripts that should fail fast). Include clear error logging using `scriptLogger`.
    *   **Usage:** Scripts use `await readValidateJson(...)` to load and validate critical JSON files, failing the script if invalid.

2.  **Improving `runCommand` with Error Summaries:**
    *   **Modify `runCommand`:** In the `catch` block for `execa`, format a more concise summary of the error (e.g., command, exit code, first few lines of stderr) before re-throwing the full error object. This gives quicker feedback in logs.

3.  **Adding a Utility to Check for Dirty Git Status:**
    *   **Define:** Add `async function checkGitDirty(cwd: string): Promise<boolean>` to `scripts/lib/gitUtils.ts`.
    *   **Implement:** Use `runCommand` (or `execa` directly) to execute `git status --porcelain`. Check if the `stdout` is empty. Return `true` if not empty (dirty), `false` otherwise. Handle errors if `git` isn't found or fails.
    *   **Usage:** Build or release scripts can call this to ensure a clean working directory before proceeding.

**Best Practices:**

*   **Reusability:** Extract common script logic into `scripts/lib/` or `evals/src/lib/`. Share with `src/utils/` only if completely independent of VS Code APIs.
*   **Error Handling:** Scripts should usually fail fast on unexpected errors. Utilities should throw informative errors or return clear failure indicators. Use `scriptLogger` for consistent error reporting.
*   **Cross-Platform:** Use Node.js `path`. Use `execa` or `spawn` with argument arrays for commands. Use `cross-env` in `package.json` scripts for setting environment variables.
*   **Logging:** Provide clear `info`, `warn`, `error`, `success` messages using `scriptLogger` or similar.
*   **Async:** Use `async/await` for I/O operations. Use concurrency limiters (`p-limit`) if running many parallel tasks.
*   **Dependencies:** Use `devDependencies`.

**Potential Pitfalls:**

*   **VS Code API Usage:** Cannot use `vscode` module in standalone scripts.
*   **Path Resolution:** Scripts run from different CWDs. Use `path.resolve`, `__dirname`, `process.cwd()`.
*   **Shell Command Failures:** Need reliable error checking (exit codes, `stderr`). `execa` helps.
*   **Unhandled Rejections:** Await promises and handle rejections in scripts.
*   **Environment Differences:** Scripts might fail in CI due to missing tools, dependencies, or different env vars.

## Conclusion

Script Utilities provide essential helper functions that streamline the development, build, testing, and evaluation processes for the Roo-Code project. By encapsulating common tasks like argument parsing, robust file system operations, reliable command execution, and consistent logging into reusable modules tailored for the Node.js environment, they reduce code duplication, improve consistency, and make individual scripts cleaner and easier to maintain. While potentially sharing some low-level logic with runtime utilities, script utilities can directly leverage the full Node.js API and external CLI tools, distinct from the constraints of the VS Code extension runtime environment.

Next, we examine the specific system used to build the Roo-Code extension itself: [Chapter 56: Build System](56_build_system.md).
---
# Chapter 56: Build System

Continuing from [Chapter 55: Script Utilities](55_script_utilities.md), which covered helpers used in various project scripts, this chapter focuses on the specific processes and tools used to compile, bundle, and package the Roo-Code extension for distribution and development: the **Build System**.

## Motivation: Transforming Source Code into a Deployable Extension

The Roo-Code source code, written primarily in TypeScript (`src/`) with React for the WebView UI (`webview-ui/`) and including assets like images (`images/`), locale files (`public/locales`), and WASM modules (`node_modules/`), needs to be transformed into a format that VS Code can load and execute. This involves several steps:

1.  **TypeScript Compilation:** Converting TypeScript code (both extension host `src/` and WebView UI `webview-ui/src/`) into JavaScript.
2.  **Bundling:** Combining multiple JavaScript/TypeScript files and their dependencies (from `node_modules`) into fewer, optimized bundles to improve loading performance and simplify distribution.
3.  **Asset Handling:** Copying necessary assets (locales, images, WASM files, NLS files, WebView build output) to the correct locations within the final distribution directory (`dist/`).
4.  **Environment Configuration:** Injecting build-time environment variables (e.g., `NODE_ENV=production` or `development`, PostHog API keys for [Chapter 52: TelemetryService](52_telemetryservice.md)).
5.  **Development vs. Production Builds:** Providing different outputs: development builds include source maps for easier debugging and might leverage watch modes for faster rebuilds; production builds prioritize smaller size and performance through minification.
6.  **WebView UI Build:** Handling the separate build process for the React-based WebView UI, using Vite ([Chapter 1: WebView UI](01_webview_ui.md)), and ensuring its output is correctly placed within the main `dist` folder.
7.  **Packaging Preparation:** Ensuring the `dist/` directory contains all necessary files (`extension.js`, `package.json`, assets, WebView build) correctly structured for packaging into a `.vsix` file using the `vsce` tool.

Manually performing these steps is tedious, error-prone, and slow. A dedicated **Build System** automates this process, ensuring consistency, efficiency, and correctness. Roo-Code utilizes `esbuild` for its speed in bundling the extension host code and relies on Vite for building the WebView UI, orchestrated via custom Node.js scripts (`esbuild.js`, potentially others like `scripts/prepare.js`) and managed through `pnpm` scripts in `package.json`.

**Central Use Case:** A developer wants to create a production-ready build of the Roo-Code extension to package it for the VS Code Marketplace.

1.  The developer runs `pnpm run build:prod` (or `pnpm run vscode:prepublish`, which usually calls the production build).
2.  This command executes `pnpm prepare` then `cross-env NODE_ENV=production node ./esbuild.js`.
3.  **`scripts/prepare.js` (Pre-build):** Might perform tasks like generating schema types from Zod definitions if not done automatically elsewhere.
4.  **Build Script (`esbuild.js`):**
    *   Sets `NODE_ENV` to `production`. Cleans the `dist/` directory using utilities like `rimraf` (or `fs.rm`).
    *   Defines `esbuild` configuration for the **Extension Host** (`src/extension.ts`):
        *   `platform: 'node'`, `format: 'cjs'`, `target: 'node16'`.
        *   `external: ['vscode']`.
        *   `bundle: true`, `minify: true`, `sourcemap: false`.
        *   `define`: Injects production environment variables (e.g., PostHog key).
        *   `outfile: 'dist/extension.js'`.
    *   Executes `await esbuild.build(extensionConfig)`.
    *   Triggers the **WebView UI Build:** Executes `pnpm --filter webview-ui build` (via `child_process.execSync`). This runs Vite's production build (`vite build`), outputting optimized JS/CSS bundles and assets into `webview-ui/build/`.
    *   **Asset Copying:** Uses script utilities ([Chapter 55: Script Utilities](55_script_utilities.md), [Chapter 42: File System Utilities](42_file_system_utilities.md)) like `copyRecursiveSync` / `fs.cpSync`:
        *   Copies `webview-ui/build/*` into `dist/webview-ui/`.
        *   Copies WASM files (`tree-sitter-*.wasm`, `tree-sitter.wasm`) to `dist/`.
        *   Copies `public/locales/*` into `dist/locales/`.
        *   Copies `package.nls*.json` files to `dist/`.
        *   Copies `images/*` to `dist/images/`.
5.  **Output:** The `dist/` directory contains the production-ready `extension.js`, the optimized WebView UI assets in `dist/webview-ui/`, locale files, WASM files, NLS files, and other static assets.
6.  **Packaging:** The developer can now run `pnpm package` (which calls `vsce package`). `vsce` reads `package.json` and `.vscodeignore` to bundle the contents of `dist/` into a `.vsix` file.

## Key Concepts

1.  **`esbuild`:** Used for its speed to bundle the **extension host** TypeScript code (`src/`) into a single JavaScript file (`dist/extension.js`). Configured via `esbuild.js`.
2.  **Vite:** Used to build the **WebView UI** React application (`webview-ui/`). Configured via `webview-ui/vite.config.ts`. Produces optimized static assets (JS, CSS) in `webview-ui/build/`. ([Chapter 1: WebView UI](01_webview_ui.md)).
3.  **Build Script (`esbuild.js`):** A Node.js script serving as the main orchestrator.
    *   Uses `esbuild`'s JavaScript API for the host build.
    *   Uses `child_process` to trigger the separate Vite build for the WebView UI.
    *   Includes logic (using Node.js `fs` or script utilities) to copy assets into `dist/`.
    *   Handles different modes (development/production, watch).
4.  **`package.json` Scripts:** Defines commands (`build:prod`, `watch:host`, `watch:webview`, `compile`, `vscode:prepublish`, `package`) that trigger the build script (`esbuild.js`), Vite (`pnpm --filter ...`), `tsc`, or `vsce` with appropriate environment variables or flags. Uses `cross-env` for cross-platform `NODE_ENV` setting.
5.  **Entry Points & Outputs:**
    *   Host: `src/extension.ts` -> `dist/extension.js`.
    *   WebView: `webview-ui/src/index.tsx` -> `webview-ui/build/assets/*` (via Vite) -> copied to `dist/webview-ui/assets/*`.
6.  **Configuration (`esbuild.js` for Host):** Key settings: `platform: 'node'`, `format: 'cjs'`, `target: 'node16'`, `external: ['vscode']`, `bundle: true`, `minify: isProduction`, `sourcemap: !isProduction` (often `'inline'`), `define` (for env vars), `outfile`.
7.  **Asset Handling:** A critical build step, executed by `esbuild.js` after bundling. Copies runtime assets to `dist/`: WASM, locales, WebView build output, NLS files, images.
8.  **Development Mode (`watch:host`, `watch:webview`):** Uses separate watch processes. `esbuild --watch` for host code (`dist/extension.js`). `vite dev` for WebView UI (HMR dev server).
9.  **Production Mode (`build:prod`):** Orchestrates the full sequence: clean `dist`, `esbuild` production build for host, `vite build` for WebView, copy all assets.
10. **Type Checking (`compile` script):** `tsc --noEmit` provides comprehensive type checking, complementary to `esbuild`/`vite`'s faster transpilation. Crucial for CI.
11. **Packaging (`vsce`, `package` script):** `vsce package` bundles the contents of `dist/` (respecting `.vscodeignore`) into a `.vsix` file. `.vscodeignore` prevents source code, dev dependencies, and build artifacts from being included.
12. **Preparation Script (`scripts/prepare.js`):** An optional script run before builds (e.g., via `pnpm prepare`). Could be used for code generation tasks like creating type files from Zod schemas if not handled by other means.

## Executing the Build

Builds are run using `pnpm` (or `npm`/`yarn`) commands defined in `package.json`.

```json
// --- File: package.json (Excerpt with Key Scripts) ---
{
  "scripts": {
    // Installs dependencies everywhere
    "install:all": "pnpm install && pnpm --filter webview-ui install",
    // Prepares files needed by esbuild/vite (e.g., schema type generation)
    "prepare": "node ./scripts/prepare.js",
    // Type checking only
    "compile": "tsc -p ./ --noEmit",
    // Clean output directories
    "clean": "rimraf dist webview-ui/build", // Using rimraf for cross-platform deletion
    // Production build for host and webview, plus asset copying
    "build:prod": "pnpm clean && pnpm prepare && cross-env NODE_ENV=production node ./esbuild.js",
    // Dev build for host (used by watch)
    "build:dev": "pnpm clean && pnpm prepare && cross-env NODE_ENV=development node ./esbuild.js",
    // Watch host code using esbuild watch
    "watch:host": "pnpm build:dev --watch",
    // Start Vite dev server for WebView HMR
    "watch:webview": "pnpm --filter webview-ui dev",
    // Runs before 'vsce package' or 'vsce publish'
    "vscode:prepublish": "pnpm run compile && pnpm run build:prod", // Add type check
    // Package using vsce (ensure vsce is installed globally or via devDeps)
    // --yarn flag helps vsce work better with pnpm's node_modules structure
    "package": "vsce package --out ./releases/roo-code.vsix --yarn",
    // Publish (requires PAT)
    "publish": "vsce publish --yarn"
    // ... test, lint, etc.
  }
}
```

*   **Development:** `pnpm install:all`, then run `pnpm watch:host` and `pnpm watch:webview` in separate terminals. Launch debugger (F5).
*   **Production Build & Package:** `pnpm install:all`, then `pnpm compile`, `pnpm build:prod`, then `pnpm package`.

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
const chalk = require("chalk"); // Use chalk for logging (dev dependency)
const rimraf = require("rimraf"); // Use rimraf for cleaning (dev dependency)

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

    // Copy WASM files (from node_modules to dist/)
    // ... (Logic as in Chapter 56) ...
    log.info("Copied WASM files.");

    // Copy Locales for WebView (from public/locales to dist/locales)
    const localeWebViewSrc = path.join(__dirname, "public", "locales");
    const localeWebViewDest = path.join(outDir, "locales");
    if (fs.existsSync(localeWebViewSrc)) {
        copyRecursiveSync(localeWebViewSrc, localeWebViewDest);
        log.info("Copied WebView locale files.");
    } else { log.warn(`WebView locales not found: ${localeWebViewSrc}`); }

    // Copy NLS files for Extension Host (from root to dist/)
    const nlsFiles = fs.readdirSync(__dirname).filter(f => f.startsWith('package.nls') && f.endsWith('.json'));
    nlsFiles.forEach(f => fs.copyFileSync(path.join(__dirname, f), path.join(outDir, f)));
    log.info("Copied NLS files.");

    // Copy other assets (e.g., images)
    const imagesSrc = path.join(__dirname, "images");
    if (fs.existsSync(imagesSrc)) { /* ... copy ... */ log.info("Copied images."); }

    log.info("Asset copying finished.");
}

// --- esbuild Common Config ---
const commonOptions = { /* ... bundle, sourcemap, minify, external, define ... */ };

// --- Extension Host Config ---
const extensionConfig = { /* ... common + platform:node, target, format:cjs, entryPoints, outfile ... */ };

// --- Build Function ---
async function build() {
	try {
        log.info(`Starting ${isProduction ? 'production' : 'development'} build...`);
        // Clean output directory using rimraf
        log.info(`Cleaning output directory: ${outDir}`);
        await rimraf(outDir); // Asynchronous rimraf
        ensureDirSync(outDir);

		// 1. Build Extension Host with esbuild
        log.info("Building extension host...");
		await esbuild.build({
			...extensionConfig,
			watch: isWatch ? { onRebuild(error) { log.info(error ? `❌ Host rebuild failed` : `✅ Host rebuilt`); if(error) log.error('', error); } } : undefined,
		});
        log.info("Extension host build complete.");

        // 2. Build WebView UI using Vite (only for production or non-host-watch builds)
        if (!isWatch || isProduction) {
            log.info("Building webview UI via Vite...");
            try {
                const webviewBuildCommand = isProduction ? 'build' : 'build --mode development';
                // Execute `pnpm build` within the `webview-ui` package
                childProcess.execSync(`pnpm ${webviewBuildCommand}`, {
                    stdio: 'inherit', // Show vite output
                    cwd: path.join(__dirname, "webview-ui") // Run in correct directory
                });
                log.info("Webview UI build complete.");

                // 3. Copy WebView build output to dist
                const webviewBuildSrc = path.join(__dirname, "webview-ui", "build");
                const webviewBuildDest = path.join(outDir, "webview-ui");
                if (fs.existsSync(webviewBuildSrc)) {
                    log.info(`Copying webview build to ${webviewBuildDest}...`);
                    ensureDirSync(webviewBuildDest);
                    copyRecursiveSync(webviewBuildSrc, webviewBuildDest);
                } else { log.warn(`Webview build output not found: ${webviewBuildSrc}`); }

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

*   **Dependencies:** Uses `chalk` for logging, `rimraf` for cleaning `dist`.
*   **`copyAssets`:** Explicitly copies required assets (WASM, WebView locales, Host NLS files, images) to the correct subdirectories within `dist/`. Uses `fs` and helper functions.
*   **Build Function:**
    1.  Cleans `dist` using `rimraf`.
    2.  Builds host with `esbuild` (with watch option if `isWatch`).
    3.  Conditionally builds WebView UI with Vite (`pnpm --filter webview-ui build`) using `child_process.execSync` *unless* in host watch mode.
    4.  Conditionally copies Vite output (`webview-ui/build/`) to `dist/webview-ui/`.
    5.  Calls `copyAssets` to copy remaining static files.
*   **Watch Mode:** Host watch (`esbuild --watch`) rebuilds `dist/extension.js`. WebView watch (`vite dev`) uses its own HMR server and doesn't involve this script directly after initial setup.

### Vite Configuration (`webview-ui/vite.config.ts`)

*(See code in Chapter 1)*
*   Configures Vite for the React app, including plugins, aliases, dev server, and crucially, `build.outDir: "build"`.

### `.vscodeignore` (Conceptual)

*(See code in Chapter 56 - Key Concepts)*
*   Crucial for excluding source files (`src/`, `webview-ui/src/`), dev dependencies (`node_modules/`), config files, intermediate build outputs (`webview-ui/build/`), etc., from the final `.vsix` package, while ensuring the contents of `dist/` *are* included.

## Internal Implementation

1.  **Script Execution:** `pnpm build:prod` -> `node ./esbuild.js` (`NODE_ENV=production`).
2.  **Cleanup:** `rimraf dist` deletes output directory. `ensureDirSync` recreates it.
3.  **Host Build:** `esbuild.build(extensionConfig)` bundles `src/` into `dist/extension.js` (minified).
4.  **WebView Build:** `execSync('pnpm --filter webview-ui build')` runs Vite. Vite bundles `webview-ui/src/` into `webview-ui/build/`.
5.  **Asset Copying:** `copyRecursiveSync` copies `webview-ui/build/*` to `dist/webview-ui/`. `copyAssets` copies WASM, locales, NLS files into `dist/`.
6.  **Completion:** `dist/` contains all runtime files.
7.  **Packaging:** `vsce package` reads `package.json`, uses `.vscodeignore` to select files from `dist/` and root, creates `.vsix`.

## Modification Guidance

Modifications involve changing build targets, adding entry points, configuring plugins, or managing assets.

1.  **Changing Target Node Version:**
    *   **`esbuild.js`:** Modify `target: 'node16'` in `extensionConfig`. Check VS Code compatibility.

2.  **Adding a New Asset Type (e.g., Fonts):**
    *   **`esbuild.js`:** Modify `copyAssets`. Add logic to find source font files and copy them to `dist/` (e.g., `dist/fonts` or `dist/webview-ui/assets/fonts`).
    *   **Code:** Update references to use correct relative paths within the packaged extension.
    *   **`.vscodeignore`:** Ensure the new assets within `dist/` are *not* ignored.

3.  **Adding an esbuild Plugin:**
    *   **Install:** `pnpm add -D some-esbuild-plugin`.
    *   **`esbuild.js`:** Import the plugin. Add it to the `plugins` array in `extensionConfig` (or a conceptual `webviewConfig` if using esbuild there).

4.  **Configuring Vite Build Options:**
    *   **`webview-ui/vite.config.ts`:** Modify options within the `build` section (e.g., `chunkSizeWarningLimit`, `rollupOptions`).

**Best Practices:**

*   **Use Correct Tools:** `esbuild` for host speed, Vite for WebView features (HMR, optimized builds).
*   **Clean Output (`dist/`):** Standard output dir. Clean before builds (`rimraf`).
*   **Separate Scripts:** Use `package.json` scripts for orchestration (`build:prod`, `watch:host`, `watch:webview`, `compile`, `package`).
*   **Environment Variables:** Use `NODE_ENV` via `cross-env`. Use `define` for build-time constants. Handle build secrets via CI/build environment variables.
*   **`external: ['vscode']`:** Essential for host.
*   **Asset Management:** Explicitly copy *all* required runtime assets to `dist/`. Verify paths.
*   **`.vscodeignore`:** Maintain carefully for minimal package size. Exclude source, dev deps, build artifacts. Include `dist/`.
*   **Type Checking (`tsc --noEmit`):** Run separately for full type safety.

**Potential Pitfalls:**

*   **Missing Assets in `dist/`:** Forgetting WASM, locales, NLS, images, or the WebView build output. Leads to runtime errors.
*   **Incorrect `external`/`platform`.**
*   **Asset Path Issues:** Code referencing assets using paths incorrect relative to the final `dist/` structure.
*   **`.vscodeignore` Errors:** Ignoring needed files in `dist/` or including source files.
*   **Build Environment:** Build failing in CI due to missing global tools (`vsce`), dependencies, or different Node/pnpm versions. Ensure CI environment mirrors local setup.
*   **Vite/Esbuild Conflicts:** Ensure the two build processes target the correct output locations and don't interfere. Orchestrate sequential execution correctly in `build:prod`.

## Conclusion

The Build System, orchestrated via `esbuild.js` and `package.json` scripts, reliably transforms Roo-Code's source code into a functional and distributable VS Code extension. It leverages the speed of `esbuild` for the extension host and the rich features of Vite for the WebView UI. Key aspects include distinct configurations for host vs. web, handling development (watch mode, source maps) vs. production (minification) builds, injecting environment variables, and crucially, ensuring all necessary runtime assets (WASM, locales, WebView bundles, NLS files) are correctly copied to the `dist/` directory, ready for packaging with `vsce`. This automated system is fundamental for both efficient development and creating optimized, distributable extension builds.

Next, we will look at how the project ensures code quality and functionality through automated tests: [Chapter 57: Testing Framework](57_testing_framework.md).
---
# Chapter 57: Testing Framework

Continuing from [Chapter 56: Build System](56_build_system.md), which focused on compiling and packaging Roo-Code, this chapter delves into the strategies and tools used to ensure the quality, correctness, and stability of the codebase: the **Testing Framework**.

## Motivation: Ensuring Quality and Preventing Regressions

As Roo-Code grows in complexity, with interactions between the extension host, WebView UI, LLM APIs, file system, terminal, and various utilities, the risk of introducing bugs or regressions increases significantly. Manually testing every feature and edge case after each change is impractical and unreliable.

A robust testing framework provides an automated way to:

1.  **Verify Correctness:** Ensure individual functions, components, and modules behave as expected according to their specifications.
2.  **Prevent Regressions:** Automatically detect when changes inadvertently break existing functionality.
3.  **Improve Design:** Writing testable code often encourages better design practices (modularity, dependency injection, pure functions).
4.  **Facilitate Refactoring:** Provide confidence that code can be refactored or updated without breaking core functionality.
5.  **Document Behavior:** Tests serve as executable documentation, illustrating how components are intended to be used and what their expected outputs are.

Roo-Code employs a combination of testing strategies, primarily using **Jest** as the test runner and framework, along with **React Testing Library** for testing WebView UI components.

**Central Use Case:** A developer refactors the `normalizeLineEndings` utility ([Chapter 45: Text Normalization Utilities](45_text_normalization_utilities.md)). They need to ensure the refactoring didn't break its functionality for different line ending combinations.

1.  **Write/Update Test:** The developer ensures a test file exists (e.g., `src/utils/text-normalization.test.ts`) with comprehensive test cases:
    ```typescript
    // --- File: src/utils/text-normalization.test.ts ---
    import { normalizeLineEndings } from './text-normalization';

    describe('normalizeLineEndings', () => {
      it('should replace CRLF with LF', () => {
        expect(normalizeLineEndings("Hello\r\nWorld")).toBe("Hello\nWorld");
      });
      it('should replace CR with LF', () => {
        expect(normalizeLineEndings("Hello\rWorld")).toBe("Hello\nWorld");
      });
      // ... other test cases for mixed endings, empty strings, no endings, null/undefined ...
    });
    ```
2.  **Run Tests:** The developer runs `pnpm test` (or a more specific command like `pnpm test src/utils/text-normalization.test.ts`).
3.  **Jest Execution:** Jest discovers the test file, executes the code within the `describe` and `it` blocks using the Node.js test environment (as configured for the "Host" project in `jest.config.js`), runs the `expect` assertions, and compares the actual output of the refactored `normalizeLineEndings` with the expected output.
4.  **Results:** Jest reports whether all tests passed or failed. If the refactoring introduced a bug, the corresponding test case(s) will fail, immediately alerting the developer *before* the code is merged or deployed.

## Key Concepts

1.  **Jest:** A popular JavaScript/TypeScript testing framework. Provides:
    *   **Test Runner:** Discovers and executes test files (typically `*.test.ts`, `*.test.tsx`, `*.spec.ts`).
    *   **Assertion Library:** `expect()` with matchers (`.toBe()`, `.toEqual()`, `.toHaveBeenCalledWith()`, `.toThrow()`, etc.).
    *   **Mocking:** `jest.fn()`, `jest.mock()`, `jest.spyOn()` for isolating units under test and faking dependencies (VS Code API, network calls, modules).
    *   **Snapshot Testing:** Capturing component render output or data structures to detect unexpected UI or data changes. (Used sparingly).
    *   **Configuration (`jest.config.js`):** Defines test environments, module resolution (`moduleNameMapper` for path aliases), coverage settings, setup files, and **projects** for separate host/UI testing configurations.

2.  **React Testing Library (`@testing-library/react`, `@testing-library/jest-dom`):** Library for testing React components focusing on user interaction and accessibility. Used for WebView UI tests.
    *   **`render`:** Renders components into a simulated DOM (JSDOM).
    *   **Queries (`screen.getByRole`, `getByLabelText`, `getByText`, etc.):** Finds elements based on user-visible attributes, promoting user-centric testing.
    *   **`fireEvent`/`userEvent`:** Simulates user interactions. `@testing-library/user-event` is preferred for realism.
    *   **Custom Matchers:** `@testing-library/jest-dom` adds DOM-specific assertions like `.toBeInTheDocument()`, `.toHaveValue()`.

3.  **Testing Environments (`jest.config.js` `projects`):** Crucial for Roo-Code.
    *   **Extension Host Tests (`displayName: "Host"`, `testEnvironment: 'node'`):** For code in `src/`. Tests utilities, core logic, services. VS Code APIs must be mocked.
    *   **WebView UI Tests (`displayName: "WebView"`, `testEnvironment: 'jsdom'`):** For code in `webview-ui/src/`. Tests React components. Simulates a browser environment. VS Code toolkit ([Chapter 32: VSCode Webview UI Toolkit Wrappers](32_vscode_webview_ui_toolkit_wrappers.md)) and `postMessage` ([Chapter 3: Webview/Extension Message Protocol](03_webview_extension_message_protocol.md)) must be mocked.

4.  **Mocking Dependencies:** Essential for isolation.
    *   **`vscode` API:** Mocked via `src/__mocks__/vscode.ts` (using `jest.fn()`). Allows testing host code.
    *   **`@vscode/webview-ui-toolkit/react`:** Mocked via `src/__mocks__/@vscode/webview-ui-toolkit/react.ts`. Allows testing UI components in JSDOM.
    *   **`postMessage`:** The `vscode` utility wrapper (`webview-ui/src/utils/vscode.ts`) is mocked (`jest.mock('@/utils/vscode', ...)`) in UI tests to assert messages sent to the host.
    *   **API Handlers/Network:** Modules making external calls (`ApiHandler`, `axios`) are mocked using `jest.mock()`.
    *   **File System (`fs`):** Often mocked using `jest.mock('fs/promises')` or `memfs`.

5.  **Test Structure (`describe`, `it`/`test`):** Standard Jest structure for grouping tests and defining individual test cases with the Arrange-Act-Assert pattern. Setup/teardown hooks (`beforeEach`, etc.) manage shared state.

6.  **Code Coverage:** Generated using `jest --coverage`. Helps identify untested areas. Configured in `jest.config.js` (`collectCoverageFrom`).

## Executing Tests

Tests are run via scripts defined in `package.json`.

```json
// --- File: package.json (Excerpt) ---
{
  "scripts": {
    "test": "jest", // Run all tests (host + webview projects)
    "test:watch": "jest --watch", // Run in watch mode
    "test:coverage": "jest --coverage", // Run with coverage
    // Use displayName from jest.config.js projects array
    "test:host": "jest --selectProjects Host",
    "test:ui": "jest --selectProjects WebView",
    // ... other scripts ...
  }
}
```

*   `pnpm test`: Runs all tests.
*   `pnpm test:watch`: Interactive watch mode.
*   `pnpm test:coverage`: Runs tests and generates coverage reports.
*   `pnpm test:host` / `pnpm test:ui`: Runs only the specified project's tests.

## Code Walkthrough

### Jest Configuration (`jest.config.js`)

*(See full code in Chapter 57 - Key Concepts)*

*   Defines two **projects**: "Host" (`node` env, tests in `src/`) and "WebView" (`jsdom` env, tests in `webview-ui/src/`). Using projects allows separate environments and configurations within a single Jest run.
*   Configures `ts-jest` preset.
*   Sets up `moduleNameMapper` for path aliases (`@/`, `@roo/`, `@src/`) and CSS module mocking (`identity-obj-proxy`).
*   WebView project config sets `rootDir: "webview-ui"` and crucially `roots: ["<rootDir>/.."]` to allow Jest to find the top-level `src/__mocks__` directory relative to the WebView project's root.
*   Includes coverage configuration (`collectCoverageFrom` specifies included/excluded files).

### Mocking `vscode` (`src/__mocks__/vscode.ts`)

*(See full code in Chapter 57 - Key Concepts)*

*   Provides `jest.fn()` mocks for core VS Code API functions needed by Roo-Code's host code (e.g., `window.showInformationMessage`, `workspace.getConfiguration`, `commands.executeCommand`, `secrets.get/store`, `authentication.getSession`, `languages.getDiagnostics`, `Uri.file`, etc.).
*   Provides mock objects/values for properties (`env.machineId`, `workspace.workspaceFolders`).
*   Provides simple mock classes/factories for `Uri`, `Range`, `Position`, `Selection`.
*   Provides mock enums like `DiagnosticSeverity`, `CodeActionKind`.
*   Allows tests to run host code and assert interactions with the mocked VS Code API. Tests can import the mock and use Jest methods like `(vscode.window.showInformationMessage as jest.Mock).mockResolvedValue('OK')` to control behavior for specific tests.

### Unit Test Example (Host Utility - `src/utils/path.test.ts`)

*(See code in Chapter 57 - Key Concepts)*

*   Uses `jest.mock('vscode', ...)` to provide a specific mock implementation for `workspace.workspaceFolders` within the test file.
*   Uses `describe`/`it`. Uses `beforeEach`/`afterAll` for setup/teardown (setting platform). Uses `expect(...).toBe(...)`.

### React Component Test (WebView UI - `webview-ui/src/components/settings/About.test.tsx`)

*(See code in Chapter 57 - Key Concepts)*

*   Uses `@testing-library/react` (`render`, `screen`, `fireEvent`).
*   Explicitly mocks dependencies using `jest.mock`: `@/utils/vscode` (for `postMessage`), `@/i18n/TranslationContext` (for `t`), and `@vscode/webview-ui-toolkit/react` (for toolkit components - relies on `__mocks__` via Jest config `roots` setting).
*   Uses `screen.getByRole`/`getByLabelText` for querying.
*   Uses `fireEvent.click` for interaction.
*   Uses `expect` with `@testing-library/jest-dom` matchers (`.toBeInTheDocument()`) and `jest.fn` matchers (`.toHaveBeenCalledWith()`) to verify behavior and communication.

## Internal Implementation

1.  **Test Execution:** `pnpm test` invokes `jest`.
2.  **Configuration/Projects:** Jest reads `jest.config.js`, identifies and runs tests for each project ("Host", "WebView") according to its configuration.
3.  **Environment Setup:** Jest sets up `node` or `jsdom`. JSDOM simulates browser APIs.
4.  **Mock Resolution:** Jest intercepts `import`. Uses mocks from `__mocks__` (via `roots` config) or explicit `jest.mock()` calls. `moduleNameMapper` resolves path aliases.
5.  **Test File Execution:** Code in `*.test.ts(x)` runs within the specified environment.
6.  **Rendering (UI):** `@testing-library/react`'s `render` uses JSDOM + mocked toolkit components.
7.  **Interaction/Assertion:** Tests simulate events and assert outcomes against JSDOM state or mock function calls.
8.  **Code Coverage:** Instrumentation via `ts-jest` tracks execution, generating reports.

## Modification Guidance

Modifications involve adding tests for new code, updating tests for changed code, or improving the mocking infrastructure.

1.  **Adding Tests for a New Utility Function:**
    *   Create `<filename>.test.ts` in `src/utils/`.
    *   Import function, use `describe`/`it`, `expect`. Mock dependencies. Run with `pnpm test:host`.

2.  **Adding Tests for a New React Component:**
    *   Create `<ComponentName>.test.tsx` in `webview-ui/src/components/`.
    *   Import React, testing library, component. Mock dependencies (`vscode` util, hooks, toolkit). Use `render`, `screen`, `fireEvent`, `expect`. Run with `pnpm test:ui`.

3.  **Updating the `vscode` Mock (`src/__mocks__/vscode.ts`):**
    *   If new code uses a `vscode` API not currently mocked, add a `jest.fn()` or mock value for it.
    *   If a test needs specific mock behavior, use `mockReturnValue`, `mockResolvedValue`, `mockImplementation` within the test file.

4.  **Mocking a New Module Dependency:**
    *   Use `jest.mock('../path/to/module', () => ({ exportName: jest.fn().mockResolvedValue(...) }));` at the top of the test file. Import the mock to make assertions on it.

**Best Practices:**

*   **Test Behavior, Not Implementation:** Focus tests on verifying the intended functionality from an external perspective. Use React Testing Library's philosophy for UI components.
*   **Isolate Units:** Use mocks effectively to isolate the unit under test from its dependencies.
*   **Mock Boundaries:** Mock external dependencies (VS Code API, network, toolkit) and potentially internal modules at logical boundaries. Avoid mocking trivial internal helpers.
*   **Coverage as Guide:** Use coverage reports to find gaps, but prioritize testing critical logic, edge cases, and potential failure points over achieving 100% coverage arbitrarily.
*   **Maintainability:** Write clear, readable tests. Reset mock state between tests (`clearMocks: true` or `beforeEach`). Keep mocks reasonably simple.
*   **CI Integration:** Run tests (`pnpm test`) automatically in the CI pipeline ([Chapter 58: Documentation & Community Files](58_documentation___community_files.md)) to catch regressions before merging.

**Potential Pitfalls:**

*   **Incomplete/Incorrect Mocks:** Tests passing due to faulty mocks while real code fails (or vice-versa). Requires careful mock maintenance and occasional validation against real behavior.
*   **Over-Mocking:** Mocking too much internal logic makes tests brittle and resistant to refactoring.
*   **Testing Implementation Details:** Relying on component internals (CSS classes, state names) instead of user-facing behavior.
*   **Testing Environment Mismatch:** Bugs appearing only in the real VS Code environment. Requires some level of manual or end-to-end testing (outside the scope of Jest).
*   **Flaky Tests:** Timing issues, unhandled promises, state leakage between tests. Use async utilities (`waitFor`) correctly.
*   **Slow Tests:** Complex setup, inefficient mocks, too many integration-style tests. Keep unit tests fast.

## Conclusion

The Testing Framework, primarily utilizing Jest and React Testing Library, is crucial for maintaining the quality and stability of the Roo-Code extension. By providing separate testing environments and configurations for the extension host (Node) and WebView UI (JSDOM), and leveraging comprehensive mocking for dependencies like the `vscode` API and UI toolkits, the framework enables automated verification of individual units and components. This automated testing catches regressions early, facilitates safer refactoring, and ultimately contributes to a more reliable and robust product for users. Running tests frequently, especially in CI pipelines, is a key practice for sustainable development.

Finally, we'll look at the documentation and community-related files that support the project: [Chapter 58: Documentation & Community Files](58_documentation___community_files.md).
---
# Chapter 58: Documentation & Community Files

Continuing from [Chapter 57: Testing Framework](57_testing_framework.md), which detailed how Roo-Code ensures code quality through testing, this final chapter looks at the essential supporting files that explain the project, guide contributors, and foster a healthy community: **Documentation & Community Files**.

## Motivation: Guiding Users and Contributors

A software project, especially an open-source one like Roo-Code, needs more than just functional code. To be successful, usable, and sustainable, it requires clear documentation and standard community health files to:

1.  **Explain Usage:** Guide end-users on how to install, configure, and effectively use the extension's features.
2.  **Onboard Developers:** Provide instructions for contributors on how to set up the development environment, understand the architecture (like this tutorial!), run builds ([Chapter 56: Build System](56_build_system.md)), run tests ([Chapter 57: Testing Framework](57_testing_framework.md)), and follow contribution guidelines.
3.  **Document Architecture:** Explain the design choices, core components (like `ClineProvider`, `ApiHandler`, `Cline`, etc.), data flow, and workflows within the codebase (as this tutorial series aims to do).
4.  **Set Expectations:** Define codes of conduct, contribution processes, and security policies for the community.
5.  **Facilitate Collaboration:** Make it easier for users to report bugs, request features, and for contributors to participate effectively.
6.  **Meet Marketplace Requirements:** Provide necessary information (README, LICENSE, CHANGELOG) for publishing to the VS Code Marketplace.
7.  **Define Project Standards:** Document coding conventions, code quality rules (`.roo/rules/rules.md`), and specific guidelines like those for localization (`.roo/rules-translate/`).

These files, typically located in the project root, a dedicated `docs/` directory, or standardized locations like `.github/` or `.roo/`, are crucial for the project's usability, maintainability, and community engagement.

## Key Concepts & Files

1.  **`README.md`:** The primary entry point for the project repository (e.g., on GitHub).
    *   **Purpose:** High-level overview, features, installation, quick start, links to further documentation/community.
    *   **Audience:** Users and contributors.
    *   **Format:** Markdown, often including badges, screenshots.

2.  **`LICENSE` (or `LICENSE.md`):** Contains the full open-source license text (e.g., MIT).
    *   **Purpose:** Defines legal usage and distribution rights. Essential.
    *   **Audience:** Users, contributors, legal.

3.  **`CHANGELOG.md`:** Records notable changes for each version.
    *   **Purpose:** Informs users about updates, tracks project history.
    *   **Format:** Markdown, reverse chronological order, follows conventions like Keep a Changelog ([https://keepachangelog.com/](https://keepachangelog.com/)).

4.  **`CONTRIBUTING.md`:** Guidelines for contributing code, documentation, or reporting issues.
    *   **Purpose:** Standardizes the contribution process.
    *   **Content:** Setup instructions (linking to Build/Test chapters), coding standards (linking to `rules.md`), running tests, PR process, issue templates pointers.
    *   **Audience:** Potential contributors.

5.  **`CODE_OF_CONDUCT.md`:** Establishes community behavior standards.
    *   **Purpose:** Fosters a welcoming and respectful environment. Often uses templates like Contributor Covenant.
    *   **Audience:** All community members.

6.  **`SECURITY.md`:** Outlines the process for responsibly reporting security vulnerabilities.
    *   **Purpose:** Provides a secure channel for disclosure.
    *   **Content:** Contact info (private email), scope, expected response.
    *   **Audience:** Security researchers, users.

7.  **`.github/` Directory:** GitHub-specific files:
    *   **`ISSUE_TEMPLATE/`:** Markdown templates for bug reports, feature requests. Guides users to provide necessary context.
    *   **`PULL_REQUEST_TEMPLATE.md`:** Template for PR descriptions (e.g., checklist for tests, docs).
    *   **`workflows/`:** GitHub Actions YAML files for CI/CD (lint, test, build checks on PRs; potentially release automation).

8.  **`docs/` Directory:** For more detailed documentation:
    *   **User Guides:** In-depth feature explanations, configuration details.
    *   **Architecture Documentation:** This tutorial series (`docs/architecture/*.md`), diagrams ([Chapter 51: IPC (Inter-Process Communication)](51_ipc__inter_process_communication_.md), etc.), design decisions. Links back to relevant code components.
    *   **API Documentation:** If the extension exposes APIs for other extensions.

9.  **`.roo/` Directory:** Project-specific rule files:
    *   **`rules/rules.md`:** Defines project-specific coding standards (beyond linters), architectural principles, testing requirements (e.g., minimum coverage goals), UI styling guidelines ([Chapter 48: Configuration Rules](48_configuration_rules.md)).
    *   **`rules-translate/`:** Localization guidelines ([Chapter 50: Localization System (i18n)](50_localization_system__i18n_.md)). Includes general rules (`001-general-rules.md`) and language-specific instructions/glossaries (`instructions-<lang>.md`).

## Using These Files

*   **Users:** Start with `README.md`. Refer to `docs/` for details. Check `CHANGELOG.md` for updates. Use `ISSUE_TEMPLATE`s for reporting issues.
*   **Contributors:** Start with `README.md`, then `CONTRIBUTING.md`. Refer to `docs/architecture/`, `CODE_OF_CONDUCT.md`, and `.roo/rules/rules.md`. Use GitHub templates for issues/PRs. Check CI results from `workflows/`.
*   **Translators:** Focus on files within `.roo/rules-translate/` for specific instructions and terminology. Refer to `CONTRIBUTING.md` for general workflow if applicable.
*   **Maintainers:** Use all files to manage the project, ensure consistency, onboard contributors, review submissions, and uphold community standards.
*   **VS Code Marketplace:** Reads `README.md`, `CHANGELOG.md`, `LICENSE`.

## Code Walkthrough

These files are primarily Markdown, YAML, or JSON. Their value lies in their content and structure.

### `.roo/rules/rules.md` (Example Structure)

*(Based on Chapter 48 description)*

```markdown
# Roo-Code: Code Quality & Project Rules

Refer to [CONTRIBUTING.md](../../CONTRIBUTING.md) for setup and PR process.

## 1. General Principles
*   Clarity, Simplicity, Consistency.
*   Follow TypeScript best practices.

## 2. Testing ([Chapter 57](57_testing_framework.md))
*   New features/fixes require unit/integration tests.
*   Use Jest and React Testing Library. Mock dependencies appropriately.
*   Aim for >80% statement coverage (measured by `pnpm test:coverage`) for new code.
*   All tests (`pnpm test`) must pass in CI.

## 3. TypeScript Usage
*   Enable and adhere to `"strict": true`.
*   Avoid `any`. Use `unknown` or specific types. Infer types from Zod schemas ([Chapter 40](40_schemas__zod_.md)).

## 4. UI Components (WebView)
*   Use themed primitives ([Chapter 33](33_shadcn_ui_primitives__webview_.md), [Chapter 32](32_vscode_webview_ui_toolkit_wrappers.md)).
*   Use Tailwind classes mapped to VS Code variables for styling. Avoid inline styles for themable properties.
*   Ensure accessibility.

## 5. State Management
*   Host: Use `ContextProxy` ([Chapter 11](11_contextproxy.md)), define schemas/keys ([Chapter 40](40_schemas__zod_.md)).
*   WebView: Use `ExtensionStateContext` ([Chapter 12](12_extensionstatecontext.md)) for global state, local state (`useState`) or specific context (`ChatProvider`) otherwise.

## 6. Error Handling & Logging
*   Use shared `logger` (`src/utils/logging.ts`). Provide context (`ctx`).
*   Handle errors gracefully, provide user feedback.

## 7. Dependencies
*   Minimize runtime dependencies. Justify additions. Use `devDependencies` for build/test tools.
```

### `.roo/rules-translate/` Files (Example Structure)

*(Based on Chapter 50 description)*

*   **`001-general-rules.md`:** Workflow (sync with `en` first), QA script usage (`find-missing-translations.js`), tone (informal), non-translatable terms ("token", "API", "Roo-Code"), placeholder syntax (`{{var}}`), `Trans` component usage (`<1>...</1>`).
*   **`instructions-de.md`:** Rule: Use "du". Glossary: English -> German.
*   **`instructions-zh-cn.md`:** Glossary: English -> Simplified Chinese. Formatting rules. UI element constraints.
*   **`instructions-zh-tw.md`:** Glossary differences from Simplified Chinese. Punctuation rules.

## Internal Implementation

These are primarily static documentation or configuration files used by humans or external tools.

*   **Markdown:** Rendered by GitHub, VS Code Marketplace, documentation generators. Read by humans. Potentially parsed by scripts or AI for context.
*   **License:** Legally binding text.
*   **YAML (GitHub Actions):** Parsed and executed by the GitHub Actions runner service.
*   **JSON (NLS):** Parsed by VS Code host during extension loading.

## Modification Guidance

Modifications involve updating content to reflect project changes, improve clarity, or add new guidelines.

1.  **Updating `README.md` / `docs/`:** Add sections for new features, update screenshots, correct installation/usage instructions, fix broken links.
2.  **Updating `CHANGELOG.md`:** Add entries for new releases following the established format.
3.  **Modifying `CONTRIBUTING.md`:** Update setup, build ([Chapter 56](56_build_system.md)), or test ([Chapter 57](57_testing_framework.md)) instructions if the process changes. Refine contribution workflow or coding standards (linking to `rules.md`).
4.  **Updating Code Quality Rules (`.roo/rules/rules.md`):** Add new rules (e.g., specific async patterns, performance guidance). Update testing requirements. Clarify existing rules. Ensure alignment with linter/formatter configurations.
5.  **Updating Translation Guidelines (`.roo/rules-translate/`):** Add new globally non-translatable terms. Update language-specific glossaries or rules based on feedback or new UI elements. Ensure QA script instructions are correct.

**Best Practices:**

*   **Keep Updated:** Essential for all documentation. Outdated docs are harmful. Assign ownership or review periodically.
*   **Clarity & Conciseness:** Write for the intended audience. Use clear language, formatting, and examples.
*   **Standard Formats:** Follow conventions (Keep a Changelog, Contributor Covenant). Use standard GitHub file locations (`.github/`, root files).
*   **Automation:** Use CI (`workflows/`) to enforce checks mentioned in guidelines. Use GitHub templates.
*   **Discoverability:** Link related documents (e.g., README links to CONTRIBUTING and docs/, CONTRIBUTING links to rules.md).
*   **Accuracy:** Ensure technical instructions (build, test, setup) are correct and tested.

**Potential Pitfalls:**

*   **Outdated Documentation:** The biggest risk.
*   **Inaccurate Instructions:** Build/setup steps that fail.
*   **Missing Files:** Forgetting LICENSE, CHANGELOG, etc.
*   **Broken Links.**
*   **Unclear Contribution Process:** Deters contributors.
*   **Inconsistent Rules:** Contradictions between `CONTRIBUTING.md`, `rules.md`, and actual practice/linting.
*   **Guidelines Ignored:** Rules exist but aren't followed or enforced.

## Conclusion

Documentation and Community Files are the essential scaffolding that supports the Roo-Code project's development, usage, and community interaction. Files like `README.md`, `CONTRIBUTING.md`, `CHANGELOG.md`, `LICENSE`, and detailed architecture documents (like this tutorial series) provide crucial information for users and developers. Standard community files (`CODE_OF_CONDUCT.md`, `SECURITY.md`) and GitHub templates (`ISSUE_TEMPLATE`, `PULL_REQUEST_TEMPLATE`) facilitate healthy collaboration. Project-specific guidelines, such as code quality rules in `.roo/rules/rules.md` and localization instructions in `.roo/rules-translate/`, ensure consistency and quality aligned with the project's specific goals. Maintaining these resources accurately is vital for the project's success, usability, and collaborative potential.

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

