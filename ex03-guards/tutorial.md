# Tutorial — ex03: Guards (Conditions) & Document Order

The conceptual narrative behind exercise 3. After reading, open [`runbook.md`](./runbook.md) to build the chart and watch the tests go green.

---

## The story we're building

A login gate. The user lands in `:login`, types a password, and hits `:submit`. The chart inspects the password:

- correct (`= "secret"`) → `:authenticated`
- anything else → `:rejected`, with a `:retry` event that brings them back to `:login`

This is the smallest chart that demonstrates *conditional* transitioning. The chart has two transitions on the same event (`:submit`) leaving the same state (`:login`). What makes them behave differently is a **guard** — a function attached to the first transition that decides whether it's allowed to fire.

If you've been waiting for the moment statecharts stop looking like flat enums and start looking like *programs* — this is it. Guards are the seam where chart machinery meets domain logic.

What you'll have grounded by the end of this exercise:

1. What a **guard** (`:cond`) is, how its function is called, what it can see.
2. How the runtime picks between multiple transitions matching the same event — the **first match wins** rule, and why **document order** is suddenly load-bearing.
3. What the chart's **data model** (a.k.a. session data) is, where it lives, and how to get values into it for tests.
4. The **fallback pattern**: a guarded transition first, an unguarded one second, used as a "default branch."
5. (Surface mention; full coverage in ex03b/ex03c) `op/assign` and `on-entry` for setting up data the production way.

---

## Step 1 — A guard is a predicate attached to a transition

A guard is a function declared on a transition's `:cond` key. The runtime calls it when an event arrives that matches the transition's `:event`. If the guard returns truthy, the transition fires. If falsy, the runtime moves on (and tries the next matching transition, in [[document-order]]).

```clojure
(transition {:event  :submit
             :cond   (fn [env data] (= (:password data) "secret"))
             :target :authenticated})
```

The signature is **fixed**: `(fn [env data] -> boolean)`. The library calls it with two args, in that order.

- **`env`** — the runtime environment map. Has internal keys like `::sc/working-memory-store`, `::sc/data-model`, `::sc/event-queue`, `::sc/processor`, etc. You rarely read from it directly in ex03; later exercises (especially ex09 invocations) will.
- **`data`** — the **session data** map. This is what you read in 99% of guards. Top-level keys are whatever your chart has assigned (via `op/assign`, `on-entry`, or `goto-configuration!` in tests).

`data` also contains a special key, **`:_event`**, holding the *current event*:

```clojure
data
;; => {:password "secret"
;;     :_event {:type :external
;;              :name :submit
;;              :data {}
;;              :com.fulcrologic.statecharts/event-name :submit}}
```

So a guard can read the session data *and* the event payload. The exercise uses the session-data form; later exercises and real Fulcro apps lean on `:_event` to read fields that came in with the event.

> **Common misconception** — "The guard takes the event as its first argument."
>
> It doesn't. The signature is `[env data]`. The event is reachable through `data._event` — but if you write `(fn [event data] ...)` thinking `event` is the event, you're actually binding the *env* to a misleadingly-named local. The chart will run, the guard will compile, and it'll probably return the wrong answer.

---

## Step 2 — Two transitions on the same event: first match wins

In ex01 each state had one transition per event. In ex02 each state had distinct events on each transition (so document order didn't come into play within a state). Now ex03 puts *two transitions on the same event in the same state*:

```clojure
(state {:id :login}
  (transition {:event  :submit
               :cond   (fn [_env data] (= (:password data) "secret"))
               :target :authenticated})
  (transition {:event :submit :target :rejected}))
```

When `:submit` arrives while the chart is in `:login`:

1. The runtime walks `:login`'s transitions in [[document-order]] (the order they appear in your source).
2. For each transition matching the event (both match here), the runtime evaluates the `:cond`. **No `:cond` means the transition is unconditionally enabled** (treat absent as "always true").
3. The **first** transition whose `:cond` returns truthy fires. Subsequent transitions are *not consulted*.
4. The chart moves to the firing transition's `:target`.

So with password `"secret"`:

- Transition 1: event matches ✓; guard returns `true` ✓ → fires; target `:authenticated`. Done.

With password `"wrong"`:

- Transition 1: event matches ✓; guard returns `false` ✗ → skipped.
- Transition 2: event matches ✓; no guard, so unconditionally enabled → fires; target `:rejected`. Done.

This is the **fallback pattern**: the guarded specific case first, the unguarded "default" second. Document order is the rule that makes it work.

> **Common misconception** — "Guards are tried in priority / specificity / order-of-likelihood / alphabetical order."
>
> They aren't. They're tried in **document order** — the order you wrote them in the source. Reorder the transitions and you reorder which one wins on overlapping matches. ex02's "Try breaking it" #3 (catch-all before specifics) demonstrated the failure mode that comes from forgetting this.

### Guards interact with the ancestor walk

ex02 established that when no transition in the active state matches an event, the runtime walks up to ancestor states. ex03 introduces a subtlety: **a transition with a failing guard counts as "no match,"** so the walk continues. Concretely:

```clojure
(state {:id :outer}
  (transition {:event :go :target :elsewhere})                  ; ancestor — no guard
  (state {:id :inner}
    (transition {:event :go :cond falsy-guard :target :other})))  ; inner — guard fails
```

When the chart is in `:inner` and `:go` arrives:

1. `:inner`'s transition matches the event, but its guard returns falsy → skipped.
2. The walk continues to `:outer`.
3. `:outer`'s transition matches, no guard → fires; chart enters `:elsewhere`.

So a failing guard doesn't *block* further consideration of ancestor transitions — it just makes the inner transition non-enabled, and the runtime carries on as if the inner one didn't exist for this event.

### A short-circuit, not a vote

The runtime stops at the first matching+passing transition. The second guard is **not called**, even if it would have returned true. This matters when guards have side effects (they shouldn't, but sometimes do):

```clojure
;; Guard B is NEVER evaluated when guard A passes.
(transition {:event :go :cond guard-a :target :state-1})
(transition {:event :go :cond guard-b :target :state-2})
```

Reorder the transitions and you change which guard runs at all. Keep guards pure to avoid being surprised by this — see [`gotchas.md`](./gotchas.md).

---

## Step 3 — The chart's data model: session data

Every running chart has a **data model** — a per-session storage area the chart can read and write. In the default library configuration (and what every exercise in this curriculum uses), the data model is a plain in-memory map.

Before anything sets data, the model is `nil`:

```clojure
(t/start! env)
;; data model => nil
```

After `op/assign` or `goto-configuration!` puts something in, it's a map:

```clojure
(t/goto-configuration! env [(op/assign :password "secret")] #{:login})
;; data model => {:password "secret"}
```

The data model is what `data` contains when your guard is called. It's also what `data` will contain when on-entry / on-exit / transition-body actions run (covered in ex03b).

### Two ways to put values into the data model

For the **production** chart, you'd typically use an `on-entry` action with `op/assign`:

```clojure
(state {:id :login}
  (on-entry {}
    (script {:expr (fn [env data] [(op/assign :password (:from-form data))])}))
  ;; ... transitions ...
  )
```

ex03 doesn't ask you to write this — the focus is on guards, not data initialization. **ex03b covers `op/assign` and on-entry in depth.**

For **tests**, the library gives you `t/goto-configuration!`. It's a *test-time-only* helper that:

1. Sets the data model to whatever the ops you pass produce.
2. Sets the configuration to the leaf states you name.

```clojure
(t/goto-configuration! env
                       [(op/assign :password "secret")]   ; ops on the data model
                       #{:login})                          ; target leaf states
```

This skips the need to write production code for "how does the password get into session data" inside the exercise. The test fakes it directly.

> **Common misconception** — "`op/assign` is a function call that mutates the data model."
>
> It isn't. `op/assign` returns a *map* (`{:op :assign :data {:password "secret"}}`) — an *operation descriptor*. The runtime (or `goto-configuration!`, in tests) interprets those descriptors and applies the mutation. You don't get to call `op/assign` standalone and have anything happen; it always feeds into a sink that processes ops.

---

## Step 4 — `goto-configuration!` is a test shortcut, not a production API

The exercise's test setup pattern looks like:

```clojure
(t/new-testing-env ...)
(t/start! env)
(t/goto-configuration! env [(op/assign :password "secret")] #{:login})
(t/run-events! env :submit)
(is (t/in? env :authenticated))
```

That third line bypasses the actual entry into `:login` — it teleports the chart there with the data model pre-seeded. **You wouldn't use `goto-configuration!` outside tests.** In production, the chart's own machinery (on-entry actions, event handlers) is what mutates data; the runtime never accepts arbitrary "place me in state X with data Y" commands.

The reason this helper exists: tests often need to set up a specific scenario without driving the chart through every event that would *naturally* lead there. The traffic-light cycle from ex01 is easy to test from scratch — every state is one event away. A login flow's "user lands on the home page with a session token in localStorage and clicks 'edit profile'" is much harder to reach from `start!` and several `run-events!` calls. `goto-configuration!` gives you a hammer for these cases.

There's one quirk worth knowing about now and seeing later: **`goto-configuration!` produces a configuration set that includes `:ROOT`, unlike the normal `start!` lifecycle which excludes it.** See [`gotchas.md` #1](./gotchas.md) for the implications.

---

## Step 5 — Putting it together

The chart, in plain English:

> The user is on the *login* page. When they submit, check their password. If it's correct, *authenticate* them. Otherwise, *reject* them — and let them *retry* back to the login page.

The chart, as a tree:

```
:login
  ├─ on :submit (if password = "secret") → :authenticated
  └─ on :submit                          → :rejected
:authenticated
:rejected
  └─ on :retry                            → :login
```

The chart, in Clojure (the solution shape):

```clojure
(def login-gate
  (statechart {}
    (state {:id :login}
      (transition {:event  :submit
                   :cond   (fn [_env data] (= (:password data) "secret"))
                   :target :authenticated})
      (transition {:event :submit :target :rejected}))
    (state {:id :authenticated})
    (state {:id :rejected}
      (transition {:event :retry :target :login}))))
```

The runbook walks you through building this incrementally and verifying each layer.

---

## What you should be able to explain

When you finish the runbook and the tests pass, you should be able to explain — out loud, in plain English — each of these:

- **Why the guarded transition must come first in document order.** First match wins; if you put the unguarded one first, it always fires.
- **What `data` contains when a guard is called.** Session data (whatever's in the data model) plus `:_event` (the current event's name and payload).
- **The difference between `op/assign` and a side-effecting function.** `op/assign` returns a *descriptor*; the runtime interprets it. Calling it standalone has no effect.
- **Why `goto-configuration!` exists and when not to use it.** It's a test-time shortcut for setting up scenarios you can't easily reach from `start!`. It does not belong in production code.
- **What happens when both guards fail and there's no fallback.** The event is silently dropped; the chart stays put. Same rule as ex01/ex02 — no transition fires, nothing moves.

If any of these still feel hazy after the runbook, [`glossary.md`](./glossary.md) has the entries to revisit. Then [`quiz.md`](./quiz.md) checks recall, and [`gotchas.md`](./gotchas.md) collects the silent footguns specific to guards.
