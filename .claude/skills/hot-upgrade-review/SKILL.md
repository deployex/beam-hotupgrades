---
name: hot-upgrade-review
description: Review an Elixir/Erlang diff or PR for changes that would break a BEAM hot code upgrade. Use when a change is meant to ship as a hot upgrade, or before merging a release that will be hot-upgraded into production.
---

# Hot upgrade review

Check the diff against the rules in this repository's [README](../../../README.md), not from memory, read the relevant section if a check below is unclear.

Go through the diff and flag anything in these categories. Report concrete file/line findings, not a restatement of the rules.

## Blocking (this diff cannot be a hot upgrade)

- `.tool-versions`, `mix.exs` Erlang/OTP or Elixir version pins changed.
- A Config Provider module's code changed (not just the values it produces).
- A dependency that loads a NIF or port driver was upgraded, and the library does not document hot-upgrade support for that version range.

## Needs an explicit appup instruction, not just the generated one

- A `Supervisor`'s `init/1` gained or lost a child spec. Jellyfish cannot see this, see "Adding or removing a process in the supervision tree" in the README. Confirm the appup has `apply` instructions for the added/removed child, in the right order, or that this release is going out as a full deployment instead.
- A `GenServer`'s state shape changed (new/removed/renamed fields, different variant). Confirm `code_change/3` was added or updated to migrate it. No `code_change/3` plus a changed state shape is a crash waiting for the next message.

## Anonymous function / closure lifetime

For each new or changed closure (`fn ... end`, `&(...)`, `&private_fun/1`), ask: can it still be called after the current call returns? If yes, it needs to be stored as `{module, function, args}` instead, not as the function value itself. Look specifically at:

- Values stored in `GenServer` state, `:persistent_term`, or ETS.
- Callbacks handed to something long-running (a stream, a `Task` that loops, a download).
- `assign/3` calls in LiveView and function props passed to components or slots.
- Any value written to application config (`config :app, key: fn -> ... end`), which also breaks `sys.config` parsing, not just hot upgrades.

`&Module.fun/arity` captures of exported functions are fine, they resolve through the module. Private-function captures and literal closures are not.

## Long-running processes that bypass OTP behaviours

A hand-written `receive` loop (not a `GenServer`/`GenStatem`/etc.) that calls `loop(state)` instead of `__MODULE__.loop(state)` never picks up new code and gets purged (killing the process) at the next load of a third version. Flag any hand-rolled loop and confirm it re-enters through the module.

## Phoenix / LiveView specific

- Any change under `assets/` or `priv/static/` in a release meant to hot-upgrade: the static manifest is cached at endpoint boot and will not pick up new digests. This needs a full deployment, not a hot upgrade, unless the team has a separate mechanism to refresh it.
- A LiveView's assigns shape changed in a way existing connections rely on, and there is no tolerant read path for the old shape. There is no `code_change/3` for per-connection LiveView state, see the README's LiveView section, so this is either expand-then-contract across two releases or an accepted remount.

## Cross-version compatibility at the boundary

During a rolling hot upgrade, old and new code run side by side. For anything persisted or sent across that boundary (ETS entries, PubSub/Presence payloads, messages already in a mailbox, rows written by the old code and read by the new one), confirm the change is additive this release (expand) rather than removing/renaming in the same release (contract). A contract should be a later release, after every node has the expand version.

## Output

List findings as: `file:line - what breaks - which rule above`. If nothing in the diff triggers these checks, say so plainly rather than padding the report.
