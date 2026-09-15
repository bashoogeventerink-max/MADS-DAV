# Feedback on the election-window comparing-categories chart

Feedback received on `slide-attractors-detractors-pct.png` (and by extension
`slide-two-authors-drive-it.png`) — the "attractors and detractors" chart built
for class from `02-election-length.ipynb`. Logged here to pick back up later,
not acted on yet.

## 1. Too much happening in one graph — simplify

Current chart stacks a lot of decisions into one image: 9 bars, a 3-way
colour threshold (attractor / neutral / detractor), a sample-size annotation
per bar, a headline + italic subtitle, and a legend explaining the colour
cutoff. Individually each was a deliberate, logged choice (see
`analysis-log.md`, "Presentation draft" section under Analysis 4) — but
together it's reportedly too dense for a reader to take in quickly.

**To think about when revisiting:** what's the one thing this chart needs to
say, and what can move to speaker notes / the write-up instead of living on
the chart itself? Candidates to cut or move: the sample-size annotations
(could go in a caption instead of on every bar), the 3-way threshold legend
(if only highlighting 1-2 people per point 4 below, a threshold-based
legend may not be needed at all).

## 2. Title should state the finding, not ask a question

Current title: *"How do political elections affect group chat activity per
user?"* (with *"Election windows create attractors and detractors..."* as
subtitle). Feedback: the chart already has a clear, specific answer — some
individuals are more active during election windows than outside them — and
the title should say that directly rather than pose the general question.

**Suggested direction:** something like *"`pliable-tiger` becomes more
engaged during election windows"* — naming the actual finding (and the
specific person it's about), not a generic framing question. This also
implies the chart itself should probably center on that one person's story
rather than all 9 people at once (ties into point 4).

## 3. Who is `pliable-tiger`, really? — resolved

Looked up from the untouched intermediate CSV (`data/processed/whatsapp-20260912-222116.csv`,
which holds real, pre-anonymisation names) and confirmed via `humanize()`,
which is a deterministic SHA1 hash — same input string, same alias, every
time:

- `pliable-tiger` = **Thijs Spin** (largest attractor, +38%)
- `rib-tickling-curlew` = **Jop Van Der Woning** (second-largest attractor, +34%)
- `vibrant-barracuda` = **Mark Schopman** (largest detractor, −30%)

**Gotcha hit along the way, worth remembering:** `load_own_chat()` always
reloads whatever `config.toml`'s `current` points at. After the first run,
that's already the *anonymised* parquet — so re-running
`01.3-your-own-chat.ipynb` doesn't re-anonymise real names, it re-hashes the
*aliases themselves* (`humanize("pliable-tiger")` → `"radiant-zebra"`, a
dead end). This happened once already tonight; nothing was lost because the
original parquet and the intermediate CSV were both untouched, but don't
rely on re-running that notebook to look up more names — go straight to the
intermediate CSV instead.

**Still to do (you):** add real context for *why* Thijs and Jop might be
more engaged, and Mark less, during election windows (politically-adjacent
job, known for being opinionated in the group, or the opposite — genuinely
uninterested in politics). This context turns "an anonymous author code
moved" into an actual story.

**Careful before it goes anywhere public:** re-check the audience/sharing
plan from stage 1 (course group + teacher, possibly shown to the friends
themselves) before real names appear in anything committed or shown in
class — naming someone's real political engagement is more sensitive than
an anonymous code. This file itself is a private working note, not meant to
be shared as-is.

## 4. Highlight only 1-2 people, grey out the rest

Revert from the current "every bar coloured by its own sign" (attractor vs.
detractor vs. neutral, all 9 bars) back toward highlight-one-and-grey-the-rest
— colour `pliable-tiger` (and maybe `rib-tickling-curlew`) directly, leave
the other 7 (or 8) grey.

**Worth remembering why the multi-colour version happened in the first
place**, so this isn't a step backward without reason: the original
highlight-only-2 draft (`slide-two-authors-drive-it.png`, absolute-numbers
version) was critiqued as implying "only these two changed, everyone else
didn't" — which wasn't true (`fluffy-beaver` and `vibrant-barracuda` also
moved by a real amount). The attractor/detractor reframe was a deliberate
fix for that honesty problem, not an arbitrary choice.

**So if going back to highlight-only-1: decide explicitly** whether that's
because the *title's claim* is now narrower (just about `pliable-tiger`
specifically, not a group-wide attractor/detractor pattern), which would
make the highlight choice honest again — versus simplifying in a way that
re-introduces the "only this one changed" false impression. If the latter,
consider a one-line caption acknowledging the other movers even if they're
not the chart's focus.

## 5. Add standard errors (or another statistical test) to the % change

Right now each bar is a point estimate (% change in messages/week) with no
uncertainty shown — the `n=` labels hint at sample size but don't say how
much a given % could plausibly vary. Feedback: add error bars (standard
error, or a confidence interval) or another statistical test, so a reader
can see whether e.g. `hypnotic-rabbit`'s −29% (on a small n=117) is
actually distinguishable from noise, versus `pliable-tiger`'s +38% (on a
much larger n=619).

**To think about when revisiting:** what's the right unit for the error
bar here — the underlying quantity is a rate (messages/week) built from a
count over a fixed number of weeks per author per period, not a simple
per-message measurement, so a standard "mean ± SE of individual
observations" framing doesn't directly apply. Options worth considering:
treating message arrivals as a Poisson/count process (a natural fit for
"messages per week" rates, with variance related to the count itself), or
a bootstrap over weeks within each period. `scripts/plots.py` already has
`BarPlotWithError` (built in lesson 2.4) for drawing bars with pre-computed
intervals, so the plotting side is likely already there — the open
question is how to *compute* the interval correctly for a rate built from
counts, not how to draw it.

## 6. Permutation/shuffle test: would a randomly-placed window look this strong?

Directly the stage-6 verification shuffle-test that was judged "by eye,
not computed" back when this analysis was first done (see
`analysis-log.md`, Analysis 4 stage 6) — now wanted as an actual computed
check rather than a gut estimate. Idea: repeatedly pick a random ~60-day
window (matching the real election windows' length) from somewhere else in
the ~6-year chat history, recompute the same per-author % change against
that random window vs. the rest of the chat, and see how often a random
window produces a swing as large as the real election windows do —
especially for `pliable-tiger`/Thijs and `rib-tickling-curlew`/Jop's
attractor pattern and `vibrant-barracuda`/Mark's detractor pattern.

**To think about when revisiting:**
- How many random windows to draw (100? 1000?), and whether to draw them
  with replacement / allow overlap with each other or with the real
  election windows.
- Whether "as strong" means matching the same *direction* (e.g. only count
  a random draw as matching if it also shows Thijs going up, not just any
  large swing in either direction) or just the same magnitude.
- This test could produce an actual p-value-like statistic (e.g. "X% of
  random windows showed a swing this large for this author"), which would
  also partly address item 5's uncertainty question — the two might end up
  sharing the same simulation code.
- Same caveat already named in the analysis-log's Analysis-4 verdict: this
  chat's overall activity has a known general decline over the later years,
  so random windows drawn from different parts of the timeline aren't
  fully comparable to each other either — worth deciding whether random
  windows should be restricted to a similar era as the real elections, or
  whether that's exactly the kind of confound this test is supposed to
  catch.

## Open questions for next session

- Does simplifying mean dropping the % chart in favour of the absolute one,
  or simplifying the % chart itself?
- Does the title change mean the chart's *scope* changes too (one person,
  not nine), or just the headline?
- Should the sample-size context (`n=619` etc.) move to a caption/footnote
  instead of per-bar labels if the chart is narrowed to 1-2 people anyway?
- Order to tackle items 1-6 in — simplification/title/highlighting (1, 2,
  4) change what the chart shows; the statistical work (5, 6) changes
  whether what it shows can be trusted. Worth agreeing which comes first
  before starting, since a redesign done before the stats land might need
  redoing once the stats are in (e.g. if the shuffle test undercuts the
  Thijs/Jop pattern, item 2's title would need to change again).
