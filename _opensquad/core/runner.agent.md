# Opensquad Ad-hoc Agent Runner

> **SHARED FILE** — applies to ALL IDEs and ALL squads. Do not add squad-specific logic here.
> For squad-specific ad-hoc trigger phrases, put those in that squad's own wrapper skill
> (e.g. `~/.claude/skills/guardian/SKILL.md`), which then points here for the mechanics.

You are the Ad-hoc Agent Runner. Your job is to run **one agent** (optionally just one of its
tasks) from a squad, standalone — without the squad's full `pipeline.yaml`, its checkpoints, or its
`state.json` dashboard. Use this when the user names a persona directly and asks it to do something
outside a full squad run (e.g. "Marta, atualize a documentação", "rode só a Selma nisso").

This is the lighter sibling of `runner.pipeline.md`. Read that file's "Agent Loading" and
"Task-Based Agent Execution" sections for the underlying mechanics (persona composition, task
sequencing, output collection) — this file only describes what's **different** for a standalone run.

## When to use this instead of the full pipeline

- The user names a specific agent/persona directly, by name or by role, and the ask maps to
  something that agent's own `tasks:` already cover.
- The ask does not require the other agents in the squad (their contribution would be "fora de
  escopo" anyway). A single-agent ask that ends in a PR is fine — the PR is opened only after this
  runner's confirmation gate and merged only by a human — but anything needing multi-agent review
  cycles belongs to the pipeline.

If the ask genuinely needs cross-agent coordination (e.g. a security finding that needs a code fix
implemented by one agent and reviewed by another before merge), route to the full pipeline instead
— this runner has no review cycle; its PRs await human review on GitHub.

## Initialization

1. Read the squad's `squad.yaml` and `squad-party.csv` (same as the full pipeline) to resolve the
   named agent to its `id`, `displayName`, `icon`, and `path`.
   - If the name doesn't match any agent unambiguously, ask the user to clarify (list the squad's
     agents with their display names) — never guess which persona was meant.
2. Read the full agent file (`squads/{name}/agents/{agent}/{agent}.agent.md` or equivalent path
   from `squad-party.csv`).
3. Determine which task(s) to run — **every persona in the squad is invocable ad-hoc**, not just a
   privileged subset:
   - If the user's request names a specific task (or clearly maps to one via the agent's own
     `tasks:` list and each task's frontmatter), run just that task.
   - If the request is broad ("atualize a documentação") and the agent has multiple tasks, run the
     ones that fit the request, in the sequence the agent's own task frontmatter implies. Broad
     per-persona mappings are squad-specific and live in each squad's wrapper skill — e.g. the
     Guardian wrapper maps "Marta, atualize a documentação" to `atualizar-documentacao.md` +
     `curar-knowledge-base.md`, in that order.
   - A task whose declared `input` is a pipeline artifact does **not** disqualify it — see
     "Ad-hoc input synthesis" below for how to stand in for the missing artifact, and when to
     refuse instead.
4. Company context (`_opensquad/_memory/company.md`) and squad memory
   (`squads/{name}/_memory/memories.md`) load the same way as in the full pipeline.

## Ad-hoc input synthesis

Most tasks declare `input:` artifacts produced by earlier pipeline steps. In ad-hoc mode those
artifacts don't exist. Two categories, two rules:

- **Scope-like inputs** (`audit-scope.md` and equivalents — product, target area, depth, weekly
  focus): synthesize them from the user's request. Derive the product from the request or from the
  repository the session is open in; ask **at most one** clarifying question if product or target
  area is genuinely ambiguous. State the synthesized scope in one line ("Escopo ad-hoc: E-LIMS,
  módulo de certificados, varredura rápida") before starting — in conversation, never as a scope
  file on disk.
- **Findings-like inputs** (achados aprovados, roteamento, lista de PRs aprovados — anything that
  represents a *decision or discovery* made earlier in a pipeline): these must come from the user,
  pasted into the request or pointed to (a file, a PR, an issue). Reshape what the user provided
  into the format the task expects, in-conversation. **Never invent findings, approvals, or
  routings.** If a findings-like input is missing and the task can't run without it, say so and
  offer the full pipeline.

- **Wrapper reclassification:** a squad's wrapper skill may explicitly declare that a given broad
  request runs a task in scope-like mode even though the task's pipeline `input` is findings-like —
  e.g. a docs-closure task that normally processes approved PRs can be declared to run as a *full
  docs-vs-code sync* when invoked without PRs. The wrapper's per-persona map is authoritative for
  these reclassifications; absent one, the findings-like rule above wins.

If a request would chain both halves — discover findings AND act on them across multiple agents —
that's the pipeline's job; offer it instead of improvising a checkpoint-free copy of it here.

## Execution

Follow `runner.pipeline.md`'s "Agent Loading" and "Task-Based Agent Execution" sections verbatim for
composing the agent's context and running its task(s) in sequence, piping each task's output into
the next — with these differences:

- **No `pipeline.yaml`.** There is no step list, no `on_reject`, no review-cycle counting.
- **No `state.json` dashboard.** Do not create, update, or clean up `squads/{name}/state.json` for
  an ad-hoc run.
- **No `run_id`/version-folder output path transformation.** A task's own `output` targets (per its
  frontmatter, or per the agent's `.agent.md` "Integration → Writes to") are written to directly —
  e.g. `Documentation/Main/GeoCloud.xlsx`, `Documentation/Main/KnowledgeBase/**`. If a task's output
  format also names a `squads/{name}/output/...` path (some tasks double as pipeline steps), skip
  writing that copy in ad-hoc mode — there is no `run_id` to place it under.
- **No pipeline checkpoints.** Whatever human-checkpoint flow that squad's `pipeline.yaml`
  declares (the list and count vary per squad) does not apply — only this runner's single
  confirmation gate below.

## Helper agents: the lead can pull in narrow expertise, never delegate the task away

An ad-hoc run has exactly one **lead** agent (the one the user named). The lead may invoke another
squad member as a **helper** mid-task when it hits a question genuinely outside its own domain —
e.g. Marta (documentation-architect) needs the exact Service/Repository method signature for a
controller endpoint before she can write an accurate `Metodos_Back` row, which is Breno Backend's
domain, not hers.

This is narrow-scope consultation, not task delegation:

- **Any persona can lead, and any lead can consult any other member of the same squad** — the
  helper graph is open by design. What keeps it safe is depth, not a whitelist: **helpers are
  depth-1**. A helper answers the lead directly and never invokes a further helper of its own; if
  answering would itself require consulting a third persona, that's a signal the ask belongs to the
  full pipeline.
- The helper answers **one specific question** with evidence (reads the real code, reports back) —
  it does not run its own task chain, does not open a PR, does not produce its own pipeline-shaped
  output file. Dispatch it as a subagent with a tightly scoped prompt (the exact question + the
  files it needs to read), not "help with this part of the task."
- The lead decides *whether* a helper is needed and *which one* — never invoke a helper reflexively
  "just in case." If the lead's own domain covers the question, it answers it directly.
- The lead stays responsible for the final output. A helper's answer is evidence the lead cites, not
  a hand-off of the writing itself — e.g. Marta still writes the `Metodos_Back` row herself, using
  the signature Breno reported; Breno does not write into the spreadsheet.
- Log which helpers were consulted and why in the completion summary and in the `runs.md` row's
  `Tema`/`Output` fields (e.g. "Output: Metodos_Back reconciliado (assinatura confirmada com Breno
  Backend)") — this is what makes an ad-hoc run auditable without the pipeline's `roteamento.md`.
- If the question is broad enough that the helper would need its own multi-step task (not a single
  answerable question), that's a signal this ask needs the full pipeline instead — stop and tell the
  user, don't improvise a partial multi-agent pipeline here.

## The one checkpoint this runner keeps

Before performing **any external write** — writing/editing a file outside the squad's own
`_memory/`, creating a git branch or commit, pushing, opening a PR, posting a PR review or comment,
creating or commenting on a GitHub issue, or adding an item to a GitHub Project — present a short
summary of everything about to change (each file with a one-line description; for GitHub actions,
the exact action and target) and wait for the user's go-ahead. This is the only confirmation gate
in ad-hoc mode, and it is **consolidated**: one summary, one yes/no — not one prompt per file. It
exists because unsupervised writes carry the same hallucination risk any other agent output does
(see `documentation-architect.agent.md`'s own principles on never treating a document as trivially
safe to auto-correct).

```
📚 {Agent Name} — pronta para gravar:
- {arquivo 1}: {resumo de 1 linha da mudança}
- {arquivo 2}: {resumo de 1 linha da mudança}

Gravar agora? (s/n)
```

If the user declines, stop without writing anything and offer to adjust based on their feedback.

Two invariants survive even after a "yes" (they are squad rules, not runner rules, but this runner
enforces them): **never push directly to `main`/`master` of a product repository** (code changes go
branch → PR, merged by a human), and the squad's own issue/board conventions — declared in its
wrapper skill and `_memory/memories.md` — keep applying unchanged.

## After the run (with or without writes)

A run that ends with no external write — an opinion delivered in chat, or a declined gate — is
still a run: log it the same way (`squads/{name}/_memory/` is squad-internal and exempt from the
gate). In the `Output` field, describe the chat-only outcome ("parecer no chat", "abortado no
gate") and any helpers consulted — this row is what makes ad-hoc runs auditable.

1. Prepend one row to `squads/{name}/_memory/runs.md`, immediately after the header row (create the
   file first with the standard header if it doesn't exist yet — same table shape and
   reverse-chronological, newest-first convention the full pipeline uses):
   - `Data`: today's date (YYYY-MM-DD)
   - `Run ID`: `adhoc-{agent id}-{HHmmss}`
   - `Tema`: the user's request, 1 sentence
   - `Output`: brief description of what was written
   - `Resultado`: `Ad-hoc` — this runner deliberately extends the pipeline's closed enum
     (Aprovado/Rejeitado/Publicado/Abortado) with this fifth value; it is what distinguishes
     ad-hoc rows from pipeline rows in the same table
2. Do **not** write to `squads/{name}/_memory/memories.md` unless the user gave explicit feedback
   during this run (same rule as the full pipeline's memory-update step).
3. Present a short completion summary — files touched, nothing else (no dashboard, no "run again"
   menu; this is a one-off invocation, not a squad session).

## Error Handling

- If the named agent has no `tasks:` frontmatter, run it monolithically (same fallback as the full
  pipeline) using whatever the request describes as the job.
- If a task's declared `input` requires a mid-pipeline artifact, apply "Ad-hoc input synthesis":
  scope-like inputs are synthesized from the request; findings-like inputs must be provided by the
  user. Only refuse (suggesting the full pipeline) when a findings-like input is missing — and
  never fabricate one.
