# LINE MCP Agent — full LINE MCP coverage under Gemini

| workflow | file | n8n id | active |
|---|---|---|---|
| LINE MCP Agent | `line-mcp-agent.json` | `D040YFvnbPnBhwHc` | no (run from the editor chat) |
| LINE Send Text (tool) | `line-send-text-tool.json` | `LnSendTxtTool01` | **yes** |
| LINE Send Flex (tool) | `line-send-flex-tool.json` | `LnSendFlexTl01` | **yes** |
| LINE MCP Call (tool) | `line-mcp-call-tool.json` | `LnMcpCallTool1` | **yes** |

## Does the LINE MCP server need a new Docker container?

No — `docker-compose.yml` already defines the `line-mcp` service. It is
`@line/line-bot-mcp-server` behind `supergateway`, which turns its stdio
transport into HTTP Streamable at `http://line-mcp:8000/mcp` (compose network
only, no host port).

It **did** need a rebuild, though: `create_rich_menu` renders its menu image
with Marp + Puppeteer, and the image had no browser because
`NPM_CONFIG_IGNORE_SCRIPTS=true` skips Puppeteer's Chromium download. The
Dockerfile now installs Alpine's `chromium` plus `font-noto-thai` (Thai menu
labels would otherwise render as tofu) and points `PUPPETEER_EXECUTABLE_PATH`
at it. After changing `line-mcp/Dockerfile`:

```
docker compose up -d --build line-mcp
```

## Why the workflow failed

```
[400] Invalid JSON payload received. Unknown name "const" at
'tools[0].function_declarations[0].parameters.properties[1].value.properties[0].value'
```

Gemini's `function_declarations[].parameters` accept an OpenAPI 3.0 subset,
which has no `const`. Several LINE MCP tools declare literals
(`z.literal("text")` in `textMessage.js` → `"const": "text"`), so Gemini
rejects the whole request before any tool runs.

Excluding only the broadcast tools cannot work: `push_text_message` is itself
`function_declarations[0]` in the error. `DESTINATION_USER_ID` does not help
either — it changes a default value, not the schema.

## How all 12 tools stay reachable

| LINE MCP tool | `const` | reached through |
|---|---|---|
| `get_profile` | no | `LINE MCP` node (native schema) |
| `get_message_quota` | no | `LINE MCP` node |
| `get_follower_ids` | no | `LINE MCP` node |
| `get_rich_menu_list` | no | `LINE MCP` node |
| `delete_rich_menu` | no | `LINE MCP` node |
| `set_rich_menu_default` | no | `LINE MCP` node |
| `cancel_rich_menu_default` | no | `LINE MCP` node |
| `push_text_message` | **yes** | `send_line_text` → sub-workflow |
| `push_flex_message` | **yes** | `send_line_flex` → sub-workflow |
| `create_rich_menu` | **yes** | `line_mcp_call` → sub-workflow |
| `broadcast_text_message` | **yes** | `line_mcp_call` (gated) |
| `broadcast_flex_message` | **yes** | `line_mcp_call` (gated) |

The pattern for every const-bearing tool: the model supplies only flat strings,
and a sub-workflow assembles the real arguments. Gemini never sees the schema
it cannot parse.

## How the userId reaches the LINE MCP node

Two different nodes, two different mechanisms:

- **`MCP Client Tool`** (attached to an agent) — arguments come only from the
  LLM. You cannot inject a field; the best you can do is put the value in the
  system message and hope the model copies it. Fragile.
- **`MCP Client`** (regular node) — `inputMode: json` lets you build the
  arguments with an expression. Deterministic. `line-push-form.json` already
  uses this.

Every sub-workflow tool wraps the second one, so the agent gets a tool while the
recipient stays wired:

```
LINE settings (Set)  ──►  AI Agent  ──►  send_line_flex (toolWorkflow)
  userId                                   │ userId       ← {{ $('LINE settings').item.json.userId }}
  allowBroadcast                           │ altText      ← {{ $fromAI('altText', …) }}
                                           │ contentsJson ← {{ $fromAI('contentsJson', …) }}
                                           ▼
                       Execute Workflow Trigger (userId, altText, contentsJson)
                                           ▼
                       Parse Flex  (strip ``` fences, JSON.parse, check container type)
                                           ▼
                       MCP Client → push_flex_message
                         {{ ({ userId: $json.userId, message: $json.message }) }}
```

Verified: with the Set node holding `DRYRUN-Ueedaa…`, the sub-workflow received
exactly that string — the value comes from the workflow, not the model.

`line_mcp_call` uses the same shape but dispatches dynamically: its MCP Client
node has `tool.value` set to `={{ $json.tool }}`, which n8n reads per item.

### Settings

The **`LINE settings`** node (formerly `LINE userId`) holds both switches:

- `userId` — the recipient. The repo file ships it empty so the id stays out of
  git; the running workflow has it filled. **Re-importing the repo file clears
  it.**
- `allowBroadcast` — `false` by default. A broadcast reaches every follower and
  cannot be recalled, so the switch lives in the workflow where a human flips
  it; the model can read it but never set it.

## Writing Flex cards

`send_line_flex` takes `contentsJson` as a **string**, not an object — a string
keeps the Gemini declaration flat. `Parse Flex` strips markdown fences, accepts
an object if the model sends one anyway, unwraps a full message object, and
rejects anything that is not a `bubble` or `carousel` with a message the agent
can act on.

The tool description carries a minimal bubble and the rules flash-lite breaks
most often: header/hero/body/footer must each be a `box`; colors are `#RRGGBB`;
`size` is an enum; an action `label` is ≤20 chars; padding and offsets look like
`"10px"`; a carousel holds 1–12 bubbles.

## Rich menus

`create_rich_menu` does the whole sequence itself — create, render the image,
upload it, set it as default. You only supply `chatBarText` and 1–6 actions; the
layout is fixed at 1600×910 and the areas are derived from the action count.

```json
{"chatBarText":"เมนู","actions":[
  {"type":"message","label":"สินค้า","text":"product"},
  {"type":"message","label":"ราคา","text":"price"},
  {"type":"uri","label":"เว็บไซต์","uri":"https://mounchon.com"}]}
```

To remove one: `get_rich_menu_list` → `cancel_rich_menu_default` →
`delete_rich_menu`, all native tools the agent can call directly.

## Test results (2026-09-22, gemini-3.1-flash-lite)

| test | result |
|---|---|
| `get_message_quota` through the agent | pass — no `const` error, replied `300 / 1 used` |
| `get_profile` on the configured recipient | pass — `Poo`, language `th` |
| userId wiring | pass — sub-workflow received the Set node value verbatim |
| failure path | pass — a push to `U` + 32 zeros returned `ok: false, "Failed: … 400 - Bad Request"` |
| **`send_line_text`** | delivered — reported by the owner, not re-tested after the node rename |
| **`send_line_flex`** | pass — card delivered to `U4dd0e19a…` |
| **`create_rich_menu`** via `line_mcp_call` | pass — `richmenu-ba54e0e659716a05b116139bd3bed6da`, Thai labels, set as default |
| broadcast gate | pass — refused for `allowBroadcast` `false` and `"false"` |
| allowlist | pass — `push_text_message` through `line_mcp_call` refused |
| `broadcast_*` actually sending | **not run** — needs `allowBroadcast: true`, and it cannot be undone |

`get_follower_ids` returns `403 Forbidden`. That endpoint requires a verified or
premium LINE Official Account; it is not a configuration error.

The failure path matters because `MCP Client` v1.1 turns an `isError: true` tool
result into a `NodeOperationError`; with `onError: continueRegularOutput` that
lands in `$json.error`, which `Tool Result` reports as `ok: false`. A LINE API
rejection therefore reaches the agent as a failure, not a silent success.

## Operational notes

- **The three sub-workflows must be Active.** Called from a production
  (non-manual) execution, an inactive sub-workflow fails with
  `Workflow is not active and cannot be executed.` `n8n import:workflow`
  deactivates on every import, so re-activate after importing.
- `n8n update:workflow` from the CLI needs a container restart to take effect.
- **Two workflows share the name "LINE MCP Agent".** `wsoWQZifni8LA2B4` is
  archived and unused; the live one is `D040YFvnbPnBhwHc`. There is also an
  unused duplicate of the text sub-workflow at `q431yJiWF3x2jFMn`.

## Alternative: switch the chat model

OpenAI and Anthropic accept `const`, so with one of those every LINE MCP tool
works natively with no wrappers at all — the model would call
`push_flex_message` directly instead of hand-writing a JSON string. That costs
API credits; the structure above keeps the Gemini free tier (20 requests/day).

Even on a `const`-tolerant model the wiring is worth keeping for the recipient:
`$('LINE settings').item.json.userId` cannot be mistyped by a model, and the
broadcast gate stays a human decision.
