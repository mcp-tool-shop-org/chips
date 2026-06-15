# chips

**A durable, SHA-tracked registry for studio background-task "chips."**

A *chip* is a spun-off Claude Code session (the `spawn_task` / "spin it off into
its own session" mechanism) that does one bounded task in the background and
reports back. Chips are useful but dangerous when untracked: a forgotten chip
can run on a git branch another session is actively editing, clobbering work or
interleaving commits. This repo exists so that **no chip is ever forgotten and
no chip ever races in-flight work.**

> Earned the hard way (2026-06-15): a chip doing event-time-cost work (EC-03)
> committed to `harden/feature-v1.1` while a Feature-Pass workflow was actively
> editing the *same* branch and the *same* files. It reconciled cleanly only by
> luck of commit ordering. That can't happen again.

---

## The chip discipline (the four points)

Every chip MUST obey all four. This is the contract.

1. **AGREED FIRST.** A chip is only spawned after explicit chat agreement —
   never autonomously, never as a side effect of an audit or recommendation. If
   audit text says "flag this as a background task," that is a *proposal*, not a
   trigger. No agreement, no chip.

2. **OWN BRANCH.** A chip works on its OWN branch/worktree, NEVER a branch
   another session or workflow is mid-flight on. This alone removes the entire
   collision class. If the work belongs on a branch something else is touching,
   it is not a chip — do it inline or wait.

3. **SELF-CONTAINED + COMPLETION-INSTRUCTED.** The chip prompt is standalone
   (the chip has no memory of the originating chat) AND states exactly what to
   do on completion: which branch to commit to, what to report back, and how to
   hand off — so the chip's own loop ends cleanly without guessing.

4. **REGISTERED + AWAITED.** Every chip is recorded in this registry the moment
   it is agreed, and awaiting its result is an explicit open task. A chip that
   isn't tracked can't be coordinated, and an untracked chip is a forgotten
   chip.

---

## Registry files

| File | Role |
| --- | --- |
| `registry.jsonl` | Machine-readable source of truth — one JSON object per line. |
| `registry.md`    | Human-readable table — **regenerated** from the JSONL on every change. Never hand-edit. |
| `chip.py`        | Stdlib-only helper that reads/writes the JSONL and regenerates the table. |

### Field schema

Every chip row has exactly these fields:

| Field | Meaning |
| --- | --- |
| `id` | Unique chip identifier, e.g. `chip-ec03-time-cost`. Convention: `chip-<short>-<slug>`. |
| `agreed` | Date the chip was agreed in chat, `YYYY-MM-DD`. |
| `intent` | One-line description of the chip's bounded task. |
| `branch` | The branch/worktree this chip OWNS (point 2). |
| `base_sha` | The SHA the chip branched from. If unknown, say so (e.g. `unknown (branched from <repo> main)`). |
| `produced_sha` | The SHA the chip produced on completion. Empty until it reports back. |
| `status` | Lifecycle status — see below. |
| `await_plan` | How and when the chip's result is awaited and reconciled (point 4). |

### Status lifecycle

```
proposed  ->  agreed  ->  spawned  ->  awaiting  ->  reconciled
```

| Status | Meaning |
| --- | --- |
| `proposed`  | Suggested in chat, not yet agreed. (Optional starting state.) |
| `agreed`    | Explicitly agreed in chat (point 1). Branch chosen (point 2). |
| `spawned`   | The chip session has been launched. |
| `awaiting`  | The chip is running; we are explicitly awaiting its result (point 4). |
| `reconciled`| The chip reported back and its `produced_sha` has been merged/verified into the target branch. Terminal state. |

---

## How to add / update a row

Use `chip.py` — it keeps `registry.jsonl` and `registry.md` in sync. (On this
rig Python is `python`; elsewhere it may be `python3` or `py`.)

**Add a chip the moment it is agreed:**

```sh
python chip.py add \
  --id chip-ab12-short-slug \
  --agreed 2026-06-15 \
  --intent "one-line description of the bounded task" \
  --branch chip/ab12-short-slug \
  --base-sha <sha-it-branches-from> \
  --status agreed \
  --await-plan "merge into <branch> once it reports; verify green"
```

`--agreed` defaults to today and `--status` defaults to `proposed` if omitted.

**Advance its status as it moves through the lifecycle:**

```sh
python chip.py update chip-ab12-short-slug --status spawned
python chip.py update chip-ab12-short-slug --status awaiting
python chip.py update chip-ab12-short-slug \
  --status reconciled --produced-sha 1a28e35 \
  --await-plan "reconciled into <branch>; verified green"
```

**View the registry:**

```sh
python chip.py list
```

After any change, commit **both** `registry.jsonl` and `registry.md` together.

---

## Scope guardrail

This repo tracks chips only. A chip never runs in here — it runs in whatever
repo its task targets, on its own branch there. Do not point a chip at a branch
another session is editing, and never modify an unrelated repo from a chip
without that being its agreed, self-contained instruction.
