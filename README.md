# bashd — disconnect-proof local & remote bash for agents

One file. Python stdlib only. Two faces: an **MCP stdio server** and a **CLI** over the same core.

**v0.2 — one substrate: persistent tmux sessions.** Every command runs *inside* a named
tmux session on the target host (default `main`). What that buys: **the environment is
persistent by design** — `cd`, `export`, venv activation survive across calls and across
connections. Output is still captured to per-stream files with real exit codes, so cursor
reads, regex waits and idempotent retries work exactly as before.

**v0.3.1 — submits are one typed line.** The eval *and* its exit-code capture are a single
constant-size line typed into the session, and the command payload (plus `--cwd`/`--env`)
is written to the job's files by the submit script itself over ssh — the session tty only
ever carries ~230 bytes per job. Concurrent submits to one session serialize on a
target-side lock (which also covers session creation), and a job holds its lock for its
whole execution — reads, the fleet overview and prune use that to tell *executing*,
*queued* and *dead orphan* jobs apart, so a submit whose lines were lost finalizes as 143
instead of lingering as "running" forever.

**v0.4 — a human terminal and a wire transport.** `bashd term HOST` attaches your
terminal to any persistent session: turn-based by nature (a local line editor, so only
Enter and Ctrl-C ever cross the network) yet an ordinary terminal over a good link (raw
byte passthrough, output *pushed* by the target, resize followed, vim-class apps work).
It rides a custom transport built for extremely slow and unstable networks: one
long-lived ssh channel running a tiny frame agent instead of one ssh exec per operation —
gzip-tagged payloads, 10 Hz output push, and reconnect-and-resume by byte cursor when
the link dies mid-stream. Details below.

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
  `ServerAlive` keepalives, cost-ordered multi-path fallback (direct / args overrides /
  nested hops), host aliases resolved through your normal ssh config. No paramiko.
- **Routes self-heal.** A mid-command transport death fails the call over to the next path
  automatically — sync `run` retries its idempotent submit across short outages (up to 15 s)
  and its poll loop simply switches routes, reporting `link_recovered`/`link_drops`; errors
  carry a per-path attempt trail (`attempts: path0:unreachable; path1:timeout`). Once a
  costlier fallback latches, a detached background probe keeps testing the cheaper path
  (backoff 30 s → 15 min) and re-latches it on recovery — no calls are blocked by probing.
- **Host registry with notes.** `~/.bashd/hosts.json` maps aliases to connection details and
  per-host notes that ride along in every response (fleet gotchas become first-class metadata).
- **A human terminal and a wire transport (v0.4).** `bashd term` attaches you to any session —
  raw passthrough with pushed output on good links (an ordinary terminal), a local line editor
  on terrible ones (only Enter/Ctrl-C cross the network). The wire transport behind it is one
  long-lived ssh channel per attach with gzip-tagged frames, 10 Hz push, and
  reconnect-and-resume by cursor — see "v0.4" below.

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

- `paths` — tried cheapest-first per attempt; the last-good path is **sticky** until it
  fails, and a detached background probe re-latches the cheapest path when it recovers
  (exponential backoff, 30 s → 15 min; `bashd paths` shows the state, `--repath` forces a
  probe now). Forms — a dict form may carry **`"cost"`** (lower = preferred; default cost =
  position, so config order IS the preference order):
  - `"direct"` — plain connection to `ssh`;
  - ssh args (string or list) — e.g. `"-o HostName=203.0.113.99"` (override destination),
    `"-J jump@host"` (TCP relay through a jump);
  - `{"args": ["-J", "jump@host"], "cost": 1}` — args form with an explicit cost;
  - `{"hop": "root@relay.example", "hop_args": "-i /root/.ssh/id_ed25519_relay", "cost": 5}` —
    **nested ssh hop**: first ssh to `hop`, then from that shell ssh onward to `ssh` (or
    `"target"`) using the hop's own keys. The reliable shape for flaky relays where `-J`
    TCP-forwarding stalls at banner exchange, and for keys that live on the relay rather
    than locally. Give it a higher cost than the direct path so the auto-probe promotes
    the direct path back once it recovers.
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
                                               #   (running jobs in full incl cmd+age;
                                               #    finished summarized: count+bytes+5 recent)
bashd prune web                                # remove finished job records + dead tty logs
bashd prune web --stale 3600                   # also reap codeless jobs quiet 1h (wedged, session alive)
bashd paths web                                # route table: costs, sticky path, probe backoff
bashd paths web --repath                       # probe cheaper paths now, re-latch the first that works
bashd term web                                 # HUMAN terminal on session 'main' (v0.4)
bashd term web --session ops --mode turn       # line-editor input (bad links); ~l switches back
bashd hosts                                    # show registry
bashd selftest                                 # 26 end-to-end checks
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

## v0.4 — `bashd term`: a human terminal + the wire transport

```bash
bashd term HOST [--session main] [--mode auto|live|turn] [--no-wire] [--scrollback 4096]
```

Attaches your terminal to a persistent session — the same substrate the agents use, so
you can join `main` and see the environment every `bashd run` built, or attach to any
other session. It is a **turn-based terminal that behaves like an ordinary one over a
good connection**:

- **live mode** — raw byte passthrough both ways over the wire transport: the target's
  *pushed* output deltas make echo latency ≈ one network RTT; every keystroke, arrow key,
  Ctrl-C and full-screen app (vim, htop) works exactly as over ssh. Resizing your
  terminal resizes the remote pane. Auto-selected when the measured RTT is healthy
  (< 0.7 s on wire, < 0.25 s on per-call ssh, always for `local`).
- **turn mode** — a local line editor (history, Ctrl-A/E/U/K/W, arrows, Home/End): only
  **Enter** and **Ctrl-C** ever cross the network, one small frame each — built for very
  slow or unstable links, where per-keystroke round trips and remote echo are torture.
  Output still streams as it is produced. Auto-selected when RTT is high, or after two
  link drops within two minutes (auto-degrade, never auto-upgrade).
- tilde escapes: `~.` detach (the session lives on), `~l`/`~t` switch input mode, `~i`
  link stats, `~~` a literal tilde. In turn mode type them as a line + Enter; in live
  mode they are the first keys after a fresh line, ssh-style.
- attach shows the last `--scrollback` bytes of history; if the session was killed by a
  bare `exit`, the next line you submit recreates it (fresh env).
- `--no-wire` forces the classic one-ssh-exec-per-operation transport — same UX, higher
  latency, zero long-lived channels (restrictive relays, fire-and-forget environments).

**The wire transport** (what `term` rides whenever it can establish one) is a custom
protocol for extremely slow and unstable networks: ONE long-lived ssh channel running a
tiny target-side agent, instead of one `ssh … sh -s` exec per operation. Frames are
single text lines; payloads are base64, gzip-tagged when that is smaller, and the ssh
channel itself is compressed (`-C`). The agent **pushes** output deltas at 10 Hz (a
POSIX-sh flavor without push covers targets that lack bash). If the channel dies, it is
re-spawned over the next configured path (sticky-first, like every bashd call), output
resumes from the last delivered cursor — nothing is lost or duplicated, because sessions
and logs live in target-side tmux the whole time. Input that was queued locally while
the link was down flushes on reconnect; input that was *sent but not acked* when the
link died is reported and dropped — never replayed, because a replayed Enter could
execute a line twice. Field-tested over a real flaky link: a forced mid-stream channel
death on a ~200 ms two-hop relay path reconnects in under a second with byte-exact
stream continuity.

Two humans (or a human and the tilde escapes) attaching the same session see the same
stream; note that a human typing into a session *while an agent submits a job into it*
interleaves on the pty like two typists — use a separate `--session` for interactive
work if that matters.

Fix in passing: `write`/`bash_write` with `append_newline=false` no longer presses
Enter (the Enter send used to land in the script unconditionally — visible as commands
executing when they should only have been typed).

## Requirements & limits

- Targets need a POSIX shell, tmux (all modes), `flock` (util-linux — everywhere on
  Linux) and `base64`/`wc`/`tail`.
- Key-based ssh auth (BatchMode); password prompts are never attempted.
- Session-lifetime output cap (`ulimit -f`, default 512 MB, applied at session creation) so
  runaway commands can't fill the target's disk while unwatched.
- One command at a time per session (extras queue in the shell — order preserved). Submits
  to one session serialize on a target-side lock, so several clients may safely submit
  concurrently; each job then holds its own lock while executing.
- Sessions survive any disconnect but not a target reboot (tmux dies; job records persist
  and are finalized as 143 — on read, on `sessions`, and at prune — so nothing lingers as
  "running" forever). v0.3.1 adds the **orphan rule**: even with the session alive, a
  codeless job whose lock nobody holds, whose files have been quiet ≥ 15 min, and whose
  session has no executing job is finalized as 143 — that is the fate of a submit whose
  typed line was lost, which previously stuck as "running" forever. `prune --stale
  SECONDS` additionally reaps codeless records whose output has been quiet that long even
  while held (wedged executor; opt-in — a legitimately silent long run could match).
- Ctrl-C (`bashd signal`, any sig — jobs always get Ctrl-C) aborts the job's line; the
  code file lands via signal's 3-second fallback as 130. A job that *ignores* SIGINT keeps
  running with a 130 record — stop it with signal KILL on its tty session or kill the
  session.
- `HOME` on targets must be slash-free (standard on Linux) — pipe-pane log paths rely on it.

## Status

v0.4.0 — selftest 26/26; the wire transport and `term` verified end-to-end locally, over
a nested-hop path, and over a real ~200 ms two-hop relay link (push latency, forced
mid-stream channel death → 0.9 s reconnect with byte-exact resume, Ctrl-C delivery,
resize following, auto live/turn selection). `write --no-newline` fixed (it used to
press Enter anyway). v0.3.1 — concurrency-stress verified (10×6 concurrent submits, zero
lost/stuck jobs). Root-caused and fixed the "stuck running forever" class found in field
use (16 records on one host): a job's exit-code capture used to be a *separate typed
line* queued behind the command, and queued tty input can be discarded (input-buffer
overflow under concurrent submits, SIGINT input flush, readline edges) — the finished
command then never recorded a code. Submits now type exactly one constant-size line
(payload and cd/env travel in the submit script over ssh, never through the tty),
serialize per session on a target-side `flock` that also covers session creation (the
creator's 0.3 s init sleep used to race concurrent submitters into its stale-stamp),
and hold the job lock during execution as the liveness marker for the new orphan rule
(read/sessions/prune self-heal lost submits as 143 after a 15-minute grace). The fleet
overview gained cmd previews and ages and now shows running jobs in full with finished
jobs summarized. v0.3.0 added **cost-aware routing with auto re-latch** (per-path `cost`,
sticky path, detached background probe that promotes a recovered cheap route without
blocking any call, `bashd paths [--repath]` / `bash_hosts {repath:true}` for state and
manual probing) and **mid-command failover** (`run` retries its idempotent submit across
short outages; errors carry a per-path attempt trail; `run`/`wait` report
`link_drops`/`link_recovered` around route switches). v0.2.1: dead-session job records
finalize as 143 everywhere, `prune --stale`, ControlMaster on the inner leg of nested-hop
paths. v0.2.0 was fleet-tested 2026-09-24 on remote hosts over real ssh:
**cross-connection env/cwd persistence verified end-to-end**, and the multi-path fallback
proven in the field — one fleet host was reachable that day only through its
second-choice relay path, and the sticky-path logic latched onto it transparently
(`bashd sessions <host>` re-probes every configured path in one call once the network
heals). Design rationale and tool survey: `docs/RESEARCH.md`.
