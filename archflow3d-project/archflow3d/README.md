# ArchFlow 3D

A browser-based 3D designer for software architecture. Drop servers, databases, microservices, clients, networks and AWS / Google Cloud / Azure blocks onto a 3D stage, connect them, and simulate how API requests and responses travel through the system. Export sharp PNG stills (up to 8K) or animated GIFs (up to 1600 px wide).

Built with **React 18** for the interface and **three.js** for the 3D scene. No build step: the whole app is one `index.html`.

## How to run

There are three ways. Pick one.

### Option 1 — Docker (recommended)

Needs [Docker](https://docs.docker.com/get-docker/). The image bundles all JavaScript libraries, so it runs without internet access once built.

```bash
cd archflow3d
docker compose up -d --build
```

Open **http://localhost:8080**. Stop it with `docker compose down`.

Without Compose:

```bash
docker build -t archflow3d .
docker run -d --name archflow3d -p 8080:80 archflow3d
```

To use another port, change the left number, for example `-p 3000:80` (or `"3000:80"` in `docker-compose.yml`).

Useful commands:

| Task | Command |
| --- | --- |
| Check it's healthy | `docker ps` (STATUS shows `healthy`) or `curl http://localhost:8080/healthz` |
| See logs | `docker logs -f archflow3d` |
| Rebuild after editing `index.html` | `docker compose up -d --build` |
| Remove | `docker compose down --rmi local` |

### Option 2 — any static web server

```bash
cd archflow3d
npx serve .                   # Node.js
# or
python3 -m http.server 8080   # Python
```

Open the printed URL (for Python, http://localhost:8080). The libraries load from pinned CDN URLs, so this needs internet access.

### Option 3 — open the file

Double-click `index.html` (Chrome, Edge, Firefox or Safari). Everything works except one thing: **Download project** can only pack the app itself, because browsers don't let a page read neighbouring files from disk. Use option 1 or 2 if you need that.

### Deploying

The app is static files, so any static host works: Nginx, Apache, S3 + CloudFront, GitHub Pages, Netlify, Vercel, Azure Static Web Apps, or the Docker image on any container platform (ECS, Cloud Run, AKS, Kubernetes). No backend, database or environment variables are needed. Designs are saved in each visitor's browser and as JSON files they download.

## What's inside

```
archflow3d/
├── index.html                      The complete app (React + three.js, single file)
├── README.md                       This file
├── Dockerfile                      Builds an offline nginx image (libraries vendored from npm)
├── nginx.conf                      nginx site config (gzip, caching, /healthz)
├── docker-compose.yml              One-command run on port 8080
├── .dockerignore
├── SKILL.md                        Skill file for extending the app with Claude
├── docs/
│   ├── USER_GUIDE.md               How to design, simulate and export
│   ├── ARCHITECTURE.md             How the code is organised, and how to extend it
│   └── FILE_FORMAT.md              JSON schema for saved designs
├── examples/
│   ├── checkout-microservices.json
│   ├── single-server.json
│   ├── serverless-aws.json
│   └── iot-azure.json
└── designs/                        Your design at the time you downloaded the zip
```

## Features

- **100+ blocks** across Clients, Network & Edge, Compute, Services & APIs, Data, Messaging, AWS, Google Cloud, Azure, and Software & Ops. Each family has its own 3D shape (cylinders for databases, hex prisms for services, stacked trays for queues, translucent zones for VPCs and subnets).
- **Seven connector types**: request/response, one-way, two-way, half duplex, full duplex, multiplexed and streaming. Each has its own look (dashed, double-headed, twin lanes, braided lanes, moving pulse) and its own packet behaviour, plus protocol labels such as HTTPS, gRPC, SQL or Kafka.
- **Flow simulation** with fixed packet colours: amber requests, cyan responses, violet one-way events, green stream messages. The playing flow's path is highlighted in its own colour, and a hop log traces each step.
- **Requests per second**: each flow has a rate; blocks and connections show their load. Set capacity and instances per block to see utilization (green, amber at 70%, red at 90%), a traffic summary, and live traffic for all flows at once.
- **Descriptions** for every block type, editable per block, shown in hover tooltips and optionally on the labels.
- **Groups**: shift-click blocks and press `G` to put them in one server/host, VM, Docker host, Kubernetes pod, cluster, availability zone or region. The box follows its blocks and moves them together.
- **High-quality export** in 16:9, 4:3, 1:1 or stage shape: PNG is re-rendered at the chosen width (1920 / 2560 / 3840 / 7680 px) with 4096 px shadows and 2× supersampled labels. GIF frames are rendered at the GIF's own resolution, quantised with a median-cut palette that reserves room for packet colours, and encoded in the browser.
- **Guided tour** on first visit (replay from **Tour**), and a **Help** sheet with shortcuts.
- **Save / open JSON**, and **Download project** as a zip.
- Templates: checkout microservices, single-server deployment, serverless on AWS, IoT telemetry on Azure, blank.

## Keyboard

| Key | Action |
| --- | --- |
| `V` | Select tool |
| `C` | Connect tool |
| `R` | Record a flow |
| `F` | Fit design in view |
| `G` | Group selected blocks |
| `L` | Toggle descriptions on labels |
| `Shift` + click | Multi-select |
| `Delete` | Delete selection |
| `Esc` | Back to Select |
| `?` | Help |

## Documentation

- [User guide](docs/USER_GUIDE.md)
- [Architecture and extension guide](docs/ARCHITECTURE.md)
- [Design file format](docs/FILE_FORMAT.md)
- [SKILL.md](SKILL.md) for Claude

## Browser notes

- 8K PNG needs a GPU that supports 8192 px render buffers. If it doesn't, the export is capped at the largest size your GPU allows and the saved file name shows the real width.
- A 12-second GIF at 1600 px can reach 20–40 MB. Use 1280 px and 8 seconds for docs and slides.
- Designs autosave to the browser's local storage on this device only. Use **Save JSON** to keep or share a design.
