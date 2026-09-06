## Designed for execution. Engineered for recovery.

![Animated architecture heading](https://readme-typing-svg.demolab.com?font=Fira+Code&size=17&pause=1500&color=38BDF8&vCenter=true&width=720&lines=Accept+deliberately.+Execute+predictably.;Bound+concurrency.+Make+retries+safe.;Observe+failures.+Design+the+recovery.)

A reference architecture for how I approach asynchronous AI and business
workflows: explicit admission decisions, durable execution, bounded
concurrency, and recoverable failure paths.

```mermaid
flowchart TB
    subgraph INGRESS["01 / ADMISSION"]
        direction LR
        A["API · Webhook · Event"] --> B["Authenticate<br/>Resolve tenant · Authorize"]
        B --> C{"Capacity and<br/>quota available?"}
        C -->|"Accept"| D["Persist job<br/>Idempotency key · Durable outbox"]
        C -->|"Reject or defer"| BP["Backpressure<br/>Retry-After · Rate limits"]
    end

    subgraph EXECUTION["02 / DURABLE EXECUTION"]
        direction LR
        Q[("Durable queue")] --> W["Worker / Supervisor<br/>Bounded concurrency"]
        W --> CHECK{"Already<br/>completed?"}
        CHECK -->|"No"| ROUTE["Provider / Domain routing<br/>Timeout · Circuit breaker"]
        ROUTE --> SERVICE["AI or domain service"]
        CHECK -->|"Yes"| EXISTING["Return stored result<br/>Skip repeated execution"]
    end

    subgraph RECOVERY["03 / FAILURE HANDLING"]
        direction LR
        CLASSIFY{"Classify failure"}
        RETRY["Bounded retry<br/>Backoff · Jitter · Same job identity"]
        FALLBACK["Approved fallback<br/>Alternate provider · Reduced capability"]
        DLQ[("Dead-letter queue")]
        RECON["Inspect · Reconcile<br/>Controlled replay"]
        CLASSIFY -->|"Transient · Within budget"| RETRY
        CLASSIFY -->|"Permanent · Budget exhausted"| DLQ
        DLQ --> RECON
    end

    subgraph COMPLETION["04 / RESULTS & OPERATIONS"]
        direction LR
        RESULT[("Persist result<br/>Commit completion state")]
        ACK["Acknowledge job<br/>After durable completion"]
        DELIVERY["Result delivery<br/>API · Event · Notification"]
        OBS["Operational visibility<br/>Traces · Metrics · Logs · Audit"]
        RESULT --> ACK
        RESULT --> DELIVERY
    end

    D -->|"Outbox dispatcher"| Q
    SERVICE -->|"Success"| RESULT
    SERVICE -->|"Execution failure"| CLASSIFY
    ROUTE -->|"Circuit open / unavailable"| FALLBACK
    FALLBACK -->|"Success"| RESULT
    FALLBACK -->|"Unable to recover"| CLASSIFY
    RETRY -->|"Delayed requeue"| Q
    RECON -->|"After safety checks"| Q
    EXISTING --> ACK

    B -.-> OBS
    W -.-> OBS
    CLASSIFY -.-> OBS
    RESULT -.-> OBS

    classDef entry fill:#0F172A,stroke:#38BDF8,color:#F8FAFC,stroke-width:2px
    classDef process fill:#102A43,stroke:#0EA5E9,color:#F8FAFC,stroke-width:1.5px
    classDef decision fill:#123B56,stroke:#7DD3FC,color:#F8FAFC,stroke-width:2px
    classDef storage fill:#0C4A6E,stroke:#38BDF8,color:#F8FAFC,stroke-width:2px
    classDef recovery fill:#172554,stroke:#60A5FA,color:#F8FAFC,stroke-width:1.5px
    classDef observe fill:#083344,stroke:#22D3EE,color:#ECFEFF,stroke-width:2px

    class A entry
    class B,D,W,ROUTE,SERVICE,EXISTING,ACK,DELIVERY process
    class C,CHECK,CLASSIFY decision
    class Q,RESULT storage
    class BP,RETRY,FALLBACK,DLQ,RECON recovery
    class OBS observe

    style INGRESS fill:#080F1D,stroke:#1E3A5F,color:#BAE6FD
    style EXECUTION fill:#080F1D,stroke:#1E3A5F,color:#BAE6FD
    style RECOVERY fill:#080F1D,stroke:#1E3A5F,color:#BAE6FD
    style COMPLETION fill:#080F1D,stroke:#1E3A5F,color:#BAE6FD
```

**The engineering contract**

- **Admission is explicit:** Enforce tenant permissions, quotas, and capacity before accepting work.
- **Acceptance is durable:** Persist the job before reporting acceptance; dispatch through an outbox.
- **Retries preserve identity:** Use idempotency and reconciliation to handle duplicate delivery and ambiguous outcomes.
- **Execution is bounded:** Apply concurrency limits, deadlines, and retry budgets.
- **Completion survives failure:** Persist results before acknowledging the job.
- **Recovery is controlled:** Inspect failed work before replay; apply fallbacks only where domain rules permit.

> At-least-once delivery requires deliberate duplicate handling.
> External side effects need provider idempotency or reconciliation;
> a queue alone cannot guarantee exactly-once execution.
