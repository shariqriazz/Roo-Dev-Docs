# Chapter 27: Bedrock Model Identification

Continuing from [Chapter 26: Message Transformation](26_message_transformation.md), where we focused on adapting conversation history formats for different LLM providers, this chapter specifically addresses a unique challenge within the AWS Bedrock ecosystem: accurately identifying the target model, especially when using advanced features like custom ARNs or prompt routers.

## Motivation: The Complexity of Bedrock Endpoints

AWS Bedrock offers various ways to invoke models: standard foundation model IDs, provisioned throughput endpoints, custom models, inference profiles, and prompt routers. Users might provide a simple model ID (e.g., `anthropic.claude-3-haiku-20240307-v1:0`) or a complex Amazon Resource Name (ARN) representing one of these other resource types (e.g., `arn:aws:bedrock:us-west-2:123456789012:provisioned-model/my-claude-provisioned`).

Furthermore, Bedrock allows cross-region inference, sometimes indicated by prefixes in model IDs within ARNs (e.g., `us.anthropic.claude...` when called from `us-west-2` but targeting `us-east-1`). A prompt router might even select a *different* underlying model than the one initially targeted, revealing the actual model used only within the API response stream.

This complexity poses a challenge for Roo-Code:
1.  **API Calls:** The correct identifier (model ID or full ARN) must be used when calling the Bedrock API (`InvokeModel`, `Converse`, `ConverseStream`).
2.  **Cost Calculation & Context Limits:** To accurately calculate costs ([Chapter 29: Cost Calculation Utilities](29_cost_calculation_utilities.md)) and manage context windows ([Chapter 23: Sliding Window Context Management](23_sliding_window_context_management.md)), Roo-Code needs the *underlying base model's* information (pricing, context size), even if the user invoked a provisioned model ARN or a prompt router.
3.  **Dynamic Behavior:** When a prompt router selects a model, Roo-Code must dynamically update its understanding of the model being used *for cost/token counting purposes* based on information (`invokedModelId`) returned mid-stream.

The Bedrock Model Identification logic within the `AwsBedrockHandler` ([Chapter 5: ApiHandler](05_apihandler.md)) is designed to parse ARNs, map different resource types to the correct identifiers for API calls versus cost calculation, handle region prefixes, and dynamically adapt based on stream information.

**Central Use Case:** A user configures Roo-Code to use a Bedrock provisioned throughput ARN (`arn:...:provisioned-model/my-claude-provisioned`). When Roo-Code makes an API call:
*   It must use the full ARN in the `modelId` parameter of the Bedrock SDK command.
*   However, for calculating the cost of the request/response, it needs to know that the underlying model is, for instance, `anthropic.claude-3-sonnet...` and use *Sonnet's* pricing rates.
*   The model identification logic needs to parse the ARN, identify it as a provisioned model, extract the underlying base model ID (if possible from naming conventions, or determine it later), and store both the ARN (for API calls) and the base model info (for cost/limits) appropriately. If a prompt router ARN is used, it must initially use default info and then update the cost/limit info based on the `invokedModelId` returned in the stream.

## Key Concepts

1.  **Bedrock ARNs:** ARNs identify specific AWS resources. Bedrock uses different resource types in ARNs:
    *   `foundation-model/model-id`: Standard AWS-provided models.
    *   `provisioned-model/model-name`: User-purchased provisioned throughput.
    *   `custom-model/model-name`: User-fine-tuned models.
    *   `inference-profile/model-id`: Often used for cross-region inference targets.
    *   `prompt-router/router-name`: A router that selects an underlying model.

2.  **`AwsBedrockHandler` (`src/api/providers/bedrock.ts`):** The specific `ApiHandler` implementation for Bedrock. It contains the logic for identifying models based on user configuration.

3.  **Configuration Options:** Users configure Bedrock via `ProviderSettings` ([Chapter 9: ProviderSettingsManager](09_providersettingsmanager.md)), primarily using:
    *   `apiModelId`: Can be a simple ID or potentially part of an ARN if `awsCustomArn` isn't used.
    *   `awsRegion`: The primary AWS region for API calls.
    *   `awsCustomArn` (Optional): If provided, this ARN takes precedence for identifying the target endpoint.
    *   `awsUseCrossRegionInference` (Optional): Flag to enable cross-region inference (might modify the effective model ID used).

4.  **ARN Parsing (`parseArn`):** A helper function within `AwsBedrockHandler` that takes an ARN string and attempts to extract:
    *   `region`: The region specified in the ARN.
    *   `modelType`: The resource type (e.g., `foundation-model`, `provisioned-model`).
    *   `modelId`: The resource identifier (e.g., `anthropic.claude...` or `my-custom-model`).
    *   `isValid`: Whether the ARN matches the expected format.
    *   `crossRegionInference`: Whether a recognized cross-region prefix was detected in the `modelId` part.
    *   It uses `parseBaseModelId` internally.

5.  **Region Prefix Parsing (`parseBaseModelId`):** A helper function that checks if a model ID string starts with a known region prefix (e.g., `us.`, `eu.`, `apne3.`). If found, it strips the prefix to get the base model ID and signals whether it indicated cross-region inference. Uses the `AMAZON_BEDROCK_REGION_INFO` map (`src/shared/aws_regions.ts`).

6.  **Model Info Lookup (`getModelById`):** A function that takes a base model ID (e.g., `anthropic.claude-3...`) and looks it up in the statically defined `bedrockModels` map (`src/shared/api.ts`) to retrieve its associated `ModelInfo` (context window, pricing, capabilities). Returns a default model's info if the ID isn't found. Crucially, it returns a deep copy to allow modifications.

7.  **Determining Effective IDs (`getModel`):** This core method in `AwsBedrockHandler` orchestrates the identification logic based on the configuration:
    *   Checks if `awsCustomArn` is provided. If yes, it uses `parseArn`.
    *   Looks up the base `ModelInfo` using `getModelById`.
    *   **Crucially**, it determines the `id` to be stored in `costModelConfig` (used for API calls *and* as the identifier in results):
        *   If the type is `foundation-model`, it uses the **base model ID**.
        *   If the type is *not* `foundation-model` (e.g., provisioned, custom, router, profile), it uses the **full ARN** as the ID.
    *   If `awsCustomArn` is *not* provided but `awsUseCrossRegionInference` is true, it prepends the appropriate region prefix to the `apiModelId` based on the configured `awsRegion`.
    *   Stores the resolved `{ id, info }` in `this.costModelConfig`.

8.  **`costModelConfig`:** A property within `AwsBedrockHandler` storing the resolved `{ id, info }`.
    *   `costModelConfig.id`: The identifier used in the `modelId` field when calling the Bedrock API. This is either the base model ID (for foundation models) or the full ARN (for other resource types).
    *   `costModelConfig.info`: The `ModelInfo` (context window, pricing, etc.) associated with the *underlying base model*, used for cost calculation and context management.

9.  **Dynamic Update (via Stream Trace):** The `createMessage` method processes the Bedrock Converse API stream. If it encounters a `trace` event containing `promptRouter.invokedModelId`:
    *   It calls `parseArn` on the `invokedModelId`.
    *   It calls `getModelById` using the base model ID extracted from the `invokedModelId` ARN.
    *   It updates `this.costModelConfig.info` with the `ModelInfo` of the *actually invoked* model.
    *   It **keeps** `this.costModelConfig.id` as the original prompt router ARN to ensure subsequent requests still go to the router.

## Using Bedrock Model Identification (Examples)

The model identification logic runs primarily during the `AwsBedrockHandler` constructor and potentially during the `createMessage` stream processing. Let's trace the examples from the provided context (`cline_docs/bedrock/model-identification.md`).

**(Example 1: Standard Model)**

*   **Input:** `apiModelId: "anthropic.claude-3-5-sonnet...", awsRegion: "us-east-1"`. No `awsCustomArn`.
*   **`constructor` calls `getModel()`:**
    *   No ARN, no cross-region flag.
    *   `getModelById("anthropic.claude-3-5-sonnet...")` retrieves Sonnet's `ModelInfo`.
    *   `costModelConfig` becomes `{ id: "anthropic.claude-3-5-sonnet...", info: <Sonnet Info> }`.
*   **API Call:** Uses `"anthropic.claude-3-5-sonnet..."` as `modelId`.
*   **Cost Calc:** Uses Sonnet's pricing from `<Sonnet Info>`.

**(Example 2: Custom ARN for Foundation Model)**

*   **Input:** `apiModelId: "anthropic.claude-3-5-sonnet..."`, `awsRegion: "us-east-1"`, `awsCustomArn: "arn:...:foundation-model/anthropic.claude-3-5-sonnet..."`.
*   **`constructor` calls `parseArn(...)`:** Returns `{ modelType: "foundation-model", modelId: "anthropic.claude-3-5-sonnet...", ... }`.
*   **`constructor` calls `getModel()`:**
    *   `getModelById("anthropic.claude-3-5-sonnet...")` retrieves Sonnet's `ModelInfo`.
    *   Since `modelType` is `foundation-model`, the final `id` remains the base model ID.
    *   `costModelConfig` becomes `{ id: "anthropic.claude-3-5-sonnet...", info: <Sonnet Info> }`.
*   **API Call:** Uses `"anthropic.claude-3-5-sonnet..."` as `modelId`.
*   **Cost Calc:** Uses Sonnet's pricing from `<Sonnet Info>`.

**(Example 3: Custom ARN for Prompt Router)**

*   **Input:** `awsRegion: "us-west-2"`, `awsCustomArn: "arn:...:prompt-router/my-router"`.
*   **`constructor` calls `parseArn(...)`:** Returns `{ modelType: "prompt-router", modelId: "my-router", ... }`.
*   **`constructor` calls `getModel()`:**
    *   `getModelById("my-router")` doesn't find it, returns default info (using `bedrockDefaultPromptRouterModelId`).
    *   Since `modelType` is *not* `foundation-model`, the final `id` is set to the **full ARN**.
    *   `costModelConfig` becomes `{ id: "arn:...:prompt-router/my-router", info: <Default Info> }`.
*   **API Call:** Uses `"arn:...:prompt-router/my-router"` as `modelId`.
*   **Cost Calc (Initial):** Uses default pricing from `<Default Info>`. *(Will be updated by stream)*.

**(Example 5: Prompt Router with `invokedModelId` in Stream)**

*   **Initial State:** `costModelConfig` is `{ id: "arn:...:prompt-router/my-router", info: <Default Info> }`.
*   **`createMessage` receives stream event:** `{ trace: { promptRouter: { invokedModelId: "arn:...:inference-profile/anthropic.claude-3-5-sonnet..." } } }`.
*   **Inside `createMessage`:**
    *   Calls `parseArn` on the `invokedModelId`, gets `{ modelType: "inference-profile", modelId: "anthropic.claude-3-5-sonnet..." }`.
    *   Calls `getModelById("anthropic.claude-3-5-sonnet...")`, gets actual Sonnet `ModelInfo`.
    *   Sets `invokedModel.id = modelConfig.id` (preserves router ARN).
    *   Updates `this.costModelConfig` to `{ id: "arn:...:prompt-router/my-router", info: <Sonnet Info> }`.
*   **Cost Calc (Updated):** Now uses correct Sonnet pricing from the updated `costModelConfig.info`.
*   **Subsequent API Calls:** Still use `"arn:...:prompt-router/my-router"` as `modelId` because `costModelConfig.id` was preserved.

**(Example 6: Cross-Region ARN with Region Prefix)**

*   **Input:** `awsRegion: "us-east-1"`, `awsCustomArn: "arn:...us-west-2...:inference-profile/us.anthropic.claude-3-5-sonnet..."`.
*   **`constructor` calls `parseArn(...)`:**
    *   Detects region `us-west-2` from ARN.
    *   Calls `parseBaseModelId("us.anthropic...")`, which strips `"us."` and returns `"anthropic.claude-3-5-sonnet..."`.
    *   Recognizes `"us."` as multi-region prefix, sets `crossRegionInference: true`.
    *   Returns `{ modelType: "inference-profile", modelId: "anthropic.claude-3-5-sonnet...", region: "us-west-2", crossRegionInference: true, ... }`.
*   **`constructor` updates options:** Sets `this.options.awsRegion` to `"us-west-2"`.
*   **`constructor` calls `getModel()`:**
    *   `getModelById("anthropic.claude-3-5-sonnet...")` retrieves Sonnet's `ModelInfo`.
    *   Since `modelType` is *not* `foundation-model`, the final `id` is set to the **full original ARN**.
    *   `costModelConfig` becomes `{ id: "arn:...us-west-2...:inference-profile/us.anthropic...", info: <Sonnet Info> }`.
*   **API Call:** Uses the full ARN `"arn:...us-west-2...:inference-profile/us.anthropic..."` as `modelId`. The SDK client uses the updated region `"us-west-2"`.
*   **Cost Calc:** Uses Sonnet's pricing from `<Sonnet Info>`.

## Code Walkthrough

### Core Handler Logic (`src/api/providers/bedrock.ts`)

```typescript
// --- File: src/api/providers/bedrock.ts ---
// ... (Imports: Bedrock SDK, Schemas, Shared API models, Regions, Transformer) ...

export class AwsBedrockHandler extends BaseProvider implements SingleCompletionHandler {
	// ... (client, options, constructor setup) ...

	private costModelConfig: { id: BedrockModelId | string; info: SharedModelInfo } = {
		id: "", info: { /* default empty info */ },
	};
	private arnInfo: ReturnType<typeof this.parseArn> | undefined;

	constructor(options: ProviderSettings) {
		super();
		this.options = { ...options }; // Clone options
		let region = this.options.awsRegion;

		// --- ARN Processing ---
		if (this.options.awsCustomArn) {
			this.arnInfo = this.parseArn(this.options.awsCustomArn, region);
			if (!this.arnInfo.isValid) { /* Throw error */ }
			// Use region from ARN if it differs from user selection
			if (this.arnInfo.region && this.arnInfo.region !== this.options.awsRegion) {
				logger.info(/* Mismatch message */);
				this.options.awsRegion = this.arnInfo.region;
			}
			// Use modelId extracted from ARN
			this.options.apiModelId = this.arnInfo.modelId;
			// Apply cross-region flag if detected in ARN
			if (this.arnInfo.crossRegionInference) this.options.awsUseCrossRegionInference = true;
		}

		this.options.modelTemperature ??= BEDROCK_DEFAULT_TEMPERATURE;
		// --- Set Initial costModelConfig ---
		this.costModelConfig = this.getModel();

		// --- Initialize SDK Client ---
		// ... (Client config logic using credentials, updated region) ...
		this.client = new BedrockRuntimeClient(clientConfig);
	}

	// --- ARN Parsing Logic ---
	private parseArn(arn: string, region?: string): {
        isValid: boolean; region?: string; modelType?: string; modelId?: string;
        errorMessage?: string; crossRegionInference: boolean;
    } {
		const arnRegex = /^arn:aws:(?:bedrock|sagemaker):([^:]+):([^:]*):(?:([^\/]+)\/([\w\.\-:]+)|([^\/]+))$/;
		let match = arn.match(arnRegex);

		if (match && match[1] && match[3] && match[4]) {
			const result = { isValid: true, crossRegionInference: false } as ReturnType<typeof this.parseArn>;
			result.modelType = match[3];
			const originalModelId = match[4];
			result.modelId = this.parseBaseModelId(originalModelId); // Strip prefix if exists
			const arnRegion = match[1];
			result.region = arnRegion;

			// Detect cross-region inference based on prefix presence/type
			if (originalModelId && result.modelId !== originalModelId) {
				let prefix = originalModelId.replace(result.modelId, "");
				result.crossRegionInference = AwsBedrockHandler.prefixIsMultiRegion(prefix);
			}

			// Check for region mismatch
			if (region && arnRegion !== region) {
				result.errorMessage = `Region mismatch...`; // ARN region overrides selected region
				result.region = arnRegion;
			}
			return result;
		}
		// Invalid format
		return { isValid: false, crossRegionInference: false, errorMessage: "Invalid ARN format..." };
	}

	// --- Region Prefix Handling ---
	private parseBaseModelId(modelId: string): string {
		if (!modelId) return modelId;
		const knownRegionPrefixes = AwsBedrockHandler.getPrefixList();
		const matchedPrefix = knownRegionPrefixes.find((prefix) => modelId.startsWith(prefix));
		if (matchedPrefix) {
			return modelId.substring(matchedPrefix.length); // Strip known prefix
		}
		// Handle generic prefix pattern as fallback
		const genericPrefixMatch = modelId.match(/^([^.:]+)\.(.+\..+)$/);
		if (genericPrefixMatch) { return genericPrefixMatch[2]; }
		return modelId; // No prefix found
	}

	private static getPrefixList(): string[] { return Object.keys(AMAZON_BEDROCK_REGION_INFO); }
	private static prefixIsMultiRegion(arnPrefix: string): boolean {
		return AMAZON_BEDROCK_REGION_INFO[arnPrefix]?.multiRegion ?? false;
	}
	private static getPrefixForRegion(region: string): string | undefined {
		// Find prefix where pattern matches start of region
		return Object.entries(AMAZON_BEDROCK_REGION_INFO)
            .find(([_, info]) => info.pattern && region.startsWith(info.pattern))?.[0];
	}

	// --- Model Info Lookup ---
	getModelById(modelId: string, modelType?: string): { id: BedrockModelId | string; info: SharedModelInfo } {
		const baseModelId = this.parseBaseModelId(modelId) as BedrockModelId;
		let model: { id: string; info: SharedModelInfo };

		if (baseModelId in bedrockModels) {
			model = { id: baseModelId, info: JSON.parse(JSON.stringify(bedrockModels[baseModelId])) };
		} else if (modelType?.includes("router")) { // Default for router if baseModelId not found
			model = { id: bedrockDefaultPromptRouterModelId, info: JSON.parse(JSON.stringify(bedrockModels[bedrockDefaultPromptRouterModelId])) };
		} else { // General default
			model = { id: bedrockDefaultModelId, info: JSON.parse(JSON.stringify(bedrockModels[bedrockDefaultModelId])) };
		}
		// Override maxTokens if set by user
		if (this.options.modelMaxTokens && this.options.modelMaxTokens > 0) {
			model.info.maxTokens = this.options.modelMaxTokens;
		}
		return model;
	}

	// --- Determining Effective ID and Info ---
	override getModel(): { id: BedrockModelId | string; info: SharedModelInfo } {
		// Return cached config if already resolved
		if (this.costModelConfig?.id?.trim().length > 0) {
			return this.costModelConfig;
		}

		let modelConfig: ReturnType<typeof this.getModelById>;
		let effectiveId: string;

		if (this.options.awsCustomArn && this.arnInfo) { // ARN Path
			modelConfig = this.getModelById(this.arnInfo.modelId!, this.arnInfo.modelType);
			// Use ARN as ID unless it's a foundation model
			effectiveId = (this.arnInfo.modelType === "foundation-model")
				? modelConfig.id
				: this.options.awsCustomArn;
		} else { // No ARN Path
			modelConfig = this.getModelById(this.options.apiModelId!);
			effectiveId = modelConfig.id; // Start with base ID
			// Apply cross-region prefix if flag is set
			if (this.options.awsUseCrossRegionInference) {
				const prefix = AwsBedrockHandler.getPrefixForRegion(this.options.awsRegion!);
				effectiveId = prefix ? `${prefix}${effectiveId}` : effectiveId;
			}
		}
		// Ensure maxTokens is set
		modelConfig.info.maxTokens ??= BEDROCK_MAX_TOKENS;

		// Return the final structure for costModelConfig
		return { id: effectiveId, info: modelConfig.info };
	}

	// --- Dynamic Update in createMessage ---
	override async *createMessage(systemPrompt: string, messages: Anthropic.Messages.MessageParam[]): ApiStream {
        let modelConfig = this.getModel(); // Get initial config
        // ... (Prepare payload using modelConfig.id) ...

        // ... (Send command, start processing stream) ...
        for await (const chunk of response.stream) {
            // ... (Parse chunk into streamEvent) ...

            // --- Handle Router Trace ---
            if (streamEvent?.trace?.promptRouter?.invokedModelId) {
                try {
                    const invokedArn = streamEvent.trace.promptRouter.invokedModelId;
                    let invokedArnInfo = this.parseArn(invokedArn);
                    // Get ModelInfo for the *actually invoked* model
                    let invokedModel = this.getModelById(invokedArnInfo.modelId!, invokedArnInfo.modelType);
                    if (invokedModel) {
                        // --- CRITICAL UPDATE ---
                        // Keep the original ID (router ARN) for subsequent API calls
                        invokedModel.id = modelConfig.id;
                        // Update the cached config with the *invoked* model's info for cost/context
                        this.costModelConfig = invokedModel;
                        // Update local modelConfig variable for this request's cost calculation
                        modelConfig = invokedModel;
                        logger.info(`Bedrock Router invoked model: ${invokedArnInfo.modelId}. Updated cost/context info.`, { ctx: "bedrock" });
                    }
                    // ... (Yield usage data from router trace) ...
                } catch (error) { /* Log error */ }
                continue; // Skip rest of chunk processing for trace event
            }

            // ... (Handle metadata, content deltas, etc.) ...
             if (streamEvent.metadata?.usage) {
                // Use modelConfig.info (potentially updated by router trace) for cost calculation
                // const cost = calculateCost(usage, modelConfig.info);
                 yield { type: "usage", /* ..., totalCost: cost */ };
             }
             if (streamEvent.contentBlockDelta?.delta?.text) {
                 yield { type: "text", text: streamEvent.contentBlockDelta.delta.text };
             }
        }
        // ... (Cleanup) ...
    }
	// ... (Other methods: completePrompt, error handling, etc.) ...
}
```

**Explanation:**

1.  **Constructor:** Handles initial ARN parsing (`parseArn`, `parseBaseModelId`) if `awsCustomArn` is provided, potentially updating `options.awsRegion` and `options.apiModelId`. It then calls `getModel` to set the initial `this.costModelConfig`. Finally, it initializes the SDK client using the potentially updated region and credentials.
2.  **`parseArn` / `parseBaseModelId` / Region Helpers:** Implement the logic for dissecting ARNs, identifying and stripping region prefixes, and checking if prefixes imply cross-region use based on the `AMAZON_BEDROCK_REGION_INFO` map.
3.  **`getModelById`:** Looks up the base model ID in the static `bedrockModels` map, returning a deep copy or a default if not found. Applies `modelMaxTokens` override.
4.  **`getModel`:** This is the key logic determining the final `id` and `info` stored in `costModelConfig`. It differentiates based on whether an ARN was provided and its `modelType`. It also applies the cross-region prefix logic if the `awsUseCrossRegionInference` flag is set without an ARN.
5.  **`createMessage` (Update Logic):** Inside the stream processing loop, it specifically checks for the `trace.promptRouter.invokedModelId`. If found, it re-parses the *invoked* ARN, looks up the corresponding *invoked* model's info using `getModelById`, and updates `this.costModelConfig.info` while preserving the original `costModelConfig.id` (the router ARN). This ensures future cost/token calculations use the correct rates/limits, but future API calls still target the router.

### Region Info (`src/shared/aws_regions.ts`)

*(See full code in chapter context)*

*   **`AMAZON_BEDROCK_REGION_INFO`:** A map where keys are prefixes (including trailing `.`) and values are objects containing the canonical `regionId`, a human-readable `description`, an optional `pattern` for matching region strings, and a `multiRegion` flag.
*   **`AWS_REGIONS`:** An array derived from `AMAZON_BEDROCK_REGION_INFO`, providing unique region IDs suitable for UI dropdowns.

## Internal Implementation

The identification happens in two phases: initially in the constructor based on configuration, and potentially dynamically during stream processing if a router is used.

**Initialization Phase (Constructor):**

1.  Check if `options.awsCustomArn` exists.
2.  If Yes:
    *   Call `parseArn(arn, options.awsRegion)`.
    *   `parseArn` calls `parseBaseModelId(modelIdFromArn)` to strip prefix.
    *   `parseArn` checks if prefix indicates cross-region, checks for region mismatch.
    *   Store `arnInfo` and update `options.awsRegion`, `options.apiModelId`, `options.awsUseCrossRegionInference` based on `arnInfo`.
3.  Call `getModel()`.
4.  `getModel()` checks `arnInfo` (if ARN was used) or `options.apiModelId` and `options.awsUseCrossRegionInference` (if no ARN).
5.  It calls `getModelById()` using the determined base model ID to get the base `ModelInfo`.
6.  It determines the `effectiveId`: the full ARN if `arnInfo.modelType` is not `foundation-model`, otherwise the base model ID (or prefixed base ID if cross-region flag set without ARN).
7.  Set `this.costModelConfig = { id: effectiveId, info: <base ModelInfo> }`.
8.  Initialize `BedrockRuntimeClient` using `options.awsRegion` and credentials.

**Dynamic Update Phase (`createMessage` Stream):**

1.  Start processing stream chunks.
2.  Receive a chunk containing `trace.promptRouter.invokedModelId`.
3.  Extract `invokedModelId` ARN.
4.  Call `parseArn(invokedModelId)` to get `invokedArnInfo`.
5.  Call `getModelById(invokedArnInfo.modelId)` to get the `invokedModelInfo`.
6.  Update `this.costModelConfig.info = invokedModelInfo`. (Leave `this.costModelConfig.id` unchanged).
7.  Continue processing stream. Subsequent cost/token calculations within this `createMessage` call (and future calls if `costModelConfig` isn't reset) will use the updated `info`.

**Mermaid Diagrams:** (Included from provided context)

*   **Initialization Flow:** Shows `Constructor` -> `parseArn` -> `parseBaseModelId` -> `getModel` -> `getModelById` -> setting `costModelConfig`.
*   **Stream Processing Flow:** Shows `createMessage` -> `parseArn` (on `invokedModelId`) -> `getModelById` -> updating `costModelConfig`.

## Modification Guidance

Modifications might involve supporting new Bedrock resource types, adding new regions/prefixes, or changing the default model logic.

**Common Modifications:**

1.  **Adding a New AWS Region/Prefix:**
    *   **Region Map:** Add a new entry to `AMAZON_BEDROCK_REGION_INFO` in `src/shared/aws_regions.ts` with the correct prefix (e.g., `"eun2."`), `regionId`, `description`, and `multiRegion` status.
    *   **Regions Array:** The `AWS_REGIONS` array will update automatically.
    *   **Test:** Ensure `parseBaseModelId` correctly strips the new prefix and `getModel` correctly applies it if `awsUseCrossRegionInference` is set for that region. Update UI components displaying region lists if necessary.

2.  **Supporting a New Bedrock Resource Type via ARN:**
    *   **`parseArn`:** Update the `arnRegex` if the new resource type has a significantly different ARN structure (unlikely for standard Bedrock resources). Ensure the `modelType` (capture group 3) is correctly extracted.
    *   **`getModel`:** Modify the logic that determines the `effectiveId`. Decide if the full ARN or the base model ID should be used for API calls when this new `modelType` is encountered. Update the condition `if (this.arnInfo.modelType !== "foundation-model")`.
    *   **`getModelById`:** If the new resource type consistently maps to a *specific* known base model, you might add logic here to return that model's info by default when the `modelType` matches, even if the `modelId` part of the ARN isn't in `bedrockModels`.
    *   **Test:** Test with ARNs for the new resource type.

3.  **Changing Default Model Lookups:**
    *   **Constants:** Modify `bedrockDefaultModelId` or `bedrockDefaultPromptRouterModelId` in `src/shared/api.ts`.
    *   **`getModelById`:** Change which default ID is used in the fallback logic if a specific ID isn't found in `bedrockModels`.

**Best Practices:**

*   **Centralize Region Info:** Keep all region prefix and ID mappings within `src/shared/aws_regions.ts`.
*   **Robust ARN Parsing:** Ensure `parseArn` correctly handles variations in ARN structure and extracts the necessary components. Use specific regex capture groups.
*   **Clear ID Distinction:** Maintain the clear distinction in `getModel`: use the full ARN as the effective `id` for API calls involving provisioned/custom/router/profile resources, but use the base model ID for foundation models. Use the base model `info` for cost/context calculations (updated dynamically for routers).
*   **Deep Copy:** Always deep copy model info retrieved from the static `bedrockModels` map (as done using `JSON.parse(JSON.stringify(...))` in `getModelById`) before modifying it, especially when overriding `maxTokens` or preparing it for `costModelConfig`.
*   **Error Handling:** Provide clear error messages if ARN parsing fails (`INVALID_ARN_FORMAT`) or if region mismatches occur.

**Potential Pitfalls:**

*   **Incorrect ARN Regex:** The regex might fail to match valid ARNs if AWS introduces new formats or if the regex isn't robust enough for edge cases (e.g., unusual characters in resource names).
*   **Outdated Region/Prefix Map:** If AWS adds new regions or changes prefix conventions, the `AMAZON_BEDROCK_REGION_INFO` map needs updating.
*   **Missing Base Model Info:** If `getModelById` cannot find info for a base model ID (e.g., a new model not yet added to `bedrockModels`), it falls back to defaults. This might lead to incorrect cost calculations or context limit estimations until the `bedrockModels` map is updated.
*   **Router Update Timing:** The `costModelConfig.info` is only updated *after* the first stream chunk containing the `invokedModelId` arrives. Cost/token calculations for the *initial* part of the response might still use the default router info if the trace event doesn't arrive immediately.
*   **Cross-Region Complexity:** Incorrectly setting or detecting the `crossRegionInference` flag, or failing to apply the correct prefix in `getModel`, could lead to API errors if the Bedrock client is configured for a different region than the target model ID implies.

## Conclusion

Bedrock Model Identification in `AwsBedrockHandler` is a critical piece of logic that bridges the gap between user configuration (model IDs, ARNs, regions) and the requirements of both the Bedrock API and Roo-Code's internal mechanisms (cost calculation, context management). By carefully parsing ARNs, handling region prefixes, distinguishing between API call identifiers and cost calculation models, and dynamically updating based on router responses, the handler ensures accurate and flexible interaction with the diverse endpoints offered by AWS Bedrock.

Following this deep dive into Bedrock's specifics, we will examine another provider-specific challenge: caching strategies, particularly relevant for Bedrock's potential prompt caching features. The next chapter discusses [Chapter 28: Bedrock Cache Strategy](28_bedrock_cache_strategy.md).

