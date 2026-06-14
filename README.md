# acme-ai

Lux + Ollama integration for [9front](http://9front.org/) acme. Select code in acme, middle-click **Analyze**, **Explain**, or **Fix** — Lux calls Ollama on your Mac Mini and inserts the response.

Inference runs on the Mac (48GB M4). 9front only runs the thin Lux client and rc glue.

## Architecture

```
acme tag → bin/acme-ai (rc) → lib/dispatch.lux (Lux) → Ollama on Mac Mini :11434
                ↑ reads rdsel/addr              ↑ httpPost + parseJSON
                ↓ insert result               ↓ stdout
```

## Requirements

- **Lux** on 9front (`$home/bin/$objtype/lux` from [lux](https://github.com/yourusername/lux))
- **Ollama** on Mac Mini, listening on the network
- **9front** network entry for the Mac in `/lib/ndb`
- **acme** with `$winid` in tag commands (standard)

## Install (9front)

```rc
cd /path/to/acme-ai
rc install.rc
```

This copies:

- `lib/*` → `$home/lib/acme-ai/`
- `bin/acme-ai` → `$home/bin/acme-ai`

Edit `$home/lib/acme-ai/config.json`:

```json
{
  "host": "macmini",
  "port": 11434,
  "models": {
    "analyze": "deepseek-r1:32b",
    "explain": "qwen2.5:14b",
    "fix": "qwen2.5-coder:32b",
    "chat": "qwen2.5-coder:32b"
  }
}
```

`host` must resolve via `/lib/ndb` (e.g. `ip=192.168.1.10 sys=macmini`).

## Mac Mini Ollama setup

Ollama binds to localhost by default. Expose it on the LAN:

```bash
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

Pull models you reference in `config.json`:

```bash
ollama pull deepseek-r1:32b
ollama pull qwen2.5:14b
ollama pull qwen2.5-coder:32b
```

Verify from 9front:

```rc
hget http://macmini:11434/api/tags
```

## acme tag setup (same pattern as t+ / t-)

Put commands in `/acme/bin/` (like your `t+` and `t-` scripts). After `rc install.rc`:

```
Del Snarf | Look | t+ | t- | aixplain | aianalyze | aifix
```

**How to use (same as t+):**

1. Select the code in the **body**
2. Highlight **`|aixplain`** in the tag (column 1) — same as **`|t+`**
3. Middle-click

The script reads your selection from **stdin** and prints the answer to **stdout**. acme replaces the selection with the output — exactly like `t+` runs `sed`. No `$winid`, no `{` braces, no quotes in the tag.

**Do not** middle-click only the code in the body — that runs C/rc code as a command. Always highlight the **tag command** (`|aixplain`) while the body text is selected.

### Chat window

Middle-click **`|aichat`** (with optional code selected) to open a chat window. In the chat window tag add **`|aichat-send`** and use the same highlight + middle-click pattern to send a message.

## Request files

The rc wrapper writes these under `$home/lib/acme-ai/` before invoking Lux:

| File | Contents |
|------|----------|
| `request.action` | `analyze`, `explain`, or `fix` |
| `request.addr` | File path from acme |
| `request.sel` | Selected text |

Lux reads them from the lib directory (rc `cd`s there first). Split files avoid JSON-escaping selected code in rc.

## Development (Mac)

Test Ollama connectivity with POSIX Lux (note: 30s curl timeout may fail on large models):

```bash
cd tests
/path/to/lux/posix/lux test_ollama.lux
```

Edit `lib/config.json` — set `"host": "127.0.0.1"` for local Ollama.

Test dispatch manually:

```bash
cd lib
echo -n analyze > request.action
echo -n /tmp/foo.go > request.addr
echo -n 'func main() {}' > request.sel
/path/to/lux dispatch.lux
```

## POSIX vs Plan 9 timeouts

- **Plan 9 Lux**: HTTP reads until connection close — fine for 32B models
- **POSIX Lux**: libcurl 30s timeout — use smaller models for Mac-side dev, or increase timeout in Lux upstream

## Layout

```
acme-ai/
  bin/acme-ai          rc entrypoint
  install.rc           install to $home
  lib/
    config.json        host, port, models
    ollama.lux         generic Ollama client (upstream Lux candidate)
    prompts.lux        prompt templates
    dispatch.lux       analyze / explain / fix
    chat.lux           chat turns with history
  tests/
    test_ollama.lux    connectivity test
```

## Future Lux contribution

`lib/ollama.lux` is intentionally acme-free. Copy to the Lux repo as `examples/ollama.lux` when ready.
