# Session 02 — Z3Wire packet_view example + CI cache investigation

Repo: https://github.com/qobilidop/z3wire

## Outcomes

- Shipped PR #29 (merged squash as `abcd348`): structured-view demo `examples/packet_view.cc`. 8 PR commits, +251 LOC, 4 files in `examples/`. No library headers touched.
  - URL: https://github.com/qobilidop/z3wire/pull/29
- Posted + revised analysis on CI-slowness issue #28: https://github.com/qobilidop/z3wire/issues/28#issuecomment-4417862498

## Workflow used (full superpowers loop)

1. `brainstorming` → 4 clarifying Qs (domain, leaf-field API, declaration style, reusable-via-BaseOffset) → spec at `.agent_scratch/2026-05-10-structured-view-example-design.md` (in z3wire repo `.agent_scratch/` per AGENTS.md, gitignored).
2. `writing-plans` → 7-task plan at `.agent_scratch/2026-05-10-structured-view-example-plan.md`.
3. `using-git-worktrees` → `EnterWorktree(name: packet-view-example)`. Worktree path: `/Users/qobilidop/i/z3wire/.claude/worktrees/packet-view-example`. Branch `worktree-packet-view-example`.
4. `subagent-driven-development` → 3 implementer phases + spec-compliance + code-quality reviews after each.
5. Final code review + `gh pr create`.
6. After merge: `ExitWorktree(action: remove, discard_changes: true)` discarded 8 PR commits (squash merge made original SHAs orphan).

## Design captured (so a future session can rebuild it)

- 4 class templates, each templated on `(int BufW, int BaseOffset)`, holding `z3w::SymUInt<BufW>* buf_`:
  - `Ipv4AddrView` — `value()/set_value()` for full 32 bits + `octet0..3()/set_octet0..3()`.
  - `Ipv4View` — leaf fields `version/ihl/dscp_ecn/total_len` + public sub-view members `src_addr` (local offset +32) and `dst_addr` (+0).
  - `EthView` — leaf fields `dst_mac/src_mac/ethertype`.
  - `HdrView<BufW>` — composite, no `buf_` of its own, `static_assert(BufW >= 432)`, holds `EthView<BufW, 320>` + `Ipv4View<BufW, 160>`.
- Local bit indices are LSB-relative within each view. `extract<Hi + BaseOffset, Lo + BaseOffset>` + `replace<Lo + BaseOffset>`.
- Sub-view members init order must match declaration order (C++ language rule) — caught a `-Wreorder-ctor` warning that surfaced only on Phase 2's first template instantiation.
- API style: explicit methods, no proxies/operator= magic. Asymmetry between leaf `field()/set_field()` and sub-view `value()/set_value()` is documented as accepted.

## References worth remembering (closest cousins)

- Chisel `Bundle` (Scala HW DSL): `pkt.ipv4.src_addr.octet0 := 10.U` — same shape, `:=` operator out of reach in plain C++.
- SystemVerilog packed structs.
- P4 headers.
- CMSIS / modm-io register access.

## Code-review findings I had to act on

- Phase 1 review (minor): unused `buf_` in `HdrView` — fixed in Phase 2 (dropped member, simplified ctor).
- Phase 2 implementer surfaced `-Wreorder-ctor` in `Ipv4View` — fixed init list order `: src_addr(buf), dst_addr(buf), buf_(buf) {}` to match declaration order. Amended into Phase 2 commit (user preference: trivial fixups amend, not new commit).
- Phase 2 code-quality review (important): commit message body omitted 2 of 4 bundled changes; format had awkward line breaks in one constraint. Re-ran `./dev.sh ./tools/format.sh` (changed 3 places); amended message to list all 4 bundled changes.
- Phase 3 lint surfaced 12 `modernize-use-nodiscard` errors — implementer added `[[nodiscard]]` to all const accessors. Project convention: state-returning accessors get `[[nodiscard]]` (e.g. `z3w::SymBitVec::expr()`). The 7-task plan missed this; should have been baked into Phase 1.

## Lessons (saved to memory)

- `feedback_pr_test_plan_verified.md` — verify every PR test plan item locally before opening, mark `[x]` for passing. User caught me leaving boxes unchecked.
- Added pointer in `MEMORY.md`.

## Issue #28 investigation (CI is slow)

**Setup:**
- Bazel CI uses `actions/cache@v5` for disk cache + BuildBuddy remote cache.
- Bazel runs inside dev container via `devcontainers/ci@v0.3` with `cacheFrom: ghcr.io/qobilidop/z3wire-dev`.
- Disk cache key: `bazel-${runner.os}-${hashFiles('.bazelversion', 'MODULE.bazel.lock', '.bazelrc')}`.

**Key data points (gh run view summaries):**
- Main post-#27 (`gh run view 25645863174 --log`): `2448 processes: 245 disk + 916 remote + 1287 internal` — 0 fresh.
- Main post-#29 (`25652225593`): `2454 processes: 245 disk + 918 remote + 1291 internal` — 0 fresh, 1m56s wall.
- **PR #27 build (`25644651556`):** `2448 processes: 245 disk + 667 remote + 1287 internal + 249 processwrapper-sandbox` — **249 fresh actions**, 43m48s wall.
- PR #29 (`25652067045`): 1m49s wall (faster than main baseline).
- Same runner image both runs: `ubuntu-24.04` version `20260413.86.1`. Same disk cache key. Ruled out runner drift.

**Conclusions reached (after correction):**
- Main is fully cached. `internal` ≠ cache miss; it means "ran without subprocess" (symlinks, header scans). The protobuf `Compiling ...` log lines on main are Bazel **replaying cached stderr**, not fresh compile. (Original issue's interpretation was wrong.)
- PR slowness comes from external dep rebuilds. The 249 fresh actions on PR #27 sampled in the log:
  - `Compiling absl/strings/cord.cc`
  - `Compiling src/google/protobuf/compiler/cpp/helpers.cc [for tool]`
  - `Compiling src/google/protobuf/compiler/command_line_interface.cc [for tool]`
  - etc. — abseil + protobuf-for-tool, not Z3Wire's own code.
- User correctly intuited this; I dismissed it in first comment draft and had to retract.

**Open: why do 249 specific dep actions miss BuildBuddy on PR but hit on main?**
Two hypotheses (different fixes):
1. BuildBuddy retention eviction between last main run (May 7) and PR #27 (May 11).
2. Non-hermetic input causing action-hash drift. Most likely candidate: dev container layer hashes drift between runs since the workflow builds it per run (only base layers cached via `cacheFrom`). Different toolchain binary content → different action hashes.

**Next investigation step (if revisiting):** Use BuildBuddy "Compare invocations" UI to diff a PR-run invocation against a same-day main invocation. If same action shows different hashes between the two = (2). If hashes match but PR says "miss", main says "hit" = (1).

**Implications for issue's 3 proposals:**
- Proposal 1 (BuildBuddy diff) — highest leverage; right comparison is PR vs main, not main vs main.
- Proposal 2 (manual fuzz tests / drop protobuf from default `bazel build //...` graph) — independently good; shrinks surface that (1) or (2) can affect.
- Proposal 3 (nightly pre-warm) — only useful if (1).
- Earlier "split sym_bit_vec.h" angle from my first draft: **retracted** — wouldn't help. The cascade isn't through Z3Wire's headers; it's at the dep layer.

## Useful gh / bazel commands used

- `gh run list --workflow=bazel.yml --branch main --repo qobilidop/z3wire --limit 10`
- `gh run view <id> --repo qobilidop/z3wire --log | grep -E "processes:|Cache restored|disk cache hit|remote cache hit"`
- `gh pr view 27 --repo qobilidop/z3wire --json statusCheckRollup,mergedAt,createdAt`
- `gh issue comment 28 --repo qobilidop/z3wire --edit-last --body "$(cat <<'EOF'...EOF)"` — editing my last comment is the way (skill default would otherwise add a new comment).
- `./dev.sh bazel build //...`, `./dev.sh bazel test //...`, `./dev.sh ./tools/lint.sh`, `./dev.sh ./tools/docs.sh`, `./dev.sh ./tools/format.sh` — z3wire's standard quality gate per `docs/dev/workflow.md`.
- CMake mirror also tested: `./dev.sh cmake -B build && ./dev.sh cmake --build build --target packet_view`.

## Pitfalls hit

- Bazel cache hit was so good after edit (action hash identical because reorder-ctor change is cosmetic; codegen unaffected) that I couldn't tell from build output whether my fix took effect. Resolved by trusting C++ semantics (declaration order rules construction order regardless of init-list ordering); spec reviewer verified independently.
- Editor LSP (clang) repeatedly emitted `'z3++.h' file not found` diagnostics on the example file. These are noise — clang outside the Bazel sandbox can't find the include paths. Real Bazel build is fine. Don't act on them.
- Initial PR test plan had all unchecked boxes; user pushed back. Lesson saved to memory.
- First draft of issue #28 comment dismissed the user's dep-rebuild intuition. Had to retract and revise.

## Worktree quirk

When ExitWorktree on a squash-merged branch: original PR commits are not on main (squash → new SHA), so the tool reports "Discarded 8 commits". Expected; the work is on main via the merge commit. `discard_changes: true` is correct here.
