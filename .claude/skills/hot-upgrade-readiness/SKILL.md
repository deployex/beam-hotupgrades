---
name: hot-upgrade-readiness
description: Audit a whole Elixir/Erlang project for what would break or misbehave under BEAM hot code upgrades, before the project adopts them. Use when a team wants to start hot-upgrading an existing application, or wants to know how far an application is from it. For a single change that will ship as a hot upgrade, use hot-upgrade-review instead.
---

# Hot upgrade readiness

This audits the project as it is today, not a diff. Code written years ago is in scope: a function reference stored at boot breaks the first hot upgrade that touches its module, however small that release is.

The rules are in this repository's [README](../../../README.md). Read the section a check names before judging a finding, do not work from memory.

Work through the five parts in order. For every finding give `file:line`, what happens during or after a hot upgrade, and the fix. Count the findings of each kind: the size of the work is what the team needs to know.

A search pattern below only proposes candidates. Open each hit and decide whether the value really outlives the call that created it before reporting it.

Leave out code that only runs in tests or in development, and say that it was left out.

## 1. The build

A hot upgrade is built from two versions of the same project. Check that the build can produce one.

- **A version for every build.** A fixed version in `mix.exs` or the `.app.src` means there is no "from" and "to". See "Give every build its own version". A build identifier belongs after a `+` (`1.4.0+9cad9cd`): it cannot make the version invalid, and it is not a pre-release.
- **The pipeline builds it, not only a script.** Find every place a release is built: CI workflows, the Dockerfile, deploy scripts. Each one that ships to an environment meant to be hot-upgraded has to set the version, and has to have the previous release in the build directory when it builds the next one, which a clean checkout does not. A second build path left on a fixed version is a finding.
- **Appups are generated.** Jellyfish in `mix.exs` with `&Jellyfish.generate/1` in the release steps, or `rebar3_jellyfish_plugin` with its hook and overlay. Read the version that is locked, in `mix.lock` or `rebar.lock`, not the requirement. Flag Jellyfish before 0.2.11 or the plugin before 0.1.4: they list every module of the application as changed, so every upgrade reloads all of them. A requirement that still allows such a version is worth a note.
- **The release is a tarball** the deployment tool can unpack next to the running one (`:tar` in the release steps).
- **One toolchain.** `.tool-versions`, the Dockerfile and CI must build both versions with the same Erlang and Elixir. See "Build both versions with the same Erlang and Elixir".
- **The node can be reached.** Look at `rel/env.sh.eex` and `rel/vm.args.eex`: a node name, distribution mode, cookie or VM option that the file sets unconditionally overrides what the deployment tool passes. If the host uses TLS for distribution, the release has to turn it on too.
- **`runtime_tools` is in the release**, for tracing and inspection from outside.

## 2. Function values that outlive their call

This is the most common finding in an existing project, and the one a diff review misses. See "Do not let an anonymous function outlive the call that created it".

A value is unsafe when it is `fn ... end`, `&(...)`, `&local_fun/arity` or a partial application such as `&local_fun(&1, id)`, and it is kept somewhere that survives the call. `&Module.fun/arity` of an exported function is safe, it is resolved by name each time.

A function passed to `Enum`, `Stream` run to the end, `Repo.transaction` or a function component is called before the call returns, and is fine wherever it appears.

Search for them where they are kept:

- **LiveView hooks**: `attach_hook(` with any of the unsafe forms. The hook is stored in the socket of every open page. Read each call rather than trusting a pattern, the function is often on a later line.
  `rg -n -A3 'attach_hook\(' lib`
  Hooks attached from an `on_mount` that the router or a `live_session` applies to every page matter most: one module change breaks every open page. List those first.
- **Functions as values in maps, keyword lists and structs**: this is where most of them hide in a mature project, behind the project's own helpers. A registry of columns, tasks or actions with `lookup: fn ... end` or `runner: &run/1` fields, a debounce or callback registry, options with `on_change: &handle/2`.
  `rg -n '^\s*\w+: (fn\b|&[a-z_(])' lib`
  For each hit, follow the value to where it ends up. It is a finding when that is an assign, process state or shared storage, and not when it is consumed in the same call.
- **LiveView assigns and component props**: `assign(`, `assign_new(` and `update(` whose value is a function, or a struct or list that holds one. There are too many `assign(` calls to read, search for the ones with a function on the same or the next lines. `assign_async` and `start_async` take a function that runs once and are not this problem, see part 3 for them. `JS` commands are data and are fine.
- **Process state**: a function in the state a `GenServer`, `Agent` or `:gen_statem` returns from `init` or a callback. `Agent.start_link(fn -> ... end)` only runs once and is fine, an `Agent` whose state holds a function is not.
- **Shared storage**: `:persistent_term.put`, `:ets.insert`, `Application.put_env`, a `Registry` value.
- **Something long-running**: a callback given to a stream that is consumed later, a `Task` that loops, a retry or backoff option, a telemetry handler. `:telemetry.attach` with a local capture or anonymous function is the usual one, and it logs a warning about it at attach time.
- **Configuration files**: a function in `config/*.exs`. See "Keep functions out of your configuration".

A function that a macro generates, from a `use` of a library or of the project, may or may not be exported. Read the macro before proposing `&__MODULE__.fun/arity`: it needs a `def`, not a `defp`.

For each one, say which module version it is tied to and what triggers the failure: the first call after an upgrade that changed that module, with `BadFunctionError ... points to an old version of the code`.

The fix is `&__MODULE__.fun/arity` with the function exported, or `{module, function, args}`.

## 3. Processes that hold state

List every module that runs as a process: `use GenServer`, `use Agent`, `:gen_statem`, `:gen_server`, `Supervisor`, `GenStage`, `Broadway`, and hand-written `receive` loops.

- **State and its migration.** For each stateful process, note the shape of its state and whether the module has `code_change/3`. Without one, a release that changes the shape crashes the process on its next message. See "Give stateful processes a `code_change/3`". This is not a defect today, it is a rule for every later release, so report the list rather than a failure.
- **Hand-written loops.** A `receive` loop that calls `loop(state)` and not `__MODULE__.loop(state)` stays in old code and is killed when it is purged, right after the upgrade. So is a long-lived `Task` started with an anonymous function of a module that changes, whatever its loop does. See "Let long-running processes re-enter their loop through the module".
- **Long work.** Anything that can still be running the previous version of a module when that module is upgraded: background jobs, queue consumers, `Task`s and LiveView `start_async` or `assign_async` work, and requests that stream a response such as a chunked download. Purging old code kills a process still inside it. See "Know that purging kills by default". It only concerns an upgrade that changes the module the work runs in. Name the ones that take seconds or more and what a kill costs: a job that is retried is fine, a half-written file or a multi-step write outside a transaction may not be. Do not list every short task.
- **Structs kept across a call.** For each struct that a process, an ETS table or a cache holds, a release that adds a field to it needs a migration, and the migration must not call the module that defines the struct. See "Give stateful processes a `code_change/3`".
- **Supervision trees.** Note where children are declared. A release that adds or removes one needs a hand-written instruction. See "Treat a supervision tree change as a step of its own".

## 4. Values read once at boot

A hot upgrade restarts nothing, so anything computed when the application started keeps its value.

- **Caches filled in `start/2` or `init/1`**: a `persistent_term`, an ETS table, a file read once.
- **Lists handed to a library at boot**: the metrics of a reporter (`Telemetry.Metrics`, PromEx plugins), telemetry handlers attached in `start/2`, the queues, plugins and schedule of a job library, the topology of a consumer. A release that adds to one is not picked up. See "Register again what was registered at boot". List each one with the process that holds it.
- **Values read from the environment at boot**: a version, a feature flag, a URL. `System.get_env` in `runtime.exs` is evaluated again by a deployment tool that resolves the runtime configuration, a value copied into process state is not.
- **The version on traces and logs**: where it comes from, and whether it is the release version. A tool that updates it during the upgrade can only recognise the release version.
- **Connections that keep a credential**: a database pool or an HTTP client keeps what it connected with.
- **Phoenix**: the static manifest is read when the endpoint starts. Check that `config_change/3` calls `Endpoint.config_change/2`, which reloads it, and that the deployment tool triggers that callback. See "Phoenix and LiveView pitfalls".

- **What the browser already has.** No page reconnects, so nothing that waits for a remount runs: `Phoenix.LiveView.static_changed?/1` never reports a change, and a "new version, reload" banner built on it never shows. An open page keeps the JavaScript and CSS it loaded. If the project has such a banner, it needs a broadcast after the upgrade.
- **Options a process took when it started**: a pool size, queue and consumer settings, a clustering query. A release that changes one needs a restart of that process.

`Application.compile_env` and module attributes are compiled into the module, so they move with the code and are not findings.

For each one say whether it has to change after an upgrade, and where the application would do it: `config_change/3` is the hook OTP gives for this.

## 5. What cannot be hot-upgraded

Name these so the team knows which releases stay full deployments. See "When NOT to hot-upgrade".

- **Native code**: dependencies with a NIF or a port driver. The reliable check is the built release: `find <release>/lib -name '*.so' -o -name '*.dll'`, leaving out the ones of Erlang itself. Without a build, look for `rustler` and `elixir_make` in `mix.lock` and for `load_nif` in `deps`, and expect matches that are not NIFs. A NIF that is optional may be built on one platform and not on another, check the platform that is deployed.
- **Config Providers**: a change to their code is not applied by a hot upgrade.
- **Protocol implementations, with a tool that does not load them.** Elixir keeps the consolidated protocols outside of every application, so no appup covers them, see "Load the consolidated protocols when a release adds an implementation". Find out whether the deployment tool loads them as a step of the upgrade. If it does not, every release that adds a `defimpl` or a `@derive` is a full deployment, and count how often the project adds one: `rg -c 'defimpl |@derive' lib`. With `consolidate_protocols: false` in the release there is nothing to load, and dispatch is slower.
- **Erlang, Elixir and the release's own boot files.**
- **Database migrations**: note how they run. They have to be compatible with the code that is running while they run. See "Keep the two versions compatible at their boundaries".

## Trying it

Reading the code finds what is written down. Running an upgrade finds the rest. See "Exercise the upgrade before production". Recommend this as the next step, on a developer's machine or a staging host:

1. Run the release, not `mix phx.server`, under the deployment tool, pointed at local services.
2. Publish a second build with a small visible change and confirm it is applied without a restart: same OS process, new behaviour.
3. Read the relup that was applied. It should load the modules that changed and no others.
4. Repeat with a page open and a job in flight, changing the module each of them runs, and use the page after the upgrade.

## Output

Start with a verdict in one or two sentences: ready, ready after the listed fixes, or not suited, and why.

Then one section for each part above, with:

- the count of findings of each kind,
- each finding as `file:line - what happens - fix`, grouped when twenty lines say the same thing,
- what was checked and found in order, so a clean part reads as checked and not as skipped.

End with the fixes in the order to do them: what blocks a first hot upgrade, what makes one visible to a user, and what is a rule for later releases. Do not pad a part that has no findings.
