# OpenTelemetry Pizza Workshop

A small pizza-ordering app built from three Node services and a web frontend.
It is instrumented with OpenTelemetry and sends traces, metrics and logs to
Dash0 through an OpenTelemetry Collector — see
[pizza-app/README.md](pizza-app/README.md#telemetry) for how that is wired up.
This file only covers getting the app running.

## Prerequisites

- **Docker Desktop** — [download here](https://www.docker.com/products/docker-desktop)
- **Node.js** v18 or higher — [download here](https://nodejs.org/) (only needed
  once you start changing the services)
- Ports `3000`, `3001`, `3002`, `4317`, `4318` and `8080` free
- A Dash0 auth token and your region's OTLP endpoint
  ([Settings → Auth Tokens](https://app.dash0.com/settings/auth-tokens),
  [Settings → Endpoints](https://app.dash0.com/settings/endpoints))

## Get the code

**Fork first, then clone your fork.** Later stages of the workshop open pull
requests against your repository, so you need to own the remote.

1. Open <https://github.com/dash0-community/otel-pizza-workshop> and click **Fork**.
   Keep the default name.

2. Clone your fork and keep a link to this repository:

```bash
# replace YOUR-USERNAME with your GitHub username
git clone https://github.com/YOUR-USERNAME/otel-pizza-workshop.git
cd otel-pizza-workshop

git remote add upstream https://github.com/dash0-community/otel-pizza-workshop.git
git remote -v
```

## Run it

```bash
cd pizza-app
cp .env.template .env   # then fill in DASH0_AUTH_TOKEN and DASH0_ENDPOINT
docker compose up
```

The first build takes a few minutes. When all five containers are up, open
<http://localhost:8080> and order a pizza. The order shows up in Dash0 as a
single trace spanning the browser and all three services.

Stop with `Ctrl+C`, or:

```bash
docker compose down            # stop
docker compose down -v         # and volumes
docker compose down --rmi all  # and images
```

## What's running

| Service | Port | Does |
|---|---|---|
| Frontend | 8080 | Order form |
| Order Service | 3000 | Takes the order, calls the other two |
| Kitchen Service | 3001 | Checks availability, cooks |
| Delivery Service | 3002 | Assigns a driver |
| OTel Collector | 4317/4318 | Forwards telemetry to Dash0 |

Logs from all four are interleaved in the terminal you ran `docker compose up`
in. For one service on its own:

```bash
docker compose logs -f order-service
```

## Failure modes you can switch on

```bash
SLOW_KITCHEN=true docker compose up   # the oven takes ~5s per pizza
NO_DRIVERS=true docker compose up     # nobody is available to deliver
```

## Troubleshooting

**Containers won't start** — check Docker Desktop is running, then
`docker compose down && docker compose up --build`.

**Port already in use** — something else holds 3000, 3001, 3002 or 8080.
`lsof -i :3000` will name it.

**Changed a file and nothing happened** — the services are baked into images.
Rebuild: `docker compose up -d --build`.

**Build fails** — `docker compose build --no-cache`, and check the JSON in any
`package.json` you edited.

## Credits

The pizza app and the original workshop are the work of
[Julia Morgado](https://github.com/juliafmorgado/otel-pizza-workshop).

## License

MIT License — feel free to use this for learning!
