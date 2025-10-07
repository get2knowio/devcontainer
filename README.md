# get2know.io DevContainer

Single multi-language development container with modern tooling for Python, TypeScript, and Rust. One image. One workflow. Less maintenance.

## 🧰 Tooling & Features Inventory
Comprehensive list of what the image bakes in (multi-arch: linux/amd64 & linux/arm64). Items sourced either from the upstream base, devcontainer features, or the Dockerfile.

Language & Runtimes:
- Python 3.12 (base image) + `pip`, `venv`, `poetry` (installed globally; in-project virtualenvs enabled)
- Node (via `nvm` LTS) + global package managers: `npm`, `pnpm`, `yarn`, `bun`
- Rust (via feature: `ghcr.io/devcontainers/features/rust:1`) + `rustc`, `cargo`, `rust-analyzer`, `rustfmt`, `clippy`
- UV (Python package manager) via feature: `ghcr.io/jsburckhardt/devcontainer-features/uv:1`

TypeScript / JS Toolchain (globally installed):
- `typescript`, `ts-node`, `tsx`, `@types/node`, `nodemon`, `concurrently`, `vite`, `esbuild`, `prettier`, `eslint`, `@biomejs/biome`, `tsc-watch`

Rust Toolchain:
- `rustc`, `cargo` (compiler and package manager)
- `rust-analyzer` (language server for IDE support)
- `rustfmt` (code formatter)
- `clippy` (linter for better Rust code)
- `cargo-watch` (automatically run commands on file changes)
- `cargo-edit` (manage dependencies from command line)
- `cargo-audit` (security vulnerability scanner)

AI / LLM CLIs:
- `@google/gemini-cli`
- `@anthropic-ai/claude-code`
- `@openai/codex` (Codex CLI)
- `@github/copilot` (GitHub Copilot CLI)
- `opencode-ai` (OpenCode AI)

Dev & CI Utilities:
- Docker CLI (with in-container daemon from feature) + Buildx
- AWS CLI (feature: `ghcr.io/devcontainers/features/aws-cli:1`)
- `act` (GitHub Actions local runner)
- `actionlint` (GitHub Actions workflow linter)
- `ast-grep` + `sg` binaries (structural code search / rewriting)
- `neovim` (apt)
- `gh` (GitHub CLI for PRs/issues/releases)
- `lazygit` (terminal UI for advanced git workflows)

Modern Terminal UX:
- `zsh` (default) + `starship` prompt
- Terminal multiplexers: `tmux`, `zellij` (zellij fetched from GitHub release for amd64/arm64)
- Smart directory jumper: `zoxide`
- `eza` (ls replacement), `fzf`, `bat`, `ripgrep (rg)`, `fd`, `jq`

Other Tools / Helpers:
- `git` (up-to-date; may be source-built by base)
- `curl`, `wget`, `unzip`, `ca-certificates` (bundled / apt)

### Why include both `ast-grep` and `sg`?
Some distributions provide a smaller `sg` wrapper binary. The image installs **both** to ensure parity with official docs and avoid unexpected tool differences.

---

## ⚡ Shell Aliases
Convenience aliases injected into the default `zsh` environment (see Dockerfile). Use `which <name>` or `type <name>` to inspect. All are simple wrappers; adjust or extend in your own dotfiles as needed.

File / Directory Listing:
- `ls` → `eza --icons`
- `ll` → `eza -l --icons`
- `la` → `eza -la --icons`

TypeScript / Node Workflow:
- `tsc` → `npx tsc` (ensures local project version if present)
- `tsx` → `npx tsx`
- `tsw` → `npx tsc-watch`
- `dev` → `npm run dev`
- `build` → `npm run build`
- `test` → `npm test`
- `lint` → `npm run lint`
- `format` → `npm run format`

Rust Workflow:
- `cr` → `cargo run`
- `cb` → `cargo build`
- `ct` → `cargo test`
- `cc` → `cargo check`
- `cf` → `cargo fmt`
- `cl` → `cargo clippy`
- `cw` → `cargo watch`
- `cn` → `cargo new`
- `ca` → `cargo add`
- `cup` → `cargo update`

Notes:
- Aliases prefer project-local binaries via `npx` when applicable.
- Safe to override in your own `.zshrc` or extend with additional project automation.

---

## 📦 Example devcontainer.json
```jsonc
{
  "name": "get2know.io devcontainer",
  "image": "ghcr.io/get2knowio/devcontainer:latest",
  "remoteUser": "vscode",
  "features": {},
//   "postCreateCommand": "npm install",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-vscode.vscode-typescript-next",
        "esbenp.prettier-vscode",
        "dbaeumer.vscode-eslint"
      ]
    }
  },
  "initializeCommand": "docker pull ghcr.io/get2knowio/devcontainer:latest"
}
```

## ⚙️ Build Customization

The image supports several build arguments for customization:

- `INSTALL_AI_CLIS` (default: `true`) - Install AI CLI tools (Gemini, Claude, OpenAI Codex, GitHub Copilot)
- `INSTALL_HEAVY_TOOLS` (default: `true`) - Install heavy development tools (act, actionlint, ast-grep, zellij, lazygit, gh)

Example of building a minimal version without heavy tools:
```bash
docker build \
  --build-arg INSTALL_HEAVY_TOOLS=false \
  -t devcontainer:minimal \
  containers/default
```

This can save significant build time and image size for users who don't need these specific tools.

---

## 🚀 Quick Interactive Dev Container Shell

An example helper script is provided at `examples/devcontainer-enter.sh` to drop you into an interactive `zsh` inside a Dev Container for the current directory.

Usage:
```
./examples/devcontainer-enter.sh [id]
```
Where:
- `id` (optional) adds a label `devcontainer-example.id=<id>` so multiple sessions can coexist or be targeted.

Behavior:
- If a matching container is running (workspace + optional id) it just opens `zsh`.
- If a stopped matching container exists, it starts it, then opens `zsh`.
- If none exists, it performs `devcontainer up` to create one.
- On shell exit: if the script created the container this session, it stops (does not remove) the container for fast reuse; otherwise leaves it as-is.

Requirements:
- `devcontainer` CLI on PATH
- Docker daemon available
- `.devcontainer/` directory present in the workspace

This offers a repeatable "jump in / jump out" workflow that preserves the container (stopped) for rapid restart while avoiding resource use when idle.

---

## 📚 Further Reading / Contributing
Looking for build internals, CI, migration history, troubleshooting, or how to extend the image? See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## 📄 License
See LICENSE file.

---

## ♻️ Automated Dependency Updates (Renovate)
This repository uses [Renovate](https://docs.renovatebot.com) to keep DevContainer tooling current.

What it updates:
- Base image tag in `containers/default/Dockerfile` (Dockerfile manager)
- Tool version `ARG`s (via custom `regexManagers` in `renovate.json`): `NVM_VERSION`, `POETRY_VERSION`, `EZA_VERSION`, `ACT_VERSION`, `ACTIONLINT_VERSION`, `AST_GREP_VERSION`, `ZELLIJ_VERSION`, `LAZYGIT_VERSION`, `GH_VERSION`

Schedule:
- Weekly window: before 06:00 UTC every Monday (cron `0 4 * * 1`) keeps noise low.

Workflow:
1. GitHub Action (`.github/workflows/renovate.yml`) runs on schedule or manual dispatch.
2. Renovate opens/updates PRs; similar updates are grouped.
3. A Dependency Dashboard issue tracks pending upgrades.

Adjusting behavior:
- Change grouping or schedule in `renovate.json`.
- Trigger an ad‑hoc run: Actions tab → Renovate → Run workflow.
- Pin / ignore versions: add `packageRules` entries.

Authentication:
- Defaults to `GITHUB_TOKEN`; optionally add a PAT secret `RENOVATE_TOKEN` (scopes: `repo`, `workflow`) for higher rate limits.

Tips:
- Merge base image updates promptly; they often include security patches.
- Review grouped tooling PRs for changelog links (Renovate annotates release notes when available).

### Semantic Commit PR Titles
Renovate is configured with `:semanticCommits`, so PRs follow Conventional Commit prefixes:
- `feat(deps)!` – Major updates that may be breaking
- `feat(deps)` – Minor feature-level updates
- `fix(deps)` – Patch / bugfix-level updates
- `chore(deps)` – Non-code-impacting tasks (lockfile maintenance, pinning, etc.)

This improves downstream changelog or release automation compatibility.

### Major vs Minor/Patch Separation
Package rules split updates:
- Major upgrades: labeled `major devcontainer tooling upgrades` (require manual review)
- Minor & patch: grouped as `routine devcontainer tooling (minor+patch)`

Both sets share the same weekly schedule window but remain distinct for risk assessment. You can later enable selective automerge for safe patch updates by adding a rule with `"matchUpdateTypes": ["patch"], "automerge": true.

### Patch Automerge
Patch-level tooling updates are now auto-merged:
- Rule: labels include `patch` and `automerge` (see `renovate.json`).
- Commit style: `fix(deps): update <dep> to vX.Y.Z`.
- Safety rationale: Patch releases should be backward-compatible; still review occasionally for unexpected regressions.
- To disable: Remove `automerge` or set `"automerge": false` in that rule.
- To require status checks: add the required check names to `requiredStatusChecks` array.
