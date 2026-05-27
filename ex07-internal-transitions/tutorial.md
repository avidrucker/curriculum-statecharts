# Tutorial — ex07: Internal vs External Transitions

The conceptual narrative behind exercise 7. After reading, open [`runbook.md`](./runbook.md) to build the chart and watch the tests go green.

ex07 is a "typical" density module — the chart is structurally simple (one compound state, two children), but the new mechanic (`:type :internal`) has surprisingly narrow applicability. Knowing exactly when it matters is what the exercise teaches.

---

## The story we're building

A document editor. When the user opens it, the chart calls `load-doc!`. While they work, they switch between `:viewing` and `:editing` modes. When they're done, the chart calls `save-doc!` and closes.

The key requirement: **switching between `:viewing` and `:editing` should NOT trigger `load-doc!` or `save-doc!`**. The document is loaded once at open, saved once at close — not re-loaded every time the user toggles between view/edit modes.

The straightforward way to model this is a compound state `:editor` with on-entry calling `load-doc!`, on-exit calling `save-doc!`, and two atomic children for the two modes:

```clojure
(state {:id :editor}
  (on-entry {} (script {:expr load-doc!}))
  (on-exit  {} (script {:expr save-doc!}))
  (state {:id :viewing})
  (state {:id :editing}))
```

But here's the catch: how the chart transitions between `:viewing` and `:editing` matters. **External transitions exit and re-enter `:editor`**, re-firing on-entry/on-exit each time. **Internal transitions don't.**

What you'll have grounded by the end:

1. **`:type :external` (default)** — transitions exit the source state and re-enter the target's path, even when both are inside the same compound.
2. **`:type :internal`** — for transitions declared on a compound state with descendants as targets, the compound state itself stays "in" — its on-entry/on-exit don't fire.
3. **When internal matters and when it doesn't** — internal is only meaningful when the source is a compound state and the target is a strict descendant. Other cases are no-ops.
4. **Why the editor's design puts the switching transitions on the parent** — declaring them on the parent (with `:type :internal`) is what makes the no-re-fire behavior work.

---

## Step 1 — External is the default

Every `transition` element has a `:type` key. By default — when you omit `:type` — the library treats the transition as `:external`[^p3]. From `elements.cljc`:

```clojure
(transition {:event :edit :target :editing})
;; equivalent to:
(transition {:event :edit :target :editing :type :external})
```

For most transitions, external is what you want. ex01–ex06 used external (implicitly) without any of the curriculum's tests caring about on-entry/on-exit timing.

ex07 is the first exercise where on-entry/on-exit side effects matter — and where the external/internal distinction becomes load-bearing.

---

## Step 2 — External transitions on a compound state exit and re-enter the compound

Consider this chart with the editor pattern, but using *external* transitions:

```clojure
(state {:id :editor}
  (on-entry {} (script {:expr load-doc!}))     ; fires on every entry
  (on-exit  {} (script {:expr save-doc!}))     ; fires on every exit
  (transition {:event :edit :target :editing}) ; default :external
  (state {:id :viewing})
  (state {:id :editing}))
```

When the user fires `:edit` while in `:viewing`, the runtime's transition-firing algorithm computes the **exit set** (states to exit) and the **entry set** (states to enter). For an external transition declared on `:editor` with target `:editing` (a descendant), the LCA (Least Common Ancestor) is *above* `:editor` — so the exit set *includes `:editor` itself*.

The sequence:

1. Chart in `#{:editor :viewing}`.
2. `:edit` arrives. Runtime matches the external transition on `:editor`.
3. Exit set: `[:viewing :editor]`. On-exit of `:editor` fires (`save-doc!` runs). On-exit of `:viewing` fires (no script).
4. Entry set: `[:editor :editing]`. On-entry of `:editor` fires (`load-doc!` runs again). On-entry of `:editing` fires (no script).
5. Final: `#{:editor :editing}`. **But `save-doc!` and `load-doc!` both ran.**

Empirically[^p2]: with the chart using external transitions, switching `:edit → :preview → :done` produces `entries=3, exits=3` — the parent compound re-fires on every tab switch.

This is wrong for the editor: the document shouldn't be saved and re-loaded every time the user toggles modes. Hence `:type :internal`.

> **Common misconception** — "External transitions only exit the source state, not its ancestors."
>
> For transitions declared *on* a compound state, the source *is* the compound. External transitions out of it exit the compound itself. This is the most counter-intuitive piece of SCXML semantics, and it's why `:type :internal` exists.

---

## Step 3 — `:type :internal` keeps the compound active

The fix: declare the transitions with `:type :internal`:

```clojure
(state {:id :editor}
  (on-entry {} (script {:expr load-doc!}))
  (on-exit  {} (script {:expr save-doc!}))
  (transition {:event :edit    :target :editing :type :internal})
  (transition {:event :preview :target :viewing :type :internal})
  (transition {:event :done    :target :closed})  ; default external — exits :editor
  (state {:id :viewing})
  (state {:id :editing}))
```

With internal transitions on `:editor`, the exit set for `:edit` is just `:viewing` (not `:editor`). On-entry/on-exit of `:editor` don't fire. The chart efficiently swaps the active leaf without re-running side effects.

Empirically[^p1]: with internal transitions, switching through `:edit → :preview → :done` produces `entries=1, exits=1` (only `:done` triggers the parent's exit; the internal transitions leave the parent alone).

The `:done` transition stays external — it targets `:closed`, which is outside `:editor` — so the parent *should* exit (saving the document). Internal would be incorrect for `:done`: targeting outside the compound, internal is treated like external anyway (see Step 5), but you should still write `:type :external` (or omit it) to express intent.

---

## Step 4 — Internal only matters when source is compound AND target is a descendant

The narrow applicability of `:type :internal`:

| Source | Target | Internal makes a difference? |
| --- | --- | --- |
| Compound state | Descendant of source | **Yes** — internal keeps source from exiting/re-entering |
| Compound state | Sibling (or any non-descendant) | **No** — internal is silently treated as external[^p4] |
| Atomic state | Sibling | **No** — atomic can't have descendants; LCA is the parent |
| Any state | Itself (self-target) | **No** — internal self-target still exits and re-enters[^p6] |

The exercise's chart hits the meaningful case: source `:editor` is compound, targets `:viewing` and `:editing` are descendants. Internal works as advertised.

> **Common misconception** — "I should always use `:type :internal` to be safe."
>
> No. For transitions that *should* exit the source state (e.g., a transition from inside a flow to a separate state), external is the correct semantics. Use internal *only* when you specifically don't want the source state's on-entry/on-exit to re-run. The default is sane for most cases.

---

## Step 5 — Internal with a non-descendant target is a no-op (not an error)

If you write `:type :internal` on a transition whose target is *not* a descendant of the source, **the library silently treats it as external**[^p4]:

```clojure
(state {:id :editor}
  (on-entry {} (script {:expr load-doc!}))
  ;; Internal, but target :closed is OUTSIDE :editor
  (transition {:event :go :target :closed :type :internal})
  (state {:id :viewing}))
(state {:id :closed})
```

When `:go` fires, the chart exits `:editor` and enters `:closed`. The `:type :internal` is essentially ignored — the LCA-based exit-set computation doesn't include `:editor` in the *internal* form, but since `:closed` isn't a descendant of `:editor`, *external* would also exit `:editor`. Either way, the result is the same.

No error. No warning. The annotation is silently meaningless.

**Defense:** verify your transition's `:type` matters by reasoning about source/target relationships. If you're unsure whether internal applies, omit it (let it default to external) — then add it only when on-entry/on-exit side effects on the source compound state are firing when you don't want them to.

---

## Step 6 — Where the transition is *declared* matters too

The exercise puts the switching transitions on `:editor` itself, not on `:viewing` or `:editing`. This isn't accidental.

If the transitions were declared on the children:

```clojure
(state {:id :editor}
  (on-entry {} (script {:expr load-doc!}))
  (state {:id :viewing}
    (transition {:event :edit :target :editing :type :internal}))    ; on :viewing, not :editor
  (state {:id :editing}))
```

The transition's source is now `:viewing`, not `:editor`. The LCA of `:viewing` and `:editing` is `:editor`. Per SCXML rules, an external transition's exit set is "descendants of LCA, including the source." That includes `:viewing` but NOT `:editor` (the LCA itself isn't in the exit set).

So even with `:type :external` on a child-declared transition, `:editor` doesn't exit/re-enter. The `:type :internal` is *redundant* — the behavior would be identical without it[^p5].

**The takeaway: `:type :internal` matters when the transition is declared on the compound state whose on-entry/on-exit you don't want to re-fire.** Declaring the transition on a child instead gets the same effect without needing the annotation.

This is one of those "two ways to express the same intent" situations — both are valid; the exercise picks the parent-declared form because it's also more terse (one transition on the parent vs. one per child).

---

## Step 7 — Putting it together

The chart, in plain English:

> An editor that loads a document on open and saves on close. The user can switch between `:viewing` and `:editing` without triggering load or save — those happen only at the editor's outer boundaries.

The chart, as a tree:

```
:editor (compound)
  ├─ on-entry: load-doc!
  ├─ on-exit:  save-doc!
  ├─ on :edit    → :editing  (type :internal — parent NOT exited)
  ├─ on :preview → :viewing  (type :internal — parent NOT exited)
  ├─ on :done    → :closed   (default external — parent IS exited)
  ├─ :viewing (initial)
  └─ :editing
:closed
```

The chart, as Clojure (the solution):

```clojure
(def editor-chart
  (statechart {}
    (state {:id :editor}
      (on-entry {} (script {:expr load-doc!}))
      (on-exit  {} (script {:expr save-doc!}))
      (transition {:event :edit    :target :editing :type :internal})
      (transition {:event :preview :target :viewing :type :internal})
      (transition {:event :done    :target :closed})
      (state {:id :viewing})
      (state {:id :editing}))
    (state {:id :closed})))
```

The runbook walks you through building this incrementally.

---

## What you should be able to explain

When you finish the runbook and the tests pass, you should be able to explain — out loud, in plain English — each of these:

- **What `:type :external` (the default) does for transitions declared *on* a compound state.** Exit the compound, enter the target's path, re-fire on-entry/on-exit on the way.
- **What `:type :internal` does in the same setup.** Skip the compound's exit/re-entry; just switch the active descendant.
- **Why internal is only meaningful for compound source + descendant target.** Other cases — non-descendant target, self-target, child-declared transition — don't trigger the compound to exit anyway, so internal has no effect.
- **The cost of using external where internal was intended.** On-entry/on-exit side effects on the parent fire on every transition. For the editor: load and save run on every tab switch.
- **Why `:done` stays external.** Its target `:closed` is outside `:editor` — the parent should exit (and `save-doc!` should fire) when the user closes the editor.

If any of these still feel hazy after the runbook, [`glossary.md`](./glossary.md) has the entries to revisit. Then [`quiz.md`](./quiz.md) checks recall, and [`gotchas.md`](./gotchas.md) collects the silent footguns.

---

> **Verified empirically (probes run during ex07 authoring):**
>
> - [^p1]: `:type :internal` with source = compound state, target = descendant — `entries=1, exits=1` across `:edit, :preview, :done` (only `:done` triggers parent exit)
> - [^p2]: `:type :external` (or default) — `entries=3, exits=3` across the same sequence; parent re-fires on every transition
> - [^p3]: When `:type` is omitted, the library defaults to `:external` (verified by counter equivalence)
> - [^p4]: `:type :internal` with a target OUTSIDE the source's descendant chain still exits the source (annotation silently ignored)
> - [^p5]: A transition declared on a *child* state (not the parent compound) doesn't trigger the parent's exit/re-entry regardless of `:type` — `:internal` is redundant in this case
> - [^p6]: `:type :internal` on a self-target transition still causes the state to exit and re-enter (annotation silently ignored)
