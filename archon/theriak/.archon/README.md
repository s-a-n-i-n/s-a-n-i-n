# Archon for Theriak — test drive

Three workflows and one command, written against Archon v0.11.1 and validated
with `archon validate workflows` on 2026-10-02. They encode what CLAUDE.md
already demands; they do not replace it. Claude Code still reads CLAUDE.md in
every AI node.

| File | What it is | Runs where |
|---|---|---|
| `workflows/theriak-fix-issue.yaml` | reproduce → implement with guard → mechanical RED proof → suites for touched areas → commit (no attribution) → rule-0 handover comment. **Never pushes.** | Archon worktree |
| `workflows/theriak-ship.yaml` | the "push it" macro: prod-idle + trailer gate → one approval → push master → watch Deploy → served-sha check → optional reingest with second approval | live checkout (`worktree.enabled: false`) |
| `workflows/theriak-delegate.yaml` | brief → cheap worker (budget cap, no web, no `.env`) → tests from the brief → opus reviewer with red-proof → ACCEPT/REJECT | Archon worktree |
| `commands/eval-spend-gate.md` | the "never fire the paid judge without an explicit yes" rule as a step | — |

## Setup on the Windows box (once)

```powershell
irm https://archon.diy/install.ps1 | iex
archon --version            # expect 0.11.x
```

`~/.archon/config.yaml` (create it):

```yaml
assistants:
  claude:
    claudeBinaryPath: C:\Users\<you>\AppData\Roaming\npm\claude.cmd   # `where claude`
telemetry:
  enabled: false
```

Then, in `C:\Code\Theriak` on this branch:

```powershell
archon validate workflows        # all three should read "ok"
archon workflow list             # theriak-* visible, bundled ones hidden (config.yaml)
```

## First run

Pick a pure rag-service bug with no prompt/rubric change so the paid-run
gate stays out of the way. `#300` ("check_citation_correct_or_hedged awards a
PASS for citing nothing at all") is one function and one test — ideal.

```powershell
archon workflow run theriak-fix-issue "#300"
```

What you will see, in order:

1. `preflight` prints the worktree path, the main checkout it found, and the
   python it will use. If it says "no rag-service venv found" the workflow stops
   here — nothing was spent.
2. `frame` reproduces the symptom and writes `frame.md`. If it did not
   reproduce, the run posts one comment on the issue and ends.
3. `implement` is a loop of up to 3: code + guard, then the `red-proof` script
   applies `bug.patch`, expects the guard to FAIL, reverses it. A loop iteration
   that does not reach `RED-PROOF OK` feeds its failure line to the next one.
4. `suites` runs only the areas the diff touched (rag-service / api-gateway /
   frontend / doc-data guards).
5. `commit` refuses any attribution text, stages only the changed files,
   appends the red-proof line to the message.
6. `handover` posts the rule-0 summary on the issue. **Nothing is pushed.**

Artifacts of the run: `~/.archon/workspaces/s-a-n-i-n/Theriak/artifacts/<run>/`
— `frame.md`, `red-proof.txt`, `guard-red.log`, `commit-final.txt`, `summary.md`.
Logs: `~/.archon/workspaces/s-a-n-i-n/Theriak/logs/`.

Then, if you like what you see, from the main checkout:

```powershell
git log --oneline -3 <worktree-branch>   # `archon isolation list` names it
git cherry-pick <sha>                    # or merge
archon workflow run theriak-ship ""      # the push gate; "reingest" instead of "" when the corpus changed
```

## Things to expect to go wrong on the first run

- **Git Bash vs PowerShell.** `bash:` nodes run in Git Bash. `dotnet`, `gh`,
  `npm` must be on that PATH. CRLF: the scripts here use `\n`; if Git converts
  them on checkout, `git config core.autocrlf false` for this repo.
- **The worktree has no `.venv` and no `node_modules`.** `preflight` resolves
  python from the main checkout; `suites` symlinks `node_modules` from the main
  checkout (needs Developer Mode or admin for symlinks on Windows; falls back
  to `npm ci`).
- **Approval gates.** Only `needs-eval-approval` can pause `theriak-fix-issue`,
  and only when the fix touches a prompt/rubric. The CLI pauses the run; approve
  it with `archon workflow resume <run-id>` or in `archon serve`.
- **Model names.** `opus`/`sonnet`/`haiku` are Archon aliases. If your Claude
  Code is routed to GLM via `ANTHROPIC_BASE_URL`, Archon inherits that env —
  check the first node's transcript to see which model actually answered.
- **The bug.patch contract.** The red-proof step is a script and cannot ask
  questions. If the agent writes a patch that also reverts the guard, the proof
  fails with "guard is not green WITH the fix" — that is the loop doing its job.
