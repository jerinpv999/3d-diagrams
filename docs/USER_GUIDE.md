# ArchFlow 3D user guide

## 1. The workspace

| Area | What it does |
| --- | --- |
| **Header** | Diagram title (click to rename), template picker, Open JSON, Tour, Help, Export menu |
| **Block palette** (left) | Every block type, grouped by family, with search |
| **3D stage** (centre) | Your design. Tools and camera buttons at the top left, request/response legend at the top right |
| **Inspector** (right, top) | Properties of the selected block or connection |
| **Flows** (right, middle) | Saved request paths, playback controls |
| **Hop log** (right, bottom) | Step-by-step trace of the flow that is playing |

On screens narrower than about 980 px the panels stack under the stage.

## 2. Adding and arranging blocks

1. Search the palette or open a family.
2. Click a block. It is dropped at the nearest free spot around the centre of the view and selected.
3. Drag it on the stage. Positions snap to a 0.5-unit grid.

**Zones** (VPC / Network, Subnet, AWS VPC, GCP VPC Network, Azure Virtual Network) are flat translucent areas used to group blocks. Set their width and depth in the Inspector. Zones sit underneath blocks, so you can click blocks on top of them.

### Block families

| Family | Examples | Shape |
| --- | --- | --- |
| Clients | Web App, Mobile App, Desktop App, IoT Device, User, Partner System | Monitor, phone, device, figure |
| Network & Edge | DNS, CDN, WAF / Firewall, Load Balancer, API Gateway, Service Mesh, VPN Gateway, VPC, Subnet | Beacon, globe, wall, disc, gate, ring, zone |
| Compute | Server, Virtual Machine, Node.js Runtime, Container, Kubernetes Cluster, Serverless Function | Rack, glass cube, ribbed box, cube cluster, octahedron |
| Services & APIs | Microservice, REST, GraphQL, gRPC, BFF, Auth, Orchestrator / Saga, Background Worker, Scheduler | Hex prism, stepped hex |
| Data | SQL, NoSQL, Redis Cache, Object Storage, Search Index, Data Warehouse, Vector DB | Cylinder, octagon, bucket, magnifier, wide drum |
| Messaging | Message Queue, Event Stream, Pub/Sub Topic, Event Bus, WebSocket Hub | Stacked trays, segmented bar, broadcast mast |
| AWS | EC2, Lambda, ECS/Fargate, EKS, API Gateway, CloudFront, ALB, Route 53, S3, RDS, DynamoDB, ElastiCache, SQS, SNS, Kinesis, EventBridge, Cognito, Step Functions, VPC | By function |
| Google Cloud | Compute Engine, Cloud Run, Cloud Functions, GKE, API Gateway, Load Balancing, Cloud CDN, Cloud Storage, Cloud SQL, Firestore, Spanner, Memorystore, BigQuery, Pub/Sub, Workflows, VPC | By function |
| Azure | Virtual Machines, App Service, Functions, AKS, API Management, Front Door, Load Balancer, Blob Storage, SQL Database, Cosmos DB, Cache for Redis, Service Bus, Event Hubs, Logic Apps, Entra ID, Virtual Network | By function |
| Software & Ops | Monitoring, Log Aggregation, Distributed Tracing, CI/CD, Secrets Vault, Container Registry, Email/SMS, Payment Provider, Third-party API, LLM / AI Model | Orbit sphere, trays, bar, dodecahedron |

Colour shows the family of what a block *does* (data blocks are yellow whether they are generic, AWS, GCP or Azure). The provider name appears on the label, for example `AWS · S3`.

## 3. Connecting blocks

1. Choose **Connect** (or press `C`).
2. Pick the **connector type** in the box that appears in the toolbar. New links use it and get a matching default label.
3. Click the source block, then the target.
4. Keep clicking: each new click connects from the last target, so you can draw a chain quickly.
5. Press `Esc` to finish.

Select a connection to change its type, label (pick from the list or type anything), or direction, or to delete it. If two blocks are connected both ways, the two links curve apart so both stay visible.

### Connector types

| Type | Use it for | On the stage | During a flow |
| --- | --- | --- | --- |
| Request / response | HTTP, REST, SQL, RPC calls | Solid line, arrow at the target | Amber request out, cyan response back |
| One-way (fire and forget) | Events, webhooks, enqueueing a job | Dashed line | Violet event out, **no reply** on this link |
| Two-way (bidirectional) | Peers that both start requests | Arrows at both ends | Request and response |
| Half duplex | Both directions, one side at a time | Arrows at both ends, disc in the middle | Request and response |
| Full duplex | WebSocket, raw TCP | Two parallel lanes with opposite arrows | Request plus a simultaneous push back on the other lane |
| Multiplexed | HTTP/2, gRPC, QUIC | Three braided lanes | Three packets travel side by side |
| Streaming | Kafka consumers, gRPC streams, SSE | A pulse moves continuously along the line | A train of green messages, **no reply** |

The connection tag shows label, type and load, for example `gRPC · multiplex · 1.2k/s`.

## 4. Simulating request flows

A flow is the path one request takes, for example *Mobile App → API Gateway → Order Service → Orders DB*.

### Record a flow

1. Press **Record flow** (or `R`, or **+ Record** in Flows).
2. Click blocks in order, starting at the block that sends the request.
3. In the right panel, name the flow and type a note for each hop, such as `POST /v1/orders` or `SELECT * FROM orders`.
4. Press **Save flow**. Any hop without a connection gets a request/response link automatically. New flows start at 100 req/s.

### Packet colours

Colours always mean the same thing, whichever flow is playing:

| Colour | Meaning |
| --- | --- |
| Amber | Request |
| Cyan | Response |
| Violet | One-way event (no reply expected) |
| Green | Stream message |

A flow's own colour (the dot on its card) tints its path while it plays, so you can see which links it uses.

### Play a flow

- **Play** traces one request hop by hop. Each block pulses as a packet arrives.
- The **Hop log** lists every hop: direction arrow (→ request, ← response, ⇢ event, ⇉ stream), blocks, protocol, connector type and note. The current hop is highlighted.
- **Speed** runs from 0.25× to 3×. **Loop flow** repeats it.

### Requests per second

- Each flow card has a **Rate** in requests per second.
- Every block and connection on the flow's path adds that rate to its load. Loads show on labels (`980 req/s`) and connection tags (`900/s`).
- **Live traffic** sends requests from every flow at once. Higher rates send packets more often (on a log scale, so 10 req/s and 10,000 req/s both stay readable).
- The **traffic summary** above the flows shows total req/s, how many blocks are near or over capacity, and the busiest block.

### Capacity and utilization

Select a block and set **Capacity per instance** (req/s one instance can handle) and **Instances**. Utilization is load ÷ (capacity × instances).

| Utilization | Label |
| --- | --- |
| Under 70% | Green load line |
| 70–89% | Amber border |
| 90% and over | Red border and a red ring on the ground |

Add instances or raise capacity until the block is green. Group boxes show the total load of everything inside.

## 5. Groups: blocks on one server

Put a gateway, an API and a database "in one server", or services in one pod, availability zone or region.

1. With **Select**, shift-click two or more blocks. Each gets a blue ring.
2. Press `G` to group them as a **Server / Host** straight away, or pick a type and name in the Inspector and press **Group**.
3. A box appears around the blocks and resizes itself when you move them.

Group types: Server / Host, Virtual Machine, Docker Host, Kubernetes Pod, Cluster, Availability Zone, Region.

- Drag the group's floor to move everything inside.
- Select the group to rename it, change its type, describe it (for example `8 vCPU / 32 GB`), jump to a member, or **Ungroup**.
- Select a member and press **Remove from group** to take just that block out.
- The **Single-server deployment** template shows a gateway, Node.js API, Redis and PostgreSQL on one host, with a separate worker VM.

## 6. Descriptions

Every block type has a built-in description. Override it per block in the Inspector's **Description** box.

- **Hover** a block, group or connection to see its description and load in a tooltip.
- Switch the toolbar's **Names / Details** toggle (or press `L`) to print descriptions on the 3D labels. Details also appear in PNG and GIF exports.

## 7. Camera

| Action | Mouse | Touch |
| --- | --- | --- |
| Orbit | Drag empty space | One-finger drag |
| Pan | Right-drag | Two-finger drag |
| Zoom | Scroll | Pinch |

**Fit** (`F`) frames the whole design at an isometric angle. **Top** gives a plan view, useful for clean documentation images. **Front** gives a low side view.

## 8. Exporting

Open **Export**.

### Frame

Pick the output shape first: **16:9** (slides and video, default), **4:3** (documents), **1:1** (square posts) or **Match the stage**. It applies to both PNG and GIF. **Fit the whole design to the frame** (on by default) keeps your viewing angle but moves the camera so every block and label fills the image; turn it off to export exactly what you framed. While a GIF records at a shape different from the stage, the on-screen preview may look stretched; the file is not.

### PNG

Choose a width: Full HD (1920), QHD (2560), 4K (3840, default) or 8K (7680). The app re-renders the current camera view at that size with high-resolution shadows. It does not upscale the screen, so text stays sharp. The height comes from the frame you picked.

Tips:
- Press **Fit**, then zoom in slightly so the design fills the frame.
- **Top** view makes the most diagram-like images.
- The title and component count are stamped in the bottom-left corner.

### GIF

Choose width (960 / 1280 / 1600 px), length (3–12 s) and frame rate (10 / 12 / 16 fps), then **Record GIF**.

- If a flow or live traffic is playing, it is recorded. If not, the first flow starts automatically; with no flows, live traffic runs for the recording.
- Frames are rendered at the GIF's own size, then encoded in the browser. A progress bar shows recording and encoding.
- Rough file sizes: 1280 px · 5 s · 12 fps ≈ 4–10 MB. 1600 px · 12 s · 16 fps can exceed 30 MB.

### JSON and project zip

- **Save JSON** saves the design (blocks, connections, flows). **Open JSON** in the header loads it.
- **Download project** saves `archflow3d-project.zip` with the app, docs, SKILL.md, examples and your current design.

## 9. Keyboard shortcuts

| Key | Action |
| --- | --- |
| `V` | Select tool |
| `C` | Connect tool |
| `R` | Record a new flow |
| `F` | Fit design |
| `G` | Group selected blocks into a server |
| `L` | Toggle descriptions on labels |
| `Shift` + click | Add or remove a block from the selection |
| `Delete` / `Backspace` | Delete the selected block(s) or connection. Deleting a group ungroups it. |
| `Esc` | Cancel the current tool and clear selection |
| `?` | Help |
| `←` `→` | Previous / next step during the tour |

## 10. Saving your work

The current design autosaves to this browser's local storage. It is not synced across devices or browsers. Save JSON for anything you want to keep.

## 11. Troubleshooting

| Problem | Fix |
| --- | --- |
| The stage is blank | Your browser needs WebGL. Try Chrome or Edge, and check hardware acceleration is on. |
| 8K PNG comes out smaller | Your GPU's render-buffer limit was reached. The file name shows the real width. |
| GIF is very large | Lower the width to 1280 or 960, shorten the clip, or use 10 fps. |
| Save prompt doesn't appear | In the hosted version, saves go through a confirmation dialog. If it says saving isn't available, open the app in a full browser tab. |
| "Download project" only has the app | You opened `index.html` straight from disk. Serve the folder (`npx serve .`) so the docs can be read. |
