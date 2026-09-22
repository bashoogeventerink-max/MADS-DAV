# Feedback on the football-tournaments time-series chart

Feedback received on `football-tournaments-activity.png` (built from
`03-football-tournaments.ipynb` / `03.3-events-in-your-chat.ipynb`, Analysis 5
in `analysis-log.md`) — the three-panel small-multiples chart showing daily
message volume around Euro 2020, World Cup 2022, and Euro 2024. Logged here to
pick back up later, not acted on yet.

## 1. The trend isn't strong across all tournaments

Feedback: looking at the three panels side by side, the rise during `during`
isn't equally convincing in all three — Stage 6's verdict called it "holds,
3/3" based on the numbers (10.1 baseline vs. 18.9/21.0/19.9), but visually the
strength of the effect doesn't read as uniform panel to panel.

**To think about when revisiting:** this may be a chart-reading issue rather
than a numbers issue (three separate small y-axis panels make it harder to
compare relative rise than an overlay would — see item 2), or it may be a real
signal that one tournament's rise is weaker/noisier than the other two and the
single "holds 3/3" verdict is smoothing over that. Worth checking the
per-tournament numbers again with this specific question in mind before
redesigning.

## 2. Overlay tournaments on one graph, aligned by tournament day (t=0)

Feedback: to make the "football tournaments bring activity" claim land more
clearly, put multiple tournaments on **one** graph instead of three separate
panels — aligned not by calendar date but by **tournament day**: `t=0` =
start of tournament, `t+1` = second day, etc. This turns the x-axis from three
different calendar ranges into one shared "days since kickoff" axis, so the
three lines can be compared directly on top of each other.

**To think about when revisiting:**
- Needs a new derived column (`day_offset` or similar) per tournament: for
  each row, `(date − tournament_start).days`, computed separately per
  tournament window using the existing reference table in `analysis-log.md`
  Stage 2 (`Euro 2020`, `World Cup 2022`, `Euro 2024` start/end dates).
- Decide what `t=0` means exactly — official start date (as currently
  defined) or first match date, if those differ.
- Windows are different lengths (Euro 2020: 30 days During, World Cup 2022:
  28 days, Euro 2024: 30 days) — decide whether to truncate to the shortest
  common length or let lines end at different `t` values.
- This is a genuinely different chart family choice than the current
  small-multiples layout that Stage 4 picked (`FacetPlot`, "average away the
  window-by-window detail" was the reason multiples were chosen over one
  pooled view) — an overlay reintroduces some of that averaging-together risk
  unless each tournament stays a separately-coloured line rather than pooled
  into one.
- Whether to keep the "vs. this chat's overall daily average" baseline
  reference line from the current chart, and how it fits on a `t`-aligned
  x-axis (it would be a flat horizontal line, not tied to any one
  tournament's calendar dates).

## 3. Add Netherlands match days as dashed vertical lines

Feedback: does the Dutch national team's own matches specifically drive more
activity than the tournament in general — especially the World Cup 2022
Netherlands–Argentina match, called out as a likely special case (well-known
as a highly-watched, dramatic match). Suggestion: mark individual NL match
days as dashed vertical lines/bars on the chart.

**To think about when revisiting:**
- **No match-level data exists yet anywhere in this pipeline.** The current
  reference table (Stage 2) only has tournament-level start/end dates, not
  individual match dates, opponents, or results — this needs a new small
  static lookup table (same pattern as the tournament/lockdown/election
  tables already in `analysis-log.md`), one row per NL match: date,
  opponent, stage (group/knockout), outcome.
- Decide the falsification/comparison the way earlier analyses did: is the
  claim "NL match days show a bigger spike than other days within the same
  tournament window" (a within-tournament comparison) — that needs the
  non-NL-match `during` days as the baseline, not the pre-tournament
  baseline.
- The Netherlands–Argentina match (World Cup 2022, quarter-final, decided on
  penalties) is explicitly named as expected to be the largest single spike
  — flagged as the standout case to check first once match dates are added,
  same "one dramatic day" caveat pattern already used for the deferred
  wedding-announcement candidate in Analysis 5 Stage 1 (single-day
  sample-size risk, not a sustained pattern).
- Dashed vertical lines will need to sit inside whichever chart layout is
  chosen from item 2 (three panels vs. one `t`-aligned overlay) — worth
  deciding item 2's layout first, since match-day markers look different on
  a calendar-date axis vs. a `t`-since-kickoff axis.

## 4. Plot count minus daily average, not raw count, to control for seasonality

Feedback: instead of plotting raw daily message count, plot raw count minus
a daily-average baseline — a deseasonalized/anomaly view — so the chart
controls for seasonality directly rather than needing a separate confound
check.

**To think about when revisiting:**
- Decide precisely what "daily average" means here, since it changes what
  the subtraction actually controls for:
  - the single flat overall average (10.1 msgs/day) already used as the
    reference line on the current chart — subtracting this is just a
    y-axis shift, it does **not** control for seasonality (summer vs.
    winter baseline activity is still folded into the number);
  - a **seasonal baseline per calendar date** (e.g. average count on that
    same calendar date across the other available years) — this is the one
    that actually accounts for seasonality, and is very close to what
    Stage 6's already-built shuffle test compares against (same calendar
    dates in every other available year, excluding years overlapping a
    real tournament window). Plotting the residual (raw − this seasonal
    baseline) would essentially turn that existing verification check into
    the plotted quantity itself, rather than a separate confirmatory test
    reported in prose;
  - a smoothed seasonal curve (e.g. day-of-year rolling/harmonic fit)
    instead of a per-date lookup, if the per-date comparison years are too
    thin (Stage 6 already notes as few as 3 comparison years for Euro
    2020).
- If this reuses Stage 6's per-calendar-date comparison-year baseline,
  reuse that computation rather than rebuilding it — it already exists in
  `03.3-events-in-your-chat.ipynb`.
- Axis/label consequence: "messages per day" would become something like
  "messages per day, relative to a typical day this time of year" — needs a
  clear, non-technical label given the audience (course group + teacher,
  possibly the friends themselves).
- Decide whether this replaces the current raw-count-plus-rolling-average
  chart, or becomes an additional panel/version alongside it (the raw view
  is more intuitive at a glance; the deseasonalized view is the more
  rigorous one).
- Worth checking together with item 1 — deseasonalizing may sharpen or
  soften the "trend isn't equally strong across all three tournaments"
  observation, so it's useful to look at both together rather than fixing
  item 1 first on the raw-count view.

## 5. Check time-of-day, not just daily volume — are people chatting later?

Feedback: alongside (or instead of) message *count*, look at *when* during
the day people chat during tournament windows vs. other weeks. Idea to test:
people are active later in the day/evening during tournaments than usual.

**To think about when revisiting:**
- This is the same "daily rhythm, not overall trend" question already
  named as a genuinely separate, parked question in Analysis 1's Stage 4
  (there for lockdown vs. non-lockdown; here it's tournament vs.
  non-tournament) — same shape of check, different period definition. Worth
  reusing that framing rather than treating it as unrelated.
- Needs an hour-of-message feature (extracted from `timestamp`, not built
  yet in this analysis) and a distribution comparison (e.g. hour-of-day
  histogram or median/mean send-hour) for `during` tournament days vs.
  `baseline` days — a genuinely different chart family (distribution
  comparison) than the current time-series volume chart.
- **Caveat named directly by the student, must be handled explicitly:**
  World Cup 2026 is not currently in the tournament reference table at all
  (only Euro 2020, World Cup 2022, and Euro 2024 are listed — see Stage 2)
  even though it falls inside the Aug 2020 – Aug 2026 data window per
  Stage 1. If it's added for this check, its matches were played at night
  in European time (different host-country time zone than the other three
  tournaments, which were played at European-friendly hours) — so a later
  send-hour during WC 2026 could just reflect match kickoff time shifting,
  not a "people stay up later for football in general" effect. Any
  cross-tournament comparison of send-hour needs to either exclude WC 2026,
  or align by hours-relative-to-kickoff-time rather than by raw
  clock-hour, so the timezone difference doesn't get mistaken for the
  behavioural effect being tested.
- Before adding WC 2026 to the reference table for this or any other check,
  its start/end dates need to be looked up and added the same way the
  three existing tournaments were (Stage 2's static lookup pattern) —
  not done yet.

## Open questions for next session

- Does item 2's redesign replace the current three-panel chart, or does the
  three-panel version stay as a companion (per-tournament detail) alongside
  a new single overlay (cross-tournament comparison)?
- Order to tackle items in: item 2 changes the chart's whole layout, so
  probably comes before item 3's match-day markers are placed on it. Item 1
  might resolve itself once item 2's overlay makes the three tournaments
  directly comparable — worth checking before treating it as a separate fix.
  Item 4 changes the underlying *quantity* being plotted (deseasonalized
  vs. raw), which is independent of the layout choice in item 2 but should
  probably be decided before item 2 is built, so the overlay isn't built
  once on raw counts and then rebuilt again on the deseasonalized version.
  Item 5 (time-of-day) is a separate question/chart family from items 1-4
  (which are all about daily *volume*) — it can be worked on independently,
  but its World Cup 2026 timezone caveat needs deciding before that
  tournament is added to the reference table for *any* check, not just
  this one.
