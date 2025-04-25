# Chapter 59: Cache Strategy (Bedrock)

This chapter number duplicates the content intended for [Chapter 28: Bedrock Cache Strategy](28_bedrock_cache_strategy.md). There appears to be a mix-up in the chapter list provided in the initial request.

To avoid redundancy and maintain the correct tutorial structure, please refer to **[Chapter 28: Bedrock Cache Strategy](28_bedrock_cache_strategy.md)** for the detailed explanation of how Roo-Code implements strategies to leverage AWS Bedrock's prompt caching feature.

**[Chapter 28: Bedrock Cache Strategy](28_bedrock_cache_strategy.md)** covers:

*   The motivation for using Bedrock Prompt Caching (reducing latency and cost).
*   Key concepts like cache points (`<cache_point/>`), relevant `ModelInfo` properties (`supportsPromptCache`, `maxCachePoints`, `minTokensPerCachePoint`, `cachableFields`), the `BaseStrategy` interface, and the `MultiPointStrategy` implementation.
*   How the strategy determines cache point placement, including handling previous placements (N-1 rule) and token thresholds.
*   How `AwsBedrockHandler` integrates the strategy within its `convertToBedrockConverseMessages` method.
*   Code walkthroughs of `BaseStrategy`, `MultiPointStrategy`, relevant types, and the integration within `AwsBedrockHandler`.
*   Sequence diagrams illustrating the process.
*   Detailed examples of cache point placement in initial and growing conversations.
*   Guidance on modifying or extending the caching strategies, including best practices and potential pitfalls.

All information regarding the implementation and usage of Bedrock caching strategies within Roo-Code can be found in **[Chapter 28: Bedrock Cache Strategy](28_bedrock_cache_strategy.md)**. The code snippets provided in the prompt for Chapter 59 (`cline_docs/bedrock/bedrock-cache-strategy-documentation.md`, `src/api/providers/bedrock.ts`, `src/api/transform/cache-strategy/multi-point-strategy.ts`, `src/api/transform/cache-strategy/base-strategy.ts`, `src/api/transform/cache-strategy/types.ts`) are relevant to and discussed within Chapter 28.

