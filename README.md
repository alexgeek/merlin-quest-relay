# Merlin Relay

Merlin is an LLM-driven procedural 3D authoring tool. Merlin for Meta Quest (Quest 3 / 3S) builds scenes from node graphs on your desk. The Merlin Relay is the computer side of it: it runs on a PC or Mac on the same network as the headset, and chat prompts you send from the headset run as agent turns on the PC. The PC cooks and renders candidate graphs to check its work; the headset receives the final graph and cooks it itself.

This repo only hosts the downloads. It has no source code. The instructions below are for Windows; for macOS (Apple Silicon), download `Merlin-Relay-<version>-macos-arm64.dmg` and follow the release notes.

## Download

Get `Merlin-Relay-<version>-windows-x64.zip` from the [latest release](../../releases/latest).

It contains:

| File | What it is |
|---|---|
| `merlin_relay_app.exe` | The relay as a system tray app. Use this one. |
| `merlin_relay.exe` | The same relay without a UI, for a terminal (see below). |
| `merlin_render.exe` | Renders candidate scenes so the agent can see and score its work. |
| `merlin_cook.exe` | Cooks node graphs into meshes for the agent's checks. |
| `recipes/` | Example graphs the agent looks up while building. |
| `msvcp140*.dll`, `vcruntime140*.dll` | The Visual C++ runtime, so no separate install is needed. |

Keep all of these together in one folder. The relay looks for `merlin_render.exe` and `merlin_cook.exe` next to its own exe.

## Requirements

- Windows 10 or 11, x64.
- A GPU with current Vulkan (or DirectX 12) drivers, for rendering.
- An LLM provider. The default is **Claude Code**: install the [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code), log in once, and make sure `claude` is on your `PATH` (version 2.1.221 or newer; `claude update` updates it). The tray app's Provider list also offers:
  - `codex`: the Codex CLI (`codex` on `PATH`, logged in).
  - `ollama`: a local [Ollama](https://ollama.com) server on `localhost:11434`.
  - `openrouter`: needs an OpenRouter API key in the `OPENROUTER_API_KEY` environment variable.
- The PC and the headset on the same Wi-Fi / LAN.

## Setup

1. Unzip the release anywhere, for example `Documents\Merlin Relay`.
2. Run `merlin_relay_app.exe`. It lives in the system tray (check the `^` overflow area). Click the icon and choose **Open Merlin Relay…** (or run the exe again) to see its status window, which shows the address it listens on (port 7777).
3. When Windows Firewall asks, allow it on **Private networks only**.
4. Put the headset on the same network. In Merlin, press **Connect** and pick your PC.
5. Accept the pairing request on the PC (a notification, the tray menu, or the status window). The headset then says it is paired.
6. Type a prompt on the headset and press **Send**.

Pairing is remembered, including across IP changes. The relay keeps its key, paired devices and provider choice in `%USERPROFILE%\.merlin\`. To unpair everything, use Settings → "Forget all devices…" in the status window. Settings also has "Start Merlin Relay at login".

### Headless

`merlin_relay.exe` is the same relay in a terminal:

```
merlin_relay.exe --pair            # open a 120 s pairing window; asks y/N for each request
merlin_relay.exe --port 7777       # listen on another port (default 7777)
merlin_relay.exe --browse          # list relays advertised on the LAN
merlin_relay.exe --mock --pair     # testing only: loopback, canned replies, no LLM
```

Run only one relay (tray or headless) at a time; they share the same key.

## Choosing models

By default the provider picked in the tray app is used for everything. For finer control, create `%USERPROFILE%\.merlin\roles.toml`. When it exists it overrides the Provider picker. Example: Claude Code drives the turn and also builds and judges:

```toml
schema = 1

[driver]
kind = "claude-code"
claude_model = "sonnet"

[roles.builder]
provider = "claude"
model = "sonnet"

[roles.vision.judge]
provider = "claude"
model = "sonnet"
```

Or keep Claude Code as the driver and run the builder and judge on a local Ollama:

```toml
schema = 1

[driver]
kind = "claude-code"
claude_model = "sonnet"

[roles.builder]
provider = "ollama"
model = "qwen3.5:9b"
context_length = 32768

[roles.vision.judge]
provider = "ollama"
model = "qwen3-vl:8b"

[roles.vision.analyst]
provider = "ollama"
model = "qwen3-vl:8b"
```

`[roles.vision.analyst]` describes reference images you attach to a prompt. Provider values: `claude`, `codex` (builder only), `ollama`, `openrouter`, `anthropic`, `openai_compat` (needs `base_url`), `llamacpp`. API keys are never stored in the file; a role reads them from an environment variable (`api_key_env`, default `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY` or `OPENAI_API_KEY`). The `MERLIN_ROLES` environment variable can point at a different file.

## Troubleshooting

| Problem | Fix |
|---|---|
| Windows says "Windows protected your PC" | The exes are not code-signed. Click **More info → Run anyway**. |
| The headset doesn't list the PC | Check that Windows Firewall allows Merlin Relay on **Private** networks and that your Wi-Fi is set to a Private network (Settings → Network & internet → your network → Private). Both devices must be on the same subnet; guest Wi-Fi and client isolation block it. If discovery still fails, enter the PC's IP and port by hand (`192.168.x.y:7777`, shown in the status window). |
| The status window says "Not serving", or the tray icon never appears | Port 7777 is taken, or `merlin_relay.exe` is already running (the tray app then exits silently). Quit the other relay, or start the tray app with `--port <n>`. |
| Turns run but never render or score | `merlin_render.exe` / `merlin_cook.exe` are missing from the relay's folder. Keep the exes together, or set `MERLIN_RENDER_BIN` / `MERLIN_COOK_BIN` to their full paths. Rendering also needs a working Vulkan or DX12 GPU driver. |
| "is Claude Code installed?" / turns fail at once | `claude` is not on `PATH` for the relay, or not logged in. Run `claude --version` and `claude` once in a terminal, then restart the relay. |
| Agent ignores the recipes | Recipes are read from `recipes\` in the working directory. Starting the exe from Explorer works; when started some other way (e.g. at login), set `MERLIN_RECIPES_DIR` to the full path of the `recipes` folder. |

## Security

- **LAN only.** Never port-forward the relay or expose it through a tunnel. Anyone who reaches it while pairing is open can ask to pair, and one Accept lets them spend your LLM subscription. The relay also refuses non-private IPs, but that is a backstop, not a reason to expose it.
- When Windows Firewall asks, allow it on **Private** networks only.
- **Only accept pair requests you started.** Pairing is open for 120 s after you start it. A device's name is just a claim; check the IP shown on the request.
- After pairing, traffic is encrypted and authenticated (X25519 + XChaCha20-Poly1305). There is no TLS and no web UI.
- Revoke a device in the status window, or "Forget all devices…", if a headset is lost or you are unsure.
