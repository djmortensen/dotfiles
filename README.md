# Dotfiles

chezmoi-managed dev environment bootstrap. v1 — minimal viable; expanded over time. See [docs/2026-06-02-design.md](docs/2026-06-02-design.md) for the full design and [docs/2026-06-03-dev-environment-bootstrap-plan.md](docs/2026-06-03-dev-environment-bootstrap-plan.md) for the full plan (features beyond v1 are deferred there).

## Bootstrap a new Mac

### Manual prerequisites (before the one-liner)

1. **Sign into the Mac App Store** app (Apple ID).
2. **Have your 1Password credentials** ready (account URL, email, secret key, master password).

### One-liner

````bash
sh -c "$(curl -fsSL get.chezmoi.io)" -- init --apply dj-morty/dotfiles
````

chezmoi will:
- Prompt for machine type (`work` or `home`)
- Install Homebrew (if missing)
- Run `brew bundle` for all formulae + casks: 1Password (app + CLI), Docker, Ghostty, VSCode, claude-code, gemini, gh, mise, pnpm, jq, gnupg, gcloud-cli, etc.
- Install VSCode extensions (Salesforce, ESLint, Prettier, Python, Java)
- Try to clone OVO repos (skips if `gh` not yet authed — that's fine, re-run later)
- Write `~/.gitconfig` with the right email for this machine

### Manual follow-ups (after the one-liner)

In this order:

1. **Sign into 1Password** — open the 1Password macOS app, complete account sign-in. Then in a terminal:
   ````bash
   op account add   # only if `op account list` is empty
   eval "$(op signin)"
   ````
2. **Authenticate `gh`** (HTTPS, browser flow):
   ````bash
   gh auth login --hostname github.com --git-protocol https --web
   ````
3. **Re-run chezmoi apply** — now clones the OVO repos:
   ````bash
   chezmoi apply --verbose
   ````
4. **gcloud auth** (only if you need cloud work):
   ````bash
   gcloud auth login
   gcloud auth application-default login
   gcloud config set project <your-default-project>
   ````
5. **Salesforce CLI** (for Salesforce work):
   ````bash
   sf org login web
   ````
6. **Install per-repo mise tools** — for each cloned repo:
   ````bash
   cd ~/ovo-repos/<repo> && mise install
   ````

### Optional
- **Magnet** (window manager) — install from the Mac App Store.
- **Keyboard repeat rate** — System Settings → Keyboard, drag both sliders to the right.
- **Accessibility permissions** — System Settings → Privacy & Security → Accessibility, enable for Raycast / Magnet / any other tools that ask.

## Day-to-day

| Action | Command |
|---|---|
| Edit a managed dotfile | `chezmoi edit ~/.gitconfig` |
| Preview pending changes | `chezmoi diff` |
| Apply pending changes | `chezmoi apply` |
| Pull updates from the other machine | `chezmoi update` |
| Add a brew formula/cask | edit `.chezmoidata/packages.yaml`, then `chezmoi apply` |
| Add a VSCode extension | edit `.chezmoidata/vscode-extensions.yaml`, then `chezmoi apply` |
| Add an OVO repo to clone | edit `.chezmoidata/repos.yaml`, then `chezmoi apply` |

## Recovery

| Problem | Fix |
|---|---|
| A script fails mid-apply | Fix the underlying issue, then `chezmoi apply` again (resumes from failure) |
| Want to re-run a `run_once_*` script | `chezmoi state delete-bucket --bucket=scriptState && chezmoi apply` |
| Render dry-run (no scripts execute) | `chezmoi apply --dry-run` |
| Skip scripts this run | `chezmoi apply --exclude=scripts` |

## Not yet in v1 (see plan)

- 1Password-backed SSH keys
- GitHub PAT auto-fetched from 1Password (currently interactive)
- Claude MCP server registration
- gcloud auth scripting
- macOS `defaults write` automation
- Per-machine variance (work/home casks lists currently empty)
- Pre-commit hooks (shellcheck, render check)

These live in the [implementation plan](docs/2026-06-03-dev-environment-bootstrap-plan.md) and can be added incrementally without disturbing v1.
