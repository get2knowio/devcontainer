# Phase 2 Dockerfile Optimization - Before & After Comparison

## Quick Stats

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| RUN statements | 14 | 10 | 28% reduction |
| tmpfs mounts | 0 | 4 | +4 new mounts |
| Build args (configurable tools) | 2 | 9 | +7 new options |
| Cleanup operations | 8 | 9 | More aggressive |
| USER switches | 7 | 7 | No change |

## Detailed Changes

### 1. Shell Configuration Consolidation

**Before (2 RUN statements):**
```dockerfile
# Starship and shell setup
RUN curl -sS https://starship.rs/install.sh | sh -s -- -y && \
    # ... starship config ...
    echo 'alias la="eza -la --icons"' >> ${USER_HOME}/.zshrc

# Separate nvm bootstrap
RUN echo 'export NVM_DIR="$HOME/.nvm"' >> ${USER_HOME}/.zshrc && \
    echo '[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"' >> ${USER_HOME}/.zshrc && \
    chown -R ${USERNAME}:${USERNAME} ${USER_HOME}
```

**After (1 RUN statement):**
```dockerfile
RUN curl -sS https://starship.rs/install.sh | sh -s -- -y && \
    # ... starship config ...
    echo 'alias la="eza -la --icons"' >> ${USER_HOME}/.zshrc && \
    echo 'export NVM_DIR="$HOME/.nvm"' >> ${USER_HOME}/.zshrc && \
    echo '[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"' >> ${USER_HOME}/.zshrc && \
    chown -R ${USERNAME}:${USERNAME} ${USER_HOME} && \
    rm -rf /tmp/* /var/tmp/*
```

**Result:** 1 fewer layer, added cleanup

---

### 2. Node.js Installation with Cleanup

**Before (2 RUN statements):**
```dockerfile
# Node.js installation
RUN --mount=type=cache,target=/home/vscode/.cache,uid=1000,gid=1000 \
    --mount=type=cache,target=/home/vscode/.npm,uid=1000,gid=1000 \
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v${NVM_VERSION}/install.sh | bash && \
    # ... npm installs ...
    npm cache clean --force && \
    rm -rf /tmp/* /var/tmp/*

# Separate nvm use command
RUN echo 'nvm use --silent default >/dev/null 2>&1 || nvm use --silent --lts >/dev/null 2>&1' >> ${USER_HOME}/.zshrc
```

**After (1 RUN statement with tmpfs):**
```dockerfile
RUN --mount=type=cache,target=/home/vscode/.cache,uid=1000,gid=1000 \
    --mount=type=cache,target=/home/vscode/.npm,uid=1000,gid=1000 \
    --mount=type=tmpfs,target=/tmp \
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v${NVM_VERSION}/install.sh | bash && \
    # ... npm installs ...
    echo 'nvm use --silent default >/dev/null 2>&1 || nvm use --silent --lts >/dev/null 2>&1' >> ${USER_HOME}/.zshrc && \
    npm cache clean --force && \
    rm -rf ${HOME}/.cache/* && \
    rm -rf /var/tmp/*
```

**Result:** 1 fewer layer, tmpfs prevents temp files in layer, aggressive cache cleanup

---

### 3. Bun + Shell Aliases Consolidation

**Before (3 RUN statements):**
```dockerfile
# Bun installation
RUN curl -fsSL https://bun.sh/install | bash && \
    echo 'export PATH="$HOME/.bun/bin:$PATH"' >> ${USER_HOME}/.zshrc && \
    rm -rf /tmp/* /var/tmp/*

# TypeScript aliases
RUN echo '# TypeScript development aliases' >> ${USER_HOME}/.zshrc && \
    echo 'alias tsc="npx tsc"' >> ${USER_HOME}/.zshrc && \
    # ... more TS aliases ...
    echo 'if command -v npm >/dev/null 2>&1; then eval "$(npm completion zsh)"; fi' >> ${USER_HOME}/.zshrc

# Rust aliases
RUN echo '# Rust development aliases' >> ${USER_HOME}/.zshrc && \
    echo 'alias cr="cargo run"' >> ${USER_HOME}/.zshrc && \
    # ... more Rust aliases ...
    echo 'fi' >> ${USER_HOME}/.zshrc
```

**After (1 RUN statement):**
```dockerfile
RUN curl -fsSL https://bun.sh/install | bash && \
    echo 'export PATH="$HOME/.bun/bin:$PATH"' >> ${USER_HOME}/.zshrc && \
    echo '# TypeScript development aliases' >> ${USER_HOME}/.zshrc && \
    echo 'alias tsc="npx tsc"' >> ${USER_HOME}/.zshrc && \
    # ... all TS aliases ...
    echo '# Rust development aliases' >> ${USER_HOME}/.zshrc && \
    echo 'alias cr="cargo run"' >> ${USER_HOME}/.zshrc && \
    # ... all Rust aliases ...
    echo 'fi' >> ${USER_HOME}/.zshrc && \
    rm -rf /tmp/* /var/tmp/*
```

**Result:** 2 fewer layers, all shell config in one place

---

### 4. Heavy Tools with Individual Control

**Before:**
```dockerfile
ARG INSTALL_HEAVY_TOOLS
RUN if [ "${INSTALL_HEAVY_TOOLS}" = "true" ]; then \
        # Install all tools or none
        # ... act installation ...
        rm -rf /tmp/* /var/tmp/* && \
        # ... actionlint installation ...
        rm -rf /tmp/* /var/tmp/* && \
        # ... repeated for each tool ...
    fi
```

**After (with granular control and tmpfs):**
```dockerfile
ARG INSTALL_HEAVY_TOOLS
ARG INSTALL_ACT=true
ARG INSTALL_ACTIONLINT=true
ARG INSTALL_AST_GREP=true
ARG INSTALL_ZELLIJ=true
ARG INSTALL_LAZYGIT=true
ARG INSTALL_GH=true
RUN --mount=type=tmpfs,target=/tmp \
    if [ "${INSTALL_HEAVY_TOOLS}" = "true" ]; then \
        if [ "${INSTALL_ACT}" = "true" ]; then \
            # ... act installation ...
        fi && \
        if [ "${INSTALL_ACTIONLINT}" = "true" ]; then \
            # ... actionlint installation ...
        fi && \
        # ... other tools ...
    fi && \
    rm -rf /var/tmp/*
```

**Result:** Granular control, tmpfs prevents temp files from being committed, single cleanup at end

---

### 5. Poetry & Rust with tmpfs

**Before:**
```dockerfile
# Poetry
RUN --mount=type=cache,target=/root/.cache/pip \
    curl -sSL https://install.python-poetry.org | python3 - --version ${POETRY_VERSION} && \
    ln -s ${POETRY_HOME}/bin/poetry /usr/local/bin/poetry && \
    rm -rf /tmp/* /var/tmp/*

# Rust tools
RUN --mount=type=cache,target=${USER_HOME}/.cargo/registry \
    --mount=type=cache,target=${USER_HOME}/.cargo/git \
    cargo install cargo-watch cargo-edit cargo-audit 2>/dev/null || \
    echo "Note: Some cargo tools may fail to install during build but will be available at runtime" && \
    rm -rf /tmp/* /var/tmp/*
```

**After:**
```dockerfile
# Poetry
RUN --mount=type=cache,target=/root/.cache/pip \
    --mount=type=tmpfs,target=/tmp \
    curl -sSL https://install.python-poetry.org | python3 - --version ${POETRY_VERSION} && \
    ln -s ${POETRY_HOME}/bin/poetry /usr/local/bin/poetry && \
    rm -rf /var/tmp/*

# Rust tools  
RUN --mount=type=cache,target=${USER_HOME}/.cargo/registry \
    --mount=type=cache,target=${USER_HOME}/.cargo/git \
    --mount=type=tmpfs,target=/tmp \
    cargo install cargo-watch cargo-edit cargo-audit 2>/dev/null || \
    echo "Note: Some cargo tools may fail to install during build but will be available at runtime" && \
    rm -rf /var/tmp/*
```

**Result:** tmpfs prevents temporary build artifacts from bloating layers

---

## Build Args Comparison

### Before
```dockerfile
ARG INSTALL_AI_CLIS=true
ARG INSTALL_HEAVY_TOOLS=true
```

### After
```dockerfile
ARG INSTALL_AI_CLIS=true
ARG INSTALL_HEAVY_TOOLS=true
ARG INSTALL_ACT=true
ARG INSTALL_ACTIONLINT=true
ARG INSTALL_AST_GREP=true
ARG INSTALL_ZELLIJ=true
ARG INSTALL_LAZYGIT=true
ARG INSTALL_GH=true
ARG UPDATE_NPM=true
```

**Usage Examples:**

```bash
# Minimal build - skip all optional tools
docker build --build-arg INSTALL_HEAVY_TOOLS=false .

# Keep gh and lazygit, skip the rest
docker build \
  --build-arg INSTALL_ACT=false \
  --build-arg INSTALL_ACTIONLINT=false \
  --build-arg INSTALL_AST_GREP=false \
  --build-arg INSTALL_ZELLIJ=false \
  .

# Skip AI CLIs for smaller image
docker build --build-arg INSTALL_AI_CLIS=false .
```

---

## Expected Benefits

### Space Savings
- **tmpfs mounts (4):** ~1-2GB saved by not committing temporary files
- **Layer consolidation (28% reduction):** ~2-4GB saved from reduced layer overhead
- **Aggressive cleanup:** Additional ~1-2GB from better cache management
- **Total estimated:** ~4-8GB reduction (will be measured in CI)

### Build Performance
- **Fewer layers:** Faster image pulls and builds
- **tmpfs mounts:** Faster I/O for temporary operations
- **Better caching:** Build mounts preserve important caches

### Flexibility
- **Granular control:** Users can create minimal images by disabling unneeded tools
- **Easy customization:** Clear build args make it obvious what can be configured
- **No breaking changes:** Default configuration keeps all tools enabled

---

## Testing Impact

All existing tests remain compatible because:
- Default configuration enables all tools
- No tools were removed, only made optional
- Shell aliases and configurations preserved
- No changes to tool versions or locations

Tests expect these tools:
- ✅ bat, rg, fd, jq, fzf, eza (essential tools - always installed)
- ✅ starship, zoxide (shell enhancements - always installed)
- ✅ tmux, zellij (terminal multiplexers - enabled by default)
- ✅ gh, lazygit (git tools - enabled by default)
- ✅ node, npm, pnpm, yarn, bun (Node ecosystem - always installed)
- ✅ rustc, cargo (Rust ecosystem - via devcontainer feature)

---

## Migration Guide

### For Users
No changes needed! The default configuration matches the previous behavior.

### For Custom Builds
To reduce image size, add build args to your build command:

```bash
# Example: Minimal dev environment
docker build \
  --build-arg INSTALL_HEAVY_TOOLS=true \
  --build-arg INSTALL_ACT=false \
  --build-arg INSTALL_ZELLIJ=false \
  .
```

Or update devcontainer.json:

```json
{
  "build": {
    "args": {
      "INSTALL_ACT": "false",
      "INSTALL_ZELLIJ": "false"
    }
  }
}
```

---

## Conclusion

Phase 2 optimizations deliver:
- ✅ 28% fewer layers (14 → 10)
- ✅ 4 tmpfs mounts for cleaner builds
- ✅ 9 configurable build args for flexibility
- ✅ More aggressive cleanup (9 operations)
- ✅ No breaking changes
- ✅ All tests remain compatible

Ready for CI/CD validation and actual image size measurement!
