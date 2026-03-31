# OpenClaw Workarounds — srv1318115

> Documented on 2026-03-31. Host: srv1318115 (Ubuntu, Node.js runtime, OpenClaw latest).
> All patches are manual edits to compiled `dist/` bundles. They will be lost on every
> OpenClaw update. This document is a reference for the OpenClaw team to port these into
> source code.

---

## 1. Gateway Probe False Timeout on Healthy Loopback

**Symptom:** `openclaw gateway probe` shows `Reachable: no` / `Connect: failed - timeout`
on a gateway that is healthy and responding.

**Root cause:** The internal probe budget for loopback was capped at 800ms, while the CLI
reported a 3000ms budget to the user, creating a mismatch. Healthy gateways on slightly
slower systems or under load triggered false timeouts.

**Files affected:** `dist/gateway-cli-*.js`, `dist/auth-profiles-*.js`

### Fix — `gateway-cli-*.js`

```js
// 1. localLoopback probe budget (was Math.min(800, overallMs))
Math.max(4000, Math.min(overallMs, 8000))

// 2. sshTunnel budget: 2000ms → 5000ms
// 3. Default remote budget: 1500ms → 4000ms
// 4. CLI probe timeout: 3000ms → 10000ms
// 5. parseTimeoutMs: 3000ms → 10000ms
```

### Fix — `auth-profiles-*.js`

```js
// 1. probeGatewayReachable: 1500ms → 5000ms
// 2. waitForGatewayReachable (probeTimeoutMs): 1500ms → 5000ms
```

**Result:** `Reachable: yes` / `Connect: ok (29ms) · RPC: ok`

**Related issues:** #46226

---

## 2. Discord Thread Context bloat + CLI Routing

**Symptom:** Discord thread sessions grew unboundedly because full inbound metadata was
re-injected on every message. After ~20 messages, context exceeded 100K tokens. CLI
commands also routed to the wrong session.

**Files affected:** `dist/plugins/discord-*.js`, `dist/gateway-cli-*.js`

### Fix

1. Strip full historical message list from thread re-injection; keep only the last 5
   messages or a summary vector.
2. Add a `routingTag` / `sessionSelector` so `--channel discord --to` routes to the
   correct thread session.

**Related issues:** #44586, #44584, #44447, #44449, #44453

---

## 3. Premature Success Wording in Subagent Flows

**Symptom:** Subagent agents reported "done" immediately after child session ended,
before parent received and verified the output. Caused race conditions.

**Files affected:** `dist/subagent-*.js`, `dist/gateway-cli-*.js`

### Fix

Change wording from "done" / "completed successfully" to "**available for review** —
waiting for parent to verify output." Parent Q-gate must verify external state changes
before declaring success.

**Related issues:** #44587, #44471, #44472

---

## 4. ACP Thread-Bound Spawn Error

**Symptom:** Spawning an ACP session with `thread: true` failed with a `spawnedBy`
validation error.

**Root cause:** `spawnedBy` validation checked for exact agent ID match, but the parent
session's ID was formatted with a workspace prefix that the child didn't expect.

**Fix:** Normalize `spawnedBy` to compare raw IDs without workspace prefix, or relax
exact-match validation to a compatibility check.

---

## 5. Performance Degradation — Memory Leak + Probe Latency

**Symptom:** Gateway memory climbed to 577MB+ RSS over time. Probe latency reached 4.5s.
Gateway eventually became unresponsive.

**Root cause:** Session maintenance was not aggressive enough. Node heap had no cap.

**Fix:**

1. Cap Node heap: `--max-old-space-size=1024` in gateway start command
2. Add systemd memory limit: `MemoryMax=1500M`
3. Increase session cleanup frequency (reduce TTL or add max-session cap)
4. Add periodic probe latency monitoring; alert if >2s

**Files affected:** `systemd service file`, `dist/gateway-*.js`

**Related issues:** #44582

---

## 6. Cron Payload Kind False-Positive

**Symptom:** `normalizePayloadKind()` flagged valid cron payloads as legacy, causing
re-submission or ignore.

**Root cause:** Legacy marker existed in both old and new payload formats.

**Fix:** Distinguish formats using a version field or structural difference, not just
the legacy marker.

**Related issues:** #43796

---

## 7. Discord Streaming: Reply Text Loss + maxTokens Truncation

**Symptom:** With `streaming: "partial"`, reply text was occasionally lost.
`maxTokens` silently truncated at 8192 even when model supported more.

**Fix:**

1. Set Discord streaming mode from `"partial"` to `"off"`
2. Adjust `maxTokens` per model tier (MiniMax 2.7: 32,768)
3. Add hard floor cap at `ctx.maxTokens || 8192`

**Files affected:** `dist/plugins/discord-*.js`, `dist/gateway-cli-*.js`

**Related issues:** #44586

---

## 8. trustedProxies Not Set

**Symptom:** Nginx reverse proxy warnings about untrusted headers (`X-Forwarded-For`,
`X-Real-IP`).

**Fix:** Add to gateway config:

```json
{
  "gateway": {
    "trustedProxies": ["127.0.0.1", "::1", "172.17.0.1"]
  }
}
```

---

## 9. Session Idle Timeout Too Short

**Symptom:** Discord sessions timed out after 60 minutes during long operations.

**Fix:**

```json
{
  "channels": {
    "discord": {
      "sessionIdleTimeoutMs": 14400000
    }
  }
}
```

---

## 10. Raider Long-Session Degradation

**Symptom:** After >1 hour sessions, Raider experienced server errors and `SIGTERM`
aborts.

**Fix (Patch A runbook):**
1. `contextBudget: 60000`
2. `execTimeout: 120000`
3. On `server_error` / `Request was aborted`: compact → retry 1–2 turns → `/new`

---

## 11. Kimi CLI 401 Errors

**Symptom:** `kimi-cli` returned 401 despite valid API key.

**Fix:** Add auth profile to `~/.openclaw/providers/kimi.json`:

```json
{
  "provider": "kimi",
  "apiKey": "YOUR_KEY",
  "authType": "bearer"
}
```

---

## 12. Codex Thinking Catalog Misalignment

**Symptom:** Codex reasoning produced erratic behavior, ignoring system-level thinking
settings.

**Fix:** Ensure `thinking: "off"/"on"` correctly maps to Codex `--thinking` flag.
Add catalog sync step on startup.

---

## 13. Writer 401 with gpt-5.3-codex

**Symptom:** Writer agent (gpt-5.3-codex) returned 401 while same model worked elsewhere.

**Root cause:** Provider/transport mismatch — Writer used a different transport with
different auth configuration.

**Fix:** Align Writer transport with main agent transport; ensure same
`OPENAI_API_BASE` and auth headers.

---

## Summary

| # | Issue | File(s) | Type |
|---|-------|---------|------|
| 1 | Gateway probe false timeout | `gateway-cli-*.js`, `auth-profiles-*.js` | Bug |
| 2 | Discord thread context bloat + CLI routing | `discord-*.js`, `gateway-cli-*.js` | Bug |
| 3 | Premature success wording | `subagent-*.js`, `gateway-cli-*.js` | UX |
| 4 | ACP spawn spawnedBy validation | `acp-*.js` | Bug |
| 5 | Memory leak + probe latency | `gateway-*.js`, systemd | Performance |
| 6 | Cron payload kind false-positive | `cron-*.js` | Bug |
| 7 | Discord streaming text loss + maxTokens | `discord-*.js`, `gateway-cli-*.js` | Bug |
| 8 | trustedProxies not set | `gateway.json` | Config |
| 9 | Session idle timeout too short | `gateway-*.js` | Config |
| 10 | Raider long-session degradation | `gateway-*.js`, `openclaw.json` | Bug |
| 11 | Kimi CLI 401 errors | provider config | Config |
| 12 | Codex thinking catalog misalignment | Codex runtime | Bug |
| 13 | Writer 401 with gpt-5.3-codex | provider config | Bug |

**All fixes are on compiled bundles. They will NOT survive `openclaw update`.**
