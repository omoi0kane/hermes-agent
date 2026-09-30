# Skill guard delta: agent-review entry point

## Contract

With `skills.guard_agent_created=true`, an existing critical finding in an unchanged reference must not block a safe patch to SKILL.md. Newly introduced critical findings must still be rejected, restoring original bytes. `baseline` expects the observed buggy behavior; `fixed` expects the proposed corrected behavior. A PASS for baseline proves reproduction, not correctness.

## Prerequisites and isolation

Use a **disposable source checkout**, not the running Hermes checkout. Select a Python interpreter with the checkout's Hermes dependencies installed using CONTRIBUTING.md. No pytest, model/API credentials, private skills, provider configuration or LLM call is needed for this standalone probe. The script uses only stdlib directly, but imports Hermes modules and therefore needs Hermes dependencies.

The script starts a separate process with temporary HOME/HERMES_HOME/HERMES_RUNTIME_DIR, synthetic config and skills, a credential-free environment and bytecode writing disabled. It never executes the documentation's curl text. No scanner/config/approval mocks are used. Only fixture seeding writes directly; the updates being tested go through real `skill_manage`, in single and batch forms. The reference includes `example.invalid` and the literal variable name `$API_TOKEN`, not any secret value.

## Run on the PR checkout

From its repository root, with the dependency interpreter activated:

```sh
python scripts/repro_skill_guard_delta.py --expect fixed
```

Alternatively replace `python` with the dependency interpreter's absolute path. It emits JSON plus:

```text
PASS: expectation=fixed; cases=4; failures=0
```

Exit 0 means all expected relations passed; 1 means a behavior mismatch. Setup/import/timeout errors are nonzero and must not be interpreted as a successful rejection test. Assertions use explicit checks, not Python `assert`, so `python -O` does not remove the checks.

## Reproduce on the independently verified upstream baseline

Keep the **same probe from the PR** and point it at the baseline source. Example POSIX commands from the disposable PR checkout (the interpreter must already be selected):

```sh
REPRO="$PWD/scripts/repro_skill_guard_delta.py"
git fetch https://github.com/NousResearch/hermes-agent.git 6ec05205a943cf813bd56c3c79d64bcf922dac67
git worktree add --detach ../skill-guard-baseline-116091 6ec05205a943cf813bd56c3c79d64bcf922dac67
python "$REPRO" --repo ../skill-guard-baseline-116091 --expect baseline
```

Expected summary: `PASS: expectation=baseline; cases=4; failures=0`, exit 0. An existing destination directory is a setup conflict; choose another disposable path. The probe does not fetch anything itself. The worktree setup above requires GitHub network access; execution of the probe does not require network.

Negative controls (verify the checker is discriminating):

```sh
python "$REPRO" --repo ../skill-guard-baseline-116091 --expect fixed
python "$REPRO" --expect baseline
```

Both must report FAIL and exit 1. Execute individually if your shell stops on the first nonzero status. Baseline worktree cleanup is optional and should follow the operator's normal disposable-checkout policy; this probe does not delete repository worktrees.

## Four tested cases

- `clean-control`: harmless edit succeeds in both modes and changes the expected marker.
- `legacy-single`: unchanged reference has critical finding(s); harmless single patch is rejected/restored in baseline and succeeds in fixed.
- `legacy-batch`: same contract through the atomic batch entry point.
- `new-critical-control`: introducing critical content into a clean skill is rejected in both modes, with original bytes restored and no remaining critical after rollback.

Legacy reference bytes and before/after critical counts are checked. Failed edits must specifically report `Security scan blocked`; unrelated setup or read-before-write errors do not satisfy rejection expectations.

## Observed evidence and limits

On macOS 26.5.2 / Python 3.11.15 with an existing Hermes dependency environment:

- Upstream baseline `6ec05205a943cf813bd56c3c79d64bcf922dac67`: baseline PASS/exit 0; fixed FAIL/exit 1.
- Original PR implementation `3205273e339d9d174c142e97d6075f396a44b1a5`: fixed PASS/exit 0; baseline FAIL/exit 1.

The probe also passed with `python -O --expect fixed` on the reproduction-enhanced PR working tree. See PR body for fresh verification receipts.

This is tool/config/filesystem integration evidence, not a full model/Discord/TUI end-to-end test. It does not establish all security invariants, other OS behavior, or latest-main merge compatibility. The original PR implementation was tested independently, not merged into latest main. Do not infer that existing critical content is safe: the proposed policy only distinguishes legacy findings from findings introduced by the current write. Hard-denial preservation and further move/duplicate/scan-failure coverage remain separate review concerns.

The historical unit-suite result in the original PR body is historical evidence; this update does not claim that suite was freshly rerun. The reproduction environment did not have pytest or ruff installed.
