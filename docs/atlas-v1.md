# Atlas v1 — from one long notebook to a named flow

v1 for Tue 3 Nov. Atlas (`atlas check`, `atlas view`, `atlas serve`) run on a copy
of this repo at commit `30044a6` (31 Oct 2026).

## 1. The current route: analysis 19 (*who writes like whom*)

My data reaches the final chart in two routes:

1. **`main --device`** (`wa_analyzer.preprocess`): reads `config.toml`, processes the
   WhatsApp export and writes a CSV to `data/processed/`. About 6 steps.
2. **`notebooks/analysis/15-who-writes-like-whom.ipynb`**: from the processed file to
   the PNG. Atlas shows **207 steps in one flat list**:

| My stage | Atlas steps | What happens |
|---|---|---|
| load | 5 | `load_own_chat` reads `config.toml` and `data/processed/` itself |
| clean | 1–4, 6–9 | regexes for links and the edit marker; `Pipeline` + `CleanForStyle` |
| features | 10–18, 26 | 1,500-character stretches, character trigrams, distances |
| checks / statistics | 19–191 | silhouette, PCA/t-SNE sketches, perplexity sweep, word readback, 20 draws |
| plot | 192–207 | the final chart, saved in the notebook folder |

The notebook also reads `../../data/processed/author-groups.csv` and `author-city.csv`
itself, outside `load_own_chat`.

*[picture: Atlas view of the analysis 19 route]*

## 2. What Atlas shows me

- **The stages are not separated.** Load, clean, features and plot all live in one
  notebook, without named steps, so I cannot run, test or reuse one stage on its own.
- **Most of the route is checks, not pipeline** (steps 19–191). They belong together in
  one place, apart from the steps that make the chart.
- **The data comes in through three doors and leaves wherever a cell saves it.** It
  should be loaded centrally from one location; the script then decides how the data
  continues to a PNG and where that PNG goes.

`atlas check` found 37 questions. The ones I act on are the ones above:

- *places* (4×): library code decides itself where data lives (`load_own_chat`,
  `data/processed/`) → rule 2.
- *doors* (1×): `wa_analyzer` names nothing a user may import → rule 1 (`main`).
- *gives* (32×): functions without a return type, so Atlas cannot follow the route →
  rule 4. I leave the 25 in the analysis notebooks as they are: those notebooks are
  replaced by the new repo after the report.

## 3. Design rules for the new repository

1. **Each stage is its own named function** — load, clean, features, plot — and
   `main.py` calls them in order.
2. **The caller decides where data lives.** `main.py` reads `config.toml` once and
   passes every step its paths; no function picks a path itself. A test can then hand
   `clean()` a tiny fake file instead of my real chat.
3. **Checks and statistics live in their own module**, apart from the pipeline, and are
   not run by `main.py`: they were how I *decided* a chart, not how I make it.
4. **Every step function says what it returns**, so Atlas can follow the whole route.
5. **One style module for every chart:** colours, font, font sizes and background, so all
   charts look like one report.
6. **Raw data is never changed.** Each stage reads the previous folder and writes its
   own: `raw/` → `processed/` → `features/`.
7. **Every choice is a setting in `config.toml`** (window size, perplexity, seed, date
   ranges), not a number buried in code.
8. **A rerun gives the same result and overwrites, never adds:** fixed seeds and fixed
   output names, so `img/` holds one current version of each chart.
9. **Private data stays out of git, and tests use fake data.** `data/` is in
   `.gitignore`.

## 4. The planned flow, sketched in Atlas

The new flow as signatures only (not built yet), drawn by Atlas next to the current code:

```
config.toml ─▶ load_config ─▶ clean ─▶ add_features ─▶ load_frame ─▶ plot_who_writes_like_whom ─▶ img/*.png
                 (paths,       raw/ →    processed/ →    features/ →    + Style
                  settings)    processed/ features/      dataframe
```

`main.py` is the only caller that reads `config.toml`; `checks.py` sits beside the flow
and is run by hand. `clean()` reads the WhatsApp export and cleans it in one step: the
separate preprocess command of route 1 goes away. Atlas draws one route of five named steps instead of 207.

*[picture: Atlas vision of the planned flow]*

## Next

- 11–17 Nov: build the new repository along these rules, running `atlas check` after
  each step.
- Final Atlas picture: the new repository, next to the "before" above.
