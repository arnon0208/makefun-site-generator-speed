# VERIFICATION — site-generator-speed-01 entry

Verified locally on this VM (Node v24.22.1; the entry uses nothing newer
than Node 22). Practice set: the 3 cases in the kit
(`g-0001`, `g-0002`, `g-0003`).

## 1. Correctness — byte-identical output

Method: run the entry and the baseline on the practice set, compare each
case's full answer with a canonical JSON comparison (the same comparison
the scorer does). `work/compare.cjs` in the workspace.

| Case | Markdown files | Templates | Paths in answer | Result |
|------|---------------|-----------|-----------------|--------|
| g-0001 | 250 | 27 | 276 | **PASS — byte-identical** |
| g-0002 | 340 | 22 | 350 | **PASS — byte-identical** |
| g-0003 | 480 | 21 | 509 | **PASS — byte-identical** |

**Result: 3/3 cases byte-identical.**

Differential fuzzing against the reference implementation (in addition to
the practice set):

- **Markdown:** 100,093 random + targeted inputs (star runs of length
  1–6 with/without backslashes, escaped brackets, links, nested emphasis,
  code spans, fences, headings, lists, blockquotes, HR variants, `&<>`
  etc.) — **0 mismatches** vs the reference `renderMarkdown`.
- **Templates:** 3,001 random templates (`{{ }}`, `{{{ }}}`,
  `{% if %}`/`{% if not %}`/`{% else %}`/`{% endif %}`,
  `{% for %}`/`{% endfor %}`, `{% include %}`, standalone-tag lines,
  self-including template) — **0 mismatches** vs the reference
  `renderNodes`, compiled renderer included.
- **Fallback path:** with `--disallow-code-generation-from-strings`
  (which makes `new Function` throw, forcing the tree renderer),
  **3/3 practice cases still byte-identical**.

## 2. Speed — beats the baseline at 95% confidence

Method: the grader's methodology — 1 untimed warmup + 9 timed runs,
trimmed mean dropping the 2 fastest and 2 slowest, CPU time net of a
null-entry control (same driver, `run` returning `{}`), measured with a
preload that reads the child's own `process.cpuUsage()` at exit.
Final paired run, same session, same machine:

| | Trimmed-mean CPU | Net of null control | sd (n=5) |
|---|---|---|---|
| Baseline (makefun-site 1.0.0) | 1258.3 ms | **1066.6 ms** | 55.4 ms |
| This entry | 739.2 ms | **560.4 ms** | 21.7 ms |
| Difference | 519.1 ms | **506.2 ms** | — |
| Speedup | **1.70x** | **1.90x** | — |

95%-confidence assessment: standard error of the difference ≈
√(55.4²/5 + 21.7²/5) ≈ 26.6 ms. The net difference (506.2 ms) is ≈
**19 standard errors** — far beyond the one-sided 95% bar (1.645 SE).
The entry is faster than the baseline at 95% confidence by a wide margin.

Earlier intermediate numbers (same methodology): v1 900.5 ms / 706.0 ms
net; v2 (template compilation) 841.3 / 659.6; v3 (indexOf inline scanner)
786.1 / 602.7; v4 (block char-dispatch) 751.3 / 561.9. The final run
above (739.2 / 560.4) is the shipped code.

In-process phase split (warm, 3 cases): Markdown ~79 ms, layouts ~36 ms,
front matter ~10 ms, template parse + compile ~1.4 ms, everything else
~5 ms. The rest of the process time is node startup, JSON parse of the
cases, and JSON serialisation of the answers — costs the baseline pays
too.

## 3. Constraints check

- Node 22 compatible: yes (no API newer than Node 22).
- JS/TS source only; no WASM, native addon, or prebuilt binary: yes.
- Nothing cached between cases: yes — `run` builds a fresh pipeline per call.
- No network calls from the entry: yes.
- Repo size: ~37 KB (`solve.js` 35,643 B + `main.ts` 1,397 B +
  `package.json` 23 B + docs), under the 4 MiB limit.
- `main.ts` logic is identical to the baseline driver (only the header
  comment differs); it keeps the baseline's
  `--no-incremental-marking` V8 flag, which the rules allow.

## 4. What still blocks submission

Nothing in the entry itself. Submission (repo creation + upload to
make.fun) is the parent's step — this task was build + verify only.
Deadline: **25 Nov 2026, 03:00 WIB** (2026-11-24T20:00:00Z). At sweep
time the competition had 0 submissions; the final ranking set is drawn
from drand round 34249212, so the practice-set numbers above are the
best available proxy — the entry's margin (1.90x net) leaves room for
set-to-set variation.
