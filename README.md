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

## acme tag setup

Add to your acme tag template or window column 1:

```
Analyze '{acme-ai analyze $winid}' Explain '{acme-ai explain $winid}' Fix '{acme-ai fix $winid}' Chat '{acme-ai chat $winid}'
```

**Usage:**

1. Select code in the body
2. Middle-click **Analyze**, **Explain**, or **Fix**
3. Response is inserted at the cursor

### Chat window

Middle-click **Chat** on a source window (optionally with code selected for context). A new acme window opens. Add this to the chat window tag:

```
Send '{acme-ai chat-send $winid}'
```

Type or select your message, middle-click **Send**. Conversation history is stored in `$home/lib/acme-ai/chat-<source-winid>.json`.

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
