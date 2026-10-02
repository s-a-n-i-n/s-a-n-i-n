# Handover — Archon test drive on Theriak (2026-10-02, ~21:30 CEST)

For a local Claude Code session in `C:\Code\Theriak`. Written by the cloud
session that set this up. Read it once; everything it refers to is on disk.

## What this is for

Trying Archon (https://github.com/coleam00/archon, v0.11.1) as a fixed, repeatable
pipeline for the Theriak "fix one issue" loop: reproduce → implement with a guard →
mechanical red-proof → suites → commit without attribution → handover comment.
Never pushes. The point is that the three gates that matter (reproduce, red-proof,
no attribution) are scripts, not promises from the model.

## State right now

| Thing | State |
|---|---|
| Archon | installed, `C:\Users\sanin\.archon\bin\archon.exe`, v0.11.1. Not on PATH in old terminals: `$env:PATH = "C:\Users\sanin\.archon\bin;$env:PATH"` |
| Claude binary for Archon | native `C:\Users\sanin\.local\bin\claude.exe` (2.1.287), installed via `claude install`. The npm `claude.cmd` cannot be spawned by Archon (Node EINVAL). |
| `~\.archon\config.yaml` | `assistants.claude.claudeBinaryPath` = the .exe above; `telemetry.enabled: false`. Also set `$env:DO_NOT_TRACK=1` per shell. |
| Git Bash first on PATH | required per shell: `$env:PATH = "C:\Program Files\Git\bin;$env:PATH"`. WSL's `System32\bash.exe` otherwise wins and breaks every `bash:` node. |
| `core.longpaths` | set globally. |
| **Windows LongPathsEnabled** | **NOT YET DONE — blocks every pytest run inside an Archon worktree.** Admin PowerShell: `New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name LongPathsEnabled -Value 1 -PropertyType DWORD -Force` |
| Repo branch | `archon/test-drive` in `C:\Code\Theriak`, branched from `fix/mermaid-answer-path` (6 unpushed docs commits over master). Working tree was already dirty before this started (~112 status lines); none of that is ours. |
| `.archon/` in the repo | **untracked, uncommitted**: `config.yaml`, `README.md`, `commands/eval-spend-gate.md`, `workflows/theriak-{fix-issue,ship,delegate}.yaml`. Plus one appended line in `.gitattributes`: `.archon/** text eol=lf`. All LF. Commit them on `archon/test-drive` if you want to keep them. |
| Canonical copy | GitHub `s-a-n-i-n/s-a-n-i-n`, branch `dev/compassionate-wozniak-0v8ib7`, folder `archon/theriak/.archon/` (commit `e4467da`). It carries the latest `suites` fix; the local copy may still have the older line — see step 1 below. |
| Run 1: #271 | completed in 4 min. Branch `not-reproduced` taken correctly: #271 was already fixed on master (572af6ab, 7bacb91c). Posted one comment: https://github.com/s-a-n-i-n/Theriak/issues/271#issuecomment-5958331516. Close #271. Its side finding — `eval_compare` prints "REGRESSIONS PRESENT" when only DID NOT RUN / MISSING block — deserves its own small issue. |
| Run 2: #362 | run id `abfce71f-8410-4ec9-b885-8c419ac9b1b6`, **FAILED at `suites`** after 1 h 12 min. `frame` reproduced (scorer-only change moves held-out verdicts, existing guard stays green). **`implement` reached RED-PROOF OK** — the guard exists and was proven to go red on a simulated ruler move. The fix is **uncommitted** in worktree `C:\Users\sanin\.archon\workspaces\s-a-n-i-n\Theriak\worktrees\archon\task-theriak-fix-issue-1790964400946`, branch `archon/task-theriak-fix-issue-1790964400946`. |
| Run 2 artifacts | `C:\Users\sanin\.archon\workspaces\s-a-n-i-n\Theriak\artifacts\<run>\`: `frame.md`, `repro/repro_362.py`, `repro/eval_scoring.py.orig`, `guard-cmd.txt`, `bug.patch`, `commit-msg.txt`, `consumers.md`, `red-proof.txt`, `guard-red.log`. Transcript: `...\logs\abfce71f-...jsonl`. Web UI: `archon serve`. |

## Why `suites` failed (two causes, both fixed or fixable)

1. `$BASE_BRANCH` in a `bash:` node is the branch the MAIN checkout is on
   (`archon/test-drive`), not `worktree.baseBranch`. The old `suites` line did
   `git diff origin/$BASE_BRANCH...HEAD` → "unknown revision". Fixed in the
   canonical copy: `CHANGED="$(git diff --name-only HEAD; git ls-files --others --exclude-standard)"`.
2. pytest collection died on `data/preprocessed/gkv/AM_7_Ergaenzungs...md`:
   Archon's worktree prefix (~101 chars) + that 135-char filename ≈ 260. Git
   created the file (`core.longpaths`), Python cannot open it until Windows
   `LongPathsEnabled=1`. Main checkout is unaffected because `C:\Code\Theriak` is short.

## Next steps, in order

1. Make the local `suites` node match the canonical one. Either copy
   `archon/theriak/.archon/workflows/theriak-fix-issue.yaml` from the
   `s-a-n-i-n/s-a-n-i-n` branch above, or check `Select-String -Path .archon\workflows\theriak-fix-issue.yaml -Pattern 'BASE_BRANCH'` returns nothing.
   Then `archon validate workflows` → 3 ok.
2. Admin PowerShell: the `LongPathsEnabled` line above. No reboot.
3. Prove Python can open the long file from the worktree:
   `& rag-service\.venv\Scripts\python.exe -c "from pathlib import Path; p=next(Path(r'C:\Users\sanin\.archon\workspaces\s-a-n-i-n\Theriak\worktrees\archon').glob('task-theriak-fix-issue-*/data/preprocessed/gkv/AM_7_*')); print(len(str(p)), p.read_text(encoding='utf-8')[:40])"`
4. `archon workflow run theriak-fix-issue --resume` — `frame` and `implement` are
   cached, only `suites` → `commit` → `handover` re-run, in the same worktree.
   Unknown whether resume uses the frozen YAML or the patched one; if `suites`
   fails again on `origin/archon/test-drive`, it used the frozen one — then a
   fresh run (`archon workflow run theriak-fix-issue "#362 ..."` with the same
   argument text as before, see `$env:TEMP\archon-362.log` line 1) costs ~1 h.
5. When green: read `red-proof.txt`, `commit-final.txt`, `summary.md`; look at
   `git show --stat HEAD` in the worktree; decide. Cherry-pick onto a branch of
   your choosing. **Do not run `theriak-ship` from the dirty main checkout** — it
   gates on upstream drift and prod idleness but NOT on uncommitted changes.
6. Clean up: `archon complete archon/task-theriak-fix-issue-1790964400946`
   (removes worktree + local branch; it also tries the remote branch, which never
   existed — harmless). `archon isolation list` to confirm.

## Known defects in the workflows (fix before relying on them)

- `frame` can write to production files (it made `eval_scoring.py.orig`, i.e. it
  edited the scorer to simulate the ruler move). It appears to have restored it,
  but nothing enforces that. Add to the `frame` prompt: "restore every file you
  touched; `git status --porcelain` must be empty before you return", and consider
  a bash node after `frame` that fails on a dirty tree.
- Model aliases: `opus` resolved to `claude-opus-5-5[1m]`, `sonnet` to
  `claude-sonnet-5-5` with a "resolved_model_ambiguous" warning each time. Pin full
  IDs (`model: claude-opus-5-5`) to remove the dependency on list order.
- `theriak-ship` has no "working tree clean" gate. Add
  `[ -z "$(git status --porcelain)" ] || { echo dirty; exit 1; }` to its `gate` node.
- `commands/eval-spend-gate.md` is not referenced by any workflow yet.
- `theriak-delegate` has `model: sonnet` as the worker placeholder; GLM/DeepSeek
  routing through Archon is untested (Archon providers: claude, codex, pi, copilot).
- VIMS drafts (`archon/vims/.archon/`) were never validated.
- `--verbose` floods the terminal with `claude.system_message_unhandled` debug
  lines (thinking tokens Archon's formatter doesn't render). Harmless; don't use it.

## Picking the next issue — read the comments, not just the body

Dead candidates (already fixed per comments): #271, #264, #382. #300 is a
rubric-shape change that must be measured against the whole bank — it trips the
eval-approval gate and needs a paid run; not a test-drive issue. #362 is the live one.

## Rules this setup must keep honouring

- No `Co-Authored-By` or any AI attribution in Theriak commits, PRs or comments.
  The `commit` node greps for it and refuses. Keep that.
- No paid run without an explicit yes and a ledger row first. Nothing in the
  fix-issue workflow spends money; `frame` was told so explicitly.
- Never push from a workflow. `theriak-ship` is the only path and it has an
  approval node.

## Unrelated but found during the session review

Plaintext YouTrack permanent token in VIMS session scratch scripts; test-account
passwords echoed in shell calls; six SQL logins + the dev JWT key still in
`.pubxml` history on GitHub (your own 2026-09-13 session noted it). Rotate before
wiring any automation that reads them.
