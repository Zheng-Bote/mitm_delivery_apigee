# MitM Apigee Delivery Layer (CLI Batch Job)

This directory contains the Go implementation of the MitM Delivery Layer. It serves as the final stage of the Data Aggregator pipeline, reliably pushing transformed JSON packages from PostgreSQL into external target systems.

## Overview

The Delivery Layer is designed for extreme resilience and high concurrency:

- **Idempotency:** Every delivery attempt is bound to an `Idempotency-Key` (UUID) to prevent duplicated records if network timeouts occur.
- **Retry Engine:** Implements Exponential Backoff for transient errors (e.g., `HTTP 429 Too Many Requests` or `HTTP 503 Service Unavailable`).
- **Dead Letter Queue (DLQ):** Permanently failing deliveries (e.g., `HTTP 400 Bad Request` or exhausting retry limits) are safely moved into the `dead_letter_queue` table to prevent queue blocking.
- **Interchangeable Adapters:** Extensible architecture using Strategy Pattern to support multiple target platforms.

## Supported Adapters

1. **APIGEE Adapter (`APIGEE`)**: Enterprise gateway support featuring Mutual TLS (mTLS), JWT injection, and specific routing headers.

## Building

To build the executable, run:

```bash
go build -o bin/mitm-apigee ./cmd/apigee/main.go
```

## Usage

The `mitm-apigee` module is executed as a short-lived batch job by the `mitm_scheduler`. It expects:

1. **IPC Scheduler Connection** (Preferred) to dynamically fetch PostgreSQL credentials and the `MASTER_KEY`, or direct **Environment Variables** (Fallback).
2. **A single JSON Argument (`os.Args[1]`)** specifying the Job constraints and the `Topic`.

### Example Execution

```bash
# 1. IPC Execution (Running under Scheduler - Preferred)
export RUN_ID="123"
export SCHEDULER_SOCKET_PATH="/tmp/scheduler.sock"

# OR via Direct Environment Variables (Fallback for standalone execution)
export MITM_DB_CONFIG_JSON='{"db":{"host":"192.168.7.31","port":5432,"user":"mitm_user","password":"...","database":"mitm"}}'
export MASTER_KEY="<base64_encryption_key>"

# 2. Define the Job parameters
ARGS_JSON='{
  "topic": "Employee",
  "workers": 5,
  "batch_size": 200,
  "max_retries": 5,
  "source_name": "DELIVERY",
  "blocking_jobs": ["mitm_deliver_Org"]
}'

./bin/mitm-apigee "$ARGS_JSON"
```

The Delivery job will automatically connect to the database (fetching credentials via IPC if running under the scheduler), query the `delivery_targets` table for the target configuration matching the `Topic` ("Employee"), decrypt the `config_payload` using `MASTER_KEY`, and dynamically instantiate the correct adapter.

### Config Payload Examples (Database / Admin UI)

When configuring the Target via the Admin Frontend, the `Config Payload` JSON differs per adapter.

#### APIGEE mTLS (Adapter: APIGEE)

If you route via APIGEE, the payload might look like:

```json
{
  "client_cert_path": "/opt/certs/client.crt",
  "client_key_path": "/opt/certs/client.key",
  "jwt_token": "...",
  "routing_key": "vendor-x"
}
```

## Database Interaction

The job interacts strictly with the tables predefined in `./migrations`:

- `packages`: Pulls pending/retryable records.
- `dead_letter_queue`: Safely unloads fatal records.

Concurrency is handled natively in Postgres using `FOR UPDATE SKIP LOCKED`.
