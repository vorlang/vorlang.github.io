# Vor

**Verified state machines and protocols for the BEAM**

*Named for the Norse goddess who witnesses oaths. Vor programs are oaths about system behavior — declared, witnessed by the compiler, and enforced.*

[GitHub](https://github.com/vorlang/vor)

---

## What it looks like

A distributed lock with a proven safety invariant:

```vor
agent LockManager(lock_timeout_ms: integer) do
  state phase: :free | :held
  state holder: atom
  state wait_queue: list

  protocol do
    accepts {:acquire, client: atom}
    accepts {:release, client: atom}
    emits {:grant, client: atom}
    emits {:queued, position: integer}
    emits {:ok}
  end

  on {:acquire, client: C} when phase == :free do
    transition phase: :held
    transition holder: C
    emit {:grant, client: C}
  end

  on {:acquire, client: C} when phase == :held do
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

The `safety` invariant is proven at compile time — the compiler exhaustively walks every state × emit combination and rejects the program if the property can be violated. The `liveness` invariant is monitored at runtime — if the lock is held longer than the timeout, the resilience handler fires and auto-recovers.

This compiles to a gen_statem. Start it like any OTP process:

```elixir
{:ok, pid} = :gen_statem.start_link(LockManager, [lock_timeout_ms: 5000], [])
:gen_statem.call(pid, {:acquire, %{client: :alice}})
# => {:grant, %{client: :alice}}
```

---

## Multi-agent model checking

Vor doesn't just verify individual agents — it model-checks entire systems. Wire agents together in a system block with a system-level invariant, and `mix vor.check` explores all reachable combined states through all possible message interleavings:

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

The explorer constructs the product state space (all agents' states × all pending messages), systematically delivers messages in every possible order, and checks the invariant at every reachable state. If any message interleaving can produce two leaders, the compiler reports the exact counterexample trace — which message arrived at which agent in which order to cause the violation.

Same code that runs in production. No separate spec. No drift.

```bash
mix compile        # single-agent verification (fast, <5ms)
mix vor.check      # multi-agent exploration (thorough)
```

---

## What the compiler checks

**Safety invariants** — "this must never happen." Proven at compile time by exhaustive graph traversal. If any reachable state can violate the invariant, compilation fails.

**System-level invariants** — properties across multiple agents. Verified by product state exploration via `mix vor.check`. Counterexample traces show the exact message interleaving that causes a violation.

**Handler coverage** — every message type declared in the protocol has at least one handler.

**Protocol composition** — when agents are wired together, the compiler verifies that send/accept declarations match.

**Liveness monitoring** — "this must eventually happen." Enforced at runtime with declared timeouts and recovery handlers. The recovery path is itself checked against the safety invariants.

**Gleam type boundary** — extern calls to Gleam modules are validated against Gleam's package-interface.json.

**Internal type tracking** — the compiler propagates types through handler body expressions and catches guaranteed crashes (map operations on integers, arithmetic on maps) at compile time.

**Extern proven boundary** — a `proven` invariant whose verification path depends on an extern result is rejected at compile time. The compiler never claims to prove what it can't see.

---

## What's working

328+ tests, 9 property-based test suites. Five examples: rate limiter, circuit breaker, Raft consensus, G-Counter CRDT, distributed lock. Three CRDT types (G-Counter, PN-Counter, version-based OR-Set) expressed entirely in native Vor with zero extern calls. Compilation under 5ms, verification under 2ms. The safety verifier is itself verified by TLA+ specifications.

A CRDT-based distributed database ([VorDB](https://github.com/vorlang/vordb)) is the first real consumer, driving language features through practical use.

---

## How it fits with Elixir and Gleam

Vor doesn't replace OTP — it compiles to OTP. It's designed to complement Elixir and Gleam:

- **Vor** handles the coordination layer — state machines, protocols, invariants, model checking. The part where getting it wrong causes silent failures in production.
- **Gleam** handles the data processing layer — type-safe transformations, CRDT operations, business logic. Called from Vor via type-validated `extern gleam` blocks.
- **Elixir** handles the infrastructure — supervision trees, deployment, Phoenix, Ecto, and the rest of the ecosystem. Called from Vor via `extern` blocks.

Each language does what it does best. The boundaries between them are explicit and checked.

---

## Design principles

**The spec is the program.** The `.vor` file is both the specification and the implementation. No separate model to maintain. No drift.

**Guarantee tiers are explicit.** Every property is labeled `proven` or `monitored`. The compiler fails closed — it never claims to verify what it can't.

**The extern boundary is a trust boundary.** Proven invariants cannot depend on extern results. Gleam externs are type-validated. The boundary between verified and unverified code is explicit and enforced.

**Failure is first-class.** Invariants can be violated. Resilience handlers define what happens. Recovery paths are verified.

**The BEAM is the foundation.** Thirty years of production-proven concurrency, fault tolerance, and distribution underneath.

---

MIT License
