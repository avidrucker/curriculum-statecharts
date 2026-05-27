# Tutorial — ex02: Hierarchical Event Names & Prefix Matching

The conceptual narrative behind exercise 2. After reading, open [`runbook.md`](./runbook.md) to build the chart and watch the tests go green.

---

## The story we're building

A two-tier error handler. The system is `:operational`, with a `:working` child that's the day-to-day happy path. Things go wrong; *how* wrong determines where the chart goes:

- `:error.critical` → `:shutdown` (something irrecoverable; halt)
- `:error.recoverable` → `:recovery` (a known-fixable problem; try to recover)
- `:error.<anything-else>` (e.g., `:error.network`, `:error.unknown`) → `:error-display` (we don't know what to do; show it to the user)

The specific handlers live close to where the work happens (inside `:working`). The catch-all lives one level up (on `:operational`), so it triggers only when no nearer handler claims the event.

What this exercise is really teaching you, behind the error-handler story:

1. **Event names are hierarchical.** `:error.critical` isn't just a name — it's two segments, `error` and `critical`, separated by a dot.
2. **Transition matching is *prefix* matching.** A transition that listens for `:error` will fire on `:error`, `:error.critical`, `:error.recoverable`, `:error.foo.bar.baz` — any event whose name *starts with* `error` on a segment-by-segment basis.
3. **Transition selection walks the ancestor chain.** When `:error.unknown` arrives while the chart is in `:working`, the runtime first asks `:working` "do you handle this?" When `:working` says no, it asks `:working`'s parent (`:operational`), and so on up to the root.
4. **The configuration set contains *multiple* IDs when nested.** This is the first exercise where the configuration isn't a one-element set — when the chart is in `:working`, it's *also* in `:operational`. Both appear in the configuration.

---

## Step 1 — Event names are segments, not opaque keywords

In ex01, every event was a single keyword: `:next`. The runtime didn't care about the name beyond "are these two keywords equal?"

Starting with ex02, event names earn structure. Dots inside a keyword's name are **segment separators**:

| Event keyword | Segments |
| --- | --- |
| `:next` | `["next"]` |
| `:error` | `["error"]` |
| `:error.critical` | `["error", "critical"]` |
| `:error.recoverable` | `["error", "recoverable"]` |
| `:user.logged-in.via-sso` | `["user", "logged-in", "via-sso"]` |

This is SCXML's convention. The library follows it: dots are special; hyphens aren't. (`:logged-in` is a single segment; `:logged.in` is two.)

You don't write these keywords any differently than ex01's `:next` — Clojure's reader doesn't care about dots inside a keyword's name. The hierarchy is purely semantic, applied at transition-matching time by the library's `name-match?` function.

> **Common misconception** — "Dots in keyword names are namespaces."
>
> They aren't. A *namespaced* keyword has a slash: `:user/error`. A *segmented* keyword has dots: `:user.error`. Different concepts; both are legal Clojure. The library treats them very differently — see Step 5 on namespaces.

---

## Step 2 — Transitions match on prefixes, not on exact equality

Here's the core mechanic. When a transition declares:

```clojure
(transition {:event :error :target :error-display})
```

…it doesn't just listen for the literal `:error` event. It listens for **any event whose name has `:error` as a prefix on a segment basis**:

| Incoming event | Does `:event :error` match? | Why |
| --- | --- | --- |
| `:error` | ✓ | exact match — one segment, equal |
| `:error.critical` | ✓ | segments `[error]` is a prefix of `[error critical]` |
| `:error.foo.bar` | ✓ | still a prefix |
| `:warning` | ✗ | first segment differs |
| `:other` | ✗ | no overlap |

This is why the exercise's error handler works: a single `:event :error` transition catches `:error.critical`, `:error.recoverable`, `:error.unknown`, `:error.network`, and any other future `:error.*` event we haven't named yet.

> **Common misconception** — "I need to enumerate every event name I might receive."
>
> You don't. Declare the prefix you care about and let the matcher do the rest. The point of segmented event names is that handlers compose by hierarchy: a general `:error` handler at one level coexists with specific `:error.critical` and `:error.recoverable` handlers at another.

### Wildcards: `:error.*` is the same as `:error`

The library lets you write `:event :error.*` if you find it more readable, but the `.*` is stripped before matching. It's purely a notational convenience — `:error.*` and `:error` match exactly the same set of events.

### `:event` can be a vector — that's logical OR

The `:event` key on a transition accepts either a single keyword *or* a vector of keywords. A vector is the OR of all its candidates:

```clojure
(transition {:event [:user-cancel :timeout] :target :abandoned})
```

Fires when *either* `:user-cancel` *or* `:timeout` arrives. Each candidate inside the vector follows the same prefix-matching rules from Step 2. Useful when two unrelated events should trigger the same transition without writing two transition elements.

ex02's exercise doesn't use the vector form, but it's a common idiom in real charts. (See [gotchas #6](./gotchas.md) for the trap of mixing semantically-distinct events in one vector.)

---

## Step 3 — Transition selection walks from the active state outward

When an event arrives, the runtime doesn't search the whole chart for a matching transition. It walks the **ancestor chain** of the currently-active state, from deepest to shallowest, and fires the **first matching transition it finds**.

For the error-handler chart with the configuration `#{:operational :working}`, the walk is:

1. Check `:working`'s transitions in document order. Any match?
2. If not, check `:operational`'s transitions in document order. Any match?
3. If not, check the chart root's transitions in document order. Any match?
4. If still not, drop the event.

When `:error.critical` arrives:

- Step 1: `:working` has `:error.critical → :shutdown`. Matches. **Done.** The walk stops; `:operational`'s `:error` transition is never consulted, even though it *would* have matched.

When `:error.unknown` arrives:

- Step 1: `:working`'s transitions are `:error.critical` and `:error.recoverable`. Neither matches `:error.unknown` (different segment 2).
- Step 2: `:operational` has `:error → :error-display`. The prefix `:error` matches `:error.unknown`. **Done.**

This is what "more specific handlers take priority" means in practice: closer-to-the-leaf handlers get first dibs, and the ancestor walk falls through to general handlers only when nothing closer claims the event.

> **Common misconception** — "The runtime checks every matching transition, then picks the most specific one."
>
> It doesn't. It walks the ancestor chain and fires the *first* match. Specificity emerges from *placement* (where you put the transition in the chart), not from the matcher comparing event-name lengths.

### Document order is the tiebreaker

If a single state has multiple transitions that all match the same event, the *first one in document order* wins. This is the same rule from ex01's initial-state selection, applied to transitions. For ex02 each state has at most one matching transition per event, so document order doesn't come up — but it's the rule you'll need in ex03 (guards) where two transitions in the same state both listen for `:submit`.

---

## Step 4 — Compound states change the configuration

ex01's configuration was a one-element set: `#{:red}` or `#{:green}` or `#{:yellow}`. Three flat states, exactly one active at a time.

ex02 introduces nesting. The chart structure:

```
:operational (compound state, contains children)
  :working      ← initial child
  :recovery
  :error-display
:shutdown       (sibling of :operational at the top level)
```

When `(t/start! env)` runs, the chart enters `:operational` (the first top-level state). Entering a compound state means *also* entering its initial child — `:working` (first in document order, no `:initial` declared). So the configuration becomes:

```
#{:operational :working}
```

Two elements. **Both states are active.**

After `:error.critical` fires and the chart transitions to `:shutdown`:

```
#{:shutdown}
```

One element. `:shutdown` is a sibling of `:operational`, not nested inside anything, so the compound is no longer active.

This is why `(t/in? env :operational)` returns `true` when the chart is in `:working`: `:operational` is literally in the configuration set, alongside its active child. `t/in?` is still the same `(contains? configuration state-name)` check from ex01; it's just that the set has more members now.

> **Common misconception** — "The chart is 'in' the deepest state."
>
> The chart is in *every* state along the active path: the active leaf, every compound ancestor up to but excluding `:ROOT`. For nested charts, `(t/in? env <some-compound>)` is just as likely to return true as `(t/in? env <the-active-atomic>)`.

---

## Step 5 — Namespaces are strict (briefly)

The exercise itself doesn't use namespaced events, but you'll encounter them in real Fulcro apps, so the rule is worth knowing:

When the event name has a namespace (e.g., `:user/error`), the candidate must have the **same namespace** to match. The library extends SCXML here:

| Candidate | Event | Match? |
| --- | --- | --- |
| `:my/error` | `:my/error.x` | ✓ |
| `:my/error` | `:other/error` | ✗ |
| `:error` (no ns) | `:my/error` (ns) | ✗ |

A candidate without a namespace will *not* match an event with a namespace. This is a guardrail against accidentally cross-wiring two unrelated subsystems that happen to share an event name.

---

## Step 6 — One footgun worth knowing about

The library's docstring describes matching as "segment-by-segment." In practice, there's also a **string-prefix fallback** for candidates without namespaces. This means:

| Candidate | Event | Match? | Why |
| --- | --- | --- | --- |
| `:error` | `:errors` | **✓** | string `":errors"` starts with `":error"` |
| `:error` | `:error-thing` | **✓** | string `":error-thing"` starts with `":error"` |
| `:error.crit` | `:error.critical` | **✓** | string `":error.critical"` starts with `":error.crit"` |
| `:error` | `:erroneous` | ✗ | `":erroneous"` doesn't start with `":error"` (5th char diverges) |

The first three rows are surprising — they're not segment matches, they're raw string prefix matches that the docstring doesn't advertise.

**What this means for you:** don't name two distinct events with one as a string-prefix of the other unless you intend them to be aliases. `:error` and `:errors` will both be caught by a single `:event :error` transition. `:user` and `:user.logged-in` are fine because they segment cleanly; `:user` and `:users` are *not* fine.

In ex02 the test event names (`:error.critical`, `:error.recoverable`, `:error.unknown`) are all segment-clean, so the footgun doesn't bite. It's named here so you remember it before it bites you in a real codebase.

> **Common misconception** — "A candidate without a namespace only does segment matching."
>
> It tries segment matching first, then falls back to raw string prefix matching. The fallback is what causes the `:error` matches `:errors` surprise.

---

## Step 7 — Putting it together

The chart, expressed in plain English:

> The system is *operational* by default, doing its *working* job. If the work hits a *critical* error, shut down. If the work hits a *recoverable* error, switch to recovery mode. If the work hits any *other* kind of error, surface it on the error display.

The chart, expressed as a tree:

```
:operational
  ├─ on :error → :error-display
  ├─ :working (initial)
  │    ├─ on :error.critical → :shutdown
  │    └─ on :error.recoverable → :recovery
  ├─ :recovery
  └─ :error-display
:shutdown
```

The chart, expressed in Clojure (the solution structure):

```clojure
(def error-handler
  (statechart {}
    (state {:id :operational}
      (state {:id :working}
        (transition {:event :error.critical :target :shutdown})
        (transition {:event :error.recoverable :target :recovery}))
      (state {:id :recovery})
      (state {:id :error-display})
      (transition {:event :error :target :error-display}))
    (state {:id :shutdown})))
```

The runbook walks you through building this incrementally and verifying each layer.

---

## What you should be able to explain

When you finish the runbook and the tests pass, you should be able to explain — out loud, in plain English — each of these:

- **Why `:error.critical` ends up at `:shutdown` and not `:error-display`.** Ancestor walk: `:working`'s `:error.critical` transition matches first; `:operational`'s `:error` is never consulted.
- **Why `:error.unknown` ends up at `:error-display`.** Ancestor walk: `:working` has no match; `:operational`'s `:error` matches by prefix.
- **What's in the configuration when the chart is in `:working`.** `#{:operational :working}` — both the compound parent and the active atomic child.
- **Whether `:error` would match `:errors`.** Surprisingly, yes — the string-prefix fallback. Worth flagging in your future code.
- **How `:my/error` and `:error` differ for matching purposes.** Namespaces must match exactly; `:error` (no ns) does *not* match `:my/error` (ns).

If any of these still feel hazy after the runbook, [`glossary.md`](./glossary.md) has the entries to revisit. Then the [`quiz.md`](./quiz.md) checks recall.
