# Assessment — delivery

`legacy/delivery` -> `/Users/huynhluong/Desktop/delivery/skills-hello-github-actions`. This system is very small, so I read it directly and did not launch the three subagents. Tools used: `scc`, `grep`.

## Executive Summary
"Hello GitHub Actions" is a GitHub Skills self-paced exercise. It is a chain of six GitHub Actions workflows that teach a learner to write a workflow. Each step checks the learner's `welcome.yml`, posts feedback as an issue comment, then switches on the next step. It is tiny (744 code lines, 14 files) and low risk. It has no application code, data store or tests. The one real dependency is the external `skills/*` actions and reusable workflows. Headline: there is nothing to rewrite, so the analysis is best used to document how the exercise behaves today.

## System Inventory
| Language | Files | Code lines |
|---|---|---|
| YAML (workflows) | 6 | 594 |
| Markdown (steps, README) | 7 | 146 |
| License | 1 | 4 |
| Total | 14 | 744 |

Plus one image (`.github/images/create-branch-screenshot.png`). Complexity per scc is 0 (no code). Build system: none. Data stores: none. Tests: none. Git history: 2 commits.

Technology fingerprint: GitHub-hosted runner `ubuntu-latest`, `actions/checkout@v4`, `gh` CLI with `GITHUB_TOKEN`, `raven-actions/actionlint@v2.1.1` (listed in the workflow files; not in the trigger chain above).

## Architecture at a Glance
Diagram: `ARCHITECTURE.mmd`.

| Domain | Files | Role |
|---|---|---|
| Exercise start | `workflows/0-start-exercise.yml` | On push to `main`, opens the exercise issue (via `start-exercise.yml@v0.8.1`), posts step 1 text, enables "Step 1" |
| Step checkers (1–3) | `workflows/1-step.yml`, `2-step.yml`, `3-step.yml` | On push to `welcome-workflow`, check `.github/workflows/welcome.yml` for required keyphrases, post a results table, then disable themselves and enable the next step |
| Step checkers (4–5) | `workflows/4-step.yml`, `5-step.yml` | 4: on PR opened to `main`, post step 5. 5: on PR closed to `main`, post the review, finish the exercise, disable itself |
| Learner content | `steps/1..5-step.md`, `steps/x-review.md`, `README.md` | Instruction text posted to the issue, filled in with `{login}` and `{full_repo_name}` |
| External actions | outside the repo | `skills/exercise-toolkit`, `action-keyphrase-checker`, `action-text-variables` |

Dependencies: each checker workflow uses its step file and the toolkit's markdown templates. The shared state is the exercise issue (found with `find-exercise-issue.yml`) and the enabled/disabled flag of each workflow.

## Production Runtime Profile
No telemetry available.

## Technical Debt (top items)
| # | Item | Where |
|---|---|---|
| 1 | Heavy duplication: the find-exercise / checkout / build-comment / `gh issue comment` / disable-and-enable sequence repeats in every step workflow | `workflows/1-step.yml` through `5-step.yml` |
| 2 | Two toolkit versions in use (`@v0.8.1` for start, `@v0.6.0` for the rest) | `0-start-exercise.yml:20`, others |
| 3 | Third-party actions pinned to tags, not commit SHAs | all workflows |
| 4 | Step order depends on workflow names (`gh workflow enable "Step 2"`): renaming one breaks the chain | `1-step.yml:172`, `2-step.yml:150`, `3-step.yml:173`, `4-step.yml:57` |
| 5 | Checks are plain text matches (`keyphrase: "name:"`), so a comment or wrong indentation also passes | `1-step.yml`, `2-step.yml`, `3-step.yml` |
| 6 | Hardcoded repo-specific strings in the image URL and the README link (`huyluongdev/...`) | `steps/1-step.md`, `README.md` |
| 7 | Steps 1–3 trigger on the same branch push, so unrelated pushes can start checkers (guarded only by enabled/disabled state and `paths`) | `1-step.yml:3`, `2-step.yml:3` |
| 8 | No tests or lint for the workflows beyond the actionlint action | whole repo |

## Security Findings
| Severity | CWE | Finding | Where |
|---|---|---|---|
| Low | CWE-1357 (unpinned third-party components) | `skills/*` and `raven-actions/actionlint` are used by tag, not by SHA | all workflows |
| Low | CWE-250 | Broad token scopes: `contents: write` in steps 0, 2, 4 and 5 where only issues/actions look needed | `0-start-exercise.yml:9`, `2-step.yml:9`, `4-step.yml:9`, `5-step.yml:9` |
| Info | CWE-78 | Issue text is passed through env vars (`ISSUE_BODY`), not interpolated into the shell: this is the safe pattern | `0-start-exercise.yml` |

No credentials found, and no SECRETS.local.md was needed. Only `secrets.GITHUB_TOKEN` is referenced. Not a full audit (the external actions were not read).

## Documentation Gaps
1. How the six workflows chain together (the enable/disable hand-off).
2. What each step's automatic check looks for (the keyphrases) and what counts as pass.
3. The exercise issue as shared state, and how it is found.
4. What the external `skills/exercise-toolkit` templates contain (not in this repo).
5. Required repository settings (Actions on, workflows enabled, branch names `main` and `welcome-workflow`).

## Relative Scale
Index `2.94 × KSLOC^1.10` = about 2.1 for 0.744 KSLOC. This is a relative size for ranking, not a timeline or a cost.

## Recommended Modernization Pattern
**Rehost (no code change).** This is a learning template with no business logic or runtime of its own, and its behaviour is defined by GitHub and the `skills/*` actions. No build command applies. The analysis is still useful: `map` shows the step chain, `extract-rules` records what each step checks, and `harden` can cover the pinning and token-scope findings.
