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

**Resolved by the v3 redesign** (see `analysis-log.md`, "Presentation draft
v3"): statistics went first (SE + shuffle test), the chart narrowed to
`pliable-tiger` alone with a real story, all 9 authors stayed visible in
grey, and `n=`/SE moved off the other 8 bars onto the focus bar only.

## Follow-up items after v3, parked for a future session

Logged after reviewing the v3 chart (`slide-pliable-tiger-election-effect.png`)
— not acted on yet, same as items 1-6 above when they were first written down.

7. **Neaten the title. — resolved.** Real-name/audience question (item 3)
   still not settled, so the alias stays; shortened to *"pliable-tiger gets
   more active during election windows"* — states the finding directly
   (matching item 2's original direction) instead of the punchier-but-long
   "historian... awakes and activates" draft.
8. **Pick one of the x-axis label or the subtitle, not both. — resolved.**
   Neither the axis-level subtitle nor the x-axis label survives on the
   chart now — both were dropped, and the metric description they used to
   split between them (the % change definition, and "5 windows over a
   6-year period") moved into the footnote instead. One place, not two.
9. **Simplify the footnote wording. — resolved.** Cut to the first sentence
   only ("An error bar shows the range the true value could plausibly fall
   in, just from week-to-week randomness."); the second sentence (spelling
   out how to read a specific bar's whisker) was dropped as more than a
   class audience needs on the slide itself. The footnote now also carries
   the metric description that moved out of the subtitle/x-axis label
   (item 8), so it reads as one combined line rather than two separate
   pieces of chart text.
10. **Rethink what `n=619` on the `pliable-tiger` bar should mean. —
    resolved.** Kept the `n=`, dropped the bare number: the label now reads
    `n=619 election-window messages` so a reader doesn't need to already
    know the chat's scale to judge whether 619 is a small or large sample.
    No rest-of-chat `n` added alongside it — the error bar already carries
    that reliability information visually, per the original note.

## Follow-up round 2: title focus and footnote formatting

11. **Title now leads with the "historian" backstory — resolved, with a
    scope decision.** Title changed to *"pliable-tiger, the group's
    historian, gets more active during election windows"*, making the real
    behavioural story (not just the alias) the focus, per the student's
    request. **Deliberately left out of the title:** the other half of the
    same backstory — former political-party membership — was requested too,
    but decided (with the student, after a direct trade-off question) to
    keep off the visible chart for now. Reasoning: a political affiliation
    is a more specific, more identifying fact within a 9-person friend group
    than "historian" is, and item 3's audience/sharing-plan check is still
    unresolved. The political-party detail stays where it already lived —
    the private v3 markdown cell in `02-election-length.ipynb` — not on
    anything that gets shown in class. Revisit if item 3 gets resolved.
12. **Footnote reformatted as bullet points — resolved.** The combined
    metric-definition + error-bar sentence from item 9 is now two stacked
    `•`-prefixed lines instead of one run-on sentence, left-aligned under
    the chart.

## Follow-up round 3: second highlighted author, from teacher-feedback critique

Surfaced while walking the current chart through `goad_critique_visual`'s gestalt
section against the teacher's original feedback — `rib-tickling-curlew`'s +34% was
almost as extreme as `pliable-tiger`'s +38%, but grey grouped it with the flat bars
purely by colour, fighting the eye's own read of the magnitudes.

13. **`rib-tickling-curlew` promoted to the same highlight as `pliable-tiger` —
    resolved.** Confirmed with the student first that "history fanatic" is lived
    knowledge about Jop (rib-tickling-curlew's real name, per item 3) too, not an
    assumption invented to fit his bar being high — same evidentiary bar
    pliable-tiger's "historian" story was held to. Both bars now share the blue
    highlight colour; the other 7 stay grey.
14. **Both highlighted bars get the same `n=`/SE label** — same format as before,
    applied to `rib-tickling-curlew` too (`+34% (n=515 election-window messages,
    +-7 SE)`).
15. **Title generalised to both people.** Changed to *"The group's history
    fanatics get more active during election windows"* — no longer names an
    alias directly (both aliases are already the y-axis tick labels), states the
    now-two-person finding. Political-party detail from the earlier backstory
    still excluded — item 3's audience/sharing-plan question is about the trait's
    identifiability, not about how many people it's currently attached to.

## Follow-up round 4: the five guidelines, applied to the two-author version

Walked `goad_critique_visual`'s guidelines section against the current chart.
Show-the-data, avoid-spaghetti, and start-with-grey all held already; two items
acted on:

16. **On-bar labels shortened — resolved.** `+38% (n=619 election-window
    messages, +-6 SE)` ran too long on the bar itself. Cut to just `+38%` /
    `+34%`; the `n=`/SE detail moved to a new third footnote bullet
    ("Election-window sample sizes: pliable-tiger n=619 (+-6 SE),
    rib-tickling-curlew n=515 (+-7 SE).") instead of crowding the bar.
17. **Legend replaced with a direct annotation — resolved.** The
    "error bar: ±1 SE (Poisson)" legend box (a lookup) is gone; replaced with an
    `ax.annotate` callout pointing straight at the whisker of the first grey bar
    (`fluffy-beaver`) — chosen because it's explaining the error-bar concept in
    general, not something specific to either highlighted author, so it's
    anchored away from the two focus bars on purpose.

## Follow-up round 5: the claim section, and a subtitle/footnote swap

Walked `goad_critique_visual`'s final `claim` section. **Not fully resolved —
flagged, not fixed.** The student's stated claim ("people more interested in
politics/history show this interest in the chat and chat more, probably with
each other") bundles two things: (a) message *rate* going up for these two
people — measured, shown directly — and (b) that the extra messages are
actually *about* politics/history and represent mutual conversation — **not
measured by this chart or anywhere in this analysis's pipeline** (no
content/keyword check like Analysis 2's `is_planning` feature, no reply/mention
network). The student named the exact falsifier themselves in Q2 ("topic isn't
politics-related") and named topic data as what's "deliberately not shown" in
Q5 — meaning the claim's central mechanism is the untested part. This directly
overlaps `Analysis 4`'s own already-logged, unresolved alternative: "already
high-volume authors post more during *any* notable/busy period," not something
election-specific. **Left open, two options put to the student:** narrow the
title's claim to what's shown (activity/volume only), or spend time actually
testing the topic angle with a keyword check on the *extra* election-window
messages. Not decided yet.

18. **Subtitle restored, footnote trimmed further.** Independent styling
    request: the metric-definition bullet moved back out of the footnote into
    an italic subtitle directly under the headline (`ax.set_title`, matching
    the original v1/v2 headline+subtitle pattern) — layout tightened
    (`pad=6`, `rect=[0, 0.09, 1, 0.93]`) after the first attempt left too much
    blank space between subtitle and plot. Footnote now carries only the
    error-bar explanation and the sample-size bullet from item 16.
19. **"(5 windows over a 6-year period)" moved out of the subtitle into its
    own footnote bullet, spelled out.** Subtitle shortened to just the metric
    definition. New first footnote bullet: *"The election windows are based on
    5 periods of elections (3 within NL, 2 in USA) over a 6-year period."* —
    names which elections these actually are instead of just a bare count.
