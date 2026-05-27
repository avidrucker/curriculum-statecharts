# Glossary — ex02: Hierarchical Event Names & Prefix Matching

Terms introduced (or first deeply covered) in ex02. Cross-cutting vocabulary — [[statechart]], [[state]], [[transition]], [[event]], [[configuration]], [[document-order]] — lives in the top-level [`glossary.md`](../glossary.md).

Each entry has a 0–9 criticality rating (see the top-level glossary for the scale).

---

### event-name segments  *(criticality: 9)*

Event names are split into segments on the `.` character. `:error.critical` is two segments: `["error", "critical"]`. `:user.logged-in.via-sso` is three: `["user", "logged-in", "via-sso"]`.

Hyphens (`-`) are **not** separators — they stay inside a segment. `:logged-in` is one segment; `:logged.in` is two. This matters because the library's prefix matcher compares segment vectors, not raw strings (mostly — see [[string-prefix-fallback]] for the exception).

You write segmented event names exactly like any other keyword. Clojure's reader doesn't care about dots inside the name part of a keyword. The segmentation is purely semantic, applied by the library at match time.

### prefix matching  *(criticality: 9)*

The rule the library uses to decide whether a transition's `:event` matches an incoming event. A candidate's segments must be a *prefix* of the event's segments — that is, equal to the first `N` segments of the event, where `N` is the candidate's segment count.

| Candidate | Event | Match? |
| --- | --- | --- |
| `:error` | `:error` | ✓ (1 segment, equal) |
| `:error` | `:error.critical` | ✓ (1 of 2 segments matches) |
| `:error` | `:error.foo.bar` | ✓ (1 of 3) |
| `:error.critical` | `:error.critical` | ✓ (exact) |
| `:error.critical` | `:error` | ✗ (candidate is longer than event) |
| `:warning` | `:error.critical` | ✗ (first segment differs) |

Implemented in `com.fulcrologic.statecharts.events/name-match?`. The function takes the candidate first and the event second — order matters.

### event wildcard (`.*`)  *(criticality: 4)*

The library treats `:error.*` as a notational synonym for `:error`. Internally, the matcher strips a trailing `.*` from the candidate before comparing segments. So:

```clojure
(name-match? :error.*  :error.x)   ; => true
(name-match? :error    :error.x)   ; => true   (same result)
```

The wildcard is purely a readability convenience — "this is a prefix matcher" is sometimes clearer with `.*` than without. Use whichever reads better in context.

### ancestor walk  *(criticality: 9)*

The transition-selection algorithm. When an event arrives, the runtime checks transitions in this order:

1. The transitions declared on the currently-active deepest state, in [[document-order]].
2. If none match (or none has a passing guard), the transitions on the state's parent, in document order.
3. Continue walking up until either a match fires or the chart root is reached with no match (in which case the event is dropped).

This is what "more specific handlers take priority" means: closer-to-the-active-state handlers get first dibs. In ex02, the `:error.critical` transition inside `:working` fires before the catch-all `:error` on `:operational` ever gets consulted.

The ancestor walk is *not* a "pick the most specific match" algorithm. The runtime fires the **first** matching transition it encounters. Specificity emerges from *placement* (which state you put the transition in), not from any comparison of event-name lengths.

### catch-all handler  *(criticality: 6)*

A transition whose `:event` is a short prefix (`:error`, `:user`, or just no `:event` at all for "any event"). Placed at a higher level in the state hierarchy than the specific handlers it backs up.

The ex02 chart's `:event :error → :error-display` transition on `:operational` is the canonical example. It catches every `:error.*` event that `:working`'s specific handlers don't handle.

Catch-all handlers are powerful and dangerous in equal measure. The danger: a too-broad catch-all that's placed *too deep* in the hierarchy can shadow more-specific handlers. Runbook "Try breaking it" #3 demonstrates the failure mode.

### compound-state in configuration  *(criticality: 8)*

ex01 established that a chart in `:red` has configuration `#{:red}` — one element. ex02 introduces nesting, and the configuration set grows accordingly.

When the chart is "in" `:working` (a child of `:operational`), the configuration set is `#{:operational :working}`. Both elements appear; `(t/in? env :operational)` and `(t/in? env :working)` both return `true`.

This generalizes: the configuration set contains every active state along the active path, from the deepest atomic leaf up to (but not including) `:ROOT`. The compound parents are "in" because the chart, when running, is operating in their context — events declared on those parents are still eligible to match.

### `name-match?` (the function)  *(criticality: 6)*

The library's exported event-matching function. Lives in `com.fulcrologic.statecharts.events`. Signature:

```clojure
(name-match? candidate event-or-name) => boolean
```

`candidate` is either a single keyword or a vector of keywords (the value of a transition's `:event` key — note that `:event` accepts either form per the `transition` element docstring). `event-or-name` is either an event keyword or an event map containing `::sc/event-name`.

Useful to call directly in the REPL when a transition isn't firing and you can't tell whether the matcher is the problem. See runbook Probes 2, 3, 4.

### namespaced events (strict matching)  *(criticality: 5)*

When the incoming event's name has a namespace (e.g., `:my/error`), the matcher requires the candidate to share that exact namespace.

| Candidate | Event | Match? |
| --- | --- | --- |
| `:my/error` | `:my/error` | ✓ |
| `:my/error` | `:my/error.x` | ✓ (segment prefix within `my` ns) |
| `:my/error` | `:other/error` | ✗ (namespaces differ) |
| `:error` (no ns) | `:my/error` (ns) | ✗ (candidate has no ns, event does) |

This is the library's extension of SCXML event naming. It exists to prevent cross-wiring: two subsystems can use `:auth/error` and `:billing/error` without their transitions tangling.

### string-prefix fallback (footgun)  *(criticality: 5)*

A behavior of `name-match?` that the docstring doesn't advertise: for candidates **without a namespace**, after the segment-prefix check fails, the matcher falls back to a raw string-prefix check on the keyword's string representation.

Consequences:

| Candidate | Event | Match? | Why |
| --- | --- | --- | --- |
| `:error` | `:errors` | ✓ | string `":errors"` starts with `":error"` |
| `:error` | `:error-thing` | ✓ | string `":error-thing"` starts with `":error"` |
| `:error.crit` | `:error.critical` | ✓ | string `":error.critical"` starts with `":error.crit"` |
| `:error` | `:erroneous` | ✗ | `":erroneous"` doesn't start with `":error"` (diverges at character 5) |

**Defense:** don't name distinct events such that one is a string-prefix of another (other than via clean segment splits). `:error` and `:error.critical` are fine (segment split). `:error` and `:errors` are *not* fine (plural-s creates a string-prefix collision). `:user.created` and `:users.created` are *not* fine for the same reason.

In ex02 the test events (`:error.critical`, `:error.recoverable`, `:error.unknown`) all segment cleanly off `:error`, so the footgun doesn't bite. Knowing about it now prevents an hour of confusion later.

### the `:initialNN` auto-element (revisited)  *(criticality: 2)*

In ex01 the per-exercise glossary noted that the `statechart` factory auto-inserts an `initial` element pointing at the first child in [[document-order]]. ex02's chart has the same machinery applied to two levels: one auto-initial at the chart root (pointing at `:operational`), and one auto-initial inside `:operational` (pointing at `:working`).

You'll see these IDs (e.g., `:initial17446`, `:initial17447`) if you `pprint` the chart. They're library bookkeeping; they never appear in the configuration set, and you never reference them directly.
