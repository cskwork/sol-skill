# sol

`/sol` is an agent skill for orchestrating work through GPT-6.1 Sol workers. The orchestrator splits work into slices, writes a brief for each worker, integrates the results, and verifies the whole. Independent slices run in parallel; dependent slices run in sequence.

[Landing page](https://cskwork.github.io/sol-skill/) | [Skill instructions](SKILL.md)

## Requirements

- Codex CLI installed and logged in with `codex login`.
- Access to `gpt-6.1-sol`.
- An agent host that loads skills and can launch Codex CLI workers.

- For builds that write to user caches or talk to system services (Xcode DerivedData, `~/.gradle`, simulators): `approvals_reviewer = "auto_review"` in `~/.codex/config.toml`, so workers' sandbox escalations are approved. Without it, pass cache dirs with `--add-dir` or keep build output inside the repo.

The skill runs a preflight once before launching workers:

```bash
command -v codex && test -f ~/.codex/auth.json
codex exec -m gpt-6.1-sol -c model_reasoning_effort=low --ephemeral --skip-git-repo-check \
  -C "${TMPDIR:-/tmp}" "Reply with exactly: ok" < /dev/null
grep -Eq '^approvals_reviewer *= *"auto_review"' ~/.codex/config.toml
```

The `< /dev/null` matters: with an open stdin, `codex exec` waits for it even when a prompt is given.

## Install

Clone into the shared skills directory:

```bash
git clone https://github.com/cskwork/sol-skill ~/.agents/skills/sol
```

If your host uses another skills directory, symlink the cloned directory into it as `sol`, for example at `~/.claude/skills/sol` or `~/.codex/skills/sol`.

## Usage

```text
/sol add input validation to the signup API
```

Explicit `/sol` routes every slice to Sol, including frontend and documentation work, overriding model routing in `rules.md`.

When the host already runs `gpt-6.1-sol` at high reasoning in Codex, it uses native subagents. Otherwise, it gives each parallel slice its own git worktree and launches the worker as a background task with the host's maximum timeout (the Claude Code default of 30 minutes kills workers mid-build):

```bash
git -C "<repo>" worktree add "<scratch>/wt/<slice>" -b "sol/<slice>"
codex exec --json -m gpt-6.1-sol -c model_reasoning_effort=high -c service_tier=fast \
  -s workspace-write --skip-git-repo-check -C "<scratch>/wt/<slice>" \
  -o "<scratch>/<slice>.out.md" - < "<scratch>/<slice>.brief.md" > "<scratch>/<slice>.jsonl"
```

Replace `<repo>`, `<scratch>`, and `<slice>` with the repository path, scratch directory, and slice name.

## Worker briefs and verification

Each brief defines one objective, the parent rules and docs the worker would not load itself (Codex reads `AGENTS.md` only from the git root down), skills to load, owned paths, interfaces to preserve, constraints, acceptance checks, and verified facts with file and line references. Every brief also carries default acceptance: existing build, lint and tests pass; changed behavior has evidence (for UI, captures at the smallest and largest viewport plus an empty or error state); the result is recorded where the project records verification; and every process, server, device, emulator or simulator the worker started is stopped. Workers report changed files, check results, evidence paths, and open problems once at the end.

The orchestrator reads the results, checks that each worktree's diff stays within owned paths, merges the slice branches, and runs acceptance checks itself on the merged result. A failed slice resumes its own session with the failure evidence (`codex exec resume <thread_id>`, id from the first line of `<slice>.jsonl`). A second failure is reported with what remains.

## Safety

- CLI workers run sandboxed with `-s workspace-write`. Never bypass approvals or the sandbox.
- Workers never commit to `main`, push, merge, or publish. The orchestrator performs those actions only when authorized.
- Independent slices share no files, state, or ordering.
- Don't add `--approve-for-me`: codex-cli 0.164.0 rejects it with a usage error.

## License

[MIT](LICENSE). Copyright (c) 2026 cskwork.
