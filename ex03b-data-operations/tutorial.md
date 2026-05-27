# Tutorial — ex03b: Data Model Operations & Event Data

The conceptual narrative behind exercise 3b. After reading, open [`runbook.md`](./runbook.md) to build the chart and watch the tests go green.

---

## The story we're building

A counter. One state, `:active`. The counter starts at `0`. Events drive it up and down without ever changing which state the chart is in:

- `:increment` → `:count` += 1
- `:decrement` → `:count` -= 1 (floor at 0)
- `:set-value` (with event data `{:value N}`) → `:count` ← N
- `:reset` → `:count` ← 0

This is the smallest possible chart that does **work inside a state** rather than **moving between states**. ex01–ex03 were all about transitions and configurations — moving between named situations. ex03b is the first exercise where the chart *modifies its data* without changing the configuration at all.

What you'll have grounded by the end:

1. How to **initialize session data** when a state is entered (the production equivalent of ex03's `goto-configuration!` shortcut).
2. How **`script`** elements execute side-effecting expressions inside a chart.
3. How to write a **targetless transition** — a transition that fires on an event but stays in the same state, used purely to update the data model.
4. The **`handle`** sugar from the convenience namespace, which collapses `(transition + script)` into one line.
5. How to **read the event's payload** inside an expression: `(-> data :_event :data :key)`.
6. The vector-of-ops contract: expressions must return `[op1 op2 ...]`, *never* a single op or a non-vector value.

ex03 introduced `op/assign` as a thing tests use to seed data. ex03b is where you put `op/assign` into the *chart itself* and see the data model evolve over time as events arrive.

---

## Step 1 — `on-entry` runs side effects when a state activates

Every `state` element can carry an `on-entry` block. The runtime executes its contents *every time the state is entered*. For the counter chart, on-entry is where `:count` gets initialized to 0:

```clojure
(state {:id :active}
  (on-entry {}
    (script {:expr (fn [_env _data] [(ops/assign :count 0)])}))
  ...)
```

Three pieces here:

- **`on-entry`** — the element that wraps entry-time actions. The `{}` is an attribute map (almost always empty in early exercises).
- **`script`** — the executable-content element that holds an expression.
- **`:expr`** — the function the script runs. Signature `(fn [env data] -> ops-vector)` (same shape as a guard, but the return value is a *vector of operations* instead of a boolean).

When the chart enters `:active`, the runtime calls the `:expr` function, gets back `[{:op :assign :data {:count 0}}]`, interprets the op, and writes `:count 0` into the session data. The state is now "in" with the data model populated.

> **Common misconception** — "`on-entry` runs once, when the chart starts."
>
> It runs *every time the state is entered* — including re-entries. If a state's transition targets itself (or an ancestor and then back), `on-entry` fires again. This matters when you use `on-entry` to reset data: the side effect re-applies each entry, not just the first one.

### A timing subtlety worth knowing

Inside the on-entry's `:expr`, **`data` is the data model state *before* this script's ops are applied.** That means on the chart's first entry, `data` is `nil` — even if the script's *job* is to initialize the data.

```clojure
(on-entry {}
  (script {:expr (fn [_ data] [(ops/assign :count (inc (:count data)))])}))
;; First entry: data = nil → (:count nil) = nil → (inc nil) → NPE!
```

The exercise's solution avoids this by writing a value, not reading-then-writing:

```clojure
(script {:expr (fn [_ _] [(ops/assign :count 0)])})    ; pure write, no read
```

If you ever do need to read-then-modify in on-entry (e.g., to merge with previous data), guard the read against `nil`: `(or (:count data) 0)`.

---

## Step 2 — `script` returns a *vector* of operations

The shape of a script's return value is **non-obvious and unforgiving**:

| Return value | What the runtime does |
| --- | --- |
| `[(ops/assign :x 1)]` | Applies the assign. `:x` is now 1. |
| `[(ops/assign :x 1) (ops/assign :y 2)]` | Applies both ops in order. |
| `(ops/assign :x 1)` *(not in a vector!)* | **Applies nothing.** The runtime silently ignores it. |
| `nil` | Applies nothing. (Reasonable — script had nothing to do.) |
| `[]` | Applies nothing. (Reasonable — explicit empty list.) |

The danger row is the third one. `ops/assign` returns a map (`{:op :assign :data ...}`), which *looks* like an op. If you forget the vector wrapping, your script "succeeds" — no error, no warning — and your data model just doesn't change.

```clojure
;; LOOKS RIGHT. SILENTLY DOES NOTHING.
(script {:expr (fn [_ _] (ops/assign :count 0))})

;; CORRECT.
(script {:expr (fn [_ _] [(ops/assign :count 0)])})
```

The vector wrapping is *mandatory*. The runtime expects `[op op op]` and iterates; if it receives a single op, it tries to iterate over a map (treating each key-value pair as an "op"), nothing matches the op interpreter's expectations, and the effect is dropped.

> **Common misconception** — "If I'm only doing one op, the brackets are optional."
>
> They're not. The runtime always expects a sequence. One op still needs `[op]`. See [`gotchas.md` #1](./gotchas.md) — this is the most common bug authors hit when writing their first ex03b-style chart.

---

## Step 3 — Targetless transitions: events that update data without changing state

The counter's `:increment` event shouldn't move the chart anywhere — it should just increment `:count` and keep the chart in `:active`. Statecharts express this with a **targetless transition**:

```clojure
(transition {:event :increment}                    ; no :target!
  (script {:expr (fn [_ data]
                   [(ops/assign :count (inc (:count data)))])}))
```

When `:increment` arrives, the runtime:

1. Finds the matching transition.
2. Runs the body (the script's `:expr`).
3. Applies the returned ops to the data model.
4. **Doesn't move the chart** — there's no `:target`, so the configuration is untouched. `:active` stays "in."

This is how charts evolve data over time without an explosion of states. Imagine the alternative: a `:counter-1`, `:counter-2`, ..., `:counter-N` state per possible value. Absurd. With targetless transitions, one `:active` state hosts the whole counter's lifetime.

> **Common misconception** — "A transition without `:target` is malformed and should error."
>
> It's not — it's a documented pattern. The library treats `:target nil` as "stay where you are." The transition still runs its body (the script's side effects), which is exactly the point.

---

## Step 4 — `handle` — sugar for "targetless transition + script"

Writing `(transition {:event :increment} (script {:expr (fn [_ data] [(ops/assign :count (inc (:count data)))])}))` for every event handler gets verbose fast. The library's convenience namespace gives you `handle`:

```clojure
(handle :increment
  (fn [_ data] [(ops/assign :count (inc (:count data)))]))
```

Three lines collapse to two. The two forms are **structurally equivalent** — `handle` literally emits the longer form internally — but `handle` reads more like a normal event handler.

```clojure
;; These produce equivalent chart structure:
(handle :increment my-expr)
(transition {:event :increment} (script {:expr my-expr}))
```

You'll see both forms in real Fulcro charts. The convenience namespace's `handle` is the idiomatic choice for "event arrives, mutate data, stay put."

> ⚠️ **API stability warning.** The convenience namespace's docstring says **"ALPHA. NOT API STABLE."** Tony Kay reserves the right to change `handle`'s name, arity, or behavior in future library versions. The underlying `transition + script` pattern is SCXML-aligned and stable; `handle` is sugar that might evolve. ex09 and later may use `handle` heavily; if you're shipping production code, weigh the sugar against the stability gradient.

> **Common misconception** — "`handle` does something `transition` can't."
>
> It doesn't. `handle` is sugar. Anything `handle` does, you can write as `(transition {:event ...} (script {:expr ...}))`. Use whichever reads better in context.

---

## Step 5 — Reading event payloads via `:_event`

ex03 established that **`data`** contains the session data *plus* a special `:_event` key. ex03b is the first exercise where you actually read event-time payload from `:_event`:

```clojure
(handle :set-value
  (fn [_env data]
    (let [v (-> data :_event :data :value)]
      [(ops/assign :count (or v 0))])))
```

The path `(-> data :_event :data :value)`:

- `data` — the function's second arg.
- `:_event` — the runtime-injected key holding the current event.
- `:data` — the event map's payload key.
- `:value` — your application's chosen key inside the payload.

So when the test fires `{:name :set-value :data {:value 42}}`, the `:_event` value is `{:type :external, :name :set-value, :data {:value 42}, ::sc/event-name :set-value}`, and `(-> data :_event :data :value)` resolves to `42`.

The `(or v 0)` guard is paranoia: if a malformed `:set-value` arrives without a `:value` in its payload, we default to 0 rather than assigning `nil`. Production code regularly does this.

> **Common misconception** — "Event data is in `env`, not `data`."
>
> ex03's gotchas already covered this — `env` is *plumbing* (the runtime infrastructure); `data` is *content* (what the chart is currently working with, including the current event). When in doubt about where to look for a value: if it's about "what just happened" or "what's in the model," it's `data`. If it's about "how do I send a message to the chart" or "where's the data model implementation," it's `env`.

---

## Step 6 — Reading the data model in tests with `t/data`

ex03's tests used `(t/in? env :state-name)` to check *where* the chart was. ex03b's tests check *what the chart's data looks like* — and for that, the library provides `t/data`:

```clojure
(is (= 0 (:count (t/data env))))
```

`t/data` returns the current data-model map. Before `(t/start! env)` runs, it returns `nil`. After `start!` runs and the on-entry script has assigned values, it returns the populated map.

`t/data` is a *test-helper* in the same family as `t/in?` and `t/run-events!`. You won't reach for it in production charts — production code that wants to know the data model uses the runtime's data-model protocol directly.

---

## Step 7 — Putting it together

The chart, in plain English:

> One state, `:active`. On entry, initialize `:count` to 0. Listen for `:increment`, `:decrement` (floored at 0), `:set-value` (from event data), and `:reset`. Each event updates `:count` and leaves the chart in `:active`.

The chart, as a tree:

```
:active
  ├─ on-entry: assign :count 0
  ├─ on :increment   (no target)  → assign :count (inc :count)
  ├─ on :decrement   (no target)  → assign :count (max 0 (dec :count))
  ├─ on :set-value   (no target)  → assign :count (event :value, or 0)
  └─ on :reset       (no target)  → assign :count 0
```

The chart, as the solution's Clojure:

```clojure
(def cart-chart
  (statechart {}
    (state {:id :active}
      (on-entry {}
        (script {:expr (fn [_ _] [(ops/assign :count 0)])}))

      (handle :increment
        (fn [_ data] [(ops/assign :count (inc (:count data)))]))

      (handle :decrement
        (fn [_ data] [(ops/assign :count (max 0 (dec (:count data))))]))

      (handle :set-value
        (fn [_ data]
          (let [v (-> data :_event :data :value)]
            [(ops/assign :count (or v 0))])))

      (handle :reset
        (fn [_ _] [(ops/assign :count 0)])))))
```

The variable name (`cart-chart`) is from an earlier draft of the exercise; the topic is a counter, not a cart. Treat it as cosmetic — the test references this name, so the chart definition must use it.

---

## What you should be able to explain

When you finish the runbook and the tests pass, you should be able to explain — out loud, in plain English — each of these:

- **Why `[(ops/assign :count 0)]` and not `(ops/assign :count 0)`.** Scripts must return a *vector* of ops; a bare op is silently ignored.
- **What a targetless transition is and why the counter uses one.** A transition without `:target` runs its body but doesn't move the chart — perfect for "update data, stay put" semantics.
- **What `handle` adds over `transition + script`.** Nothing functionally — it's pure sugar. Choice between them is stylistic; the convenience namespace is marked ALPHA so the underlying form is the safer bet for stability-sensitive code.
- **Where event-time data lives.** `(-> data :_event :data <key>)` — `data` is the function arg, `:_event` is the runtime-injected current event, `:data` is the event's payload key.
- **Why on-entry's `data` is `nil` on first entry.** The script runs *before* its own ops are applied. The data model starts empty; on-entry populates it.

If any of these still feel hazy after the runbook, [`glossary.md`](./glossary.md) has the entries to revisit. Then [`quiz.md`](./quiz.md) checks recall, and [`gotchas.md`](./gotchas.md) collects the silent footguns specific to data operations.
