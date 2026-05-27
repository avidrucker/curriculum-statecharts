# Gotchas — ex02: Hierarchical Event Names & Prefix Matching

Silent failures, hidden behaviors, and easy-to-miss configurations the learner is likely to hit while working through (or after) ex02. See [`gotchas-overview.md`](../gotchas-overview.md) for what belongs here vs. in `runbook.md`'s "Common breakages."

Each entry verified empirically against `com.fulcrologic/statecharts` 1.2.25.

ex01's gotchas — REPL caching, missing `:id`, `:ROOT` not in configuration, `t/in?` returning false silently for typos — *still apply*. This file adds what ex02 newly puts on the table.

---

### 1. `:error` matches `:errors` — the undocumented string-prefix fallback

**What you'll see.** You define a transition `(transition {:event :error :target :error-display})` to catch any `:error.<whatever>` event. Then someone sends `:errors` (with an `s`) and the chart routes to `:error-display`. You didn't write that — the matcher caught it by accident.

**What's really happening.** `events.cljc`'s `name-match?` has two branches for non-namespaced candidates: a segment-based match (the documented behavior), and a *string-prefix fallback* for when the segment match fails. `(str/starts-with? ":errors" ":error")` is `true` — so the candidate matches the event even though `["error"]` ≠ `["errors"]` as segment vectors. The docstring calls the matcher "segment-by-segment" and doesn't mention this fallback.

**Defense.** Don't name distinct events such that one is a string-prefix of another (other than via clean dot-splits). `:error` and `:error.critical` are fine (dot-split). `:error` and `:errors` are *not* fine — they'll alias. Same for `:user` and `:users.created`, `:auth` and `:authentication.failed`. Probe at the REPL with `(events/name-match? candidate event)` before relying on any catch-all keyword.

*See also:* [tutorial Step 6](./tutorial.md), [glossary `string-prefix fallback (footgun)`](./glossary.md), [runbook Probe 3](./runbook.md).

---

### 2. `:error.crit` matches `:error.critical` (partial-segment match)

**What you'll see.** You write `(transition {:event :error.crit :target :somewhere})` intending only to catch `:error.crit`. Someone sends `:error.critical` — and your transition fires.

**What's really happening.** Same string-prefix fallback as #1, hidden inside the segment matcher. Per segments, `["error" "crit"]` is *not* a prefix of `["error" "critical"]` (since `"crit"` ≠ `"critical"`), so the segment match returns false. But then the fallback kicks in: `(str/starts-with? ":error.critical" ":error.crit")` is `true`. The transition fires.

**Defense.** Use full segment names — `:error.critical`, not `:error.crit`. Treat each segment as an atom; never write a segment that's a prefix of another segment in your event taxonomy.

---

### 3. Namespace mismatch is bidirectional — neither direction matches

**What you'll see.** You expect `:event :error` to be a generic catch-all that matches any `:error`-shaped event, including `:my/error`. Or you expect `:my/error` to be a generic catch-all for `:error`. Both fail.

```clojure
(name-match? :error  :my/error)     ; → false
(name-match? :my/error :error)      ; → false
(name-match? :my/error :other/error); → false
```

**What's really happening.** When the event has a namespace, the candidate must have the **same** namespace. When the event has no namespace, the candidate must also have no namespace. There's no "candidate without ns matches any namespace" fallback. This is the library's extension of SCXML's event naming.

**Defense.** Pick a namespacing convention per chart and stick to it. Charts at module boundaries (e.g., a Fulcro routing chart) commonly use namespaced events like `:auth/error`, `:billing/error`; charts internal to a feature commonly use unnamespaced. Don't mix the two and expect prefix matching to bridge them.

*See also:* [tutorial Step 5](./tutorial.md).

---

### 4. The compound parent is *in* the configuration alongside its active atomic child

**What you'll see.** Your chart is in `:working` (a child of `:operational`). You write a test like `(is (= :working <whatever-fetches-the-state>))` — but the configuration is actually `#{:operational :working}`, not just `:working`. If your test uses set equality, you get a confusing mismatch.

**What's really happening.** When the runtime enters a compound state, it adds *both* the compound state *and* the entered child to the `::sc/configuration` set. The chart "is in" both at the same time. `(t/in? env :operational)` and `(t/in? env :working)` both return `true`. The exception is the auto-generated chart root `:ROOT`, which is excluded.

**Defense.** Use `t/in?` (set-membership) for individual state checks, not equality on the whole configuration. If you really want to know "is the chart in `:working` and *only* `:working`," you're probably asking the wrong question — that's a property of flat charts, not nested ones.

*See also:* [tutorial Step 4](./tutorial.md), [glossary `compound-state in configuration`](./glossary.md).

---

### 5. Hyphens stay inside a segment; only dots split

**What you'll see.** You write `:event :user.logged-in` and reason about it as if `logged-in` were two segments. Then you wonder why `(name-match? :user.logged :user.logged-in)` returns `true` (per the string-prefix fallback) but `(name-match? :user.logged.in :user.logged-in)` returns `false`.

**What's really happening.** The matcher splits the keyword's name on the *dot* character only. `logged-in` is one segment. `logged.in` is two. Hyphens are part of the segment's name.

**Defense.** Pick a separator convention and stick to it. Most Fulcro projects use hyphens within a segment (`:user.logged-in`, `:auth.session-expired`); dot-separation is reserved for actual hierarchy. Don't mix.

---

### 6. `:event` accepts either a single keyword *or* a vector — vectors are logical OR

**What you'll see.** You read someone else's chart and see `(transition {:event [:error.critical :timeout] :target :shutdown})`. You're not sure what that means: both conditions? either? something else?

**What's really happening.** The library's `name-match?` (events.cljc:25) accepts a vector of candidates and checks each in turn — *any* match returns true. So `:event [:foo :bar]` matches either an incoming `:foo` *or* an incoming `:bar`. Useful for collapsing two unrelated event names into one handler.

```clojure
(name-match? [:foo :bar] :foo)       ; → true
(name-match? [:foo :bar] :bar)       ; → true
(name-match? [:foo :bar] :baz)       ; → false
```

**Defense.** Use vectors when two unrelated events should fire the same transition (e.g., `[:user-cancel :timeout]` both abandoning a flow). Use prefix matching when events share a hierarchy (`:error` for any `:error.*`). Don't combine them in confusing ways — if you write `[:error :user]`, that catches `:error*` OR `:user*`, which is two different responsibilities in one transition.

---

### 7. A `:target` referencing a nonexistent state throws "Cannot register invalid chart" — opaque

**What you'll see.** You typo `:target :recoverry` (extra `r`). When you build the chart or call `(t/new-testing-env)`, you get:

```
Execution error (ExceptionInfo) at ...
Cannot register invalid chart
```

No mention of *which* target is bad, or which transition the problem is in.

**What's really happening.** The chart's `configuration-validator` runs on construction and detects that `:recoverry` isn't an ID anywhere in the chart. It throws a generic error rather than pinpointing the bad transition.

**Defense.** When you hit "Cannot register invalid chart," grep your chart definition for `:target` and confirm every target keyword exists as an `:id` somewhere in the chart. Editor multi-cursor across all `:target :foo` to all `:id :foo` is the fastest way. (Future library improvement: the error could name the bad reference — worth filing an issue upstream.)

---

### 8. A catch-all transition in the same state as specifics works *by accident* of document order

**What you'll see.** You place the catch-all `:event :error` transition inside `:working` *after* the specific `:error.critical` and `:error.recoverable` transitions. All tests pass. You move on. A teammate refactors the chart, alphabetizes the transitions, and now the catch-all is *first* — and every `:error.*` event lands in `:error-display`, not `:shutdown` or `:recovery`.

**What's really happening.** Within a single state, transitions are tried in *document order*; the first match wins. If the specifics come before the catch-all, things work. If the catch-all comes first (because someone reordered), it eats every event before the specifics get checked. The "more specific takes priority" rule doesn't operate within a single state — it operates *across* the state hierarchy (closer states win over ancestors).

**Defense.** Place catch-alls *at a higher level in the hierarchy* than the specifics they back up. ex02's solution does this exactly right: specifics in `:working`, catch-all in `:operational`. The ancestor walk guarantees specifics win regardless of declaration order within their state. Putting everything in one state is fragile.

*See also:* [runbook 'Try breaking it' #3](./runbook.md), [glossary `ancestor walk`](./glossary.md).

---

### 9. One-way chart paths need fresh envs between scenarios

**What you'll see.** Your chart has a path that ends in a leaf with no outgoing transitions (like ex02's `:shutdown`). You write a test that drives the chart through scenario A, then tries to drive the same env through scenario B. Scenario B's events have no effect — the chart is stuck.

**What's really happening.** Once the chart reaches `:shutdown` (or any leaf-final state with no transitions), no event matches. The chart stays put. There's no error — the events are silently dropped (see ex01 gotcha #5).

**Defense.** When a chart has one-way paths, create a fresh testing env per scenario:

```clojure
(let [env (...)] (t/start! env) (t/run-events! env :error.critical) ...)
(let [env2 (...)] (t/start! env2) (t/run-events! env2 :error.recoverable) ...)
(let [env3 (...)] (t/start! env3) (t/run-events! env3 :error.unknown) ...)
```

This is exactly the pattern ex02's `deftest` uses. The cost is some `let` boilerplate; the payoff is each scenario starts cleanly and is independent.

---

### 10. `name-match?` argument order: candidate first, event second

**What you'll see.** You're debugging at the REPL. You type `(name-match? :error.critical :error)` thinking "does the event `:error.critical` match the candidate `:error`?" You get `false`. Confusion.

**What's really happening.** The function signature is `[candidate event]`. The *first* arg is what the transition declares (`:event` value); the *second* is what was fired. Reading "candidate" as "what came in" is the inversion that bites you.

```clojure
(name-match? :error :error.critical)   ; → true   (correct direction)
(name-match? :error.critical :error)   ; → false  (reversed; candidate is more specific than event)
```

**Defense.** Read the call as "(name-match? *transition's declared event* *actual event fired*)." Or memorize the prefix-match direction: the candidate must be a prefix-or-equal of the event, so the *shorter* keyword (the catch-all) goes first.

*See also:* [glossary `name-match? (the function)`](./glossary.md).
