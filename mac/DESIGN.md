# ocu-lane — macOS design note

What the three upstream projects do, what was taken from each, and what was left
on the floor. The stack itself is `ocul.py` (one file, one rotator, one proxy) plus
the keeper, the lane wrapper and the HUD.

## The three upstreams

### ShadowySphinX / Opencode-Unlimited-Limit-Bypass- (Windows + VPS relay, v4.0)

The most engineered of the three. 20k lines across a Python `src/`, a `.exe`
bundled UI, systemd/PowerShell/Task Scheduler installers, and a server-side relay.

| Piece | What it does | Verdict for macOS |
|---|---|---|
| `zen_proxy.py` — pinned edge + source bind | Binds every OpenCode connection to the WARP IPv6 and pins a validated edge. Falls over to another edge rather than stalling. | **Adopted.** This is the right shape and we already had most of it. |
| stale-bind repair (`EADDRNOTAVAIL` → wait for the new address → rebind) | A rotation removes the local address; connections in that window bind a dead source. | **Adopted.** Was our worst live failure mode during a hop. |
| `stream_reset` accounting + edge retirement | A reply cut after its first token is not a connect failure; two inside five minutes retires the edge. | **Adopted.** Nothing else measures this and it is exactly what a user sees as "the answer just stopped". |
| strict fail-closed (`OUC_STRICT_PROXY`) | Never silently uses a weaker route for quota traffic. | Already ours — `REFUSED … (fail-closed)` predates this work. |
| Tor lane, `tor_manager.py` | Rotates through Tor circuits. | **Dropped.** Tor exits are refused by the upstream outright; it buys a 403 and a slower one. |
| `relay/blockrelay.py` + `warpgate_pool.py` | Server-side lane that manufactures fresh `/64`s from a routed prefix or a pool of WARP registrations. | **Dropped.** Needs a VPS, a routed prefix and `ip_nonlocal_bind`. Compelling if the WARP pool ever runs dry; out of scope for one Mac. |
| `account_pool.py`, `desktop_brand.py`, `avatar_ui.py` | Account switching, rebranded desktop shell, 2.3k-line avatar UI. | **Dropped.** macOS has no OpenCode desktop shell to rebrand. |

### ElectronBreach / opencode-unlimited-usage (Linux, the ancestor)

Our own lineage — `warp-keeper.sh`, `zen-guard.sh`, `zen-lane.sh`, the kill-switch
firewall. Nothing new to take: the mac port already carries the lane, the edge pin,
the keeper and the blacklist, and replaced the `iptables` kill switch with a
fail-closed proxy (`ALLOW_DIRECT=0`) because macOS cannot drop its own traffic
without a root packet filter.

### jekooo / opencode-infinite (Linux shell + Windows `.bat`, plus an OpenCode fork)

A 1.6k-line bash launcher wrapped around a full OpenCode app fork, with a
config table, an i18n menu and a rotation policy. Two things in it are better than
anything we had:

| Piece | What it does | Verdict |
|---|---|---|
| three-way line classifier (`noop` / `rotate` / `ask`) | Compaction, context-length, model-not-found and content-policy are *not* quota. A bare 429 is a transient throttle, not a spent allowance. Only free-usage/quota is a wall. | **Adopted** — rewritten as `classify_wall()`. |
| `ask.last` cooldown (600 s) | One logged decision per ten minutes, so a retry burst does not become five spent `/64`s. | **Adopted** as `ask_allowed()` / `ask_note()`. |
| probe off by default | Every probe message is a free-quota message. | Already ours: `probe_enabled(reason)` arms walls only. |
| star topology of settings, Tor/proxy-list lanes, `TARGET_COUNTRY` hunting | Config table, free-proxy hunting, country targeting. | **Dropped.** No proxy lane on this machine by choice. |
| OpenCode app fork (11k files) | Ship a modified client. | **Dropped.** We use the stock CLI. |

## What changed in this pass

1. **Wall classification** — `WALL_RE` matched `rate limit exceeded|too many
   requests`, so a transient 429 rotated a full `/64`. Now:

   ```
   noop   compaction | context length | maximum context | model not found |
          content filter                → record, never rotate
   rotate free usage | quota exceeded | usage limit | out of usage |
          insufficient credits | insufficient balance | payment required |
          ip blocked | 403 region       → rotate
   ask    rate limit | too many requests | 429        → log once / 10 min, no rotation
   ```

   `noop` is checked first, because `Failed to drain Session` is a compaction
   wrapper whose cause can carry the wall words. Verified against all 172
   historical `detected` lines in `walls.jsonl`: 172/172 classify as `rotate`.

2. **Stale-bind repair** — `_dial()` re-reads the tunnel address and rebinds once
   instead of failing every request for the length of a hop. Measured: bogus
   source → repair → connected in 0.1 s.

3. **Stream-reset accounting** — `relay()` now returns `(read, write, moved)`.
   A reset on the upstream→client direction after bytes moved is a cut reply, not
   a connect failure. Two inside 300 s retires the edge for 600 s in `edge.json`,
   which survives a proxy restart.

4. **Surfaced, not swallowed** — proxy health carries `stream_reset` and
   `repairs`; the brain publishes them; `ocul doctor` prints
   `proxy recovery: N stream resets, M address repairs` and the health score
   warns when resets exceed 2 % of traffic.

5. **Ignored walls are logged** — `walls.jsonl` gains `wall-ignored` with the
   reason, deduped per 10 minutes. A decision *not* to rotate is now auditable;
   before, a rate-limit storm was indistinguishable from a quiet guard.

## Deliberately not built

- A second rotator. One lock, one owner, one process writing the tunnel.
- A live-stream hold before a rotation. A wall means the stream is already dead,
  and a user hop is an instruction, not a guess.
- WARP protocol/endpoint switching (UDP 500/4500/1701) for ISP-filtered
  networks. Real, but it is a knob that can break a working tunnel, and this Mac
  is not behind one.
- Any proxy. Chosen, twice.
