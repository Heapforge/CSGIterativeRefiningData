# CSGIterativeRefiningData

The attempt store of CharacterStrokeGenerator (`strokegen`): every version ever made of
every glyph's stroke decomposition, good or bad, with what produced it, what the checker said about it and what
a reviewer decided. Model output, fix-up scripts, LLM repairs, human edits in the review dashboard and refiner
rounds all land here. Approved glyphs are the end result; the versions that led to them are the training data
for the iterative refiner (`strokegen.refine_dataset` turns every earlier version of an approved glyph into a
pair with its approved version).

This repository is a git submodule of CharacterStrokeGenerator at `training_data/attempts/`. It holds only data;
the code that reads and writes it is `strokegen/attempts/` in the parent repository.

## Layout

```
log/<font>/<U+XXXX>.jsonl      events of one glyph (font + character), append-only    tracked
log/_checkpoints.jsonl         model checkpoints, in the order they first produced a version   tracked
index.sqlite                   query index over log/                                   derived, gitignored
staging/<font>/<U+XXXX>.json   the version to work from, per glyph not yet approved    derived, gitignored
```

`<font>` is the font's export name (as in `training_data/font_glyphs/` of the parent repository). The file stem is
the character's code points, joined by `_` when the glyph name has several (`U+6C38.jsonl` is 永).

Only `log/` is the store. `index.sqlite` and `staging/` are rebuilt from it at any time, so after a pull run

```sh
uv run python -m strokegen.attempts stage      # sync the index, rebuild staging/
uv run python -m strokegen.attempts reindex    # or: forget the index and rebuild everything
```

## Rules

- **Append only.** Never edit, reorder or delete lines in `log/`. A correction is a new event: a new version, a
  new check, a new verdict. The latest check and the latest verdict of a version are the ones that count.
- **Merge conflicts:** keep both sides' lines and delete the conflict markers. Order does not matter, since
  events are read by timestamp; the indexer skips lines that are not JSON, so a leftover marker loses nothing.
- **Approval is human.** A version becomes `approved` only through the dashboard's Save button, after a person
  looked at it. Scripts and agents may add versions, checks, rejections and `unresolved` notes, never approvals.
- **Outlines are never modified.** Every version is cut from the font's own outline segments, stored once per
  glyph in its `glyph` event.

## Events

One JSON object per line, keys sorted, UTF-8. Every event has a `type`.

### `glyph`

The outline a glyph's versions refer to, written before its first version (again only if the font's outline
changes).

| field | |
|---|---|
| `font`, `char` | which glyph |
| `glyph_key` | the outline hash (`build_dataset.outline_hash`); also decides the train/val/test split |
| `outline` | contours as `[[kind, points], ...]`, kind `L`/`Q`/`C`, hanzi-writer coordinates (1024 em, y up) |
| `at` | ISO 8601 UTC timestamp |

### `attempt`

One version of a glyph.

| field | |
|---|---|
| `attempt_id` | `<ms since epoch, hex>-<random hex>`: time-ordered, never collides across machines |
| `parent_attempt_id` | the version this one was made from, or `null` |
| `font`, `char`, `codepoint`, `script` | `script` is `cjk`, `latin`, `greek`, `cyrillic` or `other` |
| `glyph_key` | the `glyph` event this version is cut from |
| `source_type` | what made it (see below) |
| `source_ref` | how to find exactly what made it (see below) |
| `hw` | the hanzi-writer data: `{"strokes": [svg path, ...], "medians": [[[x, y], ...], ...]}` in writing order |
| `labels` | the same version in segment-label space (see below) |
| `content_hash` | sha1 of `hw`; identical content from the same producer is stored only once |
| `tags` | error tags the producing pass addressed |
| `created_at` | when the version was made |

`labels` makes any two versions of a glyph directly comparable:

| field | |
|---|---|
| `seg_stroke` | stroke index of every outline segment (flattened over contours), `-1` if unmatched |
| `n_strokes`, `order` | stroke count; strokes are already in writing order, so `order` is the identity |
| `flip` | per stroke, whether its median runs against the canonical direction |
| `medians` | the medians, rounded to integers |
| `bridges` | per stroke, the segments that close it: `[kind, points, from, to]` |
| `vetoed` | segments in merges the decoder's bridge validator refused (`model` / `refiner` only, else `null`) |
| `match` | how well `hw` maps onto the outline: `coverage`, `unassigned`, `n_bridge_bad`, `n_bridge_on_edge` |

Model and refiner versions carry their own exact segment labels; all others are mapped onto the outline with
`build_dataset.decompose`, the matcher that builds the training records.

#### Source types

| `source_type` | made by | `source_ref` |
|---|---|---|
| `model` | `StrokeGenerator`, single pass (dashboard) | `cut_sha`, `order_sha` (sha256 prefix of the checkpoint files), `cut_epoch`, `order_epoch`, `cut_feat_version`, `order_feat_version`, `decode` (the decoder's parameters) |
| `refiner` | a refinement round (`--refine-iters`) | the `model` fields plus `refine_sha`, `refine_epoch`, `refine_version`, `refine_iters`, `round` |
| `fixup_script` | `attempts import --source fixup_script` | `script` (repo path), `git_sha` (HEAD), `blob_sha` (the file as run), `dirty`, `args` |
| `llm_repair` | `attempts import --source llm_repair` | `llm_model`, `run_id`, `notes` |
| `human_edit` | dashboard Save of content no stored version has | `reviewer`, `edits` |
| `imported` | `attempts backfill-hwdata` | `repo`, `commit` (or `worktree`), `committed_at` |
| `synthetic` | reserved; `refine_dataset` currently generates its synthetic pairs on the fly without storing them | |

Fix-up scripts live in the parent repository under `tools/fixups/<run_id>/` and are never edited after they
produced imported versions, so `script` + `blob_sha` always points at the code that made a version.

### `check`

A checker result for a version (`tools/verify_glyph.check_glyph`: union IoU ≥ 0.99, every median point inside
its stroke, median coverage ≥ 0.9, no stroke hanzi-writer leaves undrawn).

| field | |
|---|---|
| `attempt_id`, `at` | |
| `checker.passed`, `checker.iou` | |
| `checker.failures` | glyph-level failures (`low_iou`, `stroke_median_count`, `checker_error`) |
| `checker.strokes` | per-stroke rows with their own `failures` (`short_median`, `median_outside`, `low_coverage`, `hidden`) |
| `checker.deficit_segments` | outline segments next to area the strokes miss or add |
| `checker.version` | `CHECK_VERSION`; `attempts check --stale` re-runs results from older checkers |

### `verdict`

| field | |
|---|---|
| `attempt_id`, `at`, `by` | |
| `verdict` | `pending`, `approved`, `rejected`, `superseded` or `unresolved` |
| `reason` | required to reject a version the checker passed, and to mark one `unresolved` |
| `tags` | error tags |

A version without a verdict is `pending`. Approving a version supersedes the glyph's other pending, unresolved and
approved versions, and the approved versions of the same character in other fonts (whose training file it
replaces). Rejected versions stay rejected.

A **glyph** is `approved` once any of its versions is, otherwise `pending` or `unresolved` by its newest live
(pending, unresolved or rejected) version, which is the one staged. A rejection that passed the checker is the
most useful kind of record: it is wrong in a way only a person could see, and its `reason` says how.

### `checkpoint` (in `log/_checkpoints.jsonl`)

`sha`, `at`, `kind` (`cut`, `order`, `refine`), `epoch`, `val`, `feat_version`, written the first time a checkpoint
produces a version. Their order sets the recency weight of refiner training pairs (older checkpoints count less).

## Staging

`staging/<font>/<U+XXXX>.json` exists for every glyph without an approved version. It holds the version to work
from (`attempt_id`, `strokes`, `medians`, `segStroke`, the latest `checker` result, `verdict` and `reason`) and a
summary of the glyph's whole `history`, including reviewers' rejection reasons. It is what the dashboard's Staging
tab lists and what `attempts export` hands to a repair pass.

## Working with the store

From the parent repository (`uv run python -m strokegen.attempts --help` for every option):

```sh
attempts export --font F [--failing] -o queue.json     # staged glyphs, the fix-up tools' input format
attempts import out.json --font F --source fixup_script --script tools/fixups/RUN/x.py --tags ...
attempts import out/ --font F --source llm_repair --llm-model M --run RUN
attempts reject CHAR --font F --reason "..."            # a version is wrong
attempts unresolved CHAR --font F --reason "..."        # give up on a glyph for now, and say why
attempts history CHAR [--font F]                        # every version of a glyph
attempts stats --by font|script|source|fixup            # how glyphs got approved
attempts check [--stale]                                # check versions without (current) results
```

The repair-pass workflow (run ids, notes, recording every attempt) is in the parent repository's `CLAUDE.md`
under "Repair passes".

## License

MIT, see [LICENSE](LICENSE).
