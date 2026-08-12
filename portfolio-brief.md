# TransitPulse BC — portfolio page brief

Handoff document for building a project page. Everything below is measured, not
estimated, unless marked otherwise.

---

## One-liner

A streaming lakehouse on AWS that ingests Metro Vancouver's live transit feed —
about 16 million events a day — and predicts bus arrival delay more accurately
than the published schedule.

## Positioning

Most AWS portfolio projects are a notebook reading a CSV from S3. Five things
separate this one, and the page should lead with them:

1. **The data is live and messy.** Protobuf over HTTP, refreshing every ~30
   seconds, with missing fields, duplicates, and vehicles that vanish mid-route.
   During the build, TransLink's vehicle-location system went dark for over an
   hour — the pipeline kept running and logged it.
2. **There is a real baseline.** The published schedule. Beating its mean
   absolute error is a claim someone can check, rather than an unfalsifiable
   accuracy number.
3. **Ground truth arrives late.** You predict now; the truth appears 10–40
   minutes later. That forces a proper backtest rather than a train/test split.
4. **Online/offline feature parity.** Training features are built in Spark over
   Parquet; serving features come from DynamoDB in under 50 ms. An automated test
   asserts the two paths agree.
5. **It has a cost story with real numbers**, not an architecture diagram with a
   hand-wave.

---

## Status — be accurate about this

| Stage | State |
|---|---|
| Infrastructure (~120 AWS resources, all Terraform) | **complete** |
| Streaming ingestion | **running continuously** |
| Lakehouse: bronze → silver → gold | **complete, verified end to end** |
| Data quality gate + orchestration | **complete** |
| Model training and evaluation | **pending — needs ~3 weeks of collected data** |
| Prediction API | **deployed, not yet backed by a trained model** |

The page should describe this as a working data platform with the modelling
layer in progress. **Do not present model accuracy numbers — none exist yet.**
Leave a clearly-marked placeholder for the results table.

---

## Project map — the architecture

```
TransLink GTFS-Realtime API
        │  every 60 seconds
        ▼
   Lambda: poller ──────▶ Kinesis Data Streams (1 shard, 24h retention)
                                │
                ┌───────────────┴───────────────┐
                ▼                               ▼
        Kinesis Firehose                 Lambda: online features
        (buffers 5 min,                  (aggregates in memory)
         JSON → Parquet)                        │
                │                               ▼
                ▼                          DynamoDB
          S3 bronze                    (live state, 2h TTL)
                │                               │
                │  nightly 02:20 UTC            │
                ▼                               │
        Step Functions                          │
          ├─ Glue: silver  (Iceberg, MERGE)     │
          ├─ Glue: data quality gate            │
          │    └─ fail → quarantine + alert     │
          └─ Glue: gold    (features)           │
                │                               │
                ▼                               │
          S3 gold ──▶ SageMaker Pipeline        │
                       train → evaluate →       │
                       gate → registry          │
                              │                 │
                              ▼                 │
                    Serverless endpoint ◀───────┘
                              ▲
                              │
                   API Gateway → Lambda
                   GET /v1/predict
```

**The single most important structural point:** the stream splits at Kinesis into
two independent paths.

- **Live path** — answers "how late is this bus right now" in milliseconds
- **Batch path** — answers "what patterns exist across millions of past arrivals"

Two consumers reading the same records is the entire justification for using
Kinesis Data Streams rather than Firehose alone.

---

## Measured numbers

Use these; they are all real.

| Metric | Value |
|---|---|
| Events ingested per day | ~16,000,000 |
| Rows per poll | 11,000–21,000 |
| Raw predictions collapsed per stop event | **25 : 1** (2,021,771 → 79,945) |
| Duplicate keys after dedupe | **0** |
| Realtime ↔ static schedule join validation | 1,521,008 rows matched |
| Model features | 26 defined, 20 currently populated |
| AWS resources, all Terraform-managed | ~120 |
| EC2 instances | **0** |
| Running cost | ~$0.70/day (~$21/month) |
| Offline test suite | 22 passing, no AWS account required |

---

## Tech stack

**Ingestion** — AWS Lambda (Python 3.12), Kinesis Data Streams, Kinesis Firehose,
GTFS-Realtime protobuf, Secrets Manager

**Storage** — S3 medallion architecture (bronze/silver/gold), Apache Iceberg,
Parquet, Glue Data Catalog, Athena with partition projection

**Processing** — AWS Glue (PySpark), Step Functions, EventBridge

**ML** — SageMaker Pipelines, XGBoost, Model Registry, Serverless Inference

**Serving** — DynamoDB, API Gateway (HTTP API), Lambda

**Infrastructure** — Terraform (7 modules), GitHub Actions with OIDC federation,
CloudWatch alarms and dashboards

---

## Engineering decisions worth featuring

Pick two or three; don't list all of them.

**No NAT Gateway.** The only component needing outbound internet is the poller,
and a Lambda not attached to a VPC gets internet free. Two gateway VPC endpoints
(S3, DynamoDB) cost nothing and replace a ~$32/month appliance — which would have
been the single largest line item.

**Serverless inference over a real-time endpoint.** ~$0 idle versus ~$96/month
for the smallest always-on instance. The trade is a 1–3 second cold start,
documented rather than hidden.

**Kinesis write pacing instead of a second shard.** At ~21,000 records per poll
the function exceeded the shard's 1,000 records/sec write limit, causing
throttling, failed invocations, and lost polls. The obvious fix is a second shard
at $11/month. But the Lambda had a 120-second timeout and was finishing in 4.5
seconds — the capacity was already paid for. Pacing writes to 900/sec took
throttling to zero at no cost.

**Quality failure and infrastructure failure take different branches.** A failing
data-quality check routes to quarantine: the gold layer is never rebuilt, the bad
partition stays inspectable, an alert fires. A failing job routes to a generic
failure path. Treating them identically would either crash the pipeline on
recoverable data issues or silently promote bad data on retry.

**Point-in-time correctness.** Historical aggregates use service dates *strictly*
earlier than the run date. Include the current day and the label leaks into the
features, producing metrics that look excellent and mean nothing.

---

## The best story on the page

Athena **partition projection** computes partition paths from a formula instead of
registering them in a catalog — no crawler, no `MSCK REPAIR`, no crawler cost. It
is the right choice for Athena.

But projection is an Athena-only feature. When the Spark job read the same table,
it asked the Glue catalog for a partition list, received an empty one, read zero
files, and reported **SUCCEEDED in 73 seconds**.

Two engines reading the same table definition: one found 1.5 million rows, the
other found nothing, and neither raised an error. Fixed by addressing the S3
prefix directly, since Firehose writes a deterministic path.

That class of bug — succeeds loudly, produces nothing — is what the project's
verification discipline exists to catch.

---

## Assets available

| Asset | Path | Use |
|---|---|---|
| Step Functions state machine graph | `img/stepfunctions_graph.png` | Architecture section — shows the quarantine branch clearly |
| Full technical brief | `docs/interview-brief.html` | Link as "read the technical write-up" |
| Architecture decision records | `docs/adr/` | Link for depth |

**Still to capture** (page should leave room): CloudWatch dashboard screenshot,
Cost Explorer figure, model results table, a short demo video.

---

## Tone guidance

- **State limitations plainly.** Six of 26 features are currently stubs; weather
  isn't wired up yet. Saying so reads as competence, not weakness.
- **Prefer specific numbers to adjectives.** "25:1 collapse ratio" beats
  "efficiently deduplicated."
- **Don't claim the model works yet.** It hasn't been trained.
- Avoid "leveraged", "utilized", "cutting-edge", "robust". The numbers carry it.

## Suggested page structure

1. Hero — one-liner, the live-messy-data hook, key numbers strip
2. Architecture — the project map diagram, the two-path split explained
3. What makes it different — the five positioning points, condensed to three
4. Engineering decisions — two or three, with the cost figures
5. The debugging story — partition projection, or the Kinesis throttling fix
6. Stack — grouped by layer
7. Status and what's next — honest about the modelling stage
8. Links — repo, technical brief
