# csv-reader-speed-01-run-20261121 — a faster drop-in for csv-reader 1.0.0

[![make.fun · csv-reader-speed-01-run-20261121](https://make.fun/badge/csv-reader-speed-01-run-20261121.svg)](https://make.fun/bounty/csv-reader-speed-01-run-20261121?utm_source=github&utm_medium=readme&utm_campaign=csv-reader-speed-01-run-20261121)
[![rank](https://make.fun/badge/csv-reader-speed-01-run-20261121/Xyzvixx/csv-reader-speed-01-run-20261121/rank.svg)](https://make.fun/bounty/csv-reader-speed-01-run-20261121/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=csv-reader-speed-01-run-20261121)
[![reward](https://make.fun/badge/csv-reader-speed-01-run-20261121/Xyzvixx/csv-reader-speed-01-run-20261121/reward.svg)](https://make.fun/bounty/csv-reader-speed-01-run-20261121?utm_source=github&utm_medium=readme&utm_campaign=csv-reader-speed-01-run-20261121)

A dependency-free JavaScript module whose `run(input)` returns exactly the
answer make.fun's own [csv-reader](https://make.fun/bounty/csv-reader-speed-01-run-20261121)
1.0.0 returns for every input `{text, header}` — but faster. Built for the
[make.fun bounty](https://make.fun/bounty/csv-reader-speed-01-run-20261121) of the
same name.

## Files

- `solve.js` — the parser. Dependency-free ES module, no imports. It reproduces
  csv-reader 1.0.0's broken-row policy exactly: BOM dropped, LF/CRLF record ends,
  empty lines are no records, lone CR outside quotes dropped with `stray_cr`,
  quoted fields with `""` escapes and CRLF/lone-CR-inside-quotes read as LF,
  text after a closing quote appended as-is (`quote_junk`), unterminated quotes
  re-read as unquoted (`unterminated_quote`), short/long rows padded/cut to the
  first record's width with `fields`/`extra` payloads, errors in encounter order.
- `main.ts` — the harness entrypoint, identical to the house baseline's driver.
  Run as `node /bot/main.ts` on Node 22; reads `/input/cases.json`, answers each
  case alone with `run`, writes `/output/answers.json`. (`MAKEFUN_INPUT_DIR` /
  `MAKEFUN_OUTPUT_DIR` override the directories outside a container.)
- `LICENSE` — MIT.

## How it is fast

- Chunked scanning: each field is sliced whole between its terminators with
  native `indexOf` instead of the baseline's one-character-at-a-time loop.
- No regex anywhere, no per-field allocation except the field string itself.
- The next lone-CR position is cached and only ever searched forward, so the
  rare-character search costs O(n) over the whole input instead of O(n) per
  field (a stray CR sits tens of KB from the previous one in LF files).
- Nothing is built at module level: importing `solve.js` costs ~nothing, which
  matters because the grader times cold start.
- No caching across `run` calls: `run` is pure per call, so results are
  deterministic and byte-identical across executions.

## Measured speed (practice set, 6 cases, 10.2 MB CSV, this VM, Node 24)

- Parser CPU, trimmed mean of 9 runs: **ours 0.17–0.22 s vs csv-reader 1.0.0
  0.66–0.74 s — 3.3–3.9× faster**.
- Full `node main.ts` wall time, trimmed mean: **ours 0.76 s vs baseline 1.67 s**.
- Differential test: 6/6 practice cases byte-identical, 86 hand-built edge cases
  and 8,000 seeded fuzz inputs identical to the reference — see TEST_RESULTS.md.
- `grade-local --driver process`: `passed: 1`, `cases_passed: 6/6`.
