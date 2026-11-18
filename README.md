# Forge: specification

Requirements are written in EARS. The system name is **Forge** throughout.

## 1. Scope and terms

Forge is the set of Claude Code loops that build, verify, research, prototype, review, report
on, and release the app in this repository.

| Term | Meaning |
| --- | --- |
| Product owner | The person who owns the app. The only one who starts a release or a loop session. |
| Loop | A Claude Code session that runs one loop skill on a cron schedule (`/loop <interval> /goal …`). |
| Loop skill | `.claude/skills/forge-<name>/SKILL.md`: the prompt one loop runs. |
| Run | One firing of a loop's cron, or one manual start of its skill. |
| Mailbox | `.forge/mailbox/`: OKF bundles `issues`, `inbox`, `backlog`, `research`, `retro`. |
| Issue | One OKF markdown file in `.forge/mailbox/issues` with `status: open` or `done`. |
| Decision issue | An issue that needs the product owner (`severity: decision` or `needs: product-owner decision`). |
| Worktree | A git worktree under `.forge/worktree/<name>` on its own branch. |
| Loop Station | The dashboard https://claude.ai/artifact/ELxJW2YUQ5UEmmeimCkMj8 and its collector `.forge/station/collect.py`. |
| Loop registry | `.forge/loops.json`: one row per pane, the loop it runs, and its status. |
| Alpha | The staging slot (wrangler env `staging`). |
| Outcome | One objective of `OUTCOMES.md` (O1 to O4); its key results are its measure. |
| Attempt | One forge-fix run on an issue; the issue's `attempts` field counts them. |

## 2. Loops

| Loop | Skill | Schedule (local time) | Purpose |
| --- | --- | --- | --- |
| forge-fix | `/forge-fix` | `7,37 * * * *` | Works one issue to verified done |
| forge-verify | `/forge-verify` | `7 * * * *` | Re-verifies every BDR, files issues |
| forge-retro | `/forge-retro` | `13 6 * * *` | Writes a daily Keep / Stop / Start retrospective |
| forge-research | `/forge-research` | `13 */2 * * *` | Answers research requests, adds research concepts |
| forge-prototype | `/forge-prototype` | `13 */2 * * *` | Builds Storybook prototypes of backlog items |
| forge-station | `/forge-station` | `13 * * * *` | Updates the Loop Station dashboard |
| forge-release | `/forge-release` | `23 */6 * * *` | Runs `/release alpha` |

`/forge-up` (section 13) is a setup skill, not a loop: the product owner runs it once, and it schedules no cron.

## 3. Requirements for all loops

- **F-1** The Forge shall run each loop in its own Claude Code session with the repository root as its working directory.
- **F-2** The Forge shall store each loop's prompt in `.claude/skills/forge-<name>/SKILL.md`.
- **F-3** When the product owner runs a loop skill, the Forge shall schedule that loop's cron in the same session.
- **F-4** When a loop's cron fires, the Forge shall run the loop skill's `/goal` steps once.
- **F-5** When a loop session has run for 7 days, the Forge shall stop that loop's cron.
- **F-6** The Forge shall make each code or document change in a worktree under `.forge/worktree/`.
- **F-7** When a loop finishes a change, the Forge shall fast-forward `main` to the worktree's branch.
- **F-8** When a loop has merged its worktree, the Forge shall delete that worktree and its branch.
- **F-9** If `make check` does not print `PASS`, then the Forge shall keep the change out of `main`.
- **F-10** If `main` holds uncommitted changes that a loop did not make, then the Forge shall leave those changes as they are.
- **F-11** The Forge shall run black-box checks against `make run-test` (port 5188, `.wrangler/test-state`) or a worktree dev server.
- **F-12** If a loop is about to use the product owner's `make run` instance or data, then the Forge shall use `make run-test` instead.
- **F-13** The Forge shall give each test account an `@example.com` email.
- **F-14** When a loop is about to merge a change that created test data, the Forge shall require `make check-test-data` to print `PASS`.
- **F-15** If a loop is about to point a binding at production, then the Forge shall stop that step.
- **F-16** If a loop other than forge-release is about to run `/release`, then the Forge shall stop that step.
- **F-17** The Forge shall store each issue, research concept, backlog item, and retro as an OKF concept file in its mailbox bundle.
- **F-18** When a loop changes a mailbox bundle, the Forge shall update that bundle's `index.md` and `log.md`.
- **F-19** When a loop sets a BDR's status to Accepted, the Forge shall update that BDR's row in `docs/bdr/README.md` in the same commit.
- **F-20** If a loop meets a choice that only the product owner can make, then the Forge shall open a decision issue with the numbered options and a recommendation.
- **F-21** While a decision issue is open, the Forge shall leave the decided behavior unbuilt.
- **F-22** When the product owner answers a decision issue, the Forge shall record the answer, its time, and the next steps in that issue.
- **F-23** forge-fix builds a Proposed BDR in `docs/bdr` only when its Compliance section is complete and no legal issue for it is open, unless its status says to wait for the product owner. `make check-docs` fails a Proposed BDR whose Compliance section is missing or incomplete.
  - Open decision (2026-11-17): whether a Proposed BDR needs the product owner's approval in Station Mail before it is built. Until the product owner answers, F-23 holds.
- **F-24** The Forge shall store temporary and cache files in `.forge/tmp`.
- **F-28** The Forge shall keep a compliance register in `docs/compliance/register/`, an OKF bundle with one `Compliance Rule` concept per file.
  - Each rule holds its `laws`, the file globs it `watches`, the BDR or ADR ids that meet it (`met_by`), `status` (`open` or `met`), and `confirmed`.
  - A rule is `met` only when an Accepted BDR or ADR meets it.
  - Only the product owner sets `confirmed: true`; a loop leaves it `false`.
  - When a loop adds, changes, or removes a rule file, it updates the register's `index.md` and `log.md`. `make check-docs` fails when a rule file is not in the index, or an index entry has no file.
- **F-32** When an issue has 3 attempts and forge-verify still finds its failure, the Forge shall open a decision issue that links each attempt and its evidence, and stop working that issue.
- **F-33** If two BDRs, or a BDR and an ADR, require behaviors that conflict, then the Forge shall open a decision issue that quotes both, and leave the conflicting part unbuilt.

## 4. forge-fix

- **FX-1** When forge-fix runs, the Forge shall pick one open issue from `.forge/mailbox/issues`.
- **FX-2** When forge-fix has picked an issue, the Forge shall work it in a new worktree with the `/ub-mode` principles.
- **FX-3** When forge-fix has verified the fix, the Forge shall set the issue's status to `done`.
- **FX-4** If forge-fix cannot finish an issue in one run, then the Forge shall open a new issue that says what remains.
- **FX-5** If forge-fix runs with no open issue it can work, then the Forge shall keep its cron scheduled.
- **FX-6** When forge-fix picks an issue, the Forge shall add 1 to the issue's `attempts`.
- **FX-7** When an issue names a failing test that forge-verify wrote, the Forge shall make that test pass without weakening or removing it.

## 5. forge-verify

- **VE-1** When forge-verify runs, the Forge shall re-verify each BDR against a `make run-test` instance.
- **VE-2** When forge-verify finds a BDR criterion that fails, the Forge shall add one issue file for that failure.
- **VE-3** When forge-verify finds no failure for a BDR, the Forge shall set that BDR's status to Accepted.
- **VE-4** When a row in `docs/bdr/README.md` differs from its BDR file's status, the Forge shall add an issue for that row.
- **VE-5** When forge-verify finishes a pass, the Forge shall record the pass in `docs/bdr/README.md`.
- **VE-7** When a Proposed BDR has a criterion with no test, the Forge shall write a worker or e2e test for that criterion that fails on `main`, and add one issue that names the test.
- **VE-8** If a new test fails because of the test environment and not a missing behavior, then the Forge shall fix the test before it adds an issue.
- **VE-9** When forge-verify finds the failure of an issue that is `done`, the Forge shall set that issue's status to `open` again.

## 6. forge-retro

- **RE-1** When forge-retro runs, the Forge shall write one retro file `.forge/mailbox/retro/<YYYY-MM-DD-HHMM>-retro.md`.
- **RE-2** The Forge shall cover in each retro what each loop did since the last retro, with evidence.
- **RE-3** The Forge shall list in each retro the status of the last retro's action items.
- **RE-4** The Forge shall state each retro action item as a change to one loop skill, with the loop, the change, and the reason.
- **RE-5** If forge-retro is about to change code, an issue, or a loop skill, then the Forge shall stop that step.
- **RE-6** When forge-retro has written a retro, the Forge shall require `/okf validate .forge/mailbox/retro` to pass.
- **RE-7** The Forge shall organize each retro's findings as Keep, Stop, and Start.
- **RE-8** The Forge shall compare in each retro the issues closed and BDRs accepted since the last retro with the change in each key result from `make okr JSON=1`.

## 7. forge-research

- **RS-1** When forge-research runs, the Forge shall answer the open research requests in `.forge/inbox` before any other research, oldest first.
- **RS-2** If `.forge/inbox` is missing or empty, then the Forge shall skip the research requests.
- **RS-3** When forge-research has no open research request, the Forge shall research improvements drawn from the backlog, the BDRs, and the issues.
- **RS-4** The Forge shall add each research result as one OKF concept in `.forge/mailbox/research`.
- **RS-5** When forge-research answers a request, the Forge shall set the request's `status` to `done` and add a `# Response` section that links each new concept.
- **RS-6** If forge-research cannot answer a request, then the Forge shall add a `# Response` section that says why and what is needed.
- **RS-7** When forge-research finds a gap between a key result and the current behavior, the Forge shall propose one BDR with status Proposed that names the key result it moves.
- **RS-8** When forge-research proposes a BDR, the Forge shall add its row to `docs/bdr/README.md` in the same commit.
- **RS-9** While an Outcome is stopped, the Forge shall propose no BDR for it.

## 8. forge-prototype

- **PR-1** When forge-prototype runs, the Forge shall pick the best backlog item that has no `prototype:` key.
- **PR-2** When forge-prototype has picked an item, the Forge shall build its frontend in Storybook in a new worktree.
- **PR-3** When the prototype passes `make check`, the Forge shall commit it on the prototype branch.
- **PR-4** When forge-prototype commits a prototype, the Forge shall mark its backlog item `ready-for-review` with a `prototype:` block and a `# Prototype` section.

## 9. forge-station

- **ST-1** When forge-station runs, the Forge shall list the other Claude Code sessions for this repository.
- **ST-2** When forge-station runs, the Forge shall run `python3 -I .forge/station/collect.py`.
- **ST-3** When the collector has run, the Forge shall upload `history`, `loops`, `issues`, `usage`, `runs`, `retros`, `retro_actions`, `changelog`, `key_results`, and `behaviors` as dashboard assets, one file per upload.
- **ST-4** When the uploads finish, the Forge shall point each dashboard dataset at its new asset in one batch write.
- **ST-5** When the batch write commits, the Forge shall delete the assets that were replaced one run earlier.
- **ST-6** When the batch write commits, the Forge shall record the assets it replaced in `.forge/station/pending-deletes.json`.
- **ST-7** When forge-station has updated the dashboard, the Forge shall read the page's `check` and `runs` records and fix any current fault.
- **ST-8** When forge-station finishes, the Forge shall report the changes since the last run, each loop's status and run count, the open and done issues, whether the project is hill climbing, and the 5-hour and weekly usage with their forecasts.
- **ST-9** If a loop is idle for twice its interval or more, then the Forge shall report that loop as stalled.
- **ST-10** If a usage limit is forecast at 80% or more at its reset, then the Forge shall call out that limit in the report.
- **ST-11** The Forge shall keep Station Mail, the Mail section of the Loop Station dashboard, as the mailbox between the product owner and forge-station, one document per message in the dashboard store's `mail` collection.
- **ST-12** When forge-station runs, the Forge shall read the product owner's unread Station Mail messages before any other step.
- **ST-13** When forge-station reads an unread message from the product owner, the Forge shall do what the message asks in the same run, within the requirements for all loops.
- **ST-13a** When a message needs research, a report, or a file change that would delay the dashboard update by more than about 10 minutes, the Forge shall give it to a background subagent, with at most 3 subagents running at a time.
- **ST-13b** When a subagent finishes a message, the Forge shall reply in its thread with the result.
- **ST-13c** If a thread is marked `pending` and its subagent is not running, then the Forge shall start a subagent for that thread again.
- **ST-14** If a message asks for a change that a requirement for all loops forbids, then the Forge shall reply with the requirement and leave the change undone.
- **ST-15** When forge-station has handled a message, the Forge shall add a station reply in its thread that says what was done, and set the message's `read` to `true`.
- **ST-16** When a message answers a decision issue, the Forge shall record the answer in that issue.
- **ST-17** When the report names something that needs the product owner, the Forge shall keep one open Station Mail thread about it.
- **ST-18** When forge-station runs, the Forge shall run `make okr JSON=1` and add each key result's value to the `key_results` history, counts only, with `fewer than 5` kept as it prints.
- **ST-19** When forge-station runs, the Forge shall add to the `behaviors` history the number of BDRs that are Proposed, Accepted, and failing (a BDR with an open issue).
- **ST-20** The Forge shall show the key results and the behavior counts over time on one dashboard view, beside the issues opened and closed.
- **ST-21** If 10 or more issues closed in the last 7 days and no key result moved, then the Forge shall report that the work is not moving the Outcomes.
- **ST-22** When every BDR that names a key result is Accepted and that key result is below its target or not measured, the Forge shall keep one open Station Mail thread that asks the product owner to keep going, change the target, or stop the Outcome.

## 10. forge-release

- **RL-1** Where forge-release is running, when its cron fires, the Forge shall run `/release alpha`.
- **RL-2** If any session other than the product owner's is about to start forge-release, then the Forge shall stop that step.
- **RL-3** If `main` holds uncommitted changes, then the Forge shall stop the release and list the files.
- **RL-4** If a release gate fails, then the Forge shall stop the release and report the step and its output.
- **RL-5** When a release deploys, the Forge shall tag the deployed commit `alpha-<YYYY-MM-DD>-<n>`.
- **RL-6** If a release is about to deploy to production, then the Forge shall stop the release.

## 11. Loop registry

- **LR-1** The Forge shall describe each pane in `.forge/loops.json` as one row with `loop`, `pane`, and `command`.
- **LR-2** The Forge shall give each loop row the fields `session`, `cron`, `every_min`, `status`, `last_run`, `next_run`, `runs`, and `expires`.
- **LR-3** The Forge shall write `pane` as `"<column>,<row>"` in a grid that starts at `"0,0"`.
- **LR-4** The Forge shall give the row `home` the pane `"0,0"` and no command.
- **LR-5** The Forge shall compute `every_min` from every time in the loop's cron.
- **LR-6** The Forge shall set `status` to `running`, `stalled`, or `not running`.
- **LR-7** The Forge shall keep `.forge/loops.json` as a hand-edited file that `collect.py` leaves unchanged.

## 12. Outcomes

- **OC-1** The Forge shall treat each objective in `OUTCOMES.md` as an Outcome and its key results as the Outcome's measure.
- **OC-2** The Forge shall let only the product owner mark an Outcome achieved, change a target, or stop an Outcome.
- **OC-3** When the product owner marks an Outcome achieved or stopped, the Forge shall record the decision, its date, and the key result values in `OUTCOMES.md`.
- **OC-4** A verified BDR shall not count as an achieved Outcome; only the key results do.

## 13. forge-up

- **UP-1** The Forge shall store the setup prompt in `.claude/skills/forge-up/SKILL.md`.
- **UP-2** When the product owner runs `/forge-up`, the Forge shall create each missing folder: `.forge/mailbox/issues`, `.forge/mailbox/inbox`, `.forge/mailbox/backlog`, `.forge/mailbox/research`, `.forge/mailbox/retro`, `.forge/inbox`, `.forge/station`, `.forge/tmp`, and `.forge/worktree`.
- **UP-3** When forge-up creates a mailbox bundle, the Forge shall add the bundle's `index.md` and `log.md`.
- **UP-4** If `.forge/loops.json` is missing, then the Forge shall create it with the `home` row and one row for each loop in section 2, each with `status` `not running`.
- **UP-5** If a folder or file that forge-up would create already exists, then the Forge shall leave it unchanged.
- **UP-6** The Forge shall schedule no cron and start no loop during forge-up.
- **UP-7** When forge-up has created the mailbox bundles, the Forge shall require `/okf validate` to pass on each bundle.
- **UP-8** When forge-up finishes, the Forge shall report each folder and file it created, each one it found already in place, and the command that starts each loop.
