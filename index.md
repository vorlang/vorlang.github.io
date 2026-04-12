# Vor

**Verified state machines, protocol checking, chaos testing, and compiler-generated telemetry for the BEAM**

*Named for the Norse goddess who witnesses oaths. Vor programs are oaths about system behavior — declared, witnessed by the compiler, and enforced.*

[GitHub](https://github.com/vorlang/vor)

---

## What it does

The BEAM already provides process isolation, supervision, and distributed message passing — eliminating entire classes of bugs. Vor adds the layer the BEAM doesn't: verification that your state machines are correct, your protocols are compatible, and your system recovers from failures.

One source file declares agents, protocols, invariants, and constraints. Two commands verify it, one stress-tests it:

```
mix compile        →  proves safety properties              (milliseconds)
mix vor.check      →  bounded-verifies multi-agent invariants  (seconds)
mix vor.simulate   →  chaos-tests real BEAM processes        (minutes)
```

The compiled binary is a standard OTP process, pre-instrumented with telemetry — no separate spec, no instrumentation code.

---

## What it looks like

```vor
agent LockManager(lock_timeout_ms: integer) do
  state phase: :free | :held
  state holder: atom
  state wait_queue: list
  state auth_token: binary sensitive

  protocol do
    accepts {:acquire, client: atom, priority: integer} where priority >= 1 and priority <= 10
    accepts {:release, client: atom}
    emits {:grant, client: atom}
    emits {:queued, position: integer}
    emits {:ok}
  end

  on {:acquire, client: C, priority: P} when phase == :free do
    transition phase: :held
    transition holder: C
    emit {:grant, client: C}
  end

  on {:acquire, client: C, priority: P} when phase == :held do
    transition wait_queue: list_append(wait_queue, C)
    qlen = list_length(wait_queue)
    emit {:queued, position: qlen}
  end

  on {:release, client: C} when phase == :held do
    if list_empty(wait_queue) == :true do
      transition phase: :free
      transition holder: :nil
      emit {:ok}
    else
      next_client = list_head(wait_queue)
      transition wait_queue: list_tail(wait_queue)
      transition holder: next_client
      emit {:ok}
    end
  end

  safety "no grant when held" proven do
    never(phase == :held and emitted({:grant, _}))
  end

  liveness "lock released eventually" monitored(within: lock_timeout_ms) do
    always(phase == :held implies eventually(phase != :held))
  end

  resilience do
    on_invariant_violation("lock released eventually") ->
      transition phase: :free
      transition holder: :nil
  end
end
```

The `safety` invariant is proven at compile time. The `where` constraint rejects invalid input before handlers run. The `sensitive` field is redacted in telemetry. The `liveness` invariant is monitored at runtime with automatic recovery. Every state transition and message generates telemetry. This compiles to a standard OTP gen_statem.

---

## Multi-agent model checking

Wire agents together and `mix vor.check` bounded-verifies distributed invariants by exploring all message interleavings within configured bounds:

```vor
system RaftCluster do
  agent :n1, RaftNode(node_id: :n1, cluster_size: 3)
  agent :n2, RaftNode(node_id: :n2, cluster_size: 3)
  agent :n3, RaftNode(node_id: :n3, cluster_size: 3)

  connect :n1 -> :n2
  connect :n1 -> :n3
  connect :n2 -> :n1
  connect :n2 -> :n3
  connect :n3 -> :n1
  connect :n3 -> :n2

  safety "at most one leader" proven do
    never(count(agents where role == :leader) > 1)
  end
end
```

```
Tracked fields:    current_term, role, vote_count
Abstracted fields: commit_index, log, voted_for
Symmetry:          enabled (3 identical agents, 6× reduction)
✓ Bounded-verified (1001 states, depth 10)
```

---

## Chaos testing

Verification proves properties within bounds on a model. Chaos testing complements it by exercising real compiled code under failure — catching implementation bugs, timing issues, and recovery failures the model checker can't reach.

```bash
mix vor.simulate --partition --delay --workload 10
```

Starts real BEAM processes, injects real failures, checks invariants against live state:

- Kill agents randomly, let supervisors restart them
- Partition connections via proxy processes
- Delay messages for configurable durations
- Generate client workload from protocol declarations
- Check invariants periodically against live state
- Replay any run with `--seed N`

For BEAM-native systems, no external chaos infrastructure is needed — the BEAM provides failure injection as function calls.

---

## Auto-generated telemetry

The compiler knows every state field, message type, and transition. It generates telemetry calls in the compiled bytecode — no instrumentation code in the source file.

| Event | Fires on |
|---|---|
| `[:vor, :agent, :start]` | Agent initialization |
| `[:vor, :message, :received]` | Handler invocation |
| `[:vor, :transition]` | State field change (sensitive fields redacted) |
| `[:vor, :message, :emitted]` | Reply sent |
| `[:vor, :constraint, :violated]` | Protocol constraint rejection |

Attach any `:telemetry` backend and every agent is observable. You still need a metrics backend (Prometheus, Grafana) to view the data — Vor generates the events.

---

## What the approach simplifies

For BEAM-native actor systems, Vor addresses several concerns that typically require separate tools: formal verification replaces a separate specification language, chaos testing uses BEAM-native primitives instead of external infrastructure, protocol composition checking at compile time replaces contract testing, compiler-generated telemetry replaces manual instrumentation, and protocol constraints replace validation libraries.

These simplifications apply to BEAM-native systems. Infrastructure-level concerns — provisioning, external HTTP routing, metrics backends, CI/CD, load testing, secrets management — remain outside the language.

---

## How it fits with Gleam

Vor is designed as a two-language stack:

- **Vor** — coordination logic: state machines, protocols, invariants, verification, chaos testing, telemetry
- **Gleam** — data processing when needed: type-safe transformations called through validated `extern gleam` blocks

All five examples are fully native Vor. Gleam is there when you need typed library functions for complex data operations.

---

## Design principles

**The spec is the program.** No separate specification. No drift.

**Guarantee tiers are explicit.** `proven` at compile time, `monitored` at runtime. The compiler fails closed — it never claims to verify what it can't.

**Observable by default.** The compiler generates telemetry from the program's structure.

**Input validated at the protocol level.** `where` constraints reject invalid messages before handler code runs.

**Sensitive data declared.** Fields marked `sensitive` are redacted in telemetry.

**Failure is first-class.** Resilience handlers define recovery. Recovery paths are verified.

**The BEAM is the foundation.** Thirty years of production-proven concurrency, fault tolerance, and distribution.

---

394+ tests  ·  9 property-based test suites  ·  Raft bounded-verified in 1,001 states  ·  MIT License
