# ArchFlow 3D design file format

Designs are saved as UTF-8 JSON. **Save JSON** writes this format and **Open JSON** reads it. Files in `examples/` and `designs/` use it too.

Version 2 adds connector kinds, request rates, capacity, descriptions and groups. Every new field is optional, so version 1 files still open unchanged.

## Top level

```json
{
  "app": "ArchFlow 3D",
  "version": 2,
  "title": "Checkout platform",
  "nodes": [ ... ],
  "edges": [ ... ],
  "flows": [ ... ]
}
```

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `app` | string | no | Always `"ArchFlow 3D"` when written by the app |
| `version` | number | no | `2` (older files say `1`) |
| `title` | string | no | Diagram title, stamped on exports |
| `nodes` | array | **yes** | Blocks, zones and groups |
| `edges` | array | **yes** | Connections |
| `flows` | array | no | Request flows |

## Node (block)

```json
{ "id": "orders", "type": "microservice", "name": "Order Service", "x": -1.5, "z": 0,
  "desc": "Creates and tracks orders.", "capacity": 250, "replicas": 2, "group": "pod1" }
```

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | Unique within the file |
| `type` | string | A catalog type, for example `web`, `mobile`, `apigw`, `microservice`, `sql`, `kafka`, `aws-lambda`, `gcp-bigquery`, `az-cosmos`. Unknown types are dropped on import. |
| `name` | string | Label shown on the stage |
| `x`, `z` | number | Ground position in scene units. One grid square is 1 unit; blocks are about 2 units wide. |
| `desc` | string | Optional description. If empty, the type's built-in description is shown. |
| `capacity` | number | Optional requests per second one instance can handle |
| `replicas` | number | Optional instance count (default 1). Total capacity = `capacity × replicas`. |
| `group` | string | Optional id of the group node this block belongs to |
| `w`, `d` | number | Zone blocks only (`vpc`, `subnet`, `aws-vpc`, `gcp-vpc`, `az-vnet`): width and depth, 4–40 |

## Node (group)

Groups are nodes whose `type` starts with `grp-`. They have no size of their own: the box is computed around the blocks that point at them with `group`.

```json
{ "id": "host1", "type": "grp-server", "name": "app-server-01", "x": 0, "z": 0, "desc": "8 vCPU / 32 GB" }
```

| `type` | Shown as |
| --- | --- |
| `grp-server` | Server / Host |
| `grp-vm` | Virtual Machine |
| `grp-docker` | Docker Host |
| `grp-pod` | Kubernetes Pod |
| `grp-cluster` | Cluster |
| `grp-az` | Availability Zone |
| `grp-region` | Region |

A group with no members is removed on load. A block belongs to at most one group. Groups can't contain zones or other groups.

## Edge

```json
{ "id": "e_apigw_auth", "from": "apigw", "to": "auth", "label": "gRPC", "kind": "multiplex" }
```

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | Unique |
| `from`, `to` | string | Node ids. The arrow points at `to`. |
| `label` | string | Protocol or any short text, for example `HTTPS`, `gRPC`, `SQL`, `Kafka` |
| `kind` | string | Connector type, default `request`. See below. |

### Connector kinds

| `kind` | Meaning | Look | Packets |
| --- | --- | --- | --- |
| `request` | Synchronous request/response | Solid line, arrow at target | Request forward, response back |
| `oneway` | Fire and forget | Dashed line, arrow at target | Violet event forward, **no response** |
| `twoway` | Either side may initiate | Arrows at both ends | Request and response |
| `halfduplex` | Both directions, one at a time | Arrows at both ends, disc in the middle | Request and response |
| `duplex` | Both sides send at once | Two parallel lanes, opposite arrows | Request plus a simultaneous push on the other lane |
| `multiplex` | Concurrent streams on one connection | Three braided lanes | Three packets in parallel per hop |
| `stream` | Continuous one-direction flow | Moving pulse along the line | A train of green packets, **no response** |

## Flow

```json
{
  "id": "f1",
  "name": "Place order",
  "color": "#ff8f5a",
  "rps": 120,
  "steps": [
    { "node": "mob",    "note": "",                 "ret": "order shown" },
    { "node": "apigw",  "note": "POST /v1/orders",  "ret": "201 + order JSON" },
    { "node": "orders", "note": "createOrder()",    "ret": "201 Created" }
  ]
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | Unique |
| `name` | string | Shown in the Flows panel |
| `color` | string | Hex colour used to highlight the flow's path while it plays. Packet colours are fixed: request `#ffb547`, response `#4fd8ff`, event `#c58bff`, stream `#7cf29a`. |
| `rps` | number | Requests per second entering this flow. Adds to the load of every block and connection on its path. |
| `steps` | array | At least 2. Order is the request path. |
| `steps[].node` | string | Node id (a block, not a group or zone) |
| `steps[].note` | string | Request arriving at this node (shown in the Hop log) |
| `steps[].ret` | string | Response leaving this node on the way back (defaults to `response`) |

### How playback uses steps

For steps `A, B, C` the hops are:

1. `A → B` request, note from `B.note`
2. `B → C` request, note from `C.note`
3. `C → B` response, note from `C.ret`
4. `B → A` response, note from `B.ret`

If a link is `oneway` or `stream`, its forward hop is an event or stream message and its response hop is skipped. A hop with no connection still plays, along a temporary curve.

### How load is computed

- Block load = sum of `rps` of every flow that visits the block (a block visited twice by one flow counts once).
- Connection load = sum of `rps` of every flow that uses the connection, in either direction.
- Group load = sum of its members' loads.
- Utilization = load ÷ (`capacity × replicas`). Labels turn amber at 70% and red at 90%.

## Writing files by hand or with a script

- Keep blocks at least 3 units apart. Leave about 4 units between groups.
- A readable left-to-right layout puts clients around `x = -14`, gateways around `x = -7`, services between `-2` and `5`, and data and external systems from `5` to `15`.
- Validate by opening the file with **Open JSON**. If the file has no `nodes` or `edges` array, the app shows an error and keeps your current design.
