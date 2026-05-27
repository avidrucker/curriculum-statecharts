# Gotchas — ex05: Multi-Target Transitions

Silent failures, hidden behaviors, and easy-to-miss configurations introduced by ex05. See [`gotchas-overview.md`](../gotchas-overview.md) for the doc-type framing.

ex01–ex04 gotchas still apply. This file adds what multi-target transitions newly put on the table.

All entries verified empirically against `com.fulcrologic/statecharts` 1.2.25.

---

### 1. The exercise's test passes with single-target — multi-target is *not* required

**What you'll see.** You solve the exercise with `(transition {:event :reset :target :stopped})` (single keyword, no vector). The test passes. You wonder whether you actually learned anything about multi-target.

**What's really happening.** When a transition enters a parallel without specifying every region's leaf, each unspecified region falls back to its **default initial state** (first document-order child, or whatever `:initial` declares). In the exercise's chart, `:unmuted` is the first child of `:volume` — so single-target `:reset :target :stopped` re-enters the parallel with `:stopped` for `:playback` and the default `:unmuted` for `:volume`. The test's `(t/in? env :unmuted)` assertion passes "by accident."

This isn't a bug in the curriculum — it's a deliberate teaching trap. The exercise's docstring says "multi-target lets you specify exactly which child state in each region to enter," which is true but doesn't say *needed*. Many learners reach for multi-target by reflex; the value of the exercise is realizing that for *this* test, multi-target is one of two equivalent solutions.

**Defense.** Use multi-target when you need *non-default* initials. Use single-target when the defaults work. The rule of thumb: if your transition reads more clearly with `[:stopped :unmuted]` (because you want both spelled out for clarity), use the vector. If it reads more clearly with `:stopped`, use the keyword. Both are correct here.

*See also:* [tutorial Step 2](./tutorial.md), [runbook Probe 1 and "Try breaking it" #1](./runbook.md).

---

### 2. Multi-target with both targets in the same region produces a structurally invalid configuration

**What you'll see.** You write `:target [:stopped :playing]` (both in `:playback`). The chart loads without complaint. The transition fires; both `:stopped` and `:playing` end up in the configuration:

```
config => #{:player :playback :stopped :playing :volume :unmuted}
;;                                ^^^^^^^^^^^^^^^^
;;                                two atomic leaves in :playback — invalid!
```

The chart is now in a state that violates the compound-state rule ("exactly one active child at a time"). The runtime doesn't catch this.

What happens on the *next* event determines whether the chart self-corrects. Empirically (verified during ex05's review pass): if the next event has a transition declared on one of the two ambiguously-active states, that state exits cleanly and the chart settles back to a valid configuration. For example, from `#{:stopped :playing}`, firing `:play`:

```
before :play  →  #{:player :playback :stopped :playing :volume :unmuted}
after  :play  →  #{:player :playback :playing :volume :unmuted}        ; self-corrected
```

The `:play` transition was declared on `:stopped` (`:stopped → :playing`), so `:stopped` exited. `:playing` was already active and stays — net result, the chart is now legally in just `:playing`.

But during the broken period (between the bad multi-target firing and the next event that resolves the ambiguity), `(t/in? env :stopped)` and `(t/in? env :playing)` *both* return `true`. Any code that asserts on one without expecting the other will be confused.

**What's really happening.** The library's chart-construction validator catches invalid `:target` keywords (state IDs that don't exist) but doesn't check that multi-target vectors hit *different* regions. The exit-and-re-entry logic processes each target independently and adds them all to the configuration without enforcing the compound-state invariant.

**Defense.** Every element of a `:target` vector must be in a different region of the parallel (or otherwise non-conflicting in the chart's state tree). The library trusts you; the chart's structure is the constraint. If you must verify, `pprint` the configuration after firing the transition and count atomic leaves per region — anything above one per compound state is broken.

*See also:* [tutorial Step 5](./tutorial.md), [runbook Probe 3 and "Try breaking it" #2](./runbook.md).

---

### 3. Order of targets in the vector doesn't matter — even though it might *look* like it does

**What you'll see.** You write `:target [:unmuted :stopped]` and `:target [:stopped :unmuted]` in two different versions of your chart. You notice them produce the same outcome and wonder if you're missing something.

**What's really happening.** The library resolves each target to its region in the chart's structure; the order in the vector is irrelevant. Internally, the runtime processes targets as a set, not a sequence.

**Defense.** Treat the target vector as a set, not a sequence. **Convention:** list targets in the order their regions appear in the chart definition (e.g., `:playback` first → `:stopped` first). This is a style choice for readability; the runtime doesn't care.

If you find yourself relying on order, you're misunderstanding the semantics — re-read [tutorial Step 4](./tutorial.md).

---

### 4. A misspelled target in the vector throws at chart-registration time

**What you'll see.** You write `:target [:stopped :unmute]` (missing 'd'). When you call `t/new-testing-env`, you get:

```
Cannot register invalid chart
```

The error doesn't tell you *which* target is bad.

**What's really happening.** The chart's `configuration-validator` runs at registration time and catches any `:target` keyword that doesn't resolve to a state in the chart. The error is the same opaque message you've seen since ex02.

**Defense.** When `Cannot register invalid chart` shows up, grep for `:target` in your chart definition and confirm every keyword in every target vector matches a state's `:id`. The pattern from ex02 applies — make a list of all targets, a list of all IDs, and diff them.

For multi-target specifically: even a single typo in a vector of two correct targets will fail the whole chart. Errors here aren't partial.

*See also:* ex02 gotcha #7 (the original "opaque error" entry).

---

### 5. Multi-target on a non-parallel chart produces an invalid configuration too

**What you'll see.** You write `:target [:a :b]` in a chart that has no `parallel` element — just sibling top-level states. The chart loads. The transition fires:

```clojure
(statechart {}
  (state {:id :start}
    (transition {:event :go :target [:a :b]}))
  (state {:id :a})
  (state {:id :b}))
```

After `:go`, the configuration is `#{:a :b}` — **both** states are active simultaneously, despite being siblings at the chart root (which is implicitly a compound state — only one child should be active at a time).

**What's really happening.** The library doesn't restrict multi-target to parallels. The runtime processes each target in the vector independently and adds them to the configuration. There's no validator step that says "wait — these targets aren't in different regions; this can't be valid." The result is the same kind of structurally-invalid configuration as gotcha #2, just applied at a different level of the chart tree (here, the chart root acting as the implicit compound).

**Defense.** Reserve multi-target for parallels. If your chart has no `parallel` and you find yourself wanting `:target [:a :b]`, you probably want either:

1. Two separate transitions (one targeting `:a`, one targeting `:b`, gated by guards or different events).
2. A choice (ex03c) that dispatches conditionally between targets.
3. To introduce a `parallel` if the states are genuinely independent enough to be simultaneously active.

Multi-target is the right tool for **simultaneous activation of independent regions inside a parallel**, not for sequential or alternative state moves.

---

### 6. `:target` vector with one element produces equivalent chart to single keyword — but reads differently

**What you'll see.** You write `:target [:stopped]` (vector with one element). The chart loads. The behavior is identical to `:target :stopped`. Reading the chart later, you wonder why the brackets are there.

**What's really happening.** The library normalizes single-keyword `:target` to a one-element vector internally. `[:stopped]` and `:stopped` produce structurally identical chart definitions (modulo IDs). Both work.

**Defense.** Use the form that signals intent:

- **`:target :stopped`** when there's exactly one target and you want it to read as a "simple" transition.
- **`:target [:stopped :unmuted]`** when there are multiple targets and the vector form is necessary.
- **`:target [:stopped]`** is rarely the right call — it's vector form for a single target, which reads as "this used to be multi-target and I removed all but one" or "I'm not sure if I want one or two."

This is style, not correctness. The runtime treats them the same.

---

### 7. Multi-target with regions that have no default initial defined

**What you'll see.** You build a parallel chart where one region has multiple children but doesn't specify `:initial`. You write a single-target transition that targets only the other region. You expect the unspecified region to "do something reasonable" but the chart enters a weird state, or `(t/in? env <region-id>)` returns true but no atomic leaf in that region is in the configuration.

**What's really happening.** When a region has multiple children and no `:initial`, the runtime picks the *first child in document order* as the default. This usually works, but if your "first child" happens to be a non-trivial state with on-entry effects, you might land in an unexpected initial state.

For ex05's exercise this doesn't come up — both regions have their intended initial as the first child. But in your own charts, **always check what each region's default initial is** when relying on single-target transitions into the parallel.

**Defense.** Two practices:

1. **Be explicit:** declare `:initial` on every region (even when document order would pick the right state). It's two extra keys but documents the intent.
2. **Be explicit in transitions:** for resets and similar "entry from outside" transitions, use multi-target with every region named. It's more verbose but removes any default-initial guessing.

The exercise's exact-test-equivalence between single-target and multi-target is a curriculum convenience. In production code, prefer explicit over implicit.
