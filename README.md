# observability_tasks
Observability Task Design Take-Home — Solution Set

These tasks are designed to evaluate a learner (or an AI system) on the ability to reason across multiple observability signals, handle noisy/incomplete telemetry, and reach root cause + mitigation + prevention—not just run one query.

Each task includes:

A realistic on-call scenario

A multi-step investigation workflow

Datasets (logs/metrics/telemetry) and schemas

Dataset characteristics and scale

What makes the task hard (real-world messiness)

Task 1: Investigate an API Latency Regression Using High-Throughput Application Logs
Scenario

A customer-facing API (checkout-api) has been stable. On Day 8 ~14:00 UTC, an alert fires: p95/p99 latency spikes.
Median latency looks normal and error rates don’t meaningfully change. The slowdown affects only certain endpoints and mostly one region shortly after a deployment.

End-to-end investigation steps

Confirm alert context (p95/p99 vs baseline) and identify start time.

Compare latency by region to isolate blast radius.

Break down latency by endpoint to find impacted paths.

Inspect request logs for slow requests and patterns (endpoint, tenant, status).

Correlate the spike window with deployment events (version/time).

Validate hypothesis: slow requests correlate with higher db_time_ms / downstream time.

Mitigate: rollback to previous version or disable feature flag/config.

Prevent: propose an alert on tail latency per endpoint and DB time tails.

Datasets required

A. Application Request Logs

B. Service Latency Metrics

C. Deployment Events

Dataset schemas

A. Application Request Logs

Field	Type	Notes
timestamp	datetime	UTC
service_name	string	checkout-api
region	string	e.g., us-east-1, us-west-2, eu-west-1
request_id	string	missing in ~10%
endpoint	string	high cardinality
method	string	GET/POST
status_code	int	HTTP status
latency_ms	int	end-to-end
db_time_ms	int	time spent in DB calls
user_tier	string	free, pro, enterprise
log_level	string	INFO/WARN/ERROR

B. Service Latency Metrics (30s rollups)

Field	Type	Notes
timestamp	datetime	UTC
service_name	string	checkout-api
region	string	
p50_latency_ms	float	
p95_latency_ms	float	
p99_latency_ms	float	
request_count	int	

C. Deployment Events

Field	Type	Notes
deploy_time	datetime	UTC
service_name	string	checkout-api
version	string	e.g., 2025.12.02-rc3
change_summary	string	short description
rolled_back	boolean	
Key dataset characteristics

Tail-only regression: p95/p99 spike, p50 steady

Only 5–8% of requests slow

High-cardinality endpoints create expensive group-bys

Missing request_id complicates correlations

Impact concentrated in one region (partial rollouts)

db_time_ms increases without obvious DB errors (silent inefficiency)

Scale

Time range: 14 days (7 baseline + incident + recovery)

Logs: ~50M rows

Metrics: ~1M rows

Deploy events: <200 rows

What makes it hard

Global averages look healthy and the issue hides in tail latency, endpoint cardinality, and partial regional rollout.

Task 2: Trace a Cross-Service Latency Spike Using Distributed Telemetry Data
Scenario

A user workflow (“place order”) becomes slow end-to-end. No single service shows clear errors. Latency spikes are intermittent and appear only for certain request paths and tenants. Telemetry is partially sampled, and not all services emit complete trace context.

End-to-end investigation steps

Confirm user-facing symptom: end-to-end latency elevated for “place order” workflow.

Identify which services participate in the workflow (service graph / known call chain).

Compare service-level latency metrics to locate candidate bottlenecks.

Use tracing-style telemetry to attribute time across spans/services.

Filter to slow traces and locate the most frequent “dominant span.”

Correlate slow spans with downstream dependency metrics (DB/cache/queue).

Mitigate: adjust timeout, add caching, rollback change, or throttle expensive path.

Prevent: define an alert on dominant-span latency or dependency p95 + saturation.

Datasets required

A. Distributed Trace Spans

B. Service Request Metrics

C. Dependency Metrics (DB/cache/queue)

Dataset schemas

A. Trace Spans

Field	Type	Notes
timestamp	datetime	span start
trace_id	string	join key
span_id	string	
parent_span_id	string	
service_name	string	emitting service
operation	string	e.g., POST /orders
duration_ms	int	span duration
region	string	
status	string	OK, ERROR
tenant_id	string	high cardinality
sampled	boolean	only ~20% traces retained

B. Service Request Metrics (30s)

Field	Type	Notes
timestamp	datetime	
service_name	string	
region	string	
p95_latency_ms	float	
error_rate	float	
request_count	int	

C. Dependency Metrics

Field	Type	Notes
timestamp	datetime	
dependency_name	string	orders-db, pricing-cache, fraud-api
region	string	
p95_latency_ms	float	
timeout_rate	float	
saturation_percent	float	optional
Key dataset characteristics

Partial observability: only 20% traces sampled

Some spans missing parent linkage (broken context propagation)

Multi-region variability; one region has worse downstream latency

Tenant-specific effect: only certain tenants hit expensive code path

Heavy-tailed latency distributions and bursty traffic

Scale

Time range: 10 days

Spans: ~200M rows (high volume, sampled)

Metrics: ~2–5M rows depending on resolution/services

Dependencies: ~1M rows

What makes it hard

The bottleneck is hidden behind sampling, missing trace links, and multi-tenant variance—learners must triangulate across metrics + partial traces + dependency telemetry.

Task 3: Diagnose Resource Saturation Impacting Containerized Services
Scenario

user-profile-service intermittently times out in one availability zone. Users report slow reads and occasional failures. Error rate is initially low, but latency worsens during bursts. Global aggregates look fine; only AZ-level views reveal the problem.

End-to-end investigation steps

Validate the symptom: timeouts/latency spikes and start time.

Slice performance by zone/cluster to identify partial outage.

Check pod/node metrics for CPU, memory, throttling, and restarts.

Correlate infra saturation with application latency spikes.

Inspect application logs for timeouts, queueing, and retry patterns.

Identify root cause (CPU throttling, memory pressure, noisy neighbor, bad autoscaling).

Mitigate: scale replicas, adjust requests/limits, rebalance traffic across zones.

Prevent: add alerts for CPU throttling + p95 latency by zone and saturation early-warnings.

Datasets required

A. Container/Node Metrics

B. Application Request Logs

C. Kubernetes Event Logs (optional but helpful)

Dataset schemas

A. Container/Node Metrics (30s)

Field	Type	Notes
timestamp	datetime	
cluster	string	
availability_zone	string	
namespace	string	
pod	string	
container	string	
cpu_usage_percent	float	
cpu_throttle_percent	float	key signal
memory_working_set_mb	float	
oom_kill_count	int	
restart_count	int	

B. Application Request Logs

Field	Type	Notes
timestamp	datetime	
service_name	string	user-profile-service
region	string	
availability_zone	string	
pod	string	
request_id	string	missing ~5–10%
endpoint	string	
latency_ms	int	
status_code	int	
timeout_flag	boolean	
log_level	string	

C. Kubernetes Events

Field	Type	Notes
timestamp	datetime	
cluster	string	
availability_zone	string	
pod	string	
event_type	string	Evicted, BackOff, Killing
reason	string	
message	string	noisy
Key dataset characteristics

Failure localized to one AZ

CPU throttling ramps gradually before latency spikes

Global averages mask the issue

Noisy K8s events (many irrelevant warnings)

Bursty traffic causes nonlinear saturation effects

Scale

Time range: 7 days

Metrics: ~10–30M rows (pods x time)

Logs: ~20–60M rows

K8s events: ~0.5–2M rows

What makes it hard

The incident is invisible at global aggregation levels; learners must zoom into AZ/pod and correlate infra throttling with app latency under burst conditions.

Task 4: Debug Noisy Log Ingestion Issues in a Multi-Tenant Observability Platform
Scenario

Several tenants report missing logs or delayed log availability in the observability platform. Other tenants are unaffected. Overall system throughput looks fine, but certain tenants experience intermittent drops and large delays due to backpressure, quota enforcement, and uneven key distributions.

End-to-end investigation steps

Confirm tenant reports: identify which tenants and time windows.

Measure ingestion lag and drop rates by tenant/source.

Compare ingestion pipeline metrics across collectors/shards.

Inspect ingestion logs for throttling, retries, and backpressure signals.

Determine whether failures correlate with tenant volume spikes or cardinality explosions.

Identify root cause (hot partition, rate-limits, malformed payloads, bursty sources).

Mitigate: rebalance partitions, adjust quotas, tune batching/backoff, fix tenant schema.

Prevent: add per-tenant SLO dashboards and alerts on lag, drops, and hot keys.

Datasets required

A. Ingestion Pipeline Metrics

B. Collector / Ingestion Service Logs

C. Tenant Application Logs (sampled view of what should have arrived)

Dataset schemas

A. Ingestion Pipeline Metrics (30s)

Field	Type	Notes
timestamp	datetime	
tenant_id	string	high cardinality
collector_id	string	
shard_id	string	
incoming_records	int	
accepted_records	int	
dropped_records	int	
ingest_lag_seconds	float	key
queue_depth	int	
throttle_rate	float	
avg_payload_bytes	int	

B. Ingestion Service Logs

Field	Type	Notes
timestamp	datetime	
collector_id	string	
tenant_id	string	
event_type	string	THROTTLE, RETRY, PARSE_FAIL, BACKPRESSURE
error_code	string	
retry_count	int	
payload_schema_version	string	may change mid-day
message	string	semi-structured

C. Tenant “Source of Truth” Logs (expected)

Field	Type	Notes
timestamp	datetime	
tenant_id	string	
source_service	string	
log_id	string	unique
severity	string	
message	string	
Key dataset characteristics

Multi-tenant: some tenants burst 10–50x baseline

Hot partition/shard: one shard overloaded due to skewed keys

Schema evolution mid-day for a tenant (parse failures increase)

Ingestion lag spikes without obvious global throughput changes

Collector retries create duplicates; dedup needed

Scale

Time range: 14 days

Pipeline metrics: ~10–40M rows

Ingestion logs: ~50–150M rows

Tenant expected logs: ~5–20M rows (sampled)

What makes it hard

Overall system health looks fine; the incident is tenant-specific and driven by skew, quotas, schema drift, and retries that require careful correlation across ingestion metrics and logs.

Task 5: Investigate Suspicious Authentication Activity Using Security Logs
Scenario

Security alerts show a sharp increase in failed login attempts. Most failures are normal (forgotten passwords), but a subset indicates possible credential abuse. The activity spans regions and IP ranges and includes both bot-like and human-like patterns.

End-to-end investigation steps

Validate alert: confirm deviation from baseline (failed logins rate).

Segment failures by user_id, IP, ASN (if available), region, device fingerprint.

Detect patterns: password spraying vs brute force vs credential stuffing.

Correlate with successful logins and session creation events.

Identify targeted accounts/tenants and potential compromise indicators.

Mitigate: rate limits, CAPTCHA, temporary lockouts, block lists, MFA enforcement.

Prevent: define alerts on anomalous failure patterns (per IP/user/tenant) and dashboards.

Datasets required

A. Authentication Event Logs

B. Session / Token Issuance Logs

C. Geo/IP Enrichment Table (lookup dataset)

Dataset schemas

A. Authentication Event Logs

Field	Type	Notes
timestamp	datetime	
tenant_id	string	
user_id	string	high cardinality
ip	string	high cardinality
region	string	
device_id	string	high cardinality
auth_result	string	SUCCESS, FAILURE
failure_reason	string	bad_password, user_not_found, mfa_failed
attempt_count	int	per session/window
user_agent	string	noisy

B. Session / Token Logs

Field	Type	Notes
timestamp	datetime	
user_id	string	
tenant_id	string	
session_id	string	
ip	string	
mfa_used	boolean	
token_issued	boolean	

C. Geo/IP Enrichment (lookup)

Field	Type	Notes
ip_prefix	string	
asn	string	
isp	string	
country	string	
risk_score	float	optional synthetic
Key dataset characteristics

High noise: common user mistakes generate many failures

Attack signal is small: only 1–3% of events are malicious-like

Cardinality challenges: IP/user/device explode group-by costs

Attackers rotate IPs and user agents; some behave “low and slow”

Partial data: device_id missing for some clients; geo enrichment incomplete

Scale

Time range: 21 days

Auth logs: ~80–200M rows

Session logs: ~10–30M rows

Enrichment: ~50K–500K rows

What makes it hard

Learners must separate real threats from normal churn, handle extreme cardinality, and propose alerts that don’t flood with false positives.

Global synthetic data generation notes

All datasets are assumed to be synthetically generated to preserve realism:

Heavy-tailed latency (log-normal / Pareto-like tails)

Bursty traffic and diurnal seasonality

Partial observability: sampling, missing IDs, delayed metrics

Correlated failure modes: retries, backpressure, regional skew, schema evolution

Noise injection: irrelevant warnings/events that resemble real production clutter

Generation should preserve temporal correlation (cause precedes symptom) and cross-dataset joinability via keys like tenant_id, request_id, trace_id, pod, region, and aligned timestamps (with small clock skew).
