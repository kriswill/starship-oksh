# starship-oksh

Initialize the [Starship](https://starship.rs/) prompt under
[oksh](https://github.com/ibara/oksh) — a portable fork of OpenBSD's
pdksh-based `ksh(1)`.

Starship ships init scripts for bash, zsh, fish, elvish, ion, powershell,
tcsh, nu, xonsh, and cmd, but **not** for ksh / oksh. Simply sourcing
`starship init bash` under oksh does not work: the bash init leans on
`PROMPT_COMMAND`, `PS0`, the `DEBUG` trap, `PIPESTATUS`, `shopt`,
`BASH_VERSION`, and bash-specific readline markers — none of which exist
in pdksh. This repo contains a small hand-ported init that works.

## Contents

| File                      | Purpose                                                                 |
| ------------------------- | ----------------------------------------------------------------------- |
| `starship-init-oksh.sh`   | The oksh-compatible init script. Source this from `~/.kshrc`.           |

Git history preserves the upstream `starship init bash` output as the
first commit, then renames and rewrites it as `starship-init-oksh.sh`,
so the port is reviewable as a diff:

```
git log --follow --patch starship-init-oksh.sh
```

## Install

```sh
git clone https://github.com/<you>/starship-oksh ~/src/starship-oksh
echo '. $HOME/src/starship-oksh/starship-init-oksh.sh' >> ~/.kshrc
```

Adjust the absolute path to the `starship` binary inside the script if
yours is not at `/etc/profiles/per-user/k/bin/starship` (the Nix profile
default on this machine). `command -v starship` will tell you.

## What was changed, and why

The port is deliberately minimal. Full commentary is inlined at the top
of `starship-init-oksh.sh`; the summary:

1. **`PROMPT_COMMAND` + `PS0` + `DEBUG` trap → `PS1` command substitution.**
   pdksh has none of those hooks. Instead, ksh re-expands `PS1` on every
   prompt draw, so we set `PS1='$(_starship_prompt)'` and let the shell
   run starship fresh each time.
2. **Command-duration timing dropped.** Without a pre-exec hook there is
   no reliable way to capture the start time of the user's command, so
   the `$cmd_duration` module will not render. `SECONDS`-based fakes are
   unreliable across subshells and pipelines.
3. **ble.sh, bash-preexec, and BASH_VERSION branches removed.** Those
   integrations are bash-only.
4. **`PIPESTATUS` removed.** Not available in pdksh/oksh.
5. **`local` → `typeset`**, and POSIX `f()` → ksh `function f { ... }`.
   In pdksh, POSIX-style function definitions give all variables global
   scope even with `typeset`. The `function` keyword is what enables
   function-local scope.
6. **`shopt -s checkwinsize` removed.** Bash builtin; oksh tracks
   `$COLUMNS` automatically via `SIGWINCH`.
7. **Non-printing prompt markers translated.** Starship with
   `STARSHIP_SHELL=bash` emits the literal two-character sequences
   `\[` and `\]` to mark zero-width regions (bash/readline convention).
   pdksh's line editor recognises the single bytes `0x01` (SOH) and
   `0x02` (STX) instead. An `awk` filter in `_starship_fix_markers`
   does the translation so oksh's wrap math stays correct.

## References

- Starship — <https://starship.rs/>
- Starship advanced configuration — <https://starship.rs/advanced-config/>
- Upstream bash init template — <https://github.com/starship/starship/blob/master/src/init/starship.bash>
- oksh (portable OpenBSD ksh) — <https://github.com/ibara/oksh>
- OpenBSD `ksh(1)` manual — <https://man.openbsd.org/ksh.1>
- Bash prompt markers (`\[` / `\]`) — <https://www.gnu.org/software/bash/manual/html_node/Controlling-the-Prompt.html>
- GNU readline `rl_expand_prompt` (for the `\001` / `\002` convention) — <https://tiswww.case.edu/php/chet/readline/readline.html#SEC36>

## Known gaps

- No `$cmd_duration` module (see #2 above).
- No right-prompt (`bleopt prompt_rps1`) — that path in the bash init
  is specific to ble.sh.
- The absolute path to `starship` is hard-coded; adjust for your system
  or replace with `$(command -v starship)` at source time if you prefer.
