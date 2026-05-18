# Z3Wire comparison semantics redesign

Repo: https://github.com/qobilidop/z3wire
PR: #27 (merged at 217dd8f8 on 2026-05-11T01:39:26Z)
Follow-up: issue #28 (Bazel CI speed)

## Decision summary

Replaced heterogeneous symbolic comparison operators (`==`, `!=`, `<`, `<=`, `>`, `>=`) on `SymBitVec` with **strict** versions requiring matching width AND signedness. Added free-function family `z3w::math_eq`/`math_ne`/`math_lt`/`math_le`/`math_gt`/`math_ge` carrying the previous heterogeneous mathematical (value-based) semantics. Concrete `BitVec` and `Bool`/`SymBool` untouched.

Rationale (in PR description and `docs/design/mathematical-comparison.md`, title now "Comparison Semantics"):

- Old behavior was mathematically correct but silently traversed type boundaries; conflicted with library's "explicit over implicit" principle (bitwise already strict, arithmetic uses bit-growth).
- For `==`, `SymUInt<8>(255) == SymSInt<8>(-1)` is `false` mathematically vs `true` bit-pattern — strict matching forces user to pick `math_eq` or cast.
- Cross-language precedent: Rust, Go, Haskell, Zig all require matching types for `==`/`<`. Swift went heterogeneous-for-all-six and has documented problems with overload resolution + integer literal overflow. C++20 `std::cmp_*` is the direct precedent for the `math_*` family.

Out of scope (flagged in design doc / roadmap, not done):
- Concrete heterogeneous `math_*`.
- New ordered comparison operators on concrete `BitVec` (only `==` exists; `!=` via C++20 rewriting).
- Native-integer comparison (`x == 0`). Design note: under strict semantics, native int must convert at *symbolic operand's* width/signedness, not its native width, so the mixed-operand strict operator works without `math_*`. Roadmap and `docs/design/lossless-auto-promotion.md` updated.
- Compile-fail tests for `!=`/`<=`/`>`/`>=` (only `==` and `<` have dedicated tests; the other four reuse the same static_assert mechanism).

## Implementation notes

Branch progression (additive → migrate → tighten, every commit green):

1. Add `math_eq`/`math_ne` (sym×sym + mixed concrete+sym overloads).
2. Add `math_lt`/`math_le`/`math_gt`/`math_ge`. Derived three delegate to `math_lt`.
3. Migrate cross-type operator call sites in `examples/midpoint_overflow.cc` and `examples/usage/operations.cc`.
4. Tighten `==`/`!=` with `static_assert(W1 == W2 && S1 == S2, ...)`. Add 2 compile-fail tests. Surfaced 3 cross-type tests in `z3wire/sym_bit_vec_test.cc` that needed migration too.
5. Tighten `<`/`<=`/`>`/`>=` same way. Add 2 compile-fail tests. Surfaced 4 more cross-type tests needing migration.
6. Docs: rewrote `docs/usage/operations.md` (Integer comparison section), `docs/design/mathematical-comparison.md`. Updated cross-refs in `docs/design/{index,concrete-types,bit-growth-arithmetic,lossless-auto-promotion}.md`.

Post-tasks cleanup commit: refactored `math_ne` to `!math_eq` (consistency with the `math_le/gt/ge → math_lt` pattern), updated stale `lossless-auto-promotion.md` row + `roadmap.md` bullet. Final pre-PR commit: formatting fixes for CI (clang-format wrapped two section headers; shortened to `// --- Mathematical equality (heterogeneous) ---`).

Implementation pattern for strict operators — keep two-parameter template form `<W1, S1, W2, S2>` so the diagnostic is the targeted `static_assert` not generic "no matching overload":

```cpp
template <size_t W1, bool S1, size_t W2, bool S2>
SymBool operator==(const SymBitVec<W1, S1>& lhs, const SymBitVec<W2, S2>& rhs) {
  static_assert(W1 == W2 && S1 == S2,
                "comparison requires matching width and signedness; use "
                "z3w::math_eq(a, b) for mathematical equality, or cast one "
                "operand to the other's type");
  return SymBool(lhs.expr() == rhs.expr());
}
```

Compile-fail tests expect substring `"comparison requires matching width and signedness"`; `tests/compile_fail/BUILD.bazel` uses fixed-string match.

## Workflow lessons

- Subagent-driven-development worked well end-to-end (6 plan tasks × {implementer + spec-review + code-quality + occasional fix} = ~24 subagent invocations). The two-stage review caught real issues every task — typically docstring/comment cleanup or test-name consistency.
- Forward-reference trap: comments referencing "operators are strict" landed in commits *before* operators became strict. Reviewer caught it; fix is to phrase comments so they're true at every commit on the branch, or omit the claim and let the section header carry it.
- Cross-type comparison call sites are hard to grep for. My planning-time regex (`SymUInt<N>.*SymUInt<M>` on same line, or literal `operator==`) missed many — Tasks 4 and 5 had to migrate 3 and 4 additional test sites respectively. Mitigation: bake "surface unexpected sites in Step 6 of the full build" into the plan, which it did.
- Doc renames have ripple effects. Renaming `mathematical-comparison.md`'s *title* (file kept its name) required updating `docs/design/index.md` display name + cross-link display texts in `bit-growth-arithmetic.md`, `concrete-types.md`. Always grep for old title strings.
- IDE clang diagnostics in this repo are noisy (`'z3++.h' file not found` etc.) — clang LSP doesn't see Bazel's include paths. Bazel build is ground truth; ignore the IDE diagnostics.

## Tooling notes

- Worktree workflow: used `EnterWorktree` (native tool) for `comparison-semantics`. `.agent_scratch/` is gitignored and doesn't propagate to new worktrees — had to `cp` spec/plan files from main checkout. After merge, `ExitWorktree action: remove discard_changes: true` cleans up (the discarded commits live in the merge).
- Auto-format wrapped `// --- Mathematical equality (heterogeneous: any width/signedness combination) ---` ugly because clang-format col limit is 80. Workaround: shortened to `(heterogeneous)`. The detail moved to per-function docstrings.
- Project preference confirmed via this session: trivial fixups amend (with `--amend --no-edit`), not new commit. Used freely throughout per-task review cycles. Did NOT amend after pushing to remote — created a new "Fix formatting per CI" commit instead to avoid force-push.
- Implementer commit messages: subagents (Sonnet 4.6) credited themselves correctly via `Co-Authored-By:`. Controller (Opus 4.7) credited self. Don't normalize across; respect what each actor wrote.

## Spec/plan artifacts

Still in local scratch (gitignored):
- `~/i/z3wire/.agent_scratch/2026-05-10-comparison-semantics-design.md`
- `~/i/z3wire/.agent_scratch/2026-05-10-comparison-semantics-plan.md`

## Follow-up: issue #28 (Bazel CI speed)

Observation: Bazel CI rebuilds protobuf on every run (transitive dep via `fuzztest`). On main, runs take ~1m42s–1m58s with ~48% action cache hit rate. On this PR, the Bazel job once ran ~30 min vs main's ~2 min baseline — the cache-miss problem can degrade dramatically. PR did not touch any cache-key input (`.bazelversion`, `MODULE.bazel.lock`, `.bazelrc`, `MODULE.bazel`), so the slowdown isn't ours; it's a pre-existing CI characteristic.

Issue #28 proposes three options (ranked by leverage):
1. Use BuildBuddy's "Compare invocations" view to find action-hash drift root cause. Highest leverage; ~30–90 min investigation. The CI is already wired to BuildBuddy.
2. Tag `tests/fuzz/*` cc_test targets with `tags = ["manual"]` so `bazel build //...` skips them. Eliminates protobuf from default build; fuzz tests gated to dedicated CI job. ~30 min implementation. Cleanest sidestep.
3. Pre-warm cache nightly via `schedule:` trigger on main. Easy (~15 min) but only effective if option 1 reveals stable hashes; otherwise pre-warmed cache misses too.

Issue has timing data, cache-stats from a representative main run, dep-path verification, and explicit closing criteria.

## Quick-recall facts

- Final merge commit on main: `217dd8f8`.
- PR description in https://github.com/qobilidop/z3wire/pull/27 captures the design rationale verbatim from the design doc; useful if the design doc is later changed.
- Z3 + Bazel: `bazel_dep(name = "z3", version = "4.15.2")`. C++20 required.
- Worktree symlink (`bazel-comparison-semantics`) breaks `bazel //...` wildcard inside worktrees; use explicit globs `//z3wire/... //examples/... //tests/...`. Known pre-existing quirk.
- `docs/dev/style.md` rule: free functions `snake_case` (deviates from Google's PascalCase for free functions to match Z3 + std).
- `docs/dev/workflow.md` quality checks before commit: `./dev.sh bazel build //...`, `bazel test //...`, `./tools/lint.sh`, `./tools/docs.sh`, `./tools/format.sh` (format last).
