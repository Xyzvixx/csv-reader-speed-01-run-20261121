# TEST_RESULTS — csv-reader-speed-01-run-20261121 (solve.js v1)

Date: 2026-11-21 (WIB). Runner: Muse Spark (coordinator) + Node 24.22.1 on this VM.
SolAPI gpt-6.1-sol was UNAVAILABLE at build time (401 USER_INACTIVE, 11:41 WIB;
AstraAPI same 401 — provider-side account problem, needs Ridho); critique pass
fell back to GeminiAPI/100milionseek — see test/critique-v1.txt.

## 1. Differential test vs csv-reader 1.0.0 (`test/diff.mjs`)
- Practice set: **6/6 cases byte-identical** (`JSON.stringify` of `{columns, rows, errors}`).
- Hand-built edge corpus: **86/86 identical** (BOM-only, empty text, no trailing EOL,
  CRLF-only, lone-CR-only lines, `""` / `""""`, junk variants incl. `x"y` and junk of
  only CRs, unterminated quote at EOF/mid-file/followed by commas, quote_junk +
  stray_cr ordering, short/long rows incl. header-width edge, multiline quoted with
  CRLF + lone CR, blank lines LF/CRLF/mixed, header true + zero records).
- Seeded fuzz: **8,000/8,000 identical** (5,000 raw random strings + 3,000 structured
  random records, seed 20261121); second seed 777001 (longer strings, 3,000 raw +
  2,000 structured): **4,092/4,092 identical**.
- **TOTAL: 12,184 checked, 0 failed.**
- One real bug found and fixed by the fuzzer: the cached next-CR position went stale
  when the unterminated-quote path moves `pos` backward — the re-read then missed
  stray CRs for the rest of the file (429 fuzz mismatches). Fix: rescan for `\r`
  from the reset position in that path; re-ran to 0 failures.

## 2. Speed (practice set, 6 cases, 10,193,953 chars)
Parser CPU in-process (`test/bench-parse.mjs`, `process.cpuUsage`, trimmed mean
of 9 runs after warm-up):
- Run A: ours **0.224 s** vs baseline **0.744 s** → **3.31×**
- Run B: ours **0.167 s** vs baseline **0.655 s** → **3.92×**
- JSON.parse of cases.json costs 0.051 s (netted out by the grader's do-nothing control).
Full process (`test/bench.mjs`, 1 warm-up + 9 timed, trimmed mean):
- Wall: ours **0.759 s** vs baseline **1.672 s** (2.2×)
- CPU readings on this shared VM were too noisy to quote (1.1–2.1 s range for the
  baseline across runs); wall + in-process CPU agree on a >3× parser margin.
House reference numbers for context (RULES.txt): papaparse 0.78 s, d3-dsv 0.73 s,
csv-parser 0.72 s vs csv-reader 1.38 s CPU on 10.2–11.8 MB sets — all three miss
the semantic gate; ours is faster than all three AND semantically identical.

## 3. grade-local (kit check, `--driver process`)
- `node cli/makefun.mjs grade-local --kit . --bot bot --driver process`
- `scores {"cases_passed":6,"cases_total":6,"passed":1}`, checks 6/6,
  `eligible ranked yes, prize yes`, result_sha256
  `abee77a8c3e1b27318986d3495a7178cc386f5ece47dfed373e7c36fe752df23`
  (L1 regrade reproduced byte for byte; node version differs from the server's
  v26.13.1, as expected locally).

## 4. Clean-room run (zero-fail-qa)
- Fresh copy of `bot/` in an empty dir, no kit files, `node main.ts` with
  MAKEFUN_INPUT_DIR/OUTPUT_DIR on the practice set: answers byte-identical to
  `practice/reference/expected.json` (6/6). No dependencies, no hardcoded paths,
  no network, no secrets in the repo (scan below).

## 5. Secret scan (pre-push)
- `grep -rE` for private-key / API-key / token / seed patterns over `bot/`:
  0 hits for any value. Only the agent wallet appears, in `.makefun/owner.json`
  (required, NOT the Primary wallet). The Primary wallet address appears NOWHERE
  in the repo (verified by grep for its prefix).
- `.gitignore` covers .env / *.key / *.pem / *secret* / credentials / keypair files.
- Repo size 14,822 bytes (limit 1,048,576).
