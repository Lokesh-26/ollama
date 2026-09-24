# Remote Ollama + OpenCode

Use an Ollama model hosted on a Windows GPU server from an OpenCode client on a Linux or Windows PC. Model inference happens on the GPU server; OpenCode reads and edits files and runs approved tools on the client PC. No paid model API is needed.

> **Scope:** This README covers client setup only. It assumes the server already exposes an authenticated HTTPS endpoint backed by Ollama. Reverse-proxy, IIS, and server certificate *issuance* are not covered. All server addresses and credentials below are placeholders.
>
> **OpenCode version:** 2.0.16. The configuration below was verified against the installed binary's own config validator, not from documentation. The correct keys are `provider`, `npm`, `options` and `permission`. Do **not** use `providers`, `package`, `settings`, `modelID` or `capabilities`: those are OpenCode's *internal* post-parse shape, and a config written that way is accepted by the file reader, silently ignored by the provider loader, and the model never appears.

## Architecture

```text
Linux or Windows PC                          Existing Windows GPU server
┌─────────────────────────────┐              ┌─────────────────────────────┐
│ Local project files         │              │ HTTPS gateway (port 443)    │
│ OpenCode coding agent       │              │ Authentication + TLS        │
│          │ HTTP (localhost) │              │          │                  │
│          ▼                  │              │          ▼                  │
│ stunnel 127.0.0.1:11435     │─ HTTPS+auth ▶│ Ollama (internal HTTP API)  │
│ Approved local tools        │              │          │                  │
└─────────────────────────────┘              │          ▼                  │
                                             │ Qwen3.5-27B / RTX A4500     │
                                             └─────────────────────────────┘
```

- **Ollama** hosts models and performs inference on the server.
- **OpenCode** is the coding-agent interface on your PC; it operates on the project directory where it runs.
- The PC needs network access to the existing **HTTPS gateway**, not direct access to Ollama's internal port.
- The examples use HTTP Basic Authentication at the gateway and a separately trusted self-signed certificate. Adjust the authentication step if your gateway uses a different scheme.
- The URL in OpenCode ends in `/v1` (OpenAI-compatible API); `/api/tags` is the native model-list endpoint used for checking the server.
- **stunnel** terminates TLS locally. It is required whenever the gateway uses a self-signed or privately-issued certificate: OpenCode 2.0.16 ships as a self-contained Bun binary that **ignores `NODE_EXTRA_CA_CERTS`, `SSL_CERT_FILE` and the system trust store** (all three verified to fail with `UNABLE_TO_VERIFY_LEAF_SIGNATURE`). stunnel verifies the certificate properly and hands OpenCode a plain localhost HTTP endpoint. If your gateway ever gets a publicly-trusted certificate, you can point OpenCode straight at `https://SERVER_HOST_OR_IP/v1` and skip stunnel entirely.

## Prerequisites

1. OpenCode **V2** installed on the client: [official installation guide](https://opencode.ai/v2/docs/).
2. The existing server HTTPS URL, gateway username and password, and the server's **public** TLS certificate (`.cer`/`.crt`/`.pem`).
3. Verify the certificate's **SHA-256 fingerprint** through a trusted channel (for example, against the active certificate on the server). Its Subject Alternative Name (SAN) must match the host/IP used in the URL; an IP address must appear as an **IP Address** SAN, not just `DNS:...` or the Common Name.
4. A model available on the server. Its exact Ollama tag can be checked with `/api/tags` or `ollama list` on the server.

Never copy the TLS private key to the client. Do not disable certificate verification to work around trust errors.

## 1. Install OpenCode V2 on the client

### Linux

```bash
curl -fsSL https://opencode.ai/v2/install | bash
opencode --version
```

Alternatively, if Node.js/npm is available:

```bash
npm install -g @opencode/cli
```

### Windows PC

Install a current Node.js/npm release, then in PowerShell:

```powershell
npm install -g @opencode/cli
opencode --version
```

The OpenCode V2 installation page also provides downloadable Windows binaries. Use the V2 installation instructions rather than a V1 package name.

## 2. Obtain and convert the server certificate

Get the **active public certificate** from the server administrator. Compare its SHA-256 fingerprint against the live server certificate over a trusted channel before using it. An older exported certificate may not match the one currently served.

### Linux

If the exported certificate is DER (`.cer`):

```bash
mkdir -p "$HOME/.config/ollama"
openssl x509 -inform DER \
  -in "$HOME/Downloads/ollama-server.cer" \
  -out "$HOME/.config/ollama/server.crt"

openssl x509 -in "$HOME/.config/ollama/server.crt" \
  -noout -fingerprint -sha256 -ext subjectAltName
```

If it is already PEM (the file begins with `-----BEGIN CERTIFICATE-----`), copy it to `~/.config/ollama/server.crt` instead of converting it.

> The extension is arbitrary — `.crt`, `.pem` and `.cer` are all used for both encodings. What matters is that the file stunnel and curl read is **PEM text**, not DER binary. Check with `head -c 30 ~/.config/ollama/server.crt`: PEM starts with `-----BEGIN CERTIFICATE-----`. If you use a different filename, change it in `stunnel/ollama.conf` and the `--cacert` commands to match.

Keep this PEM file; stunnel uses it in Step 2b.

### Windows PC (PowerShell)

If the exported certificate is DER (`.cer`), convert it to PEM using built-in PowerShell/.NET:

```powershell
$dir = Join-Path $HOME '.config\ollama'
New-Item -ItemType Directory -Force -Path $dir | Out-Null
$bytes = [IO.File]::ReadAllBytes((Join-Path $HOME 'Downloads\ollama-server.cer'))
$pem = "-----BEGIN CERTIFICATE-----`n" +
       [Convert]::ToBase64String($bytes, [Base64FormattingOptions]::InsertLineBreaks) +
       "`n-----END CERTIFICATE-----`n"
$certPath = Join-Path $dir 'server.crt'
[IO.File]::WriteAllText($certPath, $pem, [Text.Encoding]::ASCII)
```

If it is already PEM, copy it to `$HOME\.config\ollama\server.crt`.

> **Verified on 2.0.16:** the certificate is *not* trusted by setting an environment variable. `NODE_EXTRA_CA_CERTS`, `SSL_CERT_FILE` and `BUN_CONFIG_EXTRA_CA_CERTS` were each tested against a self-signed gateway and **all three were ignored** — the request still failed with `UNABLE_TO_VERIFY_LEAF_SIGNATURE`, as did the certificate installed in the OS trust store. The standalone OpenCode binary is Bun-compiled and does not consult them. (This was measured on the Linux standalone binary; an npm/Node installation may behave differently, but the tunnel in Step 2b works on both platforms either way.) Do **not** work around this with `NODE_TLS_REJECT_UNAUTHORIZED=0`.

## 2b. Terminate TLS locally with stunnel

stunnel verifies the server certificate and exposes a plain **localhost** HTTP port for OpenCode. Certificate verification stays on; it simply happens in stunnel instead of OpenCode.

Skip this step only if the gateway presents a publicly-trusted certificate.

### Linux

```bash
sudo apt install stunnel4        # or your distribution's package
mkdir -p "$HOME/.config/stunnel"
cat > "$HOME/.config/stunnel/ollama.conf" <<'EOF'
pid = /home/YOUR_USER/.config/stunnel/stunnel.pid
foreground = yes

[ollama]
client = yes
accept = 127.0.0.1:11435
connect = SERVER_HOST_OR_IP:443
CAfile = /home/YOUR_USER/.config/ollama/server.crt
verifyPeer = yes
EOF
```

Replace `YOUR_USER` and `SERVER_HOST_OR_IP`. The `pid` line matters: without it stunnel tries to write `/var/run/stunnel4.pid` and exits with `Cannot create pid file … Permission denied`.

Start it (it does not auto-start; `foreground = yes` means it dies with its terminal, so detach it):

```bash
pgrep -x stunnel >/dev/null || nohup stunnel "$HOME/.config/stunnel/ollama.conf" > "$HOME/.config/stunnel/stunnel.log" 2>&1 &
ss -ltn | grep 11435          # expect a LISTEN line
```

To start it automatically at login, create a systemd user unit at `~/.config/systemd/user/stunnel-ollama.service` and enable it with `systemctl --user enable --now stunnel-ollama`.

### Windows PC (PowerShell)

Install stunnel for Windows from the [official downloads](https://www.stunnel.org/downloads.html), then create `%USERPROFILE%\.config\stunnel\ollama.conf`:

```powershell
$dir = Join-Path $HOME '.config\stunnel'
New-Item -ItemType Directory -Force -Path $dir | Out-Null
$conf = @"
[ollama]
client = yes
accept = 127.0.0.1:11435
connect = SERVER_HOST_OR_IP:443
CAfile = $HOME\.config\ollama\server.crt
verifyPeer = yes
"@
[IO.File]::WriteAllText((Join-Path $dir 'ollama.conf'), $conf, [Text.Encoding]::ASCII)
```

Start stunnel with that configuration (the Windows build can also be installed as a service so it starts at boot), then confirm the listener:

```powershell
Test-NetConnection 127.0.0.1 -Port 11435
```

## 3. Test the authenticated HTTPS API

Replace the server placeholder in the example URL. Use the **gateway login**, not your local PC username. The password is entered interactively and should not be placed in a URL, JSON file, or command history.

### Linux

```bash
curl --cacert "$HOME/.config/ollama/server.crt" \
  -u 'YOUR_GATEWAY_USERNAME' \
  'https://SERVER_HOST_OR_IP/v1/models'
```

### Windows PC (PowerShell)

```powershell
$GatewayUser = Read-Host 'Gateway username'
curl.exe --cacert "$HOME\.config\ollama\server.crt" `
  -u $GatewayUser `
  'https://SERVER_HOST_OR_IP/v1/models'
```

Curl prompts for the password. A successful JSON response should list the available model(s). If `/v1/models` works but an inference request does not, check the server gateway's forwarding of `/v1/chat/completions` with its administrator.

With stunnel running, the same call through the local tunnel must also succeed — this is the exact endpoint OpenCode will use:

```bash
curl -u 'YOUR_GATEWAY_USERNAME' http://127.0.0.1:11435/v1/models
```

## 4. Store the authentication header in a credential file

OpenCode needs the gateway's `Authorization: Basic ...` header. Write the Base64 pair into a permissions-restricted file; the config in Step 5 reads it with `{file:...}`.

A file is used rather than an environment variable because `{env:...}` is resolved **inside OpenCode's long-lived background service**, using that process's own environment. The variable therefore has to be exported in a shell *before* the service starts, and changing it requires `opencode service restart`. A stale service environment is a common cause of `401` even when `curl` works. `{file:...}` is re-read whenever the config loads, so no shell profile and no service restart are involved.

Basic Auth is Base64 encoding, **not encryption**; TLS protects the header in transit. Treat the file as a credential.

### Linux (Bash/Zsh)

```bash
mkdir -p "$HOME/.config/opencode"

printf 'Gateway username: '; read -r  GATEWAY_USER
printf 'Gateway password: '; read -rs GATEWAY_PASSWORD; printf '\n'

umask 077
printf '%s:%s' "$GATEWAY_USER" "$GATEWAY_PASSWORD" | base64 | tr -d '\n' > "$HOME/.config/opencode/basic_auth"
unset GATEWAY_USER GATEWAY_PASSWORD
chmod 600 "$HOME/.config/opencode/basic_auth"
```

> **Paste this block as whole lines.** If your terminal wraps or splits a line mid-command you get `zsh: parse error near '\n'` — most often from a line break landing right after a `>` redirect. If that happens, run the steps one line at a time, or use the single-line form: `umask 077; printf "Gateway username: "; read -r U; printf "Gateway password: "; read -rs P; printf "\n"; printf "%s:%s" "$U" "$P" | base64 | tr -d "\n" > "$HOME/.config/opencode/basic_auth"; unset U P; chmod 600 "$HOME/.config/opencode/basic_auth"`

> **Do not use `read -p` here.** It prompts in Bash, but in Zsh `-p` means *read from a coprocess*, so the command fails with `read: -p: no coprocess`, both variables stay empty, and the pipeline still writes a file — containing `Og==`, the Base64 of a bare `:`. Nothing reports an error, and OpenCode then returns `401` against a gateway that works fine in `curl`. The form above uses `printf` for the prompt and `read -rs` for the silent read, both of which behave identically in Bash and Zsh.

Verify immediately — this is the step that catches the failure above:

```bash
wc -c < "$HOME/.config/opencode/basic_auth"        # must be well above 4
base64 -d < "$HOME/.config/opencode/basic_auth"; echo
```

The decoded value must read `yourusername:yourpassword`. A bare `:` means both prompts were skipped. Then confirm the gateway accepts it:

```bash
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Basic $(cat "$HOME/.config/opencode/basic_auth")" http://127.0.0.1:11435/v1/models
```

`200` is correct; `401` means the credential is wrong.

### Windows PC (PowerShell)

```powershell
$dir = Join-Path $HOME '.config\opencode'
New-Item -ItemType Directory -Force -Path $dir | Out-Null
$GatewayUser = Read-Host 'Gateway username'
$SecurePassword = Read-Host 'Gateway password' -AsSecureString
$ptr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($SecurePassword)
try {
    $plain = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($ptr)
    $pair = '{0}:{1}' -f $GatewayUser, $plain
    $b64 = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($pair))
    [IO.File]::WriteAllText((Join-Path $dir 'basic_auth'), $b64, [Text.Encoding]::ASCII)
} finally {
    [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($ptr)
    Remove-Variable plain, pair, b64, GatewayUser, SecurePassword -ErrorAction SilentlyContinue
}
```

Then restrict the file to your account:

```powershell
$f = Join-Path $HOME '.config\opencode\basic_auth'
icacls $f /inheritance:r /grant:r "$($env:USERNAME):(R,W)"
```

The file contains no newline requirement — OpenCode trims surrounding whitespace. Never commit it to Git or paste its value into screenshots or logs. To rotate the password, overwrite this one file and run `opencode reload`.

## 5. Configure OpenCode V2

Use the same global config on Linux and Windows; OpenCode resolves `~/.config/opencode/opencode.json` to the user's home directory on each platform.

**Linux:**

```bash
mkdir -p "$HOME/.config/opencode"
nano "$HOME/.config/opencode/opencode.json"
```

**Windows PowerShell:**

```powershell
$dir = Join-Path $HOME '.config\opencode'
New-Item -ItemType Directory -Force -Path $dir | Out-Null
notepad (Join-Path $dir 'opencode.json')
```

Save the following JSON exactly as shown. With stunnel in place the endpoint is the **local** tunnel, so no server address appears in this file at all, and neither does the password.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "flw-ollama/qwen3.5:27b",
  "small_model": "flw-ollama/qwen3.5:27b",
  "share": "disabled",
  "enabled_providers": ["flw-ollama"],
  "provider": {
    "flw-ollama": {
      "name": "GPU Ollama Server",
      "npm": "@ai-sdk/openai-compatible",
      "options": {
        "baseURL": "http://127.0.0.1:11435/v1",
        "headers": {
          "Authorization": "Basic {file:~/.config/opencode/basic_auth}"
        }
      },
      "models": {
        "qwen3.5:27b": {
          "name": "Qwen3.5 27B (GPU server)",
          "tool_call": true,
          "interleaved": "reasoning",
          "modalities": { "input": ["text"], "output": ["text"] },
          "cost": { "input": 0, "output": 0 },
          "limit": { "context": 16384, "output": 4096 }
        }
      }
    }
  },
  "compaction": {
    "auto": true,
    "reserved": 4096,
    "preserve_recent_tokens": 4096
  },
  "permission": {
    "edit": "ask",
    "bash": "ask",
    "webfetch": "ask"
  }
}
```

Field notes, all verified against 2.0.16:

- `provider` / `npm` / `options` are the user-facing keys. Internally OpenCode rewrites them to `package` / `settings` / `headers`, which is why configs copied from that internal shape parse without complaint but never register a provider.
- `options.headers` is split out of `options` by the loader, so a custom `Authorization` header belongs there, not beside `options`.
- `{file:~/...}` expands `~`, accepts absolute or config-relative paths, and trims trailing whitespace. If the file is missing, OpenCode fails loudly with `ConfigInvalidError` rather than falling back to a hosted provider.
- `enabled_providers` denies every other provider, including paid and bundled free ones. **It is applied at service start, so it needs `opencode service restart`, not just `opencode reload`.**
- `tool_call: true` is required for coding-agent use. `interleaved: "reasoning"` tells OpenCode to read Qwen's chain-of-thought from the `reasoning` field that Ollama returns; without it, replies can arrive blank.
- `permission` is an object in 2.x. A `permissions` array is not a valid key and is ignored.
- Settings that do **not** exist in 2.0.16 and are dropped with an `omitted unsupported legacy setting` warning: `modelID`, `capabilities`, `reasoning`, `temperature`, `attachment`, `compaction.prune`, `limit.input`.

The 16K context limit is OpenCode's **declared budget**; actual allocation is controlled by the server and any request overrides.

Validate the file before going further — the plain reader stays silent about unknown keys, so ask the validator instead:

```bash
opencode models --print-logs --log-level warn
```

Any `configuration normalization diagnostic` line names a key this version does not support. `opencode debug config` will **not** tell you this: it echoes the raw file, and its redaction of `Authorization` proves only that a key with that name exists.

## 6. Start OpenCode

There is nothing to export and no per-terminal setup. Two things must be true before launching: **stunnel is listening on `127.0.0.1:11435`**, and **`~/.config/opencode/basic_auth` exists**.

### Linux

```bash
# 1. tunnel (once per boot; skip if already running)
pgrep -x stunnel >/dev/null || nohup stunnel "$HOME/.config/stunnel/ollama.conf" > "$HOME/.config/stunnel/stunnel.log" 2>&1 &

# 2. launch in your project directory
cd "$HOME/opencode-test"
opencode
```

### Windows PC (PowerShell)

```powershell
# 1. tunnel (once per boot; skip if installed as a Windows service)
if (-not (Get-Process stunnel -ErrorAction SilentlyContinue)) {
    Start-Process stunnel -ArgumentList "$HOME\.config\stunnel\ollama.conf" -WindowStyle Hidden
}

# 2. launch in your project directory
Set-Location (Join-Path $HOME 'opencode-test')
opencode
```

That is the whole routine on both platforms, every session. Only the first run needs Steps 1–5.

### Verify it is really using your server

```bash
opencode models
```

It must print exactly one line, `flw-ollama/qwen3.5:27b`. If other providers appear, `enabled_providers` has not been applied — run `opencode service restart`.

> Use plain `opencode models`, **not** `opencode models --standalone`. On 2.0.16 the `--standalone` variant returns an empty list under all conditions, including with no config file at all, so it cannot confirm or deny anything. (`opencode run --standalone` does work; it is just the model listing that is broken.)

Then a real request:

```bash
opencode run "Reply with exactly: OK"
```

Once that answers, start the interactive session with `opencode` and try a small prompt:

> Explain this directory. Do not edit files or run commands without asking.

Then a small coding task. Files and approved tools run on the PC, while model inference runs on the GPU server. On the server, `ollama ps` displays the loaded model, context and GPU/CPU allocation — that is the authoritative confirmation the request reached your GPU and not a hosted provider.

### Changing the configuration later

| Change | What to run |
|---|---|
| Edited `opencode.json` (models, limits, permissions) | `opencode reload` |
| Rotated the password in `basic_auth` | `opencode reload` |
| Added or changed `enabled_providers` | `opencode service restart` |
| Rebooted the PC | start stunnel, then `opencode` |

## Current server model and changing models

| Setting | Current value |
|---|---|
| GPU server | Windows, NVIDIA RTX A4500 (20 GB VRAM) |
| Ollama model | `qwen3.5:27b` |
| Working context | `16384` tokens (16K) |
| Parallel requests | `1` |
| Observed residency at 16K | `100% GPU` |
| Cloud features | Disabled (`OLLAMA_NO_CLOUD=1`) |

These are the working settings observed for this setup, not a guarantee that a larger context will fit in VRAM. At a 131,072-token context, the model previously used CPU offloading.

**Budgeting 16K.** Measured against this server, an empty agent session already costs **7,937 prompt tokens** — OpenCode's system prompt plus twelve tool schemas — before you type anything. With `compaction.reserved: 4096` that leaves roughly 4.4K tokens of actual conversation before the first compaction, and every turn re-processes ~8K tokens, which is the main reason responses take minutes rather than seconds. The `compaction` block in Step 5 is sized for this. To buy back context, drop tool schemas you do not need:

```json
"tools": { "websearch": false, "webfetch": false, "subagent": false, "skill": false, "question": false }
```

Measured effect: 7,937 → 6,608 prompt tokens.

**To switch models:**

1. On the server, install the new model with `ollama pull MODEL_TAG` and verify it with `ollama list`.
2. Check the model and context with `ollama run MODEL_TAG`, followed by `ollama ps` after sending a prompt. If necessary, set context through Ollama's application setting, server environment, or a model's Modelfile; restart/reload Ollama as appropriate.
3. In `opencode.json`, replace the model key, its `name`, and the top-level `model` and `small_model` references with the new tag. The model key *is* the model ID — there is no `modelID` field, and a colon in the tag is fine. Adjust `limit.context` and `limit.output` to match the **actually usable** server context (`limit.input` is not a supported field). Alternatively, add a second entry under `provider.flw-ollama.models` and choose it with `/models`.
4. Run `opencode reload`, then verify GPU residency on the server with `ollama ps`.

A larger model or context may trigger CPU offloading; check performance before keeping the change.

## Troubleshooting

| Symptom | What to check |
|---|---|
| `curl: (77)` | PEM file missing, unreadable, or wrong format. |
| Self-signed certificate error with `--cacert` or in stunnel | Local file may be an older certificate; compare the live server and exported certificate SHA-256 fingerprints. |
| Certificate name/SAN mismatch | URL must match a DNS or IP SAN; a numeric IP needs an **IP Address** SAN. |
| `401 Unauthorized` | Decode the credential file first: `base64 -d < ~/.config/opencode/basic_auth`. If it prints a bare `:`, the prompts were skipped — see the `read -p` warning in Step 4. Otherwise confirm `curl -u USER http://127.0.0.1:11435/v1/models` succeeds. If you are still using `{env:...}`, the background service is holding a stale value — `opencode service restart`. |
| `404`, `502`, or timeout | Verify the existing gateway forwards the OpenAI-compatible `/v1` routes. |
| `ConnectionRefused` / nothing on `127.0.0.1:11435` | stunnel is not running. `pgrep -x stunnel` (Linux) or `Get-Process stunnel` (Windows); restart it as in Step 2b. Check `stunnel.log` for `Cannot create pid file` — add the `pid =` line. |
| `UNABLE_TO_VERIFY_LEAF_SIGNATURE` | OpenCode is talking to the gateway directly instead of the tunnel. Confirm `options.baseURL` is `http://127.0.0.1:11435/v1`. Certificate environment variables do not help on 2.0.16; do not disable verification. |
| `ConfigInvalidError` | `{file:...}` points at a file that does not exist. Create `basic_auth` as in Step 4. |
| Model unavailable / provider missing from `opencode models` | Almost always a config-key problem, not a server problem. Run `opencode models --print-logs --log-level warn` and fix every `omitted unsupported legacy setting`. Confirm `provider` (not `providers`), `npm` (not `package`), `options.baseURL` (not `settings.baseURL`). |
| Hosted or free providers still listed | `enabled_providers` needs `opencode service restart`; `opencode reload` alone does not apply it. |
| Blank replies from the model | Qwen returns its reasoning in a separate field. Ensure `interleaved: "reasoning"` is set on the model. |
| Unexpected CPU use / sluggishness | Check `ollama ps`; reduce context or use a smaller model if necessary. Minutes-long turns at 16K are expected — see *Budgeting 16K*. |

## Security notes

- Keep authentication on the HTTPS gateway and direct Ollama access restricted. Do not expose Ollama's internal port as an unauthenticated public API.
- Use a *verified* public server certificate; never transfer its private key. Avoid `curl -k` and `NODE_TLS_REJECT_UNAUTHORIZED=0` as permanent fixes.
- Never hard-code actual host addresses, credentials, or authorization headers in a README or committed project config. With the local tunnel, the server address lives only in `stunnel/ollama.conf` and the credential only in `basic_auth` — keep both out of version control.
- Do not echo the credential in a shell. `echo "${VAR:-fallback}"` prints the value when the variable *is* set, which is an easy way to leak it into a terminal log. If a credential is ever exposed, rotate it at the gateway and overwrite `basic_auth`.
- OpenCode sends relevant prompts and code to the GPU server, so a self-hosted endpoint is not the same as keeping all content on the client PC.
- A local model avoids paid *model API* charges, but any optional web tools or third-party integrations should be configured separately.

## Official references

- [OpenCode V2 installation](https://opencode.ai/v2/docs/)
- [OpenCode V2 providers and remote Ollama endpoints](https://opencode.ai/v2/docs/providers)
- [OpenCode V2 models and limits](https://opencode.ai/v2/docs/models)
- [OpenCode V2 network and certificate settings](https://opencode.ai/v2/docs/network/)
- [OpenCode V2 permissions](https://opencode.ai/v2/docs/permissions)
- [OpenCode V2 provider policies](https://opencode.ai/v2/docs/policies/)
- [Ollama API documentation](https://docs.ollama.com/api)
