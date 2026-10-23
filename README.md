# ⚡ HRAMY Omni V1.3

A Windows-first distributed AI compute gateway with a live server console, compute-node console, real hardware telemetry, model discovery, outbound WebSocket registration, automatic reconnect, and an OpenAI-compatible API.

Turn old PCs into one AI pool: each machine runs the **HRAMY Node**, connects out to the **HRAMY Server**, and your normal OpenAI code runs against every model in the fleet.

## What is real

The dashboard does not use placeholder node rows. CPU, RAM, GPU, runtime state, models, node registration, request counters, and latency are read from the running processes. If no nodes are connected, the UI says so. If a model is missing, the API says so. Nothing is faked.

## Quick Windows setup

Use Python 3.11. This project intentionally uses the Windows `py` launcher in PowerShell.

```powershell
py -3.11 -m venv .venv311
.\.venv311\Scripts\Activate.ps1
py -3.11 -m pip install -r requirements.txt
```

Run tests:

```powershell
py -3.11 -m pytest -q
```

End-to-end verification (starts a real gateway, a real node process, and a stand-in OpenAI-compatible runtime; useful without a GPU):

```powershell
py -3.11 scripts\verify_e2e.py
```

Build:

```powershell
.\scripts\build_windows.ps1
```

## Runtime

Ollama is supported by default at `http://127.0.0.1:11434/v1`. The node discovers its models and can attempt to start `ollama serve` automatically when enabled.

The Node console's **Model library** tab lists locally installed Ollama models and can download a model by its Ollama name or tag. Model downloads use Ollama's local `/api/pull` endpoint; other OpenAI-compatible runtimes can serve inference but may not support downloads.

## Downloading models

Models can be downloaded on any machine in the fleet, and you choose the folder on that machine's disk. Ollama always pulls into its own model store, so HRAMY pulls the model and then copies the manifest and blobs into the folder you picked, preserving Ollama's `manifests/` + `blobs/` layout. Restore an archived model by copying that folder back into an Ollama models directory (`%USERPROFILE%\.ollama\models` or `$OLLAMA_MODELS`).

### Flutter control plane

The operator UI is [flutter_ui](flutter_ui) — a Flutter desktop app that is a pure client over the same gateway API the Python console used, so the backend is unchanged. It has Overview, Models, Downloads, Chat, Skills and Developer API pages. It reads the gateway API key from the config directory (`%USERPROFILE%\.hramy\api_key`) on startup, and the status chip at the bottom of the nav rail opens a connection dialog to change the base URL or key.

```powershell
cd flutter_ui
flutter pub get
flutter run -d windows     # run it
flutter build windows      # only when you want the .exe
```

The Tkinter consoles remain in the tree as the fallback: if the Flutter bundle is missing — for example a source checkout where you have not run `flutter build windows` — the EXE opens the old console and tells you why instead of failing. `desktop/server.py` and `desktop/node.py` are that console, and `flutter_ui/` is the UI the EXEs open.

**Both desktop EXEs open this UI.** The Python EXE is the backend and the launcher. `HRAMY Omni Server.exe` starts the gateway and then opens the control plane against it; `HRAMY Omni Node.exe` registers this machine as a compute node and then opens the control plane against the gateway saved in `%USERPROFILE%\.hramy\node.json`. The launcher passes that gateway with `--base-url`, so the UI starts already connected. The API key comes from the config directory, which the Server EXE writes for itself — a node machine that has never run the server has no master key, so open the connection dialog in the nav rail and paste one there.

`scripts\build_windows.ps1` builds the control plane for you and copies its release output into each app bundle as `ui\`, next to the EXE. That is where `desktop\ui_launcher.py` looks for it when the app is frozen.

The Server console has a **Downloads** page:

- **DOWNLOAD TO** — pick a connected compute node, or *This server machine* to download on the server itself with no node involved
- **MODEL NAME** — any Ollama name or tag, such as `llama3.2:3b`
- **WHERE TO DOWNLOAD** — the destination folder, with **BROWSE…** to pick it on disk (the folder is on the machine you download to)
- A live progress bar plus a **Download jobs** table showing the target, status, progress and the folder each model actually landed in

The Node console's workspace is tabbed, with a **DOWNLOAD MODEL** tab that pulls a model and archives it into a chosen folder on that node's disk — useful when the node should keep a copy on a specific drive.

Download APIs, all master-key authenticated: `POST /v1/model-downloads/local` (server machine), `POST /v1/nodes/{node_id}/model-downloads/local` (a chosen node), `GET /v1/model-downloads/local` and `GET /v1/model-downloads` (job lists). Each POST takes `{"model": "…", "archive_dir": "…"}` and returns a job id.

## API keys — no dev defaults

On first run the server generates a cryptographically secure `hramy_...` API key and node token and persists them in the config directory (`%USERPROFILE%\.hramy` by default, override with `HRAMY_CONFIG_DIR`). The old `hramy_dev_key_change_me` default is gone: leaving the env vars unset now *generates* a secret instead of shipping a known one. An explicit secure `HRAMY_API_KEY` / `HRAMY_NODE_TOKEN` env value still wins; placeholder values are ignored in favor of the persisted secret.

Rotate the API key any time from **Server GUI → Developer API → ROTATE** (or `POST /v1/developer/rotate-api-key`). Rotation invalidates the old key immediately; the new key is shown once.

**Three different credentials, do not mix them:**

| Credential | Who uses it | Where it lives |
|---|---|---|
| MASTER API Key | Server admin: managing nodes, keys and configuration | Developer API page (ROTATE) |
| Model Session Key | One app, one model/session — scoped to that model on that node | Developer API page → Model sessions |
| Node Token | Compute nodes pairing with the server | PAIR NODE dialog |

## Model session keys

Each AI model session gets its **own** API key so one application can be granted access to exactly one model on one node:

- Keys are `hramy_<32 random chars>` from `secrets.token_urlsafe` — cryptographically random, never counters, never derived from the model name. Two sessions for the same model still get different keys.
- A session key only works for its own model (`403` for any other model) and routes to its pinned node — it never silently hops to another node hosting the same model.
- Create sessions from **Server GUI → Developer API → Model sessions → CREATE SESSION**, or `POST /v1/sessions`. `GET /v1/sessions` lists them; `POST /v1/sessions/{id}/revoke` invalidates one key instantly; `POST /v1/sessions/{id}/test` runs a real inference through that key.
- Sessions persist in the config directory (`sessions.json`, same 0600 protection as the master key). If the node disconnects the key stays valid but requests report the node offline; when the node reconnects the same key works again.
- Session keys are for the OpenAI-compatible API only — they cannot call admin endpoints (`/v1/sessions`, `/v1/nodes`, `/v1/stats`, …), which still require the master key. The master key keeps full access exactly as before.
- Give each session an API name when you create it. Revoked model-session keys and rotated master keys appear under **Revoked Keys** with their purpose, creation/revocation dates, and recorded model usage. Revoked secrets are not shown.

## Developer API page

The server GUI has a **Developer API** page with everything a client app needs:

- API status, server and port
- The MASTER API key (masked) with **COPY** and **ROTATE**
- **Model sessions** table: one unique `hramy_…` key per model session with **COPY KEY**, **COPY PYTHON**, **TEST API** and **REVOKE**, plus **CREATE SESSION**
- Base URL for local (`http://127.0.0.1:8000/v1`) and LAN (`http://<your-ip>:8000/v1`), each with **COPY**
- Every routable model with the nodes that own it, double-click to copy a model ID
- **TEST API CONNECTION** — performs a real request: reachability, auth, model availability, and a real inference round-trip. It shows ✓/✗ per check and never fakes success.
- Copy-paste **Python / JavaScript / cURL** examples generated live with your real base URL and model IDs
- **API key tester** checks authentication and model access, then sends a real inference request
- **Revoked Keys** keeps an audit record for keys you revoke or rotate

## Chat and skills

The Server console and browser dashboard include a **Chat** page. Select a model currently advertised by a connected node, optionally select an enabled built-in prompt skill, and send messages with streamed responses. Conversation history remains in the current UI session.

**Skills Settings** controls the built-in Coding, Writing, Research, and Summarization prompt skills. A disabled skill is removed from chat choices and the gateway rejects requests that try to use it. These skills add instructions to a model request; they do not install tools or execute code.

## Approving a node (pairing code)

A node token alone does not let a server run anything on a node. Every time a node connects it shows a fresh 8-character code on **its own screen** (Node window → PAIRING CODE, or `%USERPROFILE%\.hramy\node.log` when it runs headless).

1. Open the Server app → **OVERVIEW**, find the row marked **LOCKED**, click **ENTER CODE** (in the Tkinter console: double-click the locked row).
2. Type the code exactly as shown on the node (case and dashes are forgiving: `m73j d01x` works).
3. The node verifies it. Until then it refuses chat, model listing, model deletion and downloads, and the gateway does not advertise its models.

Three wrong codes lock that server out for **1 minute**, and the code is single-use — reconnect shows a new one. Replacing a machine's token is done from the **NODE TOKENS** tab, which lists every token the gateway issued, marks the active one, and shows which nodes use it.

## Latency

The dashboards show **network latency** — a real gateway ↔ node ping round trip — separately from **inference time**. Nodes above `HRAMY_MAX_LATENCY_MS` (default 3000) are demoted in routing. Heartbeats no longer run `nvidia-smi` on the event loop and inference streams reuse one HTTP connection, so token streaming is no longer stalled by telemetry or a fresh TCP handshake per request.

## Using the API

```python
from openai import OpenAI

client = OpenAI(
    api_key="HRAMY_API_KEY",            # the hramy_... key from the Developer API page
    base_url="http://SERVER_IP:8000/v1" # LAN or local base URL
)

response = client.chat.completions.create(
    model="MODEL_ID",                    # any model shown on the Developer API page
    messages=[{"role": "user", "content": "Hello from HRAMY!"}]
)
print(response.choices[0].message.content)
```

cURL:

```bash
curl http://SERVER_IP:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer HRAMY_API_KEY" \
  -d '{"model": "MODEL_ID", "messages": [{"role": "user", "content": "Hello"}]}'
```

Streaming is OpenAI-compatible SSE: `data: {...}` frames terminated by `data: [DONE]`. The OpenAI SDK, cURL, and raw HTTP clients all work against `/v1/chat/completions` and `/v1/models`.

## Multi-node

- Every node has a stable identity (`node-<hostname>-<mac>` by default, or `HRAMY_NODE_ID`).
- Nodes register over outbound WebSocket (`/ws/node-v1`), so nodes never need an inbound port.
- The gateway keeps one record per node: connection, telemetry, models, jobs, latency, errors. Node B connecting never touches Node A; Node A dropping never touches Node B.
- Reconnects reuse the node identity and refresh state — no duplicate ghost nodes.
- Models belong to nodes. The same model on several nodes is valid; the router picks the healthiest owner (load, CPU, free RAM, latency) and **fails over to another owner** if the chosen node dies mid-request. It never claims failover to a node that lacks the model.
- Errors are specific: `No healthy compute node supports model 'x'`, `Invalid HRAMY API key`, `Node authentication failed`, and so on.
- An unapproved node stays connected but is excluded from `/v1/models` and routing until its pairing code is entered on the server.

## Windows output

`dist\HRAMY Omni Server\HRAMY Omni Server.exe`

`dist\HRAMY Omni Node\HRAMY Omni Node.exe`

These are PyInstaller **onedir** apps. Keep every file in each output folder together.

Each app folder also contains `ui\` — the Flutter control plane the EXE opens. The Python EXE owns the backend (gateway or compute node) and launches `ui\hramy_omni_ui.exe` as its window, so deleting or moving `ui\` sends the app back to the built-in Tkinter console.

The browser dashboard is served by the gateway at `http://127.0.0.1:8000/ui`; Docker Compose uses the same gateway-hosted page.
