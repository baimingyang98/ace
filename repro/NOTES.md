# Reproduction notes: ACE on Formula with DeepSeek

Working notes for the FTEC5660 reproducibility report. Target: Table 2 of the paper
(arXiv 2510.04618v3), DeepSeek-V3.1 as the base LLM, Formula test split (200 questions).

| Paper (Table 2) | Formula accuracy |
|---|---|
| Base LLM | 67.5 |
| ACE, offline, with GT labels | 85.5 |
| ACE, online, with GT labels | 76.5 |
| ACE, online, without GT labels | 78.5 |

## Controlled variable: the model

- Paper: DeepSeek-V3.1 in non-thinking mode (section 4.2), served by SambaNova (the repo's default provider).
- Here: the official DeepSeek API, `https://api.deepseek.com`. On 2026-10-24 it serves `deepseek-flash`
  (DeepSeek-V4.1-Flash-1017) and `deepseek-v4-pro` (DeepSeek-V4-Pro-0813). The old name `deepseek-v4-flash`
  is still accepted but its requests are served by V4.1-Flash. DeepSeek-V3.1 is not offered.
- Both current models think by default, and in thinking mode they ignore temperature. `llm.py` now sends
  `thinking: disabled` for the deepseek provider, to match the paper. `DEEPSEEK_THINKING=enabled` turns it on.
- Results are not directly comparable with the paper because the underlying LLM differs.

## Differences between the paper and the repo defaults

1. Reflector rounds: the paper uses at most 5 (section 4.2); `eval/finance/run.py` defaults to 3
   (`--max_num_rounds`). The notebook passes 5.
2. Offline epochs: the paper uses at most 5; the repo defaults to 1 (`--num_epochs`).
3. Online update granularity: the paper says the model predicts on each sample and then updates the context
   with that sample. The repo tests a window of 15 samples with the same playbook, then trains on the window
   (`--online_eval_frequency 15`, `ace/ace.py` around line 933).
4. Test order: the paper uses "the same shuffled test split" for all methods. The repo reads the test file in
   file order and does not shuffle.
5. Code version: the repo head (82709de, 2026-08-24) adds batched ACE (ComBEE, May and August 2026), after
   the paper. `--batch_size 1`, the default, keeps the original sequential path.

## Things that can hide errors

- An empty or `None` response from the API is not retried: test and training calls are scored as wrong
  answers (`llm.py` around lines 205 and 230). The notebook counts these per run (`failed_or_empty`).
- Temperature is fixed at 0.0 in `llm.py` (line 71) for every role; there is no flag for it.

## Debug diary

- 2026-10-24: DeepSeek is not a provider in the repo; added it the same way upstream added commonstack (#33).
- 2026-10-24: Run 1 with `deepseek-v4-pro` stopped after 34 min, at online sample 101 of 200, with
  HTTP 402 "Insufficient Balance". `llm.py` retries only timeouts, rate limits and 5xx, so the error ends
  the process. Online mode has no resume, so the run has to start again from sample 1. The base accuracy
  (the initial test on all 200 questions) was already saved in `initial_test_results.json`.
  Notebook changes: a failed run now stops the loop instead of starting the next run, `summarize` reads
  crashed runs, and re-running the setup cell no longer deletes `results/`.
