# AGENTS.md

Repository: `@jeffreyjyz/opencode-shell-rc` — a small opencode v2 plugin (server
half only) that makes the agent's non-interactive `zsh` load the user's aliases
and functions via a `ZDOTDIR` shim. Sits beside `cmduse`, `oc-cmd-compare`
(`mpc`), `reqshape` and `opencode-context` in `~/dev/cmdcode-tools/`; shares no
code with them (the shim idea is independent).

## Layout

```
src/index.ts   server half (v2 `{ id, setup }`): the `shell.create.before` hook
src/shim.ts    the shim: stateDir, shimContent, needsRefresh, prepare, refresh
src/ids.ts     plugin id /= state dir name
index.js       root shim for local-directory plugin loading (see Traps)
```

No TUI half, no runtime dependencies (node builtins only).

## Core rules

- **`ZDOTDIR` is the whole mechanism.** opencode's shell tool runs
  `zsh -c <command>` non-interactively, which never reads `~/.zshrc`; but zsh
  *always* reads `$ZDOTDIR/.zshenv` first. Point `ZDOTDIR` at a shim whose
  `.zshenv` sources the generated state and aliases/functions become available
  before the command string is parsed.
- **Curate with zsh, not a regex.** `state.zsh` comes from
  `zsh -ic 'alias -L; typeset -f'` — zsh's own serializer, so all aliases
  (including oh-my-zsh's) and functions are captured exactly. Never hand-parse
  `.zshrc` lines.
- **Refresh on rc change, not per command.** `needsRefresh` compares the state
  file's mtime against `~/.zshrc` / `~/.zshenv` / oh-my-zsh's mtime; the 90ms
  `zsh -ic` generation runs only when that changes. Per command is a stat + a
  small source, ~0ms.
- **`zcompile` the state after every regeneration.** `state.zsh` is ~344KB
  (oh-my-zsh contributes ~1500 functions); sourcing it raw costs ~10ms per
  command, but `zcompile` writes `state.zsh.zwc` which zsh prefers when newer and
  loads in ~0.5ms. Always `rm` the old `.zwc` first (it is read-only) and treat
  compilation as best effort — a missing `.zwc` just falls back to the text.
- **Fail open.** If zsh is missing or generation throws, `ZDOTDIR` is not set and
  the shell behaves exactly as before. Never fail the shell command.
- **PATH/env are deliberately not snapshotted.** The agent inherits opencode's
  start-up environment. Document that a `.zshrc` `PATH` change needs an opencode
  restart; alias/function changes self-refresh.

## Traps

- **`alias` expansion needs the shim sourced before the `-c` string is parsed.**
  `ZDOTDIR/.zshenv` is read at startup, before `-c` is parsed — this is why the
  shim works where appending `source ~/.zshrc` to the command string does not
  (same-line and next-line both tested, both fail; the `-c` string is parsed as a
  unit). Do not "improve" this into a command-string hack.
- **`.zshenv` is always sourced for `zsh -c`; `.zshrc` never is.** That asymmetry
  is the entire exploit.
- **Do not source the full rc.** `source ~/.zshrc` costs ~90ms and drags in
  oh-my-zsh/compinit/autosuggestions, which are interactive-only and useless in a
  non-interactive agent shell; `zsh -ic` also emits `(eval):1: can't change
  option: zle` unless stderr is discarded (`2>/dev/null` / `stdio: ignore`).
- **`ZDOTDIR` redirects the lookup for *all* rc files.** A nested `zsh -i` (or
  login shell) under the agent would look for `.zshrc`/`.zprofile`/`.zlogin` in
  the shim dir, so those are symlinked back to the real ones. Do not remove the
  symlinks.
- **Hook mutability.** `shell.create.before` receives the live `invocation`
  object; mutating `event.env` is honoured because the spawn reads
  `invocation.env` afterwards (`packages/core/src/shell.ts`). The hook callback
  may be async but the mutation is synchronous here.
- **The PTY/terminal path is a different service** (`experimental.persistentPty`)
  and is *not* covered by this hook — only `Shell.create` callers (the shell tool,
  the session shell, the server shell handler) are.
- **`biome check .` aborts on a nested root configuration when a `.delta/`
  directory is present**; this repo's `biome.json` excludes `**/.delta`.
- **Local-directory plugins resolve `index`/`server`/`tui` beside the package
  root, not the `exports` map.** So the repo ships a root `index.js` shim that
  re-exports `dist/`; npm installs use `exports`. Keep the shim in sync.

## Tests

Hermetic: every test points `HOME` and `XDG_STATE_HOME` at a temp dir and writes
a fake `.zshrc`, so the real user state is never touched. The alias-resolution
test (and the zsh half of the hook test) is skipped when `zsh` is absent.

```sh
bun install
bun test
bun run typecheck
bun run build
```

## Publishing (NEVER without explicit go)

Same policy as the sibling repos: local commits only until the user says push.
Publishing from this account must go through `npm stage publish` + the user's
`npm stage approve` — a bare `npm publish` leaves ghost versions (see
`cmduse/AGENTS.md`). Version this package independently. After a publish,
opencode's per-package install cache can lag npm: `npm cache clean`, remove
`~/.cache/opencode/npm/@jeffreyjyz/opencode-shell-rc@latest`, then the user
restarts. **Never restart or reload opencode yourself.**

See `opencode-context/AGENTS.md` for the `E401`/dead-`.npmrc`-token trap on
`npm stage publish` (same account).
