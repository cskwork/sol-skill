# sol

`/sol` is an agent skill for orchestrating work through GPT-6.1 Sol workers. The orchestrator splits work into slices, writes a brief for each worker, integrates the results, and verifies the whole. Independent slices run in parallel; dependent slices run in sequence.

[Landing page](https://cskwork.github.io/sol-skill/) | [Skill instructions](SKILL.md)

## Requirements

- Codex CLI installed and logged in with `codex login`.
- Access to `gpt-6.1-sol`.
- An agent host that loads skills and can launch Codex CLI workers.

The skill checks authentication once before launching workers:

```bash
command -v codex && test -f ~/.codex/auth.json
```

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

When the host already runs `gpt-6.1-sol` at high reasoning in Codex, it uses native subagents. Otherwise, it launches each worker with this command from `SKILL.md`:

```bash
codex exec -m gpt-6.1-sol -c model_reasoning_effort=high -c service_tier=fast \
  -s workspace-write --skip-git-repo-check -C "<repo>" \
  -o "<scratch>/<slice>.out.md" - < "<scratch>/<slice>.brief.md"
```

Replace `<repo>`, `<scratch>`, and `<slice>` with the repository path, scratch directory, and slice name.

## Worker briefs and verification

Each brief defines one objective, owned paths, interfaces to preserve, constraints, acceptance checks, and verified facts with file and line references. Workers report changed files, check results, and open problems once at the end.

The orchestrator reads the results, checks that changes stay within owned paths, and runs acceptance checks itself. A failed slice gets one re-brief with failure evidence. A second failure is reported with what remains.

## Safety

- CLI workers run sandboxed with `-s workspace-write`. Never bypass approvals or the sandbox.
- Workers never commit to `main`, push, merge, or publish. The orchestrator performs those actions only when authorized.
- Independent slices share no files, state, or ordering.

## License

[MIT](LICENSE). Copyright (c) 2026 cskwork.
