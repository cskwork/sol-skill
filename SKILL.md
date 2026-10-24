---
name: sol
description: Orchestrate application work through GPT-6.1 Sol subagents (high reasoning, fast tier). Use for /sol, "use sol", or handing build, fix, backend, browser or QA slices to Sol workers.
---

# sol

You are the **orchestrator**: slice, brief, launch Sol workers, integrate, verify. Workers own the slices; you own the whole.

Explicit `/sol` sends **every** slice to Sol, UI/frontend and docs included; it overrides model routing in `rules.md`.

## Steps

1. **Native check.** Host already runs `gpt-6.1-sol` at high reasoning (Codex)? Use its own subagents and skip step 4's shell call. Otherwise launch workers with `codex exec`. Never use a Claude/Opus subagent as a Sol stand-in.
2. **Preflight** (once): `command -v codex && test -f ~/.codex/auth.json`. Fails: report `codex login` / install, stop.
3. **Slice.** Slices that share no files, state or ordering run in parallel; dependent slices run in sequence. Write one brief per slice to `<scratch>/<slice>.brief.md`:

   ```markdown
   Objective: <one outcome>
   Owned paths: <only files/dirs this worker may change>
   Interfaces to honor: <APIs, types, contracts that must not change>
   Constraints: <repo rules, no git push/merge, no store uploads>
   Acceptance: <exact commands/checks that must pass>
   Known facts: <verified findings, file:line>
   Report once at the end: changed files, checks run with results, open problems. No progress messages.
   ```

4. **Launch** each slice as a background task:

   ```bash
   codex exec -m gpt-6.1-sol -c model_reasoning_effort=high -c service_tier=fast \
     -s workspace-write --skip-git-repo-check -C "<repo>" \
     -o "<scratch>/<slice>.out.md" - < "<scratch>/<slice>.brief.md"
   ```

   Block on completion notifications; never short-poll.
5. **Integrate.** Read each `.out.md`, confirm `git diff --stat` touches only owned paths, then run the acceptance checks yourself. A failed slice gets one re-brief carrying the failure evidence; a second failure is reported, not retried.

## Rules

- Keep `-s workspace-write`; never `--dangerously-bypass-approvals-and-sandbox`.
- Workers never commit to `main`, push, merge or publish; the orchestrator does that, only when authorized.

**Done =** every slice's acceptance checks pass under your own run, or the failing slice is reported with its evidence and what remains.
