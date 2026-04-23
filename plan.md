# Offline Log Ingestion Architecture (NiFi → OpenTelemetry → SigNoz → ClickHouse)

This document describes the recommended architecture for **offline log ingestion workflows**, especially for environments where logs are delivered as bundles from multiple teams and devices for **root-cause analysis (RCA)**.

Pipeline structure:

```
NiFi / Vector
   ↓
OpenTelemetry Collector
   ↓
SigNoz
   ↓
ClickHouse
```

This design avoids requiring agents on edge devices and works fully in **air‑gapped deployments**.

---

# What “Offline Logs” Means in This System

Offline logs are delivered manually or batch‑uploaded rather than streamed in real time.

Typical delivery patterns include:

* USB transfer
* network shared folder drops
* email attachments
* SFTP batch uploads
* object‑storage uploads
* periodic ZIP bundles
* manual landing‑zone folder copy

Example structure:

```
train_A/
   subsystem_display.log
   camera_module.txt
   diagnostics.zip
   gateway_events.log
```

---

# Landing‑Zone Ingestion Model

Recommended intake pattern:

```
Offline drop location
(shared folder / NAS / object store / SFTP)
        ↓
NiFi intake workflow
        ↓
Parsing + metadata tagging
        ↓
OpenTelemetry Collector
        ↓
SigNoz
        ↓
ClickHouse storage
```

This enables centralized processing without installing collectors on edge systems.

---

# Step‑by‑Step Offline Pipeline Behaviour

## Step 1 — Landing Zone Detection

NiFi monitors:

```
/offline_logs/intake/
```

Example arriving file:

```
trainA_20260422.zip
```

NiFi automatically:

* detects
* validates
* checksum‑verifies
* extracts
* routes
* tags metadata

Typical processors used:

```
ListFile
GetFile
FetchSFTP
UnpackContent
RouteOnAttribute
```

---

## Step 2 — Archive Extraction (ZIP Bundles)

NiFi expands archive contents automatically:

```
trainA_logs.zip
```

into:

```
display.log
camera.txt
controller.log
```

No manual preprocessing required.

---

## Step 3 — Metadata Tagging (Critical for Correlation)

Because offline logs lack live telemetry context, enrichment must occur early.

Example metadata attributes attached by NiFi:

```
train_id
car_id
device_id
team_source
upload_batch_id
upload_timestamp
file_origin
```

Example values:

```
train_id = TRAIN_A12
device = display_unit_07
batch = RCA_session_2026_04_22
```

These attributes become correlation keys downstream.

---

## Step 4 — Parsing Different Formats

NiFi (or optionally Vector) supports:

| Format | Handling                   |
| ------ | -------------------------- |
| .log   | regex / structured parsing |
| .txt   | pattern extraction         |
| .json  | native parsing             |
| .zip   | automatic extraction       |
| .gz    | automatic decompression    |

Multiline stack traces are also supported:

```
ERROR:
Traceback:
line 1
line 2
```

---

## Step 5 — Normalization via OpenTelemetry Collector

OpenTelemetry Collector converts heterogeneous logs into a unified schema:

```
timestamp
severity
body
attributes.train_id
attributes.device_id
attributes.component
attributes.batch_id
attributes.trace_id (optional)
```

This enables cross‑device correlation.

---

## Step 6 — Storage and Correlation

SigNoz stores normalized logs inside:

```
ClickHouse
```

Capabilities enabled:

* subsystem‑level search
* incident timeline construction
* failure propagation tracing
* device grouping
* train filtering
* batch comparison

Example RCA query:

```
train_id = TRAIN_A12
AND timestamp between 10:00 and 10:05
AND severity = ERROR
```

---

# Example Offline RCA Workflow (Train Scenario)

Engineer uploads:

```
trainA_logs.zip
```

Pipeline automatically:

```
extract
→ tag train_id
→ parse logs
→ normalize schema
→ store in ClickHouse
→ index in SigNoz
```

Investigator workflow:

```
filter train = TRAIN_A12
filter subsystem = display
open timeline view
```

Example result:

```
camera failure
→ controller timeout
→ display blank
```

Full causal chain becomes visible.

---

# Optional Enhancement — Batch Tracking

Add attribute:

```
batch_id = investigation_session_2026_04_22
```

Enables comparison scenarios:

```
before firmware patch
vs
after firmware patch
```

or

```
before maintenance
vs
after maintenance
```

Highly useful for engineering RCA validation workflows.

---

# When Vector Is Optional vs Necessary

Use NiFi only when:

* logs arrive as bundles
* ingestion is manual or batch‑based
* multiple teams submit archives
* archive‑heavy workflows dominate

Add Vector when:

* very high parsing throughput required
* complex regex transforms needed
* multiline stack traces frequent
* semi‑structured logs dominate

Typical enterprise offline stack:

```
NiFi → OpenTelemetry Collector → SigNoz
```

Vector remains optional but beneficial for heavy parsing workloads.

---

# Constraints of Offline Pipelines

Offline pipelines cannot assume live telemetry context.

You must explicitly define:

```
timestamp format
timezone normalization
device identity mapping
session boundaries
correlation keys
```

Otherwise correlation quality degrades.

---

# Recommended Metadata Schema for Train Diagnostics Platforms

Attach the following attributes early in the pipeline:

```
train_id
car_id
device_id
subsystem
component
firmware_version
log_source_team
batch_id
capture_time
upload_time
```

This converts raw offline logs into structured telemetry datasets suitable for RCA analytics.

---

# Final Architecture Summary

Recommended offline ingestion architecture:

```
Offline folder / SFTP / object storage
        ↓
NiFi (detect + unzip + classify + metadata tagging)
        ↓
OpenTelemetry Collector (schema normalization)
        ↓
SigNoz (correlation + investigation UI)
        ↓
ClickHouse (storage + analytics engine)
```

This architecture represents a strong open‑source approach (2026) for offline event correlation across heterogeneous device logs in air‑gapped environments.
