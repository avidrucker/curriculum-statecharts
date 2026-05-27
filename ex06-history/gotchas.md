# Gotchas — ex06: History States (Shallow & Deep)

Silent failures, hidden behaviors, and easy-to-miss configurations introduced by ex06. See [`gotchas-overview.md`](../gotchas-overview.md) for the doc-type framing.

ex01–ex05 gotchas still apply. This file adds what history states newly put on the table.

All entries verified empirically against `com.fulcrologic/statecharts` 1.2.25.

---

### 1. Targeting the compound state instead of the history node silently bypasses history

**What you'll see.** You add `(history {:id :sh :type :deep} :general)` to `:settings`. Your `:open-settings` transition targets `:settings`. The chart enters at `:general` on every open — even after the user has navigated elsewhere. Your "remember where I was" feature doesn't work.

**What's really happening.** History is opt-in via *explicit ID targeting*. The runtime doesn't search compound states for embedded history nodes and auto-use them. If your transition's `:target` is the compound (`:settings`), the runtime enters the compound's default initial. The history node sits there, recording correctly, but is never consulted.

```clojure
;; SKIPS HISTORY
(transition {:event :open-settings :target :settings})

;; INVOKES HISTORY
(transition {:event :open-settings :target :settings-history})
```

**Defense.** When debugging "history isn't working," the first check is your transition's `:target` value — it must be the history node's `:id` exactly. Verified empirically[^p4]: targeting `:settings` with a history node defined still lands at `:general`.

*See also:* [tutorial Step 2](./tutorial.md), [runbook Probe 3 and "Try breaking it" #1](./runbook.md).

---

### 2. Shallow history loses sub-state in nested compounds

**What you'll see.** You use `:type :shallow` because you don't need anything fancy. Your chart works for top-level tab switching, but as soon as a tab has its own sub-states (e.g., `:privacy > :privacy-advanced`), re-opening always lands at the tab's *default* sub-state — not where the user actually was.

**What's really happening.** Shallow history records only the immediate child of the history's compound parent. For `:settings > :privacy > :privacy-advanced`, shallow records `:privacy`; re-entry restores `:privacy`, which then enters at its own default initial (`:privacy-basic`)[^p3]. `:privacy-advanced` is lost.

```
User flow: :settings → :privacy → :privacy-advanced → :close → :open
Shallow:   re-opens to :privacy → :privacy-basic    (sub-state lost)
Deep:      re-opens to :privacy → :privacy-advanced (exact state restored)
```

**Defense.** For multi-level navigation (any chart where compound states contain further compound children), use `:type :deep`. For top-level-only navigation (a flat list of tabs), `:type :shallow` is enough. When in doubt, default to deep — the cost is negligible and the behavior matches user intuition.

*See also:* [tutorial Step 5](./tutorial.md), [runbook Probe 2 and "Try breaking it" #2](./runbook.md).

---

### 3. The history node's default target is *required*

**What you'll see.** You write `(history {:id :sh :type :deep})` thinking the third argument is optional — maybe the library will fall back to the compound's `:initial`. The chart fails to construct:

```
Wrong number of args (1) passed to: com.fulcrologic.statecharts.elements/history
```

**What's really happening.** `history`'s function signature is `[attrs default-transition]` — both arguments are positional and required[^p6]. The library asserts that the default transition has a `:target`. There's no fallback behavior; you must specify what fires on first visit.

**Defense.** Always provide the default. Pick whatever makes sense for "the user has never been here before" — typically the same state as the compound's `:initial`, but you can choose differently:

```clojure
(history {:id :sh :type :deep} :general)
;; or
(history {:id :sh :type :deep} (transition {:target :general}))    ; explicit transition form
```

---

### 4. History records what was active at exit — not earlier intermediate states

**What you'll see.** Your user navigates: `:settings → :privacy-advanced` (deep nav), then back to `:general` (via `:tab-general`), then closes settings. Re-open — and the chart lands at `:general`, *not* the deep `:privacy-advanced` you remember them visiting earlier.

**What's really happening.** History snapshots at exit time. The `:tab-general` was a within-compound transition that updated *what will be recorded*; the earlier `:privacy-advanced` visit is overwritten[^p5]. Only the state right before exit matters.

This matches SCXML standard semantics ("history is recorded as part of the exit set"). But it can be surprising to users who think "history saves *whatever* I was on at any point."

**Defense.** Two perspectives:

1. **For users**: history remembers your *last* spot, not your *every* spot. This is usually what you want for "resume where I left off" UX.
2. **For chart authors**: if you genuinely want "remember the deepest state ever visited" semantics, history doesn't provide that — you'd need to roll your own tracking via the data model (`op/assign`) and a custom restore-transition.

---

### 5. `(t/in? env :history-node-id)` is always `false`

**What you'll see.** You add diagnostic logging that checks `(t/in? env :settings-history)`. You expect it to return `true` briefly during the open/restore cycle. It never does.

**What's really happening.** History is a **pseudo-state** — it has an `:id` and can be targeted by transitions, but it isn't a state the chart can be "in." When the chart enters via the history node, the runtime resolves the history (default or restored) and enters *that* target; the history node itself never appears in the configuration set.

**Defense.** Don't use `(t/in? env :history-node-id)` for any diagnostic purpose. To check "did history just fire," look at the configuration's atomic leaf — if it matches the recorded-or-default target, history worked. To check "is history *defined*," inspect the chart definition rather than the runtime configuration.

---

### 6. Multiple history nodes in the same compound — supported but rarely useful

**What you'll see.** You add two history nodes to one compound state — one shallow, one deep — thinking you can mix-and-match restoration depending on context. The chart loads. Both history nodes get IDs. Both can be targeted by transitions.

**What's really happening.** The library accepts multiple history nodes per compound. Each one records on every exit; transitions can target whichever one they want. This *might* be useful in unusual workflows (e.g., "open from menu = shallow restore" vs. "open from notification = deep restore"), but most charts use one history node per compound.

**Defense.** One history node per compound is the idiomatic pattern. If you find yourself wanting two, ask whether the chart is doing too much in one place — sometimes the cleaner design is to factor the compound into two parents with their own histories.

---

### 7. History inside a parallel region works (verified) — but watch for event-name prefix collisions

**What you'll see.** You add a history node inside one region of a parallel. After exiting and re-entering via the history target, you expect the chart to restore the region's last state. Whether it actually does depends on whether your event names accidentally collide via prefix matching.

**What's really happening.** History inside a parallel region works mechanically the same as in any compound state — the region records its last-active descendant when the parallel exits, and restores it when targeted by ID on re-entry[^pE].

But: in a chart with multiple entry-point transitions like

```clojure
(transition {:event :enter   :target :par})        ; enter parallel directly (skip history)
(transition {:event :enter-h :target :history-id}) ; enter via history
```

…the ex02 string-prefix matching footgun (gotcha #1 in that exercise) strikes again: when you fire `:enter-h`, the runtime's `name-match?` checks the first transition's `:event :enter` against the incoming `:enter-h` — and via the **undocumented string-prefix fallback**, `:enter` matches `:enter-h` (because `":enter-h"` starts with `":enter"`). The chart fires the *first* matching transition (in document order), which targets `:par` directly, **skipping history entirely**.

The symptom: history *records* on exit (you can verify `history-value` in the chart's working memory) but never gets *restored* on re-entry. The history mechanism isn't broken; the wrong transition is firing.

**Defense.** Two practices:

1. **Name your events such that no event name is a string-prefix of another.** `:enter-par` and `:enter-h` instead of `:enter` and `:enter-h`. Or `:enter` and `:resume` instead of `:enter` and `:enter-h`. Anything that doesn't have a shared string-prefix.
2. **When debugging "history isn't restoring,"** check whether some *other* transition is matching your re-entry event via prefix. The diagnostic: `(events/name-match? <other-transition's-event> <your-event>)` — if this returns true for any other transition on the same state, that one is firing instead.

**Empirical confirmation:**

- Chart with `:enter-par` (target `:par`) + `:enter-h` (target `:h-r1`): history restores correctly[^pE].
- Same chart with `:enter` (target `:par`) + `:enter-h` (target `:h-r1`): history does *not* restore; `:enter-h` fires the `:enter` transition due to prefix matching.

This is a cross-module footgun — ex02's gotcha bites in real charts with multiple transitions whenever event names overlap by string-prefix. The mitigation belongs in ex02 (don't name events that way), but it surfaces visibly here.

*See also:* [ex02 gotchas.md #1](../ex02-event-matching/gotchas.md) (the original string-prefix-fallback footgun).

---

### 8. Forgetting `:initial` on a parent and accidentally starting in the history-bearing compound

**What you'll see.** You declare `:settings` first in `:app` (because the history node lives there and it feels like the "main" content). You forget `:initial :main-screen` on `:app`. The chart starts at `:settings` (document-order first), opens the settings panel immediately on chart construction, and lands at `:general` (settings's default initial). The test fails on block 1 because `(t/in? env :main-screen)` is `false`.

**What's really happening.** Document order picks the first child if no `:initial` is given. This rule from ex01 still applies. The history node doesn't change anything about chart startup — it only kicks in when an explicit history-targeting transition fires.

**Defense.** When declaring a parent compound with multiple children where the "entry point" isn't the first one in your source ordering, add `:initial` explicitly. Either reorder the children to put the initial first, or use `:initial :the-initial-state` on the parent. Document order is the default, not always the intent.

---

> **Verified empirically (probes run during ex06 authoring + review pass):**
>
> - [^p3]: Shallow history restores immediate child but loses sub-state in nested compounds
> - [^p4]: Targeting compound directly skips history; lands at default initial
> - [^p5]: History snapshots at exit time, not on intermediate transitions
> - [^p6]: `history` element requires default-target argument; missing throws at construction
> - [^pE]: History inside a parallel region works correctly when event names don't collide via prefix; observed restoration of `:b` in `:r1` after parallel exit + history-target re-entry. The previous probe that *appeared* to show "history doesn't work in parallel" was actually the ex02 string-prefix matching fallback firing the wrong transition.
