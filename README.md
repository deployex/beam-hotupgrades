# BEAM Hot Upgrades

A hot upgrade replaces the code of a running system without restarting it.
Processes keep the state they were holding, connections stay open, and nothing is drained or rerouted.
What it cannot replace is the VM and the language runtime underneath that code, and that single limit decides most of what follows.

This repository is a deployment-tool-agnostic reference for hot code upgrades on the BEAM (Erlang, Elixir, Gleam, and anything else running on the same VM).
It documents the mechanism itself (`appup`, `relup`, `release_handler`), the practices that make a codebase upgradable, and the pitfalls that show up in real projects.
If you use [DeployEx](https://github.com/deployex/deployex) or another deployment tool, its own docs cover how that tool drives the mechanism described here.

[Jellyfish](https://github.com/deployex/jellyfish) generates `.appup` files automatically for Elixir applications from a diff between two builds.
You can also write or extend appups by hand when an upgrade needs actions that cannot be inferred from the diff, see [Adding or removing a process in the supervision tree](#adding-or-removing-a-process-in-the-supervision-tree).

[relup-guard-elixir](https://github.com/deployex/relup-guard-elixir) and [relup-guard-erlang](https://github.com/deployex/relup-guard-erlang) are CI actions that boot a real release and apply a real upgrade against it on every pull request, for Mix and rebar3/relx releases respectively.
They automate the "exercise the upgrade before production" practice from [Good practices for a project that can be hot-upgraded](#good-practices-for-a-project-that-can-be-hot-upgraded), they do not replace it, a failure they catch is still worth reproducing by hand on a real node.

Most releases do not need a full deployment.
You do not change OTP or your libraries often, and a hot upgrade combines with migrations to ship a change with no downtime at all.
High availability is not free though, it is something you design for, and this guide covers the part of that design that shows up at deployment time.

A release is easiest to judge backwards: look first for what cannot be hot-upgraded, then decide about everything else.

## What can be hot-upgraded

* **Your applications** - the Elixir/Erlang/Gleam code you deploy.
* **Dependencies** - the libraries those applications use, once you have checked that the versions you are moving between can be replaced in a running node.
* **The deployment tool itself**, if it runs on the BEAM and is built with hot-upgrade support in mind, the same mechanism applies to it as to any other application.

Erlang/OTP and Elixir are never in scope.
They are the runtime the upgrade runs on rather than libraries loaded into it, so a release that changes either is a full deployment.
This is stated once here and is the reason behind several of the rules below.

Dependencies are versioned independently from your project, so treat an upgrade to a dependency as its own decision rather than a side effect of your own release.

## When NOT to hot-upgrade

**DO NOT HOT-UPGRADE if:**
* The new release is updating the Erlang/OTP and/or Elixir version.
* The new release modified, deleted, or added config provider modules.

Erlang and Elixir are both replaced from underneath the running code, and `systools` asks for an appup that upgrades them, which nobody ships.
Picking a release artifact built for another OTP line is the easiest way to hit this by accident, see [Choosing the right release artifact](#choosing-the-right-release-artifact).
The second is about configuration, which behaves differently from the rest of the release and is worth understanding rather than memorising.

### What happens to configuration during a hot upgrade

`runtime.exs` and Config Providers only run when an application boots, not on every request, so a hot upgrade has to decide what to do with them, since nothing restarts the node to trigger that boot-time evaluation.

`release_handler:install_release/2` first rebuilds the environment of **every** loaded application from the new `.app` files and the new build-time `sys.config`, and only then runs the relup instructions.
A deployment tool that wants the new `runtime.exs` to take effect during the hot upgrade itself (rather than at the next restart) has to resolve it and apply the result explicitly, typically with a relup hook that runs before any process is suspended or has its code changed.
DeployEx does exactly this, see its own hot-upgrade docs for the details of that mechanism.

What that means in practice, regardless of which tool drives the upgrade:

* **Changed or removed keys are applied** once the new environment is rebuilt. The environment is deleted and rebuilt, not merged, so a key you delete in the new version is really gone after the upgrade.
* **A changed `runtime.exs` only takes effect during the upgrade itself if something evaluates it as part of the relup.** Otherwise it is applied the same way the rest of the environment is, at the next restart. If your tooling does evaluate it during the upgrade, it runs on the node as it is now, so it must not rely on modules, functions, or language features that only arrive with the new release. A `runtime.exs` that reads environment variables, files, or secrets is fine. One that calls into a module the new version introduces is not, and that is a case to deploy fully.
* **A changed Config Provider is NOT applied by the upgrade.** Provider modules run on the node in its current version, before the new code is installed, so the old implementation is what resolves the configuration. A fix to a provider only takes effect once a full deployment has made it the running version.

The last point is the reason for the config-provider entry in the list above, and it applies to the provider's own code, not to the values it produces.

## How a hot upgrade works

You do not need to write any of these files by hand, but knowing what they are is what makes the failures readable.

**The pieces**

* An **appup** describes how to move one application from one version to another: which modules to load, which processes to suspend, and where to call `code_change/3`. [Jellyfish][jyf] generates one per Elixir application, as a real `.appup` plus a `jellyfish.json` sidecar that records its `from` and `to` versions. For Erlang applications, where Jellyfish does not apply, the `.appup` is written by hand or with another tool, with no sidecar.
* A **relup** is the plan for the whole release, built by `systools:make_relup/4` from the two releases and their appups. It is ordered, and it fails to build when an application changes version without an appup to describe how.
* `release_handler` is the OTP module that runs that plan on the live node.

**The standard steps, run over RPC or locally on the node**

1. `release_handler:unpack_release/1` registers the new release and puts its code on disk. Nothing is loaded yet.
2. `systools:make_relup/4` builds the plan.
3. `release_handler:check_install_release/1` evaluates the plan without applying it.
4. `release_handler:install_release/2` rebuilds the application environments and then runs the plan. From this point the node is running the new code.
5. `release_handler:make_permanent/1` makes the new version the one the node boots into next time.

Everything up to step 4 only writes files and computes, so a failure there leaves the node running exactly the code it had.
That distinction is what the failure messages from any tool built on this mechanism should report, and what decides whether there is anything to recover from.

**What happens to your processes**

For each changed module the plan loads the new code, and for a stateful process it suspends the process, calls `code_change/3`, then resumes it.
The process is never restarted and never loses its mailbox, it is paused for as long as the migration of its own state takes.
Everything that follows in [Good practices](#good-practices-for-a-project-that-can-be-hot-upgraded) is about making sure the code holding that state can survive being replaced underneath it.

That suspend step only reaches a process `release_handler` can find by walking the supervision tree down from the application's top supervisor.
A process started any other way, a bare `spawn`, or one linked directly from an application's `start/2` outside any supervisor, is invisible to every appup instruction.
The upgrade still reports success, there is simply nothing in the plan that was ever going to touch that process, and nothing surfaces to say so.

### Which version performs the upgrade

A hot upgrade is carried out by the code already running, not by the code in the package.
The relup is built and installed using the version currently installed, and only afterwards is the new code in charge.

That has one consequence worth remembering whenever a release fixes something in the hot upgrade path itself: the fix applies to upgrades performed **from** that version, not to the upgrade **into** it.
Going from a version with the bug to a version with the fix still runs the bug, because the buggy code is what does the work.

So when a release changes how hot upgrades behave, check its changelog before hot-upgrading into it.
The same applies to a monitored application's own upgrade instructions, its `.appup` is executed by the release being replaced.

## Good practices for a project that can be hot-upgraded

None of this is required to deploy.
It is what makes the difference between a release that can be hot-upgraded and one that has to restart.

**Build both versions with the same Erlang and Elixir.**
A hot upgrade replaces application code inside the running VM, it cannot replace the runtime underneath it.
If the two releases were built with different Elixir versions, `systools` asks for an appup that upgrades Elixir itself, which nobody ships:

```bash
[error] systools:make_relup failed for myphoenixapp 0.4.0 -> 0.4.2, reason: Could not open
file /var/lib/myapp/service/myphoenixapp/myphoenixapp-4pvclf/current/lib/elixir-1.20.3/ebin/elixir.appup
```

Pin the toolchain in `.tool-versions` and keep it identical across the two builds.
Bumping Erlang or Elixir is a full deployment, not a hot upgrade.

**Assess every dependency you upgrade, one at a time.**
Dependencies can be hot-upgraded, [Jellyfish][jyf] generates their appups the same way it does for your own applications.
An appup only describes how to load the new code though, it says nothing about whether that library survives being replaced under load, so each one is a decision rather than a default.

> [!WARNING]
> Before performing a hot upgrade for any dependency, treat it with the same caution as your own application code.
> Check if the dependency officially supports hot upgrades between the specific versions you're upgrading, review the changelog for breaking changes or structural modifications (e.g. process state changes, supervision tree changes, API changes), and pay special attention to stateful processes like GenServers, Agents, and ETS tables that may require special handling or coordinated state migration.

Do that per dependency, not per release, and keep the number moving in a single hot upgrade small enough that the review stays realistic.

Reading a dependency diff is worth handing to an LLM.
Ask it to summarise what changed between the two versions and, specifically, whether anything there affects state held by a running process or the shape of a supervision tree.
Treat the answer as a place to start looking, not as the verdict, and confirm what it reports against the changelog and the diff itself.

**Do not let an anonymous function outlive the call that created it.**
A function value is tied to the version of the module that built it.
Once that module is replaced the value is dead, and calling it raises:

```bash
** (BadFunctionError) function #Function<0.17932506/0 in MyApp.Worker> is invalid,
likely because it points to an old version of the code
```

This is about lifetime, not about anonymous functions being risky in themselves.
Almost all of them are fine, and rewriting them would make the code worse for no gain.

Fine, the value is created and used within the same call, so no upgrade can land between the two:

```elixir
def totals(entries) do
  entries
  |> Enum.filter(fn entry -> entry.status == :running end)
  |> Enum.map(& &1.size)
end

def report(entries) do
  render = fn entry -> "#{entry.name} #{entry.size}" end
  Enum.map_join(entries, "\n", render)
end
```

A problem, the value is stored and called later, and later can be after an upgrade:

```elixir
# in GenServer state
{:ok, %{on_complete: fn result -> notify(result) end}}

# in a message the process will handle afterwards
send(self(), {:finished, fn -> cleanup(path) end})

# handed to something that runs for a while
Downloader.stream(url, on_progress: fn pct -> broadcast(pct) end)
```

Store `{module, function, args}` in those places and reach it with `apply/3`, which resolves against whatever is loaded at the time.

Ask one question about a function value: **can this still be called after the current call returns?**
If it can, it needs to be an MFA.
The places where the answer is yes are GenServer state, ETS tables, `:persistent_term`, timers, messages already sitting in a mailbox, and callbacks handed to something long-running such as a download or a stream.

Captures have the same lifetime rule.
`&Module.fun/1` of a public function is as safe as an MFA, since it is resolved through the module.
`&private_fun/1` and `&(&1 + n)` are closures over the current module version and are not.

**Let long-running processes re-enter their loop through the module.**
The VM keeps two versions of a module, the current one and the old one, and a process that is executing old code stays there until it makes a fully qualified call.
Load a third version and the old code is purged, taking any process still lingering in it with it.

OTP behaviours do this for you, since `GenServer`, `Supervisor` and friends invoke your callbacks through the module on every message, so a `GenServer` is on the new code as soon as the upgrade completes.
A hand-written process is not:

```elixir
# stays in old code forever, and dies at the next purge
defp loop(state) do
  receive do
    msg -> loop(handle(state, msg))
  end
end

# picks up the new code on the next message
defp loop(state) do
  receive do
    msg -> __MODULE__.loop(handle(state, msg))
  end
end
```

The same applies to anything that runs for a long time inside one call, a `Task` looping over a queue or a stream consumed over hours.
It keeps running the code it started with, and there is no supported way to change that from the outside.
Write those as behaviours, or make the loop re-enter through the module so an upgrade has a point at which to take effect.

**Know that purging kills by default.**
The `update` and `load_module` instructions each take a `PrePurge` and a `PostPurge` value, and a generated appup that leaves them out, which is the normal case, gets `brutal_purge` for both.
Under `brutal_purge`, a process still executing the old code when the purge runs is killed outright, which is exactly what happens to a process that never re-entered through its module as described above, or to a plain process that happened to be mid-call when a third version loaded.
`soft_purge` is the alternative: instead of killing, `release_handler:install_release/1` returns `{error, {old_processes, Module}}` and the purge waits until nothing is running the old code anymore.
A NIF library follows the same timing from the native side, its `unload` callback only runs once the module instance it belongs to is purged, so a NIF-backed module left under `brutal_purge` can have its process killed while a call into the library is still in progress.
Hand-edit the generated appup to set `soft_purge` on a specific module when this matters, the same way as [adding or removing a supervision-tree child](#adding-or-removing-a-process-in-the-supervision-tree).

**Give stateful processes a `code_change/3`.**
An appup can suspend a process, load the new module and resume it, but only your code knows how to reshape the state it was holding.
If a `GenServer` state changes shape between versions, write the migration.
If it does not, there is nothing to do.

A `gen_statem` callback module takes a different shape, `code_change/4`, with state and data as separate arguments and `{ok, NewState, NewData}` as the return.
Implement the callback your behaviour actually calls.
An appup instruction that specifies `{advanced, Extra}` against a module missing the matching callback does not skip the migration, it crashes the process with `undef`.

**Treat a supervision tree change as a step of its own.**
An upgrade changes code, it does not change the shape of the running system, so a server you added is not started and one you removed is not stopped.
This is the one case where a generated appup is not enough on its own, see [Adding or removing a process in the supervision tree](#adding-or-removing-a-process-in-the-supervision-tree).

**Keep functions out of your configuration.**
Application environment is written to `sys.config` and read back with `file:consult/1`, so every value has to be a term that can be written down and parsed again.
An anonymous function cannot, and a configuration file that fails to parse is silently replaced with an empty one, which resets the environment of every application in the release.
Use `&Module.fun/1` captures of exported functions, or plain data your code interprets.

**Keep the two versions compatible at their boundaries.**
For a while the old and the new version coexist: messages sent before the upgrade are handled after it, replicas are upgraded one at a time so they talk to each other across versions, and rows and ETS entries written by the old code are read by the new one.
Anything crossing those boundaries has to be readable by both, which makes the usual expand-then-contract sequence the safe shape.
The release that adds a field accepts both shapes, and the release that drops the old one comes after it.

**Do not hot-upgrade native code.**
A dependency that loads a NIF or a port driver brings a shared object that the VM loads outside the normal code path, and replacing it in a live node has rules of its own.
Unless the library documents support for it, move those versions with a full deployment.

**Give every build its own version.**
Upgrades are computed from one version to another, so two builds sharing a version cannot be told apart, and the appup has nothing to describe.

**Exercise the upgrade before production.**
Build the current version, build the target, then apply one to the other against a real node.
Building the two versions the same way belongs in CI, so the artifact used for the real exercise is never built by hand.
Applying the upgrade itself is best run by hand against a real node first, until you trust the process enough to automate it.

Booting that release, in CI or by hand, is its own source of failures that have nothing to do with the upgrade mechanism.
`vm.args` is commonly templated with placeholders such as `${RELEASE_NODE}` or one of your own, filled in at boot time only when `relx`'s `RELX_REPLACE_OS_VARS` (or the equivalent Mix release mechanism) is enabled and the referenced environment variables are actually set.
Boot the release anywhere that does not set them and the literal placeholder text is passed straight to the VM, so it never comes up.
That failure looks exactly like any other boot failure, a timeout with no further detail, which makes it easy to blame the upgrade itself rather than how the node was started for the test.
Check what `vm.args` expects substituted before booting a release outside its normal deployment environment, and supply those variables explicitly rather than letting the first sign of the problem be a timed-out CI job.

## Phoenix and LiveView pitfalls

Phoenix adds a few failure modes that are easy to miss because they do not come from your own `GenServer` state, they come from parts of the framework that were never designed with hot code loading in mind.

**`phx.digest` output is cached in memory, not recomputed by the upgrade.**
The static asset manifest (`priv/static/cache_manifest.json`) is read once, when the endpoint starts, and kept in memory for the life of the node.
A hot upgrade does not restart the endpoint, so it does not reread that manifest.
If a release changes any static asset, pages rendered after the hot upgrade keep linking to the **old** digested filenames, because that is what `static_path/2` still has cached, regardless of which code version is handling the request.
Treat any release that changes `priv/static` content as a full deployment, or make reloading the manifest an explicit step of the upgrade rather than assuming it follows the code.

**Old digested assets can be pruned out from under a live browser tab.**
A tab that loaded the page before the upgrade already has the old digested URLs baked into its HTML and JS.
If release cleanup removes the previous version's `priv/static` directory as part of normal retention, those URLs start 404ing for anyone who has not reloaded the page yet, hot upgrade or not.
Keep the asset retention window at least as long as you expect a user session to live, and remember a hot upgrade does not shorten that need, it just makes it easy to forget because "nothing restarted."

**LiveView's per-connection state is not owned by your module.**
A `code_change/3` migrates the state of the process whose module it belongs to.
The process backing a live connection is Phoenix's own channel process, your LiveView module is called through it, it does not run as that process's `GenServer` behaviour itself, so there is no `code_change/3` hook of yours in the migration path for whatever that process is holding for an already-connected session.
If a release changes the shape of data your LiveView stores in assigns in a way existing connections depend on, the practical options are: keep reads tolerant of both the old and new shape for one release (the same expand-then-contract rule as everywhere else), or accept that an affected live connection may error and remount through `mount/3`, which does pick up the new code. Decide which one on purpose, instead of finding out from a user's bug report.

**Closures in assigns are the same lifetime problem as anywhere else.**
A function value handed to a component as a prop, stored in `socket.assigns`, or captured in a slot, is tied to the module version that created it exactly like a closure in any other process state.
`assign(socket, on_select: fn item -> ... end)` held across a hot upgrade is the LiveView-flavoured instance of the anonymous-function rule above, not a special case, the fix is the same: pass `{module, function, args}` or a reference the current code resolves through a module call.

## Adding or removing a process in the supervision tree

A hot upgrade replaces code, it does not restructure the running system.
When a release adds a child to a supervisor, or removes one, the upgrade alone does not act on it:

* A server you **added** is not started. Its module is loaded, its child specification is recorded, and nothing runs it.
* A server you **removed** is not stopped. It keeps running, on the old code, until the node restarts.

That is `{update, Module, supervisor}` doing what it is defined to do.
It loads the new supervisor module and re-reads `init/1`, so the restart strategy and the child specifications the supervisor knows about are the new ones, but the children that are already running are left exactly as they are.

**Jellyfish cannot generate this for you.**
It builds the appup by comparing the compiled modules of the two versions, so it sees that a module was added, changed or deleted.
A child specification lives inside `init/1`, which is data the comparison cannot reach, so no start or terminate instruction is generated.
Nothing fails at build time and nothing fails during the upgrade either, the release installs cleanly and the new server simply is not there afterwards, which is the reason this one is worth knowing before it happens rather than after.

**What the appup needs.**
The instructions to add are `apply` ones, calling the supervisor API directly.
An appup is `{NewVersion, [{FromVersion, UpInstructions}], [{FromVersion, DownInstructions}]}`, and adding a child looks like this:

```erlang
{"1.1.0",
 [{"1.0.0", [
    {add_module, 'Elixir.MyApp.Cache'},
    {update, 'Elixir.MyApp.Supervisor', supervisor},
    {apply, {supervisor, restart_child, ['Elixir.MyApp.Supervisor', 'Elixir.MyApp.Cache']}}
  ]}],
 [{"1.0.0", [
    {apply, {supervisor, terminate_child, ['Elixir.MyApp.Supervisor', 'Elixir.MyApp.Cache']}},
    {apply, {supervisor, delete_child, ['Elixir.MyApp.Supervisor', 'Elixir.MyApp.Cache']}},
    {update, 'Elixir.MyApp.Supervisor', supervisor},
    {delete_module, 'Elixir.MyApp.Cache'}
  ]}]
}.
```

Removing a child is the same list the other way round, terminate and delete the child, update the supervisor, then delete the module.
Order is the part to get right: the module has to be loaded before a child using it is started, and the child has to be stopped before its module is deleted.
`supervisor:restart_child/2` is what starts a child the supervisor knows about but is not running, which is exactly the state `{update, Module, supervisor}` leaves a newly added specification in.

**Where to edit it.**
[Jellyfish][jyf] can pause the release so you can edit the file it just generated:

```bash
EDIT_APPUP=true MIX_ENV=prod mix release
# Press any key when you're done editing rel/appups/myphoenixapp/1.0.0_to_1.1.0.appup
```

The other way is to untar the release, edit `lib/<app>-<vsn>/ebin/<app>.appup`, and tar it again.
Either way, keep the downgrade list consistent with the upgrade one, and remember that the appup is executed by the version being replaced, see [Which version performs the upgrade](#which-version-performs-the-upgrade).

**Writing those instructions is a good task to hand to an LLM.**
Give it the `init/1` of the supervisor in both versions and the appup Jellyfish generated, and ask it to add the instructions for the children that appeared or disappeared, in the correct order.
It is a mechanical translation from a diff into a known instruction set, which is the kind of work it does well.
Read the result yourself before shipping it, the same way you would read the diff of a dependency: check the order, check that the child ids match the ones in `init/1`, and check the downgrade list.
Then apply the upgrade against a real node and confirm the tree, `Supervisor.which_children/1` tells you what actually runs after it.

The alternative is always available and sometimes the better trade: ship the tree change as a full deployment, and keep the hot upgrades for changes that only touch code.

## Checking whether a hot upgrade is possible

**Compare the toolchains first.**
If the `.tool-versions` used for the two builds differ in Erlang or Elixir, stop here, it is a full deployment.

**Look for the upgrade metadata in the package.**
An Elixir release carries a `jellyfish.json` per application, an Erlang one carries `.appup` files:

```bash
tar -tzf myphoenixapp-1.1.0.tar.gz | grep -E "jellyfish.json|[.]appup$"
```

Nothing listed means the release was built without hot-upgrade information and can only be deployed fully.

**Check the versions it claims to upgrade from.**
Extract a `jellyfish.json` and read its `from` and `to`.
For a project entry the `from` has to be the version currently running, so a release built against an older version is not applicable no matter how well formed it is.
A dependency entry carries the library's own versions instead, since dependencies are versioned independently from the project.

**Ask the node what it has installed.**
`:release_handler.which_releases()` lists the releases the node knows about and their status, which tells you what a previous attempt left behind:

```elixir
iex> :release_handler.which_releases()
...> |> Enum.map(fn {_name, vsn, _apps, status} -> {vsn, status} end)
[{~c"1.1.0", :permanent}, {~c"1.0.0", :old}]
```

A version left as `unpacked` is one a previous attempt started and did not install.

### Choosing the right release artifact

A hot upgrade replaces application code inside the running VM, it cannot replace the VM or the language runtime underneath it.
A release built for another OTP line brings a different Erlang and Elixir, and `systools:make_relup/4` needs an `.appup` for every application whose version changes, which neither ships.
Match the artifact's OTP version to the one the running node uses, not to the CPU architecture or OS image alone, both of those can match while the OTP line still differs.

## Going back

There is no hot downgrade in the general case.
A relup can describe the way back, and `release_handler` can run it, but most projects never generate or exercise one, so a release that turns out to be wrong is usually undone the same way any other bad release is: publish the previous version and let it deploy fully.

That is worth knowing while deciding, because it makes the two directions asymmetric.
The upgrade avoids downtime, the way back does not, unless you have specifically built and tested the downgrade path.
The more of your release that is state living inside processes, the more that asymmetry should push you towards being conservative about what goes into a hot upgrade at all.

## Glossary

| Term | What it is |
| --- | --- |
| `appup` | Per application upgrade instructions, from one version to another. Generated by [Jellyfish][jyf] as `jellyfish.json` plus a `.appup`, or written by hand as a `.appup`. |
| `relup` | The ordered plan for the whole release, built by `systools:make_relup/4` from two releases and their appups. |
| `release_handler` | The OTP module that unpacks, installs and makes a release permanent on a live node. |
| `unpacked` | Release status. The code is on disk and registered, nothing is loaded. |
| `current` | Release status. Installed and running, but not yet the one the node would boot into. |
| `permanent` | Release status. Installed and the one the node boots into next time. |
| `old` | Release status. A previously permanent release that has been replaced. |

## Additional Resources

- [Jellyfish][jyf] - AppUp file generation for Elixir releases
- [relup-guard-elixir](https://github.com/deployex/relup-guard-elixir) - CI action that applies a real upgrade against a booted Mix release on every pull request
- [relup-guard-erlang](https://github.com/deployex/relup-guard-erlang) - the same check for rebar3/relx releases
- [DeployEx](https://github.com/deployex/deployex) - a deployment tool that applies this mechanism to monitored applications and to itself
- [FlyDeploy](https://github.com/chrismccord/fly_deploy) - hot-upgrades a Mix release running on Fly.io machines, built on `:sys.suspend`/`:code.modified_modules`/`:sys.change_code`/`:sys.resume` instead of a generated appup, and falls back to a cold `fly deploy` for supervision-tree changes, config changes, NIF/port changes and Erlang/OTP version changes - the same boundaries this guide calls out for the appup-based mechanism
- [Erlang OTP Design Principles: Release Handling](https://www.erlang.org/doc/system/release_handling.html) - what `release_handler` does and in what order
- [Erlang Appup Cookbook](https://www.erlang.org/doc/system/appup_cookbook.html) - worked appups, including supervisors and special processes
- [Erlang Appup File Format Reference](https://www.erlang.org/doc/apps/sasl/appup.html) - the instruction set itself, including the `PrePurge`/`PostPurge` defaults
- [Erlang Compilation and Code Loading](https://www.erlang.org/doc/system/code_loading.html) - current and old code, purging, fully qualified calls

[jyf]: https://github.com/deployex/jellyfish
