# RRPE – Resilience Runtime Policy Ecosystem: Component Specifications

| # | Component | Role |
|---|-----------|------|
| 1 | Resilience Protocol Specification and Reference Libraries | Defines the shared headers, rules and semantics |
| 2 | Service Interceptor Layer | Sidecar proxy that enforces the rules at runtime and emits telemetry |
| 3 | Kubernetes Enforcement Layer | Injects sidecars and distributes approved configuration |
| 4 | Resilience Intelligence Layer | Analyses telemetry and calculates adaptive policies |

---

## System Overview and Component Connections

### End-to-end flow

```
Component 1 (protocol rules)
        │ used by gateway + sidecars
        ▼
Component 2 (interceptor) ──metrics/traces──► Prometheus / Tempo
        ▲                                            │
        │ applies policies                           ▼
Component 3 (Config API) ◄──── policies ──── Component 4 (analysis)
        │                                            ▲
        └── Redis (stores proposed/approved) ────────┘
              Deadline changes: admin approval required
              Leaky Bucket / Retry Budget: sent automatically
```

1. **Component 2 provides telemetry** (metrics to Prometheus, traces to Tempo).
2. **Component 4 analyses it**, classifies service health and calculates policies.
3. **Component 4 stores/sends policies via Component 3.** Deadline changes wait in Redis for authorised approval; Leaky Bucket and Retry Budget changes are released automatically once safety checks pass.
4. **Component 3 distributes** approved policies through the Configuration API to the correct sidecars and records whether they were applied.
5. **Component 2 applies** the policies, using the rules defined by Component 1.
6. **Component 4 monitors recovery** and proposes gradual restoration.

### Connection matrix

| From → To | What is exchanged |
|-----------|-------------------|
| C1 → C2 | Header definitions, validation, propagation and budget-allocation rules via the Java/Go libraries |
| C1 → API Gateway | Library used to create/validate the resilience context and inject headers; initial deadlines and retry allowances come from configured policies |
| C1 → C4 | Consistent meaning of identifiers, budgets and outcomes in recorded observations |
| C2 → C4 | Prometheus metrics and Tempo traces |
| C3 → C2 | Sidecar injection (webhook) and delivery of approved policy updates (Configuration API) |
| C4 → C3 | Proposed and automatically released policies (stored in Redis, delivered via the Configuration API) |
| C3 ↔ C4 | Communication path so runtime events lead to proposed changes and, after authorisation, to propagation |

> **Responsibility boundary:** Component 1 *defines* the rules, Component 2 *enforces* them at runtime (deadline checks, retries, cancellation, backpressure), Component 3 *deploys and distributes* (injection, config state, Helm), and Component 4 *decides* (health classification and policy calculation).

---

## Component 1 – Resilience Protocol Specification and Reference Libraries

Component 1 defines the common communication rules used by the RRPE ecosystem. It specifies how resilience information is represented in HTTP headers, how values change between services, and how the other components interpret them. It also provides Java and Go libraries that implement these rules consistently.

### Functions

1. **Header definition and message format** – Define request and response headers, including names, data types, units, purposes, and required/optional status. Specify the meaning of request identity, deadline, remaining time, retry allowance, cancellation and backpressure.
2. **Header encoding, decoding and validation** – Provide library functions to write context into HTTP headers and read it back. Detect malformed headers, unsupported protocol versions, invalid budgets and conflicting values.
3. **Cross-service propagation rules** – Request identity remains consistent, deadlines cannot be extended, remaining time decreases, and retry allowance decreases when consumed. A triggered cancellation cannot be reset downstream.
4. **Deadline, cancellation and response semantics** – Define how deadline expiry and cancellation are represented and interpreted. Specify terminal outcomes that prohibit further retries. Provide calculation functions used by Components 2 and 3 (see note below). Cancellation delivery and application cooperation must also be defined; a header alone cannot stop an already-running operation.
5. **Retry-budget allocation during fan-out** – Define how retry allowance is divided when a service calls multiple downstream services, so every branch does not independently reuse the entire parent allowance.
6. **Topology metadata** – Define identifiers and parent relationships needed to distinguish downstream calls, parallel branches and retry attempts, so recorded observations can reconstruct service-call relationships and connect them with resilience outcomes.
7. **Java and Go reference libraries** – Implement the protocol in both languages with shared conformance tests that verify both interpret the same headers and apply the same rules.

### Connectivity

- **API Gateway** – Uses the library to initialise or validate the resilience context and inject headers. Initial deadlines and retry allowances come from configured policies.
- **Component 2 (Interceptor Layer)** – Uses the library to read and validate incoming headers, calculate remaining time, allocate downstream budgets and prepare outgoing headers. Component 1 defines the rules; Component 2 performs retries, timeout handling and application-level cancellation.
- **Component 3 (Kubernetes Enforcement Layer)** – Does not interpret the protocol itself. Traffic-level actions such as rejecting expired requests are performed by the Component 2 sidecar that Component 3 injects and configures. *(Original text assigned proxy-boundary enforcement to Component 3; corrected to match the Component 2 and Component 3 specifications.)*
- **Component 4 (Intelligence Layer)** – Analyses observations containing the identifiers, budgets and outcomes defined by Component 1. Its policies reach the gateway and sidecars through Component 3 and are applied by Component 2.

### Proposed novelty

1. **Coordinated resilience semantics across HTTP service calls** – A shared protocol defining how deadlines, retry allowance, cancellation and backpressure interact across service boundaries, including terminal outcomes and valid state changes.
2. **Validated budget propagation across retries and fan-out** – Rules preventing downstream calls from increasing their allowance or duplicating a shared retry budget; experiments should show correctness under branching and concurrency.
3. **In-band topology metadata linked to resilience decisions** – Call-relationship metadata carried with requests and linked to budget consumption, cancellation and deadline outcomes. Novelty depends on showing a benefit beyond existing trace-context propagation.
4. **Measurable protocol efficiency and interoperability** – Experiments measure header size and parsing overhead; shared Java–Go tests demonstrate consistent behaviour. Efficient encoding and multiple implementations alone do not establish novelty.

---

## Component 2 – Service Interceptor Layer

The service interceptor layer enforces deadline propagation, cancellation propagation, retry-storm prevention and backpressure handling. It uses a sidecar proxy to intercept inbound and outbound HTTP calls between services.

### Main responsibilities

- Detect deadline exceedance and propagate cancellation signals to downstream and upstream services.
- Prevent retry storms.
- Handle backpressure.

### Implemented: deadline and cancellation propagation

- The API gateway sidecar selects a deadline duration based on the request's HTTP method and path. Budgets are configured outside the Java code through an environment variable (e.g. `GET /api/orders` can differ from `POST /api/orders`). Query parameter values are ignored when matching a route.
- When a request enters the gateway, the sidecar creates a request ID and an absolute deadline and passes this context in HTTP headers (`Request_id`, `deadlinevalue`, `Request_deadline_millis`, `Request_method`, `Request_route`). The same absolute deadline is passed along the chain; services do not start new timers.
- When a service makes an outgoing call, its sidecar uses the request ID to find the saved deadline context and adds it to the outgoing call.
- Each sidecar checks whether the deadline has expired. If so, it sets `deadlineExceeded=true` and `cancellation_Triggered=true`, records the detection in metrics and on the service's trace span, and the response carries the status upstream.

> Header names above are as implemented. They should be aligned with the names defined by Component 1 (including correcting the spelling `deadlineExceded` → `deadlineExceeded`).

### To be implemented

- Retry-storm prevention (driven by the Retry-Budget policy from Component 4).
- Backpressure handling (driven by the Leaky Bucket policy from Component 4).

### Novelty

A service-mesh-level proxy that increases microservice resilience through deadline propagation, cancellation propagation, retry-storm prevention and backpressure handling.

### Connectivity

- **Component 1 (Protocol Specification)** – Defines header rules; the header validation library validates header values.
- **Component 4 (Intelligence Layer)** – The proxy exposes metrics to Prometheus and traces to Tempo; Component 4 uses this data for analysis and prediction.
- **Component 3 (Kubernetes Enforcement Layer)** – Deploys the proxy as a Kubernetes sidecar via the admission webhook (including the traffic-redirection/TPROXY setup, to be confirmed in the injection design) and delivers approved policy updates to it through the Configuration API.

---

## Component 3 – Kubernetes Enforcement Layer

The Kubernetes Enforcement Layer integrates the runtime adaptation system into an existing Kubernetes-based microservices environment, so interception, configuration management and runtime adaptation work with existing microservices **without modifying their application source code**.

It acts as the integration and enforcement mechanism between existing microservices and the adaptation platform: it injects the sidecar proxy into application pods, manages configuration state, delivers approved updates to the Interceptor Layer, and supports installation of the platform into a cluster. It also establishes communication between the Interceptor Layer and the Intelligence Layer so runtime events can lead to proposed configuration changes and, after authorisation, propagate back to the relevant sidecars.

### Main responsibilities

1. Automatic sidecar injection using a Kubernetes Admission Webhook.
2. Configuration state management using Redis.
3. Configuration distribution through a Configuration API (a small service carrying approved policy to sidecars).
4. Integration with the Interceptor Layer and Intelligence Layer.
5. Platform installation and adaptation using Helm.

### 1. Admission Webhook and automatic sidecar injection

When a new pod is created, the Kubernetes API server invokes the webhook according to its admission rules. The webhook examines the pod and decides whether it should receive the sidecar proxy; for eligible pods it modifies the pod specification to include the interceptor sidecar and its configuration. This removes the need for developers to modify each application manually.

### 2. Redis-based configuration state management

Redis centrally stores configuration required by the adaptation workflow. When the Intelligence Layer identifies a possible improvement (e.g. increasing a deadline), the proposed configuration is stored and tracked until reviewed and approved. Component 3 ensures this state is available to the Configuration API and related components.

### 3. Configuration API and runtime configuration distribution

The Configuration API is the interface between central configuration management and the Interceptor Layer. It makes approved changes available to the appropriate sidecars and records whether they were applied successfully. It also separates configuration storage from runtime enforcement: the Intelligence Layer does not need to talk to each sidecar directly.

### 4. Integration and deployment using Helm

Helm is the packaging and installation mechanism for the platform's management components and configuration resources. Helm does not itself inject sidecars or propagate configuration; it installs the Kubernetes resources that let the Admission Webhook, Configuration API, Redis, Intelligence Layer and other components operate together.

### Novelty and research contribution

Integrating a runtime adaptation platform into existing Kubernetes microservices without application source-code changes, with visibility of sidecar coverage and correct policy distribution.

### Implementation status

- **Completed:** Admission Webhook server in Go; it automatically injects the sidecar proxy into pods according to the configured injection mechanism.
- **Remaining:** Redis database; Configuration API (small service carrying updated policy to sidecar proxies once an admin approves it); Helm-based integration and deployment.

### Connectivity

- **Component 2** – Injects the sidecar and delivers approved policies to it.
- **Component 4** – Receives proposed/automatic policies, stores them in Redis, and distributes them after approval (or automatically for Leaky Bucket and Retry Budget).
- **Component 1** – No direct dependency; carries the protocol only through the sidecars it deploys.

---

## Component 4 – Resilience Intelligence Layer

The analysis and decision-making part of RRPE. It uses service metrics from Prometheus and request-chain traces from Tempo. Every minute it compares the latest five minutes of behaviour with a longer Healthy baseline, classifies each service as **Healthy**, **Stressed** or **Overloaded**, and calculates suitable resilience policies.

### 1. Service-health classification

A baseline-based adaptive threshold algorithm. For each service it learns the normal median and variation (MAD – Median Absolute Deviation):

```
Stressed Threshold   = Baseline Median + (2 × MAD)
Overloaded Threshold = Baseline Median + (3 × MAD)
```

Inputs: service p95 latency, error rate, retry rate, rejection rate, in-flight requests and successful throughput. A condition must normally persist for two analysis cycles to avoid reacting to temporary spikes.

### 2. Chain-deadline policy

A separate deadline is calculated per request chain from its end-to-end p99 latency:

```
Candidate Deadline = Recent Chain P99 × (1 + Safety Margin)
```

| Latency change | Safety margin |
|----------------|---------------|
| Up to 10% | 10% |
| 10% – 25% | 20% |
| Above 25% | 30% |

Deadline changes are shown with evidence for an authorised user to **approve or reject**. Restoration after recovery is gradual and also requires approval.

### 3. Leaky Bucket policy

Controls how many original requests a service accepts per second:

```
Capacity         = Minimum(Baseline Throughput, Recent Throughput)
Recommended Rate = Capacity × Safety Factor
```

- Stressed → safety factor 0.90
- Overloaded → safety factor 0.75

Updated **automatically** when the problem persists and all safety checks pass. After recovery, the rate is increased in small steps toward the Healthy rate.

### 4. Retry-budget policy

Controls additional retry attempts:

```
Retry Effectiveness = Successful Retries ÷ Total Retry Attempts
Budget Ratio        = Maximum Ratio × Retry Effectiveness × Health Factor
```

Health factors: Healthy → 1.0, Stressed → 0.5, Overloaded → 0.0 (retries disabled during overload). Updated **automatically** and restored gradually after recovery. Retries are permitted only for retryable failures, safe operations and requests with enough deadline time remaining.

### Novelty

Combines service-level metrics and complete request-chain evidence to calculate coordinated, explainable policies. Instead of one fixed value for every service, it learns a separate baseline per service and request chain, and explains each decision using observed values, baseline values and the triggered rule.

### Connection with other components

| Component | Relationship |
|-----------|--------------|
| **Component 1** | Defines the protocol, context headers and propagation rules whose identifiers, budgets and outcomes appear in the analysed data |
| **Component 2** | Exposes runtime telemetry (Prometheus, Tempo) and applies policies through RRPE sidecars |
| **Component 3** | Stores proposed policies in Redis and distributes approved/automatic policies to the correct sidecars via the Configuration API |

**Flow:** Component 2 provides telemetry → Component 4 analyses it → deadline changes wait for approval, while Leaky Bucket and Retry-Budget changes are sent automatically → Component 3 distributes them → Component 2 applies them → Component 4 monitors recovery.
