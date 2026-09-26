# Research: local & remote bash tool for agents

**Date:** 2026-09-24 · rev 2 (disconnect-proofing) · **Ask:** a better bash tool — local *and*
remote, sync + async modes, holds sessions like screen/tmux, commands in and stdout/stderr out
either on request or on event (e.g. process finish), with connection details encapsulated.
**Hard constraint (rev 2):** *every* mode must survive disconnects — very bad, unstable networks
are the design target.

## 1. Requirements decomposed

| # | Requirement | Meaning in practice |
|---|-------------|---------------------|
| R1 | Local & remote | Same tool surface for `local` and fleet hosts; no per-call `ssh user@…` bookkeeping |
| R2 | Sync mode | Run, block (with timeout), get stdout/stderr + exit code |
| R3 | Async mode | Fire a job, get a handle immediately; fetch output later |
| R4 | Persistent sessions | Shell state (cwd, env, venv) persists across commands; survives agent/server restarts — the screen/tmux property |
| R5 | Output on request *or* event | Cursor-based reads AND a "wake me when it finishes/errors" primitive |
| R6 | Encapsulated connections | Named host profiles: user, key, jump chain, fallback paths; agent sees aliases only |
| R7 | Disconnect-proof (all modes) | Link loss at any instant — even mid-command — loses no work and no output; every call reconnects and resumes |

## 2. Existing landscape (checked 2026-09-24)

### Terminal/tmux MCP servers (local only)
- **[terminal-control-mcp](https://github.com/wehnsdaefflae/terminal-control-mcp)** — tmux/libtmux
  backend, 6 tools: open/list sessions, send_input, `get_screen_content` (screen/since_input/history/tail
  modes), **`await_output` (block until regex, timeout)**, exit. Sessions survive server exit (tmux holds
  them). Gaps: **local only**, no exit codes (prompt-pattern guessing), no connection registry.
- Generic **[tmux MCP servers](https://mcpservers.org)** (several listings, e.g. on
  [LobeHub](https://lobehub.com)) — same shape: session persistence via tmux, snapshot reads,
  interactive input. Local-only.

### Persistent-SSH MCP servers
- **[mcp-ssh-session](https://github.com/devnullvoid/mcp-ssh-session)** — closest to the ask on the
  remote side: connections reused across commands, shell state persists, **async exec returns a command
  ID + `get_command_status` polling**, Ctrl+C by ID, SFTP helpers, and a good encapsulation trick
  (`~/.ssh/config` auto-read + `OVRD_{alias}_HOST/USER/KEY` env vars so the agent sees only aliases).
  Gaps: completion detection = prompt pattern + 2 s idle timeout (fragile), **no stdout/stderr
  separation, no exit codes**. Author's own README recommends the tmux variant instead.
- **[mcp-ssh-tmux](https://github.com/devnullvoid/mcp-ssh-tmux)** (same author) — each SSH connection
  lives in a local tmux window; agent reads visual snapshots "like a human". Connection config via
  `ssh -G` (full ssh-config semantics: aliases, ProxyJump, IdentityFile — free encapsulation).
  Gaps: no formal async API, prompt-polling, snapshot-oriented (no streams, no exit codes).
- **[ShellKeeper](https://github.com/tranhuucanh/mcp-shellkeeper)** ([HN thread](https://news.ycombinator.com)) —
  persistent SSH sessions + file transfer; motivated by exactly this pain ("each SSH command is
  stateless"). Smaller/younger project.

### Local process/session managers
- **[DesktopCommander](https://github.com/wonderwhy-er/DesktopCommander)** — the canonical local API
  shape: `execute_command` (timeout, background mode) + `read_output(pid/session, offset, length)`
  cursor pagination + process list/kill. Local only, no ssh, no fleet.

### Framework precedents worth stealing from
- **ZCode built-in Bash tool** (this harness): sync + `run_in_background` (keeps running across
  turns) + `TaskOutput` polling + **automatic task-notification on finish, which re-invokes the
  agent**. That is the one event channel verified to work in ZCode — but it exists only for the
  built-in Bash tool, and only locally. An MCP tool cannot reproduce it directly (see §3.1) — but a
  CLI + background-Bash composition can (§4.5).
- **SWE-agent's ACI** ([paper](https://arxiv.org), [walkthrough](https://dev.to)): persistent bash
  session with the contract *"send command → stdout + exit code"*. Their published finding: interface
  design, not model choice, tripled task success. Exit codes in a PTY stream are done via
  **[OSC 133](https://contour-terminal.org/docs/specs/shell-integration/)** shell integration
  (`133;D;<exitcode>` marks command end) — but note tmux strips/passthrough of OSC 133 is
  unreliable ([tmux issue](https://github.com/tmux/tmux/issues)), so robust tools embed a plain-text
  sentinel (`@@EXIT:$?@@`) instead.
- **Ansible async** (`async:` + `async_status:` poll / fire-and-forget): the job-id + poll API shape,
  15 years battle-tested.
- **[assh](https://github.com/MerlinDMC/advanced-ssh-config)** (advanced-ssh-config): per-host
  **ordered gateway fallback lists** — try gw1, fail over to gw2. In stock OpenSSH the same effect:
  `ProxyCommand sh -c 'nc -w3 %h %p || ssh -W %h:%p gw2'`, `ProxyJump a,b` + `ConnectTimeout`.
- **OpenSSH itself**: `ControlMaster=auto` + `ControlPersist` gives connection reuse (fast repeat
  execs, fewer auth round-trips on flaky links); `ssh -G` resolves an alias's effective config.

## 3. Hard facts that constrain the design

1. **Don't lean on MCP protocol push.** The protocol has `notifications/message` and
   `resources/subscribe`; Claude Code — the reference MCP client — demonstrably *receives and does
   not display* them ([issue #3174](https://github.com/anthropics/claude-code/issues/3174)).
   ZCode's display behavior is unverified; either way, a notification landing between turns can't
   drive the agent — the channel that verifiably re-invokes a ZCode agent is **background-task
   completion** (§4.5). → The in-MCP "on event" half must be a **blocking `wait` tool**
   (until=exit|regex|idle, with timeout).
2. **PTY sessions merge stdout/stderr and hide exit codes.** Two backends are needed: batch jobs
   (exec channel or `nohup` + per-stream files) for clean streams/codes, and a tmux/tty session for
   interactivity + env persistence. Exit code in tty mode = command-wrapping sentinel, not prompt
   guessing.
3. **Disconnect survival must live on the remote host.** Any process run over an SSH channel —
   sync or async — dies with the channel (sshd SIGHUPs it when the link drops). None of the
   surveyed tools survive a mid-command link loss: mcp-ssh-session's "async" still runs over its
   own persistent channel, and mcp-ssh-tmux's remote commands die with the ssh process inside its
   *local* tmux pane. Survival = target-side artifacts only (`nohup setsid` job dirs or tmux *on
   the target*), never anything "held open by the local server". On a fleet with flaky paths
   (overlay networks, multi-hop relays) this is the difference between a working tool and a lie.
4. **On unstable links the request itself is unreliable state.** A `start` whose connection dies
   en route leaves "did it run?" ambiguous → client-generated job IDs make starts idempotent (the
   standard request-id idempotency pattern), and byte-cursor reads make reconnects exactly-resumable.
5. **Don't reimplement transport.** System `ssh` gives keys, agent, known_hosts, ProxyJump chains,
   ControlMaster reuse — all free. paramiko/asyncssh means re-deriving all of it (mcp-ssh-session's
   fragile prompt-polling is the cautionary tale).

## 4. Recommended architecture: `bashd`

One small Python program with two faces (MCP stdio server + CLI over the same core). Nothing
off-the-shelf combines R1–R7 (nearest misses: §2), and the missing pieces are exactly a thin
shell around `ssh` + `tmux` — est. 600–900 lines, no heavy deps (`mcp` SDK or hand-rolled
stdio JSON-RPC; system ssh + tmux only).

**Governing principle (rev 2): the target host owns all state; the connection is a cache.** Every
mode is backed by target-side artifacts (`~/.bashd/jobs/<id>/` files or a target-side tmux
session), so a link loss at any instant — including mid-`run`, mid-wait, or during the start
request itself — loses nothing. The local server holds no authoritative state; every call is
reconnect-and-resume.

### 4.1 Two substrates, three modes

Only two execution substrates exist, and both live entirely on the target host. Sync is UX sugar
over the job substrate, not a third mechanism — so survival is structural, not per-mode work:

| Mode | Mechanism | Gives |
|------|-----------|-------|
| `sync` (default UX) | idempotent job start, then a single resumable tail-exec that streams the output files until the exit-code file appears | clean stdout/stderr + real exit code; **timeout detaches (returns `{id, running}`), never kills**; a dropped link mid-wait is internally reconnected and the tail resumes from cursor |
| `job` (async batch) | `setsid nohup sh run.sh > out 2> err; echo $? > code` in `~/.bashd/jobs/<id>/` **on the target**; `<id>` generated client-side → start is idempotent | survives any disconnect and agent/server restarts; cursor reads = ranged file reads; survives target reboots too (files persist) |
| `tty` (async session) | tmux **on the target** (isolated socket `tmux -L bashd`) + `pipe-pane -a` appending to a log file | screen/tmux persistence (env/cwd/venv/REPLs); `send-keys` stdin; output = live `capture-pane` *or* tail the pipe-pane log while disconnected — the same cursor API as jobs, just a different file; exit codes via per-command sentinel (`; printf '@@EXIT:%s@@\n' $?`); does **not** survive a target reboot (tmux dies; job dirs do) |

State (`~/.bashd/` local + `~/.bashd/` on every target) is just files + tmux → server restarts
*and connection losses* lose nothing; `bashd sessions` rebuilds by scanning.

### 4.2 Host registry (`~/.bashd/hosts.yaml`)

```yaml
hosts:
  local: {kind: local}
  web:
    ssh: deploy@203.0.113.10
    paths: ["direct", "ssh -J jump@203.0.113.7"]   # ordered fallback, emitted as ProxyCommand
    defaults: {timeout_s: 300, cwd: /opt}
    notes: "prod web node — mind the maintenance window"
  builder:
    ssh: ci@198.51.100.20
    connect: {timeout_s: 5, keepalive_s: 15, keepalive_max: 3}
    notes: "company CI box — don't touch CI runners"
```

Resolved through `ssh -G` semantics; the agent only ever sees aliases + notes. Per-host notes get
surfaced in the `sessions`/`run` responses — fleet gotchas ("DO NOT restart the VPN daemon over
a session that terminates on its tunnel address") become first-class metadata the agent can't forget. Multi-path fallback is emitted as a generated
ProxyCommand (assh's trick, zero custom code). For unstable networks: paths are tried in order per
attempt with a short `ConnectTimeout`; the last-good path stays sticky until it fails; `ServerAlive`
keepalives detect dead peers fast; and every response reports which path served it, so the agent
sees path quality instead of guessing.

### 4.3 Tool surface (7 tools)

1. `run(host, cmd, timeout_s=120)` → streams until exit: `{stdout, stderr, exit_code, duration}`;
   on timeout or unrecoverable link loss it **leaves the job running** and returns
   `{id, status: running, partial output}` — detach, never kill (R7)
2. `start(host, cmd, mode=batch|tty, name?, cwd?)` → `{id}`; **idempotent**: `<id>` is generated
   client-side, so re-issuing a start after a dropped request is a no-op, never a double-run
3. `read(id, stream=out|err|both, cursor=0, max_bytes=16k)` → `{chunk, next_cursor, status, exit_code}`;
   resumable by construction — reconnect + ranged read (`tail -c +cursor`), one round trip returns
   output + status + exit code together
4. `wait(id, timeout_s, until=exit|regex|idle, pattern?, idle_s=2)` → blocks until event or
   timeout; **reconnect-tolerant** (drops mid-wait are retried with backoff inside the same call);
   a host unreachable at call entry fails fast with `{unreachable, tried_paths}` unless
   `until_reachable: true`
5. `write(id, data)` → stdin into tty sessions (prompts, REPLs)
6. `signal(id, sig)` → TERM/KILL/Ctrl+C (explicit kill is the only way a job dies early)
7. `sessions(host?)` → list with status, age, sizes, host notes, last-good path; probes all hosts
   in parallel with fail-fast when called fleet-wide

### 4.4 Exit-code capture in tty mode

Wrap every sent command: `<cmd>; printf '@@EXIT:%s@@\n' $?`. Parse the sentinel from the
pipe-pane log / capture output. Don't rely on prompt regexes or OSC passthrough (§3.2). The job
substrate needs none of this — it reads the real exit code from its code file.

### 4.5 Events that actually fire in ZCode

- In-MCP: `wait` (§3.1) is the deliverable primitive.
- ZCode-native push: ship the **same core as a CLI** (`bashd wait --id X --timeout 600`). The agent
  runs it via the built-in Bash tool with `run_in_background=true` → the task keeps running across
  turns and its **completion re-invokes the agent with a task-notification** — a verified, native
  event path. This is the reliable way to get "wake the agent when the remote build finishes" here.
- ZCode hooks (tool-call interception) exist as an extra integration point (audit/allowlisting)
  but are not needed for the event path.

### 4.6 Unstable-network discipline

- **Idempotent starts.** Client-generated job IDs turn "did my start get through?" from a
  correctness problem into a retry (`start` with the same id is a no-op) — the standard
  request-id idempotency pattern.
- **Detach, don't kill, on timeout.** A timeout means *we* stopped watching, not that the work
  failed; killing is always an explicit `signal`. Sync auto-transitions to async instead of
  orphaning or destroying the job.
- **Fail fast, retry deliberately.** Unreachable host → structured error (`tried_paths`) in ~
  `paths × timeout_s`, not a hidden hang; `wait(until_reachable)` is the only long-blocking form.
- **Everything resumes by byte cursor.** Reads are ranged reads of append-only target-side files
  (`tail -c +N`); reconnects replay from the last confirmed cursor — at-least-once delivery,
  exactly-once effect, no duplication, no gaps.
- **Minimize round trips.** Every call is ≤ 1–2 ssh round trips regardless of job duration (the
  sync "wait" is one long-lived tail-exec, not a poll loop over the wire); output + status + exit
  code ride back in one response. ControlMaster keeps handshakes off the hot path, and a dead
  master is a non-event — nothing anchors to it.
- **No quoting hell, no clock trust.** Per-job `run.sh` is written via heredoc/base64 (no multi-hop
  shell quoting); ordering is by bytes/sequence, never by remote timestamps.
- **Output caps per job** (with a truncation marker file) so a runaway job can't fill the target
  disk while it sits unwatched behind a dead link.
- **Optional `transport: mosh`** per host for tty sessions on roaming paths (UDP, survives IP
  change wg↔ygg, local echo); ssh remains the default and the only transport for jobs (mosh has no
  exec channel).
- **Cost-aware routes with auto re-latch (shipped v0.3.0).** Field experience (2026-09-24/26:
  a fleet host reachable only through its second-choice relay while transit flapped) exposed the
  last routing gap: the sticky latch has no memory of *preference*, only of *recency* — a
  recovered cheap path was never promoted back. Fix: per-path `cost` (default = config position),
  sticky-first then cheapest-first ordering, and a detached background probe spawned after any
  successful call on a non-cheapest path; it tests the cheaper route (exponential backoff
  30 s → 15 min, state in `~/.bashd/state.json`) and re-latches it on recovery. Probing never
  blocks a call; manual forms: `bashd paths [--repath]`, `bash_hosts {repath:true}`. The probe
  child re-execs the same `bashd` file (`_repath-probe <host>`) with `BASHD_NOPROBE=1` so it
  cannot recurse.

### 4.7 Security posture

Same trust model as the Bash tool (the agent is trusted with the hosts). Aliases only in tool calls;
no credentials in context or logs; output caps (ring buffer + max_bytes per read) everywhere;
timeouts default-on; optional per-host `allow:` command patterns. Do **not** auto-enable agent
forwarding to untrusted hosts.

## 5. Verdict

- **Build vs adopt:** adopt if one of §2 fits — they don't, and rev 2 adds a disqualifier: *none*
  of them survive a mid-command link loss (their async still anchors to a live channel).
  terminal-control-mcp lacks remote+exit codes; mcp-ssh-session lacks stream separation/reliable
  completion and dies with its persistent channel; mcp-ssh-tmux is snapshot-style without a real
  async API; DesktopCommander is local-only. The glue that's missing (registry + fallback paths +
  wait-event + cursors + target-side substrates) is small and well-understood → build `bashd` thin.
- **The two load-bearing decisions:** (1) *all* state and *all* execution live on the target host —
  sync is sugar over the same job substrate (idempotent start + detach-on-timeout + resumable
  tail), so disconnect survival is structural, not per-mode; in-MCP events = blocking `wait`
  (protocol push is unreliable across clients), agent-level push = CLI `wait` run via background
  Bash (ZCode-native notification).
- **Second:** drive system `ssh` (ControlMaster + generated ProxyCommand fallbacks); never paramiko.
