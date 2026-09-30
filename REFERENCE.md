# LiveReacting MCP server reference

This is the complete reference for the LiveReacting MCP server: every tool, every input,
every response field, every limit and every error. It is written for the people who
integrate the server and for the models that call it.

To connect a client quickly, go to [section 2](#2-connecting). The REST API that the server
calls is documented at [developers.livereacting.com](https://developers.livereacting.com).

## Contents

1. [What this is](#1-what-this-is)
2. [Connecting](#2-connecting)
3. [Choosing which tools a client sees](#3-choosing-which-tools-a-client-sees)
4. [Tools](#4-tools)
5. [Behaviours that apply across tools](#5-behaviours-that-apply-across-tools)
6. [The two journeys](#6-the-two-journeys)
7. [Limits stated honestly](#7-limits-stated-honestly)
8. [Errors](#8-errors)
9. [Versioning](#9-versioning)

---

## 1. What this is

The LiveReacting MCP server lets an AI agent run live video streams. The agent can build a
stream from video files, start it, stop it, schedule it, watch it while it runs, and read
its analytics afterwards.

```
https://mcp.livereacting.com/mcp
```

| Property | Value |
|---|---|
| Transport | Streamable HTTP |
| Session model | Stateless. No session id, no server-initiated stream |
| HTTP methods | `POST` only. `GET` and `DELETE` answer `405` with `Allow: POST` |
| Response body | `application/json`. The server does not open an SSE stream |
| Protocol revisions | `2024-10-07`, `2024-11-05`, `2025-03-26`, `2025-06-18`, `2025-11-25` |
| Server name | `livereacting` |
| Server version | `1.2.0` |
| MCP SDK | `@modelcontextprotocol/sdk` 1.30.0 |
| Capabilities | Tools only. No resources, no prompts, no sampling, no elicitation |

`2026-07-28` is not supported. No published TypeScript SDK speaks it yet. When one does,
support becomes a dependency upgrade, not a rewrite. A client that asks for an unsupported
revision is not refused: as the protocol requires, the server answers with the newest revision
it speaks, `2025-11-25`, and the client decides whether it can use that. A client that speaks
only `2026-07-28` then disconnects.

Statelessness is deliberate. The server builds a fresh transport and a fresh server object
for every request. Reusing either across clients leaks one customer's tool responses to
another, because client SDKs all start their JSON-RPC ids at 0 and collide
(GHSA-345p-7cg4-v4c7).

### Authentication

Send your LiveReacting API key as a bearer token in the `Authorization` header. It is the
same key the REST API takes, so one credential works on both surfaces. Keys start with
`lr_`. Create one in the [Developer section](https://studio.livereacting.com/developer) of
the LiveReacting Studio.

```
Authorization: Bearer lr_…
```

A missing or invalid key returns `401 INVALID_API_KEY`.

**OAuth sign-in (ChatGPT, Claude).** Apps that cannot take a pasted key sign in with OAuth
2.1 instead. The customer adds the MCP URL, logs in with their normal livereacting.com
account and clicks Allow. Discovery starts from the `401` on an unauthenticated call, whose
`WWW-Authenticate` header points at
`https://mcp.livereacting.com/.well-known/oauth-protected-resource/mcp`.

- Clients identify themselves with a Client ID Metadata Document from `chatgpt.com`,
  `openai.com` or `claude.ai`. There is no dynamic client registration, so a client that
  needs it (Cursor, Gemini CLI) uses an API key.
- Authorization code with PKCE `S256`, public clients (`token_endpoint_auth_method: none`),
  `iss` on every authorization response (RFC 9207), and the `resource` parameter
  (RFC 8707).
- One scope, `livereacting`. It grants what an API key grants.
- Access tokens last one hour. Refresh tokens last 90 days from their last use, work once, and
  a second use of an old one ends the connection. The customer then connects again.
- An expired or revoked access token returns `401` with `error="invalid_token"`.

### Who can use it

API access is included with every paid plan. An account on the Free plan gets
`403 API_ACCESS_REQUIRES_PAID_PLAN` on every call, and cannot finish the OAuth sign-in. The
message explains the limit. Tool results never ask the customer to upgrade: ChatGPT's plugin
rules forbid it, so the server drops upgrade sentences from the messages it relays.

Two cases break that rule, one in each direction. An account we have given an access override
keeps API access on the Free plan. An account whose active subscription is paused loses API
access until the subscription runs again, whatever its plan says.

Some limits follow the plan tier rather than the plan type. 1080p formats need a 1080p plan
or better. 4K formats need a 4K plan. See [section 7](#7-limits-stated-honestly).

---

## 2. Connecting

Every block below uses `lr_…` as a placeholder. Never paste a real key into a file you
commit. Where a harness supports an environment variable or a prompt, use it.

### Claude Code

```bash
claude mcp add --transport http livereacting https://mcp.livereacting.com/mcp \
  --header "Authorization: Bearer lr_…"
```

### Codex CLI

In `~/.codex/config.toml`:

```toml
[mcp_servers.livereacting]
url = "https://mcp.livereacting.com/mcp"
bearer_token_env_var = "LIVEREACTING_API_KEY"
```

Then set `LIVEREACTING_API_KEY=lr_…` in your shell profile. Prefer this over
`http_headers`, which writes the key into the config file.

### Cursor

In `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "livereacting": {
      "url": "https://mcp.livereacting.com/mcp",
      "headers": { "Authorization": "Bearer lr_…" }
    }
  }
}
```

### VS Code (GitHub Copilot)

In `.vscode/mcp.json`. The `inputs` block makes VS Code prompt for the key instead of
storing it in the repository.

```json
{
  "inputs": [
    {
      "id": "livereacting-key",
      "type": "promptString",
      "description": "LiveReacting API key",
      "password": true
    }
  ],
  "servers": {
    "livereacting": {
      "type": "http",
      "url": "https://mcp.livereacting.com/mcp",
      "headers": { "Authorization": "Bearer ${input:livereacting-key}" }
    }
  }
}
```

### Gemini CLI

In `~/.gemini/settings.json`. Gemini CLI expands `${VAR}` in headers, so the key stays in
your shell:

```json
{
  "mcpServers": {
    "livereacting": {
      "httpUrl": "https://mcp.livereacting.com/mcp",
      "headers": { "Authorization": "Bearer ${LIVEREACTING_API_KEY}" }
    }
  }
}
```

### Zed

In your Zed `settings.json`, under `context_servers`, add an HTTP server that points at
`https://mcp.livereacting.com/mcp` with an `Authorization: Bearer lr_…` header.

### Goose

Add a Remote Extension of type Streamable HTTP in Goose's settings. Use the URL above and
a custom header `Authorization: Bearer lr_…`.

### Compatibility

| Harness | How the key is passed |
|---|---|
| Claude Code | `--header` |
| Codex CLI | `bearer_token_env_var` |
| Cursor | `headers` in `mcp.json` |
| VS Code Copilot | `headers` with a prompted input |
| GitHub Copilot coding agent | static header secret only. The product does not support OAuth remote servers, so the key is the only path, now and later |
| Gemini CLI | `headers` in `settings.json` |
| Zed | `headers` in `settings.json` |
| Goose | remote extension header |
| ChatGPT | OAuth sign-in; connectors never take a pasted key |
| Claude web and Desktop | OAuth sign-in as a custom connector |

ChatGPT and Claude web connect with OAuth, described under [Authentication](#authentication).

### When a client will not connect

Check the protocol revision first. The supported range is `2024-10-07` through
`2025-11-25`. The server offers `2025-11-25` to a client that asks for anything newer, so a
client pinned to `2026-07-28` disconnects on its own side. Most version errors are this.

Then check the HTTP method. The server answers `POST` only. A client that opens a `GET`
stream gets `405` with the message "This MCP server is stateless. Send tool calls as HTTP
POST to /mcp."

---

## 3. Choosing which tools a client sees

Two query parameters narrow the tool list. Both are handled on the server, so no client
support is needed. Put them on the URL you configure.

```
https://mcp.livereacting.com/mcp?toolsets=streams,media
https://mcp.livereacting.com/mcp?read_only=1
```

### `toolsets`

A comma-separated list. Valid values are `account`, `editor`, `streams`, `media` and
`analytics`. Omit the parameter to get every tool.

### `read_only`

Accepts `1`, `true`, `yes` or `on` for true, and `0`, `false`, `no` or `off` for false.
Omit it for false.

When true, the server registers only tools whose `readOnlyHint` annotation is `true`. That
leaves 13 of the 30 tools. Use it for a monitoring agent that must not touch a live
broadcast.

### Unknown values fail closed

An unrecognised value in either parameter is a `400`, not a fallback. The JSON-RPC error
names the values that are valid.

Both parameters exist to give an agent less power, so a typo must not silently give it
more. Before this rule existed, `?toolsets=stream` matched nothing, fell back to
undefined, and served all 27 tools including every mutation. A partial typo was quieter
and no better: `streams,medai` dropped media without saying so. `?read_only=treu` read as
false and served every mutation tool to a caller who was explicitly trying to prevent
that. A present but empty value, `?toolsets=` or `?toolsets=,`, names nothing and is refused
the same way.

---

## 4. Tools

The server registers 30 tools in five toolsets. Every name carries the prefix
`livereacting_`, so one search in a client that defers tool loading finds the whole server.

| Toolset | Tools |
|---|---|
| `account` | 2 |
| `editor` | 11 |
| `streams` | 7 |
| `media` | 9 |
| `analytics` | 1 |

Three conventions run through every entry below.

- Ids are 24-character hexadecimal LiveReacting ids. A value that is not one is refused
  before it reaches the database, with a message that names the argument.
- Every paged response uses the same field names: `itemTotal`, `offset`, `truncated`,
  `nextOffset` and, when something was left out, `notice`. See
  [truncation and paging](#truncation-and-paging).
- Annotations are hints for the client, not enforcement. The MCP specification tells
  clients to treat them as untrusted. Authorisation is always enforced on the server.

Every tool sets `readOnlyHint`, `destructiveHint` and `openWorldHint`. `openWorldHint` is
`true` when the tool can change what viewers of a public stream see, directly or through
[autoSynch](#autosynch-and-synced), or fetches from an outside URL. `destructiveHint` is
`true` when another call cannot undo the change.

Each tool lists a finer scope for a future split. Nothing enforces it today: an API key and
the OAuth scope `livereacting` both grant everything.

---

### account

#### `livereacting_list_destinations`

List the YouTube, Facebook, Twitch, LinkedIn, X and custom RTMP destinations connected to
the account.

**Use it** to get the destination ids that `livereacting_create_project` and
`livereacting_update_project` need. A destination cannot be created from here. The customer
connects one in the LiveReacting Studio.

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`. **Scope** `destinations:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `limit` | integer 1 to 50 | no | How many to return. Defaults to 50 |
| `offset` | integer, 0 or more | no | Skip this many. Use `nextOffset` from a truncated response |

**Response**

| Field | Meaning |
|---|---|
| `destinations` | The page of destinations |
| `destinations[].id` | The id to pass as a `destinationIds` entry |
| `destinations[].name` | Channel, page or account name, or null |
| `destinations[].type` | The raw account type, for example `youtube.channels` |
| `destinations[].provider` | One of `youtube`, `facebook`, `twitch`, `instagram`, `linkedin`, `rtmp`, `unknown` |
| `itemTotal`, `offset`, `truncated`, `nextOffset`, `notice` | Paging. See [truncation and paging](#truncation-and-paging) |

**Errors** The usual authentication and plan errors. Nothing specific to this tool.

---

#### `livereacting_validate_destinations`

Check that the given destinations still have a working login.

**Use it** before `livereacting_start_stream`, so a broken connection is reported before the
broadcast rather than during it.

Read the result carefully, because `valid: true` means different things per platform. For
YouTube, Facebook and Twitch the platform was contacted and the login works. For custom
RTMP, HLS and SRT destinations, which includes LinkedIn and X, nothing is checked at all.
`valid: true` there is returned immediately, without contacting anything and without checking
that a server URL and a stream key exist. It says only that the destination is on the account.

An expired YouTube access token is renewed during this check when the account still holds a
refresh token, so a login that has expired can still answer `valid: true`. When the renewal
fails, the customer reconnects the destination in the LiveReacting Studio.

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`. **Scope** `destinations:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `destinationIds` | array of strings, 1 to 50 | yes | Destination ids from `livereacting_list_destinations` |

**Response**

| Field | Meaning |
|---|---|
| `results` | One entry per requested id, in the order you sent them |
| `results[].id` | The destination id |
| `results[].valid` | `true` or `false` |
| `results[].error` | Present only when `valid` is `false`. Says why |

**Errors** `DESTINATION_NOT_FOUND` when an id does not belong to the account.

---

### editor

#### `livereacting_list_projects`

List the projects on the account, newest first. A project is the reusable template for a
stream: its video format, its destinations, and the scenes that hold the content.

**Use it** when the customer names a stream but you do not have its id. For the detail of
one project use `livereacting_get_project`.

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`. **Scope** `projects:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `status` | `live`, `scheduled` or `offline` | no | Only projects in this state. `live` also covers a stream that is still starting up |
| `limit` | integer 1 to 20 | no | Defaults to 20, which is also the maximum |
| `offset` | integer, 0 or more | no | Skip this many |

**Response**

| Field | Meaning |
|---|---|
| `projects[].id` | Project id |
| `projects[].name` | Project name |
| `projects[].title` | Broadcast title |
| `projects[].liveStatus` | `LIVE`, `PREPARING`, `SCHEDULED` or `OFFLINE` |
| `projects[].activeLiveId` | The running broadcast's id, or null. Null while `SCHEDULED` |
| `projects[].scheduledLivesCount` | How many future schedules the project holds |
| `itemTotal`, `offset`, `truncated`, `nextOffset`, `notice` | Paging |

---

#### `livereacting_get_project`

Read one project in full.

**Use it** to check a project is set up the way the customer expects before starting a
stream. For the live state of a running stream use `livereacting_get_stream_status`
instead, which reads the broadcast snapshot rather than the project. For scenes prefer
`livereacting_get_scenes`, which pages properly.

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`. **Scope** `projects:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | From `livereacting_list_projects` |
| `includeScenes` | boolean | no | Embed the scene list. Off by default |
| `includeScheduledLives` | boolean | no | Embed the pending schedules with their dates. Off by default |

**Response**

| Field | Meaning |
|---|---|
| `id`, `name`, `title`, `description` | Project identity |
| `format` | `{ aspectRatio, resolution }`, for example `{ "aspectRatio": "16:9", "resolution": "1920x1080" }` |
| `createdAt` | ISO 8601 |
| `autoSynch` | Whether edits reach a running stream by themselves |
| `unsyncedLive` | True when the project holds edits the running broadcast has not received |
| `audioQuality` | `128k`, `256k` or `320k` |
| `maxFps` | The project's frame-rate ceiling. 30 when the project carries none |
| `customBitrate` | A per-project override, usually null |
| `privacyStatus` | `youtube` and `facebook` keys, each present only when such a destination exists |
| `duration` | `{ manual, planned, elapsed, remaining }`. `elapsed` is non-null only while `LIVE` |
| `liveStatus` | `LIVE`, `PREPARING`, `SCHEDULED` or `OFFLINE` |
| `activeLiveId` | The running broadcast's id, or null |
| `viewers`, `peakViewers` | Null unless a broadcast is `LIVE` or `PREPARING` |
| `scheduledLivesCount` | Future schedules |
| `destinations[]` | `provider` and `name` always. While live, also `status`, `viewingUrl`, `viewers`, `peakViewers` and `error` when each has a value |
| `scheduledLives[]` | With `includeScheduledLives`. `{ liveId, date }`, earliest first |
| `scenes[]` | With `includeScenes`. At most 20 scenes, each with at most 50 layer summaries |
| `sceneTotal`, `scenesTruncated`, `scenesNotice` | With `includeScenes`. How many scenes exist and what was left out |

`viewingUrl` is only ever a public viewing page. Ingest URLs and stream keys are never
returned by any tool.

**Errors** `PROJECT_NOT_FOUND`.

---

#### `livereacting_create_project`

Create an empty project. This is the first step of building a stream.

**Use it** first, then `livereacting_create_scene_with_playlist` to put video in it, then
`livereacting_start_stream` or `livereacting_schedule_stream`. The project starts with no
scenes and no content.

For a 24/7 looping channel pass `duration: { manual: true }` and `autoSynch: true`.
`livereacting_create_scene_with_playlist` then loops the playlist by default, so there is
nothing else to set. For a stream that must stop on its own pass
`duration: { manual: false, planned: <seconds> }`.

Privacy settings and advanced YouTube options can only be set in the LiveReacting Studio.

**Annotations** `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: false`, `openWorldHint: false`. Calling it twice creates
two projects. **Scope** `projects:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `name` | string, 1 to 60 chars | yes | Project name. The customer sees this in the Studio |
| `title` | string, up to 100 chars | no | Broadcast title on the destination platform. Defaults to the project name |
| `description` | string, up to 5000 chars | no | Broadcast description on the destination platform |
| `format` | string | no | Aspect ratio and resolution. Defaults to `16:9_720p` |
| `audioQuality` | `128k`, `256k` or `320k` | no | Defaults to `128k` |
| `duration` | object | no | `{ manual: true }` or `{ manual: false, planned: <seconds> }`. Leave it out and the project gets a fixed one hour |
| `destinationIds` | array of strings, up to 50 | no | Where to stream. Can be set later |
| `autoSynch` | boolean | no | When true, later edits reach a running stream without a separate sync |

Valid `format` values are `1:1`, `1:1_720p`, `1:1_1080p`, `1:1_2160p`, `9:16`, `9:16_720p`,
`9:16_1080p`, `9:16_2160p`, `16:9`, `16:9_720p`, `16:9_1080p`, `16:9_2160p`, `1:2`,
`1:2_720p`, `1:2_1080p` and `1:2_2160p`.

`duration.planned` is a whole number of seconds, from 30 to 31536000 (365 days). It is
required when `manual` is false.

**Always send `duration`.** A project created without it gets a fixed duration of one hour, so
the stream stops an hour after it starts, whatever the content is.

**Response** The created project: `id`, `name`, `title`, `description`, `format`,
`createdAt`, `audioQuality`, `maxFps`, `customBitrate`, `autoSynch`, `privacyStatus`,
`duration`, `destinations` (each with `provider` and `name` only), and
`youtubeAdvancedSettings` when the account has any set.

**Errors** `VALIDATION_ERROR` for an unknown `format` or a bad field.
`FORMAT_NOT_AVAILABLE` when the format needs a higher plan. `DESTINATION_NOT_FOUND` for an
id that is not on the account. `DESTINATION_TOKEN_INVALID` when one of the ids has an expired
login, because the destinations are validated as they are attached.

---

#### `livereacting_update_project`

Change the settings of an existing project. Only the fields you send are changed.

**Use it** for project settings. To change a playlist use `livereacting_set_playlist`
instead.

Five fields are refused while the project is `LIVE`, `PREPARING` or `SCHEDULED`, not only
while it is on air: `format`, `duration`, `audioQuality`, `destinationIds` and
`youtubeAdvancedSettings`. To change any of those, stop the stream with
`livereacting_stop_stream` or cancel the schedule with `livereacting_cancel_schedules`
first. `name`, `title`, `description` and `autoSynch` can be changed at any time.

**Annotations** `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: true`, `openWorldHint: true`. Sending the same fields
twice leaves the same result. **Scope** `projects:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project to change |
| `name` | string, 1 to 60 chars | no | New project name |
| `title` | string, up to 100 chars | no | New broadcast title |
| `description` | string, up to 5000 chars | no | New broadcast description |
| `format` | string | no | Refused while live or scheduled |
| `audioQuality` | `128k`, `256k` or `320k` | no | Refused while live or scheduled |
| `duration` | object | no | Refused while live or scheduled |
| `destinationIds` | array of strings, up to 50 | no | Replaces the list. Refused while live or scheduled |
| `autoSynch` | boolean | no | Can be changed at any time |

**Response** The same shape as `livereacting_create_project`.

**Errors** `PROJECT_NOT_FOUND`, `VALIDATION_ERROR`, `FORMAT_NOT_AVAILABLE`,
`DESTINATION_NOT_FOUND`, `DESTINATION_TOKEN_INVALID` when a newly added destination has an
expired login, and `LIVE_PROJECT_RESTRICTION` for the five blocked fields. The
`LIVE_PROJECT_RESTRICTION` message names the fields you tried to change, and the tool adds
the list of all five and tells you to stop or unschedule first.

---

#### `livereacting_get_scenes`

Read the scenes of a project, with a summary of the layers in each one. A scene is one
arrangement of content. At most one scene is active at a time, and the active scene is what
goes out on air. A project can have no active scene at all, because a scene created without
`setActive` is inactive, and such a project streams nothing.

**Use it** to find the layer id of a playlist before `livereacting_get_layer` or
`livereacting_set_playlist`, and to find the scene id before
`livereacting_activate_scene`.

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`. **Scope** `projects:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project |
| `sceneId` | id | no | Read one scene. Leave it out to list every scene |
| `limit` | integer 1 to 20 | no | Scenes per page when listing. Defaults to 20, which is also the maximum |
| `offset` | integer, 0 or more | no | Skip this many scenes when listing |

**Response**

With `sceneId`: `{ scene }`, where the scene carries `id`, `name`, `isActive`, `layers` and
`layerTotal`.

Without `sceneId`: `scenes` plus the paging fields. Each scene carries `id`, `name`,
`isActive`, `layers` and `layerTotal`.

A layer summary carries `id`, `name` and `type`, plus `pinned: true` when the layer is
pinned and `nextScene` when the layer is wired to switch scenes. At most 50 layer summaries
per scene are returned. A scene with more carries `layersNotice` saying so.

**Errors** `PROJECT_NOT_FOUND`, `SCENE_NOT_FOUND`.

---

#### `livereacting_manage_scene`

Create, rename or delete a scene. The `action` argument chooses which.

**Use `action: "create"`** for an empty scene. It holds no content until you add a layer, so
prefer `livereacting_create_scene_with_playlist` when the scene is meant to play video.

**Use `action: "update"`** to rename a scene. It changes nothing else.

**Use `action: "delete"`** to remove a scene permanently, together with every non-pinned
layer inside it, including its playlist. There is no undo. Any other layer set to
auto-switch into the deleted scene has that link cleared, which silently breaks a 24/7
chain, so the response lists the layers it changed. Read it. The last remaining scene
cannot be deleted. Deleting the active scene makes the first remaining scene active.
Confirm with the customer before deleting anything.

**Annotations** `readOnlyHint: false`, `destructiveHint: true`, `idempotentHint: false`, `openWorldHint: true`. A client should ask the
user to confirm. **Scope** `projects:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `action` | `create`, `update` or `delete` | yes | Which operation to run |
| `projectId` | id | yes | The project |
| `sceneId` | id | for `update` and `delete` | Ignored for `create` |
| `name` | string, 1 to 60 chars | for `create` and `update` | The scene name |
| `setActive` | boolean | no | `create` only. Make the new scene active right away. A project needs an active scene to stream |

**Response**

`create`: `action: "create"`, `scene` (`id`, `name`, `isActive`, `layers` holding the
project's pinned layers), and `synced`.

`update`: `action: "update"`, `scene` (`id`, `name`, `isActive`).

`delete`: `action: "delete"`, `deletedSceneId`, `warning`, and `synced`. `warning` is null
when nothing else was touched. Otherwise it carries a `message` and `affectedLayers`, each
entry naming the `layerId`, `sceneId` and `sceneName` whose auto-switch link was cleared, and
the response adds a `nextStep`. A video or audio playlist that switched into the deleted scene
had `loop` forced off when the switch was set, and now has no switch either, so it plays once
and stops. To keep a channel running, call `livereacting_set_playlist` on it with `loop: true`,
or with `autoSwitchToSceneId` set to another scene. Other layer types that switched into the
scene (countdown, trivia, slideshow, single video) can only be linked again in the Studio.

`synced` is `true` when the project has `autoSynch` on, a LIVE or PREPARING broadcast
existed, and the change reached it. It is `false` otherwise, including for a project whose only
broadcast is scheduled: that broadcast reads the project when it starts. See
[autoSynch and synced](#autosynch-and-synced).

**Errors** `PROJECT_NOT_FOUND`, `SCENE_NOT_FOUND`, `SCENE_LIMIT_REACHED` (300 scenes per
project), `LAST_SCENE_DELETE`, and `VALIDATION_ERROR` when `name` or `sceneId` is missing
for the chosen action.

---

#### `livereacting_activate_scene`

Make one scene the active scene, so its content is what goes out on air.

**Use it** to switch what viewers see, including during a broadcast. At most one scene is
active at a time, and activating a scene deactivates the previous one. A project with no
active scene plays nothing, so activate one before starting a stream.

**Annotations** `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: true`, `openWorldHint: true`. Calling it again for the
same scene changes nothing, so it is safe to repeat. **Scope** `projects:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project |
| `sceneId` | id | yes | The scene to put on air, from `livereacting_get_scenes` |

**Response** `activeSceneId` and `synced`.

**Errors** `PROJECT_NOT_FOUND`, `SCENE_NOT_FOUND`.

---

#### `livereacting_create_scene_with_playlist`

Add a scene to a project and fill it with a video playlist in one call. This is the normal
way to put content into a project, because a new project is created empty.

**Use it** for a new scene and its playlist together. To add a playlist to a scene that
already exists, use `livereacting_add_layer`. To change an existing playlist, use
`livereacting_set_playlist`.

It behaves differently depending on what the project already holds, and you do not have to
choose.

- On a project with no scenes it creates the first scene and makes it active. A stream
  needs an active scene in order to play anything.
- On a project that already has a scene it also wires the previous scene to switch
  automatically into this new one when its playlist ends. That is how a 24/7 channel is
  chained day by day. In that case the previous scene must already contain a video playlist
  layer, otherwise the call is refused.

This builds a video playlist. For an audio playlist use `livereacting_add_layer`. Text, image
and interactive overlays are readable but can only be created in the LiveReacting Studio.
Sending audio files here produces a video playlist holding audio, which is not what the
customer wants and is not reported as an error.

Every file must have finished importing and encoding. On the first-scene path the files are
validated before anything is created, so a bad file id does not leave an empty active scene
behind.

**Annotations** `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: false`, `openWorldHint: true`. Calling it twice creates
two scenes. **Scope** `projects:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project |
| `sceneName` | string, 1 to 60 chars | yes | Name for the new scene |
| `files` | array, 1 to 600 entries | yes | The videos this scene plays, in order |
| `files[].fileId` | id | yes | From `livereacting_list_media`, an import, or `livereacting_complete_upload` |
| `files[].name` | string, up to 100 chars | no | Display name. Defaults to the file name without its extension |
| `files[].adPoints` | array | no | Ad breaks inside this video. They must not overlap, and each one must fit inside the file's duration |
| `files[].adPoints[].timestamp` | number, 0 or more | yes | Seconds from the start of this video. It cannot be past the end of the file |
| `files[].adPoints[].duration` | number, 5 to 120 | yes | Ad break length in seconds |
| `layerName` | string, up to 19 chars | no | Name for the playlist layer. Defaults to `Video playlist layer` |
| `loop` | boolean | no | Repeat the playlist forever. Defaults to `true`. Pass `false` for a clip that must play once. Ignored when this scene auto-switches into another one |
| `volume` | number 0 to 10 | no | Playback volume. Defaults to 10 |
| `previousSceneId` | id | no | Which existing scene auto-switches into the new one. Defaults to the last scene in the project. Ignored when the project has no scenes yet |

`loop` defaults to `true` here, which is not the REST default. A 24/7 channel needs it on,
and a playlist that plays once and goes quiet is the most expensive mistake this tool can
make.

**Response**

| Field | Meaning |
|---|---|
| `createdFirstScene` | `true` when this created the project's first scene |
| `scene` | The new scene: `id`, `name`, `isActive` |
| `autoSwitch` | Null on the first-scene path. Otherwise `{ previousScene: { sceneId, sceneName, layerId, layerName } }` |
| `replacedAutoSwitchSceneId`, `warning` | Present only when the previous scene's playlist already switched to another scene. This call replaced that link; the warning says how to restore it with `livereacting_set_playlist` and `autoSwitchToSceneId`, for example to close a 24/7 loop again |
| `synced` | Whether the change reached a LIVE or PREPARING broadcast. A scheduled broadcast reads the project when it starts |
| `layer` | The playlist layer, without its `files` array |
| `currentItem` | The item currently playing, or null |
| `items` | The first page of playlist items |
| `itemTotal`, `offset`, `truncated`, `nextOffset`, `notice` | Paging over `items` |

The scene the previous one now switches into is the one in `scene`.

**Errors** `PROJECT_NOT_FOUND`, `SCENE_NOT_FOUND`, `FILE_NOT_FOUND`, `FILES_NOT_USABLE` (the
tool adds guidance naming the encode tools), `PROJECT_FILE_LIMIT_REACHED`,
`SCENE_LIMIT_REACHED`, `LAYER_LIMIT_REACHED`, `NO_PLAYLIST_LAYER` when the previous scene
holds no video playlist, `ADPOINT_OVERLAP`, and `CONFLICT` when the previous layer changed
during the call.

---

#### `livereacting_add_layer`

Add one layer to a scene that already exists.

**Use it** to put a playlist into a scene that has none, for example a scene made by
`livereacting_manage_scene`, or a background music track alongside a video playlist. For a
new scene and its playlist together use `livereacting_create_scene_with_playlist` instead,
which also wires the auto-switch that chains a 24/7 channel. To change a playlist
afterwards use `livereacting_set_playlist`.

Three layer types can be created. `videoPlaylist` and `audioPlaylist` hold a list of files.
`video` holds a single file. Text, image and interactive layers are readable with
`livereacting_get_layer` but can only be created in the LiveReacting Studio.

**Annotations** `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: false`, `openWorldHint: true`. Calling it twice adds two
layers. After a timeout, read the scene with `livereacting_get_scenes` to see whether the
layer arrived, rather than calling again. **Scope** `projects:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project |
| `sceneId` | id | yes | The scene to add the layer to |
| `type` | `videoPlaylist`, `audioPlaylist` or `video` | yes | What kind of layer |
| `name` | string, up to 19 chars | no | Layer name. Defaults to a name for the type |
| `settings.files` | array, up to 600 entries | no | Playlist types only. Same entry shape as `livereacting_create_scene_with_playlist`. Leave it out and the playlist is created empty |
| `settings.fileId` | id | `video` only | The single file this layer plays. Required for that type |
| `settings.loop` | boolean | no | Defaults to `true` for a playlist and `false` for a single video |
| `settings.volume` | number 0 to 10 | no | Defaults to 10 |

The settings are applied strictly by type, but by dropping rather than by refusing. A `video`
layer drops a `files` array and a playlist layer drops `fileId`, in both cases silently. Send
the field the type uses.

A playlist created without `settings.files` is a valid empty playlist, not an error. Fill it in
the same call, or afterwards with `livereacting_set_playlist`.

**Response**

For a playlist layer: `layer`, `currentItem`, `items`, the paging fields, and `synced`.

For a `video` layer: `layer` and `synced`.

`synced` is `true` only when a LIVE or PREPARING broadcast received the layer. A scheduled
broadcast reads the project when it starts, so it reports `false` and needs no sync call.

**Errors** `PROJECT_NOT_FOUND`, `SCENE_NOT_FOUND`, `FILE_REQUIRED` when a `video` layer has
no `fileId`, `FILE_NOT_FOUND`, `FILE_NOT_USABLE` for a `video` layer or `FILES_NOT_USABLE` for
a playlist, `INVALID_FILE_TYPE` when a `video` layer is given a file that is not a video,
`PROJECT_FILE_LIMIT_REACHED`, `LAYER_LIMIT_REACHED`, `ADPOINT_OVERLAP`.

---

#### `livereacting_get_layer`

Read one layer in full: its type, name, position, settings, and for a playlist layer its
items and which one is playing.

**Use it** before `livereacting_set_playlist`, to get the complete current playlist. Read
every page before you decide what the playlist contains.

Every layer type can be read here, but only `videoPlaylist`, `audioPlaylist` and `video`
layers carry settings. A text, image or interactive layer returns `settings: {}`, so its type,
name, position and flags are all you can see. Those layers can only be edited in the
LiveReacting Studio.

This reads the saved layer document, not the running stream. On a live broadcast the saved
document can lag what viewers see, so `currentItem` and each item's `isPlaying` are not
evidence of what is on air right now.

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`. **Scope** `projects:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project |
| `layerId` | id | yes | From `livereacting_get_scenes` |
| `offset` | integer, 0 or more | no | Skip this many playlist items. Use `nextOffset` from a truncated response |

**Response**

| Field | Meaning |
|---|---|
| `layer` | `id`, `type`, `name`, `hide`, `pinned`, `locked`, `position` (`{ x, y, width, height }`) and `settings`. For a playlist layer the `files` array is removed from `settings` and returned as `items` |
| `currentItem` | The item whose `isPlaying` is true in the saved layer, or null. Not a live playback reading |
| `items` | A page of playlist items. 50 at a time |
| `items[].playlistItemId` | The item's own id, which is regenerated on every playlist write |
| `items[].fileId` | The media file |
| `items[].name`, `items[].duration`, `items[].isPlaying`, `items[].adPoints` | Item detail |
| `itemTotal`, `offset`, `truncated`, `nextOffset`, `notice` | Paging over `items` |

A non-playlist layer returns `layer` only.

A video playlist's `settings` also carry `volume`, `loop`, `nextScene`, `showFileName`,
`showProgressBar`, `textStylePreset`, `textStyles`, `shuffle`, `namePosition` and
`adPlaceholder`. An audio playlist carries the same list without `namePosition` and
`adPlaceholder`, adds `showCoverImage` and `coverImageSize`, and its items have no
`adPoints`. A `video` layer's settings are `fileId`, `fileName`, `volume`, `loop`,
`nextScene`, `isPlaying` and `duration`.

**Errors** `PROJECT_NOT_FOUND`, `LAYER_NOT_FOUND`.

---

#### `livereacting_set_playlist`

Replace the entire contents of a video or audio playlist layer.

**This is a replace, not an append.** The files you send become the whole playlist. Anything
already in it that you do not send again is removed, and playback restarts from the first
item. There is no way to append with this tool.

To add, remove or reorder a video: call `livereacting_get_layer`, page through every item
until `nextOffset` is null, apply your change to that complete list, and send the complete
list back. Sending only the new video deletes everything else.

The response reports `removedItemCount` and adds a `warning` whenever the playlist got
shorter. Check it before telling the customer the change worked.

**One editor at a time.** Because this replaces the whole list rather than merging into it,
two agents editing the same playlist at once will each overwrite the other and both will
report success. Never run parallel edits against one playlist. If the customer may be
editing it in the LiveReacting Studio at the same moment, read the list again immediately
before writing, and tell them not to edit while you work. See
[one editor at a time](#one-editor-at-a-time).

**Annotations** `readOnlyHint: false`, `destructiveHint: true`, `idempotentHint: false`, `openWorldHint: true`. Sending the same complete
list twice produces the same playlist contents, but it also restarts playback and
regenerates every `playlistItemId`, so it is not a safe retry. After a timeout, read the
layer with `livereacting_get_layer` rather than calling again. **Scope** `projects:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project |
| `layerId` | id | yes | A `videoPlaylist` or `audioPlaylist` layer |
| `files` | array, up to 600 entries | yes | The complete playlist in play order. An empty array empties the playlist |
| `loop` | boolean | no | Repeat forever. Refused while the layer auto-switches into another scene, unless the same call sends `autoSwitchToSceneId: null` |
| `volume` | number 0 to 10 | no | Playback volume |
| `autoSwitchToSceneId` | id or null | no | What happens when the playlist ends. A scene id makes it switch to that scene and turns `loop` off. `null` removes the switch; send `loop: true` with it to make the playlist repeat instead. Leave it out to keep the current setting |

The `files` entry shape is the same as `livereacting_create_scene_with_playlist`.

**Loop and auto-switch.** A playlist cannot both repeat and switch to another scene when it
ends. Setting a switch turns `loop` off, and `loop: true` is refused while a switch is set. So
to turn a chained scene into one that loops forever, send `autoSwitchToSceneId: null` and
`loop: true` in the same call. The layer in the response carries `settings.nextScene`, the
scene it now switches to, or null.

**Response**

| Field | Meaning |
|---|---|
| `layer`, `currentItem`, `items`, paging fields | The layer after the write, same shape as `livereacting_get_layer` |
| `previousItemCount` | How many items the playlist held before this call |
| `removedItemCount` | How many items this call removed. Zero when the list grew or stayed the same length |
| `playbackReset` | Always `true`. The playlist restarts from the first item |
| `synced` | Present only when the project has `autoSynch` on and a broadcast existed |
| `warning` | Present only when `removedItemCount` is above zero. Names the numbers and tells you how to append properly |

The tool reads the complete file array itself before writing, not the truncated view you
were shown, so `previousItemCount` and `removedItemCount` are counted against the real
list.

**Errors** `PROJECT_NOT_FOUND`, `LAYER_NOT_FOUND`, `FILE_NOT_FOUND`, `FILES_NOT_USABLE`,
`PROJECT_FILE_LIMIT_REACHED`, `ADPOINT_OVERLAP`, `INVALID_NEXT_SCENE` when
`autoSwitchToSceneId` names a scene the project does not have, and `LOOP_NEXT_SCENE_CONFLICT`
when you pass `loop: true` on a layer that auto-switches without also sending
`autoSwitchToSceneId: null`. Sending `loop: true` together with a scene id is refused with
`VALIDATION_ERROR`. The tool also refuses with `VALIDATION_ERROR` when
the layer has no playlist at all, naming the layer's real type, because the REST layer would
otherwise ignore the write and still answer 200.

---

### streams

The stream tools work on any project, whatever its content. A one-off pre-recorded
broadcast, a multi-destination simulcast, and a stream whose interactive layers were built
in the Studio are all operated the same way.

#### `livereacting_start_stream`

Start streaming a project to its destinations immediately.

**Use it** when the stream should start now. To start it later use
`livereacting_schedule_stream`.

If the project is already running or already scheduled, this returns the existing broadcast
id instead of starting anything.

**Annotations** `readOnlyHint: false`, `destructiveHint: true`, `idempotentHint: false`, `openWorldHint: true`.
**Scope** `streams:write`.

**This call is not idempotent.** If it times out, call `livereacting_get_stream_status` and
read the status. `PREPARING` or `LIVE` means the start succeeded. `OFFLINE` means it did
not, and only then may you call this tool again. A second start can create a duplicate
broadcast attempt.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project to put on air |

**Response when the stream started**

| Field | Meaning |
|---|---|
| `started` | `true` |
| `liveId` | The new broadcast's id |
| `status` | Usually `PREPARING` at this point |
| `destinations[]` | `id`, `provider`, `name`, `status`, `viewingUrl` |
| `pollAfterSeconds` | `5` |
| `nextStep` | Tells you to poll `livereacting_get_stream_status` until the status is `LIVE` |

**Response when the project was already live or scheduled**

This is a normal result, not an error, because the model needs the existing id to work
with.

| Field | Meaning |
|---|---|
| `started` | `false` |
| `alreadyRunning` | `true` when a broadcast is running |
| `alreadyScheduled` | `true` when a schedule exists instead |
| `liveId` | The existing broadcast id, or null |
| `status` | The existing status |
| `scheduledFor` | The scheduled time, when there is one |
| `message` | What to do: read the state, stop the stream, or cancel the schedule |
| `nextStep` | Do not call this tool again for this project |

**Errors** `PROJECT_NOT_FOUND`, `NO_DESTINATIONS` when the project has no destinations,
`DESTINATION_TOKEN_INVALID` when a login has expired, `LIMIT_REACHED` when the account is
already at its concurrent-stream limit,
`PLUGIN_MODE_NOT_SUPPORTED`, `PLATFORM_ERROR` when the destination platform refuses, and
`VALIDATION_ERROR` when the project is not ready.

---

#### `livereacting_stop_stream`

End the stream that is currently running for a project.

**Use it** to end a broadcast. It does not cancel scheduled streams. Use
`livereacting_cancel_schedules` for those.

Every viewer stops watching immediately and the broadcast cannot be resumed. A later start
creates a new broadcast. Confirm with the customer before calling this on a stream you did
not start.

**Annotations** `readOnlyHint: false`, `destructiveHint: true`, `idempotentHint: false`, `openWorldHint: true`. On
a timeout call `livereacting_get_stream_status` rather than calling this again. **Scope**
`streams:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project whose stream should end |

**Response** `message`, `liveId`, `status`, and `actualDuration` in whole seconds, or null
when nothing was recorded.

**Errors** `NO_ACTIVE_STREAM` and `PLATFORM_ERROR`. A project that does not exist also
answers `NO_ACTIVE_STREAM`, so this tool never returns `PROJECT_NOT_FOUND`.

---

#### `livereacting_get_stream_status`

Read what a project's stream is doing right now.

**Use it** after starting, after stopping, and while a stream runs. It is also the tool to
call after any timeout on `livereacting_start_stream` or `livereacting_stop_stream`. For a
finished broadcast use `livereacting_list_broadcasts` instead.

**Poll no more often than every 5 seconds.** A stream needs tens of seconds to reach `LIVE`.
See [polling](#polling).

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `idempotentHint: true`, `openWorldHint: false`.
**Scope** `streams:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project to read |

**Response**

| Field | Meaning |
|---|---|
| `projectId`, `name` | Which project this is |
| `status` | `LIVE`, `PREPARING`, `SCHEDULED` or `OFFLINE` |
| `activeLiveId` | The running broadcast's id, or null |
| `scheduledLivesCount` | How many future schedules exist |
| `unsyncedLive` | True when the project holds edits the broadcast has not received |
| `manualDuration` | True when the project runs until stopped, up to the plan's maximum stream length |
| `plannedSeconds`, `elapsedSeconds`, `remainingSeconds` | Duration, from the project |
| `viewers` | The sum of the current viewers across destinations |
| `peakViewers` | The highest peak any single destination reached, not a sum |
| `destinations[]` | Per-destination state. See below |
| `pollAfterSeconds` | `5` |
| `completed` | Whether the broadcast finished. Present only with an active live id |
| `error` | A mapped error code for the broadcast as a whole, or null. Present only with an active live id |
| `preparingStartedAt` | ISO 8601, when the start began. Present only with an active live id |
| `stopInitiator` | Who or what ended it. Present only with an active live id |
| `destinationsNeedingReconnect` | Names of destinations whose login looks broken. Present only when there are any |
| `notice` | The sentence to pass to the customer when `destinationsNeedingReconnect` is set |

The four fields above and the per-destination detail below appear only while the project has
an `activeLiveId`, which means a running or starting broadcast. A `SCHEDULED` project has no
active live id, so it answers with the project's own destination list: `provider` and `name`,
plus `viewingUrl`, `viewers`, `peakViewers` and `error` only while live. It carries no `id` and
no per-destination status.

While a broadcast exists, `destinations` is built from the snapshot the broadcast started
with, not from the project's current configuration. Those two differ whenever a customer
edits destinations mid-broadcast, and this tool answers "what is my stream doing right
now".

| Destination field | Meaning |
|---|---|
| `id` | The destination id |
| `provider` | `youtube`, `facebook`, `twitch`, `rtmp` and so on |
| `name` | Channel, page or account name |
| `status` | The per-destination stream status, or null |
| `completed` | Whether this destination finished |
| `tokenExpired` | Present only when `true`. See below |
| `error` | `AUTH_ERROR`, `PLATFORM_RATE_LIMITED`, `CONNECTION_ERROR`, `STREAM_ERROR`, `UNKNOWN_ERROR`, or null |
| `startedAt`, `stoppedAt` | ISO 8601, or null |
| `durationSeconds` | How long this destination streamed |
| `viewers`, `peakViewers` | Per destination |
| `viewingUrl` | A public viewing page, or null |

**Reading a failing destination.** Read `error` first. `AUTH_ERROR` means the customer's
connection to that platform is expired or invalid, and nobody can fix it through this API.
Tell them to reconnect that destination in the LiveReacting Studio.

`tokenExpired` appears only when a platform positively reports an expired session, which in
practice only Instagram does. Its absence means there is no signal, not that the connection
is healthy. Custom RTMP, HLS and SRT destinations have no login at all, so they never
report it. When one of those fails the cause is usually a wrong server URL or stream key, or
the receiving end rejecting the connection.

This tool reports only what the platform has recorded. It is not an encoder health check,
and a status of `LIVE` is not by itself proof that video is reaching viewers.

**Errors** `PROJECT_NOT_FOUND`.

---

#### `livereacting_schedule_stream`

Schedule a project to go live at one or more future times. Nothing starts now.

**Use it** instead of `livereacting_start_stream` when the broadcast is for later.

Four rules the API will otherwise only teach you by refusing the call.

1. A project holds one set of schedules. If it already has any, this fails with
   `ACTIVE_SCHEDULES_EXIST`. Call `livereacting_cancel_schedules` first, then schedule
   again. Scheduling does not replace existing schedules.
2. Every date must be at least 3 minutes and at most 28 days in the future.
3. More than one date requires a fixed duration on the project. A manual duration accepts a
   single date only, because the platform cannot tell when the runs would overlap. Set a
   duration with `livereacting_update_project`, or schedule one date at a time.
4. With a fixed duration, one run must end at least 3 minutes before the next one starts.
   The dates are sorted and checked in pairs, and a shorter gap is refused with a message
   naming the earliest time the later run can take.

**Annotations** `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: false`, `openWorldHint: true`.
**Scope** `streams:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project to schedule |
| `dates` | array of ISO 8601 strings, 1 to 50 | yes | Start times, for example `2026-09-01T18:00:00Z` |

**Response**

| Field | Meaning |
|---|---|
| `scheduled[]` | One entry per created schedule: `liveId`, `status`, `scheduledDate`, `createdAt`, `destinations` |
| `count` | How many were created |
| `nextStep` | Nothing is live yet. The status becomes `PREPARING` near the scheduled time |

**Errors** `PROJECT_NOT_FOUND`, `ACTIVE_SCHEDULES_EXIST` (the tool adds "Cancel first, then
schedule. The ordering is required."), `PROJECT_ALREADY_LIVE` when a broadcast is running
rather than scheduled, `MANUAL_DURATION_MULTI_SCHEDULE`, `NO_DESTINATIONS`,
`DESTINATION_TOKEN_INVALID`, `PLUGIN_MODE_NOT_SUPPORTED`, `VALIDATION_ERROR` for a date
outside the 3 minute to 28 day window, `SCHEDULING_CONFLICT` when two runs are too close
together, `LIMIT_REACHED` for the concurrent-stream or schedule-queue limit, and
`PLATFORM_ERROR` when every date failed to create.

Scheduling several dates is not all-or-nothing. Each date is created on its own, and the call
succeeds when at least one of them worked, so read `count` and `scheduled[]` rather than
assuming every date was taken.

---

#### `livereacting_cancel_schedules`

Cancel scheduled streams.

**Use it** with a `liveId` to cancel one schedule, or with only a `projectId` to cancel
every schedule on that project. Call it before `livereacting_schedule_stream` when a project
already has schedules.

This only ever touches `SCHEDULED` broadcasts. It cannot stop a running one. Use
`livereacting_stop_stream` for that. Cancelling cannot be undone. The schedule has to be
created again.

**Annotations** `readOnlyHint: false`, `destructiveHint: true`, `idempotentHint: true`, `openWorldHint: false`.
**Scope** `streams:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | unless `liveId` is given | Cancel every schedule on this project |
| `liveId` | id | no | Cancel this one schedule. Ids come from `livereacting_get_project` with `includeScheduledLives` |

**Response with `liveId`** `cancelled: 1`, `liveId`, `message`.

**Response with `projectId`**

| Field | Meaning |
|---|---|
| `cancelled` | How many schedules were cancelled |
| `failed` | How many could not be cancelled |
| `message` | A plain sentence covering all four cases |

Read `failed`, not only `cancelled`. `cancelled: 0` has two very different causes: there was
nothing to cancel, or every cancellation failed. The `message` tells them apart and says
plainly when the schedules are still going to go live.

**Errors** `PROJECT_NOT_FOUND`, `LIVE_NOT_FOUND`, `VALIDATION_ERROR` when neither
`projectId` nor `liveId` was given, and `INTERNAL_ERROR` when a single `liveId` cancellation
fails. A failure inside the whole-project call is not an error at all: it is counted in
`failed`.

---

#### `livereacting_sync_stream`

Push the project's current setup to the broadcast that is running now.

**Use it** only when the project has `autoSynch` off. With `autoSynch` on, edits reach the
stream by themselves. A running stream uses the copy of the project it was started with, so
edits made while it runs are invisible to viewers until they are synced.

**Annotations** `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: true`, `openWorldHint: true`.
Syncing twice has the same effect as once. **Scope** `streams:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project whose edits should reach the stream |

**Response** `success`, `message`, `liveId`.

**Errors** `PROJECT_NOT_FOUND`, `NO_ACTIVE_LIVE` when nothing is running.

---

#### `livereacting_list_broadcasts`

List a project's finished broadcasts, newest first.

**Use it** to get a `liveId` for `livereacting_get_stream_analytics`. Only finished
broadcasts appear here. Read a running one with `livereacting_get_stream_status`.

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `idempotentHint: true`, `openWorldHint: false`.
**Scope** `streams:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `projectId` | id | yes | The project |
| `limit` | integer 1 to 100 | no | Defaults to 20 |
| `offset` | integer, 0 or more | no | Skip this many |

**Response**

| Field | Meaning |
|---|---|
| `broadcasts[].liveId` | The broadcast id |
| `broadcasts[].date` | When it started, ISO 8601 or null |
| `broadcasts[].endedAt` | When it ended, ISO 8601 or null |
| `broadcasts[].duration` | Whole seconds, or null |
| `broadcasts[].peakViewers` | Highest concurrent viewers, or null |
| `broadcasts[].status` | `COMPLETED` or `ERROR` |
| `broadcasts[].destinations[]` | `provider` and `name` always, plus `viewingUrl`, `peakViewers` and `error` when each has a value |
| `itemTotal`, `offset`, `truncated`, `nextOffset`, `notice` | Paging |

**Errors** `PROJECT_NOT_FOUND`.

---

### media

#### `livereacting_list_media`

List the video, audio and image files in the account's media library, and optionally the
folders they are organised into.

**Use it** to get the file ids a playlist needs. A file whose `issues.errors` is not empty
cannot go in a playlist until the problem is fixed, which usually means encoding it with
`livereacting_encode_media` first.

To add a file, use `livereacting_upload_file` for a local file, `livereacting_import_from_url`
for a direct link, or `livereacting_import_media` for a Google Drive, Dropbox or YouTube link.

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`. **Scope** `media:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `type` | `video`, `image` or `audio` | required when `includeFolders` is true | Only files of this type |
| `folderId` | string | no | Only files inside this folder. Pass the string `"null"` for files at the root. Omit for every folder |
| `search` | string, 3 to 100 chars | no | Match against the file name |
| `limit` | integer 1 to 200 | no | Defaults to 50 |
| `offset` | integer, 0 or more | no | Skip this many files |
| `includeFolders` | boolean | no | Also return folders. Requires `type`. Defaults to false |
| `folderLimit` | integer 1 to 200 | no | Defaults to 50 |
| `folderOffset` | integer, 0 or more | no | Folders page independently of files |

**Response**

| Field | Meaning |
|---|---|
| `files[].id`, `files[].name`, `files[].type`, `files[].createdAt` | Always present |
| `files[].folderId` | Only when the file is in a folder |
| `files[].issues` | Video files only. `{ errors: [], warnings: [] }`. See [file issues](#file-issues) |
| `files[].encoding` | Only while an encode is queued or running. `{ status, progress }` |
| `fileTotal` | How many files match |
| `limit` | The page size used |
| `itemTotal`, `offset`, `truncated`, `nextOffset`, `notice` | Paging over files |
| `folders` | Only with `includeFolders`. `{ folders, folderTotal, folderOffset, truncated, nextFolderOffset, notice }` |
| `folders.folders[]` | `id`, `name`, `type`, `fileCount`, `createdAt` |

Files and folders are paged separately on purpose. Merging them into one page would hide how
much of either you are seeing.

---

#### `livereacting_get_media_file`

Read one file from the media library by id.

**Use it** when you need the technical detail of one file, for example to decide whether it
needs `livereacting_encode_media` before it can be streamed. To list files use
`livereacting_list_media`.

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`. **Scope** `media:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `fileId` | id | yes | From `livereacting_list_media` |
| `includeMetadata` | boolean | no | Include resolution, frame rate and codec detail. Defaults to false |

**Response** `id`, `name`, `type`, `createdAt`, plus `folderId`, `issues` and `encoding`
under the same rules as `livereacting_list_media`. With `includeMetadata` and stored
codecs, it also carries `resolution`, `frameRate`, `bitrateMbps`, `durationSec`,
`sizeBytes`, `videoCodec` and `audioCodec`. Each of those is omitted when the underlying
value is missing.

Download links and thumbnails are not returned by any tool.

**Errors** `FILE_NOT_FOUND`.

---

#### `livereacting_import_media`

Bring a file into the media library from a Google Drive, Dropbox or YouTube URL.

**Use it** when the customer's video is on Google Drive, Dropbox or YouTube. For a direct
link to a media file on any other host use `livereacting_import_from_url`, and for a file on
a local disk use `livereacting_upload_file`.

**YouTube playlist and channel URLs are rejected.** Only a single YouTube video imports
through MCP. The underlying REST import route has no way to choose which videos of a
playlist to take, so the tool refuses the URL with a message saying to pass a single video
URL instead. Do not promise a customer that a playlist can be imported here.

The import runs in the background. This returns a `fileId` straight away. Poll
`livereacting_get_import_status` with it to find out when the file is ready. Only one
import can run at a time per account.

**Annotations** `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: false`, `openWorldHint: true`.
Calling it twice with the same URL imports the file twice. **Scope** `media:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `url` | string | yes | A Google Drive, Dropbox or single-video YouTube URL |
| `type` | `video`, `image` or `audio` | yes | What kind of media this is. A video playlist does not enforce it: the playlist tools accept a file of any type, and only video files are checked for usability |
| `folderId` | string or null | no | Put the file in this folder. Omit to import to the root of the library |

**Response** `fileId`, `status` (always `in_progress` at this point), `pollWith`
(`livereacting_get_import_status`) and `pollAfterSeconds` (`10`).

**Errors** `VALIDATION_ERROR` for a YouTube playlist or channel URL. `TRANSFER_IN_PROGRESS` when another import is already running.
`STORAGE_LIMIT_EXCEEDED` when the account is out of storage. `FOLDER_NOT_FOUND`.
`IMPORT_FAILED` when the transfer service refuses the job.

---

#### `livereacting_import_from_url`

Bring a file into the media library from a direct `http` or `https` link to a video, audio or
image file on any public host, for example a CDN or storage bucket URL.

**Use it** when the customer has a public link that returns the media file itself. Google
Drive, Dropbox and YouTube links go to `livereacting_import_media`; this tool refuses them
with `VALIDATION_ERROR`. The content type of the response decides the media type (an
`application/octet-stream` answer falls back to the file extension), and it must match
`type`. A web page is refused.

Links that resolve to a private or internal address are refused with `URL_NOT_ALLOWED`,
also when a redirect leads there. If the address turns private while the file downloads, the
download stops and `livereacting_get_import_status` reports `error`. The file must fit the
plan's max file size, and a video counts against the storage quota.

The import runs in the background and is polled exactly like `livereacting_import_media`.
Only one import can run at a time per account.

**Annotations** `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: false`, `openWorldHint: true`.
Calling it twice imports the file twice. **Scope** `media:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `url` | string | yes | A direct `http(s)` link to the media file |
| `type` | `video`, `image` or `audio` | yes | What kind of media this is |
| `folderId` | string or null | no | Put the file in this folder. Omit to import to the root of the library |

**Response** `fileId`, `status` (`in_progress`), `pollWith`
(`livereacting_get_import_status`) and `pollAfterSeconds` (`10`).

**Errors** `VALIDATION_ERROR` for a non-http(s) URL or a Drive, Dropbox or YouTube link.
`URL_NOT_ALLOWED` for a private or internal address. `URL_NOT_RESOLVED` for a host that does not exist. `IMPORT_FAILED` with the reason when the
link is not a usable media file or is too large. `TRANSFER_IN_PROGRESS`,
`STORAGE_LIMIT_EXCEEDED`, `FOLDER_NOT_FOUND` as for `livereacting_import_media`.

---

#### `livereacting_upload_file`

Start uploading a file from the machine the agent runs on (or the customer's) into the media
library.

**Use it** when the file is on a local disk. This server cannot read local files, so the
upload takes three steps:

1. Call this tool with the file name, type and exact size in bytes.
2. Send each part with an HTTP `PUT` to its URL. `commands` holds one ready shell command
   per part; set `FILE` to the local path first. Each command cuts only its own byte range
   with `dd` into its own `mktemp` file, uploads it and prints `part N: <HTTP status>`, so a
   large file needs free disk for one part at a time, parts can run in parallel, and a failed
   part can be run again on its own. A command exits non-zero when its part failed, and stops a
   transfer that stays below 1 KB/s for 60 seconds.
   Each part must be exactly its `size` in bytes, or storage refuses it with `403`. A file up
   to 256 MiB is one part.
3. Call `livereacting_complete_upload` with `uploadToken`.

An agent that cannot run commands or make HTTP `PUT` requests cannot upload. Ask the customer
for a public link and use `livereacting_import_from_url`, or have them upload in the Studio.

Same limits as a Studio upload: the extension must match `type` (video `.mp4 .m4v .mov .flv
.webm .ogv .3gp .wmv .avi .mkv`, audio `.mp3 .m4a .wav`, image `.jpg .jpeg .png .gif .bmp .svg
.heic .webp`), the size must fit the plan's max file size, and a video is refused once
storage is full. The part URLs and the token expire after 3 days.

**Annotations** `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: false`, `openWorldHint: false`.
Each call opens a new upload. **Scope** `media:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `name` | string, 1 to 255 chars | yes | File name with its extension |
| `type` | `video`, `image` or `audio` | yes | What kind of media this is |
| `size` | integer | yes | Exact file size in bytes |
| `folderId` | string or null | no | Put the file in this folder. Omit for the root of the library |

**Response** `uploadToken`, `partSize`, `parts[]` (`partNumber`, `size`; each part's URL is inside its command), `expiresAt`,
`commands` (one shell command per part) and `nextStep`.

**Errors** `UNSUPPORTED_FILE_TYPE`, `FILE_TOO_LARGE`, `STORAGE_LIMIT_EXCEEDED`,
`FOLDER_NOT_FOUND`, `UPLOAD_NOT_SUPPORTED`.

---

#### `livereacting_complete_upload`

Finish an upload started with `livereacting_upload_file`, after every part has been sent.

**Use it** once every `PUT` has answered `200`. The server checks that all parts are present
and add up to the declared size, then adds the file to the library and returns it. A video
goes through the same checks, thumbnail and encoding as a Studio upload.

**Annotations** `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: true`, `openWorldHint: false`.
A second call with the same token returns the same file, so a retry is safe. A retry that arrives
while the first call is still checking the file (up to about 30 seconds for a video) answers
`UPLOAD_PROCESSING`: wait 30 seconds and call again. If the first call was interrupted, this can
repeat for about five minutes before a retry finishes the upload. Do not upload again. **Scope** `media:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `uploadToken` | string | yes | From `livereacting_upload_file` |

**Response** the file, in the shape of `livereacting_get_media_file`. When `issues.errors` is
not empty, `nextStep` names `livereacting_encode_media` or `livereacting_get_encode_status`,
and says to read the file again afterwards: a finished encode can leave other issues.

**Errors** `UPLOAD_INCOMPLETE` names the missing parts: send them again and call this tool
again with the same token. `UPLOAD_PROCESSING` while the first call is still checking the file.
`UPLOAD_NOT_FOUND` for an expired or foreign upload, or one whose check failed earlier. `FILE_TOO_LARGE`
when the plan's max file size dropped below the file's size during the upload.
`UPLOAD_INVALID` when the file is not readable media. `STORAGE_LIMIT_EXCEEDED` when storage
filled up during the upload. After `FILE_TOO_LARGE` or `STORAGE_LIMIT_EXCEEDED` this upload
cannot be completed: once the customer frees space or upgrades, start again with `livereacting_upload_file`.

---

#### `livereacting_get_import_status`

Check how far an import started by `livereacting_import_media` or
`livereacting_import_from_url` has got.

**Use it** in a poll loop after starting an import. Poll no more often than once every 10
seconds. See [polling](#polling).

**`done: true` is not success.** It means the transfer stopped, and it covers `error` as well
as `completed`. Read `status` to tell them apart: a failed import needs another call to the tool
that started it (`livereacting_import_media` or `livereacting_import_from_url`), never an encode.

**A finished transfer is not a usable file either.** An imported video very often still needs
encoding, and a playlist built with it is refused until that is done. Read `usable`, which is
true only when the transfer completed and the file can go straight into a playlist, and
`nextStep`, which names the tool to call when it cannot.

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `idempotentHint: true`, `openWorldHint: false`. This tool writes nothing.
**Scope** `media:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `fileId` | id | yes | The `fileId` returned by `livereacting_import_media` or `livereacting_import_from_url` |

**Response**

| Field | Meaning |
|---|---|
| `fileId` | The id you asked about |
| `status` | `in_progress`, `completed`, `error` or `stopped` |
| `progress` | Percentage, 0 to 100 |
| `error` | The reason the import failed. It can also be present as null after a successful import, so read `status`, not the presence of this field |
| `done` | `true` when `status` is `completed` or `error`. A `stopped` import leaves it `false` |
| `pollAfterSeconds` | `10` while running or stopped, `15` while an encode runs, null once `done` is `true` |
| `issues` | Single files only, and only once `done` is `true`. `{ errors: [], warnings: [] }` |
| `usable` | Single files only, and only once `done` is `true`. `true` only when the transfer completed and `issues.errors` is empty |
| `nextStep` | Single files only, and only once `done` is `true`. Present when the file is not usable. Names the tool to call |

A `stopped` import never reaches `done`. It carries no `issues`, no `usable` and no `nextStep`,
and polling it again returns the same answer. Treat it as a failed import and import the URL
again.

The `fileId` keeps working after the import finishes and can be polled again at any time.

**Multi-video imports.** A YouTube playlist cannot be imported through this server, but an
account may hold one started in the LiveReacting Studio. Polling such an id answers for the
whole group: `isFolder` is `true`, `totalFiles` and `processedFiles` count the videos, and
`files` lists each one as `{ name, status, progress }`. A `note` says so. The group answer
carries no `usable` or `issues`, because those would be meaningless for a group. The `files`
list names the videos but not their ids. Call `livereacting_list_media` to get those. The
list reflects the videos that exist now, so it shrinks if one is deleted later. A Google Drive
or Dropbox folder import gets the same `note`, and no `usable`, once it has finished.

**Errors** `FILE_NOT_FOUND`.

---

#### `livereacting_encode_media`

Queue video files for encoding so they can be streamed.

**Use it** when a file's `issues.errors` names a problem that encoding fixes:
`fps_exceeds_limit`, `resolution_too_high`, `bitrate_too_high`, `moov_atom_position`,
`unsupported_video_codec` or `unsupported_audio_codec`. Use
`livereacting_get_media_file` with `includeMetadata` to see why.

Encoding runs in the background. This returns immediately. Poll
`livereacting_get_encode_status` to find out when the files are ready.

**Annotations** `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: false`, `openWorldHint: false`.
Queueing a file that is already queued or encoding is skipped rather than duplicated, but a
file that has already finished encoding is encoded again. After a timeout, check
`livereacting_get_encode_status` before you call this again. **Scope** `media:write`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `fileIds` | array of ids, 1 to 100 | yes | Files to encode |

**Response**

| Field | Meaning |
|---|---|
| `queued[]` | `{ fileId, status: "in_queue" }` for each file accepted |
| `skipped[]` | `{ fileId, reason }`. `reason` is `not_found`, `already_encoding`, `not_video` or `unknown` |
| `message` | How many were queued |
| `pollWith` | `livereacting_get_encode_status` |
| `pollAfterSeconds` | `15` |

**Errors** `VALIDATION_ERROR` for a malformed list. Individual files are reported in
`skipped`, not as errors.

---

#### `livereacting_get_encode_status`

List every file in the account that is queued for encoding or being encoded right now.

**Use it** in a poll loop after `livereacting_encode_media`. Poll no more often than once
every 15 seconds. Encoding a long video takes minutes, not seconds. See
[polling](#polling).

Read `done`, or `summary`, to decide whether to keep waiting. An empty `encoding` page does
not mean the work finished: an `offset` past the end of the list returns no rows while files
are still queued.

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`. **Scope** `media:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `limit` | integer 1 to 200 | no | Defaults to 50 |
| `offset` | integer, 0 or more | no | Skip this many files |

**Response**

| Field | Meaning |
|---|---|
| `encoding[]` | `{ fileId, name, status, progress, resolution }`. `status` is `in_queue` or `in_progress` |
| `summary` | `{ inQueue, inProgress }` |
| `done` | `true` when `inQueue` plus `inProgress` is zero |
| `pollAfterSeconds` | `15`, or null when `done` is true |
| `itemTotal`, `offset`, `truncated`, `nextOffset`, `notice` | Paging over `encoding` |

This tool answers for the whole account, not for one file. After an encode finishes, read
the file again with `livereacting_get_media_file` or
`livereacting_get_import_status` to confirm it is usable. A failed encode leaves the file
unusable and the encode list empty.

---

### analytics

#### `livereacting_get_stream_analytics`

Read the analytics of one finished broadcast. Get the broadcast id from
`livereacting_list_broadcasts`.

**`type: "youtube"` is included with every paid plan. `type: "playlists"` is available on
request** rather than on every plan, limited to a list of accounts we have enabled, so it can
return `FEATURE_NOT_AVAILABLE` for an account that is otherwise working normally. Tell the
customer to email [hello@livereacting.com](mailto:hello@livereacting.com) to ask for it. That
check comes before the broadcast lookup, so an account outside the list gets
`FEATURE_NOT_AVAILABLE` even for a broadcast id that does not exist.

**Use `type: "youtube"`** for what YouTube reports about the broadcast: views, watch time,
likes, peak and average concurrent viewers, revenue where it is available, plus traffic
sources, daily trend, age and gender split, devices and geography. The channel must also have
granted analytics access.

**Use `type: "playlists"`** for what LiveReacting itself recorded: one row per playback action,
naming the file, the action, when it happened and how many people were watching at that
moment.

There is no CSV export here. Use the REST API for that.

**Annotations** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`. **Scope** `analytics:read`.

| Input | Type | Required | Meaning |
|---|---|---|---|
| `liveId` | id | yes | Broadcast id |
| `type` | `youtube` or `playlists` | yes | Which source to read |
| `startDate` | `YYYY-MM-DD` | no | First day to report on. For `playlists`, the 28-day limit is checked only when `startDate` and `endDate` are both sent. `youtube` has no local range check |
| `endDate` | `YYYY-MM-DD` | no | Last day to report on |
| `accountId` | string | no | `youtube` only. Which channel to report on when the broadcast went to more than one |
| `sections` | array of section names | no | `youtube` only. Which row-shaped sections to include |
| `limit` | integer 1 to 200 | no | Rows per page. Defaults to 100 for `playlists` and 50 per section for `youtube` |
| `offset` | integer, 0 or more | no | Skip this many rows |

Valid `sections` values are `trafficSources`, `dailyTrend`, `ageGroups`, `genders`,
`devices` and `geography`. Nothing in that list is returned unless you ask for it.

**Response for `type: "youtube"`**

| Field | Meaning |
|---|---|
| `type` | `"youtube"` |
| `overview` | Views, watch time, likes and the other headline numbers |
| `livestream` | Peak and average concurrent viewers |
| `revenue` | Revenue, where YouTube reports it |
| `meta` | The report's own metadata, including the date range used |
| `sections` | One key per section you asked for. Each holds `rows`, `itemTotal`, `offset`, `truncated`, `nextOffset` and `notice` |
| `sectionsNotReturned` | The section names you did not ask for. Present only when some were left out |
| `notice` | How to ask for one of them |

**Response for `type: "playlists"`**

| Field | Meaning |
|---|---|
| `type` | `"playlists"` |
| `liveId`, `projectId` | Which broadcast this is |
| `rows` | One row per recorded playback action: `fileId`, `fileName`, `sceneName`, `action`, `timestamp` and `concurrentLiveViews`. A file that started and stopped has a row for each |
| `itemTotal`, `offset`, `truncated`, `nextOffset`, `notice` | Paging over `rows` |

**Errors** `LIVE_NOT_FOUND`; `VALIDATION_ERROR` for a bad date, on `playlists` with both
dates sent for a range longer than 28 days, and on `youtube` in two cases the message tells
apart: "Broadcast has no YouTube destination" when nothing went to YouTube, and a message
asking the customer to link the YouTube video in the Studio when the broadcast reached YouTube
through a custom RTMP destination; `FEATURE_NOT_AVAILABLE` when the account is not
enabled for `type: "playlists"` or has no active paid plan for `type: "youtube"`, `YOUTUBE_ANALYTICS_CONSENT_REQUIRED` when the channel has
not granted access, `ACCOUNT_SELECTION_REQUIRED` when the broadcast went to more than one
YouTube channel, and `PLATFORM_ERROR` when YouTube itself fails. Neither path reads the
project, so `PROJECT_NOT_FOUND` never comes back from this tool.

`ACCOUNT_SELECTION_REQUIRED` carries the ids to choose between in an `availableAccounts`
field, relayed in the error text. Pass one of them as `accountId` and call again. A running
broadcast read with `livereacting_get_stream_status` does carry the same ids in
`destinations[].id`, but that list holds every destination, so prefer the ids the error itself
names.

---

## 5. Behaviours that apply across tools

### Polling

Long work returns an id straight away and is then polled. Each polling tool carries the
interval in its own response as `pollAfterSeconds`, so one call answers both "is it done"
and "when should I ask again".

| Tool | Interval | Why |
|---|---|---|
| `livereacting_get_stream_status` | 5 seconds | A stream needs tens of seconds to reach `LIVE` |
| `livereacting_get_import_status` | 10 seconds | A transfer takes minutes |
| `livereacting_get_import_status`, while an encode runs | 15 seconds | It is waiting on the encoder, not the transfer |
| `livereacting_get_encode_status` | 15 seconds | Encoding a long video takes minutes |

The intervals exist because of the rate limit below. Polling tighter than this spends the
account's own budget and rate-limits the customer's other work.

Progress notifications are not used. They reach the client's user interface, not the model,
so they cannot signal completion.

### Rate limits

**150 tool calls per minute per account.** The number is the same one the REST API
publishes, and it is counted per tool call rather than per HTTP request. The `initialize`
and `tools/list` handshake is free.

REST and MCP are counted separately, so the two surfaces get 150 each rather than sharing
one budget. Treat that as an implementation detail of how the two are metered, not a
promise.

A JSON-RPC batch is capped at 20 messages per request. A larger batch is refused with
JSON-RPC error `-32600` and a message telling you to send tool calls one at a time. No
client we target batches tool calls.

There is also a pre-authentication flood limit of 3000 requests per minute per source
address on the endpoint. It is a flood backstop, not a quota. It runs before authentication,
because nothing else can see an unauthenticated request. A rejection there is an HTTP 429
with JSON-RPC error `-32000` and the message "Too many requests from this address. Slow
down."

When the account limit is reached on a single tool call, the server answers HTTP 200 with a
tool error, because that is what a model reads and acts on. A transport-level 429 would be
handled by the client, and some clients retry it, which is the one thing that must not
happen here. The text the model reads is:

```
Rate limit reached: 150 requests per minute for this account. Wait N seconds before
calling another tool. If you are polling a stream, import or encode, poll less often
rather than retrying now.
```

Inside a batch, the same text comes back as a JSON-RPC error with HTTP 429 instead.

The limiter uses a per-process store, so the real ceiling across several web dynos is higher
than 150. Code against 150. The server allows more than it promises, which breaks nothing.

### Operations that are not idempotent

These tools are not safe to repeat.

| Tool | What a repeat does | What to do after a timeout |
|---|---|---|
| `livereacting_start_stream` | Can create a duplicate broadcast attempt | Call `livereacting_get_stream_status`. `PREPARING` or `LIVE` means it worked. `OFFLINE` means it did not |
| `livereacting_stop_stream` | Acts on a broadcast whose state you no longer know | Call `livereacting_get_stream_status` |
| `livereacting_set_playlist` | Restarts playback and regenerates every `playlistItemId` | Call `livereacting_get_layer` |
| `livereacting_import_media`, `livereacting_import_from_url` | Imports the same URL a second time, as a second file | Call `livereacting_list_media` and look for the file |
| `livereacting_upload_file` | Opens a second upload; the first expires unused | Nothing to check: start the `PUT`s with the newest token |
| `livereacting_schedule_stream` | Adds a second set of schedules, and a partly successful call already created some | Call `livereacting_get_project` with `includeScheduledLives` |
| `livereacting_add_layer`, `livereacting_create_scene_with_playlist`, `livereacting_manage_scene` with `create`, `livereacting_create_project` | Creates a second one | Read the project or scene back |

The rule is the same in every case: after a timeout, look, do not retry.

`livereacting_complete_upload` is the exception: it is safe to repeat. After a timeout, call it
again with the same `uploadToken`. It returns the same file.

### Truncation and paging

Claude Code truncates a tool response at 25,000 tokens. A 600-entry playlist is well past
that, so truncation is not a nicety. Every truncated response says so in words. A model that
cannot see the truncation assumes it holds the whole list, and that is exactly how a
playlist gets wiped.

Default page sizes:

| Kind of item | Default | Where |
|---|---|---|
| Projects | 20 | `livereacting_list_projects` |
| Scenes | 20 | `livereacting_get_scenes`, and `livereacting_get_project` with `includeScenes` |
| Layer summaries per scene | 50 | `livereacting_get_scenes`, and `livereacting_get_project` with `includeScenes`. Scene-creation responses are not capped: they list every pinned layer |
| Playlist items | 50 | `livereacting_get_layer`, `livereacting_set_playlist`, `livereacting_add_layer`, `livereacting_create_scene_with_playlist` |
| Media files and folders | 50 | `livereacting_list_media`, `livereacting_get_encode_status` |
| Analytics rows | 100 for `playlists`, 50 per section for `youtube` | `livereacting_get_stream_analytics` |
| Broadcasts | 20 | `livereacting_list_broadcasts` |

Every paged response uses the same five fields, whether the page was cut here or by the REST
layer.

| Field | Meaning |
|---|---|
| `itemTotal` | How many items exist in total |
| `offset` | Where this page starts |
| `truncated` | `true` when more items exist than this page holds |
| `nextOffset` | The `offset` to send for the next page, or null when this is the last page |
| `notice` | Present only when `truncated` is true. For example "Showing 50 of 612 playlist items. Call again with offset: 50 for the rest." |

Scenes and folders carry the same information under names of their own where two lists share
one response: `sceneTotal`, `scenesTruncated` and `scenesNotice` on
`livereacting_get_project`, `layerTotal` and `layersNotice` per scene, and `folderTotal`,
`folderOffset` and `nextFolderOffset` on `livereacting_list_media`.

**Read every page before you write a playlist.** `livereacting_set_playlist` replaces the
whole list, so a write based on one page deletes the rest. The tool itself reads the
complete array before writing, which is what makes `removedItemCount` honest, but it cannot
know what you meant to keep.

### autoSynch and synced

A running broadcast uses the copy of the project it was started with. Edits made while it
runs do not reach viewers by themselves unless the project has `autoSynch` on.

- `autoSynch: true` on the project: an edit reaches the running broadcast immediately, unless
  the sync itself fails. A failed sync is not an error: the edit is saved and the tool reports
  `synced: false`.
- `autoSynch: false`: the edit stays in the project until you call
  `livereacting_sync_stream`. `livereacting_get_project` and
  `livereacting_get_stream_status` report `unsyncedLive: true` while that is pending.

Tools that change what is on air report the outcome in a `synced` field:
`livereacting_activate_scene`, `livereacting_set_playlist`, `livereacting_add_layer`,
`livereacting_create_scene_with_playlist`, and `livereacting_manage_scene` for `create` and
`delete`.

`synced: true` means the project has `autoSynch` on, a broadcast existed, and the change reached
it. `synced: false` means it did not: nothing was running, `autoSynch` is off, or the sync was
attempted and failed. The write itself succeeded in every case, so `synced: false` on a live
project means the broadcast is running older content until you call `livereacting_sync_stream`.
On `livereacting_set_playlist` the field is present only when the project has `autoSynch` on
and there was a broadcast to sync with; with `autoSynch` off it is absent, not `false`.

Which broadcasts count differs by tool. `livereacting_activate_scene` and
`livereacting_set_playlist` also update a scheduled broadcast. `livereacting_manage_scene`,
`livereacting_add_layer` and `livereacting_create_scene_with_playlist` sync only a LIVE or
PREPARING broadcast; for a project whose only broadcast is scheduled they report
`synced: false`, and that is fine: these writes mark the project as changed, and a scheduled
broadcast reads the project's current content when it starts. No sync call is needed.

When a running broadcast and a scheduled one exist at the same time, these tools sync the
running one and report `synced: true`, but `unsyncedLive` stays `true` on the project: the
scheduled broadcast has not received the change yet, and it picks it up when it starts.

**Several scheduled dates.** "Picks it up when it starts" holds for the next broadcast only.
When a project is scheduled on several dates, an edit reaches the broadcast that is running,
or the next one to start. The later scheduled broadcasts keep the content they had when they
were scheduled, by design. To change those too, cancel the schedules with
`livereacting_cancel_schedules` and schedule the dates again, or tell the customer to use
"Apply changes to all scheduled streams" in the LiveReacting Studio.

### One editor at a time

`livereacting_set_playlist` replaces the whole playlist rather than merging into it, and it
carries no version or hash. Two writers editing the same playlist at once will each
overwrite the other, and both will be told they succeeded.

This is by design, and it has two consequences worth stating plainly.

- Never run parallel agent edits against one playlist.
- While a project is streaming with `autoSynch` on, do not run edits to it in parallel at all.
  Two edits that reach the running stream at the same moment can overwrite each other there,
  and the project may still report itself as in sync. This is a known issue. If it happens,
  `livereacting_sync_stream` copies the project into the stream again.
- If the customer may be editing the same playlist in the LiveReacting Studio, read the list
  again immediately before writing, and ask them not to edit while the agent works. A
  concurrent Studio edit is still lost if it lands in the gap.

The same care applies to `livereacting_add_layer` and
`livereacting_create_scene_with_playlist` running against one project at the same time.

### File issues

Video files carry an `issues` object, `{ errors: [], warnings: [] }`. A file with any entry
in `errors` cannot go into a playlist. Warnings do not block anything.

| Error | Meaning | What fixes it |
|---|---|---|
| `transfer_failed` | The import failed | Import again |
| `import_in_progress` | The transfer is still running | Poll `livereacting_get_import_status` |
| `encoding_in_progress` | An encode is already running | Poll `livereacting_get_encode_status`, then read the file again. A failed encode drops this entry and leaves the errors that made encoding necessary, so check `usable` rather than assuming the encode worked |
| `moov_atom_position` | The file's index is at the end | `livereacting_encode_media` |
| `unsupported_video_codec` | Not H.264 | `livereacting_encode_media` |
| `unsupported_audio_codec` | Not AAC | `livereacting_encode_media` |
| `bitrate_too_high` | Above the plan's ceiling | `livereacting_encode_media` |
| `resolution_too_high` | Above the plan's ceiling | `livereacting_encode_media` |
| `fps_exceeds_limit` | Above the account's frame-rate ceiling | `livereacting_encode_media` |

The one warning you will see is `high_bitrate`: the file is close to the plan's bitrate
ceiling. It does not block a playlist. `high_resolution` exists in the code but cannot happen
today, because the recommended width and the maximum width are the same number for every plan
tier, so a file above it is an error rather than a warning.

The thresholds depend on the account's plan tier (720p, 1080p or 4K) and on whether the file
is 30fps or 60fps.

### Author versus operate

The line this server draws is between building content and operating a stream.

**Authoring is limited.** `videoPlaylist`, `audioPlaylist` and `video` layers can be created.
Only the two playlist types can be changed afterwards, because `livereacting_set_playlist` is
the only layer-write tool and it refuses a layer that has no playlist. A single `video` layer
can be created here but not edited here. Text, image and interactive layers can only be built
in the LiveReacting Studio, and they read back with an empty `settings` object, so only their
type, name, position and flags are visible.

**Operating is not limited that way.** A stream whose overlays and interactive layers were
built in the Studio can still be started, stopped, scheduled, synced, monitored and analysed
here, and every one of its layers is listed. An agent cannot build an interactive overlay,
but it can run the show that uses one.

---

## 6. The two journeys

These are the journeys demonstrated end to end. They are not the limit of what works. Every
stream tool operates on any project, whatever its content.

### A 24/7 looping channel

1. `livereacting_list_destinations` to get destination ids.
2. `livereacting_validate_destinations` with those ids, so a broken login is found now.
3. `livereacting_list_media`, or `livereacting_import_media` for each new video.
4. For each import, poll `livereacting_get_import_status` every 10 seconds until `done` is
   `true`. **Then read `status`.** `done` covers failure as well as success: on `error` the
   import failed and has to be run again with the tool that started it, not encoded. On
   `completed`, read `usable`. `done` means the bytes arrived; it does not mean the file can
   be streamed. When `usable` is false, call `livereacting_encode_media` with the `fileId`,
   poll `livereacting_get_encode_status` every 15 seconds until `done`, then poll
   `livereacting_get_import_status` again and check `usable` a second time. A failed encode
   leaves the original errors in place.
5. `livereacting_create_project` with `destinationIds`, `duration: { manual: true }` and
   `autoSynch: true`.
6. `livereacting_create_scene_with_playlist` with the file ids in play order.
   `loop` defaults to `true`, which is what a 24/7 channel needs. On a fresh project this
   creates the first scene and makes it active.
7. `livereacting_start_stream` with the project id.
8. Poll `livereacting_get_stream_status` every 5 seconds until `status` is `LIVE`.

To add the next day's content to the same channel, call
`livereacting_create_scene_with_playlist` again on the same project. On a project that
already has a scene it wires the previous scene to switch into the new one when its playlist
ends. The previous scene must already hold a video playlist layer.

### A scheduled pre-recorded stream

1 to 4 are the same media preparation as above.

5. `livereacting_create_project` with `destinationIds` and
   `duration: { manual: false, planned: <seconds> }`. Give `planned` the real length of the
   video in seconds.
6. `livereacting_create_scene_with_playlist` with `loop: false`, so the clip plays once and
   stops.
7. `livereacting_schedule_stream` with ISO 8601 dates, each between 3 minutes and 28 days
   ahead. More than one date needs the fixed duration set in step 5, and each run must end at
   least 3 minutes before the next one starts. If the project already has schedules the call
   fails with `ACTIVE_SCHEDULES_EXIST`. Call `livereacting_cancel_schedules` first, then
   schedule again. Read `count` afterwards: the dates are created one by one, so a call can
   take some of them and refuse the rest.
8. Poll `livereacting_get_stream_status` when the time approaches. The status becomes
   `PREPARING` near the scheduled time and then `LIVE`.

---

## 7. Limits stated honestly

An agent that promises any of these is lying to the customer.

**Authoring covers video and audio playlists and single video layers only.** A single video
layer can be created but not changed afterwards, because the only layer-write tool needs a
playlist. Text, image and interactive layers are listed through the API, with an empty
`settings` object, and can only be created or edited in the
[LiveReacting Studio](https://studio.livereacting.com).

**Operating is not limited that way.** See [author versus operate](#author-versus-operate).

**Uploading a local file needs a shell or an HTTP client.** This server cannot read the agent's
disk. `livereacting_upload_file` returns signed part URLs and ready curl commands; the agent sends
the bytes itself, then calls `livereacting_complete_upload`.

**Playlist and channel URLs cannot be imported.** Only a single YouTube video imports
through `livereacting_import_media`. The REST import route has no way to select which videos
of a playlist to take.

**Every plan has a maximum stream length.** A stream stops when it reaches the length of the
account's plan, even with a manual duration. A 24/7 channel needs a plan without a length limit, such as the 24/7 plan.

**A Facebook-only 24/7 channel is not possible.** A Facebook-only stream stops after a
maximum of 8 hours, whatever duration the project asks for. Add another destination for
longer runs.

**Thumbnails and live announcements are not supported** through the API.

**YouTube analytics is included with every paid plan. Playlist analytics is available on
request**, limited to a list of accounts we have enabled. Tell the customer to email
[hello@livereacting.com](mailto:hello@livereacting.com) to ask for it. YouTube analytics also
needs the channel to have granted analytics access.

**Scheduling has a window.** Every date must be at least 3 minutes and at most 28 days in the
future. More than one date needs a fixed project duration.

**Destinations cannot be connected from here.** The customer connects and reconnects them in
the LiveReacting Studio. An expired YouTube access token is renewed automatically while
`livereacting_validate_destinations` runs, when the account still holds a refresh token.
Everything else has to be fixed in the Studio.

**Privacy settings and advanced YouTube options** can only be set in the Studio.

### Numeric maxima

| Thing | Maximum |
|---|---|
| Scenes per project | 300 |
| Layers per project | 800 |
| Files per playlist | 600 |
| Playlist files per project | 7000, summed across every video and audio playlist |
| Scheduled dates per call | 50 |
| Destinations per project | 50 |
| Project name | 60 characters |
| Scene name | 60 characters |
| Layer name | 19 characters |
| Broadcast title | 100 characters |
| Broadcast description | 5000 characters |
| Playlist item name | 100 characters |
| Planned duration | 31536000 seconds (365 days), minimum 30 |
| Ad break duration | 5 to 120 seconds |
| Volume | 0 to 10 |
| Files per `livereacting_encode_media` call | 100 |
| Analytics date range | 28 days, for `playlists` and only when both dates are sent |

Plan tiers add their own: 1080p formats need a 1080p plan or better, and 4K formats need a
4K plan. A file above the plan's resolution, bitrate or frame-rate ceiling has to be encoded
before it will play.

---

## 8. Errors

A failed tool call comes back as a tool result with `isError: true` and one sentence of
plain text. It is never an opaque code on its own. Where the REST message names a REST path,
the server rewrites it into the tool name the model actually has, for example
`DELETE /projects/:id/schedules` becomes "the livereacting_cancel_schedules tool". Other
API messages that name something no tool has are rewritten the same way: "Use stop endpoint"
names `livereacting_stop_stream`; "Disable nextScene first" tells the model to send
`autoSwitchToSceneId: null` to `livereacting_set_playlist`; and the refusals for `loop` together
with a scene id, and for an unknown scene id, are phrased in terms of `autoSwitchToSceneId`.

Any field a controller attaches beyond `error`, `code` and `message` is relayed to the model
as `Details: {...}`, because some of those fields are the data the model needs to recover.

### Codes with extra guidance

Fifteen codes carry a sentence the REST API cannot give, because it is about how an agent
should behave rather than about what went wrong. It follows the API's message as a new
sentence. `FILE_NOT_USABLE` and `FILES_NOT_USABLE` share one sentence, so the table below has
fourteen rows. `FEATURE_NOT_AVAILABLE` has none on purpose: each analytics refusal already names
its own remedy.

| Code | The guidance added |
|---|---|
| `PROJECT_ALREADY_LIVE` | Call `livereacting_get_stream_status` to read the current state. Do not retry `livereacting_start_stream` |
| `ACTIVE_SCHEDULES_EXIST` | Cancel first, then schedule. The ordering is required |
| `API_ACCESS_REQUIRES_PAID_PLAN` | Tell the customer to upgrade. No tool will work until then |
| `YOUTUBE_ANALYTICS_CONSENT_REQUIRED` | The customer must grant access in the Studio. No retry will succeed until they have |
| `ACCOUNT_SELECTION_REQUIRED` | The ids to choose between are in `availableAccounts`. When they are missing, call `livereacting_list_destinations` |
| `FILE_NOT_USABLE` and `FILES_NOT_USABLE` | A next step for each issue: wait for the import, import again, poll the encoder, or call `livereacting_encode_media` first |
| `LIVE_PROJECT_RESTRICTION` | Lists all five blocked fields and says to stop or unschedule first |
| `TRANSFER_IN_PROGRESS` | One import at a time: poll `livereacting_get_import_status`; a stuck import can be cancelled in the Studio |
| `STORAGE_LIMIT_EXCEEDED` | No tool deletes files: free space in the Studio, or upgrade |
| `URL_NOT_ALLOWED` | Ask the customer for a public link, or upload the file with `livereacting_upload_file` |
| `URL_NOT_RESOLVED` | The host does not exist: check the link with the customer; do not retry it unchanged |
| `UPLOAD_PROCESSING` | Wait 30 seconds and complete again with the same token; do not upload again |
| `UPLOAD_INCOMPLETE` | Send the named parts again, each exactly its size, then complete again |
| `UPLOAD_NOT_FOUND` | Check `livereacting_list_media` before starting a new upload |

### Every code a tool can surface

The HTTP column is the status the REST controller uses. It is not what an MCP client sees: a
failed tool call is an HTTP 200 carrying a tool result with `isError: true` and one sentence of
text. The code itself is not a field of that result, so read the sentence. The column is here
because it tells you how serious the failure is and whether a retry can help.

| Code | HTTP | What it means | What to do |
|---|---|---|---|
| `INVALID_API_KEY` | 401 | Missing or invalid key | Check the `Authorization` header |
| `API_ACCESS_REQUIRES_PAID_PLAN` | 403 | The account has no API access: a Free plan without an override, or a paused subscription | Upgrade at the URL in the message, or resume the subscription |
| `VALIDATION_ERROR` | 400 | An argument is wrong, missing or out of range | Read the message. It names the field |
| `PROJECT_NOT_FOUND` | 404 | No such project on this account | Check the id with `livereacting_list_projects` |
| `SCENE_NOT_FOUND` | 404 | No such scene in this project | Check with `livereacting_get_scenes` |
| `LAYER_NOT_FOUND` | 404 | No such layer | Check with `livereacting_get_scenes` |
| `FILE_NOT_FOUND` | 404 | No such file on this account. From the playlist tools, `details.missingFileIds` (the first 20) and `details.missingFileCount` name the ids that were not found | Remove those ids, or check with `livereacting_list_media` |
| `FOLDER_NOT_FOUND` | 404 | No such folder | Omit `folderId` or check it |
| `LIVE_NOT_FOUND` | 404 | No such broadcast | Check with `livereacting_list_broadcasts` |
| `FORMAT_NOT_AVAILABLE` | 403 | The format needs a higher plan | Use a lower resolution or tell the customer to upgrade |
| `LIVE_PROJECT_RESTRICTION` | 400 | One of five fields cannot change while live or scheduled | Stop the stream or cancel the schedule first |
| `DESTINATION_NOT_FOUND` | 400 | A destination id is not on the account | Re-read `livereacting_list_destinations` |
| `DESTINATION_TOKEN_INVALID` | 400 | A destination's login has expired | The customer reconnects it in the Studio |
| `NO_DESTINATIONS` | 400 | The project has no destinations | Set `destinationIds` with `livereacting_update_project` |
| `PROJECT_ALREADY_LIVE` | 400 | The project is already live or scheduled | Surfaced as a normal result by `livereacting_start_stream`, not as an error |
| `NO_ACTIVE_STREAM` | 404 | Nothing is running to stop | Read the status first |
| `NO_ACTIVE_LIVE` | 400 | Nothing is running to sync | Read the status first |
| `ACTIVE_SCHEDULES_EXIST` | 400 | The project already has schedules | Cancel them, then schedule |
| `MANUAL_DURATION_MULTI_SCHEDULE` | 400 | More than one date needs a fixed duration | Set a duration or schedule one date at a time |
| `SCHEDULING_CONFLICT` | 400 | Two scheduled runs are closer together than 3 minutes, or a run overlaps one that already exists | Move the later date to the time the message names |
| `PLUGIN_MODE_NOT_SUPPORTED` | 400 | Plugin-mode projects cannot be started or scheduled through the API | Use the web interface |
| `LIMIT_REACHED` | 403 | The account is at its concurrent-stream limit or at its combined live-and-scheduled queue limit | Read the message. The customer stops a stream or asks support to raise the limit |
| `PLATFORM_ERROR` | 502 | The destination platform or YouTube Analytics failed | Try again later |
| `SCENE_LIMIT_REACHED` | 400 | 300 scenes per project | Delete a scene |
| `LAST_SCENE_DELETE` | 400 | The last scene cannot be deleted | Create another scene first |
| `LAYER_LIMIT_REACHED` | 400 | 800 layers per project | Delete a layer in the Studio |
| `PROJECT_FILE_LIMIT_REACHED` | 400 | 7000 playlist files per project | Shorten a playlist |
| `FILE_NOT_USABLE` | 400 | One file cannot be used yet | Read the listed issues. Usually encode it |
| `FILES_NOT_USABLE` | 400 | Several files cannot be used yet | The same, per file |
| `INVALID_FILE_TYPE` | 400 | The file is not a video | Use a video file |
| `FILE_REQUIRED` | 400 | A `video` layer needs `settings.fileId` | Add it |
| `LOOP_NEXT_SCENE_CONFLICT` | 400 | `loop` cannot be on while the layer auto-switches | Send `autoSwitchToSceneId: null` with `loop: true`, or leave `loop` out |
| `INVALID_NEXT_SCENE` | 400 | `autoSwitchToSceneId` names a scene the project does not have | Take the id from `livereacting_get_scenes` |
| `NO_PLAYLIST_LAYER` | 400 | The previous scene holds no video playlist | Add one, or pass a different `previousSceneId` |
| `ADPOINT_OVERLAP` | 400 | Two ad breaks overlap, or one starts past the end of the file or runs past it | Read the message and fix the timestamps |
| `CONFLICT` | 409 | The previous layer changed during the call. Nothing was created | Read the scene and try again |
| `TRANSFER_IN_PROGRESS` | 409 | Another import is already running | Poll it, then import again |
| `STORAGE_LIMIT_EXCEEDED` | 400 | The account is out of storage | The customer frees space or upgrades |
| `IMPORT_FAILED` | 400 or 500 | 400: the direct link is not a usable media file (the reason is in the message). 500: the transfer service failed on a Drive, Dropbox or YouTube link | On 400, fix the link. On 500, try again |
| `URL_NOT_ALLOWED` | 400 | The link points to a private or internal address, or carries a user name or password | Ask for a public link, or upload the file |
| `URL_NOT_RESOLVED` | 400 | The host does not exist | Check the link. Do not retry it unchanged |
| `UNSUPPORTED_FILE_TYPE` | 400 | The extension does not match `type` | Fix the name or the type |
| `FILE_TOO_LARGE` | 400 | The file is larger than the plan's per-file maximum | Upgrade, or use a smaller file |
| `UPLOAD_NOT_SUPPORTED` | 400 | The account stores files outside the default storage | Contact support |
| `UPLOAD_INCOMPLETE` | 400 | Parts are missing or the bytes do not add up | Send the named parts again, then complete again |
| `UPLOAD_INVALID` | 400 | The file is not readable media, and the object was deleted | Upload the file again |
| `UPLOAD_PROCESSING` | 409 | The first completion is still checking the file | Wait 30 s and complete again |
| `UPLOAD_NOT_FOUND` | 404 | The upload expired, was discarded, or belongs to another account | Check `livereacting_list_media`, then start a new upload |
| `FEATURE_NOT_AVAILABLE` | 403 | Playlist analytics is not enabled for this account, or YouTube analytics has no active paid plan | Ask hello@livereacting.com for playlist analytics; upgrade the plan for YouTube analytics |
| `YOUTUBE_ANALYTICS_CONSENT_REQUIRED` | 403 | The channel has not granted analytics access | The customer grants it in the Studio |
| `ACCOUNT_SELECTION_REQUIRED` | 400 | The broadcast went to more than one YouTube channel | Pass `accountId` from `availableAccounts` |
| `INTERNAL_ERROR` | 500 | Something failed on our side | Try again in a moment. Then contact support |

### Errors that are not tool results

Five failures happen before a tool runs, so they arrive as JSON-RPC errors rather than as
tool results.

| Situation | HTTP | JSON-RPC code |
|---|---|---|
| Unknown `toolsets` or `read_only` value | 400 | `-32602` |
| More than 20 messages in a batch | 400 | `-32600` |
| Pre-authentication flood limit | 429 | `-32000` |
| `GET` or `DELETE` on the endpoint | 405 | `-32000` |
| An unhandled server failure | 500 | `-32603` |

Authentication happens even earlier, in the API-key middleware, so it answers with an ordinary
HTTP JSON body and no JSON-RPC envelope at all: `401 INVALID_API_KEY` for a missing or invalid
key, and `403 API_ACCESS_REQUIRES_PAID_PLAN` for an account without API access. A client that
only parses JSON-RPC sees a transport failure there rather than a message.

An unexpected failure inside a tool does reach the model as a tool error. It says the tool
failed on our side, that nothing was changed by the call, and to contact
[hello@livereacting.com](mailto:hello@livereacting.com) if it keeps happening. Internal
detail is written to our logs and never to the model's context.

---

## 9. Versioning

The server reports its own version in `serverInfo`, and the registry manifest publishes the
same number.

| Item | Value |
|---|---|
| Server name | `livereacting` |
| Server version | `1.2.0` |
| Registry name | `com.livereacting/livereacting` |
| Manifest | `server.json`, validated against the `2025-12-11` registry schema |

The version is the server's own. It is deliberately not the version of the API package it
lives inside. It is the number a customer quotes in a support ticket.

**Bump it when the tool list or a tool's contract changes**: a tool added or removed, an
argument added, removed or renamed, a response field removed or renamed, or an annotation
changed. A new optional argument or a new response field is still a change worth recording.
Fixing wording in a description is not.

The protocol revision is a separate thing and is negotiated per connection. See
[section 1](#1-what-this-is).
