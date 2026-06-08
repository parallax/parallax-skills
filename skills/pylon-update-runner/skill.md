---
name: pylon-update-runner
description: |
  Check whether the locally installed pylon-runner binary is up to date with
  the version the Pylon webapp requires, and download + replace it if not.
  Use when the dashboard shows "Runner outdated", when the user asks to update
  the runner, or proactively at the start of any session where the runner is
  needed and may be stale.
---

# Pylon Runner Update

Checks the installed `~/.pylon/bin/pylon-runner` version against what the
webapp requires, downloads the correct platform binary if outdated, and
restarts the watch process.

## Steps

### 1. Resolve API URL and token

```bash
export PYLON_API_URL="${PYLON_API_URL:-http://localhost:3000}"

PYLON_API_TOKEN=""
if [ -f ~/.pylon/credentials.json ]; then
  PYLON_API_TOKEN=$(python3 -c "
import json, sys
m = json.load(open('$HOME/.pylon/credentials.json'))
url = '${PYLON_API_URL}'.rstrip('/')
print(m.get(url.lower(), next(iter(m.values()), '')) )
" 2>/dev/null)
fi

if [ -z "$PYLON_API_TOKEN" ]; then
  echo "No token found in ~/.pylon/credentials.json for ${PYLON_API_URL}"
  echo "Run /pylon-connect to pair this machine first."
  exit 1
fi
```

### 2. Check installed version vs required version

```bash
release=$(curl -sS "$PYLON_API_URL/api/runner/release" \
  -H "Authorization: Bearer $PYLON_API_TOKEN")
required=$(echo "$release" | python3 -c "import sys,json; print(json.load(sys.stdin).get('version',''))" 2>/dev/null)

if [ -z "$required" ]; then
  echo "Could not reach $PYLON_API_URL/api/runner/release — is the webapp running?"
  exit 1
fi

installed=""
if [ -x ~/.pylon/bin/pylon-runner ]; then
  raw=$(~/.pylon/bin/pylon-runner --version 2>/dev/null || true)
  # Output format: "pylon-runner v0.1.0 (darwin/arm64)"
  installed=$(echo "$raw" | awk '{print $2}')
fi

echo "Installed : ${installed:-<not found>}"
echo "Required  : $required"
```

### 3. Skip if already up to date

```bash
if [ -n "$installed" ] && [ "$installed" = "$required" ]; then
  echo "Runner is up to date ($required) — nothing to do."
  exit 0
fi
```

### 4. Download and replace via the install script

```bash
curl -fsSL "$PYLON_API_URL/api/runner/install" | sh
```

This handles platform detection, atomic replacement, macOS quarantine removal, and ad-hoc signing.

### 5. Stop the old runner process (if running)

```bash
if [ -f ~/.pylon/runner.pid ]; then
  old_pid=$(cat ~/.pylon/runner.pid)
  if kill -0 "$old_pid" 2>/dev/null; then
    echo "Stopping old runner (PID $old_pid)…"
    kill "$old_pid" 2>/dev/null || true
    sleep 1
  fi
  rm -f ~/.pylon/runner.pid
fi
```

### 6. Relaunch the runner in watch mode

```bash
nohup ~/.pylon/bin/pylon-runner \
  --watch --token "$PYLON_API_TOKEN" --api "$PYLON_API_URL" \
  > ~/.pylon/runner.log 2>&1 &
echo $! > ~/.pylon/runner.pid
disown 2>/dev/null || true

echo "Runner restarted. Waiting for it to come online…"

for i in $(seq 1 30); do
  pylon_online=$(curl -sS "$PYLON_API_URL/api/runners/status" \
    -H "Authorization: Bearer $PYLON_API_TOKEN" \
    | python3 -c "import sys,json; print(json.load(sys.stdin).get('online',False))" 2>/dev/null)
  [ "$pylon_online" = "True" ] && break
  sleep 1
done

if [ "$pylon_online" = "True" ]; then
  echo "Runner online — updated to $required."
else
  echo "Runner did not come online within 30s. Check logs: tail ~/.pylon/runner.log"
  exit 1
fi
```

## Invocation triggers

Run this skill when:
- The dashboard shows **"Runner outdated"** (amber pill)
- `/api/runners/status` returns `upToDate: false`
- The user says "update the runner" or "my runner is outdated"
- Before a training job if the runner is offline and may be stale

## Notes

- Credentials are stored per-URL in `~/.pylon/credentials.json`. The script reads the token for the current `PYLON_API_URL`.
- The `--watch` binary requires `--token` on the command line. Credentials in `~/.pylon/credentials.json` are used by the MCP server internally; the watch process is always launched with an explicit token.
- The update is atomic: the install script uses a temp file + `mv` so a failed download never leaves a broken binary.
- `PYLON_API_URL` defaults to `http://localhost:3000` for local dev; in production the MCP config sets it automatically.
