# skillenv

[English](README.en.md) | [简体中文](README.md)

Manage AI agent skills per repo, the way pyenv / nvm manage language versions.

`cd` in → auto-installs the skills declared in `.skillsrc`, hides global skills as needed; `cd` out → everything reverts. One declaration works across Claude Code, Codex, OpenCode, and any tool that supports the [Agent Skills standard](https://code.claude.com/docs/en/skills).

Single-file bash script; the only dependency is [direnv](https://direnv.net).

## Install (recommended: hand it to your agent)

Send this to your Claude Code / Codex:

> Please read https://raw.githubusercontent.com/TsichiChang/skillenv/main/README.md, follow the "Agent install guide" section to install skillenv on my machine, then run the verification steps and report back.

Once installed, if you also want your agent to bring the skills already on your machine under management, send:

> Follow the "Agent onboarding guide" section of the skillenv README: inventory every skill already installed on my machine, propose how to categorize them, and after I confirm, migrate them and configure each repo.

Or install manually:

```bash
git clone https://github.com/TsichiChang/skillenv.git ~/.skillenv/repo
ln -s ~/.skillenv/repo/skillenv /usr/local/bin/skillenv
brew install direnv && echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc && source ~/.zshrc
```

Afterwards, upgrade with `skillenv update` (equivalent to `git pull` inside the install repo; takes effect immediately since it's installed via symlink).

## Agent install guide

> This section is written for the AI agent performing the install. Follow the steps in order — each has a way to verify it and a fallback for when it fails.

**Step 1 — pick a bin directory**: take the first one that exists and is writable, in order: `/opt/homebrew/bin` (Apple Silicon) → `/usr/local/bin` → `~/.local/bin` (if you use this one, confirm it's on PATH; if not, append it to the shell rc file). Called `$BIN` below.

**Step 2 — install skillenv** (git clone + symlink, so `skillenv update` keeps working later):

```bash
git clone https://github.com/TsichiChang/skillenv.git ~/.skillenv/repo
ln -s ~/.skillenv/repo/skillenv "$BIN/skillenv"
skillenv help   # should print usage
```

If `~/.skillenv/repo` already exists, run `git -C ~/.skillenv/repo pull` instead and recreate the symlink. Only fall back to a single-file curl download if the environment can't `git clone` from GitHub at all (this install method requires re-downloading to upgrade).

**Step 3 — install direnv**: check first with `command -v direnv`, skip if it's already there. Prefer `brew install direnv`; if brew is unavailable, errors out, or hangs for more than 2 minutes (corporate networks can make formulae.brew.sh very slow), fall back to a direct binary install:

```bash
# macOS Apple Silicon: darwin-arm64; Intel: darwin-amd64; Linux: linux-amd64
curl -fsSL -o "$BIN/direnv" https://github.com/direnv/direnv/releases/latest/download/direnv.darwin-arm64
chmod +x "$BIN/direnv"
direnv version   # should print a version
```

**Step 4 — wire up the shell hook**: detect the user's shell (`echo $SHELL`), check whether the corresponding rc file already has a `direnv hook` line, and append one if not (zsh → add `eval "$(direnv hook zsh)"` to `~/.zshrc`; bash → add `eval "$(direnv hook bash)"` to `~/.bashrc`; fish → add `direnv hook fish | source` to `~/.config/fish/config.fish`). This step modifies the user's shell config — tell the user before doing it.

**Step 5 — verify the install**:

```bash
command -v skillenv && command -v direnv && grep -c "direnv hook" ~/.zshrc
```

All three producing output means the install succeeded. Remind the user: a new terminal (or sourcing the rc file) is needed before the hook takes effect.

**Step 6 — enable in a repo (optional, only when the user asks for it)**:

```bash
cd <target-repo>
skillenv init        # generates a .skillsrc template + .envrc
# after editing .skillsrc:
skillenv allow && direnv allow
```

If the repo already ships a `.skillsrc` (shared by teammates), skip `init` and go straight to the next command.

**Safety rule the agent must follow**: `skillenv allow` and `direnv allow` are trust gates. Before running them, always show the user the full contents of `.skillsrc` (and any `.envrc` not generated from the template) and get their confirmation — especially for manifests that came bundled with the repo from an unknown source. Never `allow` silently.

## Agent onboarding guide: inventory and manage existing skills

> This section is written for the AI agent: when the user asks to "bring my already-installed skills under skillenv management," follow this flow.

**Step 1 — inventory**: run `skillenv scan` in the user's usual repo directories to get the full list of global (claude/codex/agents/opencode) and project-level skills, along with each one's description; also read `~/.claude/plugins/installed_plugins.json` to learn about plugins and their scope. For any skill whose description is missing or truncated, read its `SKILL.md` to fill in your understanding.

**Step 2 — categorize and get user confirmation**: sort each skill into one of three buckets and present the proposal to the user as a table — **you must get confirmation before acting**:

- **General-purpose → keep global**: anything any repo might use (document handling, general workflows);
- **Domain-specific → move to a central store, enable per repo**: tied to a specific business domain or project type (judge this from domain/framework/team names appearing in the description). The default suggested central store is `~/.skillenv/store/`; use a git repo instead if the user wants team sharing;
- **Already in place → leave alone**: hand-written project-level skills already sitting in a repo's `.claude/skills/` etc.

**Step 3 — migrate (hard rules)**:

- **`mv`, never `rm`**: move domain-specific skills into the central store; move duplicates and superseded copies to `~/.skillenv/backup/` — never delete them;
- skillenv can't manage Claude plugins: to disable one per repo, edit that repo's `.claude/settings.local.json` `enabledPlugins` (merge into an existing file, never overwrite it);
- skills containing internal company information must never be committed or pushed to a public repo.

**Step 4 — lay down repo config**: generate a `.skillsrc` (enable domain skills with `skill <central-store-path>`; hide global skills to be blocked with `disable`; use `isolation strict` for full isolation) and `.envrc` for each repo the user specifies. Don't commit a `.skillsrc` that contains machine-local absolute paths — add it to `.git/info/exclude` in git repos.

**Step 5 — trust and activate**: show the user the contents of `.skillsrc` for each repo, and after confirmation run `skillenv allow` and `direnv allow`, then `skillenv activate` to materialize for the first time.

**Step 6 — verify and report**: spot-check one enabled repo (skills materialized under `.claude/skills/` etc.) and one blocked repo (`direnv export zsh` shows `CLAUDE_CONFIG_DIR`/`CODEX_HOME`, and the shadow's `skills/` contents match expectations), then report the final layout to the user: what's left global, what's in the central store, what each repo enables/blocks, and where the backups are.

## How it works

```
.skillsrc (committed to git, shared by the team)
    │  entering the directory triggers direnv → skillenv activate
    ├─► materializes skills into .claude/skills/ .codex/skills/ … (like node_modules, generated locally)
    └─► builds a shadow config dir and exports CLAUDE_CONFIG_DIR / CODEX_HOME
          ~/.skillenv/shadow/<repo>/claude/
          ├── settings.json → ~/.claude/settings.json   (login state, config — all symlinked back to the real dir)
          └── skills/       ← the only part under control: strict = empty; merge = global skills minus disabled ones
```

Isolation is **per-process** (it only affects agents launched from that shell) and never touches any global file — other terminals and other repos are completely unaffected, and leaving the directory auto-unsets the env via direnv, restoring everything.

## `.skillsrc` reference

```bash
# which agents to manage: auto = detect whichever are installed on this machine (recommended —
# newly installed agents get picked up automatically on the next cd)
# or specify explicitly: claude / codex / opencode / agents (generic .agents/skills); default: claude codex
agents auto

# merge (default): global skills + repo skills, layered together
# strict: hide all global skills, keep only what the manifest declares
isolation strict

# skill source: a github shorthand, any git URL (@ref is a branch or tag), or a local path
skill github:anthropics/skills/document-skills/pdf@main
skill git:ssh://git@dev.example.com/group/skills.git/my-skill@main   # internal/self-hosted git; .git marks the repo boundary
skill ./skills/my-local-skill          # relative path: relative to the repo root, can be committed with the repo
skill /Users/me/store/private-skill    # absolute path: only resolves on this machine, don't commit it

# hide a global skill while in merge mode (by directory name)
disable some-global-skill
```

**Installing from a marketplace**: marketplaces like skills.sh don't host packages themselves — every listing is backed by a git repo, so no dedicated source type is needed. Just translate the skill's page URL into git coordinates. For example, `https://skills.sh/vercel-labs/agent-skills/react-best-practices` maps to `skill github:vercel-labs/agent-skills/skills/react-best-practices@main` (skills usually live under a repo's `skills/` container directory — check the repo before installing to confirm the path). The marketplace is for discovery; always write exact git coordinates in the manifest — an indirect name reference can change hands over time, but exact coordinates hold up to `skillenv allow` review.

**Behavior when a source is unreachable**: when a skill can't be fetched (no access to a private repo, an absolute path from someone else's machine), `activate` **warns and skips it** — every other skill, `disable`, and isolation still apply as normal; any previously materialized copy is left in place. Once the source becomes reachable again, it's picked up automatically on the next `cd`, no command needed. Two more warnings specifically for absolute paths: if `.skillsrc` is tracked by git and contains an absolute-path source, `skillenv allow` warns that "this only resolves on this machine"; and if resolution fails on a teammate's machine, the warning names the root cause and suggests switching to a `github:` source.

## Commands

| Command | What it does |
|---|---|
| `skillenv init` | Scaffold `.skillsrc` + `.envrc` in the current repo; every run first re-fetches and reinstalls every skillenv-managed global skill; with a TTY it then walks you through reviewing this repo's skill config (declared skills, isolation mode, enabling/hiding/uninstalling global skills — see example below) |
| `skillenv allow` | Trust the current `.skillsrc` (hash recorded under `~/.skillenv/trust/`) |
| `skillenv activate` | Sync skills and print env exports (called from `.envrc`, can also be run manually) |
| `skillenv status` | Show trust state, active mode, and current environment |
| `skillenv scan` | Inventory every skill installed on this machine (global + current repo's project-level, with managed flag and description) |
| `skillenv list` | Global skill inventory, **tree view grouped by agent**: source, install date, description; `●` = managed by skillenv / `○` = installed manually; colored under a TTY, auto-plain when piped |
| `skillenv install <source> [name...]` | **Global** install: copies a skill into every detected agent's global directory (same source format as the `skill` directive). If the source has no `SKILL.md` of its own — i.e. it's a repo/container directory — it recursively finds every skill inside: with no names given, it lists them (interactively prompts which to install if there's a TTY; otherwise just prints the list and the next command to run); with names (or `all`) given, it installs non-interactively |
| `skillenv uninstall <name>` | **Global** uninstall: removes from every global dir (moved into `~/.skillenv/backup/`, never actually deleted) |
| `skillenv prompt` | Compact status for shell prompts (see below); prints nothing without a `.skillsrc` |
| `skillenv update` | Upgrade skillenv itself (`git pull` for a git install, re-download for a curl install) |

`skillenv list` output looks like:

```
◆ codex  ~/.codex/skills · 2 skills
  ├─ ● react-best-practices     github:vercel-labs/agent-skills/skills/react-b… · 2026-07-11
  │      React and Next.js performance optimization guidelines from Vercel…
  └─ ○ pdf                      manual · 2026-03-06
         Use when tasks involve reading, creating, or reviewing PDF files…

◆ opencode  ~/.config/opencode/skills · 0 skills
  └─ (no skills)
```

**init's every run: refreshing managed global skills first**: the first thing `skillenv init` does is walk every global skill directory on this machine and, for each one carrying a `.managed-by-skillenv` marker, re-fetch it from its recorded source (`github:`/`git:` sources force-pull the latest commit, overwriting the local cache) and reinstall it; skills placed manually (no marker) are left untouched. This runs whether `.skillsrc` is new or already exists, and whether or not there's a TTY; a single source failing to refresh (offline, no access to a private repo, …) just warns and is skipped, without affecting anything else. It reuses the same fetch/install logic as `skillenv install`, just with a forced cache refresh — right now only `init` does this; rerunning `install` on its own does not.

**init's every run: interactive config review**: once the refresh finishes and the scaffold is in place, as long as there's a TTY, `skillenv init` walks you through this repo's skill config — not just the first time, every run (including repos that already ship a `.skillsrc`, so you can adjust anytime; an agent calling it without a TTY skips the whole thing rather than blocking). The review has four sections; each lists the relevant skills, explains what each action does, and lets you pick by name (`all` and enter-to-skip supported):

- **skills this repo declares** (`skill` directives in `.skillsrc`) — `remove` a declaration;
- **isolation mode** — shows whether you're on merge or strict, and offers to switch to strict (hide all global skills);
- **global skills this repo currently hides** (`disable` directives) — `enable` to inherit again, or `uninstall` from the whole machine;
- **other global skills still inherited** — `disable` to hide just in this repo, or `uninstall` from the whole machine.

`remove`/`enable`/`disable` only edit this repo's `.skillsrc`; `uninstall` affects every repo on the machine (moved to `~/.skillenv/backup/`, never actually deleted). Every change still needs `skillenv allow && direnv allow` afterward to take effect. In the list, `●` = currently active/inherited, `⊘` = hidden in this repo; adding a `disable` shows as `+`, removing a declaration or hide shows as `-`:

```
$ skillenv init
skillenv: created .skillsrc
skillenv: wrote .envrc

◆ isolation
  merge (current) — inherit global skills, add repo-declared ones on top
  › block all global skills for this repo? (y/N) n

◆ other global skills on this machine
  disable = hide in this repo · uninstall = remove from every repo
  ● bi-query-sql            Use when: (1) SQL query helper for…
  ● eden-dev                Eden dev environment wizard…
  › uninstall which? (names · all · ↵ skip)
  › disable which? (names · all · ↵ skip) eden-dev
  + disable eden-dev

skillenv: next: edit .skillsrc, then run: skillenv allow && direnv allow
```

**Division of labor between the global and repo layers**: the repo layer is declarative (`.skillsrc` is exactly what it says); the global layer is imperative (`install`/`uninstall` edit the global directories directly). How the two interact depends on the mode: when a new skill is installed globally, a merge-mode repo picks it up automatically on the next `cd` (the shadow is recomputed every time), while a strict-mode repo is unaffected; `disable` can still hide a globally-installed skill in an individual repo. `install` never overwrites a same-named skill that was placed manually (no managed marker → skipped); `uninstall` acts on any same-named global skill, but always moves it to backup rather than deleting it. To update a skill, just rerun `install` (the managed marker allows it to overwrite).

**Pointing at a whole repo instead of a specific skill path**: if the source given to `install` has no `SKILL.md` of its own, it's treated as a repo/container directory, and every skill inside it is found recursively:

```
$ skillenv install github:vercel-labs/agent-skills
skillenv: no SKILL.md at github:vercel-labs/agent-skills itself — found 2 skill(s) inside:
  skills/react-best-practices    React and Next.js performance optimization guidelines…
  skills/vercel-deploy           Deploy and manage projects on Vercel…
skillenv: install which? (names, 'all', or blank to cancel): react-best-practices
```

Pressing enter with no name cancels. You can also skip the prompt entirely by passing `all` or specific names up front, e.g. `skillenv install github:vercel-labs/agent-skills react-best-practices`. In non-TTY environments (scripts, agent calls), leaving out names just prints the list and the available next command — it never blocks waiting for input.

## Quiet mode and prompt integration

By default, direnv prints `loading` / `export +VAR` lines on every `cd`. To go fully quiet and move the status into your prompt instead:

**1. Silence direnv** — write to `~/.config/direnv/direnv.toml`:

```toml
[global]
hide_env_diff = true
log_filter = "^$"
```

skillenv's "`.skillsrc` not trusted" safety warning goes to `.envrc`'s stderr, which isn't affected by this filter — it still shows up normally.

**2. Show status in the prompt** — `skillenv prompt` prints something like `merge -4` (merge mode, 4 global skills hidden), `strict +2` (strict mode, 2 skills declared), or `!untrusted` (manifest pending confirmation). starship users: add `${custom.skillenv}` to `format` in `~/.config/starship.toml`, and append:

```toml
[custom.skillenv]
command = "skillenv prompt"
when = "test -f .skillsrc"
symbol = "⛭"
style = "bold cyan"
format = "[$symbol $output]($style) "
```

powerlevel10k users: add a custom segment in `~/.p10k.zsh`:

```zsh
function prompt_skillenv() {
  local out
  out=$(skillenv prompt 2>/dev/null) || return
  [[ -n $out ]] || return
  p10k segment -f cyan -i '⛭' -t "$out"
}
# then add skillenv to POWERLEVEL9K_LEFT_PROMPT_ELEMENTS or RIGHT_PROMPT_ELEMENTS
```

Same idea for any other framework: any prompt segment that can run a command (zsh `precmd` building `RPROMPT`, an oh-my-posh custom segment) just calls `skillenv prompt` — a single run takes about 70ms.

Status outputs at a glance: empty (no `.skillsrc`), `merge -4` (merge mode, 4 hidden), `strict +2` (strict mode, 2 declared), `!untrusted` (manifest changed, pending `skillenv allow`). Note that the prompt reflects skillenv's trust-gate state; the environment doesn't load until `.envrc` gets `direnv allow`, but the prompt still shows based on `.skillsrc`.

## Security model

Auto-installing and auto-modifying an agent's config root is a standard supply-chain attack surface (Codex once had an RCE via `.env` redirecting `CODEX_HOME`). skillenv's countermeasures:

1. **Dual trust gates**: `.envrc` is gated by `direnv allow`; the content hash of `.skillsrc` is gated by `skillenv allow` — any change to the manifest refuses to run until you review it and `allow` again.
2. Shadow directories are only ever created under `~/.skillenv/`, never at a path inside the repo.
3. Materialized skill directories carry a `.managed-by-skillenv` marker; skillenv only deletes/overwrites directories it manages itself, and never touches manually placed skills.

## Known limitations

- **OpenCode's global skills can't be hidden**: it hardcodes reading from `~/.claude/skills` and `~/.agents/skills`, so redirecting its config directory doesn't block them; repo-level skills are unaffected.
- **Claude Code's plugin skills aren't hidden by strict mode** (the shadow keeps the plugins config in place so it doesn't break login state and similar features); to disable a plugin per repo, use `enabledPlugins` in `.claude/settings.json`.
- Claude Code's **VS Code extension doesn't read `CLAUDE_CONFIG_DIR`** — isolation only applies to the CLI launched from a terminal.
- `@ref` only supports branches/tags (`git clone --branch`), not commit SHAs; skill paths can't contain spaces.
- An agent session that's already running is unaffected by entering/leaving a directory (env vars are read at process startup — this is the expected semantics).
