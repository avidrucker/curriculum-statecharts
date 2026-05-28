# Executive Summary — ex07: Internal vs External Transitions

The 5-minute refresher. Use this when you've completed ex07 once and want to re-anchor.

---

## The one big idea

**`:type :internal` on a transition declared on a compound state with a descendant target keeps the source state ACTIVE — its on-exit/on-entry don't fire.** External (the default) exits the compound and re-enters, re-running on-entry/on-exit.

The honest second lesson: **`:type :internal` is only meaningful in one specific structural case** — source must be a compound state, target must be a strict descendant. In every other case (target not a descendant, transition declared on a child, self-targeting), the library silently treats internal as external.

---

## What we built

A document editor with side-effecting on-entry (load) and on-exit (save). Switching between `:viewing` and `:editing` modes uses internal transitions to avoid re-loading/re-saving on every toggle. Exiting via `:done` uses external (default), which DOES trigger the save.

```clojure
(def editor-chart
  (statechart {}
    (state {:id :editor}
      (on-entry {} (script {:expr load-doc!}))                                  ; fires once on entry
      (on-exit  {} (script {:expr save-doc!}))                                  ; fires once on exit
      (transition {:event :edit    :target :editing  :type :internal})          ; stays in :editor
      (transition {:event :preview :target :viewing  :type :internal})          ; stays in :editor
      (transition {:event :done    :target :closed})                            ; external — exits :editor
      (state {:id :viewing})
      (state {:id :editing}))
    (state {:id :closed})))
```

The empirical contrast: with internal, `load-doc!` runs once at the start; with external (default), it runs on EVERY tab switch.

---

## Mechanics in 30 seconds

- **`:type :external`** (default): the compound source exits and re-enters on every transition. on-entry/on-exit re-fire.
- **`:type :internal`** with source=compound AND target=descendant: the source stays active; on-entry/on-exit don't fire.
- **Default `:type` is `:external`.** Omitting `:type` is the same as writing `:type :external`.
- **Internal is silently a no-op** in three cases:
  - Target outside the source's descendant chain (treated as external)
  - Self-target (still exits and re-enters)
  - Transition declared on a child state targeting a sibling (the LCA is already the parent; external would also keep parent intact)
- **Where the transition is DECLARED matters.** A transition on `:editor` with descendant target = compound exits, internal saves it. A transition on `:viewing` targeting `:editing` already keeps `:editor` intact (LCA semantics).
- **For "update data, stay put" use a TARGETLESS transition** (no `:target`, from ex03b). Internal self-target still re-enters; targetless doesn't.

---

## Composes with

- **ex03b's on-entry / script** — ex07 is about controlling *when* on-entry/on-exit fire on parents. Internal transitions suppress the re-fire.
- **ex02's ancestor walk + LCA** — internal vs external is SCXML's LCA-based exit-set computation made tangible.
- **ex03c's `handle`** — a targetless transition. Different mechanism, similar "stay put" outcome, but for atomic states.
- **ex06's history** — history is recorded at exit time. Internal transitions don't exit the compound, so they don't update history.

---

## Gotchas to remember

- **External transitions on a compound parent re-fire on-entry/on-exit on EVERY transition.** Surprising if you expect "transitions between siblings stay in the parent." For "load once, save once" semantics, internal is the fix.
- **`:type :internal` is silently ignored when target isn't a strict descendant.** No error; no warning; behavior is just whatever external would do.
- **`:type :internal` on a self-target still re-enters the state.** For "stay put + update data" use a targetless transition instead.
- **The transition's DECLARATION location matters.** A `:type :internal` on a child-declared transition is redundant; the LCA already keeps the parent intact.
- **Side-effect counters in tests persist across runs** unless explicitly reset (`(reset! atom 0)` at the top of `deftest`).

Full list in [gotchas.md](./gotchas.md).

---

## Re-engage in 5 minutes

```clojure
(require '[exercises.ex07-internal-transitions :as ex07])
(require '[com.fulcrologic.statecharts.testing :as t])

(reset! ex07/load-count 0)
(reset! ex07/save-count 0)

(def env (t/new-testing-env {:statechart ex07/editor-chart
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
[@ex07/load-count @ex07/save-count]              ; → [1 0]  (on-entry fired once)

(t/run-events! env :edit :preview)
[@ex07/load-count @ex07/save-count]              ; → [1 0]  (internal — no re-fire!)

(t/run-events! env :done)
[@ex07/load-count @ex07/save-count]              ; → [1 1]  (external — exit fired)
```

If those three counter states match, ex07 is solid. Move on to [ex08 (final + done.state)](../ex08-final-and-done/) — terminal states and the auto-events they fire.
