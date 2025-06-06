# Hybrid Execution Workflow

## WORKFLOW DESIGN

The Hybrid Execution Workflow is responsible for carrying out the content generation request based on the routing decision made by the Smart Routing Decision Workflow. It provides three primary pathways: Local, Hosted, and Hybrid, each optimized for different scenarios regarding privacy, cost, performance, and knowledge availability. This workflow ensures seamless integration between Open WebUI's local capabilities and Langbase's hosted services, handling response generation, citation management, and conflict resolution across sources.

**Input**: A content generation request and the `RoutingDecision` from the Smart Routing Decision Workflow.
**Process**:
1.  **Route Interpretation**: Interpret the `route` and `explanation` from the `RoutingDecision`.
2.  **Pathway Execution**: Execute the appropriate pathway (Local, Hosted, or Hybrid) based on the determined route.
3.  **Content Generation**: Generate the requested content using the selected pathway's resources (local models, Langbase Pipes, RAG systems).
4.  **Result Processing**: Handle citations, merge results from multiple sources (if hybrid), and resolve any conflicts.
**Output**: Optimized content generation response, potentially with citations and source attribution.

## COMPONENTS

### 1. Local Pathway Executor
**Responsibility**: Manages content generation entirely within the local Open WebUI environment.
**Logic**: 
*   Initiates RAG queries against the local vector store (ChromaDB/FAISS).
*   Feeds retrieved context to the local language model (via Open WebUI).
*   Generates responses and extracts citations from local sources.

### 2. Hosted Pathway Executor
**Responsibility**: Manages content generation using Langbase's hosted services.
**Logic**:
*   Configures and executes Langbase Pipes for complex AI workflows.
*   Utilizes Langbase Memory for cost-optimized vector storage and retrieval for simpler queries.
*   Processes responses received from Langbase, potentially translating them for local integration.

### 3. Hybrid Pathway Executor
**Responsibility**: Orchestrates content generation by combining local and hosted capabilities.
**Logic**:
*   **Local Knowledge Retrieval + Hosted Processing**: Performs RAG locally, then sends the retrieved context and query to Langbase for advanced model processing.
*   **Hosted Knowledge + Local Model Execution**: Queries Langbase Memory for knowledge, then brings the retrieved context back to Open WebUI for local model inference.
*   **Multi-Source Fusion and Conflict Resolution**: Develops strategies to merge responses from both local and hosted sources, resolve conflicting information, and maintain coherent citation management.

### 4. Citation Manager
**Responsibility**: Ensures proper attribution and citation of sources used in content generation.
**Logic**: 
*   Tracks the origin of retrieved information (local RAG, Langbase Memory, etc.).
*   Integrates citations into the final generated content.
*   Handles potential discrepancies in source attribution across hybrid pathways.

## IMPLEMENTATION

The Hybrid Execution Workflow will be implemented in TypeScript, likely within the `src/services/` directory, interacting with Open WebUI APIs locally and Langbase APIs for hosted operations.

```typescript
// src/services/hybridExecutionEngine.ts

import { UserRequest, RoutingDecision, WorkflowRoute } from '../types/routing';
import { ContentResponse } from '../types/content'; // Assuming a new type for content responses

// Placeholder for Open WebUI and Langbase API clients
interface OpenWebUIClient {
    queryRAG(query: string): Promise<string[]>; // Returns array of relevant documents/snippets
    generateResponse(prompt: string, context: string[]): Promise<{ text: string; citations: string[] }>;
}

interface LangbaseClient {
    runPipe(pipeConfig: any): Promise<any>;
    queryMemory(query: string): Promise<string[]>;
}

export class HybridExecutionEngine {
    private openWebUI: OpenWebUIClient;
    private langbase: LangbaseClient;

    constructor(openWebUI: OpenWebUIClient, langbase: LangbaseClient) {
        this.openWebUI = openWebUI;
        this.langbase = langbase;
    }

    private async executeLocalPathway(request: UserRequest): Promise<ContentResponse> {
        const context = await this.openWebUI.queryRAG(request.content);
        const { text, citations } = await this.openWebUI.generateResponse(request.content, context);
        return { generatedContent: text, citations, source: 'Local' };
    }

    private async executeHostedPathway(request: UserRequest): Promise<ContentResponse> {
        // Example: Use Langbase Memory for simple queries, Pipe for complex
        if (request.userPreferences?.prioritizeCost) { // Simplified check for demonstration
            const context = await this.langbase.queryMemory(request.content);
            // Assume Langbase client can also generate based on memory query
            const response = await this.langbase.runPipe({ type: 'generate_from_memory', query: request.content, context });
            return { generatedContent: response.text, citations: response.citations, source: 'Hosted (Langbase Memory)' };
        } else {
            const response = await this.langbase.runPipe({ type: 'content_generation', prompt: request.content });
            return { generatedContent: response.text, citations: response.citations, source: 'Hosted (Langbase Pipe)' };
        }
    }

    private async executeHybridPathway(request: UserRequest): Promise<ContentResponse> {
        // Example: Local RAG + Hosted Model
        const localContextPromise = this.openWebUI.queryRAG(request.content);
        const langbaseResponsePromise = this.langbase.runPipe({ type: 'advanced_model_inference', prompt: request.content });

        const [localContext, langbaseResponse] = await Promise.all([
            localContextPromise,
            langbaseResponsePromise
        ]);

        // Example fusion logic: Combine results, prioritize Langbase for generation, use local for citations
        const combinedContent = langbaseResponse.text; // Or a more sophisticated merging algorithm
        const combinedCitations = [...(langbaseResponse.citations || []), ...await this.openWebUI.generateResponse('', localContext).then(res => res.citations)];

        return { generatedContent: combinedContent, citations: combinedCitations, source: 'Hybrid' };
    }

    public async generateContent(request: UserRequest, decision: RoutingDecision): Promise<ContentResponse> {
        switch (decision.route) {
            case 'Local':
                return this.executeLocalPathway(request);
            case 'Hosted':
                return this.executeHostedPathway(request);
            case 'Hybrid':
                return this.executeHybridPathway(request);
            default:
                throw new Error('Invalid routing decision.');
        }
    }
}
```

```typescript
// src/types/content.ts
// This file would define the interfaces for content generation responses.

export interface ContentResponse {
    generatedContent: string;
    citations?: string[];
    source: 'Local' | 'Hosted (Langbase Memory)' | 'Hosted (Langbase Pipe)' | 'Hybrid';
    // Add other metadata like cost, performance metrics for monitoring
}
```

## ERROR HANDLING

**Pathway-Specific Error Handling:**
*   Each `executePathway` method (Local, Hosted, Hybrid) should include robust error handling for API failures, network issues, model inference errors, or data retrieval problems.
*   Implement retries with exponential backoff for transient errors in external API calls (e.g., Langbase).
*   Log detailed error messages, including the specific pathway that failed and the underlying cause.

**Inter-Pathway Fallbacks:**
*   If a primary pathway fails (e.g., Hosted Langbase API is down), the `HybridExecutionEngine` should attempt to gracefully fall back to an alternative pathway if feasible and aligned with the original routing decision's priorities.
*   For example, if the `HostedPathwayExecutor` fails, and the `RoutingDecision` indicated a medium cost sensitivity, it might attempt a `LocalPathwayExecutor` if local resources are available and privacy constraints allow.

**Data Consistency and Integrity:**
*   During hybrid content fusion, implement mechanisms to detect and resolve data inconsistencies or conflicts between local and hosted sources. This might involve flagging conflicts for human review or applying predefined resolution rules.

## OPTIMIZATION

**Resource Allocation:**
*   **Dynamic Resource Allocation (Local)**: Monitor local CPU/GPU and memory usage. If local resources are high, the `LocalPathwayExecutor` could signal this, influencing future `RoutingDecision` to favor hosted or hybrid approaches that offload computation.
*   **Langbase Cost Optimization**: The `HostedPathwayExecutor` will specifically leverage Langbase Memory for queries identified as simple or primarily knowledge retrieval, significantly reducing costs compared to full model inference via Pipes. Explore Langbase's batch processing capabilities for non-real-time requests.

**Parallel Execution within Pathways:**
*   Within the `HybridPathwayExecutor`, use `Promise.all` to execute local RAG queries and hosted model calls concurrently where dependencies allow. This minimizes overall latency for hybrid scenarios.

**Caching:**
*   **Local RAG Results**: Cache frequently accessed local RAG results to speed up subsequent queries.
*   **Langbase Responses**: Cache responses from Langbase for frequently requested content, especially for cost-sensitive pathways, to avoid redundant API calls.

**Streaming Responses:**
*   For real-time performance, implement streaming responses from both local and hosted models where supported, allowing the UI to display partial results as they become available.

## MONITORING

**Key Metrics:**
*   **Pathway Usage**: Track the frequency of use for Local, Hosted, and Hybrid pathways.
*   **Execution Latency**: Measure the end-to-end time taken for content generation for each pathway.
*   **Cost Tracking**: Monitor actual costs incurred by Langbase API calls, correlating with the `Cost Calculator`'s predictions.
*   **Error Rates**: Track errors specific to each pathway (e.g., Langbase API errors, local model failures).
*   **Citation Accuracy**: Metrics to assess the correctness and completeness of generated citations.
*   **Content Quality**: Potentially integrate automated evaluation metrics or user feedback mechanisms to assess the quality of generated content from each pathway.

**Alerting:**
*   Set up alerts for significant deviations in execution latency or unexpected cost increases for any pathway.
*   Alerts for high error rates or prolonged outages of specific pathways.

**Logging:**
*   Log the input request, the chosen pathway, the generated content (or a summary), and any relevant metrics (latency, cost, errors) for each content generation task. This provides a comprehensive audit trail and data for optimization.

**Dashboards:**
*   Visualize the distribution of pathway usage, average latencies, and actual costs over time. This helps in understanding the real-world performance and cost implications of the hybrid architecture.