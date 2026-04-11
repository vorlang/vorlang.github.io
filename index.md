# Vor

**Verified, observable, chaos-tested distributed systems on the BEAM**

*Named for the Norse goddess who witnesses oaths. Vor programs are oaths about system behavior — declared, witnessed by the compiler, and enforced.*

[GitHub](https://github.com/vorlang/vor)

---

## What it does

Write one file. The compiler produces a verified, instrumented, chaos-testable BEAM binary.

```
mix compile        →  proves safety properties           (milliseconds)
mix vor.check      →  model-checks message interleavings (seconds)
mix vor.simulate   →  chaos-tests real BEAM processes    (minutes)
```

The compiled binary is pre-instrumented with telemetry. No separate spec. No separate chaos infrastructure. No instrumentation code. No Docker. No Kubernetes.

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

Wire agents together and `mix vor.check` proves distributed invariants:

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
✓ Proven (1001 states, depth 10)
```

---

## Chaos simulation

```bash
mix vor.simulate --partition --delay --workload 10
```

Starts real BEAM processes. Injects real failures. Checks invariants against live state:

- Kill agents randomly, let supervisors restart them
- Partition connections via proxy processes
- Delay messages for configurable durations
- Generate client workload from protocol declarations
- Check invariants every second
- Replay any run with `--seed N`

No Chaos Monkey. No Toxiproxy. No Docker. The BEAM provides all the failure injection primitives as function calls.

---

## Auto-generated telemetry

Every compiled Vor agent is observable by default. Zero instrumentation code.

| Event | Fires on |
|---|---|
| `[:vor, :agent, :start]` | Agent initialization |
| `[:vor, :message, :received]` | Handler invocation |
| `[:vor, :transition]` | State field change (sensitive fields redacted) |
| `[:vor, :message, :emitted]` | Reply sent |
| `[:vor, :constraint, :violated]` | Protocol constraint rejection |

Attach any `:telemetry` backend and every agent is visible. The compiler generated the telemetry because it knows the program's complete behavioral structure.

---

## What Vor eliminates

Compared to a typical Java/Go/Kubernetes stack, Vor + BEAM eliminates:

- Separate design specification (the spec is the program)
- External design verification tooling (TLA+ for small protocols)
- External chaos testing infrastructure (Chaos Monkey, Litmus, Toxiproxy)
- External contract tests (Pact — protocol checked at compile time)
- Docker containers (no container OS, no additional CVEs)
- Kubernetes orchestration (OTP supervision + libcluster)
- Container registry
- Service mesh (BEAM message passing)
- Inter-service load balancers (direct process-to-process messaging)
- External message queue (BEAM mailboxes replace Kafka/RabbitMQ)
- Serialization between services (native BEAM terms)
- Build tool complexity (mix replaces Maven/Gradle)
- Manual concurrency primitives (no locks, semaphores, thread pools)
- Telemetry instrumentation code (compiler-generated)
- External input validation framework (protocol `where` constraints)
- Rolling deploy infrastructure (BEAM hot code reload)
- Helm charts (no Kubernetes)

What you still need: infrastructure provisioning (Terraform), reverse proxy for external HTTP (nginx/Caddy), telemetry backend (Prometheus/Grafana), CI/CD pipeline, load testing, secrets management.

---

## How it fits with Gleam

Vor is designed as a two-language stack:

- **Vor** — coordination logic: state machines, protocols, invariants, model checking, chaos, telemetry
- **Gleam** — data processing when needed: type-safe transformations called through validated `extern gleam` blocks

All five examples are fully native Vor. Gleam is there when you need typed library functions for complex data operations.

---

## Design principles

**The spec is the program.** No separate specification. No drift.

**Guarantee tiers are explicit.** `proven` at compile time, `monitored` at runtime. The compiler fails closed.

**Observable by default.** The compiler generates telemetry from the program's structure. No instrumentation code.

**Input validated at the protocol level.** `where` constraints reject invalid messages before handler code runs.

**Sensitive data declared.** Fields marked `sensitive` are redacted in telemetry automatically.

**Failure is first-class.** Resilience handlers define recovery. The compiler verifies recovery paths.

**The BEAM is the foundation.** Thirty years of production-proven concurrency, fault tolerance, and distribution underneath.

---

394+ tests  ·  9 property-based test suites  ·  Raft proven in 1,001 states  ·  MIT License
