# web-raytracer

A **distributed ray tracing web service** with a custom load balancer.

Users log in, choose a 3D scene and quality, and get a rendered image. Rendering is CPU-heavy, so tasks are distributed across several C# worker servers by a self-written **least-loaded** load balancer. Users watch progress live, can cancel tasks, and browse their history.

This is the main repository. It ties the services together as **git submodules** and holds the shared `docker-compose.yml`.

---

## Services

| Folder | Repository | Tech | Role |
|---|---|---|---|
| `client/` | [web-client](https://github.com/<you>/web-client) | React, TypeScript, Vite | UI: render form, live progress, history, admin panel |
| `load-balancer/` | [web-load-balancer](https://github.com/<you>/web-load-balancer) | Python, FastAPI | Gateway: auth, limits, least-loaded balancing, queue, history, WebSocket progress, autoscaling |
| `ray-worker/` | [web-ray-worker](https://github.com/<you>/web-ray-worker) | C#, ASP.NET Core | Renders images on all CPU cores and reports progress (2+ instances) |
| — | `postgres` image | PostgreSQL | `web-db`: users and task history |

Each service has its own README with details.

## Architecture

```
                 ┌──────────────┐
  Browser ──────▶│  web-client  │  (static React app)
                 └──────┬───────┘
                        │ HTTP + WebSocket
                 ┌──────▼────────────┐        ┌─────────┐
                 │ web-load-balancer │───────▶│ web-db  │
                 │     (FastAPI)     │        │Postgres │
                 └──┬──────┬──────┬──┘        └─────────┘
       least-loaded │      │      │
              ┌─────▼─┐ ┌──▼────┐ ┌▼──────┐
              │worker1│ │worker2│ │workerN│  web-ray-worker (C#)
              └───────┘ └───────┘ └───────┘  2 minimum, autoscaled
```

### Request flow

1. The user submits a render (scene, resolution, samples per pixel) in **web-client**.
2. **web-load-balancer** checks auth and limits, saves the task in **web-db**, then:
   - sends it to the worker with the lowest `running / capacity`, or
   - puts it in a queue with an estimated wait time if every worker is full.
3. **web-ray-worker** renders row by row in parallel, reporting progress.
4. The load balancer relays progress to the browser over WebSocket and stores the final image.
5. The user sees the result in their history. The admin panel shows the load on every worker.

---

## Assignment requirements

| # | Requirement | How it is met |
|---|---|---|
| 1 | Limit task complexity | Max resolution, samples per pixel, `pixels × spp` and render time. Checked in the client and enforced in the load balancer and workers |
| 2 | Show progress | Percentage of rendered rows plus a live partial image, sent over WebSocket |
| 3 | History, status, cancel, new task, limited count, DB storage | History page, task view, cancel button, at most 3 active tasks per user, PostgreSQL |
| 4 | Authorization (HTTPS bonus) | JWT login. HTTPS is in the backlog |
| 5 | **Load balancing over ≥ 2 servers** | Self-written least-loaded balancer in FastAPI with health checks, over 2+ C# workers |
| 6 | *Bonus:* queue and wait estimate | FIFO queue, ETA from the average render time |
| 7 | *Bonus:* horizontal autoscaling | Load balancer starts and stops worker containers through the Docker API, within min/max bounds |
| 8 | *Bonus:* admin panel | Workers, their load and status, plus all running and queued tasks |

---

## Getting started

### Requirements

- Git
- Docker Desktop (or Docker Engine + Compose)
- For local development without Docker: .NET 8 SDK, Python 3.12, Node 20

### Clone

```bash
git clone --recurse-submodules https://github.com/<you>/web-raytracer
cd web-raytracer
```

If you already cloned without submodules:

```bash
git submodule update --init --recursive
```

### Run everything

```bash
cp .env.example .env      # set JWT_SECRET, DB password
docker compose up --build
```

| URL | What |
|---|---|
| http://localhost:8080 | App (client served by nginx, which proxies `/api` to the load balancer) |
| http://localhost:8000 | Load balancer API directly |
| http://localhost:8000/docs | Load balancer API docs |

Two workers always run. The load balancer starts more containers (`ray-worker-3`, `ray-worker-4`, ...) when tasks queue up and stops them once they are idle, between `AUTOSCALE_MIN` and `AUTOSCALE_MAX` from `.env` (2 and 5 by default). For that it gets the Docker socket mounted in `docker-compose.yml`.

### Workers on other machines

The load balancer only talks to workers over HTTP, so a worker can run on any machine it can reach.

**On the worker machine** (Docker installed, same network), only the worker repo is needed. It has its own `docker-compose.yml`:

```bash
git clone https://github.com/<you>/web-ray-worker
cd web-ray-worker
docker compose up --build -d
```

This starts one worker and publishes its port 8080. Its name, port and capacity can be set in a `.env` file there (see the worker README). Check it from the main machine with `curl http://<worker-ip>:8080/health`. On Windows, allow inbound TCP 8080 in the firewall if the request times out.

**On the main machine**, list it in `.env` next to the local workers and restart the load balancer:

```bash
RENDERERS=http://ray-worker-1:8080,http://ray-worker-2:8080,http://<worker-ip>:8080
docker compose up -d load-balancer
```

The remote worker shows up in the admin panel and gets tasks like any other worker. The autoscaler never stops it, because it only manages containers it started itself, but it does count toward `AUTOSCALE_MAX`.

---

## docker-compose overview

```yaml
services:
  web-db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: web_db
    volumes: [db-data:/var/lib/postgresql/data]

  load-balancer:
    build: ./load-balancer
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql+asyncpg://app:${DB_PASSWORD}@web-db:5432/web_db
      JWT_SECRET: ${JWT_SECRET}
      AUTOSCALE_MIN: 2
      AUTOSCALE_MAX: 5
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock   # for autoscaling
    depends_on: [web-db, ray-worker]

  ray-worker:
    build: ./ray-worker
    deploy:
      replicas: 2
    environment:
      Worker__Capacity: 2

volumes:
  db-data:
```

*(This is a sketch. The real file lives at the repo root.)*

---

## Working with submodules

- **Changing a service:** go into its folder, check out `main` (`git checkout main`), then commit and push there as usual. That updates the service's own repository.
- **Recording new versions in this repo:** at a milestone, run:
  ```bash
  git submodule update --remote
  git add client load-balancer ray-worker
  git commit -m "Update services"
  git push
  ```
- **Changing shared files** (`docker-compose.yml`, this README): commit them in this repo.

## Repository structure

```
web-raytracer/
├── client/              → submodule: web-client
├── load-balancer/       → submodule: web-load-balancer
├── ray-worker/          → submodule: web-ray-worker
├── docker-compose.yml
├── .env.example
├── .gitmodules
└── README.md
```

## Known limitations

- The load balancer is a **single point of failure**. If it stops, the system is unavailable even though workers keep running.
- HTTPS is not enabled yet (backlog).

## Roadmap

- [x] Ray tracer renders a PNG from the console
- [x] Worker API with progress and cancel, running in Docker (2 instances)
- [x] Load balancer: health checks, least-loaded dispatch, history in web-db
- [x] docker-compose brings up the whole system
- [x] React client: render form, live progress, history
- [x] JWT auth
- [x] Queue with wait estimate
- [x] Autoscaling
- [x] Admin panel
- [ ] *Backlog:* HTTPS
