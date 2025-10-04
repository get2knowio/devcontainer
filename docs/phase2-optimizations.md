# Phase 2: Dockerfile Optimization - Intermediate Space Savings

## Overview
This document describes the Phase 2 optimizations implemented to reduce the DevContainer image size and improve build efficiency.

## Optimization Summary

### Layer Reduction
- **Before:** 14 RUN statements
- **After:** 10 RUN statements
- **Reduction:** 28% fewer layers

### Key Improvements

#### 1. Consolidated RUN Commands
Combined related operations into single RUN statements to reduce layers:

- **Starship + Shell Configuration:** Merged 2 RUN statements into 1
  - Combined Starship installation with nvm bootstrap and shell aliases
  - Added cleanup operations to the same layer
  
- **Node.js Installation:** Merged 2 RUN statements into 1
  - Combined Node.js installation with nvm use command
  - Added aggressive cache cleanup
  
- **Bun + Shell Aliases:** Merged 3 RUN statements into 1
  - Combined Bun installation with TypeScript and Rust aliases
  - All shell configuration in one layer

#### 2. Build Mount Optimization
Added tmpfs mounts to prevent temporary data from being committed to layers:

- Heavy tools installation: `--mount=type=tmpfs,target=/tmp`
- Node.js installation: `--mount=type=tmpfs,target=/tmp`
- Poetry installation: `--mount=type=tmpfs,target=/tmp`
- Rust tools installation: `--mount=type=tmpfs,target=/tmp`

**Total tmpfs mounts added:** 4

#### 3. Aggressive Cache Management
Enhanced cleanup operations throughout the Dockerfile:

- Cleanup operations: 9 instances of `rm -rf`
- npm cache clean after installation
- Removal of temporary caches: `${HOME}/.cache/*`
- Consistent cleanup of `/tmp/*` and `/var/tmp/*`

#### 4. Package Selection Optimization
Made heavy tools individually configurable with build args:

```dockerfile
ARG INSTALL_ACT=true
ARG INSTALL_ACTIONLINT=true
ARG INSTALL_AST_GREP=true
ARG INSTALL_ZELLIJ=true
ARG INSTALL_LAZYGIT=true
ARG INSTALL_GH=true
```

Users can now selectively disable individual tools:
```bash
docker build --build-arg INSTALL_ZELLIJ=false --build-arg INSTALL_ACT=false .
```

## Configuration Options

### devcontainer.json
Updated with explicit build args:

```json
{
  "build": {
    "args": {
      "INSTALL_HEAVY_TOOLS": "true",
      "INSTALL_ACT": "true",
      "INSTALL_ACTIONLINT": "true",
      "INSTALL_AST_GREP": "true",
      "INSTALL_ZELLIJ": "true",
      "INSTALL_LAZYGIT": "true",
      "INSTALL_GH": "true",
      "INSTALL_AI_CLIS": "true",
      "UPDATE_NPM": "true"
    }
  }
}
```

## Expected Impact

### Space Savings
- **Layer Reduction:** ~28% fewer layers (14 → 10)
- **Tmpfs Usage:** Prevents ~1-2GB of temporary files from being committed to layers
- **Cache Cleanup:** More aggressive cleanup reduces layer bloat

### Build Performance
- **Better Caching:** Build mounts and tmpfs improve cache hit rates
- **Parallel Builds:** Fewer layers can improve parallel build performance
- **Configurable Tools:** Skip unnecessary tools for faster, smaller builds

## Risks & Mitigation

### Increased Build Complexity
- **Risk:** Single large RUN statements are harder to debug
- **Mitigation:** Added echo statements for each major operation
- **Mitigation:** tmpfs mounts are independent and can be removed if issues arise

### Tool Compatibility
- **Risk:** Aggressive cleanup might remove needed files
- **Mitigation:** Only removes standard temporary directories
- **Mitigation:** Cache mounts preserve build artifacts between runs

### Debug Difficulty
- **Risk:** Cannot inspect intermediate layer states
- **Mitigation:** Can still build without cache for debugging
- **Mitigation:** Individual tool installation can be disabled for testing

## Testing Strategy

1. **Syntax Validation:** ✅ Docker buildx --check
2. **Full Build Test:** Test complete build on local platform
3. **Multi-arch Build:** Test on both amd64 and arm64
4. **Functionality Test:** Run existing test suite
5. **Size Comparison:** Compare final image sizes

## Next Steps

### Phase 3 (Future)
- Multi-stage builds for further size reduction
- Separate builder stages for tools
- Runtime-only final stage

### Monitoring
- Track actual image size reduction in CI/CD
- Monitor build times for performance impact
- Gather user feedback on configurable options

## Related Documentation
- [CONTRIBUTING.md](../CONTRIBUTING.md) - Build and development guidelines
- [README.md](../README.md) - User-facing documentation
- Issue #19 (Phase 1) - Prerequisites
- Issue #20 (Phase 3) - Multi-stage builds (blocked by this phase)
