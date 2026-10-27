# APIs

What developers and AI agents can call. This page describes the interfaces only; the implementation is private.

> **Availability.** The Recast API is built and tested end to end over real HTTP, but it is **not publicly hosted yet**. Access for design partners is by request (see the [README](../README.md#contact)). Relay runs on your own machine or server.

- [Recast API](#recast-api): turn raw text into typed objects, and place them in apps
- [Recast SDK](#recast-sdk-typescript): a typed TypeScript client
- [Recast MCP server](#recast-mcp-server): Recast as tools for AI agents
- [Relay](#relay): command line, HTTP API and agent tools

---

## Recast API

Base path `/v1/`. Every request carries `Authorization: Bearer <api key>`. Every response carries `Recast-Schema-Version`.

### `POST /v1/resolve`

Raw text in, a typed object out (Person, Event, Location, Product, Paper, Document), with where each field came from and a route to a destination.

```json
{
  "input":   { "raw": "Meera Iyer, meera@example.com", "source_url": "https://example.com/team" },
  "target":  { "type": "Person", "destination": "google_calendar" },
  "options": { "confidence_threshold": 0.5, "include_provenance": true }
}
```

The answer is one of three shapes, chosen by `status`:

| `status` | Contains |
|---|---|
| `resolved` | `object` (id, type, fields, confidence, trust, provenance), `routing` (action, connector, reliability, whether a person must confirm), `alternatives`, `degraded` |
| `ambiguous` | `alternatives[]` and the reason (two readings tie, or the best is under your threshold) |
| `unresolved` | the reason, and sometimes a suggestion ("relative dates like 'Friday' need a full date") |

Every value is traceable to the input: Recast never invents a field.

### `POST /v1/place`

Do it: create a calendar event, add an attendee, save a Gmail draft, add a task, a sheet row or a Notion page.

```json
{
  "input":   { "raw": "Meera Iyer, meera@example.com" },
  "target":  { "destination": "google_calendar" },
  "destination_target": { "calendar_id": "primary", "event_id": "abc123" },
  "consent": { "kind": "confirm" },
  "object_id": "rc_..."
}
```

| Answer | Meaning |
|---|---|
| `201 placed` | The app confirmed it. Includes a `receipt`, the data that moved, the risk level, the action's history and whether Undo is available. |
| `202 outcome_unknown` | The write was sent and no answer came back. Recast **does not resend it**. Call reconcile. |
| `403 consent_required` | This action needs a person's confirmation (or a press-and-hold), with the reasons. Nothing moved. |
| `409 object_changed` | The text now reads differently from what the person was shown. Nothing moved. |
| `409 already_placed` | The same event is already there. Nothing new is created. |
| `422 incomplete` | Something required is missing. Recast does not invent it. |

- `POST /v1/place/undo` `{ "receipt_id": "..." }` reverses a placement. Two undos at once still reverse it once.
- `POST /v1/place/reconcile` `{ "action_id": "..." }` asks the app whether an unknown write landed, reading only, and settles it once.

### Other endpoints

| Endpoint | Purpose |
|---|---|
| `POST /v1/resolve/batch` | 1 to 50 inputs, each with its own result or error |
| `GET /v1/bridges` | The connectors and what each can do |
| `GET /v1/capabilities` | What this key may use |
| `POST /v1/corrections` | Report a correction. Stored only for accounts that opted in. |
| `GET /v1/openapi.json` | The contract, limited to what this key may use |
| `GET /v1/health` | Liveness |

### Rules every call follows

- **Consent comes from the effect, not the caller.** Anything that notifies another person, cannot be undone, or rests on a guess asks for confirmation. A caller cannot lower that.
- **Idempotency.** Send `Idempotency-Key`. The same key and body replays the first answer without counting again; the same key with a different body is `422`.
- **Limits.** `429` with `Retry-After` when over the rate; `402` when the monthly allowance is used. Only resolved answers count.
- **Versioning.** `/v1/` in the path. A retiring version announces itself with `Deprecation` and `Sunset` headers.
- **Privacy.** API keys are stored only as hashes. App tokens are sealed per account and per connector, never returned. Errors never include internals.

### Planned allowances

| Tier | Requests per second | Resolutions per month |
|---|---:|---:|
| Free | 10 | 1,000 |
| Pro | 100 | 50,000 |
| Scale | 500 | 500,000 |

Prices are not set yet.

---

## Recast SDK (TypeScript)

```ts
import { Recast } from "@recast/sdk";

const recast = new Recast({ apiKey: process.env.RECAST_API_KEY!, baseUrl: "<your Recast API address>" });

// What is this text, and where could it go?
const r = await recast.resolve({ raw: "Lunch with Arjun, Saturday 1 PM at Toit", target: { type: "Event", destination: "google_calendar" } });
if (r.status === "resolved") console.log(r.object.fields, r.routing);

// The same, fluently
await recast.from("Meera Iyer, meera@example.com").as("Person").to("google_calendar").threshold(0.8).run();

// Place it, with the consent the person gave
const p = await recast.place({ raw: "...", destination: "google_calendar_event", target: { calendar_id: "primary" }, consent: "confirm" });
if (p.status === "placed") await recast.placements.undo(p.receipt.id);
else await recast.placements.reconcile(p.actionId);   // unknown outcome: check, never resend
```

Also: `resolveBatch`, `bridges.list()`, `corrections.create()`, `health()`. The SDK has no dependencies, retries safely (one idempotency key per call, so a retry never places twice), honours `Retry-After`, and stops calling a failing service for a while instead of hammering it.

---

## Recast MCP server

AI agents (Claude Desktop, or any MCP client) reach Recast as tools:

| Tool | What it does |
|---|---|
| `resolve_object` | Raw text in; typed object, provenance and routing out |
| `list_destinations` | The connectors and what each accepts |

Both are read-only: an agent can understand and route, but placing stays a separate step that a person consents to. Runs over stdio (desktop agents) or Streamable HTTP (hosted agents), locally or against the hosted API with your key.

---

## Relay

Relay runs next to your repository. Three ways in, one engine behind them: they cannot disagree about a result.

### Command line

```sh
relay contract propose user-api --definition "getUser(id) -> {id, fullName}" --accept
relay task create T-api --objective "API returns fullName" --scope src/server.js --contract user-api@1 --verify "npm test"
relay task approve T-api
relay agent-config claude-code --task T-api --root "$PWD" --out /tmp/relay-T-api   # a guarded agent session
relay check T-api && relay verify T-api                                            # Relay's own evidence
relay integrate together T-api,T-ui                                                # Check Together; never applies
```

`relay integrate together` exits `0` passed, `1` blocked, `2` usage, `3` could not complete. Other commands: `init`, `impact`, `change`, `context`, `decision`, `events`, `ide`, `remote join | status | leave`.

### HTTP API (`relay ide`)

| Endpoint | Purpose |
|---|---|
| `POST /api/integrations/check` | Rehearse tasks together on a combined copy. Returns the result, the explanation (confirmed, likely or unknown) and a text report. |
| `GET /api/integrations/:id` | Read a rehearsal back |
| `POST /api/integrations/:id/approve` | Apply, only after it passed, nothing moved since, and a person approves |
| `GET /api/coordination` | What is running, blocked, overlapping, and safe to integrate next |

### Tools for coding agents (MCP)

An agent session is bound to one task. It gets coordination tools such as `claim_task`, `get_task`, `get_contract`, `declare_intent`, `report_evidence` and `submit_for_review`. It gets no file, shell, merge or approval access through Relay: approving and applying are for people only. A guard hook in front of the agent stops edits outside its task's scope, and fails closed.
