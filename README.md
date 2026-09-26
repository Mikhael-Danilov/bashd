# bashd — disconnect-proof local & remote bash for agents

One file. Python stdlib only. Two faces: an **MCP stdio server** and a **CLI** over the same core.

**v0.2 — one substrate: persistent tmux sessions.** Every command runs *inside* a named
tmux session on the target host (default `main`). What that buys: **the environment is
persistent by design** — `cd`, `export`, venv activation survive across calls and across
connections. Output is still captured to per-stream files with real exit codes (the
submission wraps your command: `eval <cmd> > out 2> err`, then `echo $? > code`), so
cursor reads, regex waits and idempotent retries work exactly as before.

Built for agents working across a fleet over **bad, unstable networks**: all state and all
execution live on the *target* host, never in the tool process and never anchored to a
connection. Losing the link at any instant — mid-command, mid-wait, even mid-start — loses
no work and no output; every call reconnects and resumes by byte cursor.

## Design properties

- **Persistent sessions, per-host.** Sessions live in `tmux -L bashd` on the target with a
  `pipe-pane` log each. Different `--session` names are the isolation unit (separate env).
- **Sync `run` = submit + resumable tail.** Timeouts *detach* (the command keeps running
  in its session; you get its id back) — they never kill. Stopping work is an explicit
  `bashd signal` (Ctrl-C into the session; the code file still lands, as 130).
- **Idempotent starts.** Job ids are client-generated; re-issuing a start whose connection
  died en route is a no-op, never a double-run.
- **Cursor reads.** `read` returns bytes since a byte offset, plus status and exit code in
  one round trip. Reconnects replay from the last confirmed cursor — no gaps, no duplicates.
- **Events = blocking wait.** `wait` blocks until `exit` / `regex` / `idle`, tolerates link
  drops mid-wait, and fails fast with `tried_paths` when a host is down at entry. A job
  whose session died under it (target reboot, killed session, bare `exit`) is finalized as
  exit `143` on the next read — it never lingers as an eternal "running" record.
- **Transport is system ssh only.** ControlMaster reuse on *both* legs of a nested hop (the
  inner leg keeps its own mux socket on the relay, so warm calls skip the inner handshake),
  `ServerAlive` keepalives, ordered multi-path fallback (direct / args overrides / nested
  hops), host aliases resolved through your normal ssh config. No paramiko.
- **Host registry with notes.** `~/.bashd/hosts.json` maps aliases to connection details and
  per-host notes that ride along in every response (fleet gotchas become first-class metadata).

## Install

Copy `bashd` anywhere on `PATH` (it needs python3 ≥ 3.8, ssh, and tmux on **every target** —
v0.2 runs everything inside target-side tmux). Nothing else is installed on targets: sessions
and job records are created over plain ssh one-liners.

```bash
install -m755 bashd ~/bin/bashd
mkdir -p ~/.bashd && cp hosts.json.example ~/.bashd/hosts.json  # then edit
```

## Config: `~/.bashd/hosts.json`

```json
{
  "hosts": {
    "local": {},
    "web": {
      "ssh": "deploy@203.0.113.10",
      "paths": ["direct", "-J jump@203.0.113.7"],
      "connect": {"timeout_s": 5, "keepalive_s": 15},
      "defaults": {"timeout_s": 300},
      "notes": "prod; never restart nginx outside maintenance window"
    }
  }
}
```

- `paths` — tried in order per attempt; the last-good path is sticky until it fails. Forms:
  - `"direct"` — plain connection to `ssh`;
  - ssh args (string or list) — e.g. `"-o HostName=203.0.113.99"` (override destination),
    `"-J jump@host"` (TCP relay through a jump);
  - `{"hop": "root@relay.example", "hop_args": "-i /root/.ssh/id_ed25519_relay"}` — **nested ssh
    hop**: first ssh to `hop`, then from that shell ssh onward to `ssh` (or `"target"`)
    using the hop's own keys. The reliable shape for flaky relays where `-J` TCP-forwarding
    stalls at banner exchange, and for keys that live on the relay rather than locally.
- `ssh` may be a plain `user@host` **or an alias from `~/.ssh/config`** — keys, ProxyJump
  and everything else come from OpenSSH itself.
- `notes` are returned in `run`/`start`/`sessions` responses.
- `BASHD_HOME` (default `~/.bashd`) holds config + sticky-path state + ControlMaster
  sockets. The job substrate is always `~/.bashd/` *on each target host*, including local.

## CLI

```bash
bashd run web 'make test' --timeout 600        # sync; timeout detaches, returns id
bashd run web 'source ~/venv/bin/activate'     # persists in session 'main' — for all future runs
bashd run web 'make test' --session projectA   # isolation: separate env per session name
bashd start web 'backup.sh' --id backup-0417   # async; idempotent
bashd wait web:backup-0417 --until regex --pattern '^DONE$' --timeout 3600
bashd read web:backup-0417 --cursor-out 4096   # resumable read
bashd signal web:backup-0417                   # Ctrl-C into the session (code lands as 130)

bashd start web --mode tty --name ops          # bare interactive session
bashd write web:tty:ops 'sudo journalctl -u nginx -n 50'
bashd read web:tty:ops --screen                # live pane
bashd sessions                                 # fleet overview, parallel, fail-fast
bashd prune web                                # remove finished job records + dead tty logs
bashd prune web --stale 3600                   # also reap codeless jobs quiet 1h (wedged, session alive)
bashd hosts                                    # show registry
bashd selftest                                 # 19 end-to-end checks
```

**Persistent-environment semantics (v0.2):** `--cwd` and `--env` are applied in the session
and *stay applied* — that's the feature. Don't run bare `exit` (it kills the session; it
auto-recreates on next use, and queued state is lost). Commands in one session queue and
run one at a time; parallel work belongs in different `--session`s.

All output is JSON. CLI exit codes: `0` ok, `10` still running (detach/timeout), `2`
unreachable, `3` not found, `1` other error. `run`/`read`/`wait` are idempotent and
replayable by cursor — safe to retry after any failure.

**Long waits from an agent harness:** run the CLI as a background task
(`bashd wait ID --timeout 3600`) — the harness's own task-finished notification becomes
your event push; in-MCP, use the `bash_wait` tool with a modest timeout.

## MCP server

```json
{ "mcpServers": { "bashd": { "command": "/home/you/bin/bashd", "args": ["mcp"] } } }
```

Tools: `bash_run`, `bash_start`, `bash_read`, `bash_wait`, `bash_write`, `bash_signal`,
`bash_sessions`, `bash_prune`, `bash_hosts` — same semantics as the CLI. Hand-rolled stdio
JSON-RPC (line-delimited), no SDK dependency.

## Requirements & limits

- Targets need a POSIX shell, tmux (all modes now), and `base64`/`wc`/`tail` (any Linux).
- Key-based ssh auth (BatchMode); password prompts are never attempted.
- Session-lifetime output cap (`ulimit -f`, default 512 MB, applied at session creation) so
  runaway commands can't fill the target's disk while unwatched.
- One command at a time per session (extras queue in the shell — order preserved).
- Sessions survive any disconnect but not a target reboot (tmux dies; job records persist
  and are finalized as 143 — on read, on `sessions`, and at prune — so nothing lingers as
  "running" forever). `prune --stale SECONDS` additionally reaps codeless records whose
  output has been quiet that long while their session is still alive (wedged session;
  opt-in — a legitimately silent long run could match).
- `HOME` on targets must be slash-free (standard on Linux) — pipe-pane log paths rely on it.

## Status

v0.2.1 — selftest 19/19. Adds: dead-session job records finalize as 143 everywhere (the
"stuck running forever" wart), `prune --stale` for session-alive wedged records, and a
ControlMaster on the inner leg of nested-hop paths (warm hop calls drop from ~2× handshake
cost to one). v0.2.0 was fleet-tested 2026-09-24 on remote hosts over real ssh:
**cross-connection env/cwd persistence verified end-to-end**, and the
multi-path fallback proven in the field — one fleet host was reachable that day only
through its second-choice relay path, and the sticky-path logic latched onto it
transparently (`bashd sessions <host>` re-probes every configured path in one call once
the network heals). Design rationale and tool survey: `docs/RESEARCH.md`.
