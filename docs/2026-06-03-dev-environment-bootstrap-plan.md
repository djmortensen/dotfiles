# Dev Environment Bootstrap — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a chezmoi-managed dotfiles repo that fully bootstraps a developer Mac (work or home) from a single command, and keeps the two machines in sync afterward.

**Architecture:** chezmoi used natively — templating for dotfiles, `.chezmoidata/*.yaml` for inventories, `run_once_*` / `run_onchange_*` scripts for installation steps. Secrets pulled from 1Password at script-execution time (not template-render time). Public personal GitHub repo. Per-machine variance via a single `{{ .machine }}` variable set by first-run prompt.

**Tech Stack:** chezmoi · bash · 1Password CLI (`op`) · Homebrew · `mas` · mise · npm · gh · gcloud · Claude CLI · pre-commit · shellcheck

**Spec:** `docs/2026-06-02-design.md`

**Working repo location:** `~/Repos/dotfiles` (chezmoi source dir symlinked from `~/.local/share/chezmoi` once initialised).

---

## Pre-flight

Verify the operator (you) has these before starting:

- [ ] **P1: Spec is committed in the repo.**

```bash
ls ~/Repos/dotfiles/docs/2026-06-02-design.md
git -C ~/Repos/dotfiles log --oneline | head -3
```

If missing, run the install commands from the brainstorm session to drop the spec in and make the first commit.

- [ ] **P2: chezmoi is installed.**

```bash
brew list chezmoi >/dev/null 2>&1 || brew install chezmoi
chezmoi --version
```

Expected: `chezmoi version v2.x.x …`

- [ ] **P3: 1Password CLI is installed and signed in.**

```bash
brew list 1password-cli >/dev/null 2>&1 || brew install 1password-cli
op account list
```

If `op account list` is empty, run `op account add` and complete the interactive sign-in. Subsequent commands assume `op` works.

- [ ] **P4: Pick a public GitHub repo name.** Default: `<your-handle>/dotfiles`.

Decide now; you'll need it for Task 19. (You don't need to create it on GitHub until then.)

---

## Phase A — Repo skeleton

### Task 1: Initialise repo with pre-commit and README stub

**Files:**
- Create: `~/Repos/dotfiles/.gitignore`
- Create: `~/Repos/dotfiles/.chezmoiignore`
- Create: `~/Repos/dotfiles/README.md`
- Create: `~/Repos/dotfiles/.pre-commit-config.yaml`
- Modify: `~/Repos/dotfiles/` (already a git repo from spec commit)

- [ ] **Step 1: Add `.gitignore`**

```gitignore
# chezmoi auto-generated state lives outside the repo, but be safe
.chezmoiscripts/
.chezmoistate/

# editor / OS scrap
.DS_Store
*.swp
.idea/
.vscode/
```

- [ ] **Step 1.5: Add `.chezmoiignore`**

Without this, chezmoi would treat `docs/`, `README.md`, and `.pre-commit-config.yaml` as files to manage under `$HOME`.

```
README.md
.pre-commit-config.yaml
docs/**
.github/**
```

- [ ] **Step 2: Add `README.md` stub** (full runbook lands in Task 18)

```markdown
# Dotfiles

Personal dev environment bootstrap and ongoing sync, managed by [chezmoi](https://www.chezmoi.io/).

## Fresh machine

Prerequisites:
1. Sign into the Mac App Store app (needed for `mas` to install paid apps).
2. Have your 1Password account credentials ready.

One-liner:

\`\`\`bash
sh -c "$(curl -fsSL get.chezmoi.io)" -- init --apply <your-handle>/dotfiles
\`\`\`

See [docs/2026-06-02-design.md](docs/2026-06-02-design.md) for the design.
```

- [ ] **Step 3: Add `.pre-commit-config.yaml`**

```yaml
repos:
  - repo: https://github.com/shellcheck-py/shellcheck-py
    rev: v0.10.0.1
    hooks:
      - id: shellcheck
        files: \.sh(\.tmpl)?$
        args: [--severity=warning]
  - repo: local
    hooks:
      - id: chezmoi-render
        name: chezmoi template render check
        entry: bash -c 'chezmoi execute-template --init --promptString machine=home < "$1" >/dev/null'
        language: system
        files: \.tmpl$
```

- [ ] **Step 4: Install pre-commit hooks**

```bash
cd ~/Repos/dotfiles
brew list pre-commit >/dev/null 2>&1 || brew install pre-commit
pre-commit install
```

Expected: `pre-commit installed at .git/hooks/pre-commit`

- [ ] **Step 5: Verify lint runs**

```bash
pre-commit run --all-files
```

Expected: shellcheck has no `.sh` files yet, render check has no `.tmpl` files yet — passes trivially.

- [ ] **Step 6: Commit**

```bash
cd ~/Repos/dotfiles
git add .gitignore .chezmoiignore README.md .pre-commit-config.yaml
git commit -m "chore: initial repo scaffolding"
```

---

### Task 2: Add `.chezmoi.toml.tmpl` — first-run machine prompt

**Files:**
- Create: `~/Repos/dotfiles/.chezmoi.toml.tmpl`

- [ ] **Step 1: Write the template**

```go
{{- /* Prompt once on init, persist to ~/.config/chezmoi/chezmoi.toml */ -}}
{{- $machine := promptStringOnce . "machine" "Which machine? (work or home)" "home" -}}

[data]
    machine = {{ $machine | quote }}
    [data.git]
        name  = "Daniel Mortensen"
        email = {{ if eq $machine "work" }}"daniel.mortensen@ovo.com"{{ else }}"daniel.mortensen@ovo.com"{{ end }}
```

(Both branches currently the same email — keep both branches in place so flipping is a one-line edit later.)

- [ ] **Step 2: Render the template to confirm it parses**

```bash
chezmoi execute-template --init --promptString machine=home < ~/Repos/dotfiles/.chezmoi.toml.tmpl
```

Expected: prints valid TOML with `machine = "home"` and the `[data.git]` block.

- [ ] **Step 3: Point chezmoi at this repo**

```bash
chezmoi init --source ~/Repos/dotfiles
```

This will prompt for `machine` (answer `home`), then write `~/.config/chezmoi/chezmoi.toml`.

Verify:

```bash
cat ~/.config/chezmoi/chezmoi.toml
chezmoi data | jq '.machine, .git'
```

Expected output: machine is `"home"`, git block populated.

- [ ] **Step 4: Commit**

```bash
git -C ~/Repos/dotfiles add .chezmoi.toml.tmpl
git -C ~/Repos/dotfiles commit -m "feat: machine-type prompt on init"
```

---

### Task 3: First dotfile — `dot_gitconfig.tmpl` proves the pipeline

This is the smallest possible dotfile that consumes both the machine variable and a data file. Choose `.gitconfig` because it's safe to overwrite (you'll always be able to reconstruct from memory).

**Files:**
- Create: `~/Repos/dotfiles/dot_gitconfig.tmpl`

- [ ] **Step 1: Back up your existing gitconfig**

```bash
cp ~/.gitconfig ~/.gitconfig.bak
```

- [ ] **Step 2: Write the template**

```ini
[user]
    name  = {{ .git.name }}
    email = {{ .git.email }}

[init]
    defaultBranch = main

[pull]
    rebase = false

[push]
    autoSetupRemote = true

[core]
    excludesfile = ~/.config/git/ignore
```

- [ ] **Step 3: Dry-run apply**

```bash
chezmoi diff
```

Expected: shows a diff between current `~/.gitconfig` and the rendered template.

- [ ] **Step 4: Apply for real**

```bash
chezmoi apply
diff ~/.gitconfig ~/.gitconfig.bak || true   # ok if different
```

Verify `~/.gitconfig` now contains your name + email correctly.

- [ ] **Step 5: Commit**

```bash
git -C ~/Repos/dotfiles add dot_gitconfig.tmpl
git -C ~/Repos/dotfiles commit -m "feat: managed .gitconfig"
```

---

## Phase B — System foundations

### Task 4: `run_once_before_00-install-homebrew.sh.tmpl`

**Files:**
- Create: `~/Repos/dotfiles/run_once_before_00-install-homebrew.sh.tmpl`

- [ ] **Step 1: Write the script**

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[install-homebrew] %s\n' "$*" >&2; }

if command -v brew >/dev/null 2>&1; then
  log "homebrew already installed at $(command -v brew) — skipping"
  exit 0
fi

log "installing homebrew (non-interactive)…"
NONINTERACTIVE=1 /bin/bash -c \
  "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Make brew immediately available in this shell
if [[ -x /opt/homebrew/bin/brew ]]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [[ -x /usr/local/bin/brew ]]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

log "done: $(brew --version | head -1)"
```

- [ ] **Step 2: Shellcheck it**

```bash
shellcheck ~/Repos/dotfiles/run_once_before_00-install-homebrew.sh.tmpl
```

Expected: no warnings.

- [ ] **Step 3: Render check**

```bash
chezmoi execute-template --init --promptString machine=home \
  < ~/Repos/dotfiles/run_once_before_00-install-homebrew.sh.tmpl | head -5
```

Expected: shebang + `set -euo pipefail` visible (no template syntax errors).

- [ ] **Step 4: Dry-run apply**

```bash
chezmoi apply --dry-run --verbose 2>&1 | grep -i homebrew || echo "would skip"
```

Expected: chezmoi notes the script would run, but `--dry-run` doesn't execute it.

- [ ] **Step 5: Real apply (no-op on this machine, brew already installed)**

```bash
chezmoi apply --verbose
```

Expected: script prints `homebrew already installed … skipping`.

- [ ] **Step 6: Commit**

```bash
git -C ~/Repos/dotfiles add run_once_before_00-install-homebrew.sh.tmpl
git -C ~/Repos/dotfiles commit -m "feat: install homebrew on fresh machines"
```

---

### Task 5: `.chezmoidata/packages.yaml` — initial inventory

**Files:**
- Create: `~/Repos/dotfiles/.chezmoidata/packages.yaml`

- [ ] **Step 1: Inventory the current home machine**

```bash
brew leaves              # top-level formulae (not deps)
brew list --cask
mas list 2>/dev/null || true  # only if mas already installed
```

Capture the lists.

- [ ] **Step 2: Write `packages.yaml` using the captured inventory**

Note: all entries are wrapped under a top-level `packages:` key so chezmoi exposes them as `.packages.*` in templates. Without this wrapping, the file's top-level keys would land at chezmoi data root (`.brew`, `.mas`, `.npm`), polluting the namespace.

```yaml
# Initial inventory — adjust as preferences evolve.
packages:
  brew:
    formulae:
      common:
        - jq
        - ripgrep
        - fzf
        - tree
        - gh
        - mise
        - 1password-cli
        - gnupg
        - mas
        - pre-commit
        - chezmoi
        - shellcheck
        # Add output of `brew leaves` here, minus anything you don't need
      work: []
      home: []
    casks:
      common:
        - ghostty
        - visual-studio-code
        - docker
        - 1password
        - raycast
        # Add output of `brew list --cask` here, minus anything you don't need
      work: []
      home:
        - zed  # staged for future use

  mas:
    common:
      - { id: 441258766, name: Magnet }

  # npm globals installed in run_onchange_55 — kept here so all package
  # inventories live in one file.
  npm:
    globals:
      - "@anthropic-ai/claude-code"
      - "@google/gemini-cli"  # verify exact name when Task 13 runs
```

- [ ] **Step 3: Sanity-render**

```bash
chezmoi data | jq '.packages'
```

Expected: prints the YAML as a JSON object with `brew`, `mas`, `npm` keys — confirms chezmoi auto-loads it under `.packages`.

- [ ] **Step 4: Commit**

```bash
git -C ~/Repos/dotfiles add .chezmoidata/packages.yaml
git -C ~/Repos/dotfiles commit -m "feat: declare package inventory"
```

---

### Task 6: `run_onchange_10-install-packages.sh.tmpl`

**Files:**
- Create: `~/Repos/dotfiles/run_onchange_10-install-packages.sh.tmpl`

- [ ] **Step 1: Write the script**

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[install-packages] %s\n' "$*" >&2; }

log "applying brew bundle for machine={{ .machine }}"

brew bundle --no-lock --file=- <<'EOF'
{{ range .packages.brew.formulae.common -}}
brew "{{ . }}"
{{ end -}}
{{ range (index .packages.brew.formulae .machine) -}}
brew "{{ . }}"
{{ end -}}
{{ range .packages.brew.casks.common -}}
cask "{{ . }}"
{{ end -}}
{{ range (index .packages.brew.casks .machine) -}}
cask "{{ . }}"
{{ end -}}
EOF

log "done"
```

- [ ] **Step 2: Shellcheck + render**

```bash
shellcheck ~/Repos/dotfiles/run_onchange_10-install-packages.sh.tmpl
chezmoi execute-template --init --promptString machine=home \
  < ~/Repos/dotfiles/run_onchange_10-install-packages.sh.tmpl
```

Expected: the rendered output shows a literal Brewfile body — every formula and cask from `packages.yaml` appears once, no `{{ … }}` markers remain.

- [ ] **Step 3: Diff**

```bash
chezmoi diff
```

Expected: shows the new script would run.

- [ ] **Step 4: Apply (mostly no-op on this machine since packages are installed)**

```bash
chezmoi apply --verbose
```

Expected: `brew bundle` reports `Using …` for each already-installed item.

- [ ] **Step 5: Verify idempotency**

```bash
chezmoi state delete-bucket --bucket=scriptState
chezmoi apply --verbose
```

Re-runs the script. Expected: same output, no errors.

- [ ] **Step 6: Commit**

```bash
git -C ~/Repos/dotfiles add run_onchange_10-install-packages.sh.tmpl
git -C ~/Repos/dotfiles commit -m "feat: install brew formulae and casks from packages.yaml"
```

---

### Task 7: `.chezmoidata/vscode-extensions.yaml` + `run_onchange_15-vscode-extensions.sh.tmpl`

**Files:**
- Create: `~/Repos/dotfiles/.chezmoidata/vscode-extensions.yaml`
- Create: `~/Repos/dotfiles/run_onchange_15-vscode-extensions.sh.tmpl`

- [ ] **Step 1: Inventory current extensions**

```bash
code --list-extensions
```

- [ ] **Step 2: Write `vscode-extensions.yaml`**

```yaml
extensions:
  common:
    - salesforce.salesforcedx-vscode
    - salesforce.salesforcedx-vscode-soql
    - dbaeumer.vscode-eslint
    - esbenp.prettier-vscode
    - anthropic.claude-code
    # Append your `code --list-extensions` output, minus extensions you no longer use
```

- [ ] **Step 3: Write the script**

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[vscode-extensions] %s\n' "$*" >&2; }

if ! command -v code >/dev/null 2>&1; then
  log "vscode 'code' CLI not on PATH — has the cask installed? skipping."
  exit 0
fi

installed=$(code --list-extensions)

install_if_missing() {
  local ext="$1"
  if ! grep -qix "$ext" <<<"$installed"; then
    log "installing $ext"
    code --install-extension "$ext" --force >/dev/null
  fi
}

{{ range .extensions.common -}}
install_if_missing "{{ . }}"
{{ end -}}

log "done"
```

- [ ] **Step 4: Shellcheck + render**

```bash
shellcheck ~/Repos/dotfiles/run_onchange_15-vscode-extensions.sh.tmpl
chezmoi execute-template --init --promptString machine=home \
  < ~/Repos/dotfiles/run_onchange_15-vscode-extensions.sh.tmpl
```

Expected: each `install_if_missing "ext-id"` line rendered explicitly.

- [ ] **Step 5: Apply**

```bash
chezmoi apply --verbose
```

Expected: any missing extensions install; already-installed ones skip silently.

- [ ] **Step 6: Commit**

```bash
git -C ~/Repos/dotfiles add .chezmoidata/vscode-extensions.yaml run_onchange_15-vscode-extensions.sh.tmpl
git -C ~/Repos/dotfiles commit -m "feat: install vscode extensions from inventory"
```

---

### Task 8: `run_onchange_18-mas-apps.sh.tmpl` — Magnet via Mac App Store

**Files:**
- Create: `~/Repos/dotfiles/run_onchange_18-mas-apps.sh.tmpl`

- [ ] **Step 1: Prerequisite check — sign into the App Store app**

Open the App Store app on the machine and confirm you're signed in with your Apple ID. `mas signin` no longer works on modern macOS.

- [ ] **Step 2: Write the script**

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[mas-apps] %s\n' "$*" >&2; }

if ! command -v mas >/dev/null 2>&1; then
  log "mas not on PATH — was it in the brew bundle? skipping."
  exit 0
fi

if ! mas account >/dev/null 2>&1; then
  log "not signed into the Mac App Store — open the App Store app, sign in, then re-run."
  exit 0  # non-fatal: skip rather than fail the whole apply
fi

{{ range .mas.common -}}
log "ensuring {{ .name }} (id {{ .id }})"
mas install {{ .id }} || true   # `mas install` returns non-zero if already installed
{{ end -}}

log "done"
```

- [ ] **Step 3: Shellcheck + render + apply**

```bash
shellcheck ~/Repos/dotfiles/run_onchange_18-mas-apps.sh.tmpl
chezmoi execute-template --init --promptString machine=home \
  < ~/Repos/dotfiles/run_onchange_18-mas-apps.sh.tmpl
chezmoi apply --verbose
```

Expected: Magnet installs (or no-ops if already present).

- [ ] **Step 4: Commit**

```bash
git -C ~/Repos/dotfiles add run_onchange_18-mas-apps.sh.tmpl
git -C ~/Repos/dotfiles commit -m "feat: install Mac App Store apps (Magnet)"
```

---

## Phase C — Secrets

### Task 9: `run_once_20-1password-signin.sh.tmpl`

**Files:**
- Create: `~/Repos/dotfiles/run_once_20-1password-signin.sh.tmpl`

This script doesn't *manage* the sign-in — it nudges the operator to do it. The actual `op account add` is interactive and runs once per machine.

- [ ] **Step 1: Write the script**

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[1password-signin] %s\n' "$*" >&2; }

if ! command -v op >/dev/null 2>&1; then
  log "1Password CLI not installed — should have come from brew bundle. failing."
  exit 1
fi

if op account list 2>/dev/null | grep -q '^'; then
  log "1Password account(s) already configured."
  exit 0
fi

cat >&2 <<'MSG'

================================================================
  1Password sign-in required.
  Run this command interactively, then re-run `chezmoi apply`:

      op account add

  After that, every shell needs `eval $(op signin)` to mint
  a session token. Consider adding it to ~/.zshrc.
================================================================

MSG
exit 1
```

- [ ] **Step 2: Shellcheck + render**

```bash
shellcheck ~/Repos/dotfiles/run_once_20-1password-signin.sh.tmpl
chezmoi execute-template --init --promptString machine=home \
  < ~/Repos/dotfiles/run_once_20-1password-signin.sh.tmpl
```

- [ ] **Step 3: Apply**

```bash
chezmoi apply --verbose
```

Expected on this machine (P3 already done): `1Password account(s) already configured.` exits 0.

- [ ] **Step 4: Commit**

```bash
git -C ~/Repos/dotfiles add run_once_20-1password-signin.sh.tmpl
git -C ~/Repos/dotfiles commit -m "feat: prompt for 1Password sign-in on fresh machines"
```

---

### Task 10: `run_onchange_30-ssh-keys.sh.tmpl`

**Files:**
- Create: `~/Repos/dotfiles/run_onchange_30-ssh-keys.sh.tmpl`

**Pre-decision required:** which 1Password vault and item names hold your SSH key? The script assumes `op://Personal/SSH ed25519/private-key` and `…/public-key`. Confirm or adjust the paths before the script runs.

- [ ] **Step 1: Confirm the 1Password path**

```bash
op item get "SSH ed25519" --vault Personal --format json | jq '.fields[] | {label, type}'
```

Expected: fields named `private-key` and `public-key` (or similar — note the actual names). If the item doesn't exist, create it in 1Password using your existing `~/.ssh/id_ed25519` and `…pub` contents, then re-run.

- [ ] **Step 2: Write the script** (replace the `op://` paths with the names you just confirmed)

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[ssh-keys] %s\n' "$*" >&2; }

if ! command -v op >/dev/null 2>&1; then
  log "1Password CLI missing — failing."; exit 1
fi
if ! op account list 2>/dev/null | grep -q '^'; then
  log "not signed into 1Password — failing."; exit 1
fi

mkdir -p ~/.ssh && chmod 700 ~/.ssh

if [[ ! -f ~/.ssh/id_ed25519 ]]; then
  log "writing ~/.ssh/id_ed25519 from 1Password"
  op read "op://Personal/SSH ed25519/private-key" > ~/.ssh/id_ed25519
  chmod 600 ~/.ssh/id_ed25519
else
  log "~/.ssh/id_ed25519 already present — leaving alone"
fi

if [[ ! -f ~/.ssh/id_ed25519.pub ]]; then
  log "writing ~/.ssh/id_ed25519.pub from 1Password"
  op read "op://Personal/SSH ed25519/public-key" > ~/.ssh/id_ed25519.pub
  chmod 644 ~/.ssh/id_ed25519.pub
fi

log "done"
```

- [ ] **Step 3: Back up existing keys before apply (paranoia)**

```bash
mkdir -p ~/.ssh-backup-$(date +%Y%m%d)
cp ~/.ssh/id_ed25519* ~/.ssh-backup-$(date +%Y%m%d)/ 2>/dev/null || true
```

- [ ] **Step 4: Shellcheck + render + apply**

```bash
shellcheck ~/Repos/dotfiles/run_onchange_30-ssh-keys.sh.tmpl
chezmoi execute-template --init --promptString machine=home \
  < ~/Repos/dotfiles/run_onchange_30-ssh-keys.sh.tmpl
chezmoi apply --verbose
```

Expected on this machine: `~/.ssh/id_ed25519 already present — leaving alone` (the guard prevents overwriting your real keys).

- [ ] **Step 5: Verify SSH still works**

```bash
ssh -T git@github.com
```

Expected: `Hi <handle>! You've successfully authenticated…`

- [ ] **Step 6: Commit**

```bash
git -C ~/Repos/dotfiles add run_onchange_30-ssh-keys.sh.tmpl
git -C ~/Repos/dotfiles commit -m "feat: write SSH keys from 1Password on first run"
```

---

### Task 11: `run_once_40-gh-auth.sh.tmpl`

**Files:**
- Create: `~/Repos/dotfiles/run_once_40-gh-auth.sh.tmpl`

**Pre-decision required:** GitHub PAT location in 1Password. Assumed: `op://Personal/GitHub PAT/token`.

- [ ] **Step 1: Confirm or create the 1Password item**

```bash
op item get "GitHub PAT" --vault Personal --format json | jq '.fields[] | {label, type}' 2>/dev/null \
  || echo "Item missing — create one in 1Password with a 'token' field holding a fine-grained PAT with scopes: repo, read:org, gist."
```

- [ ] **Step 2: Write the script**

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[gh-auth] %s\n' "$*" >&2; }

if gh auth status >/dev/null 2>&1; then
  log "gh already authenticated — skipping."
  exit 0
fi

log "authenticating gh from 1Password token"
op read "op://Personal/GitHub PAT/token" | gh auth login --with-token

gh auth status
log "done"
```

- [ ] **Step 3: Shellcheck + render + apply**

```bash
shellcheck ~/Repos/dotfiles/run_once_40-gh-auth.sh.tmpl
chezmoi apply --verbose
```

Expected: `gh already authenticated — skipping.` (P3 satisfied).

- [ ] **Step 4: Commit**

```bash
git -C ~/Repos/dotfiles add run_once_40-gh-auth.sh.tmpl
git -C ~/Repos/dotfiles commit -m "feat: authenticate gh from 1Password token"
```

---

## Phase D — Runtimes & dev tools

### Task 12: mise — `dot_config/mise/config.toml.tmpl` + `run_onchange_50-mise-tools.sh.tmpl`

**Files:**
- Create: `~/Repos/dotfiles/.chezmoidata/mise.yaml`
- Create: `~/Repos/dotfiles/dot_config/mise/config.toml.tmpl`
- Create: `~/Repos/dotfiles/run_onchange_50-mise-tools.sh.tmpl`

- [ ] **Step 1: Decide the global tool list**

Most repo work uses per-project `.mise.toml`. Globally you probably only need a default Node so `npm install -g` works in Task 13.

- [ ] **Step 2: Write `mise.yaml`**

Wrapped under `mise:` so chezmoi exposes it as `.mise.*` rather than polluting the root namespace with `.global`.

```yaml
mise:
  global:
    node: "24"
    # add python/go here if you want them globally available
```

- [ ] **Step 3: Write `dot_config/mise/config.toml.tmpl`**

```toml
[tools]
{{ range $name, $version := .mise.global -}}
{{ $name }} = {{ $version | quote }}
{{ end -}}
```

- [ ] **Step 4: Write `run_onchange_50-mise-tools.sh.tmpl`**

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[mise-tools] %s\n' "$*" >&2; }

if ! command -v mise >/dev/null 2>&1; then
  log "mise not installed — check brew bundle. failing."; exit 1
fi

# Installs whatever is declared in ~/.config/mise/config.toml.
mise install
log "done: $(mise current | tr '\n' ' ')"
```

- [ ] **Step 5: Shellcheck, render, apply**

```bash
shellcheck ~/Repos/dotfiles/run_onchange_50-mise-tools.sh.tmpl
chezmoi data | jq '.mise.global'
chezmoi apply --verbose
```

Expected: mise installs Node 24 if missing; reports current versions.

- [ ] **Step 6: Verify**

```bash
mise current node
node --version
```

Expected: shows Node 24.x.

- [ ] **Step 7: Commit**

```bash
git -C ~/Repos/dotfiles add .chezmoidata/mise.yaml dot_config/mise/config.toml.tmpl run_onchange_50-mise-tools.sh.tmpl
git -C ~/Repos/dotfiles commit -m "feat: manage global mise tools"
```

---

### Task 13: `run_onchange_55-npm-globals.sh.tmpl` — Claude Code + Gemini CLI

**Files:**
- Create: `~/Repos/dotfiles/run_onchange_55-npm-globals.sh.tmpl`

- [ ] **Step 1: Verify the Gemini CLI npm package name**

```bash
npm view @google/gemini-cli name version 2>/dev/null \
  || npm view @google/generative-ai-cli name version 2>/dev/null \
  || echo "Neither found — search npm: https://www.npmjs.com/search?q=gemini%20cli"
```

If the package name is different from `@google/gemini-cli`, edit `.chezmoidata/packages.yaml` to use the correct name.

- [ ] **Step 2: Write the script**

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[npm-globals] %s\n' "$*" >&2; }

if ! command -v npm >/dev/null 2>&1; then
  log "npm not on PATH — mise should have provided node. failing."; exit 1
fi

installed=$(npm ls -g --depth=0 --json 2>/dev/null | jq -r '.dependencies // {} | keys[]')

want_install() {
  local pkg="$1"
  if grep -qx "$pkg" <<<"$installed"; then
    log "$pkg already installed"
  else
    log "installing $pkg"
    npm install -g "$pkg"
  fi
}

{{ range .packages.npm.globals -}}
want_install "{{ . }}"
{{ end -}}

log "done"
```

- [ ] **Step 3: Shellcheck + render + apply**

```bash
shellcheck ~/Repos/dotfiles/run_onchange_55-npm-globals.sh.tmpl
chezmoi execute-template --init --promptString machine=home \
  < ~/Repos/dotfiles/run_onchange_55-npm-globals.sh.tmpl
chezmoi apply --verbose
```

Expected: installs whichever CLIs are missing.

- [ ] **Step 4: Verify**

```bash
claude --version
gemini --version 2>/dev/null || true
```

Expected: Claude prints a version. Gemini may have a different binary name — note it for the README if so.

- [ ] **Step 5: Commit**

```bash
git -C ~/Repos/dotfiles add run_onchange_55-npm-globals.sh.tmpl
git -C ~/Repos/dotfiles commit -m "feat: install npm global CLIs (claude-code, gemini-cli)"
```

---

## Phase E — Cloud & integrations

### Task 14: `run_once_60-gcloud-init.sh.tmpl`

**Files:**
- Create: `~/Repos/dotfiles/run_once_60-gcloud-init.sh.tmpl`

- [ ] **Step 1: Write the script**

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[gcloud-init] %s\n' "$*" >&2; }

if ! command -v gcloud >/dev/null 2>&1; then
  log "gcloud not installed — add 'google-cloud-sdk' cask to packages.yaml. failing."; exit 1
fi

if gcloud auth list --filter=status:ACTIVE --format="value(account)" | grep -q '@'; then
  log "gcloud has an active account — skipping interactive login."
  exit 0
fi

cat >&2 <<'MSG'

================================================================
  gcloud authentication required (browser flow).
  Run the following interactively, then re-run `chezmoi apply`:

      gcloud auth login
      gcloud auth application-default login
      gcloud config set project <your-default-project>

================================================================

MSG
exit 1
```

- [ ] **Step 2: Add `google-cloud-sdk` cask to packages.yaml** if not already

Edit `.chezmoidata/packages.yaml`, append `google-cloud-sdk` under `brew.casks.common`. Re-run `chezmoi apply` so step 10 picks it up.

- [ ] **Step 3: Shellcheck + render + apply**

```bash
shellcheck ~/Repos/dotfiles/run_once_60-gcloud-init.sh.tmpl
chezmoi apply --verbose
```

Expected on this machine: `gcloud has an active account — skipping`.

- [ ] **Step 4: Commit**

```bash
git -C ~/Repos/dotfiles add run_once_60-gcloud-init.sh.tmpl .chezmoidata/packages.yaml
git -C ~/Repos/dotfiles commit -m "feat: gcloud auth prompt on fresh machines"
```

---

### Task 15: `.chezmoidata/mcps.yaml` + `run_onchange_70-claude-mcps.sh.tmpl`

**Files:**
- Create: `~/Repos/dotfiles/.chezmoidata/mcps.yaml`
- Create: `~/Repos/dotfiles/run_onchange_70-claude-mcps.sh.tmpl`

- [ ] **Step 1: Inventory current MCP servers**

```bash
claude mcp list
```

- [ ] **Step 2: Write `mcps.yaml`** using the inventory

```yaml
# Each entry runs:  claude mcp add <name> -- <command> <args...>
# For HTTP/SSE-based MCPs, use `type: http` and add `url`.
servers:
  - name: github
    type: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: op://Personal/GitHub PAT/token
  # Append entries from `claude mcp list` output
```

- [ ] **Step 3: Write the script**

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[claude-mcps] %s\n' "$*" >&2; }

if ! command -v claude >/dev/null 2>&1; then
  log "claude CLI missing — Task 13 should have installed it. failing."; exit 1
fi

current=$(claude mcp list 2>/dev/null | awk '{print $1}' | tail -n +2)

ensure() {
  local name="$1"; shift
  if grep -qx "$name" <<<"$current"; then
    log "$name already registered — removing and re-adding to refresh config"
    claude mcp remove "$name" >/dev/null 2>&1 || true
  fi
  log "adding $name"
  claude mcp add "$name" "$@"
}

{{ range .servers -}}
ensure "{{ .name }}" \
{{- if eq .type "stdio" }} -- {{ .command }}{{ range .args }} {{ . | quote }}{{ end -}}
{{- end }}
{{- if .env }}{{ range $k, $v := .env }} -e {{ $k }}=$(op read "{{ $v }}"){{ end }}{{ end }}
{{ end -}}

log "done"
```

Note: this script `op read`s tokens at execution time — keeps secrets out of git.

- [ ] **Step 4: Shellcheck + render + apply**

```bash
shellcheck ~/Repos/dotfiles/run_onchange_70-claude-mcps.sh.tmpl
chezmoi execute-template --init --promptString machine=home \
  < ~/Repos/dotfiles/run_onchange_70-claude-mcps.sh.tmpl
chezmoi apply --verbose
```

Expected: each MCP server adds / re-registers cleanly.

- [ ] **Step 5: Verify**

```bash
claude mcp list
```

Should show every server from `mcps.yaml`.

- [ ] **Step 6: Commit**

```bash
git -C ~/Repos/dotfiles add .chezmoidata/mcps.yaml run_onchange_70-claude-mcps.sh.tmpl
git -C ~/Repos/dotfiles commit -m "feat: register Claude MCP servers from inventory"
```

---

### Task 16: `.chezmoidata/repos.yaml` + `run_onchange_80-clone-repos.sh.tmpl`

**Files:**
- Create: `~/Repos/dotfiles/.chezmoidata/repos.yaml`
- Create: `~/Repos/dotfiles/run_onchange_80-clone-repos.sh.tmpl`

- [ ] **Step 1: Write `repos.yaml`**

```yaml
ovo:
  base: ~/ovo-repos
  list:
    - { name: ava-advisor-connect,  ssh: git@github.com:ovotech/ava-advisor-connect.git }
    - { name: cip-salesforce-orion, ssh: git@github.com:ovotech/cip-salesforce-orion.git }
    - { name: ovo-claude-plugins,   ssh: git@github.com:ovotech/ovo-claude-plugins.git }
    # Add more as needed
```

- [ ] **Step 2: Write the script**

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[clone-repos] %s\n' "$*" >&2; }

base="{{ .ovo.base }}"
mkdir -p "${base/#\~/$HOME}"

clone_if_missing() {
  local name="$1" ssh="$2"
  local dest="${base/#\~/$HOME}/$name"
  if [[ -d "$dest/.git" ]]; then
    log "$name already cloned"
    return
  fi
  log "cloning $name"
  git clone "$ssh" "$dest"
}

{{ range .ovo.list -}}
clone_if_missing "{{ .name }}" "{{ .ssh }}"
{{ end -}}

log "done"
```

- [ ] **Step 3: Shellcheck + render + apply**

```bash
shellcheck ~/Repos/dotfiles/run_onchange_80-clone-repos.sh.tmpl
chezmoi execute-template --init --promptString machine=home \
  < ~/Repos/dotfiles/run_onchange_80-clone-repos.sh.tmpl
chezmoi apply --verbose
```

Expected: all three repos already exist on this machine — every line prints `already cloned`.

- [ ] **Step 4: Commit**

```bash
git -C ~/Repos/dotfiles add .chezmoidata/repos.yaml run_onchange_80-clone-repos.sh.tmpl
git -C ~/Repos/dotfiles commit -m "feat: clone OVO repos on fresh machines"
```

---

## Phase F — Polish

### Task 17: `run_once_90-macos-defaults.sh.tmpl`

**Files:**
- Create: `~/Repos/dotfiles/run_once_90-macos-defaults.sh.tmpl`

Keep this conservative — set only the things you actually care about. Easy to add more later via `run_onchange_*`.

- [ ] **Step 1: Write the script**

```bash
#!/usr/bin/env bash
set -euo pipefail
trap 'echo "FAILED at line $LINENO in $(basename "$0")" >&2' ERR

log() { printf '[macos-defaults] %s\n' "$*" >&2; }

# Faster key repeat
defaults write -g InitialKeyRepeat -int 15   # default 35 (lower = shorter delay before repeat)
defaults write -g KeyRepeat -int 2            # default 6  (lower = faster repeat)

# Show hidden files in Finder
defaults write com.apple.finder AppleShowAllFiles -bool true

# Show file extensions
defaults write -g AppleShowAllExtensions -bool true

# Dock: minimize windows into app icon
defaults write com.apple.dock minimize-to-application -bool true

killall Finder Dock 2>/dev/null || true

log "done — log out / in to take full effect"
```

- [ ] **Step 2: Shellcheck + apply**

```bash
shellcheck ~/Repos/dotfiles/run_once_90-macos-defaults.sh.tmpl
chezmoi apply --verbose
```

- [ ] **Step 3: Commit**

```bash
git -C ~/Repos/dotfiles add run_once_90-macos-defaults.sh.tmpl
git -C ~/Repos/dotfiles commit -m "feat: macOS defaults (key repeat, finder, dock)"
```

---

### Task 18: README — the full runbook

**Files:**
- Modify: `~/Repos/dotfiles/README.md`

- [ ] **Step 1: Replace the stub README with the runbook**

(Outer fence below is four backticks so the inner triple-backtick code fences are literal.)

````markdown
# Dotfiles

Personal dev environment bootstrap and ongoing sync, managed by [chezmoi](https://www.chezmoi.io/).

## Fresh machine

### Manual prerequisites (one-time)

1. **Sign into the Mac App Store app.** Open it and complete the Apple ID sign-in. Required because `mas` cannot sign you in on modern macOS.
2. **Have 1Password credentials ready.** Account URL, email, secret key, and master password.

### Bootstrap

```bash
sh -c "$(curl -fsSL get.chezmoi.io)" -- init --apply <your-handle>/dotfiles
```

chezmoi will:
1. Prompt for machine type (`work` or `home`).
2. Install Homebrew (if missing) and run `brew bundle` for all formulae + casks.
3. Install VSCode extensions and Mac App Store apps (Magnet).
4. Pause and ask you to run `op account add` interactively (one-time).
5. After 1Password sign-in, re-run `chezmoi apply`. It will:
   - Pull SSH keys + GitHub PAT from 1Password
   - Install mise tools (Node 24)
   - Install npm globals (Claude Code, Gemini CLI)
   - Prompt you to run `gcloud auth login` (browser flow)
   - Register Claude MCP servers
   - Clone OVO repos into `~/ovo-repos/`
   - Apply macOS defaults

### Manual follow-ups

- Salesforce CLI org auth: `sf org login web` (browser flow, only on machines doing Salesforce work).
- Grant Accessibility permissions to Raycast / Magnet in System Settings → Privacy & Security.
- Optional: `gpg --import` your signing key if you use one.

## Day-to-day

| Action | Command |
|---|---|
| Edit a dotfile | `chezmoi edit ~/.zshrc` |
| Preview pending changes | `chezmoi diff` |
| Apply pending changes | `chezmoi apply` |
| Pull updates from the other machine | `chezmoi update` |
| Add a brew formula / cask | edit `.chezmoidata/packages.yaml`, then `chezmoi apply` |
| Add a VSCode extension | edit `.chezmoidata/vscode-extensions.yaml`, then `chezmoi apply` |
| Add an OVO repo to clone | edit `.chezmoidata/repos.yaml`, then `chezmoi apply` |
| Add a Claude MCP server | edit `.chezmoidata/mcps.yaml`, then `chezmoi apply` |

## Recovery

| Problem | Fix |
|---|---|
| A script fails mid-apply | Fix the underlying issue, then `chezmoi apply` again (resumes from failure). |
| Want to re-run a `run_once_*` script | `chezmoi state delete-bucket --bucket=scriptState && chezmoi apply` |
| Want to see what chezmoi thinks it's done | `chezmoi state dump` |
| Render dry-run (no scripts execute) | `chezmoi apply --dry-run` |
| Skip scripts this run | `chezmoi apply --exclude=scripts` |

## Repo layout

See [docs/2026-06-02-design.md](docs/2026-06-02-design.md).

## Testing risky changes

For changes that mutate system state (new install scripts, system-wide defaults), test in a scratch macOS user account first:

```bash
sudo dscl . -create /Users/scratch
sudo dscl . -create /Users/scratch UserShell /bin/zsh
sudo dscl . -create /Users/scratch RealName "Scratch"
sudo dscl . -create /Users/scratch UniqueID 510
sudo dscl . -create /Users/scratch PrimaryGroupID 20
sudo dscl . -create /Users/scratch NFSHomeDirectory /Users/scratch
sudo dscl . -passwd /Users/scratch <some-password>
sudo createhomedir -c -u scratch
# Log out of your main user, log in as `scratch`, run the bootstrap
# Then: sudo dscl . -delete /Users/scratch && sudo rm -rf /Users/scratch
```
````

- [ ] **Step 2: Commit**

```bash
git -C ~/Repos/dotfiles add README.md
git -C ~/Repos/dotfiles commit -m "docs: full runbook"
```

---

## Phase G — Ship

### Task 19: Create the public GitHub repo and push

- [ ] **Step 1: Create the repo on GitHub**

```bash
gh repo create dotfiles --public --source=~/Repos/dotfiles --remote=origin --description="Personal dev environment bootstrap" --push
```

Expected: repo at `https://github.com/<your-handle>/dotfiles`, all commits pushed.

- [ ] **Step 2: Verify the one-liner works**

Open a different account / scratch user, run:

```bash
sh -c "$(curl -fsSL get.chezmoi.io)" -- init <your-handle>/dotfiles
```

Use `init` (not `init --apply`) for the test — it clones without running. Inspect the source dir to confirm the clone matches the local repo:

```bash
diff -r ~/Repos/dotfiles ~/.local/share/chezmoi | grep -v '\.git/' | head
```

Expected: only diffs are inside `.git/` (which `git clone` reconstructs differently).

- [ ] **Step 3: Decide on the optional full scratch-user dry-run**

If you have appetite, log into the scratch user, run the full `init --apply`, and confirm it bootstraps cleanly end-to-end. This is the only true validation of the fresh-machine path. **Skipping is acceptable** — the next genuine fresh-machine event (work computer arrives) will be the real test, and by then you'll be glad the runbook is documented.

---

## Done

When all checkboxes above are ticked, the repo is the canonical source for both machines. Daily workflow becomes `chezmoi edit … → diff → apply → commit → push` (and `chezmoi update` on the other machine).

The design's "Open questions" section (in `docs/2026-06-02-design.md`) lists the four decisions deferred to implementation. Three were resolved during these tasks:
- `dot_zshrc.tmpl` modular vs monolithic — deferred; the current plan doesn't define a zshrc beyond what mise/brew populate via shell init scripts. Add when needed.
- chezmoi auto-commit/push for `chezmoi edit` — left off; explicit commits.
- Gemini CLI npm package name — verified in Task 13 step 1.

The fourth (1Password item paths) is locked in by Tasks 10 and 11.
