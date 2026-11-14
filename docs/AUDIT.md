# rmc audit, November 2026

An end-to-end review of `main` after PRs #3 (the rustc-MIR route and the
`-Zcodegen-backend` dylib) and #8 (front-end breadth) were merged, at commit
`28cf99f`. It records what was measured, every defect found, what was changed,
and what is still open. The plan for the open items is in
[`ROADMAP.md`](ROADMAP.md).

## Method

Everything below was run, not inferred, on an Intel Xeon (4 cores, 2.1 GHz),
Linux 6.18, with the pinned `rustc 1.102.0-nightly (3ef0c22cd 2026-11-11)` and
with stable `rustc 1.97.0` (CI's `test` job uses stable).

* every existing suite: `cargo test`, `tests/run.py`, `tests/mir`, `tests/std`,
  `tests/cb`, `tools/mir_survey.py`;
* ten **realistic programs** written for this audit and never used to tune rmc:
  a bytecode interpreter, a quicksort, an open-addressing hash table, BFS on a
  grid, Conway's Life, N-queens, fannkuch-redux, a matrix multiply, a CSV-like
  text parser and a particle simulation (now `bench/real/` and
  `tests/mir/real_*`), each built by all four pipelines and compared against
  LLVM's output;
* seven **idiomatic** programs (`Vec` + sort, `HashMap`, iterator chains with
  closures, traits and `dyn`, `String` formatting, `Option`/`Result` with `?`,
  `f64`);
* a new **differential fuzzer for the MIR route** (`fuzz.py --mode mir`),
  about 4 000 programs across both MIR pipelines, with and without overflow
  checks;
* **compile-time scaling** on generated programs of ~1k, ~9k and ~46k lines;
* profiles with `callgrind`, and bisection with the new `RMC_SKIP_MIR_PASSES` /
  `RMC_SKIP_PEEPHOLE` switches.

## Summary

Before this round, every suite passed, yet:

* the rustc-MIR route **miscompiled or crashed on 5 of the 10 realistic
  programs** (wrong output on one, segfaults on four) and failed to assemble a
  sixth; the fuzzer then found a further miscompile class;
* it **rejected every ordinary `std` program** that reads its input with
  `std::env::args()` or uses `std::hint::black_box`, including the repository's
  own `bench/rust/*.rs`; its benchmarks ran only in a `no_std` dialect;
* five MIR test cases read `argv[0]` as their input, so they never tested what
  they claim, one of them the only overflow-check test;
* building a 46k-line program took **28.5 s** through rmc's own front end
  (1.1 s before PR #3) and **15 s** through the codegen backend;
* the claim of run-time parity with LLVM held on the twelve programs the
  optimisers were tuned on, not on held-out code.

After this round, all ten realistic programs and the 4 000 fuzz programs match
LLVM through both MIR pipelines, the two quadratic compile-time costs are gone,
and CI runs the checks that would have caught each of these.

## 1. Silent miscompiles (wrong output, exit status 0)

| # | symptom | root cause | fix | test |
|---|---|---|---|---|
| 1 | `5 - x` returned `x - 5`; `0 - x` and `x.wrapping_neg()` returned `x` | peephole `fuse_imm` turned `mov $k,%t; sub %s,%t` (`k - s`) into `lea -k(%s),%t` (`s - k`) | fold only `add`; never fold when the next instruction reads the flags | `tests/mir/constleft`, `constleftovf`, 4 unit tests |
| 2 | under overflow checks, a checked `k - x` / `k + x` tested stale flags | the same folds, and `fuse_lea`, dropped the flag-setting instruction before a `seto` | flag guard in both folds; `flags_used_by` covers every `cmov` | `constleftovf` |
| 3 | `a * (4 * b)` computed `(4*b)²` | the peepholes split operands at the first comma, so `lea 0(,%r15,4), %rax` was not seen to write `%rax` and `forward_stores` reused a stale value; a three-operand `imul` had the same blind spot | `split_ops`: split at the last comma outside parentheses, at all 17 sites | 3 fuzz programs in `tests/std`, 3 unit tests |
| 4 | dynamic width/precision (`{:1$}`, `{:.*}`) printed a truncated line | `ArgumentType::Count(n)` was parsed as a one-field struct; rustc stores it in the `value` niche (`[0, n]`) | lower it to `[0, n]` | `tests/std/fmtflags` (6 shapes) |
| 5 | `[0u8; 24]` next to another array clobbered it | `[v; n]` was stored with an 8-byte stride whatever the element | store with the element's size; copy aggregate elements whole | `tests/mir/repeatelem`, `real_textparse` |

## 2. Crashes and invalid assembly

| symptom | root cause | fix | test |
|---|---|---|---|
| `fannkuch`, `matmul` segfault | after inlining a function that writes through `&mut [T; N]`, the parameter copy was deleted: `use_counts` ignored the locals a *destination* reads (`(*_p)[i] = x`) | count dereferenced bases and index locals in destination places | `real_fannkuch`, `real_matmul` |
| `vm` segfault | every function numbered its helper labels `.Lrmc1, .Lrmc2, ...`; the in-process assembler let the last definition win, so a loop jumped into another function | labels carry the function name; a duplicate symbol is an error | `real_vm`, unit test |
| `textparse` segfault (MIR route) | the 8-byte `[0u8; 1 << 20]` splat wrote 8 MiB past a 1 MiB array | as row 5 above | `real_textparse` |
| `sort` failed to assemble (`not a register`) | `emit_cmp` emitted `cmp mem, mem` when neither operand had a register | load one side first | `real_sort` |
| AST route: `[u8; 1 << 20]` segfaulted on entry | the AST back end stores every array element in an 8-byte slot, so the frame needs 8 MiB | a clean compile error above 6 MiB, with the reason | unit test |

## 3. The two link paths disagreed

`--binutils` (GNU `as` + `ld`) must behave like the in-process assembler; only
the AST suite checked that.

* Duplicate local labels: `as` refused the file; the in-process assembler
  silently retargeted jumps (section 2).
* `cmp $32, -472(%rbp)` names no register, so `as` assembled a 32-bit compare
  (with a warning) where the in-process encoder used 64 bits; a bounds check
  compared only the low half of the index. A final pass now gives every
  immediate-to-memory ALU instruction an explicit `q` suffix.

`tests/mir/run.py` now builds every case both ways, requires identical
behaviour and fails on any `as` warning.

## 4. Front-end defects (rmc's own front end)

* `let b: [u8; 24] = [0; 24]` was rejected: the declared element width reached
  list literals but not a repeat literal's value. Fixed; covers every integer
  width.
* Byte arrays take eight bytes per element (above). Rejected cleanly for now;
  packing them is on the roadmap.

## 5. Test-suite weaknesses

* **Vacuous cases.** In `tests/mir`, `arg(0)` is `argv[0]`. `ovfcheck`,
  `srread`, `drop`, `loopcount` and `loopneg` read it as their input, so their
  argument vectors never reached the program. `ovfcheck`'s "overflowing"
  vector never overflowed: both binaries exited 0, and the MIR route's overflow
  checks were untested. All five now read `arg(1)...`, with real argument
  vectors.
* **Suites outside CI.** `tests/std` was not run in CI and had a failing case
  (`fmtflags`, defect 4). It now runs.
* **No randomised coverage of the MIR route.** Every fuzz mode fed rmc's own
  front end, and the existing generators emit constructs the MIR route refuses,
  so they could not reach it. `--mode mir` generates programs inside the
  supported subset. Its validity was checked: rustc accepts 30/30, rmc compiles
  all; with defect 1 reintroduced it fails at the fifth program. It found
  defect 3.
* **Toolchain-locked harnesses.** Reference builds hard-coded mangled panic
  symbols (already being fixed upstream); `tests/mir` still stubbed them only
  for `.overflow` cases, so any case with a runtime index failed to link. It
  now stubs from each object's `nm -u`.

## 6. Structure

* **Three copies of the MIR pass list** (CLI, dylib, `rmcmir`) had drifted: two
  skipped `prune_unreachable`, and the CLI parsed with overflow checks always
  off. Now one `lib.rs::optimize_mir`.
* **Six benchmark harnesses** over three copies of the programs. Now `run.py`
  (vs rustc), `run_four.py` (four back ends, one source, `core` + held-out
  `real` sets, geometric means, a refusal or miscompile reported instead of
  aborting) and `scale.py`, sharing `pipelines.py`.
* **A flat `compiler/src`** of sixteen files up to ~10k lines. Now grouped into
  `front/`, `opt/`, `mir/` and `x86/`, with compatible re-exports.
* **A 3 348-line `AGENTS.md`.** Now a short rules file; the long form is
  `docs/design-notes.md`.
* **Inaccurate claims.** The dylib was described as receiving `mir::Body` with
  no MIR text; it renders MIR with `write_mir_pretty` and re-parses it. The
  README's runtime claims compared against the full `rustc` driver with `std`
  and against tuned programs only.

## 7. Performance

### Compile time

Two quadratic algorithms, both fixed with byte-identical output:

* `mir::block_reachability` was an O(n^3) Floyd-Warshall closure; a function
  with 2 000 calls spent most of 14.7 s there. A DFS per block gives the same
  matrix (6.4 s).
* `elf::relax_branches` (added with PR #3, so it also slowed rmc's own front
  end) re-summed every removal and pad per branch per round and spliced the
  buffer in place: 84% of a 9k-line AST build. Prefix sums and a single-pass
  rebuild take that build from 1 459 ms to 154 ms; 184 programs assemble to
  identical bytes.

Current scaling: [`bench/RESULTS-SCALE.md`](../bench/RESULTS-SCALE.md). The
codegen backend's remaining super-linear cost is the text peepholes on long
function bodies.

### Run time

From [`bench/RESULTS-4WAY.md`](../bench/RESULTS-4WAY.md) (same source for all
four pipelines; ratio = LLVM time / pipeline time, geometric mean, so below 1
means slower than LLVM):

| | `core` (tuned) | `real` (held out) |
|---|---:|---:|
| rustc + rmc back end vs LLVM | 0.92 | **0.52** |
| rmc front end + back end vs LLVM | 0.89 | **0.63** |
| rustc + rmc back end vs Cranelift | 1.24 | 0.74 |
| rmc front end + back end vs Cranelift | 1.54 | 0.89 |

The tuned set overstates rmc's code quality: on held-out code the MIR back end
is about half as fast as LLVM and slower than Cranelift. The causes measured
in the generated code are register allocation (five callee-saved registers,
loop-carried values in memory), no `cmov`, no induction-variable strength
reduction, and per-iteration bounds checks.

## 8. Coverage

`tools/mir_survey.py` (121 realistic snippets; rustc accepts 117) and the seven
idiomatic programs:

| pipeline | survey: match / refused / wrong | idiomatic programs |
|---|---|---|
| rmc's own front end | 66 / 51 / 0 | 0 / 7 |
| rustc + rmc back end | see `tools/mir_survey.py` output in CI | 0 / 7 |
| rustc + Cranelift | 117 / 0 / 0 | 7 / 7 |

The MIR route now accepts `black_box`. It still refuses `std::env::args`,
iterating an array (`<[T; N] as IntoIterator>`), every `Vec` (a `Layout`
constant it cannot materialise), non-scalar formatting, closures passed as
values and `dyn Trait`. These share one cause, the text interface that sees
only the local crate's MIR (`ROADMAP.md`, section 2).

## Reproducing

```sh
cargo build --release --manifest-path compiler/Cargo.toml --bins --examples
./backend/build.sh
python3 tests/mir/run.py && python3 tests/std/run.py && python3 tests/cb/run.py
python3 tools/fuzz.py --mode mir --n 300 --seed 1
python3 tools/fuzz.py --mode mir --pipeline cb --n 200 --seed 1
python3 bench/run_four.py --set all
python3 bench/scale.py
```
