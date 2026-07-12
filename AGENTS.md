# AGENTS.md

## Package manager

Use **bun** for all installs and scripts (`bun install`, `bun add`, `bun run …`). Do not use npm, yarn, or pnpm, and do not create or commit `package-lock.json`.

After changing dependencies, update and commit `bun.lock` (CI runs `bun install --frozen-lockfile`).

## Commits

Use [Conventional Commits](https://www.conventionalcommits.org/), lowercase:

- `feat: …` — new feature
- `fix: …` — bug fix
- `chore: …` — tooling, deps, misc
- `docs: …` — documentation
- `refactor: …` — code change with no behavior change
- `style: …` — formatting only
- `test: …` — tests

Examples: `feat: add archive link`, `fix: sticky footer overflow`, `chore: add simple-icons`.
