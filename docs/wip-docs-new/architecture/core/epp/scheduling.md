# EPP Scheduler

The EPP Scheduler is a highly modular and extensible component within the Endpoint Picker (EPP) designed to select the optimal model server (endpoint) for an LLM request. It leverages a plugin-based architecture, allowing for sophisticated scheduling strategies based on real-time metrics, prefix caching, and model-specific requirements like LoRA adapters.

### Architecture Overview

The scheduler follows a **Research -> Strategy -> Execution** lifecycle for every request. It orchestrates multiple **SchedulerProfiles**, each defining a specific set of rules for filtering and scoring candidate endpoints.

```mermaid
flowchart TD
    Req[LLM Request] --> S[Scheduler.Schedule]
    
    subgraph Scheduling Cycle
        S --> Pick[ProfileHandler.Pick]
        Pick -->|List of Profiles| Loop{For each Profile}
        
        subgraph Profile Execution
            Loop --> Filters[Filters]
            Filters -->|Filtered Endpoints| Scorers[Scorers]
            Scorers -->|Weighted Scores| Picker[Picker]
            Picker --> Result[ProfileRunResult]
        end
        
        Result -->|Collect| Pick
        Pick -->|No more profiles| PRs[ProfileHandler.ProcessResults]
    end
    
    PRs -->|SchedulingResult| Target["Selected Endpoint(s)"]
```

#### Core Components

*   **Scheduler**: The main orchestrator that manages the scheduling cycle. It invokes `ProfileHandler` to pick profiles and then runs the selected profiles to obtain target endpoints.
*   **LLMRequest**: A structured representation of the incoming request, including the target model, parsed body (Completions, ChatCompletions, etc.), headers, and objectives.
*   **Endpoint**: Represents a candidate pod. It contains metadata (e.g., name, namespace) and real-time metrics (e.g., active models, queue depth, KV cache utilization).

### Extension Points

The scheduler's logic is distributed across several extension points, implemented via plugin interfaces:

1.  **ProfilePicker**: (Implemented by `ProfileHandler`) Selects which `SchedulerProfile`s to run based on the request and previous cycle results.
2.  **Filter**: Narrows down the list of candidate endpoints (e.g., based on health, SLO headroom, or cache affinity).
3.  **Scorer**: Assigns a score between `0.0` and `1.0` to each filtered endpoint. Multiple scorers can be weighted and combined.
4.  **Picker**: Selects the final endpoint(s) from the scored list (e.g., highest score, weighted random).
5.  **ProcessResults**: (Implemented by `ProfileHandler`) Aggregates the results from all executed profiles to produce the final `SchedulingResult`.

---

### Scheduler Profile

A `SchedulerProfile` is a configured pipeline consisting of:
*   **Filters**: A list of `Filter` plugins run sequentially.
*   **Scorers**: A list of `WeightedScorer` objects, where each contains a `Scorer` plugin and its relative weight.
*   **Picker**: A single `Picker` plugin that makes the final selection.

When a profile runs, it first filters the candidate endpoints. If any remain, it calculates a weighted aggregate score for each and then passes the scored list to the picker.

---

### Concrete Plugins

#### Filters
*   **`prefix-cache-affinity-filter`**: A probabilistic filter that narrows candidates to "sticky" endpoints (those with high prefix cache scores). It includes a "TTFT load gate" to break stickiness if sticky endpoints are significantly slower than non-sticky ones.
*   **`slo-headroom-tier-filter`**: Filters endpoints based on SLO headroom tiers to ensure quality of service.

#### Scorers
*   **`kv-cache-utilization-scorer`**: Prefers endpoints with lower KV cache utilization to avoid fragmentation.
*   **`latency-scorer`**: Scores endpoints based on predicted latency.
*   **`lora-affinity-scorer`**: Prefers endpoints that already have the requested LoRA adapter active or have capacity to load it.
*   **`prefix-scorer`**: Scores based on the length of the prefix cache match.
*   **`queue-depth-scorer`**: Prefers endpoints with shorter request queues.
*   **`running-requests-scorer`**: Scores based on the number of currently active requests.
*   **`token-load-scorer`**: Scores based on the total token load (input + output) handled by the endpoint.

#### Pickers
*   **`max-score-picker`**: Selects the endpoint with the absolute highest score.
*   **`random-picker`**: Selects an endpoint randomly from the candidates.
*   **`weighted-random-picker`**: Selects an endpoint randomly, using the scores as relative probabilities (lottery scheduling).

#### Profile Handlers
*   **`single-profile-handler`**: A standard implementation that runs a single configured primary profile.

---

### Advanced Use Cases: Prefill/Decode Disaggregation

The scheduler natively supports advanced routing paradigms, such as **Prefill/Decode Disaggregation (P/D Disagg)**. This is a serving technique where the initial prompt processing (prefill) and the subsequent token generation (decode) are handled by separate, specialized model servers.

In a P/D Disagg setup, the `ProfileHandler` orchestrates two separate `SchedulerProfiles`:
1.  **Prefill Profile**: Evaluates and scores endpoints specialized for compute-heavy prompt processing. It may use filters and scorers focused on prefix cache affinity, queue depth, or token load.
2.  **Decode Profile**: Evaluates and scores endpoints specialized for memory-bandwidth-bound token generation.

The `ProfileHandler` uses the `Pick` extension point to determine which profiles need to run for a given request (e.g., if a request needs both prefill and decode, or just decode if the KV cache is already transferred). The prefill and decode endpoints are picked at the same time. The `ProfileHandler` then uses the `ProcessResults` extension point to aggregate the selected endpoints. The decode endpoint is the one that will be returned to the proxy to forward the request to, while the prefill endpoint is added as a header that the decode sidecar uses to perform remote prefill.

---

### Metrics and Observability

The scheduler automatically records:
*   **E2E Latency**: Time taken for the entire scheduling cycle.
*   **Plugin Latency**: Time spent in each plugin extension point.
*   **Scheduling Attempts**: Counts of successful and failed scheduling decisions, partitioned by target model.
