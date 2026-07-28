# Working context — Project Dynamo task

Handoff notes so a new chat session (on any machine) can pick up with full context.
Last updated: 2026-07-29

---

## 1. Who / what / where

- **Project:** Handshake AI "Project Dynamo" — fellows author tasks for
  Terminal-Bench 2 (TB2), a benchmark that evaluates coding agents in real
  terminal environments.
- **My assigned task repo:** `handshake-project-dynamo/dynamo-0fd2615-file-and-media-operations`
- **Category (pre-seeded, do not edit):** File and Media Operations
- **Sub-category (pre-seeded, do not edit):** Text editing and manipulation
- **Platform task id:** `b3c30129-b569-4428-a8d6-…` (status: In progress)
- **This scratch repo (personal, private):** https://github.com/Jyotisaini-2001/Dynamo
  Contains `docs/` — the project documentation, this file, and the proposal draft.
  Git identity is overridden repo-locally to my personal email
  (`sainijyoti042002@gmail.com`); global config remains my work email.

**Read the docs in this order:** `welcome.txt`, `intro.txt`, `installation.txt`,
`task-file.txt`, `create-task.txt`, `shipment.txt`.

---

## 2. What the project actually asks for

I am not solving problems — I am **writing an exam for an AI agent**, plus the
answer key and the grader. A task is a self-contained challenge in a Docker
container, scored deterministically.

Deliverables, all under `task/` in the forked repo:

| File | Role |
|---|---|
| `instruction.md` | The prompt — the ONLY thing the agent sees. Absolute `/app/...` paths, names every output file, describes WHAT not HOW, ≤1500 tokens, ends with the exact timeout line |
| `solution/solve.sh` | Golden/oracle solution (real logic in a `solve.py` helper). Must score reward 1.0. Never enters the image |
| `tests/test.sh` + `tests/test_outputs.py` | The grader. pytest, one test per criterion each with a docstring. Writes 1/0 to `/logs/verifier/reward.txt`. Added only at verify time |
| `environment/Dockerfile` | Single image for BOTH agent run and verifier run. Allowlisted base pinned by `@sha256:`, pinned pip deps incl. pytest + pytest-json-ctrf, `RUN mkdir -p /app` |
| `environment/data/` | Seed/input files, COPYed in |
| `task.toml` | Manifest — artifacts, timeouts, resources, metadata |
| `.dynamo/`, `.github/`, `.harbor/` | PROVIDED — do not edit |

**Golden rule:** the agent must never be able to read the solution or the tests,
and grading must be on real artifacts on disk — never stdout, never
string-matching source.

---

## 3. The gates (what can kill the task)

Local bar before opening a PR:
- `harbor run -p . --agent oracle` → reward **1.0**
- `harbor run -p . --agent nop` → reward **< 1.0**
- Image contains no `solve.sh` / `test.sh`
- Repeatable across re-runs (no unseeded randomness, no wall-clock dependence)

PR pipeline (blue = gates the PR):
1. Static checks 2. Rubric review 3. Duplicate check 4. Validation (oracle/nop)
5. **Pass@2** (max 6 runs/day) 6. Automated Review 7. **AVA** verifier audit
8. Adversarial Cheat-Pass (advisory only) 9. Per-Check QC 10. **Pass@5**

**THE gate that matters: Pass@5 ≤ 2/5.** GPT-5.4 + Terminus-2 must FAIL at least
3 of 5 attempts. 3–5/5 = rejected as too easy. 0/5 is the best result — but only
if the failures are *valid* (model finished and was wrong). A 0/5 made of
timeouts or prompt ambiguity is "broken, not hard" and also rejected.

Difficulty must come from **reasoning**, never from CPU/memory/volume/clock.
Lowering the timeout or adding busywork to force failures is gaming and gets
sent back.

Other constraints:
- 8-hour cap per task including revisions; max 2 revisions then Holding-Rejection
- Timeout hard ceiling 3600s; `[agent].timeout_sec` must match the "You have N
  seconds…" line in `instruction.md`
- **AI policy:** `instruction.md` and `solution/` must be human-written.
  Dockerfile and test boilerplate may be assisted but human-verified.

---

## 4. The chosen task idea (IDEA A)

**Reconstruct a document from its editor undo journal.**

The agent is given, in `/app`:
- `document.txt` — the final saved state of a text document
- `journal.jsonl` — the editor's append-only record of every edit
- `JOURNAL_FORMAT.md` — a full written spec of the journal format

It must write `/app/original.txt` — the document as it was before the first
recorded edit.

### Why this idea (the reasoning that selected it)

- **Answer key is correct by construction.** The generator writes the ORIGINAL
  document first, then derives the journal from it. We never have to *solve*
  anything — we build the maze from the exit. Correctness is checked by one line:
  does our solution's output equal the original we started with?
- **Verification is a single hash.** No tolerance, no ambiguity, no judgement.
- **Nothing to look up.** The journal format is ours; the document is generated.
- **Small and fast.** Difficulty from precision, not data volume — exactly what
  reviewers want.
- **Fails in the right way.** The model finishes, hands in confident work, and is
  wrong by a few characters. It literally cannot tell that it failed.
- **Good difficulty dial.** The complications stack independently, so if Pass@2 /
  Pass@5 come back too easy we add one more and push again.

### The four difficulty layers

1. **Positions are relative to intermediate states.** Each entry's offset refers
   to the document as it was at that moment. Only invertible by walking strictly
   backwards, applying each inverse to the state the previous inverse produced.
2. **Offsets are UTF-16 code units** (what VS Code / LSP / the DOM actually use),
   while the file is UTF-8 and deliberately contains emoji + CJK. An astral-plane
   emoji = 2 UTF-16 units, 1 Python char, 4 UTF-8 bytes. A byte-offset assumption
   works perfectly on the ASCII prefix and then silently drifts. **This is the
   killer.**
3. **Undone edits are in the journal but not in the document.** Append-only log
   includes operations the user performed then undid, plus undo markers. The
   effective sequence must be derived BEFORE any inversion.
4. **Line endings were normalised at save time**, so journal offsets and on-disk
   bytes are different coordinate systems.

The crux is that these must be handled **in the right order** (resolve undos →
convert coordinates → invert), and the agent has no reference copy, no error, and
no way to self-check.

### Intended solution (the oracle)

Core engine is tiny:

```python
text = read_final_document()
for entry in reversed(effective_sequence):
    if entry.kind == "insert":
        text = text[:entry.pos] + text[entry.pos + len(entry.text):]
    else:  # delete
        text = text[:entry.pos] + entry.text + text[entry.pos:]
write(text)
```

Everything else is refinement around that loop. Full pipeline: resolve undo
structure → establish coordinate system (UTF-16 ↔ native indices, denormalise
line endings) → invert backwards → write `/app/original.txt`.

Expert time estimate in the proposal: **3 focused hours**.

### Verification plan

1. `/app/original.txt` exists and is valid UTF-8
2. Byte length matches expected
3. **SHA-256 matches expected** (the exact-match criterion)
4. Input files unmodified (checked against their own digests) — blocks passing by
   rewriting inputs
5. **Replaying the effective journal forward over the submitted `original.txt`
   reproduces `document.txt` exactly** — computed fresh at grade time from the
   inputs, NOT read from a stored constant. This cross-validates the answer key
   by an independent route and is what the AVA verifier audit looks for.

Ground truth (the expected digest) lives in `tests/`, which is added only at
verify time — never on a path the agent can read.

### Build order (deliberately incremental)

1. Simplest version — plain ASCII, inserts/deletes only. Get oracle reproducing
   the original exactly. Now something works end to end.
2. Add ONE complication (e.g. emoji/UTF-16 offsets). Fix the oracle. Exact again.
3. Add the next. Fix. Exact again.
4. Stop when hard enough.

At every stage there is a working fallback. If a complication proves more trouble
than it's worth, drop it and keep the working version.

---

## 5. Ideas considered and rejected

- **IDEA D — untangle interleaved logs (KEPT AS BACKUP).** Several programs wrote
  to one log file concurrently; non-atomic writes tore lines in half and
  interleaved them. Agent must reconstruct each program's log exactly. Very
  realistic, exact hash checking. Risk: must prove exactly one valid reassembly
  exists or failures become unfair. **Pivot to this if A can't be made hard
  enough.**
- **IDEA E — recover a table from broken CSV quoting.** Checksum column + known
  column count makes it deterministic. Weaker: CSV repair is well discussed
  online.
- **IDEA F — three-way merge to an exact spec.** Rejected: it's careful
  implementation of a written spec, which GPT-5.4 is very good at. High risk of
  3+/5.
- **"Repair a documentation repo after a failed migration" (suggested by GPT).**
  Rejected. It's the exact shape LLMs excel at (broken links, frontmatter,
  heading levels) — a checklist, not a puzzle. No crux; "fix the
  inconsistencies" is unfairly vague; duplicate-check risk; and it drifts toward
  Software Engineering rather than text editing.

**The filter used for every idea:** *Where exactly does GPT-5.4 go wrong, and why
can't it tell?* If that can't be answered in one sentence, the idea fails the
difficulty gate. For A: *it never realises positions are counted in UTF-16, so
its output is silently shifted after the first emoji.*

---

## 6. Status and next steps

**Done:**
- Read all project docs
- Selected idea A, with D as backup
- Proposal drafted → `docs/proposal-draft.md`
- Scratch repo pushed to https://github.com/Jyotisaini-2001/Dynamo (private)

**Next:**
1. Edit `proposal-draft.md` into my own voice — especially the difficulty section
   and the 3-hour expert estimate. Reviewers see a lot of AI-written proposals,
   and I have to be able to defend every claim.
2. Start the timer on the Handshake platform (open the claimed task).
3. Submit the proposal — its four sections map 1:1 to the platform's four fields.
   Do NOT edit the pre-filled Category / Sub-category fields.
4. Iterate on reviewer feedback until it PASSES. Expect a round or two; that's
   normal and cheap.
5. **Only after the proposal passes:** set up tooling, fork the repo, build.

**Setup decision:** work will be done on my **personal laptop**, not the office
one. Docker Desktop was ruled out for office use (paid licence required for
companies 250+ employees, needs admin rights, may be blocked by MDM). If Docker
is ever needed on a restricted machine, use **Colima** instead
(`brew install colima docker && colima start`) — open source, home-directory
only, no admin rights.

Tooling needed when building starts (none needed for the proposal):
- Docker running
- `uv` (`curl -LsSf https://astral.sh/uv/install.sh | sh`)
- Python 3.11+
- `gh` + `gh auth login`
- `uv tool install harbor` — the package is **`harbor`**, NOT `harbor-cli`

**Worth doing early once the repo is forked:** read
`.dynamo/dynamo-rubric.toml` (the exact criteria the PR is graded on) and
`.dynamo/diversity-taxonomy.toml` (the valid labels for `task_objective` and
`artifact_type`).

Likely metadata labels for this task: `task_objective = ["recover_or_repair_artifact", "transform"]`,
`artifact_type = ["text_or_log_file"]` — confirm against the taxonomy file.
