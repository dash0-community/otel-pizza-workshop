# Pizza Order Tracker

A pizza ordering system built from three microservices and a web frontend.

## What's Inside

- **Order Service** (Port 3000): Receives pizza orders and coordinates with other services
- **Kitchen Service** (Port 3001): Checks availability and cooks pizzas
- **Delivery Service** (Port 3002): Assigns drivers for delivery
- **Frontend** (Port 8080): Simple web UI for ordering pizzas

## Architecture

```
┌─────────────┐
│   Browser   │
│  (Port 8080)│
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Order     │────▶│   Kitchen   │     │  Delivery   │
│  Service    │     │   Service   │     │   Service   │
│ (Port 3000) │     │ (Port 3001) │     │ (Port 3002) │
└─────────────┘     └─────────────┘     └─────────────┘
```

## Running the App

```bash
docker compose up
```

Then open http://localhost:8080 and order a pizza.

To stop it:

```bash
docker compose down
```

## Watching What Happens

The terminal shows all four services interleaved:

```
order-service    | {"level":30,...,"orderId":"PIZZA-123...","msg":"Order received"}
kitchen-service  | {"level":30,...,"orderId":"PIZZA-123...","msg":"Starting to cook"}
delivery-service | {"level":30,...,"orderId":"PIZZA-123...","msg":"Assigning driver"}
```

One service on its own:

```bash
docker compose logs -f kitchen-service
```

## Sending Telemetry to Dash0

All three Node services are instrumented with OpenTelemetry. There is no
tracing code in the application — `@opentelemetry/auto-instrumentations-node`
is loaded through `NODE_OPTIONS` before the app starts and patches Express,
HTTP, Axios and pino on its own.

Set your credentials up once:

```bash
cp .env.template .env
# then edit .env and paste your Dash0 auth token
```

You need `DASH0_AUTH_TOKEN` and `DASH0_ENDPOINT`. Both come from
https://app.dash0.com → Settings (Auth Tokens, and Endpoints for the OTLP/gRPC
address of your region). Then:

```bash
docker compose up --build
```

Order a pizza at http://localhost:8080 and the order shows up in Dash0 as a
single trace across all three services.

### What you get

- **Traces** — one trace per order, spanning order → kitchen → delivery, with
  the HTTP calls between them linked automatically via `traceparent` headers.
- **Logs** — the existing pino output, shipped as OTLP and stamped with the
  `trace_id` and `span_id` of the request that produced it, so you can jump
  from a log line to its trace and back.
- **Metrics** — HTTP server/client and Node.js runtime metrics.

Services appear as `order-service`, `kitchen-service` and `delivery-service`
under the `pizza-app` service namespace.

### Running without Dash0

```bash
OTEL_SDK_DISABLED=true docker compose up
```

The app behaves exactly as before and no telemetry is produced.

### Notes

- The Docker health checks poll `/health` every 5s, so expect a steady trickle
  of `GET /health` spans alongside the order traces.
- The browser frontend is not instrumented; traces start when the order reaches
  the Order Service.
- Telemetry goes straight from each service to Dash0. For production you would
  normally put an OpenTelemetry Collector in between to handle batching,
  retries and filtering.

## Failure Modes You Can Switch On

### Slow Kitchen (Oven is Broken)
```bash
SLOW_KITCHEN=true docker compose up
```

Every pizza takes about five seconds longer to cook.

### No Drivers Available
```bash
NO_DRIVERS=true docker compose up
```

Delivery has nobody to assign, so orders fail.

## Services Overview

### Order Service
- Receives orders from the frontend
- Calls Kitchen Service to check availability and cook
- Calls Delivery Service to assign a driver
- Returns order confirmation

### Kitchen Service
- Checks if kitchen is available
- Simulates cooking time
- Can be configured to be slow (SLOW_KITCHEN=true)

### Delivery Service
- Finds available drivers
- Assigns driver to order
- Can be configured to have no drivers (NO_DRIVERS=true)

### Frontend
- Simple HTML form
- Sends orders to Order Service
- Displays confirmation

## Tech Stack

- **Node.js** - Runtime
- **Express** - Web framework
- **Axios** - HTTP client
- **OpenTelemetry** - Traces, metrics and logs (zero-code instrumentation)
- **Docker** - Containerization

## Ports

- `3000` - Order Service
- `3001` - Kitchen Service
- `3002` - Delivery Service
- `8080` - Frontend
