# Project Muon — Task Playbook

A working playbook for one DeepSWE task, from a fresh local clone of the task repo to a green pull request.
Order matters: do the heavy lifting locally, and fork, push and open the PR only at the end.

> Source of truth is always the task repo's own `README.md` (Start / Finish commands) and the Muon guide.
> Where this playbook and the README differ, follow the README.

---

## 0. Golden rules (read before every task)

**Always**
- Build the *whole* feature in `instruction.md`, yourself, against the pinned base commit.
- Keep `solution.patch` and `test.patch` on **disjoint file sets**. Not one shared file.
- Take every test id from a real report. Never type ids by hand.
- Make every sentence in `instruction.md` map to a test, and every F2P test map to a sentence.
- Run the full existing suite before every push. One broken existing test = reward 0.
- Fix pipeline stages top-down: 1️⃣ → 2️⃣ → 3️⃣ → 4️⃣.

**Never**
- `git clone` the upstream repo, or fetch a branch or a tag. Only fetch the base commit by hash.
- Read or port an upstream PR, commit, or another task's code.
- Touch LOCKED fields in `task.toml`, the 20 scaffold fields, `tests/grader.py`, or the Dockerfile fetch block.
- Put the answer (solution or hidden tests) anywhere the agent image can see.
- Use `RUN --mount=type=bind`, `ADD <url>`, `FROM x:latest` or an untagged `FROM`.
- Mention grading in `instruction.md` (grade, graded, grader, grading, F2P, P2P, fail-to-pass, pass-to-pass, node ID, whitelist, allowlist).
- Let `test.sh` decide or overwrite the reward.
- Swap in a different feature. If the seed is broken, ask in `#muon-tasking`.
- Use the GitHub "Re-run" button. Push a new commit instead.

---

## Good practices from review feedback (suggestions)

These aren't pipeline rules, but tasks that followed them tended to review better.

**Worth keeping up**
- **Stay in scope.** The strongest tasks kept the seeded scope as it was and put the effort into quality: a cleaner solution, sharper tests, a clearer instruction.
- **Keep `test.patch` and the F2P list in step.** Ideally every test you add in `test.patch` shows up in `f2p_node_ids`, and every F2P id comes from `test.patch`, with nothing left over on either side.
- **Let Dockerfile errors show.** Try to drop `>/dev/null 2>&1 || true` from `RUN` lines, in every Dockerfile you touch. It hides output and failures reviewers want to see.

**Worth watching for**
- **Tests a shortcut could pass.** If one simple check could make every test pass, the tests probably aren't strong enough. Aim at the scenario that actually matters, e.g. behaviour while an operation is in progress, not only before it starts.
- **Edits to things that already work.** Leave existing tests and working code alone unless the feature needs the change. Even harmless tweaks count as out of scope and add review noise.
- **Over-formatted instructions.** `instruction.md` reads best when it sounds like an engineer writing to a coding agent: a few plain paragraphs, without H1/H2 heading structure.
- **Instructions that keep growing.** Small clarifications are fine, but try to keep the instruction close to the seeded idea, short and focused, rather than expanding it each round.

**Reviewer notes on quality**
- **Write the instruction like a real issue or feature request.** Say what's missing, what the user wants, and why. These don't need to be headings, just the content.
- **Describe the behaviour, not the code.** Don't name private functions, files to edit, algorithms or implementation steps.
- **Tests should accept any reasonable solution.** Test through the public API or CLI, not internal helper names, and not wording the instruction doesn't give.
- **Everything the tests check should follow from the instruction.** Re-check this after every edit to either side.

---

## 1. Phase A — Local setup (you already cloned the task repo)

```bash
cd <task-repo>                      # origin = handshake-project-muon/<task>
git checkout -b feat/<short-feature-name>

# Your identity, set per-repo so nothing else leaks in
git config user.name  "Pushkar Saini"
git config user.email "<your GitHub email or the noreply address>"

# Pick up any seed changes made after you cloned (task_id, hai_difficulty...)
git fetch origin && git merge origin/main
```

Read, in this order:
1. `instruction.md` — the feature.
2. `task.toml` — `repository_url`, `base_commit_hash`, `hai_difficulty`.
3. `README.md` — the exact Start and Finish commands.
4. `environment/Dockerfile`, `tests/Dockerfile`, `tests/test.sh`, `tests/config.json`, `solution/solve.sh`.
5. `tests/grader.py` — read only. Note how it turns a report entry into an id, and where `reward.json` lives.

### Fetch the base commit into `workspace/`

Use the README's Start block. It is equivalent to:

```bash
REPO=$(grep -E '^\s*repository_url'   task.toml | sed -E 's/.*"(.*)".*/\1/')
BASE=$(grep -E '^\s*base_commit_hash' task.toml | grep -oE '[0-9a-f]{40}')

git init -q -b main workspace
git -C workspace fetch -q --no-tags "$REPO.git" "$BASE"
git -C workspace checkout -q -B main FETCH_HEAD
git -C workspace submodule update --init --recursive

git -C workspace config user.name  "Pushkar Saini"
git -C workspace config user.email "<same email>"
git -C workspace log --oneline -1    # must print $BASE
```

`workspace/` is git-ignored. Don't remove that line from `.gitignore`.

---

## 2. Phase B — Triage (stop early if the seed is bad)

Ask in `#muon-tasking` **before building** if any of these is true:
- The feature already exists in the code at the base commit.
- An open/merged public PR already implements it. (Check for existence only. Don't open its diff.)
- `instruction.md` contradicts itself or the code.
- The suite needs a GPU, a browser, an external DB, a cloud account, network at test time, or a very long build.
- `task_id` in `task.toml` is not a lowercase UUID.

Also write down:
- Install command, test runner, report format available (JUnit / CTRF).
- The exact command that runs the full existing suite, and how long it takes.

---

## 3. Phase C — Learn the project's code structure and commit pattern

Reviewers and the LLM checks judge whether the work looks like a real change to *this* project. A solution that sits where the maintainers would have put it, in their style, with tests shaped like theirs, reads as genuine engineering and gets accepted more often. Spend real time here before writing any code.

The fetch in Phase A brings the base commit **and its full history**, so everything below works offline inside `workspace/`. Everything you study is at or before the base commit, so it can't contain the solution. Study the patterns; don't copy code from past commits.

### 3.1 Code structure

```bash
cd workspace
git ls-files | sed -E 's|/[^/]+$||' | sort | uniq -c | sort -rn | head -40   # where the code mass lives
ls; cat README* CONTRIBUTING* 2>/dev/null | less                                # layout, contributor rules
```

Work out and follow:
- **Module layout:** where features of this kind live, how files are split, how public API is exposed (`lib.rs` / `mod.rs` re-exports, `__init__.py` exports, `index.ts` barrels, Go package boundaries).
- **Core abstractions:** the types, traits/interfaces, builders and config objects your feature should plug into instead of bypassing.
- **Error handling:** the project's error type(s), how errors are constructed and worded, `Result`/exception conventions.
- **Options and defaults:** how existing options are declared, validated, documented, defaulted.
- **Naming:** casing, prefixes/suffixes, verb choices (`with_*`, `set_*`, `parse_*`), file names.
- **Docs:** doc-comment style and density, whether public items get examples, whether feature PRs touch `docs/` or the README.
- **Formatting and lint config:** `rustfmt.toml`, `clippy.toml`, `.editorconfig`, `pyproject.toml` (ruff/black/mypy), `.eslintrc*`, `.prettierrc*`, `golangci.yml`, and the lint/test steps in `.github/workflows/*.yml`. Run exactly those tools on your changes.
- **Test layout:** unit vs integration dirs, shared helpers and fixtures, naming of test files and functions, assertion style, snapshot/golden-file usage.

Read 2–3 existing features that are closest to yours, end to end: entry point → core logic → error paths → tests. Your feature should look like a sibling of them.

### 3.2 Commit pattern

```bash
git log --oneline --no-merges -60                                   # message style at a glance
git log --no-merges --format='%s' -500 \
  | grep -oE '^[a-zA-Z]+(\([^)]*\))?!?:' | sort | uniq -c | sort -rn | head   # conventional-commit prefixes, if any
git log --no-merges --format='%s' -500 | awk '{print length}' | sort -n | awk '{a[NR]=$1} END{print "median subject length:", a[int(NR/2)]}'
```

Note:
- Subject format: `feat(parser): ...`, `parser: ...`, `[Parser] ...`, plain imperative, or past tense. Capitalisation and trailing period.
- Whether bodies are used, how long, wrapped at 72 columns or not.
- Whether issue/PR numbers are referenced, and whether `Signed-off-by` (DCO) is standard.

### 3.3 What a typical feature commit touches

```bash
git log --no-merges --stat -15 -- <dir where your feature belongs>
git log --no-merges --oneline -i --grep='add\|support\|feat' | head -30    # find feature-sized commits
git show --stat <sha-of-a-comparable-feature>
```

For comparable features, check what changed together: source files, tests, docs, `CHANGELOG.md` / `CHANGES.rst` / changeset files, type stubs, examples. Mirror that shape in your solution (as long as the solution and test patches stay on disjoint files).

### 3.4 Use it

- Place code, name things and handle errors the way the project does.
- Write hidden tests in the project's test style and location.
- Write the two workspace commits (Phase G) in the project's commit-message style.
- If feature commits normally update the changelog or docs, do the same in the solution.
- Use `git blame` on neighbouring code when unsure how something is meant to be done.

---

## 4. Phase D — Prove it runs offline (do this before writing code)

Build both images from the seeds, adjusting only base image, dependency install and test runner bits.

```bash
docker build -t muon-agent    environment/
docker build -t muon-verifier tests/
```

> `tests/Dockerfile` is a stub. Copy the steps of `environment/Dockerfile` into it (including the same fetch block), then keep the seeded `COPY . /tests` line and everything below it.

Run the existing suite with **no network**:

```bash
docker run --rm --network none muon-agent bash -lc 'cd /app && <full test command>'
```

If it reaches for the network, move that work into the image build (install deps, pre-fetch, pre-build). The build may use the network. The test run may not.

Tip: keep `RUN` output visible. If a seeded or inherited line ends in `>/dev/null 2>&1 || true`, consider removing that suffix so build errors surface instead of being swallowed:

```dockerfile
# rather than
RUN npm ci >/dev/null 2>&1 || true
# prefer
RUN npm ci
```

Sanity checks on the agent image:

```bash
docker run --rm muon-agent bash -lc 'ls /tests /solution 2>&1; cd /app; git log --oneline -1; git tag; git remote -v; git config user.name'
# expected: "No such file" for /tests and /solution, base commit, no tags, no remotes, a git name set
```

If the agent image has no git identity / pager config, add:

```dockerfile
RUN git config --system user.name "muon-agent" \
 && git config --system user.email "muon-agent@localhost" \
 && git config --system core.pager cat
ENV GIT_PAGER=cat
```

---

## 5. Phase E — Write the solution

Work only inside `workspace/`.

**Do**
- Apply what you learned in Phase C: module layout, naming, error types, logging, docs style, and the files a feature commit normally touches.
- Build every behaviour in the instruction: options, defaults, edge cases, error paths.
- Run the repo's formatter and linter (`cargo fmt` / `clippy`, `ruff`/`black`, `prettier`/`eslint`, `gofmt`) on the files you touched, using the repo's config.
- Update public docs or changelog only if the repo normally does so in feature PRs, and only in files the tests won't touch.
- Run the full existing suite often.

**Don't**
- Reformat or "clean up" unrelated lines. Large incidental diffs break tests and read as generated.
- Change working code the feature doesn't need, even small harmless fixes. It's best kept out of scope.
- Add hidden tests to the solution. Source changes only.
- Touch any file you will also need to touch in the tests (e.g. a `mod tests` block inside a source file you edit, or a shared `conftest.py` / fixtures file).
- Add new dependencies unless unavoidable. If you must, make sure the image pre-installs them.

**Disjoint files, by language**
- Rust: put hidden tests in a **new** integration test file `tests/<feature>.rs`, not in `#[cfg(test)]` inside a source file you change.
- Python: new `tests/test_<feature>.py`. Don't edit a `conftest.py` the solution also needs.
- JS/TS: new `*.test.ts` file. Don't add fixtures to a shared file the solution edits.
- Go: a new `_test.go` file is in the same package — fine, as long as the solution doesn't edit that file.

---

## 6. Phase F — Write the hidden tests

**Principles**
- Test observable behaviour through the public API or CLI: output, return values, raised errors.
- Any reasonable implementation of the instruction must pass. Don't call internal helpers or assert on private names, call order, or exact wording unless `instruction.md` gives that wording. A useful check: picture a second engineer solving it differently from you. Would your tests still pass for them?
- Cover each edge case the instruction names. Nothing the instruction doesn't name.
- No timing, randomness, wall-clock, or filesystem-order dependence.
- Each new test must **fail at the base** and **pass with your solution**.
- Match the repo's test style: helpers, fixture patterns, naming. Avoid generic names like `test_feature_works` or `test_basic`.
- Suggestion: ask yourself whether a single simple shortcut (a hardcoded value, a flag check, handling only the starting state) could pass all your tests. If so, add a case for the scenario that really matters, such as state while an operation is running.
- Prefer leaving existing tests untouched. Add new tests in new files rather than editing tests that already pass.

**Keep a test-to-sentence map:**

| Test id | Exact sentence in instruction.md that states it |
|---|---|
| `tests/test_x.py::test_zero_inherits` | "Zero inherits the column-level value." |

If a row has no sentence, either add the sentence or drop the test. pass@ blocks on NOT STATED tests. Everything a test checks should follow from the instruction as written, so walk the map again whenever you edit the instruction or the tests.

**Compiled languages:** a new test file that uses the new API won't compile at the base, and a single `cargo test` / `go test ./...` / `tsc` build failure kills every existing test too. So in `test.sh`, run the existing suite and your new tests in **separate invocations** with separate reports.

---

## 7. Phase G — Make the two patches

Rebuild the workspace history into exactly two commits at packaging time. Write both messages in the upstream project's commit style from Phase C (e.g. `feat(table): support per-row width limits` if the project uses conventional commits):

```bash
cd workspace
git reset -q --soft "$BASE" && git reset -q          # all changes unstaged, base intact
git add <solution files only>  && git commit -q -m "<feature commit in upstream style>"
git add <test files only>      && git commit -q -m "<test commit in upstream style>"
git status --porcelain                                 # must be empty
cd ..
```

Then run the README's **Finish** commands to write `solution/solution.patch` and `tests/test.patch` (they also add the canary line at the top). Conceptually:

```bash
git -C workspace -c diff.noprefix=false -c color.ui=never diff --no-ext-diff "$BASE" HEAD~1 > solution/solution.patch
git -C workspace -c diff.noprefix=false -c color.ui=never diff --no-ext-diff HEAD~1 HEAD     > tests/test.patch
# + canary line at the top of each — use the README command, not this
```

Check:

```bash
files_of() { grep '^diff --git ' "$1" | awk '{sub(/^b\//,"",$4); print $4}' | sort -u; }
comm -12 <(files_of solution/solution.patch) <(files_of tests/test.patch)   # must print nothing
```

---

## 8. Phase H — `test.sh`, `solve.sh`, `config.json`

### `tests/test.sh` shape

```bash
#!/usr/bin/env bash
set -uo pipefail          # no -e: a failing test must not skip grading

python3 /tests/grader.py prepare
# If prepare already wrote reward.json (agent patch didn't apply), stop.
# Use the exact path the seed / grader.py uses:
[ -f /logs/verifier/reward.json ] && exit 0

mkdir -p /logs/verifier/reports
cd /app

# 1) existing suite  -> report A
<existing-suite command, writing JUnit/CTRF to /logs/verifier/reports/existing.xml> || true

# 2) new tests       -> report B (separate invocation)
<new-tests command, writing to /logs/verifier/reports/new.xml> || true

python3 /tests/grader.py grade      # last line. Nothing after it.
```

- No custom pass/fail logic, no extra build/lint gates that can change the result, never write `reward.json` yourself.
- Report paths must match `config.json` and live under `/logs/verifier/`.

### `solution/solve.sh`
Leave the seeded version. It applies, stages and commits the patch with an explicit git name/email.

### `tests/config.json`
- Keep the seeded keys. Set `base_commit` = `base_commit_hash` from `task.toml`.
- Set the grade block (`format`: `junit` or `ctrf`, `node_id`, `tool_label`, report paths) to match your runner.
- Replace every `[FELLOW TO REPLACE ...]` placeholder.
- `p2p_excluded`: `{ "<id>": "one-line reason" }`. Never an F2P id, never an id also in `p2p_node_ids`.
- Keep the canary.

---

## 9. Phase I — Derive F2P / P2P from real runs

Run **3× nop** (no solution) and **3× oracle** (with solution), each from a fresh container, network off.

```bash
mkdir -p ~/muon/<task>/runs
for i in 1 2 3; do
  # nop: base + hidden tests
  docker run --rm --network none \
    -v ~/muon/<task>/runs/nop$i:/logs/verifier \
    muon-verifier bash -lc 'cd /app && git apply /tests/test.patch && <same commands as test.sh, steps 1 and 2>'

  # oracle: base + solution + hidden tests
  docker run --rm --network none \
    -v "$PWD/solution/solution.patch:/tmp/solution.patch:ro" \
    -v ~/muon/<task>/runs/oracle$i:/logs/verifier \
    muon-verifier bash -lc 'cd /app && git apply /tmp/solution.patch && git apply /tests/test.patch && <same commands>'
done
```

(If the README gives a Harbor command for oracle/nop, prefer that — it runs the real `test.sh` and grader.)

Extract **passing** ids from each run's reports in exactly the format `grader.py` builds. Example for JUnit — **adjust the id format to match grader.py**:

```python
# ~/muon/bin/passing_ids.py  report.xml [...]  -> passing ids, one per line
import sys, xml.etree.ElementTree as ET
for path in sys.argv[1:]:
    for tc in ET.parse(path).iter("testcase"):
        if any(c.tag in ("failure", "error", "skipped") for c in tc):
            continue
        print(f'{tc.get("classname")}::{tc.get("name")}')   # <- match grader.py
```

```bash
cd ~/muon/<task>/runs
for r in nop1 nop2 nop3 oracle1 oracle2 oracle3; do
  python3 ~/muon/bin/passing_ids.py $r/reports/*.xml | sort -u > $r.ids
done

# stability: each set of three should be identical
diff nop1.ids nop2.ids; diff nop1.ids nop3.ids; diff oracle1.ids oracle2.ids; diff oracle1.ids oracle3.ids

sort -u nop*.ids > nop_any
comm -12 oracle1.ids oracle2.ids | comm -12 - oracle3.ids > oracle_all
comm -12 nop1.ids nop2.ids | comm -12 - nop3.ids > nop_all

comm -23 oracle_all nop_any            > F2P     # pass with solution, never without
comm -12 nop_all oracle_all            > P2P     # pass in all six runs
comm -12 F2P P2P                                  # must print nothing
```

- Any id that flips between runs is flaky: put it in `p2p_excluded` with a reason, or fix the test if it's yours.
- Skipped / pending tests stay out of P2P.
- P2P must contain the repo's **existing** tests. Guard tests of your own are allowed beside them, but it's usually cleaner to skip them.
- Every F2P id must be added by `test.patch`, and ideally every test `test.patch` adds is in F2P. A quick cross-check of the two lists before you paste is worthwhile.
- Paste the lists into `config.json` with a tiny script, not by hand.

---

## 10. Phase J — `instruction.md`

Rewrite it against your test-to-sentence map.

**Must**
- State, explicitly, every behaviour a test checks: output formats, exact error messages (if asserted), defaults, ordering, edge values, public names/options the tests call.
- Contain nothing that no test checks.
- End with this exact line as the **last line** (branch = the one `environment/Dockerfile` leaves `/app` on, normally `main`):

  ```
  IMPORTANT: Please work on this in a new branch from main and commit everything when you are done.
  ```
- Stay under 3,000 characters (hard fail above 6,000).

**Must not**
- Name private functions, classes, files to edit, algorithms, or implementation steps (Solution leakage). Describe the behaviour, not the code.
- Describe the hidden tests or how the work is checked. No "How this will be verified" section.
- Use any grading word.
- Use numbered acceptance criteria or requirement tables.
- Contain leftover tool tags like `<muon-instruction-generator>`.
- Turn the feature into a different task.

**Shape that reads well**

Write it like a real issue or feature request: what's missing, what the user wants, and why. The guide shows a template with `# Task:` and `##` headings, but reviewers don't require those headings and favour lighter formatting. Plain paragraphs that cover the same ground tend to land better:

```markdown
<What's missing today, and the situation where a user runs into it.>

<What the user wants instead, and why it matters to them.>

<Observable rules, edge cases and errors, in a short paragraph.>

Nothing that works today should change.

IMPORTANT: Please work on this in a new branch from main and commit everything when you are done.
```

**Voice:** write it like a maintainer briefing a colleague. Short, concrete, plain. See §13.

**Keep it close to the seed.** It's natural to add a sentence when a test needs one stated, but try not to let the instruction grow round after round. If it keeps expanding, trimming the tests back to the core behaviour is often the better fix.

---

## 11. Phase K — `task.toml`

Change only:
- `authors = [{ name = "Pushkar Saini" }]` — exactly one entry, name only, no email, not a plain string.
- `[metadata] annotator_id = "2de80ad7-3a90-44b8-b2f2-ef9e949dfa59"`

```toml
[task]
# name = "handshake/<task-repo>"   # LOCKED, leave as seeded
authors = [{ name = "Pushkar Saini" }]

[metadata]
# task_id, repository_url, base_commit_hash, hai_difficulty: LOCKED, leave as seeded
annotator_id = "2de80ad7-3a90-44b8-b2f2-ef9e949dfa59"
```
- Any field marked `# FELLOW` (e.g. `description`, `display_title`).

Leave alone: `[task] name`, `task_id`, `repository_url`, `base_commit_hash`, `hai_difficulty`, every timeout/cpus/memory/storage/gpus/os/network_mode/environment_mode/schema_version/artifacts/mcp_servers/ext_id, and both canary comments.

---

## 12. Phase L — Final three checks (restart from Check 1 after any change)

**Check 1 — nothing changed by accident**
- Patches match what you built. Overlap check prints nothing.
- `git diff origin/main -- instruction.md` shows only intended edits.
- `git status` clean, no stray files.

**Check 2 — scores correctly**
- Oracle: reward 1, every F2P passes, every P2P passes.
- Nop: reward 0, no F2P passes, every P2P passes.

**Check 3 — lists are real and stable**
- 3× nop and 3× oracle from scratch give identical passing sets.
- Re-derived F2P/P2P equal `config.json` exactly.
- Every listed id appears in the oracle report.
- Files the agent's patch would add don't stop `test.patch` from applying.

Then run the preflight script (Appendix A).

---

## 13. Hygiene — making the PR read like careful human work

This is mostly the quality bar itself: the **Task hygiene** check flags machine-generated text, AI chat phrases and notes-to-self, and reviewers send back branches with scratch files or debug code.
(If your contract restricts AI tool use, cleaning traces doesn't make the work compliant — the guide requires the solution and tests to be your own work. Check the terms.)

### Git and commits
- Make every commit yourself, from your own terminal, with `user.name` / `user.email` set per-repo (Phase A).
- No trailers: no `Co-Authored-By:`, no `Generated with ...`, no `Signed-off-by` unless you normally use it.
- If any assistant CLI is installed, turn off its commit attribution setting and don't let it run `git commit` for you. Check hooks too:
  ```bash
  git config --get core.hooksPath; ls .git/hooks | grep -v '\.sample$'
  git config --global --get commit.template
  ```
- Workspace commits (the ones the patches come from) follow the upstream project's commit style (Phase C).
- Task-branch commits: short, imperative, the way you'd normally write them.
  - Good: `Add solution and tests for row width limits`, `Fix P2P list`, `Tighten instruction wording`
  - Avoid: multi-paragraph bodies with bullet lists, emoji, "This commit introduces a comprehensive...".
- A few logical commits on the task branch is natural (`initial package`, then fixes as the pipeline reports). Don't squash every fix into a mega-commit, and don't produce ten near-identical "fix" commits either.
- Verify before pushing:
  ```bash
  git log origin/main..HEAD --format='%h %an <%ae>%n%B' | grep -inE 'co-authored|generated|claude|anthropic|copilot|chatgpt|openai|cursor|aider|🤖'
  # must print nothing
  ```
  To fix: `git rebase -i origin/main` and reword, or `git commit --amend` for the last one. Workspace commit messages never reach the patches (they are plain diffs), but keep them plain anyway.

### Files that must never be committed
Keep these out of the task repo (add them to your **global** ignore, `~/.config/git/ignore`, not the repo's `.gitignore`):

```
CLAUDE.md
AGENTS.md
.claude/
.cursor/
.cursorrules
.aider*
.copilot*
notes.md
scratch*
*.orig
*.rej
*.bak
.DS_Store
.idea/
.vscode/
```

### Code
- Follow the structure and conventions you mapped in Phase C. Code that looks like a natural sibling of existing features is the strongest signal of careful work.
- Match the repo's comment density. Most repos comment sparsely; don't add a docstring to every helper or a comment narrating each line.
- No comments like `// Added for <feature>`, `# New: ...`, `# Step 1: ...`, `// handle edge case` without content.
- No leftover `TODO`, `FIXME`, `dbg!`, `println!` debugging, `console.log`, `print(...)`, `breakpoint()`.
- No emoji, decorative unicode, or em dashes in code or comments unless the repo already uses them.
- Use the repo's existing helpers and error types instead of inventing parallel ones.
- Defensive over-engineering (validating impossible states, wrapping everything in try/except) is a common tell. Handle what the instruction asks for, the way the codebase already does.
- Variable names in the repo's convention; avoid generic `result`, `data`, `helper_fn` where the codebase uses domain names.

### Tests
- Same structure as neighbouring tests. Concrete values over abstract ones.
- Don't produce huge parametrize tables of trivial variations. A handful of meaningful cases.
- Test names describe behaviour in the repo's style (`test_zero_width_inherits_column_limit`).

### instruction.md and PR text
- Plain declarative sentences. No preamble ("This task requires you to..."), no sign-off.
- Watch for overused filler and review each hit: *comprehensive, robust, seamless(ly), leverage, utilize, delve, streamline, ensure that, it's worth noting, Note that, In summary, Additionally,* excessive **bold**, em dashes, title-cased headings everywhere.
- No "Certainly", "Here's", "Let me", "I've", "As requested".

---

## 14. Phase M — Fork, push, open the PR (last)

Only after §12 passes and the preflight is clean.

```bash
# 1. Pick up any late seed changes
git fetch origin && git merge origin/main      # re-run preflight if this changed anything

# 2. Fork on GitHub (web UI, or):
gh repo fork handshake-project-muon/<task> --clone=false --remote=false

# 3. Add your fork as a remote and push the branch
git remote add fork git@github.com:<your-user>/<task>.git
git push -u fork feat/<short-feature-name>

# 4. Open the PR into the task repo's main
gh pr create --repo handshake-project-muon/<task> --base main \
  --head <your-user>:feat/<short-feature-name> \
  --title "<Feature in a few words>" \
  --body "Solution, tests and task files for <feature>."
```

PR body: one or two plain sentences. No checklists, headings or "Test plan" templates.

Then watch the four comments on the PR: 1️⃣ Static QC → 2️⃣ Runtime → 3️⃣ LLM review → 4️⃣ pass@.
Every fix = a new commit pushed to the same branch. Submit on the platform only when all four are green.

### Submission question (once pass@ lands in the target bucket)

The submission form asks:

> Answer in very short: Were there any challenges or issues that you encountered while developing this task?

Answer in one or two plain sentences about what actually happened on this task. Name the specific problem and how you fixed it. Pick and adapt from these shapes, or write your own:

- `The new test file didn't compile at the base commit and took the existing suite down with it, so I split the existing and new tests into separate runs in test.sh.`
- `One existing test was flaky across nop runs; I moved it to p2p_excluded with a reason.`
- `pass@ first landed in the wrong bucket; I stated two more edge cases in the instruction and added tests for them.`
- `The LLM review flagged a test the instruction didn't state; I added the missing sentence.`
- `The build needed network access during tests; I moved dependency fetching into the Dockerfile.`
- `No major issues; most of the time went into matching the project's existing code and test structure.`

Keep it factual: no bullet lists, no filler, and don't claim a problem that didn't happen.

---

## 15. Pipeline triage

| Comment says | Usual cause | Fix |
|---|---|---|
| must end with `IMPORTANT: Please work on this...` | wrong branch name or text after it | copy the line from the error as the last line |
| exposes grading-only terminology | a grading word in instruction.md | rephrase as behaviour |
| missing the Project Muon canary | canary dropped from one of the 5 files | restore in README, task.toml, both patches, config.json |
| Harbor refused task.toml | plain-string author, bad field | fix the named field |
| still contains scaffold placeholder IDs | `[FELLOW TO REPLACE ...]` left | paste real ids |
| annotator_id must be ... UUID | missing / wrong | add under `[metadata]` |
| task_id / hai_difficulty changed or missing | old branch or accidental edit | merge `origin/main`; restore seeded value |
| Nop: F2P ids already pass | test doesn't need the feature | assert behaviour only the feature gives |
| Nop: none of the P2P ids passed | new test file won't compile at base | split existing/new invocations in test.sh |
| Oracle: reward 0 | solution fails own test, or solve.sh didn't commit | run oracle in a clean container, read report |
| P2P misses N of M tests | unlisted passing tests | download `p2p-missing.json`, add to P2P or `p2p_excluded` with reason |
| solution fails ... not graded tests | solution breaks an unlisted existing test | fix solution (or update the test in test.patch if intended) |
| None of the P2P ids is an existing test | P2P holds only your tests | add the repo's existing passing tests |
| 3️⃣ Prompt and tests match / Acceptance breadth / Solution leakage | spec ↔ tests gap, over-specific tests, leaked design | read the review, fix, push |
| 3️⃣ Answer unreadable | reviewer glitch | push again |
| 4️⃣ NOT STATED test | instruction doesn't state the asserted behaviour | add the exact sentence, or change the test |
| 4️⃣ solve breaks a stated rule | no test for a stated rule | add a test for it |
| 4️⃣ Bucket N full / hard task solved twice | difficulty off | adjust within the same feature (more stated edge cases to make harder; clearer wording or fewer edge cases to make easier); ask in `#muon-tasking` if it keeps missing |

Target buckets: **hard → 0–1**, **medium → 2–3**, **easy → 4–5 (prefer 4)**.

---

## Appendix A — Preflight script

Save as `~/muon/bin/muon-preflight.sh` (outside the repo), run from the task repo root after committing. Needs bash, GNU grep, python3. It mirrors the Static QC checks it can approximate locally; the pipeline remains authoritative.

```bash
#!/usr/bin/env bash
set -u
fail=0
ok()   { printf '  ok   %s\n' "$1"; }
bad()  { printf '  FAIL %s\n' "$1"; fail=1; }
warn() { printf '  warn %s\n' "$1"; }
BASE_REF=${BASE_REF:-origin/main}
BASE=$(grep -E '^\s*base_commit_hash' task.toml | grep -oE '[0-9a-f]{40}' | head -1)

echo "== required files"
for f in task.toml instruction.md environment/Dockerfile solution/solution.patch solution/solve.sh \
         tests/Dockerfile tests/config.json tests/grader.py tests/test.patch tests/test.sh; do
  [ -s "$f" ] && ok "$f" || bad "$f missing or empty"
done
git ls-files -s | awk '$1=="120000"' | grep -q . && bad "symlink tracked" || ok "no symlinks"
[ -z "$(git status --porcelain)" ] && ok "tree clean" || warn "uncommitted changes"

echo "== scope"
out=$(git diff --name-only "$BASE_REF"...HEAD | grep -vE '^(task\.toml|instruction\.md|README\.md|\.gitignore|environment/|solution/|tests/)')
[ -z "$out" ] && ok "only task files changed" || bad "out of scope: $out"
stray=$(git ls-files | grep -iE '(^|/)(claude\.md|agents\.md|\.claude/|\.cursor|\.aider|notes?\.md|scratch)|\.(orig|rej|bak)$')
[ -z "$stray" ] && ok "no stray files" || bad "stray files: $stray"

echo "== instruction.md"
n=$(wc -m < instruction.md)
[ "$n" -gt 6000 ] && bad "$n chars (>6000)" || { [ "$n" -gt 3000 ] && warn "$n chars (>3000)" || ok "$n chars"; }
branch=$(grep -oE -- '-B +[A-Za-z0-9._/-]+' environment/Dockerfile | tail -1 | awk '{print $2}')
expect="IMPORTANT: Please work on this in a new branch from ${branch:-main} and commit everything when you are done."
[ "$(tail -n 1 instruction.md)" = "$expect" ] && ok "closing line" || bad "last line must be: $expect"
grep -n -i -w -E 'grade|graded|grader|grading|f2p|p2p|fail-to-pass|pass-to-pass|node id|whitelist|allowlist' instruction.md \
  && bad "grading words above" || ok "no grading words"
grep -n -E '<[a-z]+(-[a-z]+)+>' instruction.md && warn "tool-like tag above" || true

echo "== canary"
CANARY=${CANARY:-$(grep -i canary README.md | grep -oE '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}' | head -1)}
if [ -z "$CANARY" ]; then warn "set CANARY=<guid>"; else
  got=$(git ls-files -z | xargs -0 grep -l "$CANARY" | sort | tr '\n' ' ')
  want="README.md solution/solution.patch task.toml tests/config.json tests/test.patch "
  [ "$got" = "$want" ] && ok "canary in exactly 5 files" || bad "canary files: $got"
fi

echo "== Dockerfiles"
for d in environment/Dockerfile tests/Dockerfile; do
  grep -nE '^\s*FROM\s' "$d" | grep -vE '@sha256:|:[A-Za-z0-9._-]+(\s|$)' && bad "$d: unpinned FROM (or stage alias — check)"
  grep -nE '^\s*FROM\s.*:latest' "$d" && bad "$d: :latest"
  grep -nE '^\s*FROM\s.*:agent(\s|$)' "$d" && bad "$d: local :agent base"
  grep -nE 'git +clone|--mount=type=bind|^\s*ADD +https?://' "$d" && bad "$d: forbidden source line"
  grep -q "$BASE" "$d" && ok "$d fetches base" || bad "$d does not fetch $BASE"
done
grep -nE '>/dev/null 2>&1 *\|\| *true' environment/Dockerfile tests/Dockerfile && warn "silenced RUN lines above (suggest removing)"
grep -nE '^#{1,2} ' instruction.md && warn "H1/H2 headings in instruction.md (suggest plain paragraphs)"
grep -q 'user.name' environment/Dockerfile || warn "agent image sets no git user.name"
grep -q 'core.pager' environment/Dockerfile || warn "agent image sets no core.pager"

echo "== patches"
files_of() { grep '^diff --git ' "$1" | awk '{sub(/^b\//,"",$4); print $4}' | sort -u; }
ov=$(comm -12 <(files_of solution/solution.patch) <(files_of tests/test.patch))
[ -z "$ov" ] && ok "patches disjoint" || bad "shared files: $ov"
tmp=$(mktemp -d)
if git -C workspace worktree add -q --detach "$tmp/w" "$BASE" 2>/dev/null; then
  git -C "$tmp/w" apply "$PWD/solution/solution.patch" && ok "solution applies at base" || bad "solution.patch does not apply"
  git -C "$tmp/w" apply --check "$PWD/tests/test.patch" && ok "test.patch applies on top" || bad "test.patch does not apply"
  git -C workspace worktree remove --force "$tmp/w"
else warn "could not create worktree at base (workspace missing?)"; fi
grep -nE '^\+.*(TODO|FIXME|XXX|dbg!\(|console\.log|breakpoint\(\)|import pdb)' solution/solution.patch tests/test.patch \
  && warn "debug/TODO lines above" || true

echo "== config.json + task.toml"
python3 - "$BASE" <<'PY' || fail=1
import json, re, sys
base = sys.argv[1]; bad = False
raw = open("tests/config.json").read(); c = json.loads(raw)
def err(m):
    global bad; bad = True; print("  FAIL", m)
f2p, p2p = c.get("f2p_node_ids", []), c.get("p2p_node_ids", [])
exc = c.get("p2p_excluded", {}) or {}
if "FELLOW" in raw: err("placeholder ids left")
if not f2p or not p2p: err("empty F2P or P2P")
if len(set(f2p)) != len(f2p) or len(set(p2p)) != len(p2p): err("duplicate ids")
if set(f2p) & set(p2p): err("id in both F2P and P2P")
for k, v in exc.items():
    if not str(v).strip(): err(f"p2p_excluded no reason: {k}")
    if k in f2p or k in p2p: err(f"p2p_excluded also graded: {k}")
if c.get("base_commit") != base: err("config base_commit != task.toml base_commit_hash")
t = open("task.toml").read()
if "Your Name" in t: err("authors still 'Your Name'")
if re.search(r'authors\s*=\s*\[\s*"', t): err("authors is a plain string")
if not re.search(r'annotator_id\s*=\s*"2de80ad7-3a90-44b8-b2f2-ef9e949dfa59"', t):
    err('annotator_id should be "2de80ad7-3a90-44b8-b2f2-ef9e949dfa59"')
if not re.search(r'authors\s*=\s*\[\s*\{\s*name\s*=\s*"Pushkar Saini"\s*\}\s*\]', t):
    err('authors should be [{ name = "Pushkar Saini" }]')
print("  ok   config/task.toml" if not bad else "", end="\n" if not bad else "")
sys.exit(1 if bad else 0)
PY

echo "== commits"
git log "$BASE_REF"..HEAD --format='%an <%ae>%n%B' \
  | grep -inE 'co-authored|generated (with|by)|claude|anthropic|copilot|chatgpt|openai|cursor|aider|🤖' \
  && bad "attribution/AI trailers in commits" || ok "commit messages clean"

echo; [ $fail -eq 0 ] && echo "PREFLIGHT OK" || echo "PREFLIGHT FAILED"
exit $fail
```

---

## Appendix B — Runner cheatsheet (install runners at image build time)

| Stack | JUnit/CTRF output | Notes |
|---|---|---|
| Rust | `cargo nextest` with a profile that has `[profile.<p>.junit] path = "junit.xml"`; report lands in `target/nextest/<p>/`, copy it to `/logs/verifier/reports/` | Pre-build in the image (`cargo fetch`, `cargo build --tests`) and set `CARGO_NET_OFFLINE=true` in test.sh. Select targets with `--lib --bins --test <existing>` for the existing suite and `--test <new_file>` for new tests, so a non-compiling new file can't break the existing run. Copy the report after each invocation so the second doesn't overwrite the first. |
| Python | `pytest --junitxml=/logs/verifier/reports/x.xml -p no:cacheprovider` | Watch parametrized ids with odd characters; take them from the report. |
| JS/TS (vitest) | `vitest run --reporter=junit --outputFile=/logs/verifier/reports/x.xml` | |
| JS/TS (jest) | `jest --reporters=default --reporters=jest-junit` with `JEST_JUNIT_OUTPUT_FILE=...` | `jest-junit` installed in the image. |
| Go | `gotestsum --junitfile /logs/verifier/reports/x.xml -- ./...` | Install `gotestsum` in the image; run new-test packages separately if they won't compile at base. |

Whatever the stack: the grade block in `config.json` (`format`, `node_id`, `tool_label`, paths) must match what you produce, and ids come from those reports.

---

## Appendix C — One-page checklist

- [ ] Branch created, per-repo git identity set (task repo + workspace)
- [ ] Base fetched by hash into `workspace/`, `git log -1` = base
- [ ] Seed triaged; nothing to escalate
- [ ] Upstream code structure, conventions and commit pattern studied; 2–3 comparable features read end to end
- [ ] Existing suite runs offline in the agent image
- [ ] Agent image: no `/tests`, no solution, no tags/remotes, git name + pager set
- [ ] Solution complete for every behaviour in the instruction; formatter run; no unrelated diffs
- [ ] Hidden tests in new files; fail at base, pass with solution
- [ ] Test-to-sentence map complete, no orphans either way
- [ ] Two workspace commits; patches written via README Finish; files disjoint
- [ ] `test.sh`: prepare → early exit → separate invocations → grade last
- [ ] 3× nop + 3× oracle stable; F2P/P2P derived and pasted; `p2p_excluded` reasons
- [ ] `instruction.md`: reads like an issue (what's missing, what's wanted, why); describes behaviour not code; no grading words; < 3,000 chars; exact last line
- [ ] Tests go through the public API/CLI, accept any reasonable solution, and check only what the instruction states
- [ ] `task.toml`: author `Pushkar Saini`, annotator_id `2de80ad7-3a90-44b8-b2f2-ef9e949dfa59`, `# FELLOW` fields; LOCKED untouched
- [ ] Hygiene: no trailers, no stray files, no debug lines, plain wording
- [ ] (Suggested) Scope unchanged, instruction short and heading-light, test.patch ↔ F2P aligned, no silenced `RUN` lines
- [ ] Preflight OK
- [ ] Merge `origin/main`, fork, push to fork, PR into task repo `main`
- [ ] All four pipeline comments green, pass@ in the target bucket
- [ ] Short, factual answer to the "challenges or issues" question → submit
