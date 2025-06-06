# Smart Routing Decision Workflow

## WORKFLOW DESIGN

The Smart Routing Decision Workflow is the initial phase of the enhanced content generation pipeline. Its primary responsibility is to intelligently analyze user requests and determine the optimal pathway (local, hosted, or hybrid) for content generation based on various factors: privacy sensitivity, cost constraints, performance requirements, and knowledge availability. This workflow operates by evaluating these decision factors in parallel and using a weighted scoring system to select the most appropriate route, incorporating fallback mechanisms and user preference learning.

**Input**: User content generation request (chat message, document creation, analysis task).
**Process**:
1.  **Parallel Evaluation**: Simultaneously assess the input against privacy, knowledge, cost, and performance criteria.
2.  **Weighted Scoring**: Combine the assessments into a composite score to rank potential routes.
3.  **Route Selection**: Choose the best-scoring route, considering user preferences and fallback options.
4.  **Explanation Generation**: Provide a transparent explanation for the chosen route.
**Output**: Optimal route (Local, Hosted, Hybrid) and a detailed explanation for the routing decision.

## COMPONENTS

### 1. Privacy Classifier
**Responsibility**: Determines if the input content or the generated output will contain sensitive information requiring local processing.
**Logic**: Analyzes keywords, patterns, and potentially uses a small, local ML model for PII detection or content categorization.

### 2. Knowledge Assessor
**Responsibility**: Evaluates if the local RAG system (Open WebUI) has sufficient and relevant information to fulfill the request independently.
**Logic**: Performs a preliminary local RAG query. If retrieval confidence or coverage is low, it indicates a need for broader (potentially hosted) knowledge or a hybrid approach.

### 3. Cost Calculator
**Responsibility**: Estimates the potential cost for processing the request using hosted services (Langbase).
**Logic**: Based on estimated token usage, API calls, and Langbase pricing tiers for Pipes, Memory, and Workflows. Also considers the cost-effectiveness of Langbase Memory for vector storage for simple queries.

### 4. Performance Predictor
**Responsibility**: Estimates the response time for processing the request via local and hosted pathways.
**Logic**: Considers factors like local hardware capacity, network latency to hosted services, complexity of the query, and historical performance data.

### 5. Route Selector
**Responsibility**: Consolidates the outputs from the Privacy Classifier, Knowledge Assessor, Cost Calculator, and Performance Predictor to make the final routing decision.
**Logic**:
*   Applies a weighted scoring system to each potential route (Local, Hosted, Hybrid). Weights can be configured based on user priorities (e.g., privacy over cost, performance over knowledge).
*   Implements a decision matrix based on predefined thresholds and rules.
*   Manages fallback mechanisms (e.g., if a high-priority local route fails, try a hosted one).
*   Learns user preferences over time to adapt weighting.

## IMPLEMENTATION

The Smart Routing Decision Workflow will be implemented primarily in TypeScript, likely within the `src/` directory, potentially as a service or module that can be imported by the main application logic.

```typescript
// src/services/routingDecisionEngine.ts

import { UserRequest, RoutingDecision, WorkflowRoute } from '../types/routing';

// Placeholder for external services/models
interface PrivacyClassifierResult {
    isSensitive: boolean;
    confidence: number;
}

interface KnowledgeAssessorResult {
    localCoverageScore: number;
    sufficientLocalKnowledge: boolean;
}

interface CostCalculatorResult {
    estimatedLocalCost: number; // Placeholder for local compute cost, if tracked
    estimatedHostedCost: number;
    costSavingOpportunity: boolean; // e.g., if Langbase Memory is cheaper
}

interface PerformancePredictorResult {
    estimatedLocalTimeMs: number;
    estimatedHostedTimeMs: number;
}

// Configuration for weighting decision factors
interface RoutingConfig {
    weights: {
        privacy: number;
        knowledge: number;
        cost: number;
        performance: number;
    };
    thresholds: {
        sensitiveContent: number; // Confidence score for privacy
        localKnowledgeSufficiency: number; // Score for knowledge
        costSavingThreshold: number; // Max acceptable cost difference
        performanceToleranceMs: number; // Max acceptable time difference
    };
}

export class RoutingDecisionEngine {
    private config: RoutingConfig;

    constructor(config: RoutingConfig) {
        this.config = config;
    }

    // --- Component Implementations ---

    private async classifyPrivacy(request: UserRequest): Promise<PrivacyClassifierResult> {
        // Mock implementation: Replace with actual NLP/ML model
        const sensitiveKeywords = ['confidential', 'private', 'ssn', 'credit card', 'phi'];
        const isSensitive = sensitiveKeywords.some(keyword => request.content.toLowerCase().includes(keyword));
        return { isSensitive, confidence: isSensitive ? 0.9 : 0.1 };
    }

    private async assessKnowledge(request: UserRequest): Promise<KnowledgeAssessorResult> {
        // Mock implementation: Call local RAG for preliminary check
        // In reality, this would involve a call to the Open WebUI RAG system
        const localCoverageScore = Math.random(); // Simulate RAG confidence
        return { localCoverageScore, sufficientLocalKnowledge: localCoverageScore > this.config.thresholds.localKnowledgeSufficiency };
    }

    private async calculateCost(request: UserRequest): Promise<CostCalculatorResult> {
        // Mock implementation: Estimate based on content length
        const charCount = request.content.length;
        const estimatedHostedCost = charCount * 0.0001; // Example: $0.0001 per character
        const estimatedLocalCost = charCount * 0.00001; // Example: Local compute cost
        const costSavingOpportunity = estimatedHostedCost > 1 && estimatedHostedCost - estimatedLocalCost > this.config.thresholds.costSavingThreshold;
        return { estimatedLocalCost, estimatedHostedCost, costSavingOpportunity };
    }

    private async predictPerformance(request: UserRequest): Promise<PerformancePredictorResult> {
        // Mock implementation: Simulate varying response times
        const estimatedLocalTimeMs = Math.random() * 5000 + 1000; // 1-6 seconds
        const estimatedHostedTimeMs = Math.random() * 2000 + 500; // 0.5-2.5 seconds
        return { estimatedLocalTimeMs, estimatedHostedTimeMs };
    }

    // --- Route Selection Logic ---

    public async determineRoute(request: UserRequest): Promise<RoutingDecision> {
        const [privacyResult, knowledgeResult, costResult, performanceResult] = await Promise.all([
            this.classifyPrivacy(request),
            this.assessKnowledge(request),
            this.calculateCost(request),
            this.predictPerformance(request)
        ]);

        let localScore = 0;
        let hostedScore = 0;
        let hybridScore = 0;
        let explanation = '';

        // Apply weights based on results
        if (privacyResult.isSensitive && privacyResult.confidence > this.config.thresholds.sensitiveContent) {
            localScore += this.config.weights.privacy * 10; // Strong preference for local if sensitive
            explanation += 'Content classified as sensitive, prioritizing local processing. ';
        } else {
            hostedScore += this.config.weights.privacy * 5; // Less penalty for hosted if not sensitive
        }

        if (knowledgeResult.sufficientLocalKnowledge) {
            localScore += this.config.weights.knowledge * 7;
            explanation += 'Sufficient local knowledge found, favoring local RAG. ';
        } else {
            hostedScore += this.config.weights.knowledge * 3; // Hosted or hybrid if local knowledge is insufficient
            hybridScore += this.config.weights.knowledge * 5;
            explanation += 'Local knowledge insufficient, considering hosted or hybrid. ';
        }

        if (costResult.costSavingOpportunity) {
            hostedScore += this.config.weights.cost * 8; // Favor hosted for cost savings (e.g., Langbase Memory)
            explanation += `Cost optimization identified, favoring hosted (${costResult.estimatedHostedCost.toFixed(2)}$ vs local ${costResult.estimatedLocalCost.toFixed(2)}$). `;
        } else {
            localScore += this.config.weights.cost * 2;
        }

        if (performanceResult.estimatedLocalTimeMs < performanceResult.estimatedHostedTimeMs - this.config.thresholds.performanceToleranceMs) {
            localScore += this.config.weights.performance * 6;
            explanation += `Local pathway offers better performance (${performanceResult.estimatedLocalTimeMs}ms). `;
        } else if (performanceResult.estimatedHostedTimeMs < performanceResult.estimatedLocalTimeMs - this.config.thresholds.performanceToleranceMs) {
            hostedScore += this.config.weights.performance * 6;
            explanation += `Hosted pathway offers better performance (${performanceResult.estimatedHostedTimeMs}ms). `;
        } else {
            hybridScore += this.config.weights.performance * 4; // Similar performance, consider hybrid
            explanation += 'Performance similar, hybrid approach may balance. ';
        }

        // Determine the winning route
        let selectedRoute: WorkflowRoute = 'Hybrid';
        if (localScore >= hostedScore && localScore >= hybridScore) {
            selectedRoute = 'Local';
        } else if (hostedScore >= localScore && hostedScore >= hybridScore) {
            selectedRoute = 'Hosted';
        } else {
            selectedRoute = 'Hybrid';
        }
        
        // Final check for high privacy sensitivity to override if necessary
        if (privacyResult.isSensitive && privacyResult.confidence > this.config.thresholds.sensitiveContent) {
            selectedRoute = 'Local';
            explanation += 'Overriding to Local due to high privacy sensitivity. ';
        }

        return {
            route: selectedRoute,
            explanation: `Decision: ${selectedRoute}. ${explanation.trim()}`,
            details: { privacyResult, knowledgeResult, costResult, performanceResult }
        };
    }
}
```

```typescript
// src/types/routing.ts
// This file would define the interfaces used by the routing decision engine.

export type WorkflowRoute = 'Local' | 'Hosted' | 'Hybrid';

export interface UserRequest {
    id: string;
    content: string;
    // Add other relevant request parameters like user preferences
    userPreferences?: {
        prioritizePrivacy?: boolean;
        prioritizeCost?: boolean;
        prioritizePerformance?: boolean;
    };
}

export interface RoutingDecision {
    route: WorkflowRoute;
    explanation: string;
    details: {
        privacyResult: any; // Define more specific interfaces for these results
        knowledgeResult: any;
        costResult: any;
        performanceResult: any;
    };
}
```

## ERROR HANDLING

**Component-Level Error Handling:**
*   Each component (Privacy Classifier, Knowledge Assessor, etc.) should implement its own try-catch blocks to gracefully handle failures (e.g., external API timeouts, model inference errors, database connectivity issues).
*   Failures within a component should return a specific error or a default/fallback value that the `Route Selector` can interpret. For instance, if the Knowledge Assessor fails, the `Route Selector` might default to assuming insufficient local knowledge.

**Routing Decision Engine Error Handling:**
*   The `determineRoute` method should have a top-level error handler to catch any unhandled exceptions from the parallel `Promise.all` calls.
*   In case of critical failures (e.g., inability to assess privacy), the system should have a predefined safe fallback route (e.g., always default to local processing if privacy assessment fails to ensure data safety).
*   Logging of errors is crucial for debugging and monitoring.

**Fallback Mechanisms:**
*   The `Route Selector` should include explicit fallback logic. For example, if the preferred route (e.g., Hosted for cost) fails due to an external service outage, it should automatically switch to a viable alternative (e.g., Local or Hybrid).

## OPTIMIZATION

**Parallel Evaluation:**
*   Leverage `Promise.all` in TypeScript to concurrently execute the Privacy Classifier, Knowledge Assessor, Cost Calculator, and Performance Predictor. This significantly reduces the overall decision time.

**Caching:**
*   **Knowledge Assessor**: Cache results of common RAG queries or knowledge assessments to avoid redundant lookups.
*   **Cost Calculator/Performance Predictor**: Cache historical data for similar request types to provide faster estimates.

**User Preference Learning:**
*   Implement a feedback loop where user satisfaction with chosen routes (or explicit user preferences) can influence the weights in the `RoutingConfig` over time. This can be a simple adjustment based on positive/negative feedback or a more sophisticated machine learning approach.

**Langbase Pricing Optimization:**
*   **Smart Routing to Langbase Memory**: For queries identified as simple or requiring minimal knowledge, the `Cost Calculator` and `Route Selector` should prioritize routing to Langbase Memory, which is significantly cheaper for vector storage and retrieval. This is a key cost optimization pattern.
*   **Batching**: If multiple non-real-time requests arrive, consider batching them for hosted processing to take advantage of volume discounts, if applicable with Langbase.

**Resource Management:**
*   Monitor local resource usage (CPU, memory) when considering the "Local" pathway. If local resources are constrained, the `Performance Predictor` should reflect this, potentially biasing towards hosted options.

## MONITORING

**Metrics Collection:**
*   **Route Usage**: Track how frequently each route (Local, Hosted, Hybrid) is selected.
*   **Decision Latency**: Measure the time taken for the `RoutingDecisionEngine` to make a decision.
*   **Accuracy of Prediction**: Compare predicted costs/performance with actual outcomes (requires post-processing).
*   **Fallback Activations**: Monitor how often fallback mechanisms are triggered.
*   **Privacy Detections**: Track the number of times sensitive content is detected.
*   **Cost Savings**: Quantify the savings achieved by smart routing, especially by leveraging Langbase Memory.

**Alerting:**
*   Set up alerts for high error rates within any component of the decision engine.
*   Alerts for unexpected cost spikes or performance degradations.

**Logging:**
*   Comprehensive logging of each routing decision, including the input request, the scores from each component, the chosen route, and the explanation. This is crucial for auditing, debugging, and refining the decision logic.

**Visual Dashboards:**
*   Create dashboards to visualize route distribution, decision latency, and cost savings over time. This will provide insights into the effectiveness of the routing logic.