# Concepts Map — `com.fulcrologic/statecharts`

Visual references for how the curriculum's concepts compose. Six diagrams arranged from **simplest** (where to start) to **most complete** (the full picture). GitHub renders Mermaid inline, so all diagrams should display correctly when viewing on the web.

Reading suggestions by experience level:

- **First time through?** Read diagrams 1 and 2. Skip the rest until later.
- **Mid-curriculum, refreshing context?** Diagrams 3 and 4 ground specific exercises against the broader picture.
- **Post-curriculum or debugging in production?** Diagrams 5 and 6 are the precise ones.

---

## 1. Module learning path (simple)

The dependency order. ex01 first; everything else builds on it. The dense modules (ex04, ex09) are flagged so you can budget time.

```mermaid
graph TD
    ex01[ex01: Compound States<br/>& Configuration]
    ex02[ex02: Hierarchical Event Names<br/>& Prefix Matching]
    ex03[ex03: Guards & Document Order]
    ex03b[ex03b: Data Model Operations<br/>& Event Data]
    ex03c[ex03c: Convenience Helpers<br/>on, handle, choice]
    ex04[ex04: Parallel Regions<br/>★ dense]
    ex05[ex05: Multi-Target Transitions]
    ex06[ex06: History States<br/>★ dense]
    ex07[ex07: Internal vs External]
    ex08[ex08: Finals & done.state]
    ex09[ex09: Invocations<br/>★★ densest]

    ex01 --> ex02 --> ex03
    ex03 --> ex03b --> ex03c
    ex03c --> ex04
    ex04 --> ex05
    ex05 --> ex06
    ex06 --> ex07
    ex07 --> ex08
    ex08 --> ex09

    style ex04 fill:#fef3c7
    style ex06 fill:#fef3c7
    style ex09 fill:#fecaca
```

The arrows are "what builds directly on what." You *can* read ex06 (history) before ex05 (multi-target) — they're parallel topics — but the curriculum's order keeps prior context fresh.

---

## 2. What concept lives where (simple)

A reverse lookup: "I'm stuck on a concept; which module introduces it?"

```mermaid
graph LR
    subgraph fundamentals["Fundamentals (ex01–ex03)"]
        state[state element]
        transition[transition element]
        configuration[configuration set]
        ancestor[ancestor walk]
        guard[guards / :cond]
        docorder[document order]
        datamodel[data model]
    end

    subgraph mid["Mid-level (ex03b–ex06)"]
        onentry[on-entry / on-exit]
        script[script element]
        handle[handle convenience]
        choice[choice convenience]
        parallel[parallel + regions]
        multi[multi-target<br/>:target vector]
        history[history shallow/deep]
    end

    subgraph advanced["Advanced (ex07–ex09)"]
        internal[":type :internal"]
        final[final element]
        donestate[done.state events]
        invoke[invoke element]
        doneinvoke[done.invoke events]
        childsession[child sessions]
    end

    fundamentals --> mid --> advanced

    style fundamentals fill:#dbeafe
    style mid fill:#fef3c7
    style advanced fill:#fecaca
```

Each box is a concept. Color = difficulty cluster. If you're hazy on `choice`, that's mid-level — ex03c.

---

## 3. Chart element family (simple-medium)

Every element you'll write in chart code, grouped by namespace.

```mermaid
graph TD
    chart[chart factory<br/>statechart]

    chart --> elements["com.fulcrologic.statecharts.elements"]
    chart --> convenience["com.fulcrologic.statecharts.convenience<br/>⚠ ALPHA"]

    elements --> e_state[state]
    elements --> e_transition[transition]
    elements --> e_parallel[parallel]
    elements --> e_final[final]
    elements --> e_history[history]
    elements --> e_invoke[invoke]
    elements --> e_onentry[on-entry / on-exit]
    elements --> e_script[script]

    convenience --> c_on[on]
    convenience --> c_handle[handle]
    convenience --> c_choice[choice]

    c_on -.sugar for.-> e_transition
    c_handle -.sugar for.-> e_transition
    c_choice -.sugar for.-> e_state

    style convenience stroke:#dc2626,stroke-width:2px
    style c_on stroke-dasharray: 5 5
    style c_handle stroke-dasharray: 5 5
    style c_choice stroke-dasharray: 5 5
```

**Solid lines** = part of the namespace's exports. **Dashed lines** = what the convenience helper expands to. The red border on `convenience` reflects its `"ALPHA. NOT API STABLE."` docstring — for long-lived production charts, prefer the elements namespace.

`choice` is special: it expands to a *state* (with eventless transitions inside), not a transition. That's why it's drawn under `state` in the dashed-link family.

---

## 4. Event types and where they come from (medium)

Every event the runtime can process, grouped by source.

```mermaid
graph LR
    subgraph external["External (you fire them)"]
        ext_keyword[":foo<br/>plain keyword event"]
        ext_map["{:name :foo<br/> :data {...}}<br/>event map with payload"]
    end

    subgraph internal_auto["Internal auto-events (runtime fires)"]
        auto_done_state[":done.state.&lt;parent-id&gt;<br/>final entry → parent"]
        auto_done_invoke[":done.invoke.&lt;invokeid&gt;<br/>child chart finished → parent"]
        auto_error[":error.platform<br/>invocation failed to start"]
    end

    subgraph internal_chart["Chart-fired (raise/send from chart)"]
        raise[":raise from chart code<br/>(not covered in curriculum)"]
        send[":send to delay or<br/>cross-session route"]
    end

    queue[(event queue)]

    ext_keyword --> queue
    ext_map --> queue
    auto_done_state --> queue
    auto_done_invoke --> queue
    auto_error --> queue
    raise --> queue
    send --> queue

    queue --> match[name-match? ?]
    match -- ✓ --> fire[transition fires]
    match -- ✗ --> drop[silently dropped]

    style external fill:#dbeafe
    style internal_auto fill:#fed7aa
    style internal_chart fill:#e9d5ff
```

The unifying insight: **every event flows through the same queue and matching logic**. Internal events aren't special at the matching layer — only their source differs.

Cross-reference: ex02's `name-match?` string-prefix footgun affects matching for ALL of these. ex06/ex08/ex09 gotchas track its cross-cutting damage.

---

## 5. Configuration set composition (medium-complex)

The question "what's in the configuration?" doesn't have a one-line answer — it depends on the chart's structure and how the chart was driven. This decision tree captures the rules.

```mermaid
graph TD
    Q1{Where did the chart land?}

    Q1 --> A1[Flat compound atomic leaf]
    Q1 --> A2[Nested compound chain]
    Q1 --> A3[Inside a parallel]
    Q1 --> A4[After goto-configuration!]
    Q1 --> A5[After top-level final]

    A1 --> R1["#{atomic-leaf}<br/>+ all compound ancestors below :ROOT"]
    A2 --> R2["#{leaf, parent, grandparent, ...}<br/>(not :ROOT)"]
    A3 --> R3["#{parallel-id,<br/>region1-id, region1-leaf,<br/>region2-id, region2-leaf, ...}<br/>(not :ROOT)"]
    A4 --> R4["INCLUDES :ROOT<br/>(asymmetry — see ex03 gotcha #1)"]
    A5 --> R5["#{} (empty)<br/>+ running? false"]

    note1[":ROOT is excluded from all start!-driven<br/>configurations. The exception is the test-only<br/>goto-configuration! helper."]
    note2["Sessions are siblings in the<br/>working-memory-store. A child session's<br/>states DO NOT appear in the parent's<br/>configuration. Each is its own world."]

    R4 -.-> note1
    A3 -.-> note2

    style A4 fill:#fed7aa
    style A5 fill:#fecaca
    style note1 fill:#fef3c7
    style note2 fill:#fef3c7
```

The two surprise-prone rules are highlighted (orange): the `goto-configuration!` asymmetry, and chart-exit producing an empty configuration. Both bit during the curriculum's authoring; both are now documented.

---

## 6. The cross-cutting footgun (medium)

ex02's string-prefix matching fallback in `name-match?` — and where it bites in later modules. This is the most consequential bug pattern in the curriculum.

```mermaid
graph TD
    ex02_origin[":error matches :errors<br/>via undocumented<br/>string-prefix fallback<br/>in name-match?"]

    ex02_origin --> ex06_history[ex06 — history target collision]
    ex02_origin --> ex08_donestate[ex08 — done.state.X collision]
    ex02_origin --> ex09_doneinvoke[ex09 — done.invoke.X collision]

    ex06_history --> ex06_detail[":enter matches :enter-h<br/>→ wrong transition fires<br/>→ history appears broken"]

    ex08_donestate --> ex08_detail[":done catches every<br/>:done.state.X event<br/>the runtime synthesizes"]

    ex09_doneinvoke --> ex09_detail[":done.invoke.payment matches<br/>:done.invoke.payment-process<br/>via the prefix fallback"]

    defense[Defense: never name two events<br/>where one is a string-prefix<br/>of the other.<br/><br/>Pick :enter-par + :enter-h,<br/>not :enter + :enter-h]

    ex06_detail -.-> defense
    ex08_detail -.-> defense
    ex09_detail -.-> defense

    style ex02_origin fill:#fecaca
    style defense fill:#dcfce7
```

This is why ex02's gotcha #1 is the curriculum's single most cross-referenced footgun. Once you've seen `:done` match `:done.state.processing` in your own chart and spent an hour debugging, you remember the rule forever.

The bottom line: choose event names that are **full-segment-distinct** — never one being a string-prefix of another.

---

## 7. A statechart of the statechart lifecycle (bonus, meta)

Tongue-in-cheek but instructive: the lifecycle of a running statechart, modeled *as* a statechart. Uses Mermaid's `stateDiagram-v2` syntax.

```mermaid
stateDiagram-v2
    [*] --> Constructed: (statechart {} ...)
    Constructed --> Registered: simple/register!<br/>(or auto by t/new-testing-env)
    Registered --> Running: sp/start!<br/>or (t/start! env)

    state Running {
        [*] --> Settling: initial state(s) entered
        Settling --> WaitingForEvent: on-entry actions complete
        WaitingForEvent --> ProcessingEvent: event arrives
        ProcessingEvent --> Settling: transitions fire,<br/>more events queued?

        state Settling {
            [*] --> EventlessCheck: any eventless transitions?
            EventlessCheck --> AutoFire: yes — fire them
            AutoFire --> EventlessCheck
            EventlessCheck --> [*]: no — stable
        }
    }

    Running --> Exited: top-level final entered<br/>(running? false)
    Exited --> [*]

    note right of Constructed
        Just data. No env yet.
        Re-evaluating (def) doesn't
        affect existing envs.
    end note

    note right of WaitingForEvent
        Pure idle. Configuration
        is stable. (t/in? env :x)
        queries are safe here.
    end note
```

What the meta-diagram surfaces:

- **The chart's life is broken into discrete event-processing cycles.** Between cycles, the configuration is stable and queries return consistent results.
- **The `Settling` sub-state captures the eventless-transition cascade** — the runtime keeps firing eventless transitions until no more are eligible. This is what makes choice (ex03c) "transient" — a single settling cycle passes through it.
- **Top-level final is the only way out.** A stuck chart (no transitions) stays in `Running > WaitingForEvent` indefinitely; only `Exited` distinguishes "truly done" from "stuck."

---

## How to use this document

- **First-time learning**: Read diagram 1 to pick a starting module. Read diagram 2 to know what's covered where.
- **Re-engagement**: Diagram 1 confirms the order; diagram 3 names the elements you have at your disposal.
- **Debugging your own chart**: Diagram 5 is the configuration-rules reference. Diagram 6 is the cross-module footgun cheat sheet.
- **Teaching someone else**: Diagrams 1–4 are good for whiteboarding the mental model; 5–7 are deep dives for after they've worked through 2–3 exercises themselves.

---

## When this document drifts

These diagrams are derived from the curriculum's modules and `ask_tony.md`. When you:

- Add a new module → update diagram 1 (the learning path) and diagram 2 (the concept inventory).
- Discover a new element or runtime behavior → update diagram 3, 4, or 5 depending on which.
- Add a cross-module footgun → update diagram 6.
- Find a new lifecycle wrinkle → update diagram 7.

Keep diagrams as small as the rules allow. A diagram that takes more than 30 seconds to read isn't doing its job.
