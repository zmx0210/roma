# 22. A second Runtime, reached through the model command

Date: 2026-08-03

## Status

Accepted, and deliberately not yet implemented. Every section below is gated on the
measurement spike §9 names, because three of the facts this design stands on have
never been checked against the build roma would pin. The commit that carries this
ADR changes no source. If the spike falsifies §9(1), this ADR is reopened rather
than worked around.

## Context

roma is one central Claude Code agent (ADR-0001), and every term that runs work —
Session, Transcript, Turn, Compaction, Relay — is defined against one pinned build
of one CLI. That was a strength for as long as there was one subscription to draw
on.

There are now two. The team also holds a ChatGPT subscription: a second rolling
window, shared the way the first one is, that nothing in roma can currently spend.
Codex — OpenAI's agent CLI — is the vehicle for it, and it is the same *shape* as
Claude Code where it matters: a non-interactive exec mode, a JSONL event stream, a
session record on disk with a resume. What differs is everything the glossary
currently hard-codes: the provider, the credential, the quota, the wire.

The motivation is the quota, and it was decided deliberately: this is "spend the
second subscription on purpose", never "fail over when the first is blocked". A
failover was considered and rejected below, because crossing runtimes destroys the
one thing a Conversation's users share.

## Decision

### 1. The term is Runtime, and a Session is born on one

A **Runtime** is the agent CLI a Session runs on — `claude-code` or `codex`. A
Session is created on one Runtime and never moves: neither Runtime can read the
other's Transcript, so "moving" a Session is not an operation that exists, only
ending one Session and starting another.

Runtime is an internal word. It goes in CONTEXT.md, in the Audit Record, and in
this document; it is never a word a Caller has to learn, and §2 is what makes that
possible. The glossary entries that currently say "Claude Code's" — Session,
Transcript, Turn, Compaction, Relay, Shared Window, the two Menus — are rewritten
to be per-Runtime in the CONTEXT.md change that follows this ADR.

`provider` was not available as the name: that word is the company on the other
end (ADR-0020), and the company is exactly what a Runtime is not — OpenAI is the
provider, `codex` is the CLI roma drives against it.

### 2. The way in is `/model`, with a provider prefix

`/model openai:gpt-5.6-sol` chooses OpenAI's model and, with it, the Runtime that
serves OpenAI's models. An unprefixed name keeps meaning what it means today — an
Anthropic model — so every existing spelling survives unchanged, and `anthropic:`
is accepted as the explicit form for symmetry.

There is no sixth Command. A `/runtime` or `/agent` command was rejected for three
reasons, recorded because the alternative reads well until the state is drawn out:
it puts two pieces of state where one suffices (a Chosen Runtime crossed with a
Chosen Model, every combination of which needs an answer and a sentence); it grows
a Command list CONTEXT.md explicitly says somebody has to keep true; and it makes
the Runtime a word Callers type, which §1 just promised they never would. The
prefix is atomic — one argument decides both — and there is no intermediate state
to explain.

**The prefix names the provider, not the Runtime.** `openai:`, not `codex:`. A
Caller is saying whose model answers them; which CLI carries the message is roma's
business, and the day OpenAI ships a different CLI the prefix does not move.

The Model Menu becomes namespaced and remains exactly what it is: a spending
boundary, not a typo filter. The second subscription is also a window everybody
shares, so its models go on the Menu for the same reason the first window's do —
one person must not be able to move a shared bill without the deployment having
offered the choice. A deployment with no Codex credential lists no `openai:`
entries at all (§5).

### 3. A `/model` that crosses providers starts a fresh Session, and says so

Within a provider, `/model` keeps its meaning: same Session, same context, a
different model on the next Turn. Across providers that is physically impossible —
there is no Transcript both Runtimes can read — so a crossing `/model` moves the
Session Generation exactly as `/clear` does: same Conversation, same thread, and
the next message starts from nothing.

The posture is `/clear`'s: do it immediately, and say it plainly — "switched to
openai:gpt-5.6-sol, the conversation starts over" — rather than refusing until a
`/clear` arrives or inventing a confirmation flow no Command has. A Conversation
is many people sharing one Session, so the reply is the only place the cost is
visible to the people who paid it, and the reply therefore always names it.

A summary handoff — asking the old Runtime to brief the new one — was considered
and deferred, not refused: it costs a Turn of the window being left, it is lossy,
and it tangles with Compaction. It can be v2 without this ADR moving.

### 4. The default does not move

The Pinned Model stays an Anthropic model, so a Session nobody has touched runs on
`claude-code` exactly as today. `/model default` and `/clear` both land there.
Choosing a second Runtime for a deployment's *default* is not in this ADR at all.

### 5. The Codex credential is optional, and follows the Reach pattern

A deployment gives roma a Codex credential or it does not, and both are ordinary.
Unset, roma boots normally, the Menu carries no `openai:` section, and
`/model openai:...` is answered with a sentence — unavailable, never absent
(ADR-0015 §9, ADR-0020 §2). Set, it is proved at boot the way every credential
roma holds is proved: by a real invocation, before anything that could accept an
Ingress Message exists, and a credential that exists and does not work blocks the
boot rather than surfacing as somebody's failed Turn.

### 6. Parity where measured, a sentence where it is not

The direction chosen is that using roma should feel the same on either Runtime,
and the rule that bounds it is ADR-0012's: a term defined by one build's behaviour
turns false on somebody else's release. So parity is claimed only where the pinned
Codex build has been *measured* to support it, and everywhere else roma says the
honest sentence instead of pretending.

Concretely: `/effort` maps onto Codex's own reasoning-effort setting, and the
Effort Menu — already defined as "every level the pinned build has" — becomes one
table per Runtime, which is the same definition with the build made explicit. The
Relay list was always a whitelist against one build; it becomes one whitelist per
Runtime, and what goes on the Codex list is decided by §9(3), not by analogy.
Compaction accounting records what the stream actually reports and fabricates
nothing.

### 7. The second window Parks and never Overflows

When the ChatGPT window is spent, a blocked Codex Task is Parked — kept, said out
loud, holding no slot, stoppable — exactly as a blocked Claude Task is. What it is
not offered is Overflow. Overflow stays Claude-only in v1: a Codex equivalent
means a second metered key, a second monthly cap nobody has set, and a second set
of boundary cases, bought back only in the moments the window is already gone.
Deferred until being blocked has actually hurt.

Offering "switch back to Claude" at the blocking moment was rejected outright: it
would package the one destructive operation in this design (§3) as relief, at the
exact moment somebody is most likely to take it without reading.

The Audit Record for a Codex Turn counts tokens and marks the cost **unpriced** —
Codex's stream reports usage and no dollar figure, and unpriced-rather-than-free
is a distinction the Audit Record already knows how to make.

### 8. What this costs is named, not hidden

A second Runtime is a second pinned binary, and it inherits ADR-0007 whole: the
image pins one Codex version, moving it is a re-verification event, and the drift
check learns to watch a second pin. Seam 1 needs recorded Codex streams beside the
Claude ones; seam 2 gains a second opt-in config that spends the *other*
subscription. None of that is optional decoration — it is what "pinned and
measured" costs when there are two of everything, and it is the price §9 exists to
check before anyone pays it.

### 9. Nothing is built until a spike proves three facts

The spike runs against the exact Codex build the image would pin, and its results
land in `docs/` beside the other verification documents. Three questions, in
order of how much falls if the answer is no:

1. **Does `codex exec resume` reliably continue a session across process death?**
   This is the load-bearing wall: Eviction, Reaping and the Session Pool all
   assume a process can die and the Session survive it. If resume is unreliable,
   there is no Codex Session, only Codex requests — and this ADR is reopened.
2. **Does the JSONL stream carry enough to drive the Acknowledgement and the
   accounting?** Tool progress for the one message roma keeps editing; token
   usage for the Audit Record.
3. **Does `/compact` have any hook in exec mode?** The answer decides the Codex
   Relay list. A slash command that exec mode reads as prose is not a Relay, it is
   somebody's money spent on a sentence the model shrugs at.

## Consequences

- CONTEXT.md gains **Runtime** and rewrites Session, Transcript, Turn, Compaction,
  Relay, Shared Window, Model Menu, Effort Menu and the Audit Record to be
  per-Runtime. That change follows this ADR; the glossary stays free of
  implementation detail, so the mapping "openai-prefixed models are served by
  codex" lives here and in the code, not there.
- The spike (§9) is the first ticket, and it spends ChatGPT quota by design —
  measurement before construction is this repo's standing order (ADR-0007).
- The Startup Self-Check grows a second arm on deployments that configure the
  credential (§5), and none anywhere else.
- `/model`'s parser learns a prefix; nothing else about Commands changes.
- Nothing in `src/` changes with this commit.

## Alternatives considered

**A separate `/agent` (or `/runtime`) command.** Rejected in §2: two interacting
pieces of state, a sixth entry on a kept-true list, and the Runtime leaking into
Caller vocabulary. The prefix carries the same information atomically.

**`codex:` as the prefix.** Rejected in §2 after nearly being chosen: the prefix
names the provider a Caller is choosing, and the CLI behind it stays roma's
business.

**Automatic failover when the Claude window is blocked.** Rejected in Context and
§7: crossing Runtimes destroys the memory a whole Conversation shares, and a
destructive operation must never be something roma does — or offers — on its own
initiative at a moment of scarcity.

**A summary handoff on crossing.** Deferred, §3: a Turn of cost, lossy, and
tangled with Compaction. v2 material, not v1.

**Honest refusal instead of parity (`/effort` answering "not on this Runtime").**
Rejected as the resting state, kept as the fallback: the chosen direction is that
the two Runtimes read the same wherever the measured build allows it, and the
sentence is what remains where it does not (§6).
