# Pizza Order Tracker

A pizza ordering system built from three microservices and a web frontend.

## What's Inside

- **Order Service** (Port 3000): Receives pizza orders and coordinates with other services
- **Kitchen Service** (Port 3001): Checks availability and cooks pizzas
- **Delivery Service** (Port 3002): Assigns drivers for delivery
- **Frontend** (Port 8080): Simple web UI for ordering pizzas
- **OpenTelemetry Collector** (Ports 4317/4318): Receives telemetry from the
  services and the browser, and forwards it to Dash0

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

First create your `.env` (it holds your Dash0 token and is gitignored):

```bash
cp .env.template .env
```

Fill in `DASH0_AUTH_TOKEN` and `DASH0_ENDPOINT`, then:

```bash
docker compose up
```

Then open http://localhost:8080 and order a pizza.

To stop it:

```bash
docker compose down
```

## Telemetry

The app is instrumented with OpenTelemetry and ships traces, metrics and logs to
Dash0. There is no OpenTelemetry code in the services — instrumentation is
attached at startup instead.

### How it fits together

```
Browser ──────────────┐
  (Dash0 Web SDK)     │
                      ▼
Order ──▶ Kitchen  ┌──────────────┐        ┌────────┐
  │                │ OTel         │───────▶│ Dash0  │
  └──▶ Delivery ──▶│ Collector    │  OTLP  └────────┘
     (auto-instr.)  └──────────────┘
```

### What produces the data

- **The three Node services** load
  `@opentelemetry/auto-instrumentations-node` through `node --require` in their
  `start` script. That patches Express, the HTTP client and Pino at runtime, so
  every inbound request, every outbound call between services and every log line
  becomes telemetry on its own. `traceparent` headers are propagated
  automatically, which is what stitches one order into a single trace across all
  three services.
- **Pino logs** keep printing to stdout exactly as before, and additionally get
  `trace_id` and `span_id` attached, so a log line links back to the request it
  came from.
- **The frontend** uses the Dash0 Web SDK, added as a script tag in
  `frontend/index.html`. It reports page views, Core Web Vitals and browser
  errors, and propagates trace context to the Order Service so a browser click
  and the backend work it caused end up in one trace.
- **The Collector** (`otelcol-config.yaml`) is the only component that holds
  credentials. The browser sends its telemetry to the Collector rather than
  straight to Dash0, which keeps the auth token out of the page source.

### Configuration

Everything is driven by environment variables from `.env`, which
`docker-compose.yml` passes through:

| Variable | Purpose |
|---|---|
| `DASH0_AUTH_TOKEN` | Dash0 ingest token. Required. |
| `DASH0_ENDPOINT` | Your region's OTLP/gRPC endpoint. Required. |
| `DASH0_DATASET` | Dataset to ingest into. Defaults to `default`. |
| `DASH0_ENVIRONMENT` | Sets `deployment.environment.name`. Defaults to `workshop`. |
| `VCS_REPOSITORY_URL` | Sets `vcs.repository.url.full`, linking telemetry to this repo. |
| `OTEL_SDK_DISABLED` | Set to `true` to run the app without telemetry. |

### Checking it works

```bash
docker compose logs -f otel-collector
```

A Collector that starts cleanly and stays quiet is exporting successfully. If the
token or endpoint is wrong it logs export failures here, which is the first place
to look when nothing shows up in Dash0.

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
- **Docker** - Containerization

## Ports

- `3000` - Order Service
- `3001` - Kitchen Service
- `3002` - Delivery Service
- `8080` - Frontend
