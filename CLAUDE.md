# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **literate Emacs configuration** using Org-Babel. Configuration is written in `.org` files with embedded Elisp code blocks that get tangled (extracted) to `.el` files.

## Build Process

Tangle org files to elisp:
```sh
emacs --batch --eval '(require (quote org))' \
      --eval '(require (quote ob))' \
      --eval '(org-babel-load-file (expand-file-name "~/.emacs.d/init.org"))'
```

First-time setup with debug output:
```sh
emacs --debug-init
```

**Auto-tangle**: Org files auto-tangle on save when Emacs is running (configured in init.org).

## Architecture

### Configuration Hierarchy

All modules are loaded at startup via `org-babel-load-file`:

```
init.org → init.el (main entry point)
├── emacs24.helm.org      - Helm completion framework
├── emacs24.smartparens.org - Structural editing
├── emacs24.python.org    - Python dev (Eglot + ty + ruff)
├── emacs24.sql.org       - SQL (sqlfluff formatter)
├── emacs24.eshell.org    - Shell with Git prompt
├── emacs24.elisp.org     - Elisp development
├── emacs24.ess.org       - R/statistics
├── emacs24.js2.org       - JavaScript
├── emacs24.scheme.org    - Scheme
├── emacs24.hacky.org     - Misc utilities
├── emacs29.llm.org       - LLM/AI integration (gptel)
└── private.org           - API keys (symlinked, not in repo)
```

Note: "emacs24" naming is historical, not version-specific. Assumes Emacs 30+.

### Key Sections in init.org

| Lines | Content |
|-------|---------|
| 1-150 | Core setup: GC tuning, keybindings, platform detection, straight.el bootstrap |
| 150-450 | Org mode configuration |
| 374-450 | External module loading via `org-babel-load-file` |
| 450-2100 | Language modes, UI packages, navigation |
| 2100+ | Themes, finalization hooks |

### Package Management

Uses `straight.el` with `use-package` for declarative package configuration:
```elisp
(straight-use-package 'use-package)
(setq straight-use-package-by-default t)
```

Packages are cloned to `straight/repos/` and built to `straight/build/`.

## Working with This Codebase

### Editing Configuration

1. Edit the `.org` file (not the `.el` directly)
2. Code blocks with `:tangle yes` are extracted to elisp
3. Save triggers auto-tangle when Emacs is running
4. Restart Emacs or `eval-buffer` the tangled `.el` to apply changes

### Finding Configuration

Search in org files for configuration:
```sh
grep -r "use-package PACKAGE-NAME" *.org
```

Or use Emacs directly via MCP:
```elisp
(apropos-internal "config-term" 'boundp)
(describe-function 'some-function)
```

## Emacs MCP Integration

The `.claude/skills/emacs/SKILL.md` documents the `mcp__emacs__emacs_eval` tool for interacting with the running Emacs instance. Use it for:
- LSP-aware refactoring
- Runtime state inspection
- Buffer operations
- Documentation lookup (`describe-function`, `describe-variable`)

## Runtime Data (gitignored)

- `straight/repos/`, `straight/build/` - Package caches
- `eln-cache/` - Native compilation cache
- `emacs-backups/` - File backup history
- `keyfreq` - Key frequency statistics
- `recentf` - Recent files list
- `bookmarks` - Bookmark locations

## TRAMP / Remote File Policy

**Remote files are deliberately dumb.** TRAMP to high-latency hosts
(e.g. patAvailabilityVM, ~130ms RTT) cannot sustain per-ancestor
directory walks or LSP traffic, so all project machinery is severed
for remote paths (init.org, "Remote kill-switches" section; commit
4d60da4, with history in 9133580 and adb91cb):

- **projectile**: `projectile-project-root` returns nil for remote
  dirs; remote projects never tracked in `projectile-known-projects`.
- **project.el**: `project-current` returns nil for remote dirs.
- **eglot**: `eglot-ensure`/`eglot` are no-ops in remote buffers.
- **flymake/flycheck/yasnippet/auto-revert**: disabled in remote
  buffers via `find-file-hook` guards + `my-guard-remote-mode-change`
  (re-applied after major-mode changes, depth 90).
- **vc**: inert remotely via `vc-ignore-dir-regexp` including
  `tramp-file-name-regexp` (long-standing).
- **dired**: `get-free-disk-space` (remote `df`) skipped. Note:
  setting `dired-free-space` to nil does NOT prevent the df call;
  the advice on `get-free-disk-space` is required.
- **Zombie connection reaper**: `my-tramp-reap-dead-connections`
  timer (30s). Vector must be read with
  `(process-get proc 'tramp-vector)` — `tramp-get-connection-property`
  returns nil for it.

Consequences: remote buffers have no project commands, no LSP, no
diagnostics. Do not "fix" this by re-enabling any of the above for
remote paths. Expected baseline: remote file open ~0.7s
(4 round-trips), cold dired ~0.5–1s.

## Known Technical Debt

- `emacs24.eshell.org`: Git status runs shell commands per keystroke (performance issue)
- `emacs24.hacky.org`: `date2unix` defined twice; legacy keyboard macros
- `emacs24.scheme.org`: Uses unmaintained xscheme
- `emacs24.js2.org`: js2-mode superseded by js-ts-mode
