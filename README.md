# Writing VocalBridge Agents — A Field Guide

How to plan, structure, and write a VocalBridge (VB) voice agent that works in
production. It is opinionated on purpose: every rule here is one that voice agents tend
to learn the hard way.

This guide teaches *patterns*, not products. Examples are generic and abstract — swap in
your own domain.

---

## 0. The mental model (read this before you write a word)

A VB agent is **not a chatbot with a voice**. It is a real-time controller sitting
between three things:

```
        INPUTS                          THE AGENT                        OUTPUTS
  ┌──────────────────┐         ╔═══════════════════════╗        ┌──────────────────────┐
  │  user's voice    ├────────►║                       ║───────►│  speech  (TTS)       │
  ├──────────────────┤         ║                       ║        ├──────────────────────┤
  │  app context in  ├────────►║   reason · decide ·   ║───────►│  client actions      │
  ├──────────────────┤         ║       respond         ║        │  (drive screen / app)│
  │  client actions  ├────────►║                       ║        └──────────────────────┘
  │  (events in)     │         ╚════════════╦══════════╝
  └──────────────────┘                      ║
                                 ┌──────────╨────────────┐
                                 │  tools  (MCP / API /  │   ◄─ optional, only when wired
                                 │  AI-agent delegation) │
                                 └───────────────────────┘
```

Three streams, not one. Speech is only one of the agent's outputs; for most non-trivial
agents the more important output is **client actions** — structured commands that change
what's on screen or in the app. The single biggest beginner mistake is writing a prompt
that only thinks about *what the agent says* and forgets *what the agent does*.

Before writing anything, answer four questions. They determine ~80% of the config:

1. **What interaction model is this?** Reactive support (silent until needed) ·
   autonomous driver (owns an arc) · structured planner (phased conversation) ·
   dual-stream (drives the screen *and* handles a second channel like live chat). This
   decides `mode`, persona, and the entire shape of the prompt.
2. **What does the agent control?** This is your client-action surface. Enumerate it
   before writing prose.
3. **How does context reach the agent each turn?** Via `app_to_agent` actions
   ("context-in"). Define the schema of what the app sends.
4. **What must the agent *never* do?** Voice agents fail loudly and in public, and you
   can't un-say something over TTS. Your banned-phrase / hard-rule list is as important
   as your instructions.

If you can't answer these crisply, you're still designing the agent, not writing it.
Don't skip to prose.

---

## 1. The config anatomy

A VB config is one JSON object. The fields you actually author:

| Field | What it is | Notes |
|---|---|---|
| `name` | Agent identity | — |
| `mode` | The runtime | `openai_concierge` (realtime) or `cascaded_concierge` (STT→LLM→TTS). See §2. |
| `style` | Persona dial | `Chatty` (warm, conversational) or `Focused` (efficient, professional). Behavioral, not cosmetic. |
| `greeting` | First spoken line | Often `null`. See §4. |
| `custom_prompt` | The system prompt | The bulk of the work. §5–§9 dissect it. |
| `deploy_targets` | Surface | `web`, phone, etc. |
| `background_enabled` | Process while user/media is active | Usually `true` for live agents. |
| `web_search_enabled` | Can search the web | `true` only when grounding in live facts matters. |
| `hold_enabled` / `hangup_enabled` | Phone-call affordances | Typically `false` for web agents (exception: an agent that should end its own session). |
| `builtin_tools_config.client_actions` | **The action interface** | The agent's hands. §3. |
| `model_settings` | TTS / STT / LLM / realtime blocks | Mode-dependent. Often dashboard-managed rather than CLI-pushed. |
| `mcp_servers` / `api_tools` / `ai_agent_config` | External tools | Frequently `null` — most rich agents don't need them. §10. |

> **A `null` value is a statement, not a gap.** `mcp_servers: null` means the agent
> genuinely uses no MCP tools — not that something failed to load. Don't add tool
> plumbing the agent doesn't need.

---

## 2. Choosing the mode

| | `openai_concierge` | `cascaded_concierge` |
|---|---|---|
| Pipeline | Single realtime model | STT → LLM → TTS, swappable parts |
| Latency / barge-in | Lowest; true mid-sentence interruption | Higher; better for deliberate flows |
| Choose the LLM/STT? | No (realtime model fixed) | Yes — pick a reasoning-strong LLM + a good streaming STT |
| Use it for | High-interrupt, fast reactive agents | Multi-stream or reasoning-heavy workflows |

Heuristic: **if the agent must stop talking the instant the user speaks, go
`openai_concierge`. If it runs a structured workflow or juggles two streams and can
afford a beat of latency, go `cascaded_concierge`** (and pick a strong reasoning LLM for
it).

`style` is a real behavioral dial. `Chatty` agents warm up and quip; `Focused` agents cut
pleasantries and drive. Match it to the interaction model, not to taste.

---

## 3. Client actions — design these FIRST

Client actions are the agent's only way to affect the world beyond speaking. **Design
this interface before you write the prompt.** Each action has four fields:

```json
{ "name": "render_view",
  "behavior": "respond" | "notify",
  "direction": "agent_to_app" | "app_to_agent",
  "description": "..." }
```

Two axes carry all the meaning.

### 3.1 Direction: context-in vs control-out

- **`app_to_agent` ("context-in")** — the app feeds the agent state. The agent
  *receives* these (session kickoff, periodic state updates, a typed user message, a
  "that finished" signal).
- **`agent_to_app` ("control-out")** — the agent drives the screen/app. The agent
  *emits* these (render something, highlight, advance, clear).

Describe them differently. Context-in descriptions tell the agent *how to absorb and when
to react*. Control-out descriptions tell the agent *when to fire and what the payload
means*.

### 3.2 Behavior: `respond` vs `notify`

This encodes **causality** — does this inbound action require the agent to take a turn?

- **`notify`** — absorb silently, do not speak. For ambient state: periodic position
  updates, a settings change, a piece of fetched data. *"Update your view. Do not speak
  about it."*
- **`respond`** — this requires an agent turn: session start, the user said/typed
  something, an awaited action finished.

Getting an ambient "tick" action wrong — marking a frequent state update as `respond`
instead of `notify` — is what makes an agent chatter at nothing. This single field is the
difference between a calm agent and an annoying one.

### 3.3 Payload semantics: fixed vs polymorphic

Two strategies:

- **Fixed semantics:** each action means exactly one thing (`render_view` always renders
  a view). Clear and predictable; you add one action per capability.
- **Polymorphic semantics:** a small *generic* action set (`place`, `remove`, `reveal`,
  `highlight`) whose meaning is **redefined per session by the context payload**. The
  same `remove {target, value}` means different things in different sessions; the context
  tells the agent the exact format and meaning for *this* session. This scales without
  touching the agent — a new variation is new context, not new code. Use it when you have
  many variations of one interaction shape.

### 3.4 The cardinal rules of actions

> - **Words and actions must match.** If the agent says something changed but didn't emit
>   the action, the user sees nothing. Never narrate a change you didn't actually trigger.
> - **Hide before show.** Clear old content before rendering new, or visuals overlap.
> - **Reason semantically, address exactly.** Decide *which* asset by its description, but
>   put the **exact ID** in the payload. Never let the model improvise an identifier.

---

## 4. The greeting

The greeting's length should be **inversely proportional to how much the agent should
fade into the background.**

- **`null` / immediate** — the session opens with context and the agent goes straight to
  work, no "hello, I'm…". Use when the agent is a *driver* and a greeting would just be
  dead air.
- **Minimal** (`"Hi!"`) — one syllable to establish presence without stealing focus. For
  support / co-host agents.
- **Atmospheric** — a styled, in-character opening. Only when *the drama is the product*
  (e.g. a game). A utility agent doesn't earn one.

---

## 5. Anatomy of the `custom_prompt` — the skeleton

This is the part the principles hang on. A strong VB prompt has a consistent section
order; a long prompt is this skeleton with more depth, not a different shape. Ordering
matters — **persona and prohibitions come before mechanics**, so every later instruction
is read through them.

```
1. Identity & persona            ── who the agent is
2. Voice & style rules           ── how it speaks (length, register)
3. Hard rules / prohibitions     ── the don't-list, up front and blunt
4. How context reaches you       ── the app_to_agent contract (the input schema)
5. Output protocol               ── the agent_to_app actions + any payload schemas
6. Operating loop / modes        ── the turn-by-turn cycle and mode switches
7. Worked example                ── one end-to-end interaction
```

The rest of this section walks each block with a fill-in-the-blank template and the
reasoning behind it.

### Block 1 — Identity & persona

*Purpose:* set the lens every other instruction is read through. For `Chatty` agents this
carries real behavioral weight (see §6); for `Focused` agents keep it to role + register.

```
You are <NAME>, a <ROLE> inside <SURFACE>. You <CORE JOB IN ONE SENTENCE>.
You are <PERSONA ADJECTIVES, OR "efficient and direct">.
```

Keep it to 1–3 sentences. A bloated identity section dilutes itself.

### Block 2 — Voice & style rules

*Purpose:* constrain spoken output to voice, not prose. This is where most chat-trained
prompts fail — paragraphs that read fine on screen are unbearable spoken aloud.

```
- <N> short sentences per turn. This is speech, not an essay.
- Teach/answer the IDEA; never narrate what's on screen — the user can see it.
- Stop instantly if the user speaks. Listen, answer, resume.
- Never read long lists, code, or URLs verbatim — summarize.
- Match the configured style/verbosity settings, if any are passed in context.
```

The "stop instantly" rule only fully works in `openai_concierge` (true barge-in); state
it anyway so the agent doesn't fight an interruption.

### Block 3 — Hard rules / prohibitions

*Purpose:* the explicit don't-list, stated early. See §9 for the full catalogue. Put it
*before* the mechanics — a prohibition the agent reads after the instructions is a
prohibition it has already half-ignored.

```
Hard rules (never violate):
- Never announce a tool/action call ("I'll now…", "Let me…"). Just do it.
- Never stall ("one second", "loading", "hang tight").
- Never ask permission to continue ("ready for the next?"). Autopilot is on.
- Never narrate the screen ("as you can see…", "this view shows…").
- Never emit stage directions in brackets — TTS reads them aloud.
```

### Block 4 — How context reaches you (the input contract)

*Purpose:* the most important block. Define the schema of what the app sends, and state
the **completeness contract**: the context is authoritative and complete for the turn.
Full treatment in §7.

```
Context arrives via these inbound actions, as system updates (NOT spoken by the user):

- <KICKOFF ACTION>  — sent once at session start. Payload: <fields>. Absorb, then begin.
- <TICK ACTION>     — sent periodically. Payload: <fields>. Update your view; do NOT speak.
- <USER ACTION>     — the user typed/selected something. Payload: <fields>. Address it.
- <DONE ACTION>     — an awaited action finished. Payload: <fields>. Resume from here.

Read the payload as the FULL context for this turn — nothing else exists. If a field is
absent, that thing is not true right now: don't speculate, don't ask about it, don't
mention it.
```

### Block 5 — Output protocol (actions + payload schemas)

*Purpose:* enumerate the `agent_to_app` actions the agent may emit, and embed any rich
payload schema inline. End with "Emit ONLY these."

```
You drive the surface with these actions:
- <render action>({ <schema> })  — <when to use>
- <focus action>({ target, effect }) — <when to use>; call FIRST, then narrate.
- <advance/clear action>(...) — <when to use>

Emit ONLY these actions. <If a rich visual payload exists, paste its full JSON schema here.>
```

If the agent renders structured visuals, this is where the embedded schema lives, plus an
**idea→shape grammar** (see §8.1).

### Block 6 — Operating loop / modes

*Purpose:* the turn-by-turn cycle, and any mode switches keyed off context or keywords.

```
Operating loop:
1. On <KICKOFF>: read context, decide mode, <first action>, then begin.
2. Each beat: <render/act FIRST>, then speak <N> sentences. (RULE 0, §8)
3. On <USER ACTION>: pivot — address it now, then resume the arc.
4. On <DONE ACTION>: <render follow-up>, then continue.
5. Modes: <keyword/condition> → <behavior>. (e.g. "take over" → present fully)
```

### Block 7 — Worked example

*Purpose:* disambiguate everything above with one concrete end-to-end exchange showing
context in → actions out → speech. Short, but real. This is the highest-leverage section
for getting the agent to behave; skip it and you'll re-explain the same rules three times.

---

## 6. Persona is behavioral code, not flavor

In `Chatty` agents the persona line is a *filter on every output*, not decoration. "Be
witty" is useless; a concrete spoken example is executable. Write the persona as
**constraints the agent can't violate, with 1–2 example lines**:

```
You are <PERSONA>. Examples of your voice:
- <situation> → "<exactly how you'd say it>"
- <situation> → "<exactly how you'd say it>"
You never <the thing that would break character>.
```

The strongest version of this couples a persona to a *forbidden vocabulary* — e.g. an
agent that explains a technical concept while being forbidden from naming it, forcing it
to teach through analogy. The persona stops being a coat of paint and becomes the
behavior itself.

---

## 7. Context as a contract (the single most important pattern)

Treat incoming context as **authoritative and complete for the turn**, and say so:

> *"Read the context as the FULL context for this turn — nothing else exists. If a field
> is absent, that thing is not true right now: don't speculate, don't ask, don't
> mention it."*

This one paragraph does enormous work:

- **It kills hallucination.** Absence of a "transcript" field means there is no
  transcript — not "ask the user for it."
- **Absence becomes a signal, not a gap.** An optional "focus region" field being absent
  means "there's no flagged region," a decision input — not missing data. Disambiguate
  the two readings of `null` explicitly, or the agent stalls asking for things it was
  never meant to have.
- **It moves the burden to the app.** The app is responsible for sending complete
  context; the agent is freed from interrogating the user.

How to apply it:

1. Define your context as named fields/blocks and document each in the prompt (a minimal
   agent might receive 3 fields; a rich one a dozen typed blocks).
2. State the completeness contract verbatim.
3. Add a **quality signal** for graceful degradation — e.g. a field like
   `quality: rich | partial | minimal`. On `minimal`, the agent must **not apologize**:

   > *"DO NOT say 'I don't have access to this' — that frames it as a permission error.
   > Instead, open with one warm sentence, then ASK what the user wants. Treat thin
   > context as the start of a conversation, not an apology."*

---

## 8. Driving the screen: visual-vocal parallelism

For any agent that controls a screen, the deepest UX rule:

> **RULE 0 — RENDER FIRST, THEN NARRATE. ALWAYS.** The visual must be on screen by the
> time the user hears your second sentence. Voice and visual are PARALLEL streams, not
> serial — like a teacher writing on the board *while* explaining, not after. Talking to
> a blank stage is the single worst UX in the product.

Corollaries that recur:

- **Placeholder beats blank.** If the perfect visual needs more thought, render a minimal
  title-card *now* and refine on the next action. Never let the stage sit empty while
  talking.
- **Focus, then explain.** When narrating one part of a visual, fire the focus/animate
  action *first*, then say the line, so eyes and ears land together.
- **Don't narrate the visual; convey the idea.** Both agent and user see the screen.
  Banned: "as you can see…", "this view shows…".

### 8.1 Embedded schemas: give the agent a typed vocabulary

Don't let the agent emit free-form output for anything the surface renders. Embed the
**exact payload schema in the prompt** and give the agent a *typed vocabulary* of states
and moves, so it speaks the app's language instead of improvising one.

Take a voice-played game of tic-tac-toe. The agent narrates and plays; the app draws the
board. The temptation is to let the agent say "I'll take the top-left" and hope the app
parses it. Instead, define the board and the moves as a closed schema and make the agent
emit *that*:

```
Board state (what you receive each turn):
  { "cells": [null,"X",null, null,"O",null, "X",null,null],   // indices 0..8, row-major
    "turn": "O", "winner": null }

Your move (the ONLY action you emit to play):
  place({ "index": 0..8, "mark": "X" | "O" })

Rules:
- Read `cells` as the full board — an index that's null is empty, nothing else is implied.
- Emit place() with an exact index. Never describe a move you didn't emit (§3.4).
- Speak the move as a human would ("top-left"), but the payload is the index. Reason
  in words, address by schema.
```

The lesson generalizes far past games: **whatever the surface can render, enumerate it as
a closed set with a precise payload, and map the agent's intent onto that set in the
prompt.** A grid has nine indices; a slide deck has a fixed list of element kinds; a form
has named fields. In every case the agent reasons in natural language but *acts* through
the typed vocabulary — never free-form. Ship the vocabulary in the prompt and the agent
applies it reflexively, with no parsing layer guessing what it meant.

### 8.2 Hard sizing constraints, stated as physics

> *"The canvas is SMALL and does NOT scroll. Anything past the visible area is clipped.
> 3–5 elements per view. Hard cap — not 'around 5,' 5 max. Each line ≤ 12 words. One
> display visual per view. More to say? Render a SECOND view."*

State constraints as measurable physics, not preferences. "Keep it simple" is ignorable;
"5 max" is not.

---

## 9. The don't-list: anti-patterns as hard rules

Voice agents fail in public and irreversibly. Every strong prompt carries an explicit
banned-phrase block. The recurring categories, worth copying wholesale:

- **Stalling** — "Hang tight", "Give me a moment", "Loading…", "One second". A fast agent
  just does the next thing (or renders a placeholder). Never announce latency.
- **Announcing tool calls** — "I'm going to show you…", "Let me pull up…". Just call the
  action. The mechanism is invisible; only the result matters.
- **Asking permission to continue** — "Ready for the next one?", "Still with me?".
  Autopilot is on. A driving agent assumes forward motion; the user interrupts if needed.
- **Narrating the visual** — "As you can see…", "This view shows…". Convey the idea, not
  the pixels.
- **Encouragement / coaching filler** — "Great point", "You're doing well". Reads as
  patronizing and, for support agents, steals the human's authority.
- **Stage directions in brackets** — `[silence]`, `(quiet)`. **The TTS layer reads these
  aloud** — the user literally hears the word "silence." Use actual absence (emit
  nothing), never an instruction to be silent.

> The bracket-directions rule is the sharpest tell of someone who wrote for chat but not
> for voice. In a voice agent, silence is *empty output*, never a narrated token.

---

## 10. External tools (MCP / api / AI-agent)

Most rich agents need **none** of these — client actions plus a strong context contract
cover teaching, presenting, and interactive experiences entirely. Reach for tools only
when the agent must *generate or fetch* something it cannot itself produce.

When you do wire tools, the doctrine that works:

- **Call tools proactively — don't ask permission.** The moment the user states a data
  point that warrants a chart, the chart should exist. "Ask permission to create a
  visual? No — just do it."
- **Generate during the conversation, not at the end.** Artifacts are part of the
  exchange, not a final batch.
- **Call tools directly by name**, not through a generic "submit a background query"
  wrapper.
- **Every generated asset carries a description** — the semantic anchor that keeps it from
  becoming an orphaned visual.

For a delegated **AI-agent** channel (`ai_agent_config`), default to handling things
yourself: *"Do NOT delegate basic questions — you already have what you need. Only
delegate for genuinely deep dives."* Delegation is a latency and coherence cost.

> Note: some `model_settings` / `ai_agent_config` fields may not be pushable via CLI in
> all versions — set them in the dashboard if so.

---

## 11. Two patterns worth their own section

### 11.1 Silent-until-cued (support agents)

A support agent should never speak on connect. The contract:

> *"On connect, DO NOT speak. Establish visual state silently (set the first section /
> show the title), then wait for the human to begin."*

The agent establishes *visual* state immediately but emits no audio, then waits for a
typed trigger. Triggers are explicit and typed:

- a **keyword** ("take over", "explain X") → flip to active,
- a **silence threshold** (≈3–4 seconds) → "stuck"; offer one line,
- a **priority event** (e.g. a paid/flagged message) → acknowledge within a fixed window.

Calibrate the silence window: too short interrupts thinking, too long creates dead air.

### 11.2 The strict-sequence primitive (let the source speak)

When the agent should *stop and let something else play* (a source clip, an external
media moment), use a strict turn-by-turn protocol — because the failure mode (agent audio
over source audio) is irrecoverable:

> 1. **Setup turn:** one short lead-in ("Here's the difference — watch."), then yield.
>    Don't announce mechanics. Don't call the action yet.
> 2. **Play turn (the next turn):** with **no words at all**, call the play action as the
>    *only* thing in the turn. Speaking in the same turn makes your voice play over it.
> 3. **Silence:** after the call, emit nothing. It's playing.
> 4. **Resume:** only on the "finished" signal — render the follow-up, then narrate.

This is a **state machine written in prose**, gated on an inbound `respond` action that
can also carry an `interrupted` flag, so the agent reacts if the user spoke during the
clip instead of plowing on with its plan. The general lesson: **knowing when *not* to
talk is a design feature.** LLMs default to filling silence; good voice agents are told
precisely when to step back.

---

## 12. A planning checklist

Before you ship a VB agent, you should be able to answer:

- [ ] **Interaction model** named (reactive / driver / planner / dual-stream)? → sets
      `mode`, `style`, greeting length.
- [ ] **Client-action interface** enumerated, each tagged `direction` + `behavior`?
- [ ] Context-in actions described as *absorb/when-to-react*; control-out as
      *when-to-fire/payload-meaning*?
- [ ] Ambient "tick" signals set to `notify`, not `respond`?
- [ ] **Context-completeness contract** stated ("the payload is the full context; absence
      = not true")?
- [ ] A **quality/degradation signal** that reframes thin context as a conversation, not
      an apology?
- [ ] For screen agents: **RULE 0** (render-first), hard sizing caps as physics, and a
      visual grammar (idea→shape)?
- [ ] **Banned-phrase block**: stalling, tool-announcing, permission-asking,
      visual-narrating, bracket stage-directions?
- [ ] **Persona** written as constraints + concrete spoken examples (not adjectives)?
- [ ] Tools genuinely needed, or can client actions + context do it? If used: proactive,
      in-conversation, direct calls, described assets.
- [ ] A **worked end-to-end example** in the prompt?

---

## Appendix — interaction archetypes (by shape, not product)

Pick the row that matches what you're building; it sets your defaults.

| Archetype | Typical mode | Greeting | Speaks when | Defining trait |
|---|---|---|---|---|
| **Driver** (owns an arc, e.g. a tutor/performer) | realtime | `null` | Continuously, autopilot | RULE 0, crisp termination, never asks permission |
| **Reactive support** (co-host, assistant) | realtime | minimal | Only when cued or stuck | Silent-until-cued; reactive visual manager |
| **Dual-stream** (drives screen + a second channel) | cascaded | minimal | Curated, channel-timed | Time-multiplex two streams; priority-event contracts |
| **Planner** (phased consulting conversation) | cascaded | minimal | Throughout, structured | Phase gates; proactive tool use; validate-before-finalize |
| **Minimal Q&A** (answer about current context) | realtime | minimal | When asked | Tiny action set; handle it yourself, don't over-delegate |
| **Experiential** (game / character) | realtime | atmospheric | In character, always | Persona-as-code; polymorphic context-driven actions |
```
