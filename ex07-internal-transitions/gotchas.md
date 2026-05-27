# Gotchas — ex07: Internal vs External Transitions

Silent failures, hidden behaviors, and easy-to-miss configurations introduced by ex07. See [`gotchas-overview.md`](../gotchas-overview.md) for the doc-type framing.

ex01–ex06 gotchas still apply. This file adds what `:type :internal` and on-entry/on-exit re-fire semantics newly put on the table.

All entries verified empirically against `com.fulcrologic/statecharts` 1.2.25.

---

### 1. External transitions on a compound state re-fire its on-entry/on-exit

**What you'll see.** You build a chart where `:editor` (compound) has expensive on-entry (e.g., loading from a network). You write transitions between `:viewing` and `:editing` like a normal transition:

```clojure
(state {:id :editor}
  (on-entry {} (script {:expr (fn [_ _] (println "LOADING!") [])}))
  (transition {:event :edit :target :editing})           ; default external
  (state {:id :viewing})
  (state {:id :editing}))
```

Every `:edit` event prints "LOADING!" — on-entry re-fires every transition. You expected on-entry to fire once at chart start.

**What's really happening.** Default external transitions on a compound state with descendant targets *include the compound in the exit set*. The compound exits, then re-enters, re-firing on-entry/on-exit each time[^p2]. This is SCXML-spec behavior, not a bug.

**Defense.** When you want on-entry/on-exit to fire only at "outer boundary" moments (chart enters/leaves the compound), add `:type :internal` to within-compound transitions:

```clojure
(transition {:event :edit :target :editing :type :internal})
```

Or — equivalently — declare the transitions on the child states instead of the parent (see gotcha #4).

*See also:* [tutorial Step 2](./tutorial.md), [runbook Probe 2 and "Try breaking it" #1](./runbook.md), [glossary `:type :internal`](./glossary.md).

---

### 2. `:type :internal` is silently ignored when the target isn't a strict descendant

**What you'll see.** You add `:type :internal` to a transition because you think it'll prevent some re-entry. The behavior doesn't change. You wonder if internal is broken.

```clojure
(state {:id :editor}
  (on-entry {} ...)
  ;; :type :internal but target is OUTSIDE :editor
  (transition {:event :go :target :closed :type :internal})
  (state {:id :viewing}))
(state {:id :closed})
```

When `:go` fires, `:editor` exits anyway. on-entry/on-exit fire. The `:type :internal` annotation has zero effect[^p4].

**What's really happening.** SCXML rules say `:type :internal` makes the exit set *exclude* the source state — but **only when the target is a strict descendant of the source**. When the target is anywhere else (sibling, ancestor, unrelated branch, top-level), the source must exit to reach the target, so the LCA-based exit-set computation includes the source. Internal is effectively a no-op in these cases.

The library doesn't validate the precondition; it doesn't warn that your `:type :internal` is meaningless. The annotation just sits there inertly.

**Defense.** When debugging "why isn't internal preventing re-entry?", check whether the target is *strictly* a descendant of the source. If it isn't, internal won't help — and the chart's design probably needs a different fix (e.g., move the on-entry effect elsewhere, or refactor the structure).

*See also:* [tutorial Step 5](./tutorial.md), [runbook Probe 3](./runbook.md).

---

### 3. `:type :internal` on a self-target transition is also silently ignored

**What you'll see.** You have a state that should update data on a self-event without re-firing on-entry. You write:

```clojure
(state {:id :counter}
  (on-entry {} (script {:expr (fn [_ _] (println "ENTERED") [])}))
  (transition {:event :tick :target :counter :type :internal}))
```

You fire `:tick`. The on-entry prints "ENTERED" again. The internal annotation didn't help.

**What's really happening.** When source == target, the transition is treated as a self-loop external transition regardless of `:type`. The state exits, on-exit fires, on-entry fires again[^p6]. Internal annotation has no special "stay in place" semantics for self-targets.

**Defense.** For "update data, stay put" on a single state without re-entering, use a **targetless transition** (from ex03b):

```clojure
(transition {:event :tick}                                   ; no :target!
  (script {:expr (fn [_ data] [(ops/assign :n (inc (:n data)))])}))
```

Or the convenience equivalent:

```clojure
(handle :tick (fn [_ data] [(ops/assign :n (inc (:n data)))]))
```

These don't exit the source state at all — no on-entry/on-exit cycle, no re-entry. The transition's body runs, the data model updates, the chart stays in the same configuration.

*See also:* [ex03b](../ex03b-data-operations/), [tutorial Step 5](./tutorial.md), [runbook Probe 4](./runbook.md).

---

### 4. The transition's *declaration location* determines what `:type :internal` controls

**What you'll see.** You add `:type :internal` to a transition declared inside one of `:editor`'s children:

```clojure
(state {:id :editor}
  (on-entry {} (script {:expr (fn [_ _] (println "LOAD") [])}))
  (state {:id :viewing}
    (transition {:event :edit :target :editing :type :internal}))    ; declared INSIDE :viewing
  (state {:id :editing}))
```

You expect this to be "more internal" than declaring the same transition on `:editor`. In fact, the behavior is identical to the external form — but for a *different reason*. Removing `:type :internal` from this child-declared transition also wouldn't fire `:editor`'s on-entry, because external transitions from a child to a sibling already keep the LCA (`:editor`) intact.

So the `:type :internal` here is redundant; it changes nothing[^p5].

**What's really happening.** The exit set computation uses the LCA of the transition's source state and its target. For a transition declared on `:viewing` targeting `:editing`:
- Source = `:viewing`. Target = `:editing`. LCA = `:editor` (the parent).
- External: exit set = descendants of LCA that are currently active, NOT including LCA itself. So `:editor` doesn't exit.
- Internal: same set, NOT including the source state. Same result.

**Defense.** Use `:type :internal` *only* on transitions declared on the compound state whose on-entry/on-exit you want to skip. Declaring transitions on children gives the same effect without the annotation. The exercise's design (transitions on `:editor`) is the form where `:type :internal` matters.

---

### 5. Forgetting that `:done` should be external, not internal

**What you'll see.** You think "internal is good practice, I'll use it everywhere." You set `:type :internal` on every transition, including `:done`:

```clojure
(transition {:event :done :target :closed :type :internal})    ; target :closed is OUTSIDE :editor
```

The test still passes. You feel clever. But you've muddled the intent.

**What's really happening.** `:closed` is outside `:editor`, so `:type :internal` is silently ignored (gotcha #2). The transition still fires correctly — it exits `:editor`, `save-doc!` runs, the chart enters `:closed`. The annotation doesn't cause any behavior change.

But: when someone reads your chart later, the `:type :internal` on `:done` *implies* you wanted to prevent the parent's exit. They'll be confused — "why is this internal if `:closed` is outside the parent?" The annotation has communicative meaning even when it has no behavioral meaning.

**Defense.** Use `:type :internal` *only* when you specifically want to prevent the source compound from exiting/re-entering. For transitions that *should* exit the compound (like `:done` here), omit `:type` (let it default to external) or write `:type :external` explicitly. The annotation conveys intent; misuse confuses readers.

---

### 6. Reset the side-effect counters between test runs

**What you'll see.** Your chart uses `(swap! load-count inc)` in on-entry. You run the test suite multiple times in one REPL session. After the first run passes, subsequent runs fail because `load-count` is now 2, 3, ... instead of 1.

**What's really happening.** The atoms `load-count` and `save-count` are *top-level* (per the exercise's source). They persist across test runs unless explicitly reset. The exercise's `deftest` does this:

```clojure
(deftest internal-transitions-test
  (reset! load-count 0)
  (reset! save-count 0)
  ...)
```

If you remove the reset, or if you're running the chart in the REPL outside the test, the counters accumulate.

**Defense.** Either reset the atoms at the start of every test that uses them, or scope the counters to `let` blocks inside the test (so they're recreated each run). The exercise uses the top-level approach for simplicity; production code typically uses the latter for isolation.

This isn't strictly an ex07-specific gotcha — it's a general "side-effecting atoms in test fixtures" pattern. Worth flagging because ex07 is the first exercise where it materializes.

---

### 7. Don't conflate `:type :internal` with `:type :internal` in other state-machine libraries

**What you'll see.** You're coming from XState, SCXML.io, or another state-machine library. You assume `:type :internal` means "this transition is private to the chart" or "this transition has no observable side effects" or some other library's meaning.

**What's really happening.** In `com.fulcrologic/statecharts` (following SCXML), `:type :internal` has a *very specific* SCXML-spec meaning: it affects the exit-set computation when the source is a compound and the target is a descendant. It's not a "privacy" flag, not a "no side effects" hint, not a performance optimization.

**Defense.** Forget any prior library's meaning of "internal." The SCXML definition is: *exit set excludes the source state when source is a compound and target is its descendant.* That's it. Every other semantic interpretation is wrong here.

If you find yourself reaching for `:type :internal` for a reason that isn't "I don't want this compound's on-entry/on-exit to re-fire," you probably want a different mechanism (targetless transition for "stay put while updating data," guards for "private to the chart," etc.).

---

> **Verified empirically (probes run during ex07 authoring):**
>
> - [^p2]: `:type :external` (default) on compound source + descendant target re-fires on-entry/on-exit
> - [^p4]: `:type :internal` with non-descendant target is silently ignored; source still exits
> - [^p5]: Transition declared on child state doesn't trigger parent's re-entry regardless of `:type`
> - [^p6]: `:type :internal` self-target still exits and re-enters
