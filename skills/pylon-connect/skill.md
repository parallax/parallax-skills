---
name: pylon-connect
description: |
  Install the pylon-runner binary on this machine and connect it to a Pylon
  webapp instance. Use when the user says "install Pylon", "connect my runner",
  or the Pylon dashboard shows "Connect runner".
---

# Pylon Connect

Installs the pylon-runner binary, handles macOS setup, and walks through
device-flow pairing so the runner shows as online in the Pylon dashboard.

## Before you start

You need the Pylon webapp URL. It's either in the user's message
("connect it to https://…"), in the `PYLON_API_URL` environment variable,
or ask the user: "What's your Pylon webapp URL?"

Set it for this session:
```bash
export PYLON_API_URL="<url from user>"   # e.g. https://pylon-webapp.onrender.com
```

## Step 1 — Check if already installed

```bash
~/.pylon/bin/pylon-runner --version 2>/dev/null && echo "already installed" || echo "not installed"
```

If already installed and version matches what the webapp requires, skip to Step 4.

## Step 2 — Check binaries are available

```bash
curl -fsSL "${PYLON_API_URL}/api/runner/release" | python3 -m json.tool 2>/dev/null || echo "release endpoint unavailable"
```

Stop and tell the user if the endpoint returns an error or shows no platforms — binaries may not be published yet.

## Step 3 — Download and install

```bash
curl -fsSL "${PYLON_API_URL}/api/runner/install" | sh
```

This script:
- Downloads the right binary for the current platform
- Installs to `~/.pylon/bin/pylon-runner`
- On macOS: removes the quarantine flag and applies an ad-hoc signature
- Verifies the binary runs

If it fails, check the error output. Common issues:
- **No binary for platform**: not published yet — tell the user
- **codesign: command not found**: Xcode Command Line Tools not installed — run `xcode-select --install`
- **Permission denied**: `chmod +x ~/.pylon/bin/pylon-runner` and retry

## Step 4 — Verify

```bash
~/.pylon/bin/pylon-runner --version
```

Must print a version string. If it doesn't run, stop and show the error.

## Step 5 — Reload MCP

Tell the user:

> The binary is installed. **Please reload the Pylon MCP server in your editor now:**
> - **Cursor**: click the MCP icon in the sidebar → click the reload icon next to "pylon"
> - **VS Code**: open the Command Palette → "MCP: Restart Server" → select "pylon"
> - **Claude Code**: nothing to do — the binary is already available
>
> Once reloaded, let me know and I'll complete the connection.

Wait for the user to confirm before continuing.

## Step 6 — Pair with the webapp

Call `pylon_status` (MCP tool — now available after reload):

The tool will either:
- Return "Pylon is ready" → already paired, runner is online → done
- Return a pairing URL and code → show them to the user verbatim:

> Open: https://…/pair/confirm
> Code: XXXX-XXXX
>
> Please open that URL and enter the code to authorise this machine.

Call `pylon_status` again once they confirm.

## Step 7 — Confirm runner is online

```bash
curl -s -H "Authorization: Bearer $(cat ~/.pylon/credentials.json | python3 -c 'import sys,json; m=json.load(sys.stdin); print(list(m.values())[0])')" "${PYLON_API_URL}/api/runners/status"
```

Should return `{"online":true,...}`. Tell the user the runner is online and the Pylon dashboard pill should now be cyan.

## Notes

- The binary at `~/.pylon/bin/pylon-runner` is both the MCP server (`--mcp`) and the runner (`--watch`). One binary does both.
- Credentials are stored per-URL in `~/.pylon/credentials.json`. Switching between local and production works automatically.
- If anything goes wrong, `pylon_logs` will show the runner log and credential state.
