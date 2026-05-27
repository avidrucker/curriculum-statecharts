# Tutorial — ex03c: Convenience Helpers (`on`, `handle`, `choice`)

The conceptual narrative behind exercise 3c. After reading, open [`runbook.md`](./runbook.md) to build the chart and watch the tests go green.

---

## The story we're building

The login gate again — but **rewritten using the convenience namespace**. The behavior is identical to ex03, but the chart definition is shorter and the conditional dispatch is more obviously structural:

- `:idle` — accepts `:enter-password` (stores it in session data via `handle`) and `:submit` (moves to `:checking` via `on`).
- `:checking` — wraps a `choice` element that branches on the stored password and dispatches to `:authenticated` or `:rejected`.
- `:rejected` — accepts `:retry` (back to `:idle` via `on`).
- `:authenticated` — a leaf state.

What you'll have grounded by the end:

1. The **convenience namespace** as a *sugar layer* — every helper expands to elements from the elements namespace; nothing new is added to the runtime.
2. **`on`** — the one-liner for a transition that just moves on an event.
3. **`choice`** — a state-shaped element whose body is an "if/elif/else" dispatch via eventless guarded transitions.
4. **Eventless transitions** — transitions without an `:event` key, which fire automatically when their state is entered. This is the underlying mechanic that makes `choice` work.
5. The **transient-state pattern** — a state the chart only *passes through*, never lingers in. Choice states are the canonical example.

A note on naming, before we go further: this curriculum's tutorial Step numbering tracks the *conceptual* progression, not the order the runbook builds in. The runbook builds bottom-up (states first, transitions inside); this tutorial introduces concepts top-down (sugar → underlying mechanics).

---

## Step 1 — The convenience namespace is sugar, not a parallel API

`com.fulcrologic.statecharts.convenience` exports functions and macros that *emit* the same chart elements you'd build by hand from the elements namespace. Calling `(on :submit :checking)` produces a `:transition` element identical (in shape) to what you'd get from `(transition {:event :submit :target :checking})`.

The library deliberately splits chart authoring into two layers:

- **`com.fulcrologic.statecharts.elements`** — the *primary* API. SCXML-aligned. Stable. `state`, `transition`, `parallel`, `final`, `script`, `on-entry`, `on-exit`, etc.
- **`com.fulcrologic.statecharts.convenience`** — the *sugar* layer. **"ALPHA. NOT API STABLE."** (per the namespace's own docstring). `on`, `handle`, `choice`, `assign-on`, `send-after`, etc.

You can mix the two freely. The runtime doesn't care which form a transition came from — it sees the same `:node-type :transition` map either way.

> **Common misconception** — "The convenience helpers do something the verbose form can't."
>
> They don't. *Every* convenience helper emits one or more standard elements. You could rewrite any chart authored with convenience helpers entirely using `transition`, `state`, `script`, etc., and get a structurally-equivalent chart (modulo auto-generated IDs).

The trade-off:

- **Sugar** is shorter to write, often easier to read, and reduces boilerplate at scale.
- **Verbose** is more stable across library versions, more explicit about what each element does, and works without learning the convenience namespace's vocabulary.

For curriculum learning, you'll see both. For production code, pick deliberately — for ex03c we use the sugar because the topic *is* the sugar; for a long-lived production chart, the elements-namespace form is the safer default.

---

## Step 2 — `on` is the simplest sugar

`on` collapses a one-line transition into a one-line transition:

```clojure
;; Convenience
(on :submit :checking)

;; Expands to
(transition {:event :submit :target :checking})
```

That's it. `on` saves a `{}` and three keys. It's used heavily in real Fulcro charts because most transitions *are* "fire on event X, go to state Y" — the kind where the verbose form's `{:event ... :target ...}` is repetitive noise.

`on` supports an extra-args form for body content (per its docstring): `(on :submit :checking other-element ...)` — anything after the target becomes a child of the transition. That's how you attach a script or a `log` to an `on`. We don't use that form in ex03c.

> **Common misconception** — "`on` is a special declarative form, not a function call."
>
> It's a regular function. You can `(apply on …)` it. You can `def` an event name and pass it. Convenience helpers are values-in / values-out, not magic.

---

## Step 3 — `handle` (recap from ex03b) is the targetless variant

Already established in ex03b — `handle` is sugar for a `transition + script` pair where the transition has no `:target`:

```clojure
(handle :enter-password
  (fn [_ data]
    [(ops/assign :password (-> data :_event :data :password))]))
```

expands to:

```clojure
(transition {:event :enter-password}
  (script {:expr (fn [_ data]
                   [(ops/assign :password (-> data :_event :data :password))])}))
```

Used here in ex03c for `:enter-password` (which stores data without moving the chart). The vector-of-ops contract from ex03b still applies — `:expr` returns `[op op ...]`, never a single op.

The mental model: **`on` is for moving; `handle` is for staying-but-updating.** Together they cover most chart transitions.

---

## Step 4 — `choice` is a state that dispatches on data

This is the new conceptual piece in ex03c.

`choice` looks like a Clojure `cond` baked into the chart's grammar:

```clojure
(choice {:id :auth-check}
  (fn [_ data] (= (:password data) "secret")) :authenticated
  :else                                       :rejected)
```

Read it as: "When this state is entered, evaluate the predicates in order. The first one that returns truthy decides the target. If none match, fall through to `:else`."

What it *is*, structurally:

```clojure
(state {:id :auth-check :diagram/prototype :choice}
  (transition {:cond (fn [_ data] (= (:password data) "secret")) :target :authenticated})
  (transition {:target :rejected}))                       ; eventless, unconditional → the :else
```

Three things to notice:

1. **`choice` is a `state`.** Not a new chart element — a state element with its `:node-type :state`, an `:id`, and a special `:diagram/prototype :choice` marker (for chart-rendering tools; the runtime ignores it).
2. **The transitions inside are eventless** — no `:event` key. ex03 and ex03b had transitions with `:event`; choice's are different.
3. **The `:else` clause becomes a transition with no `:cond`** (and no `:event`) — unconditionally enabled, last in document order, so it acts as the fallback.

Choice states are typically **transient** — the chart enters them, immediately fires one of their transitions, and exits. By the time you check `(t/in? env :auth-check)`, the chart has already left. Choice states only show up in the configuration if all their transitions fail to fire (i.e., no clause matched and no `:else`) — see [Step 7](#step-7--the-stuck-choice-trap).

> **Common misconception** — "`choice` is a magic conditional that doesn't appear in the configuration."
>
> It *can* appear in the configuration — if the chart enters a choice state and *no* transition fires (because all guards are false and there's no `:else`), the chart gets stuck there. The choice state is then "active" with no way out except an external transition from elsewhere.

---

## Step 5 — Eventless transitions: the mechanic that makes `choice` work

A transition is **eventless** when its `:event` key is missing (or `nil`). Eventless transitions are evaluated automatically when their state is entered, in document order:

```clojure
(state {:id :a}
  (transition {:target :b}))    ; no :event — eventless

(state {:id :b})
```

If the chart enters `:a`, it immediately leaves and goes to `:b`. The chart is never observably "in" `:a` from the test's point of view — `(t/in? env :a)` returns false because by the time you check, the runtime has already taken the eventless transition.

With a `:cond`, an eventless transition is *conditional*: it fires immediately when the state is entered *only if* the cond returns truthy:

```clojure
(state {:id :a}
  (transition {:cond predicate :target :b})
  (transition {:target :c}))                  ; fallback — fires if predicate is false
```

`choice` is exactly this pattern, with a special declaration shape.

> **Common misconception** — "Eventless transitions fire on every event."
>
> They fire when their state is *entered*, plus during the chart's "settling" phase between events. They don't fire on every external event. The exception is when an event-triggered transition delivers the chart into a state with an eventless transition — those settle as part of the same event's processing cycle.

### What's `:_event` in an eventless predicate?

ex03 and ex03b established that guards (`:cond` on event-triggered transitions) see the current event in `data._event`. **Inside an eventless guard (including a choice predicate), `:_event` reflects the event that *led to* the choice state — if any.**

```clojure
(on :submit :checking)            ; in :idle, fires :submit, takes us to :checking
                                  ; which contains the choice
;; inside choice's predicate:
;; (-> data :_event) → {:name :submit, :data {...}, ...}
```

If the choice is reached automatically (e.g., it's the initial state of the chart), `:_event` is `nil` in its predicates. This rarely matters for ex03c's example (the predicate only reads `:password` from session data), but it's worth knowing for charts that *do* dispatch on event data inside a choice.

---

## Step 6 — Putting it together: ex03's chart rewritten with convenience

Here's the chart, written two ways. Same behavior. Compare them side-by-side.

### The verbose form (echoes ex03)

```clojure
(statechart {}
  (state {:id :idle}
    (transition {:event :enter-password}
      (script {:expr (fn [_ data]
                       [(ops/assign :password
                                    (-> data :_event :data :password))])}))
    (transition {:event :submit :target :checking}))

  (state {:id :checking}
    (state {:id :auth-check}
      (transition {:cond (fn [_ data] (= (:password data) "secret"))
                   :target :authenticated})
      (transition {:target :rejected})))

  (state {:id :authenticated})

  (state {:id :rejected}
    (transition {:event :retry :target :idle})))
```

### The convenience form

```clojure
(statechart {}
  (state {:id :idle}
    (handle :enter-password
      (fn [_ data]
        (let [pw (-> data :_event :data :password)]
          [(ops/assign :password pw)])))
    (on :submit :checking))

  (state {:id :checking}
    (choice {:id :auth-check}
      (fn [_ data] (= (:password data) "secret")) :authenticated
      :else :rejected))

  (state {:id :authenticated})

  (state {:id :rejected}
    (on :retry :idle)))
```

The convenience form drops 8 lines and three nested `{...}` maps. The choice's intent (if-secret-go-authenticated-else-rejected) reads more like code prose.

Either form passes the tests identically.

---

## Step 7 — The "stuck choice" trap

The choice's `:else` is the safety net. **If you write a choice with no `:else` and all predicates return falsy, the chart gets stuck in the choice state.** No error, no warning, no exception. Just a chart that doesn't move.

```clojure
(choice {:id :decide}
  (fn [_ _] false) :a
  (fn [_ _] false) :b)
;; no :else
```

Empirically (probe P5 in the runbook): after the chart is driven into this choice, `(t/in? env :decide)` returns `true` and the chart stays there indefinitely. Subsequent events that aren't handled by the choice (or its ancestors) are dropped silently.

This is unique to `choice` (and other eventless-only states). Event-triggered transitions don't have this failure mode in the same way — an event with no matching transition is just dropped, not "stuck." But a choice's whole purpose is to dispatch *immediately on entry*, so if it can't dispatch, the chart hangs.

**Defense:** *Always* include an `:else` clause unless you're certain every reachable state of the data model is covered by the explicit clauses. Even when you "know" your guards cover everything, the unexpected nil or default-value case will eventually appear. See [`gotchas.md` #1](./gotchas.md).

> **Common misconception** — "Without `:else`, the choice just doesn't fire and the chart returns to the previous state."
>
> No. The chart *entered* the choice. It can't undo that. It's stuck in the choice with no eligible transition.

---

## What you should be able to explain

When you finish the runbook and the tests pass, you should be able to explain — out loud, in plain English — each of these:

- **What `on` is sugar for.** `(transition {:event ... :target ...})`. Nothing more.
- **What `choice` is structurally.** A state with `:diagram/prototype :choice` and eventless transitions as children. Not a new chart element type.
- **Why `(t/in? env :auth-check)` is almost always false in the test.** Choice states are transient — the chart passes through, doesn't linger.
- **Why a choice without `:else` and all-false guards hangs the chart.** Eventless transitions only fire when their cond passes; if none do and there's no fallback, the choice state has no eligible transition.
- **When to prefer the verbose form over `choice`.** When the chart is long-lived production code that needs to survive library upgrades (the convenience namespace is ALPHA), or when the conditional logic is complex enough that a real `state` with named transitions reads better.

If any of these still feel hazy after the runbook, [`glossary.md`](./glossary.md) has the entries to revisit. Then [`quiz.md`](./quiz.md) checks recall, and [`gotchas.md`](./gotchas.md) collects the silent footguns specific to convenience helpers.
