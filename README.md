# LiveReacting MCP server

Run live video streams from an AI agent. Build a channel from a video playlist, start it,
schedule it, watch it while it runs, and read the numbers afterwards — from Claude Code,
Codex, Cursor, or anything else that speaks MCP.

```
https://mcp.livereacting.com/mcp
```

Streamable HTTP. Stateless. Authenticated with your LiveReacting API key.

Full reference, with every tool, input, response field, limit and error code:
[developers.livereacting.com/mcp](https://developers.livereacting.com/mcp)
([REFERENCE.md](REFERENCE.md) in this repo).

---

## Get a key

API access is included with **every paid plan**. Create a key in the
[Developer section](https://studio.livereacting.com/developer) of the LiveReacting Studio.
Keys start with `lr_`.

An account on the Free plan gets `403 API_ACCESS_REQUIRES_PAID_PLAN` on every call, and the
error says so, so your agent can tell you what to do about it.

---

## Setup

Create the key first, put it in your shell profile as `LIVEREACTING_API_KEY`, then paste the
config below.

Every example that cannot read that variable uses `lr_…` as a placeholder. **Do not paste a
real key into a file you commit.** Where a harness supports an environment variable or a
prompt, the block uses it.

### Claude Code

In `.mcp.json`. Claude Code expands `${VAR}` in a remote server's `headers`
([docs](https://code.claude.com/docs/en/mcp)), so the key stays out of the file:

```json
{
  "mcpServers": {
    "livereacting": {
      "type": "http",
      "url": "https://mcp.livereacting.com/mcp",
      "headers": { "Authorization": "Bearer ${LIVEREACTING_API_KEY}" }
    }
  }
}
```

`claude mcp add --transport http livereacting https://mcp.livereacting.com/mcp --header
"Authorization: Bearer lr_…"` also works, but it writes the key itself into the stored config.

### Codex CLI

In `~/.codex/config.toml`:

```toml
[mcp_servers.livereacting]
url = "https://mcp.livereacting.com/mcp"
bearer_token_env_var = "LIVEREACTING_API_KEY"
```

Then `export LIVEREACTING_API_KEY=lr_…` in your shell profile. Prefer this over
`http_headers`, which puts the key in the config file.

### Cursor

In `~/.cursor/mcp.json`. Cursor resolves `${env:NAME}` in `headers`
([docs](https://cursor.com/docs/context/mcp)):

```json
{
  "mcpServers": {
    "livereacting": {
      "url": "https://mcp.livereacting.com/mcp",
      "headers": { "Authorization": "Bearer ${env:LIVEREACTING_API_KEY}" }
    }
  }
}
```

### VS Code (GitHub Copilot)

In `.vscode/mcp.json`. The `inputs` block makes VS Code prompt for the key instead of
storing it in the repo:

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

In `~/.gemini/settings.json`. Gemini CLI expands `$VAR` and `${VAR}` in every string of
the file before it validates it, headers included
([docs](https://geminicli.com/docs/reference/configuration/)), so the key stays out of it:

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

In your Zed `settings.json`, under `context_servers`, add an HTTP server pointing at the
URL with an `Authorization` header.

### Goose

Add a **Remote Extension** (Streamable HTTP) in Goose's settings, URL
`https://mcp.livereacting.com/mcp`, with a custom header
`Authorization: Bearer lr_…`.

### Compatibility

| Harness | How the key is passed |
|---|---|
| Claude Code | `${VAR}` in `.mcp.json` headers |
| Codex CLI | `bearer_token_env_var` |
| Cursor | `${env:VAR}` in `mcp.json` headers |
| VS Code Copilot | `headers` with a prompted input |
| GitHub Copilot coding agent | static header secret only. This product does not support OAuth remote servers, so the key path is the only path here, now and later |
| Gemini CLI | `$VAR` in `settings.json` headers |
| Zed | `headers` in `settings.json` |
| Goose | remote extension header |
| ChatGPT | connectors take OAuth or nothing, never a pasted key |

**Protocol revisions:** `2024-10-07` through `2025-11-25`. If your client reports a version
error, that is the first thing to check. `2026-07-28` is not supported yet — no published
TypeScript SDK speaks it.

---

## Tools

30 tools in five toolsets. Every name is prefixed `livereacting_`, so one search in a
client that defers tool loading finds the whole server.

### account

| Tool | What it does |
|---|---|
| `list_destinations` | Your connected YouTube, Facebook, Twitch, LinkedIn, X and custom RTMP destinations |
| `validate_destinations` | Checks a destination's connection still works, before you rely on it |

### editor

| Tool | What it does |
|---|---|
| `list_projects` | Your projects |
| `get_project` | One project's configuration |
| `create_project` | A new project |
| `update_project` | Change a project's settings |
| `get_scenes` | Scenes in a project, or one scene |
| `manage_scene` | Create, rename or delete a scene |
| `activate_scene` | Switch the active scene, including during a broadcast |
| `create_scene_with_playlist` | A scene plus its video playlist in one call |
| `add_layer` | Add one video playlist, audio playlist or single video layer to an existing scene |
| `get_layer` | One layer, with its playlist truncated for context |
| `set_playlist` | Replace a playlist's contents |

### streams

| Tool | What it does |
|---|---|
| `start_stream` | Go live |
| `stop_stream` | End the broadcast |
| `get_stream_status` | Live state, per destination, including expired connections |
| `schedule_stream` | Go live at set times |
| `cancel_schedules` | Cancel scheduled broadcasts |
| `sync_stream` | Push editor changes to a running stream |
| `list_broadcasts` | Past broadcasts for a project |

### media

| Tool | What it does |
|---|---|
| `list_media` | Files and folders in your library |
| `get_media_file` | One file's details |
| `upload_file` | Start uploading a local file: returns part URLs and curl commands |
| `complete_upload` | Finish an upload once every part is sent |
| `import_media` | Import from Google Drive, Dropbox or YouTube |
| `import_from_url` | Import from a direct link to a media file on any public host |
| `get_import_status` | Poll an import, including every video of a playlist import |
| `encode_media` | Re-encode a file for streaming |
| `get_encode_status` | Poll an encode |

### analytics

| Tool | What it does |
|---|---|
| `get_stream_analytics` | YouTube analytics, or per-video playlist analytics (that type on request) |

### Selecting fewer tools

Both are query parameters on the URL, so no client support is needed.

```
https://mcp.livereacting.com/mcp?toolsets=streams,media
https://mcp.livereacting.com/mcp?read_only=1
```

`read_only=1` drops every tool that can change anything. Useful for a monitoring agent you
do not want touching a live broadcast.

---

## Limits, stated honestly

An agent that promises any of these is lying to your customer.

- **You can build video and audio playlists, and nothing else.** Text, image and interactive
  layers are readable through the API but can only be created or edited in the
  [LiveReacting Studio](https://studio.livereacting.com).
- **Operating is not limited that way.** A stream whose overlays were built in the Studio can
  still be started, stopped, scheduled, synced and monitored from here. The line is *author*
  versus *operate*.
- **Uploading a local file needs a shell or HTTP client.** `upload_file` returns signed part URLs
  and ready curl commands; the agent sends the bytes itself, then calls `complete_upload`.
- **Every plan has a maximum stream length.** A stream stops when it reaches it, even with a
  manual duration. A 24/7 channel needs a plan without a length limit, such as the 24/7 plan.
- **A Facebook-only 24/7 channel is not possible.** Facebook-only streams get a fixed 8-hour
  maximum.
- **Thumbnails and live announcements are unsupported.**
- **Playlist analytics is available on request.** YouTube analytics is included with every paid
  plan, and needs the channel to have granted analytics access. Ask hello@livereacting.com for
  playlist analytics.

---

## Rate limits

**150 requests per minute per account**, shared across your API keys.

REST and MCP are counted separately, so the two surfaces get 150 each rather than sharing one budget. Do not plan around that: it is an implementation detail of how the two are metered, not a promise.

A 429 comes back as a tool error that says the number and how long to wait, because the model
is the one reading it. The polling tools — `get_stream_status`, `get_import_status`,
`get_encode_status` — each document a minimum interval. Poll tighter than that and you spend
your own budget on nothing.

---

## Things worth knowing before your agent surprises you

**`set_playlist` replaces the whole playlist.** It is not an append. To add one video, read the
layer first, then send the complete list back. The tool tells you when a write shrinks the
playlist, but it will still do what you asked.

**Starting and stopping a stream is not idempotent.** If a call times out, ask
`get_stream_status` what happened. Do not retry — a retry can start a second attempt.

---

## Docs

Full MCP reference: [developers.livereacting.com/mcp](https://developers.livereacting.com/mcp)
Full REST API reference: [developers.livereacting.com](https://developers.livereacting.com)
Support: [hello@livereacting.com](mailto:hello@livereacting.com)

