# Tutorial — ex01: Compound States & Configuration

The conceptual narrative behind the first exercise. After reading this, open [`runbook.md`](./runbook.md) to type the chart out and watch the tests go green.

---

## The story we're building

A traffic light. Three states — `:red`, `:green`, `:yellow` — and one event, `:next`, that cycles between them: red → green → yellow → red. The light starts in `:red`.

This is the smallest possible statechart that does something nontrivial. There's no nesting, no parallelism, no guards, no data — just three states and a cycle. By the end of this exercise you'll have written ~10 lines of Clojure and watched four assertions go green:

```
starts in :red          ✓
:next goes red → green  ✓
:next goes green → yellow  ✓
:next goes yellow → red (cycles)  ✓
```

What you'll have built understanding of, after the tests pass:

1. What a [[statechart]] *is* (data, not behavior).
2. What it means for a chart to *be in* a state (the [[configuration]]).
3. How [[transition]]s read events and move the chart.
4. Why [[document-order]] matters even on the very first exercise.

That last one is the sleeper lesson. Document order is what selects the initial state when you don't say "start in `:red`" explicitly — and document order will come back as the load-bearing concept in ex03 (guards) and ex06 (history). Spot it now.

---

## Step 1 — A statechart is data

The library's central trick: a chart is not a class, not a process, not a long-running thing. It's a plain Clojure map produced by a factory function.

```clojure
(statechart {} ...)
```

The `{}` is a map of *chart-level options* (almost always empty in the early exercises). What follows is a sequence of [[state]] elements, declared as nested function calls.

Because the chart is data, you can:

- Print it.
- Compare two charts with `=`.
- Store one in a `def` and pass it to a runtime later.
- Generate one programmatically.

The runtime — what actually *runs* the chart — is a separate thing. You'll meet it as the **testing env** (created with `t/new-testing-env`) in ex01 and as a production "working env" later. The chart and the runtime are decoupled by design. **The chart describes what could happen. The runtime decides what does.**

> **Common misconception** — "I need to instantiate the chart before I can use it."
>
> You don't. The chart is already a value. What you create is an *environment* (the runtime) that holds the chart, an event queue, the current configuration, and any data. Multiple envs can share the same chart definition.

A consequence of the chart/runtime split: the runtime (env) holds mutable state — the configuration set, the event queue, the data model — but the chart itself doesn't. So **calling `(t/start! env)` a second time on the same env resets the runtime back to the initial configuration** (verified empirically — see [gotchas #6](./gotchas.md)). That said, the curriculum convention is to *build a fresh env per scenario* with `t/new-testing-env`; the reset behavior works but the fresh-env idiom reads more clearly.

---

## Step 2 — States are nodes in a tree

A [[state]] is declared with the `state` element from `com.fulcrologic.statecharts.elements`:

```clojure
(state {:id :red} ...)
```

The `:id` is the state's name. IDs must be unique within the chart. The body is a sequence of child elements — child states, transitions, on-entry actions, on-exit actions.

For ex01, each state has no children — they're **atomic states**, the leaves of the tree. The body of each is just a `transition`:

```clojure
(state {:id :red}
  (transition {:event :next :target :green}))
```

There's no `(state {:id :traffic-light} ...)` wrapping the three siblings. **The chart factory auto-generates a root node with `:id :ROOT`** that acts as their parent — you don't declare it, but it's a real node in the chart data structure (you'll see `:id :ROOT` if you `pprint` the chart). Exactly one of `:ROOT`'s direct children is the "active" child at any moment.

This is why the exercise is titled "Compound States & Configuration" even though no explicit compound state is declared: the auto-generated `:ROOT` plays the compound-state role. The three atomic states are its children. When the chart is running, exactly one of `:red`, `:green`, or `:yellow` is the active child of `:ROOT`.

> **Common misconception** — "Compound states have to be declared explicitly with `state`."
>
> The chart factory adds the root for you. You only need to declare an *additional* compound state with `state` when you want to nest more states inside it (which ex04 and ex06 do). For a flat chart like the traffic light, the auto-generated `:ROOT` is enough.

---

## Step 3 — Transitions live inside states

A [[transition]] declares a directed move: when this event arrives, go to that state.

```clojure
(transition {:event :next :target :green})
```

The transition is declared *inside the source state*. That's how the chart knows where the transition starts from. The `:target` names the destination by `:id`.

```clojure
(state {:id :red}
  (transition {:event :next :target :green}))   ; from :red, on :next, go to :green
```

What this *doesn't* say is interesting:

- Transitions don't carry a "from" — that's the enclosing state.
- Transitions don't carry a "next ID" — they carry a "target," which must be the `:id` of an existing state in the chart.
- A transition that fires causes the chart to *exit* its current state (running on-exit actions) and *enter* the target (running on-entry actions). For atomic states with no actions, the entry and exit are silent.

A state can have multiple transitions. They can react to the same event (ex03 leans on this — two transitions on `:submit`, one guarded, one not). For ex01, each state has exactly one transition.

---

## Step 4 — The initial state is the first child in document order

When the chart is started — `(t/start! env)` — the runtime needs to pick which of the three top-level states to enter. Two rules govern this:

1. **If the parent declares `:initial`**, use that.
2. **Otherwise, the first child in [[document-order]] becomes the initial state.**

For the traffic light, the exercise's chart looks like:

```clojure
(statechart {}
  (state {:id :red} ...)      ; ← first in document order
  (state {:id :green} ...)
  (state {:id :yellow} ...))
```

No `:initial` is declared on the chart, so document order wins. `:red` is first, so the chart starts in `:red`. This is exactly the assertion `(testing "starts in :red" (is (t/in? env :red)))` checks.

If you reordered the chart to put `:green` first, the test would fail — `t/in?` would tell you the chart is in `:green`, not `:red`. Try this in the runbook's "Try breaking it" section.

> **Common misconception** — "States are tried alphabetically / by `:id`."
>
> No. Document order means the order they appear *in the source code*, top to bottom. The runtime walks the chart definition in the order you wrote it.

---

## Step 5 — The configuration: what the chart "is in"

The [[configuration]] is the **set of state IDs the chart is currently active in**. For ex01 it's exactly one element wide — just the active atomic state. You query it with:

```clojure
(t/in? env :red)
```

Internally, `t/in?` is a simple set-membership check: `(contains? configuration :red)`. No hierarchy walk, no ancestor traversal.

For the traffic light, here's what the configuration set actually contains at each moment:

| Moment | Configuration |
| --- | --- |
| Before `start!` | `nil` — the chart hasn't run yet, so working memory has no configuration key |
| After `start!` | `#{:red}` |
| After `(t/run-events! env :next)` | `#{:green}` |
| After another `:next` | `#{:yellow}` |
| After another `:next` | `#{:red}` (cycle complete) |

Worth knowing: the auto-generated `:ROOT` is *not* in the configuration set. `(t/in? env :ROOT)` returns `false` even when the chart is running. The set contains active *atomic* states (and, in nested charts, active compound-state ancestors below `:ROOT`); the chart root itself is excluded.

> **Common misconception** — "The configuration is a single state."
>
> It's a *set*. For ex01 the set happens to have exactly one element, but that's because the chart is flat. In ex04 (parallel regions) the set will have multiple elements simultaneously — one atomic state per region. The "set, not a value" framing pays off there.

> **Common misconception** — "`t/in?` walks up the state hierarchy."
>
> It doesn't. `t/in?` is `contains?` on the configuration set. The reason `(t/in? env <some-ancestor>)` returns true in nested charts is that the runtime *puts* the ancestor's ID into the set — `t/in?` itself is just membership.

---

## Step 6 — Events drive the chart forward

An [[event]] is a signal you send the chart. In tests, you fire one with:

```clojure
(t/run-events! env :next)
```

Three things happen:

1. The runtime walks the current configuration looking for transitions whose `:event` matches `:next`.
2. For each match, it follows the transition: exit the source state, enter the target.
3. The chart settles into its new configuration.

For the traffic light, only the *currently active* state has its transitions checked. So:

- If the chart is in `:red`, the `:red`-state's transition fires (target `:green`).
- If the chart is in `:green`, the `:green`-state's transition fires (target `:yellow`).
- If the chart is in `:yellow`, the `:yellow`-state's transition fires (target `:red`).

This is why the same event — `:next` — produces different behavior depending on where the chart is. Statecharts shine here: one event name, three meanings, all expressed declaratively.

> **Common misconception** — "I need a switch statement to handle `:next`."
>
> The chart *is* the switch statement, distributed across the source states. Each state declares how it handles each event it cares about. The runtime does the dispatch.

What happens if you fire an event nothing matches? Nothing — the event is consumed and dropped. The chart stays in its current configuration. This is not the same as an error; the chart just has nothing to do with that event. (You can verify this in the runbook's "Try this" section by firing `:bogus` and observing the chart stays put.)

---

## Step 7 — Putting it together

The full chart:

```clojure
(def traffic-light
  (statechart {}
    (state {:id :red}
      (transition {:event :next :target :green}))
    (state {:id :green}
      (transition {:event :next :target :yellow}))
    (state {:id :yellow}
      (transition {:event :next :target :red}))))
```

Ten lines. Three states, three transitions, one event. Document order picks `:red` as the initial state because it appears first. The cycle is closed because `:yellow`'s transition targets `:red`.

The tests then assert the cycle works:

1. After `start!`, the chart is in `:red`.
2. After one `:next`, in `:green`.
3. After another, in `:yellow`.
4. After another, back in `:red`.

Four `is` assertions, four `t/in?` calls. That's it.

---

## What you should be able to explain

When you finish the runbook and the tests pass, you should be able to explain — out loud, in plain English — each of these:

- **Why the chart starts in `:red`** — document order: it's the first state declared, no `:initial` overrides it.
- **What `t/in?` is asking** — is the named state anywhere in the current configuration (not "is the chart in *exactly* this state").
- **What "the configuration" means** — the set of active states, including ancestors. For ex01 that's the chart root plus whichever atomic state is active.
- **Why a transition is declared inside the source state** — the enclosing state *is* the "from"; the transition only needs to name the "to."
- **What happens when you fire an event no transition matches** — nothing. The event is dropped; the configuration is unchanged.

If any of these still feel hazy after the runbook, [`glossary.md`](./glossary.md) has the entries to revisit. Then the [`quiz.md`](./quiz.md) checks recall.
