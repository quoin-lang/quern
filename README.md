# quern

A parallel task runner for [Quoin](https://github.com/quoin-lang/quoin) projects — `make`'s dependency graph with Quoin's concurrency underneath. Tasks are blocks in a `Quernfile.qn`; quern runs them **dependency-driven**: every task starts the moment its last dependency finishes, bounded by `--jobs`, with each task's subprocess output streamed live under its own colored `[task]` prefix.

```quoin
"* Quernfile.qn
Quern.tasks:{
    (.task:'build' does:{ .sh:'cargo build --release' })
        .info:'compile everything';
    .task:'test' needs:'build' does:{ .run:#( 'cargo' 'nt' ) };
    .task:'lint' needs:'build' does:{ .sh:'cargo clippy -q' };
    var dist = .task:'dist' needs:#( 'test' 'lint' ) does:{ .sh:'./package.sh' };
    dist.sources:#( 'target/release/qn' );
    dist.produces:#( 'dist/qn.tar.gz' );
    .default:'dist'
}
```

```
$ quern              # the default task and everything it needs, in parallel
$ quern test lint    # just these (plus their deps)
$ quern --list       # what's defined
$ quern -j 8 --fail-fast --quiet
```

## The model

- **A Quernfile is ordinary Quoin.** `Quern.tasks:{ … }` runs the block against the task graph; `.task:… does:{ … }` declares, `needs:` draws the edges (a name or a list), `.default:` names the bare-`quern` target. Declarations answer the task, so the extras (`info:`, `sources:`/`produces:`, `timeout:`, `quiet!`) attach to it — bind the task to a `var` and add them as separate statements (as `dist` does above), or parenthesize each link: `((.task:'doc' …).sources:#( 'lib/' )).produces:#( 'qn-docs/' )`. Don't chain an extra *after a list or string argument* without parens — ordinary Quoin binding applies, so `.sources:#( … ) .produces:( … )` sends `produces:` to the **list**, not the task. End each declaration statement with `;` — the same separator rule as everywhere else in Quoin.
- **Dependency-driven parallelism.** The plan is the transitive closure of what you asked for, cycle-checked. Execution never waits on artificial layers: a slow leaf holds back only its own dependents. `--jobs N` bounds how many run at once (default 4).
- **Freshness, when you declare it.** A task with `produces:` is skipped when every product exists and none is older than any `sources:` entry — `make`'s contract, from file mtimes. Both sides take files or **folders**: a source folder contributes its newest file (recursively), a produce folder its oldest, so `sources:#( 'lib' ) produces:#( 'qn-docs/' )` rebuilds the docs whenever anything under `lib/` changes. Dot-entries are ignored; tasks without `produces:` always run.
- **Failure is contained by default.** A failed task marks its dependents *skipped*; unrelated tasks finish. `--fail-fast` instead cancels everything in flight — and a cancelled task's subprocesses die with it (Quoin's process lifecycle owns that), so nothing leaks.
- **Steps.** Inside `does:` blocks, `self` is the task's context: `.run:#( 'cmd' 'arg' )` (an argv list — **no shell**, nothing splits or expands), `.sh:'cmd | pipe'` (explicit `sh -c` semantics), both with `dir:` variants; `.log:'…'` prints under the task's prefix. A nonzero exit throws, failing the task. Bodies may also take the context as a parameter: `does:{ |t| t.sh:'…' }`.
- **Output you can read.** Interleaved lines carry colored `[task]` prefixes. `--quiet` captures instead, replaying a task's output only when it fails. `--dry-run` prints the plan and runs nothing; the exit code is `0` only when everything needed came out ok or fresh.

## Layout

```
Quernfile.qn      your project's tasks (loaded from the current directory, or --file)
bin/quern         the command (a Quoin shebang script; put it on your PATH)
lib/*.qn          the library: Quern, [Quern]Graph/Task/Run/Ctx, [OS]Process.shell:
tests/*.qn        qn test tests
```

The library is usable without the command: build a `[Quern]Graph`, then `([Quern]Run.new:{ var graph = g }).run:#( 'task' )` answers a `[Quern]Summary` — that's exactly how quern's own tests drive it.

## Requirements

A `qn` binary on the PATH. No other dependencies — quern is pure Quoin over the standard library (`[OS]Process`, `[CLI]Spec`, `Channel`/`Task`/`Async`, `Term`).
