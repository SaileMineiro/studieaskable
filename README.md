# Investigation of Mock Behavior with `isolate: false`

## Summary

I investigated Vitest mock behavior when one test file registers a factory mock and a later test file registers an automock for the same module while `isolate: false` is enabled.

This is more specific than the basic documented behavior of isolation because it involves a non-obvious interaction between:

- reuse of module context and cache;
- mock metadata stored on the module;
- changing from a factory mock to an automock;
- test-file execution order.

Vitest’s official regression test confirms that this scenario is significant and is associated with issue #10145.

## Environment

- Repository: Vitest monorepo.
- Investigated commit: `a0a939653` (`main`).
- Vitest: `5.0.1`.
- Node.js: `v24.12.0`.
- pnpm: `11.24.0`.
- Pool: `forks`.
- Workers: `1`.
- File parallelism: disabled with `fileParallelism: false`.
- Test concurrency: disabled.
- External services: none.

## Reduced scenario

The scenario uses an aliased `~/dep` module and two test files:

1. `a-factory.test.ts` registers a factory mock for `~/dep`.
2. `b-automock.test.ts` registers `vi.mock(import('~/dep'))`.
3. The files run with isolation disabled.
4. The automock file checks that the exports are mock functions and that `mockReturnValue` works.

The local sandbox also tested both controlled orderings:

```text
factory → automock
automock → factory
```

Both orderings passed on the current version. The sandbox validated the experiment structure and ordering control; the repository’s official regression test is the authoritative validation.

## Execution control

The reproduction used:

```ts
{
  isolate: false,
  pool: 'forks',
  maxWorkers: 1,
  fileParallelism: false,
  sequence: {
    concurrent: false,
    shuffle: false,
  },
}
```

The order was not inferred only from filenames or the order of CLI arguments. A stable sequencer was used in the reduced reproduction because `fileParallelism: false` prevents parallel execution but does not, by itself, guarantee ordering by command-line arguments.

## Official regression test

The official test is located at:

```text
test/e2e/test/mocking.test.ts
```

Test name:

```text
automocking works with isolate:false when factory mock runs first (resolve alias)
```

Command executed from `test/e2e`:

```powershell
..\..\node_modules\.bin\vitest.cmd run test/mocking.test.ts --pool=forks --maxWorkers 1 --no-file-parallelism --typecheck.enabled=false -t "automocking works with isolate:false when factory mock runs first"
```

Result:

```text
Test Files 1 passed (1)
Tests 1 passed | 17 skipped (18)
Type Errors no errors
Duration 1.96s
```

The test verified that:

- the factory mock works in the first file;
- the second file receives automocked functions;
- `mockReturnValue` works on the automock;
- there are no type errors.

## Implementation explanation

The relevant fix is commit:

```text
8e2108ddaec3c58501621fdd2d78929d87c383c8
fix: stale mock metadata breaks automocking with isolate:false (fix #10145) (#10541)
```

The parent commit is:

```text
9f23f8ec3413beff8072b837f0ed09329661dc5d
```

The commit changed:

```text
packages/vitest/src/runtime/moduleRunner/moduleRunner.ts
test/e2e/test/mocking.test.ts
```

Before the fix, the runner could continue using `mod.meta.mockedModule` even when the dependency’s current mock had changed. With `isolate: false`, stale metadata could interfere with applying the automock.

The fix obtains the current mock with:

```ts
const currentMock = this.mocker.getDependencyMock(mod.id)
```

It then behaves as follows:

- when there is no current mock, the original module is loaded;
- when the current mock is an automock/autospy and differs from stale metadata, the original module is loaded again and the current mock is applied;
- otherwise, the current mock is used normally.

The cause is therefore not simply that “`isolate: false` preserves state.” The relevant interaction combines shared module cache, stale metadata, a change in mock type, and file order.

## Plausible but incorrect predictions

### Prediction 1

`fileParallelism: false` should make Vitest respect the order of paths supplied on the command line.

This is incorrect. The option prevents files from running in parallel, but it is not, by itself, a guarantee of ordering by CLI arguments. The reproduction therefore controlled ordering explicitly.

### Prediction 2

A later `vi.mock()` always completely replaces every previous mock for the same module.

This is incorrect in the non-isolated scenario when stale metadata is reused. The runner must compare the current mock with the existing metadata and reapply the automock when they differ.

### Prediction 3

If the test passes on the current version, the scenario is not relevant.

This is also incorrect. The test is a regression test added by the commit that fixed the problem. Passing on the current version shows that the fix is active.

## Contamination probe

The probe ran on the corrected version with:

```text
isolate: false
pool: forks
maxWorkers: 1
fileParallelism: false
concurrent: false
stable file order
```

Results:

- factory → automock: passed;
- automock → factory: passed.

In the factory-first case, the second file confirmed that the exports were mock functions and that `mockReturnValue` worked.

This shows that the corrected behavior does not depend on accidental ordering or parallel scheduling.

## History and limitations

I confirmed that the official regression test was added in commit `8e2108dda` with:

```text
git log --all --oneline -S"automocking works with isolate:false when factory mock runs first" -- test/e2e/test/mocking.test.ts
```

Result:

```text
8e2108dda fix: stale mock metadata breaks automocking with isolate:false (fix #10145) (#10541)
```

I also confirmed that the fix is contained in the current commit:

```powershell
git merge-base --is-ancestor 8e2108dda HEAD
```

Result:

```text
0
```

I created a worktree at the parent commit, `9f23f8ec3`, to attempt the historical pre-fix run. The pre-fix execution was not completed because installing the monorepo dependencies exceeded the available memory on Windows. Therefore, this report does not claim that the historical failure was directly observed in this environment.

The main checkout was clean after the investigation:

```text
git status --short
```

with no output.

## Assessment

This case goes beyond the documented statement that isolation changes state sharing. It presents a concrete non-obvious interaction, traces the cause to `moduleRunner`, controls pool, worker count, parallelism, and order, and validates the correction with the official regression test.

The important facts are:

- `isolate: false` permits context/module-cache reuse between files handled by the same worker;
- `fileParallelism: false` does not define file order by itself;
- factory mock followed by automock is the relevant order;
- stale metadata can differ from the current mock;
- the corrected runner reads the current mock and reapplies automocking when necessary;
- the official regression test passes on the corrected version;
- the pre-fix execution was not completed because of the memory limitation.

## Conclusion

The investigation found the requested concrete case, pinned the environment and execution parameters, traced the mechanism in the source code, and validated the official regression test on the corrected version.

Commit `8e2108dda` is the fix for issue #10145. The investigated branch already contains that fix, so no new source patch was created. The only remaining limitation is that a pre-fix run could not be completed because dependency installation exceeded the available memory.
