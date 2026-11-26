# @hermeticor/cli

The `hermetica` command: sign in from the terminal, keep workspace setup in a file, send sandbox test mail and call any API operation. Node 20+. Hermeticor (hermeticor.com) was called Hermetica; the command, headers and env vars keep that name. This package replaces the deprecated `@hermeticacomms/cli`.

```sh
npm i -g @hermeticor/cli
```

## Quickstart

```sh
hermetica login                         # shows a code, opens the browser; approve there
hermetica whoami                        # the key's workspace and scopes
hermetica status                        # what blocks sending, and the API call that fixes each step
hermetica send --sandbox --outcome bounce   # test mail to bounce@sandbox.hermeticor.test, no provider needed
hermetica plan -f hermetica.json        # what apply would change
hermetica apply -f hermetica.json       # make the workspace match the file (run it twice: the second run changes nothing)
```

`login` stores the key in `~/.config/hermetica/credentials` (mode 0600), one entry per `--profile` (default `default`; or set `HERMETICA_PROFILE`). The key is never printed. `login --scopes profiles:read,consent:write` asks for specific scopes; `login --key hm_sk_...` or `login --paste` stores a key you already have. `--url` (or `HERMETICA_API_URL`) points at another instance; the default is `https://hermeticor.com`. `HERMETICA_API_KEY` overrides the stored key, for CI.

Missing a scope? `hermetica scopes request connections:write --reason "connect SES"` prints a link for a workspace owner and waits for the decision.

## Config as code

`plan` / `apply` send the file to `PUT /v1/config`. The file is YAML (`.yaml`/`.yml`) or JSON, same shape (shown as JSON):

```json
{
  "topics": [{ "channel": "email", "key": "product-news", "name": "Product news" }],
  "fields": [{ "key": "plan", "label": "Plan", "type": "text" }],
  "lists": [{ "name": "Customers", "topicKey": "product-news" }],
  "segments": [{ "key": "trials", "name": "Trials", "definition": { "type": "group", "op": "and", "children": [{ "type": "profile_property", "field": "attributes.plan", "predicate": { "valueType": "string", "op": "eq", "value": "trial" } }] } }],
  "templates": [{ "key": "welcome", "name": "Welcome", "kind": "html", "version": { "subject": "Welcome", "body": { "kind": "html", "html": "<p>Hi {{ profile.firstName }}</p>" } } }],
  "transactionalDefinitions": [{ "key": "receipt", "name": "Receipt", "templateKey": "welcome" }],
  "journeys": [{ "key": "onboarding", "name": "Onboarding", "status": "live", "definition": { "triggers": [{ "type": "api" }], "graph": { "start": "wait", "nodes": [{ "id": "wait", "type": "delay", "config": { "duration": { "amount": 1, "unit": "days" } } }, { "id": "done", "type": "end", "config": {} }], "edges": [{ "id": "e1", "source": "wait", "sourceHandle": "next", "target": "done" }] } } }],
  "webhookEndpoints": [{ "url": "https://example.com/hooks/hermetica", "topics": ["profile.*"], "secret": "<32+ random characters>" }],
  "senders": [{ "connection": "main", "fromEmail": "hello@example.com" }],
  "sendingDomains": [{ "domain": "example.com", "connection": "main" }],
  "consentSettings": { "cookieMunchSync": true },
  "conversions": [{ "label": "Trial started", "event": "trial_started" }],
  "postalAddresses": [{ "label": "Main", "company": "Example Ltd", "line1": "1 Main St", "city": "Springfield", "country": "US", "isDefault": true }],
  "postalAddress": "1 Main St, Springfield"
}
```

How each section is matched: topics on `channel` + `key`, fields on `key`, lists on `name`, webhook endpoints on `url`, senders on `connection` + address or number, sending domains on `domain`, conversions on `label`, saved postal addresses on `label` + `app` (none = shared). Segments, templates, transactional definitions and journeys on a `key` you choose: the file stores it on the item it creates, so renaming one keeps it matched (items made in the app have no key and are never touched). A transactional definition names its template by that key (`templateKey`), and anywhere in a journey's `definition` a step or trigger may write `templateKey` / `segmentKey` instead of `templateId` / `segmentId`, so the file works in any workspace (an unknown key is a 400 naming the step).

Only the properties written in the file are compared and changed. A template's `version` is its current content, compared as a whole (`subject`, `preheader` and `textBody` count as empty when left out); a change saves a new version. A journey's `definition` is what the workflows API takes; each part written (`graph`, `triggers`, `settings`) is compared whole, and a change shows per step, such as `definition.graph.nodes.wait1`. A webhook step's `signingSecret` is compared with the stored one without being shown: a different one is an update of that step. A journey's `status: live` publishes its draft (again after a change), `paused` pauses it; with no `status` only the draft changes. `consentSettings` is merged into the stored settings, never replaced (fields other than `cookieMunchSync` need a signed-in owner or admin). A sending domain is added at the named connection, and a changed `purpose` is an update; DNS is not published for you, and the plan shows whether each domain and sender is verified.

Nothing is deleted: `apply --prune` archives the topics, fields, lists, segments, templates and transactional definitions a section of the file leaves out, disables such webhook endpoints, removes such conversions from the named list (conversions already recorded stay), and pauses a live journey (a draft one is archived). Prune only touches segments, templates, definitions and journeys a file created. Senders, sending domains and saved postal addresses are never pruned. Sections not in the file are left alone.

`plan` exits 2 when anything would change and 0 when nothing would, so CI can fail on drift.

A webhook `secret` you choose is used on create, replaces the current one when it differs, and is never shown back (the plan prints `secret` only), so verify deliveries with the value from your own secret store. Leave `secret` out and the endpoint gets a generated one, shown once: `apply --secret-out ~/.config/<app>/hermetica-secrets.env` appends it as `HERMETICA_WEBHOOK_SECRET=…` to that file (mode 0600) instead of printing it. Keys may be read from `HERMETICA_API_KEY` or `HERMETICA_KEY`, and the URL from `HERMETICA_API_URL` or `HERMETICA_URL`.

Keep keys out of the file: any value written as `{ "env": "VAR_NAME" }` is replaced by that environment variable before the file is sent (an unset one stops the command). For a preset webhook endpoint:

```json
{ "webhookEndpoints": [{ "url": "https://watchtower.example.com/ingest", "topics": ["message.*"], "preset": { "kind": "watchtower", "environment": "production", "ingestKey": { "env": "WATCHTOWER_INGEST_KEY" } } }] }
```

A different ingest key is rotated to (the plan prints `preset.ingestKey`, never the value).

Each section needs its area's read scope to plan and write scope to apply (segments, templates, messages for transactional definitions, workflows for journeys, connections for senders and sending domains, templates for saved postal addresses, consent for consent settings, `analytics:read` / `settings:write` for conversions). A key bound to an app gets its app's slice: what it creates belongs to its app, prune touches only its app's items, and it cannot change shared items or another app's. With `connections:own` it may declare senders and sending domains on connections bound to its app (another connection is a 403 naming it). It may declare its own app's `postalAddresses` (created with its app; `isDefault` picks the one its footers print); another app's is a 403. It may declare its own app's `transactionalDefinitions` and `conversions` (created with its app): it sees only its own and shared ones, a shared one is read-only for it (noop, any change a 403), prune touches only its own, and a key or label that another app already uses is a 409 (they stay unique in the workspace, so prefix them, e.g. `crossly_order_shipped`). A definition's `templateKey` names its own app's or a shared template. Per-topic consent is `topics[].requireOptIn` (marketing email on that topic needs a recorded opt-in even under soft opt-in), which it sets on its own topics. `postalAddress` (the workspace address) and `consentSettings` (the workspace consent policy and the Cookie Munch link: one person's consent is the workspace's) are workspace-wide and need a key that is not bound to an app.

## Test a webhook receiver locally

```sh
hermetica webhooks listen --forward-to http://localhost:3000/api/email-events/hermetica
hermetica webhooks listen --forward-to http://localhost:3000/hooks --topics message.status,suppression.added
```

Streams the workspace's new events and POSTs each one to `--forward-to` with the body and headers of a real delivery (`Hermetica-Signature`, `Hermetica-Event-Id`, `Hermetica-Topic`), signed with a secret printed when the session starts: set it as your receiver's webhook secret while testing. One line per event shows your receiver's status code. No retries. Ctrl-C ends the session. Needs `webhooks:write` (included in the default login scopes through `integrations:write`).


```sh
hermetica profiles list email=ada@example.com
hermetica campaigns create --data @campaign.json
hermetica events tail name=signed_up
hermetica export lists > lists.jsonl
hermetica openapi list segments            # every operation of the server's OpenAPI document
hermetica openapi getV1SegmentsByIdMembers id=<segmentId> limit=10
hermetica help
```

Every write sends a fresh `Idempotency-Key`. Errors print the API's status, code and details and exit 1.
