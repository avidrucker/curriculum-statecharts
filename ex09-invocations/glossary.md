# Glossary — ex09: Child Chart Invocations

Terms introduced (or first deeply covered) in ex09. Cross-cutting vocabulary lives in the top-level [`glossary.md`](../glossary.md); ex01–ex08 covered the core chart elements. This file adds what ex09 newly puts on the table — the densest set of new terms in the curriculum.

Each entry has a 0–9 criticality rating (see the top-level glossary for the scale).

---

### `invoke` (element)  *(criticality: 9)*

The element from `com.fulcrologic.statecharts.elements` that declares a **child chart invocation**:

```clojure
(invoke {:id :payment-process
         :type :statechart
         :src :payment-chart})
```

Returns[^p1] `{:id :payment-process :type :statechart :src :payment-chart :explicit-id? true :node-type :invoke}`.

The element lives as a child of the state where the invocation should run. The child starts when the state is entered and is terminated when the state is exited.

The library supports many more options on `invoke` than the curriculum uses. From the source:

- `:params` / `:namelist` — pass data into the child's data model.
- `:finalize` — execute logic after every event the child sends back.
- `:autoforward` — forward external events from the parent to the child.
- `:srcexpr`, `:typeexpr` — expression-based versions of `:src` and `:type`.
- `:idlocation` — auto-generate an ID and store it in the data model.

For ex09, the simple form (`:id` + `:type` + `:src`) is sufficient. Reach for the others when production charts need them.

### invokeid  *(criticality: 8)*

The `:id` keyword on an `invoke` element. Three concurrent roles:

1. **The child session's session-id.** When the runtime starts the child, it uses this id as the session-id in the working-memory-store.
2. **The suffix of the auto-event.** When the child reaches a top-level final, the runtime fires `:done.invoke.<this-id>`.
3. **The implicit `:target` of the auto-event.** Actually no — the `:target` is the *parent's* session-id, not the invokeid. The invokeid is the `:source` / source-session-id. But it identifies "which child fired" the auto-event.

Per the library source, you can specify `:id` explicitly (as ex09 does), let the runtime auto-generate it (then store it in `:idlocation`), or both. The exercise uses the explicit form for clarity.

### child session  *(criticality: 9)*

A separate, independent session running the invoked chart. Has its own:

- **Session-id** (= the invokeid).
- **Working memory** (stored in the working-memory-store under the session-id).
- **Configuration** (the child's active states, *not* combined with the parent's).
- **Event queue position** (events are queued globally but processed per-session).

Empirically[^p2]: after the parent enters its invoking state, `(sp/get-working-memory wmstore env :child-invokeid)` returns the child's wmem with its own `::sc/configuration`. Two sessions visible side-by-side.

### `done.invoke.<invokeid>` (auto-event)  *(criticality: 9)*

The internal event the runtime auto-fires when an invoked child chart reaches a top-level final state. Analogous to `done.state.<id>` from ex08 but for invocations.

Event shape[^p4]:

```clojure
{:type :com.fulcrologic.statecharts/chart            ; not :internal — library-specific marker
 :name :done.invoke.payment-process
 :target :checkout-session                            ; the PARENT's session-id
 :data {}
 :com.fulcrologic.statecharts/source-session-id :payment-process    ; the child
 :invokeid :payment-process
 :com.fulcrologic.statecharts/event-name :done.invoke.payment-process}
```

The `:target` field tells the event queue's handler which session to route to. The parent's transition handles the event via its `:event` name:

```clojure
(transition {:event :done.invoke.payment-process :target :confirmation})
```

### `simple/simple-env`  *(criticality: 7)*

The function from `com.fulcrologic.statecharts.simple` that creates a real (non-mocking) environment with all default components:

- `working-memory-data-model` (flat in-memory model)
- `manually-polled-queue` (event queue with explicit drain)
- `local-memory-registry` (chart registry)
- `local-memory-store` (working-memory storage)
- `v20150901` processor (the SCXML algorithm)
- **`statechart-invocation-processor`** — the real invocation processor that actually starts child charts
- `lambda` execution model

The "simple" name is misleading — it's the *minimal real* env, not a stripped-down stub. Compare to `t/new-testing-env`, which adds `MockExecutionModel` and `MockInvocations` for convenient testing of single-chart behavior.

For ex09, simple-env is required because invocations need the real invocation processor.

### `MockInvocations` (the testing-env stub)  *(criticality: 5)*

A library implementation that replaces the real `statechart-invocation-processor` when using `t/new-testing-env`. The mock doesn't actually start child charts — it just records that an invoke was requested. Useful for testing "did the chart try to invoke X?" but not "what does the child do?"

Practically: every exercise except ex09 uses charts that don't have `invoke` elements, so the mocking is invisible. ex09 hits the limit; the exercise's test explicitly explains why it uses `simple/simple-env` instead.

### chart registry  *(criticality: 7)*

The component in `env` that maps keywords (registry keys) to chart definitions. Both `simple/simple-env` and `t/new-testing-env` include a registry; the latter auto-registers the chart you pass into `t/new-testing-env`.

Usage:

```clojure
(simple/register! env :checkout-chart checkout-chart)
(simple/register! env :payment-chart payment-chart)
```

The registry key is what `invoke`'s `:src` references. The chart factory function (`statechart {} ...`) produces the chart value; `register!` associates the value with a key in the env's registry.

### `sp/process-event!`  *(criticality: 6)*

Protocol function from `com.fulcrologic.statecharts.protocols`. Processes one event against a working memory map, returning the *new* working memory (does not save automatically):

```clojure
(sp/process-event! processor env wmem (new-event :pay))
;; → updated wmem (not saved yet)
```

Used in ex09's test because the manual orchestration requires explicit save calls. For single-chart testing, `t/run-events!` wraps this with auto-save.

### `sp/save-working-memory!`  *(criticality: 5)*

Protocol function to persist an updated wmem back to the working-memory-store. The "save" is just a write to the in-memory store (no disk I/O). The pattern:

```clojure
(let [wmem  (sp/get-working-memory wmstore env session-id)
      wmem2 (sp/process-event! processor env wmem event)]
  (sp/save-working-memory! wmstore env session-id wmem2))
```

This sequence is what `t/run-events!` does implicitly. ex09's test does it explicitly because the same orchestration is needed for both parent and child sessions.

### `sp/receive-events!`  *(criticality: 6)*

Protocol function that drains the event queue. Each event is passed to a handler function that decides what to do (typically: look up the event's `:target` session, process the event, save the result):

```clojure
(sp/receive-events! event-queue env
  (fn [{::sc/keys [working-memory-store processor] :as env} {:keys [target] :as event}]
    (when-let [wmem (sp/get-working-memory working-memory-store env target)]
      (let [result (sp/process-event! processor env wmem event)]
        (sp/save-working-memory! working-memory-store env target result))))
  {})
```

In ex09's test, this is how the `done.invoke` event gets routed from the queue back to the parent's session. Without this call, the event would sit in the queue forever and the parent would never react.

### parent-child lifecycle coupling  *(criticality: 8)*

The rule: a child invocation's lifetime is bounded by its parent state's activity[^p5].

| Parent state | Child session |
| --- | --- |
| Not yet entered | Not yet created |
| Entered (invoke runs) | Started; working memory populated |
| Still active | Continues running independently |
| Exited (any reason) | Automatically terminated; working memory removed |

The library guarantees cleanup. You cannot "leak" a child session by forgetting to terminate it — exiting the parent state does it for you.

This makes complex flows safe to design: a user-cancelled flow auto-cleans the child. A normal-completion flow does the same (the parent's transition out of the invoking state, triggered by `done.invoke`, also exits the state and cleans up the now-already-done child).

### `:autoforward` (mentioned, not used in exercise)  *(criticality: 3)*

An option on `invoke` that, when true, forwards every external event the parent receives onward to the child. Used when the parent is acting as a "wrapper" and wants the child to see the same events.

For ex09 the parent and child speak different event vocabularies (`:pay` is parent-only; `:charge-complete` is child-only), so `:autoforward` isn't needed.

### `:params` / `:namelist` (mentioned, not used)  *(criticality: 3)*

Options on `invoke` for passing data from the parent into the child's data model at startup. `:params` takes a map of paths to expressions; `:namelist` takes a map of source-paths in the parent to target-paths in the child.

For ex09, the payment chart needs no parent data — it's a simple workflow. Real charts often use `:params` to initialize the child with the user's selection, the current amount, etc.

### chart composition (the architectural pattern)  *(criticality: 6)*

The big-picture pattern ex09 enables: a top-level chart invokes feature-specific charts that may themselves invoke worker charts. Each invocation is a clean boundary — the parent doesn't see the child's internals; the child doesn't see the parent's.

Typical structure in production:

```
:app-routing (top-level)
  └─ invokes :feature-X (handles a UI feature)
       └─ invokes :worker-Y (handles async work)
            └─ ...
```

This is how Fulcro applications layer responsibility — UI routes invoke feature charts; feature charts invoke remote-call coordinators; remote-call coordinators invoke retry-with-backoff charts. The composability of invocations is what makes statecharts scale beyond toy examples.

---

> **Verified empirically (probes run during ex09 authoring):**
>
> - [^p1]: `(invoke {:id :a :type :statechart :src :b})` returns `{:id :a :type :statechart :src :b :explicit-id? true :node-type :invoke}`
> - [^p2]: After parent enters invoking state, child session exists with its own configuration; the two sessions are independently inspectable
> - [^p4]: Done-event has `:target` (parent session-id), `:invokeid`, `:source-session-id`, and `:type :com.fulcrologic.statecharts/chart`
> - [^p5]: Parent exiting the invoking state auto-terminates the child session (working memory removed from store)
