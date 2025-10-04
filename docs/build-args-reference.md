# Build Arguments Reference

This document describes all available build arguments for customizing your DevContainer image.

## Quick Reference

| Build Arg | Default | Purpose | Size Impact |
|-----------|---------|---------|-------------|
| `INSTALL_HEAVY_TOOLS` | `true` | Master switch for all heavy tools | ~500MB |
| `INSTALL_ACT` | `true` | GitHub Actions emulator | ~50MB |
| `INSTALL_ACTIONLINT` | `true` | GitHub Actions workflow linter | ~10MB |
| `INSTALL_AST_GREP` | `true` | AST-based code search tool | ~15MB |
| `INSTALL_ZELLIJ` | `true` | Terminal multiplexer | ~20MB |
| `INSTALL_LAZYGIT` | `true` | Git TUI | ~15MB |
| `INSTALL_GH` | `true` | GitHub CLI | ~30MB |
| `INSTALL_AI_CLIS` | `true` | Gemini, Claude, Codex CLIs | ~200MB |
| `UPDATE_NPM` | `true` | Update npm to latest version | Minimal |

## Usage Examples

### Default Build (All Tools)
```bash
docker build -t devcontainer:latest containers/default/
```

### Minimal Build (Essential Tools Only)
```bash
docker build \
  --build-arg INSTALL_HEAVY_TOOLS=false \
  --build-arg INSTALL_AI_CLIS=false \
  -t devcontainer:minimal \
  containers/default/
```
*Saves ~700MB*

### Custom Build (Selective Tools)

#### Skip GitHub Actions Tools
```bash
docker build \
  --build-arg INSTALL_ACT=false \
  --build-arg INSTALL_ACTIONLINT=false \
  -t devcontainer:no-actions \
  containers/default/
```
*Saves ~60MB*

#### Skip Terminal Multiplexers
```bash
docker build \
  --build-arg INSTALL_ZELLIJ=false \
  -t devcontainer:no-zellij \
  containers/default/
```
*Saves ~20MB (tmux remains available via apt)*

#### Development Build (No AI Tools)
```bash
docker build \
  --build-arg INSTALL_AI_CLIS=false \
  -t devcontainer:dev \
  containers/default/
```
*Saves ~200MB*

#### CI/CD Optimized Build
```bash
docker build \
  --build-arg INSTALL_ZELLIJ=false \
  --build-arg INSTALL_LAZYGIT=false \
  --build-arg INSTALL_AI_CLIS=false \
  -t devcontainer:ci \
  containers/default/
```
*Saves ~235MB, keeps gh and act for CI workflows*

## Tool Descriptions

### Heavy Tools (INSTALL_HEAVY_TOOLS)
Master switch that controls all optional heavy tools. When set to `false`, individual tool flags are ignored.

**Tools Controlled:**
- act (GitHub Actions emulator)
- actionlint (workflow linter)
- ast-grep (code search)
- zellij (terminal multiplexer)
- lazygit (git TUI)
- gh (GitHub CLI)

**When to disable:** Building minimal images or CI environments that don't need these tools.

### ACT (INSTALL_ACT)
GitHub Actions emulator for local workflow testing.

**When to disable:** Not using GitHub Actions, or only need validation (actionlint).

### Actionlint (INSTALL_ACTIONLINT)
Validates GitHub Actions workflow files.

**When to disable:** Not using GitHub Actions.

### AST-grep (INSTALL_AST_GREP)
AST-based code search and refactoring tool.

**When to disable:** Not performing complex code searches or refactoring.

### Zellij (INSTALL_ZELLIJ)
Modern terminal multiplexer (alternative to tmux).

**Note:** tmux is still available via apt even if zellij is disabled.

**When to disable:** Prefer tmux only, or don't use terminal multiplexing.

### Lazygit (INSTALL_LAZYGIT)
Terminal UI for git operations.

**When to disable:** Prefer command-line git or other git GUIs.

### GitHub CLI (INSTALL_GH)
Official GitHub command-line tool.

**When to disable:** Not interacting with GitHub from CLI.

### AI CLIs (INSTALL_AI_CLIS)
Command-line interfaces for AI services:
- Google Gemini CLI (`gemini`)
- Anthropic Claude CLI (`claude`)
- OpenAI Codex CLI (`codex`)

**When to disable:** Not using AI tools, or prefer web interfaces.

### NPM Update (UPDATE_NPM)
Updates npm to the latest version after Node.js installation.

**When to disable:** Want to use the npm version that comes with Node.js LTS.

## Configuration in devcontainer.json

Update your devcontainer.json to persist custom build args:

```json
{
  "build": {
    "dockerfile": "../Dockerfile",
    "context": "..",
    "args": {
      "INSTALL_HEAVY_TOOLS": "true",
      "INSTALL_ACT": "false",
      "INSTALL_ACTIONLINT": "false",
      "INSTALL_AST_GREP": "true",
      "INSTALL_ZELLIJ": "false",
      "INSTALL_LAZYGIT": "true",
      "INSTALL_GH": "true",
      "INSTALL_AI_CLIS": "false",
      "UPDATE_NPM": "true"
    }
  }
}
```

## Environment Variables vs Build Args

**Build Args** (ARG):
- Set at **build time**
- Determine what gets installed in the image
- Cannot be changed after build

**Environment Variables** (ENV):
- Set at **runtime**
- Control application behavior
- Can be changed in running containers

## Always Installed

These tools are **always** installed regardless of build args:

### Essential System Tools
- build-essential
- bat, ripgrep, fd-find
- jq, fzf
- curl, ca-certificates
- neovim, tmux
- zoxide (smart cd)

### Core Development Tools
- Python 3.12 + venv
- Node.js LTS (via nvm)
- npm, pnpm, yarn, bun
- TypeScript + ecosystem
- Rust (via devcontainer feature)
- Poetry (Python package manager)

### Shell Enhancements
- eza (modern ls)
- starship (prompt)
- zsh with completions

### DevContainer Features
- Docker-in-Docker
- AWS CLI
- jq-likes tools
- uv (Python package installer)

## Recommended Configurations

### Full-Stack Developer
```dockerfile
INSTALL_HEAVY_TOOLS=true
INSTALL_AI_CLIS=true
```
*Includes all tools for maximum productivity*

### Python Developer
```dockerfile
INSTALL_HEAVY_TOOLS=false
INSTALL_GH=true
INSTALL_AI_CLIS=false
```
*Focuses on Python, keeps GitHub CLI*

### TypeScript/Node.js Developer
```dockerfile
INSTALL_HEAVY_TOOLS=false
INSTALL_GH=true
INSTALL_AI_CLIS=true
```
*Optimized for TS/Node with AI tools*

### Rust Developer
```dockerfile
INSTALL_HEAVY_TOOLS=false
INSTALL_GH=true
INSTALL_AI_CLIS=false
```
*Minimal setup with GitHub integration*

### CI/CD Pipeline
```dockerfile
INSTALL_HEAVY_TOOLS=true
INSTALL_ACT=true
INSTALL_ACTIONLINT=true
INSTALL_ZELLIJ=false
INSTALL_LAZYGIT=false
INSTALL_AI_CLIS=false
```
*Includes GitHub Actions tools, skips interactive tools*

## Size Optimization Tips

1. **Start Minimal:** Begin with `INSTALL_HEAVY_TOOLS=false` and add tools as needed
2. **Test Locally:** Build with different configurations to find your optimal setup
3. **Use Multi-Stage:** Consider Phase 3 multi-stage builds for maximum size reduction
4. **Layer Caching:** Build args don't affect cache until the ARG is used
5. **Combine Builds:** Use build cache to quickly test different configurations

## Troubleshooting

### Tool Not Found After Build
**Cause:** Build arg set to `false`  
**Solution:** Rebuild with tool enabled, or install manually in running container

### Build Taking Too Long
**Cause:** Installing all optional tools  
**Solution:** Disable unused tools with build args

### Image Too Large
**Cause:** All optional tools installed  
**Solution:** Use minimal configuration, disable AI CLIs and heavy tools

### Tests Failing
**Cause:** Required tools disabled  
**Solution:** Check test.sh for required tools, ensure they're enabled

## Version Updates

Tool versions are controlled by ARG declarations in the Dockerfile:
```dockerfile
ARG ACT_VERSION=0.2.81
ARG ACTIONLINT_VERSION=1.7.7
ARG AST_GREP_VERSION=0.39.5
ARG ZELLIJ_VERSION=0.43.1
ARG LAZYGIT_VERSION=0.55.1
ARG GH_VERSION=2.80.0
```

To update a version:
```bash
docker build --build-arg ACT_VERSION=0.2.82 containers/default/
```

## Further Reading

- [Phase 2 Optimizations](./phase2-optimizations.md) - Implementation details
- [Before & After Comparison](./phase2-before-after.md) - Detailed changes
- [CONTRIBUTING.md](../CONTRIBUTING.md) - Development guidelines
- [README.md](../README.md) - User documentation
