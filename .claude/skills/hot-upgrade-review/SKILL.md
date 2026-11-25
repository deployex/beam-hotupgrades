---
name: hot-upgrade-review
description: Review an Elixir/Erlang diff or PR for changes that would break a BEAM hot code upgrade. Use when a change is meant to ship as a hot upgrade, or before merging a release that will be hot-upgraded into production.
---

# Hot upgrade review

This reviews one change. To audit a whole project before it adopts hot upgrades, use `hot-upgrade-readiness`.

Check the diff against the rules in this repository's [README](../../../README.md), not from memory, read the relevant section if a check below is unclear.

Go through the diff and flag anything in these categories. Report concrete file/line findings, not a restatement of the rules.

## Blocking (this diff cannot be a hot upgrade)

- `.tool-versions`, `mix.exs` Erlang/OTP or Elixir version pins changed.
- A Config Provider module's code changed (not just the values it produces).
- A dependency that loads a NIF or port driver was upgraded, and the library does not document hot-upgrade support for that version range.

## Needs an explicit appup instruction, not just the generated one

- A `Supervisor`'s `init/1` gained or lost a child spec. Jellyfish cannot see this, see "Adding or removing a process in the supervision tree" in the README. Confirm the appup has `apply` instructions for the added/removed child, in the right order, or that this release is going out as a full deployment instead.
- A `GenServer`'s state shape changed (new/removed/renamed fields, different variant). Confirm `code_change/3` was added or updated to migrate it. No `code_change/3` plus a changed state shape is a crash waiting for the next message.

## Dependencies that moved

- Take the list of dependencies that moved from the `mix.lock` diff, not from `mix.exs`. A dependency that is not declared in `mix.exs` and moved because another one brought it in is hot-upgraded too, and needs the same review as a declared one, see "Assess every dependency you upgrade, one at a time" in the README.
- For each one, read what changed between the two versions and flag changes to the state of a running process or to a supervision tree.
- Include the processes a library starts for each connection or request. A web server that changes the struct of its HTTP/2 connections, as Bandit 1.15 did, closes every HTTP/2 connection that is open at the upgrade, on its next request.
- If the project uses a Jellyfish version before 0.2.10, flag every moved dependency that `mix.exs` does not declare: it gets no appup, and the relup fails to build.

## Anonymous function / closure lifetime

For each new or changed closure (`fn ... end`, `&(...)`, `&private_fun/1`), ask: can it still be called after the current call returns? If yes, it needs to be stored as `{module, function, args}` instead, not as the function value itself. Look specifically at:

- Values stored in `GenServer` state, `:persistent_term`, or ETS.
- Callbacks handed to something long-running (a stream, a `Task` that loops, a download).
- `assign/3` calls in LiveView and function props passed to components or slots.
- `attach_hook/4` in LiveView with `&local_fun/arity` or an anonymous function. The hook is stored in the socket of every open page.
- Any value written to application config (`config :app, key: fn -> ... end`), which also breaks `sys.config` parsing, not just hot upgrades.

`&Module.fun/arity` captures of exported functions are fine, they resolve through the module. Private-function captures and literal closures are not.

A closure does not have to be in the diff to break. When the diff changes a module, look at the function values that module already hands out: one stored by an open LiveView or a running process was created by the version being replaced, and fails on its first call after the upgrade.

## Long-running processes that bypass OTP behaviours

A hand-written `receive` loop (not a `GenServer`/`GenStatem`/etc.) that calls `loop(state)` instead of `__MODULE__.loop(state)` never picks up new code and is killed when the old code is purged, right after the release is made permanent. Flag any hand-rolled loop and confirm it re-enters through the module.

The same goes for a task. Flag a long-lived `Task` started with an anonymous function, `Task.start(fn -> ... end)`, `Task.async`, `start_async` in a LiveView, in a module the diff changes: the function belongs to the previous version and the task is killed. One that holds a database transaction open has it rolled back. Started with a module, a function and arguments, and looping through the module, it lives through the upgrade when it does not block for long.

## Protocol implementations

- The diff adds a `defimpl`, a `@derive`, or a dependency that brings one. The consolidated protocols are outside of every application, no appup covers them, and the node keeps the protocols it booted with: the first value of the new struct raises `Protocol.UndefinedError`. See "Load the consolidated protocols when a release adds an implementation" in the README. Confirm that the deployment tool loads the consolidated protocols of the new release as a step of the upgrade, or that this release goes out as a full deployment.

## State, requests and what is registered at boot

- A struct that is kept across a call, in process state, ETS or a cache, gained or lost a field. Existing values do not have it. Confirm a `code_change/3` that builds the new state with a struct literal, and not with a call into the module that defines the struct, which may still be the old one when the migration runs.
- The diff changes a module that a long request runs through: a download, an upload, an export, a streamed response. A request in flight is killed, and nothing tries it again. See "A request in flight is a process inside your code".
- The diff adds to something the application registers once at boot: a metric, a telemetry handler, a queue, a scheduled job, or changes configuration that a child of the supervision tree was started with, such as the queues of Oban. It is not picked up until the process that holds the list restarts. See "Register again what was registered at boot".
- A migration changes the type of a column in place. The connections are kept across a hot upgrade, and a query prepared on them fails once on each. Ask for the change in two releases.

## Phoenix / LiveView specific

- Any change under `assets/` or `priv/static/`: the static manifest is read when the endpoint starts and a hot upgrade does not read it again. Confirm that `config_change/3` calls `Endpoint.config_change/2`, which reloads it, and that the release asks the deployment tool to trigger that callback (`hot_upgrade_config_change` in DeployEx). Without both, this release is a full deployment.
- The previous version's digested files have to ship in the new release, for the pages that are already open. `phx.digest` keeps them when both versions are built in the same directory.
- A LiveView's assigns shape changed in a way existing connections rely on, and there is no tolerant read path for the old shape. There is no `code_change/3` for per-connection LiveView state, see the README's LiveView section, so this is either expand-then-contract across two releases or an accepted remount.

## Cross-version compatibility at the boundary

During a rolling hot upgrade, old and new code run side by side. For anything persisted or sent across that boundary (ETS entries, PubSub/Presence payloads, messages already in a mailbox, rows written by the old code and read by the new one), confirm the change is additive this release (expand) rather than removing/renaming in the same release (contract). A contract should be a later release, after every node has the expand version.

## Output

List findings as: `file:line - what breaks - which rule above`. If nothing in the diff triggers these checks, say so plainly rather than padding the report.
