\# Webhook Delivery Platform — Design Document



\## 1. System Goal



Build a multi-tenant webhook delivery platform that accepts events

asynchronously and delivers them reliably to subscribed customer endpoints.



The platform must provide at-least-once delivery while handling:



\- Customer endpoint failures

\- Slow endpoints

\- Network failures

\- Duplicate event submissions

\- Repeated delivery failures

\- Poison messages

\- Noisy-neighbor isolation

\- Delivery inspection and replay



The system should protect the ingestion path from slow downstream

customers and should prevent one failing endpoint from exhausting

platform capacity.



\## 2. Functional Requirements



\### Tenant and Endpoint Management



\- Register and manage tenants.

\- Register webhook endpoints per tenant.

\- Configure subscribed event types.

\- Store a signing secret per endpoint.



\### Event Ingestion



\- Accept events through a REST API.

\- Return `202 Accepted` quickly.

\- Support idempotency keys.

\- Prevent duplicate event creation for the same idempotency key.



\### Delivery



\- Fan out events to matching endpoints.

\- Sign webhook requests using HMAC-SHA256.

\- Include a timestamp in the signature.



\### Reliability



\- Retry failed deliveries using exponential backoff with jitter.

\- Move permanently failing messages to a dead-letter queue.

\- Prevent a failing endpoint from blocking other endpoints.



\### Inspection and Replay



\- Store delivery attempts.

\- Record status, latency, response code, and timestamps.

\- Allow failed or historical events to be replayed.



\### Observability



\- Publish application and infrastructure metrics.

\- Maintain structured logs.

\- Provide alarms and dashboards.

\- Support distributed tracing.



\## 3. Non-Functional Targets



Initial targets will be validated through load and failure testing.



| Metric | Target | Measured |

|---|---:|---:|

| Ingestion p99 latency | < 100 ms | TBD |

| Sustained throughput | 500+ events/sec | TBD |

| Message loss during failure tests | 0 | TBD |

| Cost per 1M events | TBD | TBD |



Targets may be revised after the design and load-testing phases.

## 4. High-Level Architecture



\### Event Ingestion Flow



Producer sends an event to API Gateway.



API Gateway invokes the ingestion Lambda.



The ingestion Lambda validates the request, performs idempotency

handling, persists the event state, and places asynchronous delivery

work onto SQS.



Customer webhook endpoints are not called synchronously from the

ingestion request path.



\### Initial Responsibility Split



\- DynamoDB: durable event state, idempotency state, tenant and endpoint data.

\- SQS: asynchronous delivery work, buffering, retry/visibility semantics.

\- Lambda: request validation and asynchronous processing.



\## 5. DynamoDB Access Patterns



The DynamoDB design will be driven by application access patterns

rather than by traditional relational table normalization.



Initial access patterns:



| # | Access Pattern |

|---|---|

| 1 | Get tenant by tenant ID |

| 2 | List endpoints for a tenant |

| 3 | Get a specific endpoint |

| 4 | Get event by event ID |

| 5 | Find an event by tenant + idempotency key |

| 6 | List delivery attempts for an event |

| 7 | List recent attempts for an endpoint |

| 8 | Get circuit-breaker state for an endpoint |

| 9 | Get replay information for a failed event |



The final partition-key, sort-key, and GSI design will be derived

from these access patterns.

## 7. Secondary Index and Partitioning Considerations



A Global Secondary Index (GSI) may be required when an access pattern

cannot be efficiently served by the base table primary key.



Potential GSI access patterns include:



\- Query delivery attempts by endpoint.

\- Query events or deliveries by status.

\- Query recent activity for a tenant.



The initial tenant-based partition key also creates a potential hot

partition risk for high-volume tenants.



The final partitioning strategy will therefore be validated against

expected event rates, delivery-attempt volume, tenant distribution,

and noisy-neighbor isolation requirements before implementation.

## 8. Ingestion Reliability Decision



\### Decision



Use DynamoDB as the durable event store and DynamoDB Streams to

propagate newly created events into the asynchronous delivery pipeline.



\### Why



This separates durable event persistence from asynchronous delivery work.

A successful event write is captured by DynamoDB Streams without requiring

the ingestion Lambda to coordinate an independent DynamoDB write and SQS

send as one operation.



The downstream pipeline remains at-least-once and must therefore be

idempotent.



\### Alternatives Rejected



\- Transactional Outbox: stronger explicit coordination, but adds

&#x20; application-level publishing and reconciliation complexity.

\- SQS-first: simplifies buffering but makes durable event persistence

&#x20; dependent on downstream processing.



\### Trade-off



DynamoDB Streams introduces another asynchronous processing stage and

requires careful handling of duplicate stream processing and failures.



\### Cost Impact



Additional DynamoDB Streams reads and Lambda processing will contribute

to operating cost. Actual cost will be measured after implementation and

load testing.

## 9. End-to-End Event Flow



1\. Producer sends an event with an idempotency key.

2\. API Gateway invokes the ingestion Lambda.

3\. The ingestion Lambda validates the request and checks idempotency.

4\. A new event is persisted in DynamoDB.

5\. DynamoDB Streams captures the new event.

6\. A stream processor publishes delivery work to the SQS delivery queue.

7\. Delivery workers consume delivery work.

8\. The worker loads the endpoint configuration and signing secret.

9\. The worker generates an HMAC-SHA256 signature containing a timestamp.

10\. The worker sends the webhook to the customer endpoint.

11\. The delivery attempt is recorded with status, latency, and response code.

12\. Failed deliveries follow the retry policy.

13\. Permanently failing deliveries are moved to the DLQ.



\### Duplicate Delivery Semantics



The platform provides at-least-once delivery, not exactly-once delivery.



A worker may successfully send a webhook and fail before recording

the successful attempt. The message can then be processed again.



Every event therefore has a stable event ID that receivers can use for

their own idempotency handling.

### Event Fan-Out



A single event may match multiple subscribed endpoints.



The stream-processing stage will identify endpoints belonging to the

authenticated tenant that subscribe to the event type.



For each matching endpoint, the processor will create independent

delivery work.



Each delivery will have its own delivery identity and lifecycle so that

one failing endpoint does not block delivery to other endpoints.



Example:



Event: payment.succeeded



Tenant endpoints:

\- Endpoint A subscribes to payment.succeeded

\- Endpoint B subscribes to payment.succeeded

\- Endpoint C does not subscribe



The platform creates delivery work for Endpoint A and Endpoint B only.

## 10. Retry Architecture



Failed deliveries will use exponential backoff with jitter.



Initial retry schedule:



1\. 10 seconds

2\. 1 minute

3\. 10 minutes

4\. 1 hour

5\. 6 hours



After the configured retry limit is exhausted, the delivery will be

moved to the dead-letter queue.



EventBridge Scheduler will be used for long retry delays rather than

keeping SQS messages invisible for hours.



Jitter will be added to retry delays to reduce synchronized retry bursts

and protect downstream customer endpoints.



A retry attempt will retain the same event ID and delivery identity so

that duplicate processing can be detected and investigated.

## 11. Delivery Outcome Classification



Webhook delivery outcomes will initially be classified as follows:



\- HTTP 2xx: successful delivery; no retry.

\- HTTP 429: retryable failure; apply backoff and jitter.

\- HTTP 5xx: retryable failure; apply backoff and jitter.

\- Connection timeout: retryable failure.

\- Network/connection error: retryable failure.

\- Other HTTP 4xx responses: permanent failure; do not retry indefinitely.



The classification policy is intentionally explicit because retrying

non-recoverable failures can waste capacity and increase load on

customer endpoints.



A delivery attempt will record the HTTP status code when available,

latency, error classification, attempt number, and timestamp.

## 12. Endpoint Isolation and Circuit Breaker



Each webhook endpoint will have a configurable concurrency limit so that

one endpoint cannot consume unlimited delivery-worker capacity.



Repeated failures will activate a circuit breaker for the endpoint.



Circuit states:



\- CLOSED: normal delivery.

\- OPEN: delivery paused after repeated failures.

\- HALF-OPEN: after a cooldown, allow a limited test delivery.

\- Successful test delivery returns the circuit to CLOSED.

\- Failed test delivery returns the circuit to OPEN.



Circuit-breaker state will be persisted in DynamoDB rather than relying

on Lambda in-memory state, because Lambda execution environments are

ephemeral and multiple workers may process deliveries concurrently.



The mechanism is intended to isolate unhealthy endpoints and protect

healthy tenants from noisy neighbors.

## 13. Webhook Signing and Replay Protection



Each endpoint will have a signing secret.



For every delivery, the platform will generate a timestamp and compute

an HMAC-SHA256 signature over a canonical representation containing the

timestamp and webhook payload.



The request will include the timestamp and signature in HTTP headers.



The receiving application can independently calculate the expected

signature and reject requests whose signature does not match.



The timestamp provides replay protection when the receiver enforces an

acceptable timestamp tolerance.



The signing secret must never be stored in source code or committed to

Git. Secrets will be managed using an AWS-managed secret mechanism.



The exact canonical signing format and timestamp tolerance will be

defined before implementation.

## 14. Authentication and Tenant Isolation



The management and ingestion APIs will use API-key authentication.



The authenticated API key will identify the tenant associated with the

request. Tenant identity will not be trusted from a user-supplied

tenant ID in the request body.



Authorization will be tenant-scoped. API operations must only access

resources belonging to the authenticated tenant.



API credentials must not be stored in plaintext in source code or

committed to Git.



The authentication and credential-storage implementation will be

finalized during the security design phase.

## 15. Initial REST API Surface



\### Event APIs



\- POST /events — accept a new event and return 202 Accepted.

\- GET /events/{eventId} — inspect an event.

\- GET /events/{eventId}/deliveries — inspect delivery attempts.

\- POST /events/{eventId}/replay — request replay of an event.



\### Endpoint APIs



\- POST /endpoints — register a webhook endpoint.

\- GET /endpoints — list endpoints for the authenticated tenant.

\- GET /endpoints/{endpointId} — inspect an endpoint.

\- PATCH /endpoints/{endpointId} — update endpoint configuration.

\- POST /endpoints/{endpointId}/replay — request endpoint-scoped replay.



\### Authentication



Management and ingestion APIs require tenant API-key authentication.



\### Ingestion Semantics



POST /events accepts an Idempotency-Key and returns 202 Accepted after

the platform accepts the event for asynchronous processing.



202 Accepted does not mean that downstream webhook delivery has completed.

## 16. Idempotency Semantics



Each event ingestion request requires an Idempotency-Key.



The platform will associate the idempotency key with the authenticated

tenant and a request fingerprint.



If the same tenant submits the same idempotency key with the same

request fingerprint, the platform will treat the request as a duplicate

and return the existing event result.



If the same idempotency key is reused with a different request

fingerprint, the platform will reject the request with HTTP 409 Conflict.



Idempotency keys are tenant-scoped; the same key used by two different

tenants does not represent the same request.

## 17. Delivery Attempt Data



Each delivery attempt will record enough information for debugging,

replay, reliability analysis, and operational metrics.



Initial fields:



\- tenant ID

\- event ID

\- endpoint ID

\- attempt number

\- delivery status/classification

\- HTTP response code when available

\- latency

\- attempt start and completion timestamps

\- error category when applicable

\- next retry timestamp when applicable

\- correlation/request ID



Customer response bodies will not be stored by default because they may

contain sensitive information.



Delivery-attempt records will have TTL-based retention so historical

operational data does not grow indefinitely.

## 18. Dead-Letter Queue and Replay



A delivery that exhausts the configured retry policy will be moved to

the dead-letter queue.



DLQ placement means automatic retries have been exhausted; the event

remains available for inspection and manual replay.



Replay will preserve the original event ID because replaying an event

does not create a new business event.



A replay will create a new delivery/replay execution identity so that

the original attempts and replay attempts remain distinguishable in the

audit history.



Example:



\- eventId: original business event identity

\- deliveryId: endpoint-specific delivery execution

\- replayId: identifies a replay execution

\- attemptNumber: attempt within that execution

## 19. Data Retention and TTL



Delivery-attempt records will use DynamoDB TTL to limit long-term

operational data growth.



TTL is a retention and cost-control mechanism, not an exact-time

deletion guarantee.



Tenant and endpoint configuration will not use short operational TTLs.



Idempotency records require a deliberately chosen retention period.

The period must be long enough to cover expected producer retries and

must be documented before implementation.



The final retention periods will be selected based on operational

requirements, replay requirements, and cost measurements.

## 20. Observability and Measurement



The platform will expose metrics for both system health and project

performance validation.



\### Ingestion Metrics



\- Request count

\- HTTP 4xx/5xx count

\- Ingestion latency p50/p95/p99

\- Idempotency duplicate count



\### Delivery Metrics



\- Delivery success/failure count

\- Delivery latency p50/p95/p99

\- HTTP response-code distribution

\- Retry count

\- DLQ count



\### Isolation Metrics



\- Active deliveries per endpoint

\- Circuit-breaker state changes

\- Concurrency-limited deliveries



\### Queue and Worker Metrics



\- SQS queue depth

\- Oldest message age

\- Lambda errors

\- Lambda throttles

\- DynamoDB Stream processing failures



Performance claims will be based on measured load-test results rather

than estimates.



The load tests will report accepted events/sec, delivery throughput,

latency percentiles, error rate, and message-loss results.

## 21. Analytics Architecture



Operational delivery state and analytical history will use separate

storage paths.



Operational path:



Application → DynamoDB



Analytics path:



Delivery logs → Kinesis Data Firehose → S3 → Athena



DynamoDB will remain the operational source for event and delivery

state, replay, and inspection.



S3 will provide durable analytical storage for delivery-log data, while

Athena will support SQL-based analysis without placing analytical query

load on the operational DynamoDB workload.



Example analytical questions include:



\- Failure rate by tenant

\- Failure rate by endpoint

\- Delivery latency distribution

\- Retry volume over time

\- HTTP response-code distribution

\- DLQ volume

## 22. Scale Assumptions



Initial design target:



\- Sustained ingestion: 500 events/sec

\- Ingestion latency target: p99 < 100 ms

\- Message loss during failure testing: 0

\- Average endpoints matched per event: TBD

\- Event payload size: TBD

\- Peak ingestion rate: TBD

\- Expected retry rate: TBD

\- Load-test duration: TBD



Ingestion throughput and delivery throughput are different metrics.



For example, 500 events/sec with three matching endpoints per event can

produce approximately 1,500 delivery operations/sec before retries.



Final capacity results will be based on measured load-test data.

## 23. Cost Strategy



Cost will be measured rather than estimated as a final project result.



The project will track:



\- AWS service-level cost

\- Total test cost

\- Events ingested

\- Delivery attempts

\- Cost per million ingested events

\- Cost per million delivery attempts



Major cost-sensitive areas include Lambda, DynamoDB, SQS, EventBridge

Scheduler, Firehose, S3, Athena, CloudWatch, and X-Ray.



Load testing will be time-boxed and followed by resource cleanup.



Paid networking components such as NAT Gateway will not be introduced

unless required by the final architecture.



Final cost numbers will be based on actual AWS billing/usage data.

### Free-Tier Constraint



The project is being developed under an AWS Free Tier budget.



All resource choices and load tests must therefore be cost-conscious.

Before creating potentially billable resources, current AWS pricing and

Free Tier eligibility will be verified.



Load tests will be time-boxed, monitored, and followed by teardown.



No paid resource will be created without explicitly identifying its

potential cost and confirming that it is necessary for the experiment.

## 24. Security Baseline



The platform will enforce tenant isolation at the application and data

access layers.



Security principles:



\- API requests require tenant authentication.

\- Tenant identity comes from authenticated credentials, not user input.

\- IAM policies follow least privilege.

\- Workloads will use IAM roles rather than long-lived access keys.

\- Webhook signing secrets will be stored using AWS-managed secret

&#x20; protection and will not be committed to source control.

\- DynamoDB and S3 data will use encryption at rest.

\- S3 Block Public Access will remain enabled.

\- Secrets, credentials, and sensitive payload data must not be written

&#x20; to application logs.

\- No production-style workload will use root credentials.

\- Infrastructure resources will be tagged for project and environment

&#x20; identification.

## 25. Decision Log



| Decision | Why | Alternatives Rejected | Cost Impact | Trade-off |

|---|---|---|---|---|

| DynamoDB + Streams | Durable event state with asynchronous propagation | SQS-first, Transactional Outbox | Additional Streams/Lambda usage | Extra async stage |

| SQS for delivery buffering | Per-message processing and DLQ semantics | Kinesis as primary delivery queue | SQS request/processing cost | Ordering is not the primary guarantee |

| EventBridge Scheduler for long retries | Avoid holding messages invisible for hours | Long SQS visibility periods | Scheduler execution cost | Additional scheduling component |

| At-least-once delivery | More practical reliability model for distributed failures | Exactly-once delivery | Requires duplicate handling | Receivers must be idempotent |

| S3 + Athena for analytics | Separate analytical workload from operational database | DynamoDB scans | Storage/query costs | Analytics becomes eventually consistent |

## 26. Endpoint Disablement Alerts



When an endpoint is automatically disabled by the circuit-breaker policy,

the platform will publish an alert through Amazon SNS.



The notification will identify the affected tenant and endpoint and

provide enough information for the tenant to investigate the failure.



The alert mechanism must not include signing secrets or sensitive

payload data.

## 27. Infrastructure as Code and CI/CD



The infrastructure will be defined using AWS CDK with Python and

deployed through AWS CloudFormation.



The public repository will contain the application code,

infrastructure definitions, tests, documentation, and architecture

artifacts.



GitHub Actions will provide CI/CD automation for validation and

deployment.



GitHub Actions will authenticate to AWS using OIDC and a scoped IAM

role rather than storing long-lived AWS access keys as repository

secrets.



The CI/CD pipeline will validate code and infrastructure before

deployment.

## 28. Load Testing and Failure Testing



Load testing will use k6 or Locust against the deployed ingestion API.



The initial target is 500 sustained events/sec with an ingestion p99

latency target below 100 ms.



Each test will record:



\- Offered request rate

\- Accepted events/sec

\- HTTP error rate

\- p50/p95/p99 latency

\- Delivery throughput

\- Delivery success/failure rate

\- Retry volume

\- DLQ count

\- Queue depth and message age



Failure tests will intentionally introduce conditions such as worker

failures, slow endpoints, HTTP 5xx responses, malformed data, and

retry-heavy workloads.



Message loss will be evaluated by correlating produced event IDs with

persisted events and final delivery outcomes.



Resume/project performance claims will use only measured results from

reproducible tests.

## 29. DynamoDB Access Patterns



The DynamoDB data model will be designed from known application access

patterns rather than from entity definitions alone.



Required access patterns:



1\. List all endpoints for a tenant.

2\. Retrieve a specific endpoint.

3\. Retrieve an event by event ID.

4\. Retrieve all delivery records for an event.

5\. Retrieve recent delivery attempts for an endpoint.

6\. Look up an event using a tenant-scoped idempotency key.

7\. Find endpoints belonging to a tenant that subscribe to an event type.

8\. Retrieve and update circuit-breaker state for an endpoint.

9\. Retrieve the event and endpoint information required for replay.



The final PK, SK, and GSI design will be derived from these access

patterns and validated against expected traffic distribution and

hot-partition risk.

## 30. DynamoDB Logical Entities



The single-table design will represent the following logical entities:



\- Tenant — customer/business account.

\- Endpoint — tenant-owned webhook destination and delivery configuration.

\- Event — original business event submitted by the producer.

\- Delivery — one event-to-endpoint delivery lifecycle.

\- DeliveryAttempt — an individual attempt within a delivery.

\- Idempotency — tenant-scoped mapping between an idempotency key,

&#x20; request fingerprint, and event.



The relationship is:



Event → Delivery → DeliveryAttempt



One event may produce multiple deliveries when multiple endpoints match

the event type.

## 31. GSI Access Patterns



The design will use GSIs for access patterns that should not require

scanning or inefficient application-side filtering.



\### GSI 1: Endpoint Delivery Attempts



Purpose: retrieve recent delivery attempts for a specific endpoint.



Conceptual keys:



\- GSI1PK: ENDPOINT#<endpointId>

\- GSI1SK: <attempt timestamp>



\### GSI 2: Tenant Event-Type Subscription Lookup



Purpose: find endpoints belonging to a tenant that subscribe to a

specific event type.



Conceptual keys:



\- GSI2PK: TENANT#<tenantId>#TYPE#<eventType>

\- GSI2SK: ENDPOINT#<endpointId>



The final physical item representation will be validated because an

endpoint may subscribe to multiple event types.

## 32. Endpoint Subscription Representation



An endpoint may subscribe to multiple event types.



Rather than storing event types only as an attribute on the endpoint

item, the single-table design will represent subscriptions as separate

logical items.



Conceptually:



\- Endpoint item represents endpoint configuration.

\- Subscription item represents one endpoint-to-event-type relationship.



A subscription item will participate in the tenant + event-type GSI so

that the fan-out process can directly query matching endpoints.



This separates endpoint configuration from event subscription

relationships and avoids inefficient application-side filtering.

### Idempotency Scope Decision



Idempotency keys are scoped to a tenant rather than globally across

the platform.



Therefore, the same idempotency key may be used independently by

different tenants without collision.



The platform will also store a request fingerprint with the

idempotency record.



For the same tenant:



\- Same idempotency key + same request → return the existing event.

\- Same idempotency key + different request → return an idempotency

&#x20; conflict rather than creating another event.



Conceptual lookup:



GSI3PK = TENANT#<tenantId>#IDEMPOTENCY#<idempotencyKey>

GSI3SK = EVENT#<eventId>

## 33. Event Persistence and Fan-Out Boundary



The ingestion path will persist the event and its idempotency state

before returning a successful asynchronous acceptance response.



Where required, related DynamoDB writes will use DynamoDB transactions

to avoid partial persistence of critical ingestion state.



Delivery fan-out will remain asynchronous and will be driven from

DynamoDB Streams after the event has been committed.



The ingestion API will not synchronously depend on downstream delivery

workers or customer endpoints.



The stream-processing stage must itself be idempotent because stream

processing may be retried.

## 34. DynamoDB Physical Key Design



The platform will use a single DynamoDB table with the following

logical item structure:



| Entity | PK | SK |

|---|---|---|

| Tenant | TENANT#<tenantId> | META |

| Endpoint | TENANT#<tenantId> | ENDPOINT#<endpointId> |

| Subscription | TENANT#<tenantId> | SUBSCRIPTION#<eventType>#<endpointId> |

| Event | TENANT#<tenantId> | EVENT#<eventId> |

| Delivery | EVENT#<eventId> | DELIVERY#<endpointId> |

| Attempt | DELIVERY#<deliveryId> | ATTEMPT#<attemptNumber> |

| Idempotency | TENANT#<tenantId> | IDEMPOTENCY#<key> |



\### GSI1 — Endpoint History



\- GSI1PK: ENDPOINT#<endpointId>

\- GSI1SK: <timestamp>#<entityId>



Used to query delivery activity for an endpoint.



\### GSI2 — Subscription Lookup



\- GSI2PK: TENANT#<tenantId>#TYPE#<eventType>

\- GSI2SK: ENDPOINT#<endpointId>



Used by the fan-out process to find endpoints subscribed to an event

type within a tenant.



A separate idempotency GSI is not required because the base table

supports tenant-scoped idempotency-key lookup directly.



The schema will be validated during implementation and load testing

for hot partitions, item size, and access-pattern correctness.

## 35. Stream Processor Idempotency



DynamoDB Stream processing is treated as at-least-once.



The stream processor must therefore be idempotent.



A delivery is uniquely identified by the combination of event ID and

endpoint ID. The delivery record uses a deterministic DynamoDB key:



PK = EVENT#<eventId>

SK = DELIVERY#<endpointId>



Delivery creation will use a conditional write so that repeated

processing of the same stream record cannot create multiple delivery

records for the same event and endpoint.



This prevents duplicate delivery records caused by stream-processing

retries.



This does not provide exactly-once HTTP delivery. Network failures can

still cause the customer endpoint to receive the same webhook more than

once, so webhook consumers must use the event/delivery identity for

their own idempotency.

## 36. Delivery Outcome Classification



The delivery worker will classify customer endpoint outcomes before

deciding whether to retry.



\### Success



HTTP 2xx responses are treated as successful delivery.



\### Transient Failure



HTTP 5xx responses, connection failures, and request timeouts are

treated as retryable failures.



A timeout does not prove whether the customer received the request.

The request may have reached the receiver even if the worker did not

receive a response.



\### Permanent Failure



HTTP 4xx responses will generally be treated as non-retryable

failures because they commonly indicate an endpoint or request

configuration problem.



The final retry policy will be implemented with explicit response-code

classification rather than retrying every failure indiscriminately.



When retryable failures exceed the configured retry policy, the

delivery will move to the dead-letter path.

## 37. Retry and Backoff Strategy



Retryable delivery failures will use exponential backoff with jitter.



Initial retry schedule target:



\- Retry 1: approximately 10 seconds

\- Retry 2: approximately 1 minute

\- Retry 3: approximately 10 minutes

\- Retry 4: approximately 1 hour

\- Retry 5: approximately 6 hours

\- Further failure: dead-letter path



The exact delay may include randomized jitter to prevent many failed

deliveries from retrying simultaneously.



EventBridge Scheduler will be used for long retry delays rather than

keeping SQS messages invisible for hours.



The retry mechanism will create a delivery attempt record before or

during each retry transition so that the complete delivery history can

be inspected.



Scheduler usage and cost will be measured during load testing rather

than assumed.

## 38. Endpoint Isolation and Circuit Breaker



Each endpoint will have a configurable concurrency limit.



The delivery system must enforce the limit across workers so that a

single endpoint cannot consume the available delivery capacity and

create a noisy-neighbor problem.



Concurrency limiting protects platform capacity.



A circuit breaker provides a separate protection mechanism for

persistently failing endpoints.



Circuit states:



\- CLOSED — normal delivery.

\- OPEN — deliveries are paused after repeated failures.

\- HALF-OPEN — limited test delivery is allowed after a cooldown.



A successful test can return the endpoint to CLOSED. Continued failure

returns it to OPEN.



The final implementation will use shared coordination rather than

relying on Lambda-local in-memory counters.

## 39. Distributed Concurrency Control



Lambda workers are horizontally scalable, so endpoint concurrency cannot

be enforced safely using worker-local counters.



The design will require shared coordination for endpoint concurrency.



A DynamoDB-based conditional counter/lease mechanism is one candidate.

The coordination state will include expiry information so that a worker

failure does not permanently consume an endpoint's concurrency slot.



An alternative is SQS FIFO with endpoint-based message grouping, but

this may constrain concurrency and introduces FIFO-specific throughput

and cost considerations.



The final mechanism will be selected and validated during

implementation based on correctness, throughput, operational

complexity, and cost.

## 40. Replay Semantics



Replay creates a new delivery lifecycle rather than modifying the

original delivery history.



A replay must explicitly identify the target endpoint. It will use the

current configuration of that endpoint, including its current URL and

current signing secret.



Replay will not automatically fan out an old event to endpoints that

subscribed after the original event was created.



The original delivery and attempt history remains immutable for audit

and inspection.



Conceptual flow:



Replay API

→ validate tenant ownership and endpoint

→ create replay delivery

→ enqueue through the normal delivery pipeline

→ sign using current endpoint configuration

→ deliver

## 41. Webhook Request and Signing Contract



Webhook deliveries will use HTTP POST with a JSON payload.



Required headers:



\- Content-Type: application/json

\- X-Webhook-Id: unique event identifier

\- X-Webhook-Timestamp: Unix timestamp

\- X-Webhook-Signature: HMAC-SHA256 signature



The signature will be calculated from a canonical representation

containing the timestamp and raw request body.



The receiver can verify:



1\. The signature using the shared secret.

2\. The timestamp is within an acceptable tolerance window.

3\. The webhook/event identifier has not already been processed.



HMAC provides authenticity and integrity but does not provide

exactly-once delivery. Receivers must implement idempotency because

network failures can result in duplicate webhook deliveries.

## 42. HMAC Signature Format



The webhook signature will use HMAC-SHA256.



Canonical signed value:



&#x20;   timestamp + "." + raw\_request\_body



The resulting HMAC digest will be encoded as hexadecimal and sent in

the X-Webhook-Signature header.



The receiver must verify the signature against the raw HTTP request

body rather than a re-serialized JSON representation.



The receiver should also validate the webhook timestamp against a

configurable tolerance window to reduce replay risk.



The exact timestamp tolerance will be configured during implementation

and testing rather than hard-coded in the design document.

## 43. Design Status



The architecture and major access patterns have been reviewed before

implementation.



Implementation-specific limits and thresholds that depend on measured

workload, AWS service behavior, or cost will remain configurable until

validated through testing.



The design prioritizes tenant isolation, reliable asynchronous

processing, idempotency, observable delivery, security, and cost

awareness.

