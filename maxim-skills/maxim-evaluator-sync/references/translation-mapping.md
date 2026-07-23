# Translation mapping: eval_<name>.md → Maxim AI evaluator config

Front-matter keys read by `translate.py`:

| Key | Required | Meaning |
|---|---|---|
| `evaluator` | yes | evaluator name (snake_case) |
| `model` | yes | judge model id; `deterministic`/`none` ⇒ skipped (non-AI, out of scope) |
| `scoring_type` | yes | `numeric_scale`→Scale, `number`→Number, `binary`→YesNo, `categorical`→Multiselect |
| `provider` | no | else inferred: claude/opus/sonnet/haiku→anthropic, gpt/o1/o3→openai, gemini→google |
| `scale_min` / `scale_max` | no | else parsed from body `Scale: A-B`, else 1..5 |
| `pass_threshold` | no | entryLevel value; default = round(0.8·scale_max) for Scale, `Yes` for YesNo |
| `run_threshold` | no | runLevel value; default 0.8 |
| `description`, `tags` | no | passed through |

The markdown body (persona, rubric, edge cases, calibration, bias) becomes `instructions`, with an appended inputs block referencing `{{input}}`, `{{output}}`, `{{context}}`. Any other `{{var}}` becomes an `optionalVariable` (reserved input/output/context/expectedOutput excluded).

To adjust a mapping, edit the front-matter and re-run `plan`.
