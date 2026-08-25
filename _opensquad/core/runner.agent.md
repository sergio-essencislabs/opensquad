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
  escopo" anyway) and does not produce a PR that needs the pipeline's approval checkpoints.

If the ask genuinely needs cross-agent coordination (e.g. a security finding that needs a code fix
reviewed and merged), route to the full pipeline instead — this runner has no PR/approval flow.

## Initialization

1. Read the squad's `squad.yaml` and `squad-party.csv` (same as the full pipeline) to resolve the
   named agent to its `id`, `displayName`, `icon`, and `path`.
   - If the name doesn't match any agent unambiguously, ask the user to clarify (list the squad's
     agents with their display names) — never guess which persona was meant.
2. Read the full agent file (`squads/{name}/agents/{agent}/{agent}.agent.md` or equivalent path
   from `squad-party.csv`).
3. Determine which task(s) to run:
   - If the user's request names a specific task (or clearly maps to one via the agent's own
     `tasks:` list and each task's frontmatter), run just that task.
   - If the request is broad ("atualize a documentação") and the agent has multiple tasks, run the
     ones whose frontmatter `input`/`output` fit an ad-hoc context (no upstream pipeline artifact
     required) — for Marta (`documentation-architect`) that's `atualizar-documentacao.md` and
     `curar-knowledge-base.md` run in sequence, in that order, **not** `auditar-documentacao.md`
     (that task's input is a pipeline checkpoint artifact that doesn't exist outside a run).
4. Company context (`_opensquad/_memory/company.md`) and squad memory
   (`squads/{name}/_memory/memories.md`) load the same way as in the full pipeline.

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
- **No pipeline checkpoints.** The 4-checkpoint flow (scope, review findings, approve PRs, final
  summary) does not apply.

## Helper agents: the lead can pull in narrow expertise, never delegate the task away

An ad-hoc run has exactly one **lead** agent (the one the user named). The lead may invoke another
squad member as a **helper** mid-task when it hits a question genuinely outside its own domain —
e.g. Marta (documentation-architect) needs the exact Service/Repository method signature for a
controller endpoint before she can write an accurate `Metodos_Back` row, which is Breno Backend's
domain, not hers.

This is narrow-scope consultation, not task delegation:

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

Before writing any file, present a short summary of what will change (which files, and a one-line
description of each change — not a full diff dump) and wait for the user's go-ahead. This is the
only confirmation gate in ad-hoc mode; it exists because writing documentation unsupervised carries
the same hallucination risk any other agent output does (see `documentation-architect.agent.md`'s
own principles on never treating a document as trivially safe to auto-correct).

```
📚 {Agent Name} — pronta para gravar:
- {arquivo 1}: {resumo de 1 linha da mudança}
- {arquivo 2}: {resumo de 1 linha da mudança}

Gravar agora? (s/n)
```

If the user declines, stop without writing anything and offer to adjust based on their feedback.

## After writing

1. Prepend one row to `squads/{name}/_memory/runs.md`, immediately after the header row (create the
   file first with the standard header if it doesn't exist yet — same table shape and
   reverse-chronological, newest-run-first convention the full pipeline uses):
   - `Data`: today's date (YYYY-MM-DD)
   - `Run ID`: `adhoc-{agent id}-{HHmmss}`
   - `Tema`: the user's request, 1 sentence
   - `Output`: brief description of what was written
   - `Resultado`: `Ad-hoc`
2. Do **not** write to `squads/{name}/_memory/memories.md` unless the user gave explicit feedback
   during this run (same rule as the full pipeline's memory-update step).
3. Present a short completion summary — files touched, nothing else (no dashboard, no "run again"
   menu; this is a one-off invocation, not a squad session).

## Error Handling

- If the named agent has no `tasks:` frontmatter, run it monolithically (same fallback as the full
  pipeline) using whatever the request describes as the job.
- If a task's declared `input` requires an artifact that only exists mid-pipeline (e.g. an
  `audit-scope.md` from a checkpoint), tell the user this task can't run standalone and suggest the
  full pipeline instead — do not fabricate a stand-in input.
