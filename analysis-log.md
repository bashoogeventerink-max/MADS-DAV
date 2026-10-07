# Analysis log — friend group chat (`data/raw/_chat.txt`)

Running plan for the goad-coached analysis. Appended after every stage — this file,
not the chat, is the real first draft of the write-up.

## Stage 1 — Question

**Data:** WhatsApp export of a ~15-year friend group chat. Export itself only
covers **Aug 2020 – Aug 2026** — starts right around the first COVID lockdown,
so there is **no pre-COVID baseline** in the data.

**Known context (from lived experience, not from the data):**
- COVID restrictions in NL (2020–2021) likely affected chat behaviour.
- Post-COVID: people moved in with partners, started working, some went quieter
  (unsure who).
- Group humor is dry/sarcastic in person but reads flat/serious in plain text.
- Group has been through shared losses (family/friends) — can be emotionally
  supportive/serious.
- Some left/right political divide emerged around/after COVID.
- Some members moved away from hometown, others stayed close.

**Other curiosities (parked for later, not the stage-1 proposition):**
- Are some users less active over time?
- Are the "drivers" of the conversation changing?
- Are topics changing over time?
- Is the chat's purpose shifting from functional (planning) to purely social?

**Named boring/obvious results (to avoid mistaking for the finding):**
- Chat quiets down during work hours, picks up approaching the weekend / when
  planning a drink, more active in evenings, shorter messages during the day.

**Proposition (falsifiable, adjusted to fit the actual data window):**
> Chat activity was highest during the COVID lockdown periods (2020–2021,
> within the available data) and has shown a declining trend since
> restrictions lifted — rather than staying flat or increasing over time.

**Falsification:** would say "no" if activity is flat/noisy with no visible
lockdown-era peak, or if activity trends *upward* over the observed period.

**Origin:** lived experience / memory of being in the group, not from
skimming the export first.

**Audience:** course group members + teacher (MADS-DAV assignment), plus
wants to showcase to the friends in the group chat themselves (who helped
inspire the questions). Needs to be legible to a non-technical audience, not
just a technical one.

**Time budget:** ~1hr/day + 3–4hr on weekends.

## Stage 2 — Data (done)

- One row = one message. Fields: timestamp (Europe/Amsterdam timezone),
  author, message text after the `:`.
- **Decision: anonymize authors** (real name → `Person A`/`Person B`/... ,
  consistent mapping) before this is shared with the teacher, course group,
  or committed anywhere shared. Not done yet — first pipeline step.
- Provenance: timestamp/author/message are raw/measured, straight from the
  WhatsApp export — nothing WhatsApp-derived. **Caveat:** the file contains
  at least one system message at the very start (group created 2017, user
  added around the same time) — well before the Aug 2020 export window
  starts. Parsing needs to detect and separate system messages (group
  created/renamed/icon-changed, "added X", etc.) from real chat messages,
  not treat them as authored content.
- Unit of row (message) vs. unit of claim: differs by question.
  - Stage-1 proposition (activity over time) → claim unit is day/week
    (aggregated message counts). No pseudoreplication issue.
  - Parked curiosities (e.g. "are some users less active over time?") →
    claim unit is *person* (n = number of group members, not n = messages).
    Needs per-user aggregation, not row-level tests.
  - Also want groupings by unit: user, day-of-week, etc. for later slicing.
- Missingness: user believes the group has been stable over the whole
  export window (no one left/rejoined) — **needs validation**, not assumed.
- **Row loss (lesson 1, "1.4 What to write down" Q2):** checked via
  `logs/logfile.log` from the preprocessor run that produced
  `whatsapp-20260912-222116.csv`. Of 23,072 raw lines in
  `data/raw/_chat.txt`:
  - 22,517 became valid message rows (`Found 22517 valid records`).
  - 550 lines were **not** lost — they're continuation lines of a
    multi-line message with no new timestamp, merged into the previous
    row's text (`Appended 550 records`). Naively diffing raw-line-count vs.
    row-count would wrongly count these as loss.
  - 2 lines errored on timestamp parsing (logged as `ERROR` around lines
    16390–16391 of the raw file) and were dropped.
  - 3 lines were dropped with **no log line at all**: the system notices
    at the very start of the export (encryption banner, group-creation
    notice, "you were added"). These occur before any real message exists
    yet, so the parser has no prior row to append them to and silently
    skips them. Found by arithmetic (22,517 + 550 + 2 = 23,069; raw file
    has 23,072 → 3 unaccounted), not by reading a log line — a real gap in
    the preprocessor's logging, not just a data quirk.
  - **Total lost: 5 rows out of 23,072** (2 logged errors + 3 silent
    drops). Reproduce via `wc -l data/raw/_chat.txt` vs. the `Found N
    valid records` / `Appended N records` lines in `logs/logfile.log` for
    the matching run.
- **Number of Authors:**
  - 9 unique number of authors in this group chat. 

### Reference periods (for period-flag features — static lookup, not
recomputed per pipeline run; researched via web search, approximate —
validate against an authoritative timeline e.g. rijksoverheid.nl before
relying on exact boundaries)

**NL COVID lockdown/restriction windows (within Aug 2020 – Aug 2026 data):**
| Period | Start | End | Notes |
|---|---|---|---|
| Lockdown 2020–2021 | 2020-12-14 | 2021-04-28 | Full lockdown incl. curfew (curfew 2021-01-23 to 2021-04-28); gradual reopening from Apr 2021 |
| Partial tightening pre-lockdown | 2020-10-14 | 2020-12-13 | Evening closures, partial measures before full lockdown |
| Lockdown 2021–2022 | 2021-12-19 | 2022-01-14 | Hard lockdown (shops/hospitality closed); restrictions eased gradually through ~2022-02-25, travel restrictions fully lifted 2022-09-17 |

**Election periods (NL + US, window proposed as ±30 days around election
day — adjust if you'd rather use a different campaign-period width):**
| Election | Date |
|---|---|
| US presidential | 2020-11-03 |
| NL general | 2021-03-17 |
| NL general (snap) | 2023-11-22 |
| US presidential | 2024-11-05 |
| NL general | 2025-10-29 |

### Pipeline steps (candidates, confirmed)
1. Parse raw `.txt` → structured rows (timestamp, author, message), split
   out system messages from real messages.
2. Anonymize authors (consistent name → alias mapping).
3. Derive `date` / `week` / `day_of_week` from timestamp.
4. Join static `lockdown_period` lookup table (above) → categorical flag.
5. Join static `election_period` lookup table (above) → categorical flag.
6. Membership-consistency check: validate every author appears across the
   full date range with no long unexplained gap (flag any that don't).

## Stage 3 — Features

### Features added so far (`own_pipeline`, `notebooks/lesson1/01.3-your-own-chat.ipynb`)

All four steps run through `goad_toolkit.datatransforms` — `TimeFeatures` and
`RegexFeature` (the latter has exactly three `mode`s: `"count"`, `"has"`,
`"extract"` — `"extract"` pulls one capture group into a string column,
one row in/one row out, no row-exploding for multi-match text).

| Step | New column(s) | Mode | Question it's for |
|---|---|---|---|
| `TimeFeatures(column="timestamp")` | `day_name`, `isoweek`, `year_week` | — | Needed to aggregate by day/week for the Stage-1 lockdown-period proposition, and to check the named boring result (quiet work hours, weekend/evening pickup) so it isn't mistaken for the finding. |
| `RegexFeature(name="urls", pattern=r"https?://\S+", feature="has_url")` | `has_url` (bool) | `has` | Flags link-sharing messages — feeds the parked curiosity "is the chat's purpose shifting from functional (planning) to purely social?" |
| `RegexFeature(name="capital_letters", pattern="[A-Z]", feature="n_capital")` | `n_capital` (int) | `count` | **Added without a stated question yet** — added during the "add at least two more steps" exercise, not because a specific proposition needed it. Per the notebook's own Q4 ("a feature you cannot attach to a question is one you will never use"), this needs a real question attached (emphasis/shouting proxy? per-person tone comparison?) or it should be dropped before Stage 4. |
| `RegexFeature(name="number_of_words", pattern=r"\s+", feature="n_words")` | `n_words` (int, whitespace-count proxy — off by one from true word count, but consistent) | `count` | Same caveat as above — added as an exercise step, not yet tied to a named question. Candidate use: per-person or per-period message length/effort, but not decided.

### New curiosities (parked — not yet Stage-1-style propositions)

Media senders, emoji, and message/emoji-driven behaviour or sentiment. Checked
what the current toolchain actually supports before planning these, rather
than assuming:

- **Media-heavy senders — decided, ready to build.** No threshold: the
  interest is the shape of the distribution (per-author media count/share),
  not a cutoff for "a lot." Feasible now with the existing `RegexFeature`
  machinery, no parser changes: the Android export's placeholder text is
  literally `<Media weggelaten>` inside an otherwise normal `Name: message`
  line (e.g. `27-08-2020 11:17 - Jeroen Weda: <Media weggelaten>`), so
  `RegexFeature(pattern=r"<Media weggelaten>", feature="is_media", mode="has")`
  + group-by-author gives the per-person distribution directly.

### To-do (later, explicitly deferred — not this lesson's pipeline)

- **Emoji presence and emoji *type*.** No emoji utility exists anywhere in
  this stack to build on: `goad_toolkit` has no emoji regex or `emoji`-package
  dependency, and this repo's only emoji-related code is a hardcoded 4-emoji
  literal set in `tools/make_ci_own_chat.py` used purely to synthesize test
  fixtures — not reusable. When picked back up, still needs a few decisions,
  not assumptions:
  - a source of truth for "is this character an emoji" (a unicode-range
    regex, or adding the `emoji` PyPI package for `demojize`/categorization);
  - what "type" means — a unicode category (face/heart/hand/animal/...) is
    one option but is a taxonomy choice, not something shipped anywhere;
  - `RegexFeature`'s `extract` mode only pulls *one* capture group into a
    string column per row, so a single "emoji type" column isn't a fit for
    messages with several different emoji — likely needs either one
    `RegexFeature(mode="has"/"count")` call per emoji/category, or a custom
    `TransformBase` subclass (the notebook already says any such step works).
- **Behaviour/sentiment from message text + emoji.** Still open when this is
  picked back up:
  - message text is informal Dutch chat slang — off-the-shelf English
    sentiment tools (e.g. VADER) will likely score it badly; a Dutch-capable
    method would need to be chosen and is not picked yet;
  - using emoji as a sentiment/tone signal is a separate methodological
    choice on top of that (e.g. does 😂 read as positive or as this group's
    noted "dry/sarcastic in person, flat in text" register?) — no existing
    tool in this stack does this, it would be built from scratch;
  - worth checking against the course syllabus before building this —
    Stage 1 already notes lesson 4 covers distribution shape and lesson 5
    covers what separates people, so sentiment may genuinely belong later
    rather than in this lesson's pipeline (part of why it's parked here).
  - **Not yet drafted:** a falsifiable proposition for the underlying idea
    ("specific people, or specific time periods, shift toward more
    media/emoji-heavy messaging") in the same shape as the Stage-1
    proposition — including a falsification criterion and named
    boring/obvious results to rule out (e.g. "some people just send more
    photos than others" is not itself a finding). Needed before treating
    this as a claim to test, not just a feature to compute.

### Shape stage (goad interview, done)

- **Messages-per-day: matters.** Birthdays, and (during COVID) friends'/
  family illness updates, can cause fast-response spikes. **Decision: label
  spike days separately** (e.g. `birthday`, `family-news` tags) rather than
  switching to a spike-resistant average — spikes are their own visible
  story, not smoothed away.
- **Real risk to the Stage-1 proposition, explicitly flagged (not assumed
  away):** bad-news responsiveness may have been higher during COVID than
  after (everyone home more, more reactive) — if true, "lockdown = more
  active" could partly be an artifact of a few clustered bad-news days
  rather than a sustained shift. **Needs checking against the data**, not
  taken on faith.
- Messages-per-person: not expected to be decision-changing for now —
  assumed a fairly evenly active group of 9 with some outliers; not treated
  as a blocker.
- Group size confirmed: **9 unique authors** (independent-units count for
  any later per-person comparison).
- Related bucket-vs-continuum decision already made in the notebook: for
  "media-heavy senders," explicitly chose **not** to pick an arbitrary
  threshold for "a lot" — interested in the actual shape of the per-author
  media-count distribution instead.
- `n_capital` / `n_words` remain flagged (from the notebook's own gate) as
  needing a real attached question before Stage 4, or should be dropped.

## Stage 4 — Encoding (done)

- **Primary chart (family: time):** weekly message-volume trend across
  Aug 2020 – Aug 2026, lockdown windows shaded. Weekly, not daily — stage 3
  already decided spike days (birthdays, bad-news response spikes) get
  labeled/annotated separately rather than smoothed into the line, so
  weekly binning keeps the trend legible while the spike annotations carry
  the outlier story.
- **Companion chart, same claim (robustness check, not the parked
  per-user curiosity):** same weekly trend, split per author (small
  multiples or overlaid faded lines, group mean highlighted) — confirms the
  pattern holds across all 9 people rather than being driven by 1–2. Ties
  back to stage 3's "how many independent units back this" question.
- **Alternatives sketched and set aside for this specific claim:**
  - a during-vs-outside-lockdown bar comparison — rejected, hides whether
    it's a *declining trend*, only shows a blunt difference;
  - a daily-count distribution comparison (lockdown vs. after) — rejected
    as the primary chart (doesn't show decline directly), parked as useful
    later for the spike-day question.
- **Hour-of-day distribution** (lockdown vs. outside) — confirmed as a
  genuinely separate question (daily rhythm, not overall trend). Parked,
  not part of stage 1.
- **Single comparison the primary plot exists to make:** weekly message
  volume over time, against shaded lockdown windows, to show whether it
  peaks during lockdown and declines afterward.
- **Parameters flagged for later sensitivity-checking** (could flip the
  read): weekly vs. daily bin width; the lockdown boundary dates themselves
  (still approximate/unvalidated, per stage 2).

## Stage 5 — Critique (done)

- **Chart built:** `notebooks/analysis/01-lockdown-activity.ipynb` (new folder,
  separate from the numbered course-lesson notebooks) — loads the already
  anonymised/featured export from `01.3-your-own-chat.ipynb`, adds the
  stage-2 lockdown-period lookup, weekly bins (`week_start` = Monday of each
  ISO week, per the stage-3/4 decision to keep spike days out of the trend
  line), and renders both the primary weekly-volume chart and the per-author
  companion small-multiples chart (shared y-scale).
- **First impression (student's own read, primary chart):** activity does
  rise around the first lockdown (2020–2021) — visibly higher than the
  trend right after it, in late 2021. But the **second shaded lockdown
  (2021–2022) shows no comparable spike.**
- **Claim, stated plainly:** activity should be higher during lockdown
  because communication shifted onto WhatsApp — but the student's own
  assessment is that **this claim cannot be fully observed in the data**:
  only one of the shaded lockdown windows lines up with a visible rise, and
  several *larger, unshaded* peaks show up well after 2022 (2023–2025) that
  the lockdown story does not explain.
- **What this means for the stage-1 proposition:** this is a real
  falsification signal, not a technicality — noted here rather than argued
  away. The declining-trend-since-lockdown half of the proposition also
  doesn't match the chart (volume looks noisy/spiky throughout, with peaks
  recurring well after lockdown ended, not declining).
- **Not yet done:** the mechanical half of the critique checklist (what's
  grouped visually vs. conceptually, what could be deleted, whether colour
  is used as a pointer) — parked, secondary to the claim-level finding
  above.

## Stage 6 — Verification (done)

- **Null stated:** volume is flat/noisy regardless of lockdown status — no
  real difference in message counts tied to restriction periods.
- **Multiple-comparisons check:** honest answer — no other period splits
  were tried before landing on lockdown vs. non-lockdown. This was the only
  comparison made, not the best of several, so the "best of many" risk
  doesn't apply here — but that also means the pattern hasn't been
  stress-tested against alternatives yet.
- **Shuffle test (gut estimate):** student expects a shuffle would produce
  something "this strong" quite often — similarly-sized spikes already
  appear in 2022 and 2023, well outside any shaded lockdown window.
- **Confounder identified:** family/friends' illness news (already flagged
  as a live risk back in stage 3) keeping the group engaged — a plausible
  alternative driver of spikes, independent of lockdown status.
- **Held-out slice check (informal, done by eye):** the 2020–2021 rise still
  looks real and remarkable on its own. But the **same magnitude of rise
  recurs in mid-2022, end-2022, and end-2023** — none of which are lockdown
  windows. So the pattern does **not** survive as lockdown-specific once
  compared against other slices of the same series; it reads as general
  recurring spikiness (consistent with the bad-news-responsiveness risk
  flagged in stage 3), not something unique to lockdown.
- **Residual/model check:** not done — no rolling average or trend fit
  applied yet, so no residual shape to inspect.

### Verdict on the stage-1 proposition

**Not supported as stated.** Only the first lockdown window shows a rise;
the second does not; and later, unshaded periods (2022–2023) show spikes of
comparable or greater size. The declining-trend half of the proposition
also doesn't hold — volume stays noisy/spiky through 2026 rather than
trending down after restrictions lifted. This is the honest outcome of the
stage-5/6 checks, not a plotting problem — per goad's own guidance, the
right next move is a **sharper stage 1**, not a better chart: the real
signal in this data looks like it's about *recurring spike events*
(bad-news responsiveness, named as a risk back in stage 3) rather than a
sustained COVID-era shift.

**Decision:** write up this negative finding as the deliverable (not a
sharper stage-1 pass on the spike-driven proposition — parked as a
follow-up hypothesis, not started).

## Write-up

- **File:** `notebooks/analysis/findings.md` (+ `weekly-volume.png`,
  `weekly-volume-per-author.png` exported from the notebook for embedding).
- **Audience:** course group + teacher, written to also hold up if shown to
  the friends in the group chat — matches stage 1's audience note.
- Walks through: original expectation → what the chart shows → the stage-6
  checks → the honest verdict (proposition not supported) → what looks more
  likely instead (recurring bad-news-driven spikes, flagged as an untested
  hypothesis, not a new conclusion) → explicit list of what this check did
  *not* do (no other splits tried, boundary dates approximate, shuffle/
  held-out checks done by eye not computed, no residual fit).

---

# Analysis 2 — comparing categories (lesson 2, `02.5-your_turn.ipynb`)

New goad cycle, separate claim from the lockdown analysis above. Same chat
data, same 9 anonymised authors.

## Stage 1 — Question (done)

- **Origin:** lived experience/memory, not from skimming the export first —
  the student named real recurring phrases this group actually uses to
  organize meetups, before any regex was written:
  `Wie kan vnv?`, `Wie dit weekend iets doen?`, `Wie wat drinken?`,
  `Wie afspreken?`, `Heren, wie willen er allemaal vnv afspreken?`,
  `Wie vanavond wat doen?`, `Wie kon er vanavond`,
  `Iemand vanavond nog wat doen?`.
- **Group-specific shorthand (color for the write-up):** `vnv` = "vanavond"
  (tonight) in this group's own usage — corrected after an initial wrong
  guess ("vrijdag na vijven") got written down here first; a stranger
  reading raw rows would never guess this either way, so it still needs a
  one-line gloss for the teacher.
- **Pattern in the examples:** a "who's in" opener (`wie` / `iemand`) paired
  with an activity word (`vnv`, `afspreken`, `vanavond`, `weekend`, `wat
  doen`/`wat drinken`/`iets doen`) — not always with a question mark
  (`Wie kon er vanavond` has none), so an "ends in ?" feature alone would
  have missed real examples.
- **Proposition:** the people with the highest overall message volume are
  **not** the ones with the highest share of planning-related messages —
  the volume ranking and the planning-share ranking don't line up.
- **Falsification:** if the top-volume people also come out on top (or
  roughly matching) on planning-share, that says no.
- **Boring result guarded against:** raw planning-message *count* would
  just track overall volume — that's why the metric has to be a per-person
  *share* (their planning messages ÷ their own messages), same
  counting-messages-vs-counting-people trap 02.4 warns about.
- **Limitation, named up front:** keyword/regex detection for "planning a
  meetup" will have real misses (paraphrased plans, a one-word reply inside
  an existing planning thread the regex won't catch) — logged now, not
  discovered later.
- **Chart family (already picked, before building):** staying inside the
  comparing-categories/bar-chart toolkit from 02.2 — two ordered bar charts
  side by side, one author-by-volume, one author-by-planning-share, so a
  reader can see by eye whether the top of one list is the bottom of the
  other.

## Stage 2 — Data (done)

- **Built in:** `notebooks/lesson2/02.5-your_turn.ipynb` (the course's own
  "your turn" cell, not tracked in git per the notebooks/lesson* ignore
  rule — this is the working notebook, not the deliverable yet).
- **`is_planning` feature:** `has_who` (`RegexFeature`, pattern
  `(?i)\b(wie|iemand)\b`, mode `has`) AND `has_activity_word`
  (`RegexFeature`, pattern
  `(?i)\b(vnv|afspreken|vanavond|weekend|wat\s+doen|wat\s+drinken|iets\s+doen)\b`,
  mode `has`), combined with plain pandas `&` (the toolkit's `RegexFeature`
  only does one pattern at a time).
- **Flagged 169 of 22,517 messages (0.75%)** as planning-related.
- **Precision spot-check (done, as flagged as a limitation in stage 1):** a
  random sample of 10 flagged messages, all genuine hits by eye (e.g.
  "Iemand zin in een biertje in het zonnetje vanavond...", "Wie drinken er
  allemaal geen bier vnv?"). No obvious false positives in the sample —
  doesn't rule out misses (recall), only checks precision on what got
  flagged.
- **Claim's unit vs. row's unit, named explicitly:** the claim is about
  people, not messages — **n=9**, not n=22,517. Whatever the two-panel
  chart shows, it's a real pattern to report but a small-n one; the
  write-up needs to say so plainly rather than imply more confidence than
  9 points support.
- **Per-author table built:** `messages` (volume, via `GroupAgg` size) and
  `planning_share` (`planning_messages` ÷ `messages`, not raw count — same
  counting-messages-vs-counting-people trap as 02.4/analysis 1).

## Stage 4 — Encoding (done, decided in stage 1)

Two ordered horizontal bar charts side by side, same 9 authors: messages
per author (volume), and planning-message share per author — built with
`create_figure(n_plots=2)` + `plot_on_axes`, same pattern as 02.2's
grouped-bar/heatmap pairing. No stage-3 shape check judged necessary —
`messages` and `planning_share` are both single summary numbers per
person, not distributions to inspect.

**Result (for the record, not yet critiqued by the student):**

| author | messages | planning_share |
|---|---:|---:|
| pliable-tiger | 3436 | 0.0093 |
| rib-tickling-curlew | 2922 | 0.0027 |
| fluffy-beaver | 2907 | 0.0045 |
| effervescent-penguin | 2795 | 0.0057 |
| humorous-stingray | 2762 | 0.0043 |
| animated-elk | 2650 | 0.0098 |
| vibrant-barracuda | 2110 | 0.0118 |
| striking-rail | 1782 | 0.0168 |
| hypnotic-rabbit | 1153 | 0.0061 |

## Stage 5 — Critique (done)

- **Both panels put in the same author order** (sorted by volume, applied
  to both via `order=`) after the student asked for it — same person in
  the same row in both panels, rather than each panel independently
  sorted. `planning_share` also converted to `planning_pct` (×100) and the
  right panel's x-axis relabelled "% of own total" — a bare fraction read
  as a rounding error.
- **Student's own read:** a small negative relationship — as message
  volume goes down, planning-share tends to go up. Three named exceptions
  out of 9: **pliable-tiger** (highest volume, but also a relatively high
  planning-share — breaks the trend at the top end), **striking-rail**
  (called an outlier, though its position — low-ish volume, highest share —
  is actually the trend's strongest instance, not a break from it),
  **hypnotic-rabbit** (lowest volume, but only a middling/low share, where
  the trend would predict it highest).
- So: roughly 6 of 9 points fit a negative-relationship story, by the
  student's own read, with real named exceptions rather than a clean line.

---

# Analysis 3 — election windows vs. message length

New goad cycle, separate claim from analysis 1 (lockdown trend) and analysis 2
(volume vs. planning-share). Same chat data, same 9 anonymised authors.
**Time budget: tight — one evening**, so scope stays narrow.

## Stage 1 — Question (done)

- **Category:** election-window vs. non-election-window, using the election
  dates already logged under analysis 1 / stage 2 (±30 days around each of
  the 5 election dates listed there).
- **Metric:** message length (character count and/or word count).
- **Proposition:** average message length is higher inside election windows
  than outside them.
- **Falsification:** if average length is essentially the same in both
  windows, that says no.
- **Suspicion/mechanism:** the chat is normally short, efficient
  planning-style exchanges; a shift toward political debate during election
  periods should read as longer, more argumentative/informative messages.
- **Boring result guarded against:** some specific author just always writes
  longer messages regardless of period — that's a per-author trait, not a
  period effect. The comparison needs to check per-author, not only pool all
  messages across the group (same counting-messages-vs-counting-people /
  pseudoreplication trap as analyses 1 and 2).
- **Origin:** mixed — partly lived experience (election-time debates
  remembered as more argumentative), partly reasoning from the data's known
  structure (planning messages dominate and are short, so a topic shift
  should show up in length) rather than from already having looked at the
  actual length numbers.
- **Extra dimension flagged for stage 3/4:** per-author breakdown of the
  election-vs-not length difference, to check whether one talkative/
  argumentative author is driving the whole group average rather than the
  effect being general across the group.
- **Audience:** same as before (course group + teacher, legible to a
  non-technical audience, could be shown to the friends themselves) — but
  smaller time budget this round.

## Stage 2 — Data (done)

- **`election_period`:** reused, join of the existing static lookup from
  analysis 1 (5 elections, ±30 days each).
- **New feature `n_chars`:** character count of message text (`.str.len()`),
  alongside the already-existing `n_words`.
- **Row vs. claim unit, deliberately checked at two levels:** row = message,
  but the comparison is built as (a) a pooled message-level view
  (election vs. non-election) as the headline, and (b) a per-author
  breakdown in the *same* table from the start — not bolted on after —
  so the pooled effect can be checked against the stage-1 boring-result
  guard (one author driving the whole group average) rather than assumed
  away.
- **New decision surfaced in this stage:** exclude media-placeholder
  messages (`<Media weggelaten>`) before computing length stats, reusing
  the existing `is_media` `RegexFeature` flag — the placeholder text isn't
  a real message length and would be pure noise for this metric.
- **Provenance:** `n_chars`/`n_words` derived directly from the raw message
  text column, no missingness. `election_period` is a joined static lookup
  — same "approximate, not yet validated against an authoritative source"
  caveat as analysis 1.
- **One combined table** (`author`, `election_period`, `n_words`, `n_chars`,
  `is_media`) feeds both the pooled and per-author views.

## Stage 3 — Shape (done)

- **Report both median and mean per group** — a long tail of genuinely long
  messages could pull the mean around while the median stays resistant;
  showing both lets a reader see whether they tell the same story or
  diverge, rather than picking one number to lead with.
- **Extreme values judged genuine, not errors:** after the media-placeholder
  exclusion (stage 2), any remaining very long messages are treated as real
  long messages (e.g. a wall-of-text/rant), not something to filter out.
- **New confounder named, not assumed away:** independent units per group
  (author presence in both windows) is **not confirmed stable**. Real
  suspicion: some authors may simply be more *active* specifically during
  election windows because they personally care more about politics — so a
  pooled length effect could partly be a **composition shift** (more
  messages from naturally verbose/opinionated people during elections)
  rather than the same people writing longer messages in both windows. This
  needs checking (e.g. per-author message counts in each window) before the
  pooled result is trusted at face value — parallel to analysis 1's
  bad-news-responsiveness confounder.
- **Bucket vs. number line:** number line — plain numeric length, not
  bucketed into short/medium/long — for both the pooled and per-author
  views.

## Stage 4 — Encoding (done)

- **Obvious family:** categories/bar-chart comparison, matching the course's
  lesson-2 comparing-categories toolkit already used in analysis 2.
- **Alternative sketched and rejected:** boxplot/violin (would show the
  distribution shape and long tail flagged in stage 3) — explicitly rejected
  by the student in favor of staying in the bar-chart family, since this is
  a categories comparison.
- **Final decision — four separate bar charts (not one combined figure):**
  1. Pooled **mean** message length, election vs. non-election (2 bars).
  2. Pooled **median** message length, election vs. non-election (2 bars).
  3. **Per-author mean** message length, election vs. non-election (grouped/
     paired bars, 9 authors).
  4. **Per-author median** message length, election vs. non-election (same,
     median).
  Mean and median stay as separate charts rather than overlaid, per stage
  3's decision to report both without picking one.
- **Trade-off named and accepted:** bar charts show only the two
  central-tendency numbers per group, not the distribution shape/long tail
  stage 3 flagged as real — accepted given the tight one-evening budget and
  the deliberate choice to stay in the categories/bar-chart family.

### Build

- **Notebook:** `notebooks/analysis/02-election-length.ipynb` (new folder file,
  same pattern as `01-lockdown-activity.ipynb`) — loads the featured export,
  reuses the election-window `FlagDates` join, adds `n_chars` and `is_media`
  (`RegexFeature`, `<Media weggelaten>`, `mode="has"`), excludes media rows
  before computing length stats, and renders all four stage-4 bar charts.
- **Pooled result:** mean 39.6 chars (election) vs. 35.7 (rest) — a visible
  gap. **Median 26.0 vs. 25.0 — nearly identical.** The mean/median
  divergence stage 3 flagged as a real risk actually happened.
- **Per-author result:** `pliable-tiger` (highest-volume author from
  analysis 2) jumps from ~41 to ~59 mean chars during election windows — by
  far the largest mover — and also sends the most election-window messages
  of anyone (603, also the group's highest). The other 8 authors show small,
  mixed-direction differences. Median-per-author chart is messier still, no
  single person as dominant.

## Stage 5 — Critique (done)

- **Student's own read:** the pooled difference does **not** hold up as a
  group-wide pattern once the per-author charts are looked at — only
  `pliable-tiger` shows the election-window jump, in both length and
  volume. The other 8 authors show no consistent effect.
- **Verdict on the claim as stated:** not supported by the per-author
  picture. Reads as driven by a single person, not a shift in the group's
  behaviour — the same boring-result guard named back in stage 1 (`some
  specific author always writes longer messages`) turned out to be closer
  to the truth than the original proposition, except here it's period-
  specific to one person rather than a constant trait.
- **Mechanical half of the critique checklist** (grouping/colour/what could
  be deleted) skipped for time, same as analysis 1's stage 5 — secondary to
  this claim-level finding.

## Stage 6 — Verification (done)

- **Null stated:** message length doesn't really differ by election window
  for anyone, including `pliable-tiger` — the ~41→59 jump is noise. (Student
  needed help landing on this phrasing, but agreed it's the right null.)
- **Multiple-comparisons — honestly disclosed:** looked at all 9 authors'
  bars and `pliable-tiger` is the one that stood out *after* looking at all
  of them, not picked in advance for a specific reason. Real risk, named
  rather than argued away: this is "most extreme of 9", not a
  pre-registered single comparison.
- **Student's own gloss on what this is worth, even if real:** not a
  group-dynamic finding — at most "one person shows more interest in
  debate/writes more during elections," which the student themselves calls
  "nothing groundbreaking on group chat dynamics."
- **Confounder — agreed, not disputed:** reads as a volume/composition
  effect (`pliable-tiger` simply sends more messages during election
  windows), not a genuine "writes longer messages" behavioural shift. Same
  family of risk as stage 3's composition-shift concern, just landing on
  the one person where it's visible.
- **Shuffle-test (gut estimate):** judged rare by chance — specifically
  because `pliable-tiger`'s own *rest-of-chat* baseline length is
  unremarkable, in line with peers (`humorous-stingray`, `striking-rail`,
  `effervescent-penguin`). So the election-window jump is a real deviation
  from that person's own baseline, not a fluke reading of an already-high-
  variance baseline.
- **Held-out slice / residual checks:** not done — time budget spent
  elsewhere (same gap as analysis 1's stage 6).

### Verdict on the stage-1 proposition

**Not supported at the group level.** The pooled mean/median divergence
(mean shows a gap, median doesn't) already undercut the headline number
before the per-author chart was even looked at. The per-author breakdown
then showed the pooled mean effect is carried by one person
(`pliable-tiger`), not the group — and even that one person's jump looks
like a volume/composition effect (posting more during elections) rather
than a "writes longer messages when debating" effect specifically. The
individual-level pattern is judged likely real (rare-by-chance per the
shuffle-test gut check, given that person's otherwise-unremarkable
baseline) but explicitly **not** the group-level story the stage-1
proposition claimed, and the student's own read is that it isn't a
noteworthy finding about the group chat's dynamics even if true.

**Decision:** write this up as a second negative/narrow finding
(proposition not supported as a group effect; one person's volume-driven
length increase noted as a minor, unconfirmed aside) rather than pursuing a
sharper stage-1 pass tonight, given the time budget.

### Follow-up check: messages per week per author (requested after stage 6)

Direct test of the composition-shift confounder named in stage 3/6 — message
*rate* (messages/week, normalised for the two windows covering very
different numbers of calendar days), election window vs. rest, per author.
Added to the same notebook (`notebooks/analysis/02-election-length.ipynb`),
one more `GroupedBarPlot`.

**Result:** `pliable-tiger` goes from 9.7 → 13.8 messages/week, and
`rib-tickling-curlew` from 8.4 → 11.5 — both post noticeably *more often*
during election windows, not just longer. Several others move the other
way (`vibrant-barracuda` 6.6 → 4.7, `hypnotic-rabbit` 3.5 → 2.5). This
confirms the stage-6 read: `pliable-tiger`'s length jump lines up with a
real jump in how often they post, supporting "posts more, including some
longer messages, when there's more to talk about" over "writes longer
messages specifically."

## Write-up

- **File:** `notebooks/analysis/findings-election-length.md`.
- Walks through: original expectation → pooled chart (mean vs. median
  divergence) → per-author chart (one person driving it) → the messages-
  per-week confounder check → verdict (not a group effect) → what this
  check did not do.

---

# Analysis 4 — is group message rate itself higher during elections?

New goad cycle, prompted by the messages-per-week diagnostic chart built to
check analysis 3's composition confounder. Same chat data, same 9
anonymised authors, same election-window definition.

## Stage 1 — Question (done)

- **Origin, named honestly:** this proposition is **data-suggested**, not
  from prior lived memory — it came from looking at analysis 3's
  messages-per-week-per-author chart, not from something remembered about
  the group beforehand. That changes what counts as real confirmation: a
  second look at the same chart doesn't count as independent evidence.
- **Proposition:** overall/group message rate (messages/week) is higher
  during election windows than outside them, **and** that increase is
  concentrated in already high-volume authors (`pliable-tiger`,
  `rib-tickling-curlew`) rather than being a broad-based shift across the
  group.
- **Falsification:** (a) if a meaningful number of authors drag the
  pooled increase down (flat/decreasing rates), so there's no real net
  pattern, or (b) if excluding the already-high-volume authors makes the
  remaining group's rate increase disappear or reverse — either result
  says the claim (as a real, election-linked, volume-driven pattern)
  doesn't hold.
- **Boring result named and accepted as plausible:** already-high-volume
  authors simply post more during *any* notable/busy period, not
  something specific to elections — so this might not be an "election
  effect" at all, just "active people react more to any big event."
- **Independent check, required because the origin is data-suggested:**
  break the pooled election-window rate down **per individual election**
  (5 separate events, not pooled) — does the pattern hold consistently, or
  is it driven by one specific election?
- **Audience/time:** same as analyses 1–3, tight one-evening budget.

## Stage 2 — Data (done)

- **Claim unit, small in two different ways, both flagged:** n=9 authors
  for the per-author breadth check, and **n=5 elections** for the
  per-election independent check — small enough that one election (or one
  author) could dominate the picture, named up front rather than glossed
  over.
- **New feature `election_name`:** 6-level categorical (one of the 5 named
  elections, or "rest of the chat"), built from a day-level lookup joined
  by calendar day — confirmed the 5 ±30-day windows don't overlap, so a
  clean categorical (not just a boolean) is safe.
- **New flag `is_top_volume_author`:** `True` for `pliable-tiger` and
  `rib-tickling-curlew` (the two who showed the clearest rate increase,
  also analysis 2's top-2 by total volume), `False` for the rest — feeds
  the exclusion falsification check.
- **Provenance decision, made explicitly:** message counts/rates here
  **include** media-placeholder messages, matching analyses 1/2's
  precedent (a media share still counts as activity) — deliberately
  different from analysis 3's rate diagnostic, which excluded media only
  because it was checking a length metric.
- **Pipeline:** `election_name` built as a plain pandas day-level lookup
  (looping over the 5 named windows) — `FlagDates` only produces one
  boolean at a time, not a multi-level categorical.

## Stage 3 — Shape (done)

- **Shape matters here, specifically for sparsity:** a full 9-author ×
  5-election breakdown would have very sparse, noisy cells (some
  combinations with very few messages, swinging the rate on small counts).
- **Decision: both checks stay at the pooled group level**, not a full
  per-author-per-election matrix:
  1. Pooled rate per election (5 individual elections vs. "rest of the
     chat", 6 bars) — does the pattern hold consistently, or is it driven
     by one election?
  2. Pooled election-window rate with vs. without the two high-volume
     authors excluded — directly tests stage-1's exclusion falsification
     criterion.
- **Independent units:** n=5 elections for check 1 (small — already
  flagged), n=7 remaining authors for check 2 after exclusion (reasonably
  sized).

## Stage 4 — Encoding (done)

- **Obvious family:** categories/bar chart, consistent with the rest of
  tonight.
- **Alternative sketched and set aside:** a weekly timeline with all 5
  election windows shaded (same chart type as analysis 1) — more honest
  about the small-n-of-5 risk (a reader would see how thin each window
  actually is), but more work and overlapping with analysis 1's chart
  type. Set aside for consistency and time.
- **Final decision — two horizontal bar charts:**
  1. Pooled message rate (messages/week) per election (5 named elections)
     vs. "rest of the chat" — 6 bars, election bars highlighted. Tests
     whether the pattern holds across all 5 or is driven by one.
  2. Pooled election-window rate vs. rest-of-chat rate, grouped/paired
     bars for "all 9 authors" vs. "excluding the 2 high-volume authors" —
     directly tests the exclusion falsification criterion from stage 1.

### Build

- **Same notebook:** `notebooks/analysis/02-election-length.ipynb`, new
  section appended after analysis 3.
- **Chart 1 result — NOT consistent across elections:** US pres. 2020
  (122.0 msg/wk), NL 2021 (89.3), NL snap 2023 (92.1) all higher than the
  70.1 rest-of-chat baseline — but **US pres. 2024 (39.2) and NL 2025
  (40.3) are both clearly lower**, not higher. 3 of 5 elections support
  the pattern, 2 of the most recent 2 contradict it outright.
- **Chart 2 result — exclusion check confirms the concentration claim
  cleanly:** excluding `pliable-tiger`/`rib-tickling-curlew`, the
  remaining 7 authors show **no real difference** (51.0 rest vs. 50.6
  election — essentially flat). The top-2 alone show a real jump (19.1
  rest vs. 26.0 election). So the "concentrated in already-high-volume
  authors" half of the proposition holds up well; the "group rate is
  higher during elections" half does not — it's entirely those two
  people, and even they don't drive an effect in the two most recent
  elections' pooled total (chart 1).

## Stage 5 — Critique (done)

- **Student's own read, chart 1:** elections became "less of a talking
  point over time" — the 3 earlier elections (2020, 2021, 2023) sit above
  baseline, the 2 most recent (2024, 2025) sit below it, read as a
  declining trend in how much the group engages with elections
  specifically.
- **Student's own read, chart 2:** confirms the rate effect is a
  two-author concentration, not a broad group shift.

## Stage 6 — Verification (done)

- **Confounder confirmed as plausible, not dismissed:** student had
  already noticed, from analysis 1's own chart (not its stated goal), that
  later years show less overall chat activity. This is a real threat to
  "elections became less of a talking point" — the 2024/2025 dip could
  just be general chat decline, since the pooled "rest of the chat"
  baseline mixes early (busier) and late (quieter) years together, rather
  than comparing each election to its own local surrounding period.
- **Null pushed back on, usefully:** student doesn't accept a blunt "no
  election effect anywhere" null — their own view is that activity
  increases only for *selected/particular* elections (implying some are
  more contentious/salient than others). Real, more specific hypothesis,
  **not tested here** — parked as a future question, not a conclusion.
- **Multiple comparisons, honestly disclosed:** would not have predicted
  "declining over time" in advance — it emerged from looking at all 5,
  reinforcing the data-suggested-origin risk already named in stage 1.
- **Shuffle-test gut check:** unsure, leans toward "not necessarily rare,"
  given the chat is already known (analysis 1) to move in peaks/spikes
  generally. Not confidently judged unlikely by chance.
- **Same year-confound risk applies to chart 2:** the two high-volume
  authors could simply have been more active in the earlier years when
  most of the "positive" elections happened, independent of election
  status specifically — the concentration finding is comparatively more
  solid than the per-election trend, but not fully insulated from the
  same confounder.

### Verdict on the Analysis-4 proposition

**Left in real doubt, not confirmed.** The "concentrated in two
high-volume authors" half holds up reasonably well (chart 2's exclusion
check is clean), but the "group rate is higher during elections" and
"elections became less of a talking point over time" halves cannot be
told apart, with this analysis as built, from a much more boring
explanation already on the table from analysis 1: **the whole chat got
quieter in later years, for reasons unrelated to elections.** Verifying
that properly would mean comparing each election window to its own local
surrounding period (not a single pooled 6-year baseline) — not done
tonight, time budget spent building and reading the two charts instead.

**Decision:** write this up as an honest "real pattern, ambiguous cause"
finding — flag the general-activity-decline confound explicitly as the
leading alternative explanation, and park "some elections matter more than
others" as an untested hypothesis for a future pass, same as analysis 1
parked its recurring-spikes hypothesis.

### Presentation draft: the concentration finding (chart 2), for class

Requested separately from the six-stage cycle — a polished, single-comparison
redraw of chart 2's "concentrated in two authors" finding, run through
`goad_critique_visual`'s four sections rather than a fresh goad analysis
cycle (the underlying claim was already verified above; this is about how
it's shown).

- **v1 (`slide-two-authors-drive-it.png`):** one bar per author, the
  *increase* in messages/week (election − rest), ordered, with
  `pliable-tiger`/`rib-tickling-curlew` highlighted red, rest grey.
  Direct value labels, no gridlines, exported at 200dpi.
- **Critique, section 1 (first impression) — student's own read:** saw the
  red bars first, but on a second look noticed `fluffy-beaver` (+2.0) and
  `vibrant-barracuda` (−2.1) are also visually significant — the red/grey
  binary implicitly overclaimed "only these two changed," when several
  others moved by a real amount too.
- **Section 2 (grouping) — mismatch identified:** the 2-colour split
  visually groups the world into "these 2" vs. "the boring rest," but the
  rest isn't boring — it contains real, individually-varying ups and
  downs that only *net* to roughly zero. Proposed collapsing the other 7
  into one "net" bar; **student rejected this** — don't hide data that
  exists, keep all 9 authors visible.
- **Reframe (student's own idea, and a genuine improvement):** changed
  the claim itself, from "two people drive it" to **"election windows
  create attractors and detractors"** — a two-tailed story that uses
  every bar's own sign/magnitude rather than a pre-picked pair. Colour
  now carries data (attractor vs. detractor), not emphasis on an
  identity chosen in advance — a deliberate, named exception to
  "start with grey."
- **Section 3 (five guidelines):** show-the-data now genuinely holds (no
  aggregation). One clutter fix applied: dropped the redundant `"author"`
  y-axis label (names are already the tick labels) — required an
  explicit `ax.set_ylabel("")` *after* the seaborn draw call, since
  seaborn relabels axes from the column name at draw time and overrides
  whatever `PlotSettings` set beforehand.
- **v2, a follow-up the student asked for (`slide-attractors-detractors-pct.png`):**
  same comparison as **% change from each author's own baseline**, not
  absolute messages/week — motivated by the observation that a +3.9
  jump means less for an already-heavy poster than for a light one.
  Revealed something the absolute chart hid: `hypnotic-rabbit`'s
  proportional drop (−29%) is nearly as extreme as `vibrant-barracuda`'s
  (−30%), despite a much smaller absolute change (−1.1 vs. −2.1) — the
  two had very different baselines.
- **Threshold, decided explicitly:** ±20% change counts as a "large"
  attractor/detractor (coloured); within that band stays neutral grey.
  Chosen after seeing the actual distribution (a natural gap between
  fluffy-beaver at +22% and animated-elk at +10%), not picked blind.
  Result: 3 large attractors (`pliable-tiger` +38%, `rib-tickling-curlew`
  +34%, `fluffy-beaver` +22%), 2 large detractors (`hypnotic-rabbit`
  −29%, `vibrant-barracuda` −30%), 4 neutral in between.
- **Section 4 (claim) — student's own answers:** claim stated as "during
  election windows, a few authors post noticeably more or less relative
  to their own normal pace, rather than the whole group shifting
  together." Falsification named: if election-window message counts were
  tiny in total, the % swings would be noise, not a real effect — **this
  motivated adding sample size directly to the chart** rather than
  leaving it as an unstated caveat. Comparison judged as shown directly,
  not requiring reader computation. ±20% threshold defended, not
  arbitrary: the data itself splits into three visible clusters
  (attractors/detractors/neutral), not a forced cut.
- **Follow-up requested and applied:** (1) swapped colours — attractor =
  blue, detractor = red (was crimson/steelblue) — across both charts;
  (2) added each author's election-window message count directly to the
  % chart's labels (`+38% (n=619)` etc.), so a reader can judge sample
  size themselves. Result: the biggest movers rest on solid samples
  (`pliable-tiger` n=619, `rib-tickling-curlew` n=515, `fluffy-beaver`
  n=474), but the most extreme detractor by %, `hypnotic-rabbit` (−29%),
  rests on the smallest sample of the nine (n=117) — worth saying out
  loud in class as the one number most vulnerable to noise.
- **Final files:** `notebooks/analysis/slide-two-authors-drive-it.png`
  (absolute) and `notebooks/analysis/slide-attractors-detractors-pct.png`
  (% of own baseline, with sample sizes) — both reproducible by re-running
  `02-election-length.ipynb` top to bottom.
- **Last round of polish on the % chart:** headline/subtitle split — a
  catchy question (`fig.suptitle`, bold) as the hook, the precise claim
  demoted to an italic axis-level subtitle underneath it — and the
  x-axis label spelled out fully (`% change in # of messages/week during
  election windows (5 windows over a 6-year period)`) so the metric and
  scope are readable without the caption. `n=` in the labels clarified as
  each author's election-window message count specifically (the sample
  size the % for that bar rests on).

### Items 5 & 6 from `feedback-comparing-categories.md`, computed (not by eye)

Revisiting the % chart's stage-6 gaps, per the logged feedback. Sequencing decided
with the student first: statistics before redesign, since the redesign (items 1/2/4)
would need redoing if these checks undercut the pattern.

**Item 5 — error bars, Poisson chosen over bootstrap:** `SE(rate) = sqrt(count) /
weeks` per side (election window, rest of chat), combined via the standard
delta-method formula for a ratio of two independent rate estimates, since
`pct_change` is a ratio of two rates. Student's call, after a plain-language
trade-off explanation: Poisson is faster (a formula, no resampling) but assumes a
steady drip with no bursts — known not to hold for this chat (birthday/bad-news
spikes, per Analysis 1) — versus bootstrapping the ~9 weeks per window, which is
more honest about burstiness but thin on only ~9 weeks to resample from.

**Result — SEs land in a similar ~5–7 percentage-point band across all nine
authors**, regardless of each author's own election-window sample size. Reading each
author's change as a multiple of its own SE (change ÷ SE, a rough z-score):
`pliable-tiger` (+37.9%, 6.2×), `vibrant-barracuda` (−29.9%, 5.9×),
`rib-tickling-curlew` (+34.3%, 5.3×), `hypnotic-rabbit` (−29.1%, 4.2×), and
`fluffy-beaver` (+22.3%, 3.6×) all sit 3+ SEs from zero; `animated-elk`,
`effervescent-penguin`, `striking-rail`, `humorous-stingray` all sit under 2 SEs —
statistically indistinguishable from noise by this measure alone.

**Item 6 — shuffle test, computed:** 5 random ~61-day windows (student's number),
drawn from within the 2020–2025 span the 5 real elections already occupy (not
2026's unmatched tail — student's call, to avoid the general-activity-decline
confound deciding the answer by itself), non-overlapping with the real windows or
each other, compared against each author's own quiet baseline (excludes all 5 real
election windows, same baseline the real `pct_change` used — a like-for-like
reference, not a mixed one). A draw only counts as a "match" if it moves at least as
far in the *same direction* (student's call) as the real result. Seeded (42) for
reproducibility, in `02-election-length.ipynb` after the pct chart.

**Result:** `rib-tickling-curlew` 0/5 matches, `pliable-tiger` and
`vibrant-barracuda` 1/5 each — all read as real, larger-than-typical swings for
those three, not likely chance. `hypnotic-rabbit` matched 3/5 — despite clearing 4+
SEs by the Poisson measure above, a swing this size for this specific person
happens by chance quite often. **The two methods disagreeing on `hypnotic-rabbit`
is itself informative, not a bug:** it's exactly the failure mode named when
choosing Poisson over bootstrap — Poisson assumes no bursts, and `hypnotic-rabbit`'s
n=117 (smallest of the nine) means a handful of clustered messages would move the
rate further than Poisson's formula expects.

**Caveat, not glossed over:** only 5 shuffle draws — "0/5" and "1/5" are themselves
noisy estimates (roughly "under 20%"), not precise percentages.

**What this does and doesn't establish:** a low shuffle-match rate supports "this
swing is real, not chance- or era-driven noise" for `pliable-tiger`,
`rib-tickling-curlew`, and `vibrant-barracuda`. It does **not** by itself establish
the *election* as the cause, as opposed to some other real, unrelated event landing
in that window for that person — the same causal-vs-correlation gap Analysis 1 hit
with bad-news spikes. Left as an open caveat for the write-up, not resolved here.

**Revised confidence ranking, to build the redesign on:** `pliable-tiger`,
`rib-tickling-curlew`, `vibrant-barracuda` — solid on both checks. `fluffy-beaver` —
solid on SE alone (not one of the three the feedback named for shuffle focus).
`hypnotic-rabbit` — the number to caveat hardest or drop from a simplified chart's
highlighted set, despite looking strong on SE alone. The remaining four
(`animated-elk`, `striking-rail`, `humorous-stingray`, `effervescent-penguin`) —
statistically flat on both measures, candidates to grey out uniformly regardless of
which side of the old ±20% threshold they happened to land on.

### Presentation draft v3: narrowed to `pliable-tiger`, built on the stats above

Not a rebuild — v3 lives alongside v1/v2 in the same notebook, same pattern as
before. Decisions made with the student:

- **Scope narrowed to `pliable-tiger` alone**, not the full attractor/detractor set —
  the one person with a real, specific story to tell: ~10 years studying history,
  enormously politically engaged, briefly a member of a Dutch political party, known
  in the group for driving political-topic discussions (e.g. democracy). Answers the
  "still to do" gap named back in item 3 of `feedback-comparing-categories.md` — an
  anonymous alias's % change now has an actual behavioural story behind it. Real name
  deliberately kept out of the notebook/chart — item 3's audience/sharing-plan caution
  is unresolved, not overridden.
- **All 9 authors stay visible, in grey** (student's explicit call, not the model's) —
  only `pliable-tiger` coloured. Preserves the honesty property the original
  attractor/detractor reframe was built to protect (not implying "only this one
  person changed"), but without needing a 3-way statistical threshold to get there.
- **Old 3-way attractor/neutral/detractor legend dropped.** Replaced with a 2-entry
  legend: the highlighted author, and what the error bar means — otherwise the
  whiskers go unexplained once the threshold legend is gone.
- **Per-bar `n=` label kept only on the focus bar** (with its SE folded in:
  `+38% (n=619, ±6 SE)`); the other 8 rely on the error bar alone rather than a
  repeated text label — direct fix for item 1's "too much happening in one graph."
- **Error bars (item 5) kept on all nine bars**, not just the focus one — lets a
  reader see by eye that most of the grey bars sit close to their own zero-crossing.
- **File:** `notebooks/analysis/slide-pliable-tiger-election-effect.png`, same
  notebook (`02-election-length.ipynb`), reproducible top to bottom.

### Presentation draft v3, follow-up polish (items 7–10 from `feedback-comparing-categories.md`)

Resolved in the same notebook cell, decided with the student. Real-name/audience
question (item 3) still not settled, so the alias stays throughout.

- **Title (item 7):** shortened to *"pliable-tiger gets more active during election
  windows"* — states the finding directly, dropping the punchier-but-long "historian...
  awakes and activates" draft.
- **Subtitle/x-axis redundancy (item 8):** both dropped from the chart itself. The
  metric description they used to split between them (% change definition, "5 windows
  over a 6-year period") moved into the footnote instead — one place, not two.
- **Footnote (item 9):** cut to the error-bar explanation's first sentence only
  (dropped the second sentence spelling out how to read a specific bar's whisker —
  judged as more than a class audience needs), combined with the metric description
  that moved out of the subtitle/x-axis label per item 8.
- **`n=` label (item 10):** kept, but named — `n=619` became `n=619 election-window
  messages`, so a reader doesn't need to already know the chat's scale to judge
  whether that's a small or large sample. No rest-of-chat `n` added alongside it; the
  error bar already carries that reliability signal visually.
- **File:** same `notebooks/analysis/slide-pliable-tiger-election-effect.png`,
  regenerated top to bottom from `02-election-length.ipynb`.

### Presentation draft v3, follow-up round 2: title focus, footnote formatting

- **Title now names the backstory, not just the alias:** changed to
  *"pliable-tiger, the group's historian, gets more active during election
  windows"* — the student wanted the real story to be the focus, not a bare
  alias, following on from item 3's original point about anonymous codes
  needing an actual behavioural story.
- **Political-party detail kept off the chart, decided explicitly:** the
  student also wanted "former member of a political party" in the title —
  same backstory as "historian," logged in the v3 markdown cell above. Held
  back after a direct trade-off question: political affiliation is a more
  specific, more identifying fact within a 9-person friend group than
  "historian" alone, and item 3's audience/sharing-plan check (whether this
  can go in front of the teacher/group at all) is still unresolved. Student
  chose "historian only" — the political-party fact stays in the private
  notebook markdown, not on anything shown in class. Revisit both the title
  and this decision together once item 3 is settled.
- **Footnote reformatted as bullet points:** the combined metric-definition
  + error-bar sentence now renders as two stacked `•` lines instead of one
  run-on sentence.
- **File:** same `notebooks/analysis/slide-pliable-tiger-election-effect.png`,
  regenerated top to bottom.

### Presentation draft v3, follow-up round 3: teacher-feedback critique via `goad_critique_visual`

Student asked for a read of the latest chart against the teacher's original
feedback (`feedback-comparing-categories.md`). Walked the first-impression and
gestalt sections of the critique checklist rather than giving a verdict outright.

- **Student's own first-impression read:** saw the top bar first, both because of
  its colour and because it's the largest; named the bars/colour as the
  highest-contrast element (correctly, data-carrying); nothing bright/large that
  shouldn't be; flagged that comparing the top bar against the lowest detractor is
  hard without the colour carrying that comparison.
- **Gestalt issue surfaced (assistant-identified, per the checklist's own division
  of labour):** `rib-tickling-curlew`'s +34% sits almost as far from zero as
  `pliable-tiger`'s +38%, but grey grouped it with the flat/near-zero bars purely
  by colour similarity — fighting the eye's own read of the magnitudes, and
  directly related to the student's own "hard to compare" observation above.
- **Fix chosen, after a check on the claim it implies:** promote
  `rib-tickling-curlew` to the same highlight colour and label treatment as
  `pliable-tiger`. Before applying, confirmed with the student that "history
  fanatic" is lived knowledge about Jop (rib-tickling-curlew's real name) too —
  not an assumption invented to fit his bar being high. Same evidentiary bar
  pliable-tiger's "historian" story was held to; a categorical claim about "history
  fanatics" from n=2 is a bigger claim than "these two named people," and this
  chat's small-n risk has already been flagged repeatedly (Analysis 4).
- **Result:** both bars share the blue highlight and the `n=`/SE label format;
  title generalised to *"The group's history fanatics get more active during
  election windows"* (no longer names either alias directly — both are already
  the y-axis tick labels). Political-party detail from the original backstory
  still excluded from the chart; item 3's audience/sharing-plan question is
  about the trait's identifiability, unaffected by the highlight moving from one
  person to two.
- **Not yet worked through:** the checklist's remaining two sections
  (`guidelines`, `claim`) — parked for whenever the student wants to continue the
  critique.
- **File:** same `notebooks/analysis/slide-pliable-tiger-election-effect.png`,
  regenerated top to bottom.

### Presentation draft v3, follow-up round 4: the five guidelines

Continued the critique checklist. Show-the-data, avoid-spaghetti, and
start-with-grey all held up already (no aggregation hiding variation, no lines to
tangle, colour already reserved for the two focus bars). Two items acted on:

- **Reduce clutter — on-bar labels shortened:** `+38% (n=619 election-window
  messages, +-6 SE)` was judged too long sitting on the bar itself. Cut to just
  `+38%` / `+34%`; the `n=`/SE detail moved into a new third footnote bullet
  instead.
- **Integrate text — legend replaced with a direct annotation:** the
  "error bar: ±1 SE (Poisson)" legend box was a lookup, not integrated text.
  Replaced with `ax.annotate`, pointing an arrow straight at the whisker of the
  first grey bar (`fluffy-beaver`) — deliberately anchored away from the two
  highlighted bars, since it's explaining the error-bar concept in general, not
  something specific to either focus author.
- **Still open:** the checklist's `claim` section — parked for next.
- **File:** same `notebooks/analysis/slide-pliable-tiger-election-effect.png`,
  regenerated top to bottom.

### Presentation draft v3, follow-up round 5: the claim section — flagged, not resolved

Finished `goad_critique_visual`'s checklist with the `claim` section. Student's
stated claim: "people more interested in politics/history show this interest in
the chat and chat more, probably with each other."

- **Overclaim identified:** the claim bundles a measured half (message *rate*
  goes up for these two people during election windows — shown directly on the
  chart) with an unmeasured half (that the extra messages are actually *about*
  politics/history, and represent mutual conversation between the two). Nothing
  in this analysis's pipeline checks message content or reply structure — no
  keyword feature like Analysis 2's `is_planning`, no network/mention data.
- **Student's own answers exposed the gap:** named "topic isn't
  politics-related" as the falsifier (Q2) and then named topic data as what's
  "deliberately not shown" (Q5) — the claim's actual mechanism is the part left
  untested.
- **Direct tension with Analysis 4's own unresolved verdict:** "already
  high-volume authors post more during *any* notable/busy period" was flagged
  there as a live alternative explanation and never ruled out. The current
  title quietly resolves that ambiguity in the more interesting direction
  without new evidence.
- **Left open, decision deferred to the student:** either (a) narrow the
  title/claim to what's actually shown (activity/volume only, cause
  unstated), or (b) spend time testing the topic angle with a keyword check on
  the *extra* election-window messages specifically. Not decided this
  session.
- **Independent styling request, applied:** metric-definition text moved back
  out of the footnote into an italic subtitle under the headline (matching the
  original v1/v2 pattern); footnote trimmed to the error-bar explanation and
  sample-size bullet only. Layout retightened after the first pass left a large
  gap between subtitle and plot.
- **File:** same `notebooks/analysis/slide-pliable-tiger-election-effect.png`,
  regenerated top to bottom.

### Presentation draft v3, follow-up round 6: election-window detail moved to footnote

- **"(5 windows over a 6-year period)" removed from the subtitle**, replaced
  with a spelled-out footnote bullet: *"The election windows are based on 5
  periods of elections (3 within NL, 2 in USA) over a 6-year period."* — names
  which elections these are (matches `ELECTION_DAYS`: US 2020, NL 2021, NL
  snap 2023, US 2024, NL 2025) rather than just a bare count.
- **File:** same `notebooks/analysis/slide-pliable-tiger-election-effect.png`,
  regenerated top to bottom.

# Analysis 5 — football tournaments and chat activity (lesson 3, `03.3-events-in-your-chat.ipynb`)

New goad cycle, first "time" theme analysis. Same own chat data as lessons 1–2.
First draft, ~2 hour budget end to end, for class Tuesday 22nd.

## Stage 1 — Question (done)

- **Origin:** prior knowledge of the group, not data-mined — student already
  expects major football tournaments (World Cup, European Championship) to
  raise chat activity, before looking at any numbers.
- **Candidates considered, deferred:** a member's wedding/engagement
  announcement (~August 2023) — a single-day event, sample-size caveat noted
  up front, same trap `Analysis 4`'s per-bar `n=` labelling exists to guard
  against — and the end of the last COVID lockdown, flagged by the assistant
  as a likely confound with the general activity-decline trend already
  surfaced in `Analysis 4` above. Chosen not to pursue first for exactly that
  reason. Both parked for a later `03.3` pass, not abandoned.
- **Why football first:** unlike the other two, tournaments repeat (World Cup
  2022, Euro 2020/2024 all fall inside this chat's span) — several
  independent chances to see the same effect, rather than reading one single
  data point as if it were a pattern.
- **Proposition:** messages per day rise during football tournament windows
  (World Cup / Euro), compared to the surrounding baseline.
- **Falsification:** if the daily-count line does not rise during the
  tournament windows, that's a no.
- **Open, not yet checked:** whether "activity rises during a globally
  exciting event" is itself the boring result 03.1/03.3 warn about ("people
  sleep at night") — worth revisiting once the tournament date ranges are
  pinned down and multiple windows can be compared against each other rather
  than eyeballed once.
- **Audience / time budget:** in-class grading Tuesday 22nd; first draft
  only, ~2 hours end to end for stages 1–6 plus write-up.

## Stage 2 — Data (done)

**Tournament reference table (static lookup, approximate — same pattern as
the lockdown/election tables above; validate exact match dates if this goes
beyond a first draft):**

| Tournament | Hype start (−14 days) | Official start | End |
|---|---|---|---|
| Euro 2020 (played 2021) | 2021-05-28 | 2021-06-11 | 2021-07-11 |
| World Cup 2022 | 2022-11-06 | 2022-11-20 | 2022-12-18 |
| Euro 2024 | 2024-05-31 | 2024-06-14 | 2024-07-14 |

- **One row:** one calendar day. **Count:** messages sent that day —
  `own.set_index("timestamp").resample("D").size()`, same as 03.3.3's worked
  example. `resample("D")` already fills quiet days as zero, so no
  gap-missingness concern.
- **Unit of row vs. unit of claim:** same (day-level daily count) — no
  pseudoreplication concern for this visual check.
- **Derived feature:** a categorical `phase` column (`hype` / `during` /
  `baseline`) built from the reference table above via one `FlagDates` call
  per tournament, extended 14 days backward for the `hype` phase — student's
  explicit call to keep hype and during as separate categories rather than
  one merged window, so the chart can show whether activity actually ramps
  up before kickoff or jumps only once matches start.
- **Pipeline placement:** the `FlagDates` + static-lookup step belongs in a
  `Pipeline` (reusable), matching the existing lockdown/election pattern —
  but built fresh and self-contained inside
  `03.3-events-in-your-chat.ipynb`, not routed through the lesson-1
  `own_pipeline`.

## Stage 3 — Shape (done)

- **Gate check:** relevant, not skipped. This chat's daily/weekly counts are
  already known to be bursty (single-event spikes, not a steady Poisson
  drip — established in `Analysis 4`'s SE work above), so a raw-count-only
  read could mistake one loud day inside a tournament window for sustained
  elevated activity.
- **Decision:** show raw daily counts **and** the 7-day rolling average,
  for **each of the three tournament windows separately** — not pooled —
  so the effect's consistency (3/3, 2/3, or 1/3 windows showing a rise) stays
  visible to the reader instead of being averaged away.
- **n at the claim's unit:** 3 independent tournament windows. Named
  explicitly so the small-n limitation is stated, not hidden — same
  discipline as the wedding candidate's single-day caveat from Stage 1.

## Stage 4 — Encoding (done)

- **Obvious family (time) vs. stretch alternative (categories) sketched
  before committing:** three small-multiple line panels (chosen) vs. a
  grouped-bar chart of per-phase averages per tournament (rejected —
  would average away the window-by-window detail the shape stage just
  decided to keep visible).
- **The one comparison:** each tournament window against this chat's
  *overall* daily average (global mean over the full Aug 2020 – Sep 2026
  export, computed once before slicing), not a locally-clipped baseline —
  student's explicit call, "wants to show all the data."
- **Parameters fixed:** 7-day rolling window (matches 03.3.3's worked
  example); hype window shortened from the initially discussed 14 days to
  7 (student's revision).
- **Build:** `notebooks/lesson3/03.3-events-in-your-chat.ipynb`, cells
  after `3.3.5.1` — reference table, `FlagDates`-built `phase` column,
  `RollingAvg` computed on the full series *before* slicing into
  per-tournament panels (avoids rolling-window edge bias at each panel's
  start), `FacetPlot` for the three-panel layout, `sharey=True` so panel
  heights compare honestly.
- **Numbers behind the picture** (overall baseline 10.1 msgs/day):

  | Tournament | baseline | hype | during |
  |---|---|---|---|
  | Euro 2020 | 5.5 | 5.6 | **18.9** |
  | World Cup 2022 | 10.3 | 13.6 | **21.0** |
  | Euro 2024 | 5.2 | 5.4 | **19.9** |

  `during` clears the overall baseline 3/3 — the stage-1 proposition
  holds across all three windows, not just one. `hype` is inconsistent
  (flat for two, already-elevated for World Cup 2022) — the open question
  from Stage 3 resolved as "no clean ramp-up," which is itself the answer,
  not a failed check.
- **Pre-existing bug found and fixed while building this:** `own["timestamp"]`
  loads as a plain string column from this processed parquet, not
  datetime — broke `resample("D")` in the *existing* 3.3.1/3.3.3/3.3.4
  worked-example cells too, not just this new analysis. Fixed once in the
  shared setup cell (`own["timestamp"] = pd.to_datetime(...)`,
  `03.3-events-in-your-chat.ipynb`) rather than patched per-cell.

## Stage 5 — Critique (done)

- **First impression (student's own read):** a small rise in the red
  (`during`) band across all three years; World Cup 2022 has the highest
  absolute activity of the three — noted as a separate observation from
  the relative-rise claim being tested, not conflated with it.
- **Highest contrast:** the red-vs-yellow shading. Yellow (`hype`) judged
  "not significant" by eye — matches the mixed hype numbers above.
- **`during` vs. baseline readable by position alone**, without the
  legend — the point of the plot doesn't depend on colour lookup.
- **Fixes applied, student-directed:**
  1. Legend removed entirely, replaced with direct in-graph text labels
     (`daily`, `7-day average`, `chat's overall daily average`) stated
     once on the first panel only — same colour coding holds across all
     three, so no need to repeat.
  2. `during tournament` labelled directly inside the shaded band on
     **every** panel (student's specific request) — that band is the
     actual claim, so it doesn't get the "state it once" treatment the
     other labels got.
  3. Single shared y-axis label (`subplot_ylabels=["messages per day", "",
     ""]`) instead of three repeated ones.
- **Verdict:** legend-free version re-executed clean, all four labels
  legible and non-overlapping (first attempt placed the `daily` label
  under the baseline label's box — moved to a clear stretch of line after
  the tournament ends).

## Stage 6 — Verification (done)

- **Null named by student:** no particular rise in messages during these
  periods compared to others.
- **Confound named by student:** seasonality — summers generically more
  active than winters (Euro windows are summer; the World Cup window
  sits right before the December holidays), either of which could
  produce a fake football effect on its own.
- **Check run (student chose to build it now, not defer it):**
  seasonality-controlled shuffle test in `03.3-events-in-your-chat.ipynb`
  — each tournament's `during` mean compared against the *same calendar
  dates in every other available year* (not random dates from anywhere
  in the dataset), excluding any comparison year overlapping one of the
  3 real tournament windows.
- **Result:** **0 matches** out of 3 (Euro 2020), 5 (World Cup 2022), and
  4 (Euro 2024) comparison years — no other year's equivalent calendar
  dates came anywhere close to the real tournament window's activity, for
  any of the three. Directly answers the seasonality concern raised,
  without needing a broader confounder sweep.
- **Limitation named, not glossed over:** no 4th tournament exists in
  this data to hold out and replicate the pattern on — the check is
  strong for what it tests (seasonality), not a full replication.

### Verdict on the stage-1 proposition

**Holds, 3/3.** Messages per day rise during all three football
tournament windows relative to this chat's overall baseline (10.1 →
18.9 / 21.0 / 19.9), and the rise survives the one confound actually
named (seasonality) with zero matching comparison years. The
**hype-phase prediction does not hold** (2/3 show no ramp-up before
kickoff) — a real, stated null on a smaller claim, not a failure of the
main one. First draft complete: `03.3-events-in-your-chat.ipynb`,
reproducible top to bottom (also fixed a pre-existing `timestamp`
dtype bug blocking the whole notebook, not just this analysis).

### Post-verdict polish (student-directed, after seeing the finished chart)

- **Gold `hype` shading removed from all three panels.** The hype-phase
  numbers never showed a real, consistent shift (Stage 6's own finding —
  2/3 tournaments flat at baseline), so shading it as if it were a
  finding on the same footing as `during` was overstating it. The
  underlying `hype`/`phase` data and the numeric table are unchanged;
  only the visual band is gone.
- **All direct labels (`during tournament`, `7-day average`, `daily`,
  `chat's overall daily average`) consolidated onto the first panel
  only**, rather than repeating `during tournament` on all three — same
  claim, same colour coding, stated once.
- Re-executed clean, no errors.

### Second polish pass (student-directed)

- **Title made catchy:** "Football tournaments bring activity to the group
  chat" — states the finding as a headline, not a description of the
  chart. Same room-for-a-conclusion move as the categories-lesson slide
  titles.
- **Seasonality check surfaced as an on-chart footnote**, not left only
  in the notebook prose or this log — a reader looking at the chart alone
  now sees "0 of 3 / 5 / 4 comparison years came close," not just the
  main line.
- **X-axis removed entirely** (`set_xticks([])`, `set_xlabel("")`) —
  each panel's title already names the period, so a date axis under it
  repeated information without adding any. Had to explicitly clear
  `xlabel` after every `sns.lineplot` call, since seaborn re-fills it
  from the column name on each draw even after `PlotSettings(xlabel="")`
  set it blank at figure creation.
- **Layout fix needed for the footnote:** matplotlib's `constrained`
  layout engine doesn't reserve space for a bare `fig.text()` on its
  own — first attempt had the footnote overlapping the bottom axis
  border, and reserving bottom margin alone then pushed the suptitle
  into the panel titles. Fixed with
  `fig.get_layout_engine().set(rect=(0, 0.08, 1, 0.93))`, reserving both
  a top and bottom strip explicitly.
- Re-executed clean, no errors.

### Final draft artifacts (saved to `notebooks/analysis/`, same home as Analyses 1–4)

- **`notebooks/analysis/football-tournaments-activity.png`** — the final
  chart, extracted straight from the executed notebook's output, not
  re-rendered separately.
- **`notebooks/analysis/03-football-tournaments.ipynb`** — a duplicate of
  `notebooks/lesson3/03.3-events-in-your-chat.ipynb` (full notebook,
  lesson worked-examples included), so this analysis's methodology lives
  alongside `01-lockdown-activity.ipynb` and `02-election-length.ipynb`
  rather than only inside the lesson folder. The lesson copy stays the
  one referenced by `3.3.6.1`'s write-up; the two are duplicates as of
  this save, not kept in sync automatically — if either is edited later,
  the other won't pick it up.

### Parked for later (explicitly deferred, not this draft)

- **Content check, not just volume:** is the *talk* during these windows
  actually more about football (World Cup / Euro specific terms), or just
  louder in general? Requested by the student as a follow-up, not built
  now — would need a keyword/regex feature (lesson 1's `RegexFeature`)
  compared `during` vs. `baseline`, with the same "boring result" guard
  the categories-lesson planning-message regex needed: a raw count would
  just track overall volume, so it would need to be a *share* of that
  window's messages, not a raw count.

### Feedback received on the finished chart (parked, not acted on)

Logged in `notebooks/analysis/feedback-football-tournaments.md`, same pattern
as Analysis 4's `feedback-comparing-categories.md` — a per-chart feedback file
to pick back up later, not acted on yet:

1. The trend isn't equally strong across all three tournament panels by eye,
   even though Stage 6's numeric verdict called it "holds, 3/3."
2. Overlay the three tournaments on one graph, aligned by tournament day
   (`t=0` = start of tournament, `t+1` = second day, ...) instead of three
   separate calendar-date panels, to make the "tournaments bring activity"
   claim land more directly.
3. Check whether the Netherlands' own matches specifically drive activity
   (not just the tournament in general) — especially the World Cup 2022
   Netherlands–Argentina match — and mark NL match days as dashed vertical
   lines on the chart. No match-level data exists in the pipeline yet; needs
   a new lookup table (date/opponent/stage/outcome per NL match).
4. Plot count minus a daily-average baseline instead of raw count, to
   control for seasonality directly rather than relying on Stage 6's
   separate confound check. Open question logged in the feedback file: a
   flat overall-average baseline doesn't actually control for seasonality,
   a per-calendar-date seasonal baseline (close to what Stage 6's shuffle
   test already computes) does.
5. Check time-of-day (not just daily volume) during tournaments vs. other
   weeks — idea: people chat later in the day/evening during tournaments.
   Same "daily rhythm" question already parked in Analysis 1's Stage 4, now
   for tournament windows. **Caveat named by the student:** World Cup 2026
   isn't in the tournament reference table yet even though it falls inside
   the data window, and its matches were played at night in European time
   (different host time zone than the other three tournaments) — a
   cross-tournament send-hour comparison needs to exclude WC 2026 or align
   by hours-relative-to-kickoff, not raw clock-hour, or the time-zone shift
   could be mistaken for the behavioural effect being tested.

---

# Method ideas parked for later (not tied to one analysis)

Broader methodological ideas raised in feedback, not chart-specific fixes —
kept separate from the per-chart feedback files above since they'd change
*how* rhythm/periodicity gets found in future analyses generally, not just
one existing chart.

## Autocorrelation to find rhythm, instead of eyeballing it

Idea: use autocorrelation (e.g. the ACF of the daily message-count series) to
find recurring rhythm/periodicity in the chat directly from the data, rather
than only checking rhythm around already-known candidate events (tournaments,
elections, lockdowns).

**Where this connects to what's already logged, to think about when
revisiting:**
- **Analysis 1's still-open, parked hypothesis** ("recurring spike events,"
  e.g. bad-news responsiveness) was only ever eyeballed — spikes in 2022 and
  2023 were noticed by looking at the chart, not measured. Autocorrelation
  (or a periodogram/spectral view) could give an actual measured periodicity
  instead of an impression, and might either support or undercut that
  parked hypothesis.
- **Item 5 above** (time-of-day "rhythm" during tournaments) is currently
  scoped as a distribution comparison (`during` vs. `baseline` send-hour
  histograms) — autocorrelation is a different, complementary way of
  characterizing rhythm (e.g. daily/weekly periodicity in the *count*
  series itself) rather than a single window's hour distribution. Worth
  deciding whether these stay separate checks or get combined.
- **Items 4/5's seasonality work** currently proposes a per-calendar-date
  seasonal baseline (reusing Stage 6's comparison-year approach). An
  autocorrelation/ACF view could reveal the dominant cycle length (e.g. a
  clean weekly 7-day peak) more directly, which could simplify or validate
  that seasonal-baseline approach rather than assuming which periods to
  compare against.
- Not yet decided: which series to run this on (whole-chat daily counts vs.
  just the tournament windows), what library/method to use (a plain ACF
  plot vs. a formal seasonal decomposition like STL), and what would count
  as a large-enough effect to act on — none of this has been scoped yet,
  this is a method to explore, not a specific chart to build.

---

# Analysis 6 — does the Dutch team's own tournament run explain the within-tournament shape?

New goad cycle, prompted by feedback item 1 on Analysis 5's chart
(`notebooks/analysis/feedback-football-tournaments.md`) — "the trend isn't
strong across all tournaments." Coached via `goad_analysis_checklist`
(the six-stage tool this project's `CLAUDE.md` designates for analysis work).
Same chat data, same three tournament windows as Analysis 5
(Euro 2020, World Cup 2022, Euro 2024).

## Stage 1 — Question (done)

- **Origin, named honestly:** data-suggested, not prior knowledge — the
  suspicion came from looking at Analysis 5's finished chart (the
  mid-tournament dip in two windows, the steady climb in the third, and the
  7-day average staying elevated above baseline after World Cup 2022 ended),
  not from an expectation held before seeing it. Per this log's own standing
  discipline (Analysis 4's same caveat), a second look at that same chart
  doesn't count as independent confirmation — this needs its own check
  against something the original chart didn't already show.
- **The actual suspicion, more specific than "weaker/stronger":** the three
  tournament windows don't just differ in overall size, they differ in
  *internal shape*. Euro 2020 and World Cup 2022 both show a rise in
  week 1–2 followed by a mid-tournament drop; Euro 2024 shows a steady
  increase throughout instead.
- **Proposed mechanism, per tournament:** the Netherlands men's team's own
  run through each tournament, not the tournament in general.
  - Euro 2024: an unexpected deep run, building growing hype over time —
    matches the steady climb.
  - World Cup 2022: the dramatic NL–Argentina match plus a subsequent
    rise-drop-rise-again pattern.
  - World Cup 2020 (Euro 2020, played 2021): an early, unexpected NL exit.
  - World Cup 2026 (not yet in this analysis's data or reference table,
    flagged again as it was under Analysis 5's item 5 feedback): an easy
    group stage followed by another early, unexpected exit.
- **Boring result named and guarded against:** three small-n (n=3) windows
  will never look visually identical purely by chance — some unevenness is
  expected even under a real, consistent effect. The bar for "a real
  inconsistency" is set at one tournament showing essentially no rise over
  baseline at all, not merely a different internal shape or timing.
- **Proposition:** World Cup 2022 produces a smaller overall increase in
  activity than the two Euro tournaments (not just a different internal
  shape — an actual smaller net effect).
- **Falsification:** if all three tournaments show a comparably significant
  increase in daily messages over baseline, that says no — the shape
  difference would then be interesting on its own but wouldn't support "one
  tournament is genuinely weaker."
- **Independent check required, because the origin is data-suggested:** this
  needs evidence beyond re-reading the same aggregate chart — e.g. checking
  whether the shape difference lines up with actual NL match dates/results
  (feedback item 3's not-yet-built match-level lookup table) rather than
  just eyeballing the existing three panels again.
- **Audience / time budget:** same as Analysis 5 — course group + teacher,
  legible to a non-technical audience; this is a deferred follow-up, not
  the in-class deadline work, so no hard time pressure named yet.

## Stage 2 — Data (in progress)

- **New feature (theme 1 — NL trajectory) → new match-level lookup table,
  student-confirmed** (web-sourced, one correction applied: Morocco match
  moved from 2026-06-29 to the confirmed 2026-06-30):

  **Euro 2020 (played 2021), Group C:**
  | Date | Match | Result |
  |---|---|---|
  | 2021-06-13 | Netherlands – Ukraine | 3–2 W |
  | 2021-06-17 | Netherlands – Austria | 2–0 W |
  | 2021-06-21 | North Macedonia – Netherlands | 0–3 W |
  | 2021-06-27 | Netherlands – Czech Republic (R16) | 0–2 L — **eliminated** |

  **World Cup 2022, Group A:**
  | Date | Match | Result |
  |---|---|---|
  | 2022-11-21 | Senegal – Netherlands | 0–2 W |
  | 2022-11-25 | Netherlands – Ecuador | 1–1 D |
  | 2022-11-29 | Netherlands – Qatar | 2–0 W |
  | 2022-12-03 | Netherlands – United States (R16) | 3–1 W |
  | 2022-12-09 | Netherlands – Argentina (QF) | 2–2 aet, lost 3–4 pens — **eliminated** |

  **Euro 2024, Group D:**
  | Date | Match | Result |
  |---|---|---|
  | 2024-06-16 | Netherlands – Poland | 2–1 W |
  | 2024-06-21 | Netherlands – France | 0–0 D |
  | 2024-06-25 | Netherlands – Austria | 2–3 L (still advanced, 3rd place) |
  | 2024-07-02 | Romania – Netherlands (R16) | 0–3 W |
  | 2024-07-06 | Netherlands – Turkey (QF) | 2–1 W |
  | 2024-07-10 | Netherlands – England (SF) | 1–2 L — **eliminated** |

  **World Cup 2026, Group F:**
  | Date | Match | Result |
  |---|---|---|
  | 2026-06-14 | Netherlands – Japan | 2–2 D |
  | 2026-06-20 | Netherlands – Sweden | 5–1 W |
  | 2026-06-25 | Netherlands – Tunisia | 3–1 W |
  | 2026-06-30 | Netherlands – Morocco (R32) | 1–1 aet, lost on pens — **eliminated** |

- **Provenance:** web-sourced (Wikipedia, UEFA, FIFA, ESPN, Fox Sports,
  Al Jazeera — see chat for full source list), then checked against the
  student's own memory, which corrected one date (Morocco: 2026-06-29 →
  2026-06-30). Same "verify before relying on it beyond a first draft"
  caveat as this log's other reference tables, but now partially
  student-verified rather than purely researched.
- **Missingness:** none — all matches accounted for across all four
  tournaments, including each tournament's elimination match.
- **Not yet decided:** what derived feature this match table should produce
  for the daily-count analysis — a simple `is_nl_match_day` boolean, or a
  richer categorical (`match_type`: group/knockout, or
  `is_elimination_match`) that could test the "unexpected early exit
  suppresses/spikes activity differently than a deep run" mechanism more
  directly than a flat boolean would.
- **Decided (Stage 2 closed), scope narrowed after the shape-stage gate
  check below:** one new column joined onto the existing daily-count table
  by exact calendar date — `is_nl_match_day` (boolean, true only on the
  exact date of an NL match). **`match_result` dropped for this pass** —
  student's explicit call: the question right now is only whether NL match
  days themselves see more chat activity, not whether win/draw/loss changes
  the size of the effect. Parked as a named future investigation, not
  abandoned. **Exact-date-only join** — student's explicit call, not
  extending a few days after each match for post-match reaction/aftermath.
- **One row, both tables:** daily-count table stays one row per calendar
  day (unchanged from Analysis 5); the new match table is one row per NL
  match, joined onto the daily table by date.
- **Pipeline placement:** static lookup table, same pattern as the
  tournament/lockdown/election lookups already in this project, joined by
  exact date.

## Stage 3 — Shape (done)

- **Independent units:** all 19 NL match days pooled across the four
  tournaments (4 + 5 + 6 + 4), treated as comparable units from the start
  — group-stage and knockout matches not split out for this pass. A real
  improvement over Analysis 5's n=3/4-tournaments comparison.
- **Common vs. rare:** student expects the "more messages on match day"
  bump to show up on essentially all 19 match days, not just a few
  standout ones.
- **Secondary suspicion, named but explicitly not folded back in yet:**
  losses might produce an even bigger spike than wins — "easier to bash
  the team than to celebrate." This is the same `match_result` feature
  dropped from Stage 2, resurfacing as a real hypothesis — parked
  deliberately, not tested this pass, so it doesn't quietly reappear as an
  assumption baked into the chart.
- **Chart shape wanted:** the existing daily time-series view (as in
  Analysis 5), with a vertical line marking each of the 19 match days, so
  a reader can visually check per match whether that day shows a clear
  spike — not a single before/after summary number per match, and not (yet)
  a multi-day before/during/after trajectory per match.

## Stage 4 — Encoding (done)

- **Family confirmed:** time series (same family as Analysis 5) — a
  distributions framing (match-day counts vs. non-match-day counts within
  tournament windows) was sketched as a genuine alternative and explicitly
  parked for a future pass, not built now.
- **The one comparison the plot makes:** each match day against the
  tournament's overall baseline (same flat reference line Analysis 5
  already used) — not match day against its own immediate surrounding
  days.
- **Layout:** extend Analysis 5's small-multiple panels to **four** panels
  (adding World Cup 2026), each with a vertical line marking that
  tournament's NL match days (4–6 verticals per panel), rather than
  switching chart family.

### Build (draft, not yet critiqued)

- **File:** `notebooks/analysis/04-nl-match-days-activity.ipynb` — new
  notebook (not a modification of `03-football-tournaments.ipynb`), built
  by the assistant per the student's request, executed top to bottom with
  no errors.
- **World Cup 2026 added to the reference table** — web-verified official
  start/end: 2026-06-11 to 2026-07-19 (39 days, opening match in Mexico
  City, final at MetLife Stadium).
  The hype-phase machinery from Analysis 5 was dropped for this notebook
  (not needed for the match-day question) rather than extended with a
  fourth hype-start date.
- **19 of 19 NL match days landed inside the chat's date range** (build
  check, printed in the notebook) — the confirmed match table joins onto
  the daily-count table cleanly, exact-date-only as decided.
- **Chart:** four small-multiple panels (Euro 2020, World Cup 2022,
  Euro 2024, World Cup 2026), same style as Analysis 5 (grey daily count,
  crimson 7-day average, shaded `during` band, dashed black baseline line),
  now with a dashed red vertical line (`VerticalDate`) on every one of that
  tournament's NL match days.
## Stage 5 — Critique (done)

- **First impression (student's own read):** too much red — the dashed
  vertical match-day lines dominate the first look and drown out
  everything else on the chart.
- **Grouping mismatch identified:** the shaded `during tournament`
  background band and the dashed match-day lines both read as the same
  red, even though they're conceptually different things (the whole
  tournament window vs. individual match days) — the eye groups them
  together when the claim doesn't.
- **Which element should carry the message vs. which does:** the crimson
  7-day rolling average is the intended data-carrying element, but it
  isn't the only coloured thing — the dashed vertical lines pull focus
  away from it.
- **What to delete:** the shaded crimson `during tournament` background
  band.
- **Claim, stated by the student in one sentence — narrower than Stage 1's
  proposition:** there's more chat traffic on NL match days specifically,
  but **by the student's own visual read this only actually looks true for
  Euro 2024** — the other three panels don't show it as clearly. This is a
  real narrowing worth carrying into Stage 6, not glossed over.
- **Falsification named:** if the days immediately around each match day
  show similarly-sized spikes (not just the match day itself), that would
  say the pattern isn't really match-day-specific.

### Critique round 1 applied

- **Fix:** dropped the shaded crimson `during tournament` background band
  entirely — it was grouping the whole tournament window and the
  individual match days into the same visual colour, which the student's
  own critique identified as the wrong grouping. The `during`-window
  numeric bounds are unchanged in the data, only the shading is gone.
  Re-executed, `notebooks/analysis/04-nl-match-days-activity.ipynb`.
- **Student's second look:** confirmed the band removal wasn't enough —
  the dashed match-day lines were drawn on top of (in front of) the grey
  and crimson data lines, making it hard to actually judge whether a match
  day lines up with a real spike.

### Critique round 2 applied

- **Fix:** match-day markers switched from the toolkit's `VerticalDate`
  (fixed at linewidth=2, full opacity, drawn on top of everything) to a
  plain `ax.axvline` — thin (`linewidth=1`), semi-transparent
  (`alpha=0.35`), and `zorder=1` so the line sits *behind* the grey/crimson
  data lines (default `zorder=2`) instead of in front of them.
- **New chart added, per student request:** a single-panel focus chart for
  Euro 2024 only (the tournament that looked convincing in round 1's
  critique), same round-2 line styling, real x-axis dates kept (no clutter
  problem with only one panel), legend instead of direct labels.
- **File:** same `notebooks/analysis/04-nl-match-days-activity.ipynb`,
  re-executed, both charts render clean.
- **Student's re-critique: line-style fix confirmed as resolved.** The
  thin/semi-transparent/behind-the-data-lines styling fixes the
  "grey lines overwhelmed" problem — no further styling change requested.
- **Student's read of the Euro 2024 focus chart, further narrowing the
  claim:** apart from the very first match, every NL match day shows a
  visible increase in messages — but for at least one match (the second),
  the rise actually lands **the day before** the match, not on the match
  day itself. This directly surfaces a gap named back in Stage 2: the
  exact-date-only join can't see a day-before effect, because it was never
  built to look for one. The claim is narrowing again — from "all 4
  tournaments" (Stage 1) → "really just Euro 2024" (Stage 5 round 1) →
  "most Euro 2024 match days, one match's effect appears the day before,
  not on it" (Stage 5 round 2).

## Stage 6 — Verification (done)

- **Null stated:** NL match days show no increase in messages at all,
  relative to non-match days — matches the falsification criterion named
  in Stage 5.
- **Multiple comparisons, honestly disclosed by the student:** the claim
  kept narrowing at every look — all 4 tournaments (Stage 1) → just
  Euro 2024 (Stage 5 round 1) → most Euro 2024 matches, with a "day before"
  allowance added for one match (Stage 5 round 2). Student named this
  explicitly as making the story *weaker*, not stronger — an honest read,
  not glossed over.
- **Shuffle test:** gut estimate only, not computed — student believes a
  real correlation exists specifically for Euro 2024, expects random days
  would rarely look this strong there.
- **Confounder:** none specifically identified.
- **Held-out check — the decisive one, done by eye across the three other
  tournaments, allowing the same "day of or day before" rule:**
  - **World Cup 2022:** partial support — an increase the day after
    match 1, and on match 5 (the Argentina match). 2 of 5 matches show it.
  - **Euro 2020:** weak/partial — only an uplift between match 1 and 2, and
    after match 4. Not a clean per-match pattern.
  - **World Cup 2026:** **no relationship at all** between match days and
    chat activity.
- **Residual/model check:** not done, same gap as every prior analysis in
  this log.

### Verdict on the Analysis 6 proposition

**Not supported as a general pattern — and this result directly confirms
the original feedback item 1 concern that started this whole cycle**
(`feedback-football-tournaments.md`, "the trend isn't strong across all
tournaments"). The clean day-of/day-before match effect that looked
convincing on Euro 2024 alone does **not** replicate consistently on the
held-out tournaments: partial on World Cup 2022, weak on Euro 2020, absent
entirely on World Cup 2026. Combined with the student's own honest
multiple-comparisons disclosure (the claim only ever got narrower, never
independently confirmed), this reads as "Euro 2024 happened to show a
striking pattern," not "Netherlands match days drive group chat activity."
Per goad's own guidance when verification leaves a pattern in doubt: the
honest next step is a **sharper Stage 1**, not a better chart. Candidate
sharper questions, not started: is there something *specific* to Euro 2024
(the unexpected deep run, building hype match over match) that the other
three tournaments' trajectories don't share — i.e. does the mechanism
depend on the team still being alive and improbably good, not on "a match
happened"? `match_result` (parked since Stage 2/3) would be the natural
next feature to test that.

**Decision:** write this up as an honest "one convincing case, doesn't
generalize" finding — same shape as Analysis 4's own doubtful verdict —
rather than continuing to polish the four-panel chart's visual design.
**The sharper-question follow-up (is it specifically Euro 2024's
unexpected deep run, via `match_result`/stage-reached, not "a match
happened") is explicitly parked here, not started** — student's call:
move on to a new topic rather than keep sharpening this one.

### Files (draft, reflect the state as of Stage 6)

- `notebooks/analysis/04-nl-match-days-activity.ipynb` — four-panel chart
  (round 2 line styling) and the Euro-2024-only focus chart, both
  reproducible top to bottom.

---

# Analysis 7 — the annual friends' weekend and chat activity

New goad cycle, prompted by the student's own idea (alongside a second,
parked candidate — see below). Same 9-person chat data as all prior
analyses. Time budget: ~3 hours, more than the recent tight-evening passes.

## Parked candidate (not started this cycle)

Hour-of-day message distribution, COVID lockdown vs. after — this is the
exact "genuinely separate question" Analysis 1's Stage 4 flagged and parked
without answering. Student chose to run the friends'-weekend idea first;
this stays available as the next cycle after this one.

## Stage 1 — Question (done)

- **Data, for this question:** same chat, but the weekend itself is sliced
  in purely by known external dates — not necessarily discussed in-chat
  (planning messages might reference it, but the slicing doesn't depend on
  that).
- **Known events (dates):**
  | Weekend | Start | End | Notes |
  |---|---|---|---|
  | 2024 | 2024-04-19 | 2024-04-21 | Whole group attended. A member fell and got a facial scar — a plausible confound for that year's after-window specifically (check-in/concern messages, not necessarily "positive interaction"). |
  | 2025 | 2025-09-19 | 2025-09-21 | Whole group attended. |
- **Dynamic:** expected "business as usual" once everyone's home, but with
  photo-sharing/rehashing in the days after.
- **Suspicion, revised after a follow-up round (see below):** message
  volume rises in the month before (planning, location hints, logistics),
  **stays elevated during the weekend itself** (not a dip — even while
  together, in-the-moment sharing/logistics keeps the chat busy), rises
  further in the week after (photos, callbacks), then returns to baseline.
- **Follow-up correction, logged explicitly:** the student's first pass
  through the checklist gave two different shapes for the "during" period
  in the same answer set — rising (from the falsification-criterion
  answer) vs. dipping (from the arc answer). Named back to the student
  rather than silently reconciled; asked directly which one they actually
  believe. **Resolved: rises during too** (not a dip) — logistics/in-the-
  moment sharing keeps volume elevated through the weekend itself.
- **Boring result named and guarded against:** "people text more while
  physically together/planning, then it fades" — true of any trip for any
  group, not specific to this one. The real question needs to be sharper
  than that (the specific before/during/after shape, and the "positive
  interaction" read of the after-bump, are the parts a generic trip
  wouldn't obviously predict).
- **Proposition:** message volume is higher than the group's normal
  baseline in the month before the weekend and during the weekend itself,
  and stays elevated for about a week after, before returning to baseline.
- **Falsification:** if there is no increase in messages before/during the
  weekend compared to the group's normal baseline, that says no.
- **Origin:** lived experience — the suspicion was named before looking at
  any numbers, not suggested by a pattern noticed while skimming the data.
- **Arc:** build-up → elevated during → increase after → back to baseline.
  Student says they would have predicted this shape in advance, not just
  "more talking generally."
- **Audience:** course + teacher + the friend group itself (same as prior
  analyses).
- **Live risks, named now rather than discovered later, not yet resolved:**
  - **n=2 weekend events** in the export window is a thin sample for
    claiming a general pattern — parallel to Analysis 4's n=5-election
    caveat, but thinner still. Whatever this shows will be descriptive for
    these two specific weekends, not evidence of a stable yearly effect.
  - **2024's accident** is a plausible confound specifically for that
    year's after-window — a spike there could be concern/retelling rather
    than "positive interaction," and the two years' after-windows may not
    be comparable for that reason.

## Stage 2 — Data (done)

- **One row = one message** (timestamp, author, text), same as all prior
  analyses.
- **Claim unit = day** (message-count-per-day), not message and not
  person — the proposition is about volume over calendar time.
- **Companion check agreed:** per-author small-multiples, same pattern as
  Analysis 1, to confirm any before/during/after pattern isn't driven by
  1–2 people rather than the group.
- **New features, agreed:**
  | Feature | Definition |
  |---|---|
  | `weekend_id` | categorical: `2024`, `2025`, or `none` — isolates 2024's accident confound from 2025 |
  | `period` | categorical: `before` (60 days pre-start), `during` (start–end inclusive), `after` (7 days post-end), `baseline` (everything else, excluding both events' before/during/after windows so one weekend's halo doesn't bias the other's baseline) |
  | `days_relative` | signed integer, days from that weekend's start date (e.g. −60 to +7), only defined near an event — enables an event-aligned time series (both years overlaid or averaged), the actual chart shape originally asked for, not just 4 bucket means |
- **Window widths — units mixup caught and corrected:** student's first
  answer said "60 month window," which would have swallowed almost the
  entire ~6-year export as "before." Flagged back rather than silently
  fixed; confirmed as **60 days** (wider than Stage 1's original "month
  before," student's deliberate choice). After-window stays **7 days**
  per Stage 1.
- **Missingness:** none expected — counts derived straight from
  timestamp/author, no reason to think 2024/2025 coverage is less
  complete than the rest of the export.

## Stage 3 — Shape (done)

- **Common vs. rare matters here:** Analysis 1 already established this
  chat has birthday/bad-news spike days. A spike landing inside one of the
  before/during/after windows by coincidence would distort the read.
- **Concrete decision:** report **median** (not mean) for period
  comparisons — more robust to a single spike day. Any birthday-collision
  day gets **labeled on the chart, not excluded** — same precedent as
  Analysis 1's "label spike days separately" decision.
- **New feature: `is_birthday`**, built from a 9-alias birthday lookup
  (day/month only, no real names, supplied directly by the student against
  the already-anonymized aliases).
- **Collision check, done — 3 real hits:**
  | Alias | Birthday | Falls in |
  |---|---|---|
  | `fluffy-beaver` | Apr 13 | 2024 before-window (Feb 19–Apr 18) |
  | `effervescent-penguin` | Sep 16 | 2025 before-window (Jul 21–Sep 18) |
  | `striking-rail` | Sep 22 | 2025 after-window (Sep 22–28), day 1 |

  None fall inside either during-window. Student's explicit call: label
  these 3, don't drop them.
- **Independent units per group, checked:** `during` = 6 days total (3+3
  across both events) — notably thin next to `before` (120 days) and
  `after` (14 days). **Decision:** because `during` can't carry equal
  weight as its own bucket, the **event-aligned time series
  (`days_relative`, both years overlaid/averaged) is the primary read for
  Stage 4**; the before/during/after/baseline bucket comparison is a
  secondary supporting view, not the headline chart.

## Stage 4 — Encoding (done)

- **Sketched two families:** obvious (time — `days_relative` event-aligned
  line, 2024/2025 overlaid as two separate lines, not averaged) and an
  alternative (distributions — box/violin per period, showing the full
  spread including Stage 3's birthday outliers). **Student picked the
  time-series draft** after trying both.
- **Single comparison the plot exists to make:** chat activity is visibly
  different during the weekend-away period compared to normal.
- **Axes:** x = `days_relative` (days from/to the weekend), y = raw daily
  message count. Both years overlaid as separate lines, not averaged.
- **New confound surfaced at this stage:** the pre-weekend "before" rise
  could be an aggregation artifact — ordinary weekly
  Friday/weekend-planning chatter ("who's free this weekend?") that
  happens most weeks regardless of a trip, not something specific to
  *this* trip's buildup. Needs checking against day-of-week patterns
  elsewhere in the chat before the pre-weekend rise gets read as
  trip-specific.
- **Sensitivity parameters flagged:** the 60-day before-window width, and
  raw count vs. a day-of-week-adjusted count (given the confound just
  named) — both need a robustness check before the read is trusted.

### Build

- **Notebook:** `notebooks/analysis/05-friends-weekend-activity.ipynb` — loads
  the featured export, reindexes daily counts to a full calendar (zero-message
  days kept, not dropped), builds `weekend_id`/`period`/`days_relative`/
  `is_birthday_collision`, and renders all three stage-4 charts:
  `friends-weekend-timeseries.png` (primary), `friends-weekend-period-medians.png`
  (secondary bucket view), `friends-weekend-per-author.png` (companion).
- **Period-median result (secondary chart):** baseline 4.0, **before 2.0
  (below baseline)**, during 80.0, after 20.5 messages/day. The "before"
  half of the Stage-1 proposition does not show up as a rise at all — a
  real signal, not a plotting artifact, confirmed by the primary chart
  too (the 60-day window before is mostly flat/near-zero with scattered
  unrelated spikes, not a ramp toward day 0).
- **Day-of-week confound check (Stage 4's new question):** before-window
  Friday median (7.5) is somewhat above the chat's overall Friday median
  (5.0), but before-window Saturday median (3.5) is well *below* the
  overall Saturday median (8.0) — no clean "planning chatter" signal by
  day-of-week; doesn't explain away the missing "before" rise either.
- **Per-author companion chart:** the during-window spike appears across
  essentially all 9 authors, not 1–2 people — the "during" effect looks
  like a genuine group-wide pattern.

## Stage 5 — Critique (done)

- **v1 critique:** first impression was "quite a lot of data" — the
  during-window spike went unnoticed at first because the red boundary
  lines blended with the data lines' color/style. Grouping was hard to
  read, especially 3 overlapping birthday text labels. The one comparison
  (during/after increase) wasn't clearly visible.
- **v2 redesign, agreed as clean:** both years in grey (de-emphasized —
  year-to-year comparison isn't the point), `during` shown as a shaded
  pink span instead of blending lines, a dotted baseline-median reference
  line, birthday collisions as single-legend gold star markers instead of
  overlapping text, and the finding written directly on the chart.
- **Sensitivity check run (Stage 4's flagged parameter):** widened
  after-window 7→30 days, narrowed before-window 60→30 days. Found a
  **new birthday collision** — `hypnotic-rabbit` (Oct 19) now falls
  inside 2025's widened after-window. **Real result from widening:**
  after-window median drops from 20.5 (7-day) to 7.0 (30-day) — the
  after-effect is short-lived, decaying back toward baseline within
  roughly 1–2 weeks, not sustained for a month.
- **Bug caught and fixed:** the on-chart annotation was positioned using
  the old 60-day window's coordinates and fell off-screen once the window
  narrowed to 30 days — repositioned.
- **Readability fix:** raw daily spikes judged "harsh to read" at this
  window width. Resolved with a **3-day centred rolling-average line**
  (bold) per year on top of the raw daily points (kept, faded/thin) —
  keeps Stage 3's "label spikes, don't smooth them away" principle intact
  while fixing readability. A small pre-trip ramp is now visible on the
  smoothed line starting around day −5 to −3, not across the full 30-day
  window.
- **Claim the chart supports, locked in by the student:** a sharp rise
  during and right after the weekend, fading back toward baseline within
  ~1–2 weeks — but **no comparable rise in the 30 days before**. A real
  gap against the full Stage-1 proposition, which predicted a rise
  before, during, *and* after.

## Stage 6 — Verification (done)

- **Null, stated concretely:** no significant increase in messages during,
  before, or after these weekends — daily counts are just this chat's
  normal noisy/spiky behavior, unrelated to the trip.
- **Multiple comparisons, honestly disclosed:** only two window-width
  comparisons were tried (60-before/7-after, then 30/30) — both showed
  the same qualitative pattern (during+after elevated, before flat), not
  cherry-picked to find a positive result.
- **Shuffle test — computed exhaustively, not a gut estimate.** Student's
  initial gut guess ("other events like football probably spike this
  often too") was checked directly rather than trusted. Method: excluded
  every day inside either weekend's 30-day-before/during/30-day-after
  window, then took a 3-day sliding-window median across all 2091
  remaining baseline days (~6 years). **Result: zero of 2091 baseline
  3-day windows reach the real during-window median of 80 messages/day
  — the closest is 69** (window ending 2022-08-21), still below. The
  during-weekend spike is **the single most extreme 3-day stretch in the
  chat's entire ~6-year history** — directly contradicts the student's
  gut expectation that comparable spikes happen fairly often.
- **Confounders:** 2024's accident and the Stage-1 boring result ("people
  text more when excited/together, then it fades") remain the live,
  accepted candidates — not further investigated this round, student's
  explicit call.
- **Held-out check:** a genuine third occurrence of this annual weekend
  exists in real life but falls after the chat export's last message
  date — unavailable, honestly noted as a real limitation rather than
  worked around. Partial substitute: both years individually show a
  similar during/after shape on the primary chart (not one year driving
  it), and the per-author companion chart already ruled out 1–2 people
  driving it.

### Verdict on the Analysis 7 proposition

**Partially supported — real for during/right-after, not for before.**
The **during** effect is verified as genuinely unusual, not ordinary
chat noise: the exhaustive shuffle test found it to be the most extreme
3-day stretch in the entire ~6-year chat history (0/2091 comparable
baseline windows). The **after** effect is real but short-lived — elevated
in the first week (median 20.5), fading to only mildly above baseline
(7.0) by 30 days out. The **before** half of the original proposition is
**not supported at either window width tested** (60 or 30 days) — the
before-window median (2.0) sits *below* the overall baseline (4.0), and
the day-of-week confound check (Stage 4) found no clean "planning
chatter" signal to explain even a weak rise. Read plainly: this group's
buildup to the trip, if it exists at all, is a **few-day ramp right
before departure** (visible on the smoothed chart around day −5 to −3),
not a month-long anticipation effect.

**Decision:** write this up as a real, partially-confirmed finding — the
group visibly "shows up" for the trip and its immediate afterglow, verified
as statistically unusual by the shuffle test, but the "excited anticipation
for weeks beforehand" half of the original idea doesn't hold up. Not
pursuing a sharper Stage 1 pass tonight — student's call, given only 2
weekend events exist to test any narrower before-window hypothesis
against.

## Write-up

- **File:** `notebooks/analysis/findings-friends-weekend.md` (+
  `friends-weekend-timeseries.png`, `friends-weekend-period-medians.png`,
  `friends-weekend-per-author.png` exported from
  `05-friends-weekend-activity.ipynb`).
- Walks through: original expectation → primary chart (v2, redesigned per
  stage 5) → per-author robustness check → window-width sensitivity check
  (60/7 vs. 30/30) + day-of-week confound check → stage-6 verification
  (null, multiple comparisons, exhaustive shuffle test, confounders,
  held-out) → the honest partial verdict (during/after real, before not
  supported) → explicit list of what this check did not do.

### Presentation polish (v3), after the write-up was drafted

Requested separately, same pattern as Analysis 4's slide-chart polish passes —
refining the already-verified finding's *presentation*, not re-opening the
analysis.

- **Title sharpened to the actual claim:** "We don't anticipate the friends'
  weekend trip — we react to it," headline/subtitle split (same pattern as
  the election-length slide charts) — bold challenging headline via
  `fig.suptitle`, precise measured claim demoted to an italic subtitle.
- **X-axis:** explicit 5-day tick spacing.
- **In-chart annotation moved to the subtitle**, freeing the plot area.
- **Legend box removed, replaced with direct in-plot labels** — "2024"/"2025"
  at each smoothed line's end, "During the weekend" above the shaded span,
  a leader-line label for the baseline. No legend key to look up.
- **Year colour families clarified:** 2024 = light grey (raw `#cccccc` →
  smoothed `#999999`), 2025 = near-black (raw `#666666` → smoothed
  `#222222`) — raw and smoothed share a hue per year so the faint
  background line is now identifiable.
- **Birthday markers changed** from gold stars (too attention-grabbing) to
  small muted open circles.
- **New shuffle test added for the after-window** (7-day, matching the
  real after-median of 20.5/day), run the same exhaustive way as the
  during-window test: **68 of 2087 comparable baseline windows (~3%) reach
  or beat it** — genuinely different from during's 0/2091. Added to the
  chart footnote and the write-up: during is essentially unmatched
  anywhere else in the chat's history, after is real but not statistically
  rare.
- **Clarified on request:** the 80/day during-figure is the *pooled*
  median across both years' during-days (6 total). Per year they differ
  more than the pooled number suggests — 2024's during-median is 91,
  2025's is 53. Noted in the write-up as a caveat, not hidden behind the
  single pooled number.
- **Layout fix:** switched from `tight_layout(rect=...)` to explicit
  `fig.subplots_adjust(...)` — the rect version kept re-inserting a large
  blank gap between the subtitle and the plot that tight_layout's own
  heuristics wouldn't remove.

### Further polish, after the v3 review

- **"During the weekend" label moved** off the peak datapoints it was
  covering — now sits in open space to the left with a leader line into
  the band, and carries `(n=6 days)` directly, so a reader can judge the
  during-median's small sample size at a glance rather than needing the
  write-up.
- **New 5-day build-up band added** (lighter shade than the during band,
  days −5 to −0.5), labelled "5-day build-up" — makes stage 5's small
  pre-trip ramp finding visible directly on the chart instead of only in
  the write-up text.
- **Explained on request:** the 3-day rolling average is a simple centred
  mean (`messages.rolling(3, center=True).mean()`) — e.g. 2025's day 0
  (53 messages) plots as (41+53+87)/3 = 60.3, averaging the day before,
  the day itself, and the day after.
- **New smoothed-only variant added**, `friends-weekend-timeseries-smoothed.png`
  — same chart with the raw daily lines dropped entirely (birthday markers
  kept, since they flag specific known days rather than general noise).
  Footnote extended to explain the 3-day-average calculation itself, since
  without the raw line next to it the smoothing is no longer self-evident.

### Feedback received on the finished chart (parked, not acted on)

Logged in `notebooks/analysis/feedback-friends-weekend.md`, same pattern as
`feedback-football-tournaments.md`. It is a week 2 (time series) assignment
chart, logged to pick back up later:

1. The story is a good find.
2. The data and visualisation back up the story.
3. The shuffle test should be improved to guard against spurious
   correlation. Right now it checks whether other stretches reach the same
   **level** (avg messages/day). Instead, it should measure the weekend's
   **increase** over its own local baseline (4–5 weeks before + 4–5 weeks
   after), then repeat this for ~10,000 randomly placed fake weekends and
   count how often a similar or higher increase occurs. Open design
   questions (difference vs. ratio, mean vs. median, overlap with the real
   weekends, 10,000 draws vs. ~2,000 possible positions, pooled vs.
   per-year) are listed in the feedback file.

---

# Analysis 8 — hour-of-day distribution, lockdown vs. after

New goad cycle, for the week 4 (distributions) assignment. This is the
**parked candidate from Analysis 1's Stage 4** ("hour-of-day distribution,
lockdown vs. outside — a genuinely separate question, daily rhythm not
overall trend"), picked up now instead of started fresh. Assignment
requirements captured separately in
`notebooks/analysis/week4-distribution-requirements.md`: x-axis must show
every possible option (even zero-count), y-axis must be likelihood/relative
frequency (not raw count), and the deliverable needs a significant,
explainable *shift in shape* tied to an event in time — not a static single
distribution.

## Stage 1 — Question (done)

- **Data:** same chat as all prior analyses (Aug 2020 – Aug 2026, 9
  authors). Domain chosen for the x-axis: **hour of day (0–23)**.
- **Known context reused, not re-derived:** Analysis 1's already-validated
  NL lockdown/restriction window table — main lockdown 2020-12-14 to
  2021-04-28 (incl. curfew), second lockdown 2021-12-19 to 2022-01-14.
- **Suspicion:** during lockdown, working from home removes the
  commute/office schedule constraint — people can stay up later and
  message more during the day. After lockdown ends, people return to a
  structured work schedule — daytime messaging drops, evening messaging
  picks back up.
- **Boring result to avoid** (already named in Analysis 1): "chat quiets
  during work hours, picks up in the evening" — true in general, not
  itself the finding. The finding has to be the *specific* shift between
  periods, not just that a daily rhythm exists.
- **Proposition:** the hour-of-day distribution of messages shifts between
  the lockdown period(s) and the post-lockdown period — during lockdown,
  daytime and late-night hours gain share at the expense of the typical
  evening peak; after lockdown, the shape reverts toward a more
  structured, evening-heavy pattern.
- **Falsification:** if the hour-of-day shape looks basically the same
  during lockdown and after (same peak hours, similar proportions), that
  says no.
- **Origin:** lived experience / a parked idea from a prior analysis
  cycle, not a pattern noticed while skimming the data this time.
- **Audience:** teacher, course/classmates, and the friend group itself —
  same as all prior analyses. Needs to be legible to a non-technical
  audience.
- **Time budget:** ~3 hours for a first draft.

## Stage 2 — Data (done)

- **One row = one message** (timestamp, author, text), same as all prior
  analyses.
- **Claim unit:** the group's hour-of-day distribution as a whole — but
  **pseudoreplication risk flagged and accepted as a check to do**: a
  per-author companion view (small multiples, same pattern as Analysis
  1's robustness chart) is needed to confirm the pooled shape isn't driven
  by 1–2 chatty people rather than the whole group.
- **New features, agreed:**
  | Feature | Definition |
  |---|---|
  | `hour` | 0–23, derived from `timestamp` — the x-axis domain itself |
  | `period` | categorical: `during` (both lockdown windows pooled — student's explicit choice over using only the first/longer window) vs. `after` |
- **Open, deferred to Stage 3/4:** exact cutoff for where `after` starts —
  right when each lockdown ends, vs. only once things fully normalized
  (e.g. ~mid-2022, once travel restrictions were fully lifted
  2022-09-17 per the Analysis 1 table).
- **Missingness:** none expected — `hour` and `period` are both derived
  mechanically from `timestamp`, same provenance as the existing lockdown
  lookup table.

## Stage 3 — Shape (done)

- **Actual numbers, during (n=112 days, 2,049 msgs) vs. after (n=1,088
  days, 16,469 msgs, cutoff = last lockdown end 2022-01-14):**

  | hour | during | after | diff |
  |---|---|---|---|
  | 21 | 11.7% | 7.4% | +4.3pp |
  | 19 | 11.3% | 8.3% | +2.9pp |
  | 13 | 4.5% | 7.0% | −2.5pp |
  | 10 | 4.6% | 6.6% | −2.0pp |
  | 0 | 2.5% | 1.1% | +1.4pp |

- **Student's own-words shape description:** "more active in the evening
  during covid, less active during the day" — an evening hump (19, 21)
  during lockdown that shrinks after; daytime hours (10, 13) thin during
  lockdown, picking up after.
- **Explicitly the OPPOSITE direction of the Stage 1 prediction** (student
  expected daytime to gain *during* lockdown, evening to gain *after*).
  Named back to the student rather than silently reconciled.
- **Student's reaction, not smoothed over:** surprised, but offered a
  plausible alternative mechanism on the spot — post-COVID, hybrid/WFH
  work normalized using WhatsApp *during* work hours, rather than
  lockdown itself freeing up daytime. Read plainly: the after-period isn't
  "back to a 9-to-5 that ignores the phone" so much as "a new daytime
  phone-use habit picked up from the WFH era persisting afterward."
- **Interesting vs. not:** student judged hour 21 as the interesting
  signal, not hour 0 (small base rate both sides, judged less interesting
  than the large evening/day swing).
- **Bucket vs. continuum:** kept `hour` at full 0–23 granularity, not
  bucketed into morning/afternoon/evening/night — required by the
  assignment's "show every possible option" rule, and matches what the
  chart needs to show anyway.
- **Independent units:** 112 during-days vs. 1,088 after-days — both
  healthy, not thin.
- **Per-author evenness** (pseudoreplication check from Stage 2): share
  across 9 authors ranges 5.1%–15.3%, roughly 3× busiest-to-quietest —
  "fairly evenly active" (Analysis 1's assumption) roughly holds on raw
  share. Student still wants the actual per-author hour-shape small
  multiples plotted before fully trusting the pooled shape (not just this
  summary number).
- **Sensitivity check, requested by student:** pushed the "after" cutoff
  later — 2022-01-14, 2022-06-01, 2022-09-17 (travel restrictions fully
  lifted), 2023-01-14. **Pattern is stable across all four** (hour 21 stays
  ~6.9–7.7% after regardless of cutoff; hour 13 stays ~6.4–7.0%).
  **Decision:** use the simplest cutoff (right after the last lockdown
  ends, 2022-01-14) as the primary comparison — not re-litigating this
  further.

## Stage 4 — Encoding (in progress)

- **Sketched two families** (temp scratchpad, not committed): obvious
  (distributions — overlaid PMF, line and grouped-bar variants) and an
  unexpected one (time — a month × hour heatmap over the Oct 2020–May 2022
  transition, framing the shift as continuous rather than one before/after
  split).
- **Heatmap alternative rejected:** with ~30 days per month split across 24
  hours, single spike days dominate individual cells (one cell hit 33% of
  a month's messages) — the same birthday/bad-news spike problem Analysis
  1 already flagged, not a real monthly pattern. Not pursued further.
- **Grouped-bar PMF chosen as primary form** over the line variant — matches
  the assignment's literal "every option shown, dice-roll style" framing
  more directly than a connected line.
- **Single comparison the primary plot exists to make:** does the
  hour-of-day shape differ between `during` and `after`, and specifically
  at which hours.
- **Companion charts added:** (1) a residual/diff panel (during minus
  after, per hour, zero-referenced) — same idiom as
  `03.3-events-in-your-chat.ipynb`'s `SubtractBaseline` move, makes the
  *shift* itself the direct subject of a chart; (2) per-author small
  multiples (same pattern as Analysis 1) — the Stage 2/3 pseudoreplication
  check.

### Build

- **Notebook:** `notebooks/analysis/06-lockdown-hour-of-day.ipynb` (new,
  numbered after the existing 01–05 course-adjacent notebooks). Loads
  `load_own_chat()`, applies the same `timestamp`-string-to-datetime coercion
  as `03.3`, adds `hour` via `TimeFeatures`, tags `period` via `FlagDates`
  over the pooled lockdown windows.
- **Images saved:** `hour-of-day-lockdown-vs-after.png` (primary),
  `hour-of-day-lockdown-diff.png` (residual), `hour-of-day-lockdown-per-author.png`
  (robustness).
- **Per-author robustness read:** elevated evening share (peaking somewhere
  in the 16–22 range depending on the author) shows up for most of the 9
  people, not just 1–2 — `humorous-stingray` (peak ~23% at hour 19) and
  `hypnotic-rabbit` (peak ~21% at hour 21) are the most extreme individual
  swings, but the direction is broad-based across the group.
- **Spike-day check, run before trusting hour 21 (Stage 5 honesty check,
  done early):** of 242 during-lockdown messages at hour 21, **86 (35.5%)
  come from a single day: 2021-03-17.** That date is the **2021 Dutch
  general election** already in this project's own election-date table
  (Analysis 4/6). **This is a real, unresolved confound** — a meaningful
  chunk of the headline hour-21 spike is election-night chatter, not
  generic "people stayed up later during lockdown." Not yet decided how to
  handle (exclude the day, label it, or accept it as a partial explanation)
  — needs the student's call before Stage 5 critique is complete.

### Correction: election days excluded from both sides

- **Student's call:** exclude the confound day — and checked whether other
  elections (this project's own 5-election table from Analysis 3/4/6) could
  have a similar effect, on **either** side of the during/after split, not
  just `during`.
- **Checked which of the 5 election dates actually have messages:**
  2021-03-17 (173 msgs, falls in `during`) and 2023-11-22 (68 msgs, falls in
  `after`) do; 2020-11-03 (1 msg) falls in neither window as defined;
  2024-11-05 and 2025-10-29 have **zero** messages that exact day.
- **Both real election days excluded, one from each period, for a fair
  correction — not one-sided.**
- **Result:** hour 21's gap **collapses from +4.3pp to about +1pp** — most
  of the original hour-21 spike was the 2021 election, not lockdown. **Hour
  19 survives essentially unchanged** (+2.9pp → ~+3.0pp) — not an election
  artifact.
- **Revised headline, narrower and more honest than the first read:** the
  lockdown hour-of-day effect is real but centered on **hour 19**, not on
  both 19 and 21 as first thought.
- **Chart updated:** `hour-of-day-lockdown-vs-after-v2.png` is now the
  primary chart (v1 kept in the notebook/folder as the pre-correction
  record, same precedent as the friends-weekend v2/v3 versioning).
- **Notebook clarity fix** (student feedback: "hard to estimate a
  distribution from these charts, maybe only the bar chart"): added an
  explicit note under the diff and per-author charts that they are
  supporting/robustness evidence, not meant to be read as a distribution on
  their own — the grouped-bar chart is the one that answers "what does the
  distribution look like."

## Stage 5 — Critique

**Deferred, not skipped.** Student asked for a `checklist.md`-guided
walkthrough of the corrected chart, but Stage 6's verification result (below)
reshaped the finding enough that critiquing the original chart first would
have been wasted effort. Revisit against the final chart if this analysis
gets picked up again — not done as of this entry.

## Stage 6 — Verification (done)

- **Method:** same `NullDistribution`/shuffle-test idiom as the
  friends'-weekend analysis, applied to hour 19 specifically (the survivor
  of the election correction). Unit of observation: the **day**, not the
  message — 111 `during`-days vs. 1,087 `after`-days (election day already
  excluded from each), each day's own share of its messages sent at hour 19.
  2,000 shuffles, seed 42.
- **Result: essentially at chance.** Observed gap ≈ 0 (−0.0051, i.e. `after`
  is negligibly *higher*). Two-sided p = 0.808. The real value sits in the
  middle of the shuffled cloud, not a tail.
- **Why the pooled (message-weighted) and day-level (equal-weight-per-day)
  views disagree, investigated rather than left as a mystery:** most days
  have **zero** hour-19 messages at all — 84/111 during-days, 792/1,087
  after-days. Hour-19 activity is concentrated on a handful of unusually
  active evenings, not a daily habit. **Top 5 during-days account for 73.8%
  of all hour-19-during messages.** The single biggest — 91 messages, more
  than double the runner-up — is **2020-12-14, the first day of the
  lockdown window itself** (near-certainly the lockdown-announcement
  evening). #2 is Christmas Eve 2020 (39 messages).
- **Student's call: report the null honestly (option 1)**, rather than
  reframing the whole analysis around the single announcement-night event.

### Verdict on the Analysis 8 proposition

**Not supported as a sustained pattern.** The pooled hour-of-day comparison
looked like a real shift (hour 19 +3pp, hour 21 +4.3pp before the election
correction), but a day-level null test — the unit-of-analysis-aware version
of the same comparison — puts the real value dead center of the shuffled
cloud (p=0.81). What looked like "lockdown shifted our daily rhythm toward
evening chatting" is, on inspection, two extraordinary evenings (the
lockdown-announcement night, one Christmas Eve) plus one unrelated election
night — not a multi-month condition changing when 9 people chat with each
other.

### Write-up

- **File:** `notebooks/analysis/findings-hour-of-day-lockdown.md` (+
  `hour-of-day-lockdown-vs-after-v2.png`,
  `hour-of-day-lockdown-per-author.png`,
  `hour-of-day-lockdown-hour19-null.png`, all from
  `06-lockdown-hour-of-day.ipynb`).
- Walks through: prediction (opposite of what was found) → primary chart →
  per-author robustness → election-day confound + correction (fair,
  both-sides) → day-level null test → the honest null verdict → what this
  check did not do.

### Parked for next cycle

Student wants to continue looking for **other** distribution-shaped
questions after this one — explicitly interested in a variable/event where a
*sustained* shift (not a couple of loud days) might actually survive a
day-level null test. Candidates offered: message length, day-of-week
instead of hour-of-day, author-share tied to a personal move, or fitting a
distribution family outright (the `04.2`/`04.4` technique, unused this
session). **Student picked fitting a distribution family — continued below
as Analysis 9.**

---

# Analysis 9 — does moving to the city change how much someone posts?

New goad cycle, ~45 minute budget (tighter than Analysis 8's 3 hours).
Picks up the "fit a distribution family" idea parked above, using a fresh
event (personal moves) rather than revisiting lockdown a third time.

## Stage 1 — Question (done)

- **Variable:** daily message count, fit via Poisson/negative binomial
  (the `04.2`/`04.4` technique, unused elsewhere this session).
- **Event:** 4 of the 9 authors have known city-move dates from lived
  experience (real names known to the student, not written here or
  anywhere committed — aliases only, same anonymisation convention as
  every other analysis): `striking-rail` (Aug 2021), `animated-elk` (Sep
  2022), `effervescent-penguin` (Jul 2023), `humorous-stingray` (Sep 2023).
  A 5th person was already city-based before the export started — no
  before-window, excluded.
- **Design, confirmed with student:** fit each mover's own distribution
  before/after **their own** move — 4 separate comparisons, never pooled
  across people. Same precedent as the friends-weekend analysis (overlay
  separate events, don't average).
- **Proposition:** moving to the city decreases a person's own daily
  message rate in the group chat — same predicted direction for all 4.
- **Falsification:** no decrease (flat or higher) for most/all of the 4,
  that says no.
- **Boring result guarded against, named by the assistant and confirmed
  by the student as a real, previously-unconsidered risk:** Analysis 1
  already found this chat's overall volume declining since lockdown ended
  — an individual mover's dip could just be riding that general decline.
  **Control:** compare each mover's own before/after ratio against the
  *rest of the group's* ratio over the identical calendar window.
- **Origin:** lived experience (move dates from memory, prediction stated
  before looking). **Audience/time budget:** same as Analysis 8, ~45 min
  for this one specifically.

## Stage 2 — Data (done)

- One row = one message, same as always. **Claim unit: each mover, judged
  against their own before/after split** — 4 mini-analyses, not n=4000
  messages.
- **Features:** per-mover before window (export start → move date) / after
  window (move date → export end); daily count reindexed over every
  calendar day including zero-message days (needed for an honest rate, not
  just posting days); the same for the rest of the group over the
  identical windows, as the control.
- **Sample-size note:** `striking-rail`'s before-window is short (~1 year,
  move closest to export start) vs. 2–3 years for the other three —
  flagged, not a blocker.

## Stages 3–4 — Shape/Encoding (compressed for time)

Given the 45-minute budget, moved directly to build rather than a full
interview pass — reused `04.4`'s established fit (`DistributionFitter`,
best-likelihood family per group) and a single grouped-bar chart (person's
ratio vs. group's ratio per mover, zero-line at 1.0) rather than per-mover
histogram overlays.

### Build

- **Notebook:** `notebooks/analysis/07-city-move-distribution.ipynb`.
- **Image:** `city-move-rate-ratio.png`.
- **Result:**

  | mover | before rate | after rate | own ratio | group ratio (same window) | relative |
  |---|---|---|---|---|---|
  | striking-rail | 0.82 | 0.80 | 0.98 | 0.71 | 1.38 |
  | animated-elk | 1.68 | 0.95 | 0.56 | 0.72 | 0.78 |
  | effervescent-penguin | 1.49 | 1.06 | 0.71 | 0.71 | 1.00 |
  | humorous-stingray | 1.38 | 1.11 | 0.80 | 0.69 | 1.16 |

  Group's own ratio is consistent (~0.69–0.72) across all 4 different
  calendar windows — a nice consistency check against Analysis 1's
  declining-trend finding, computed 4 independent ways.

## Verdict

**Not supported as a uniform pattern.** Only `animated-elk` shows a
move-specific extra decline beyond the group's general trend.
`effervescent-penguin` tracks the group exactly. `striking-rail` (the
student's own data) shows **no decline at all**, contradicting the
prediction. `humorous-stingray` declined slightly *less* than the group.
**Student's explicit call:** stop here rather than push further robustness
work on a result that didn't hold up uniformly — the group-comparison ratio
serves as this pass's verification, no formal permutation test run, given
the tight time budget.

### Write-up

- **File:** `notebooks/analysis/findings-city-move-distribution.md` (brief,
  per student's request) — question → control design → the table/chart →
  the honest non-uniform verdict → what this didn't do (short baseline for
  striking-rail, no formal null test, n=4, no digging into *why*
  `animated-elk` differs).

---

# Analysis 10 — does moving in with a partner shift *which days* someone posts?

New goad cycle, ~3 hour budget. Picks up the distributions assignment
(`week4-distribution-requirements.md`) after Analyses 8 and 9 both came back
null. Lesson carried over: Analysis 8's event (lockdown) was a diffuse,
group-wide condition that pooled away; Analysis 9 compared *rates*, not a
distribution's *shape*, so it could not have met requirement 3 even if
positive. This cycle picks an event that directly changes one person's daily
constraints on a known date.

## Stage 1 — Question (done)

- **Candidates named by the student:** one member moving abroad, people
  starting jobs, people moving in with a partner. **Picked: moving in with a
  partner.**
- **Mechanism (student's words):** once living with a partner, the time after
  work goes to the partner — so texting shifts toward the weekend rather than
  weekday evenings.
- **Variable:** day-of-week share (all 7 days on the x-axis, likelihood on the
  y-axis). **Weekend = Fri + Sat + Sun**, fixed by the student before looking.
- **Parked:** "fewer messages overall after moving in" — a rate question, not
  a shape question, and Analysis 9 already showed the group's general decline
  dominates rates.
- **Proposition (student):** after moving in, a person's weekend share of
  messages increases significantly compared to the rest of the group.
  - Operationalised, agreed: *(mover's after − before change in Fri–Sun share)
    − (rest of group's change over the identical windows)*, per mover.
  - "Significantly": placebo move dates (same statistic at fake dates across
    the timeline), agreed in advance for Stage 6.
- **Falsification:** no such relative increase for most movers.
- **Boring result:** (b) weekend share rising for *everyone* over 2020–2026 —
  group drift the movers would just ride along. The rest-of-group control
  exists to rule this out. Student doesn't yet know whether it's true. Also
  named: "everyone texts more on weekends" is static, not the finding.
- **Move-in dates (aliases only, month precision, from memory):**

  | alias | moved in | job start nearby |
  |---|---|---|
  | striking-rail | 2025-08 | no |
  | pliable-tiger | 2026-08 | no |
  | rib-tickling-curlew | 2025-11 | yes, 2025-09 (just before) |
  | fluffy-beaver | 2026-01 | no |
  | humorous-stingray | 2023-07 | yes, 2023-09 |
  | effervescent-penguin | 2024-07 | yes, 2024-09 |
  | hypnotic-rabbit | 2025-07 | no |

- **Confound flagged, accepted:** a job start within ~3 months of the move
  (curlew, stingray, penguin) would push messages toward the weekend by the
  same mechanism — a before/after split can't separate the two.
- **Origin:** lived experience, prediction stated before looking.
  **Audience:** teacher, classmates, the friend group. **Budget:** ~3h.

## Stage 2 — Data (done)

- **One row = one message.** **Claim unit = the mover** — n = 6. Within a
  person, the independent unit is the **day**, not the message (Analysis 8's
  lesson).
- **pliable-tiger dropped as a mover** (only 1.2 months of data after the
  08/2026 move; export ends 2026-09-08), but **kept in the data as a control**
  — their move falls after every other mover's window.
- **Alias confirmed:** "Fluffy Bear" = `fluffy-beaver`.
- **Features, agreed:**

  | feature | definition |
  |---|---|
  | `weekday` | 0–6 from `timestamp` — parquet times verified to equal the raw export's local clock times (`+00:00` is only a label) |
  | `is_weekend` | Fri, Sat or Sun |
  | `period` | `before` / `after`, per mover, symmetric **±6 months** around the 1st of the move month |
  | `role` | `mover` / `control` |

- **Control refined (student's pliable-tiger question surfaced it):** four
  moves cluster in 07/2025–01/2026 (rabbit, rail, curlew, beaver), so "rest of
  the group" would contain other movers mid-shift and shrink the difference.
  **Control = only authors with no move-in of their own inside that mover's
  ±6-month window.** Student agreed.
- **Confounds:** humorous-stingray's move-in *is* their city move (same event,
  confirmed) and has a job at 09/2023. Job-confounded movers (curlew, stingray,
  penguin) **kept, but shown visually separate** from the 3 clean ones (rail,
  rabbit, beaver).
- **Considered and rejected:** switching to total volume because per-person
  counts are small. Volume changes the hypothesis (how much vs. when) and
  overlaps Analysis 9; the evidence here comes from consistency across 6
  movers + the placebo test, not per-person precision. **Student chose to
  keep day-of-week.** Messages-per-day count distribution parked.

## Stage 3 — Shape (done) — checkpoint reached, effect not visible

- **First look:** `notebooks/analysis/move-in-weekday-first-look.png` (scratch,
  throwaway script — not the deliverable). Six small multiples, ±6 months,
  mover before/after bars + control lines.
- **Two measures of Fri–Sun share:** per message, and **per day** (share of
  active days that fall on Fri–Sun, each day counted once). **Student prefers
  per day** — it's what "shifting *when* you text" means, and it neutralises
  loud days.
- **Numbers (difference = mover's change − control's change):**

  | mover | job? | per message | **per day** |
  |---|---|---|---|
  | striking-rail | – | −0.14 | +0.01 |
  | hypnotic-rabbit | – | −0.11 | −0.07 |
  | fluffy-beaver | – | −0.16 | −0.06 |
  | rib-tickling-curlew | yes | −0.03 | +0.00 |
  | humorous-stingray | yes | −0.30 | −0.21 |
  | effervescent-penguin | yes | −0.01 | +0.02 |

- **Extreme value:** humorous-stingray, Sunday 2023-06-25 = 52 of their 197
  before-window messages (26%; group sent 100 that day). Student: an
  extraordinary loud day, not an error. The per-day measure counts it once.
- **Student's own reading:** "no clear picture across 6 graphs … roughly half
  more weekend, others more weekday … no clear effect to be honest."
- **Corrections to readings, from the assistant:** curlew's weekend share
  actually *rose* (0.59 → 0.67 per message), just less than their control's;
  the control lines are *the other people*, whose weekend share moved up to
  14pp in one window on its own — so a single-person shift of ~10pp is within
  normal group variation.
- **Against the Stage 1 falsification rule:** no mover shows the predicted
  increase beyond ~2pp on the per-day measure; if anything the sign is
  negative. The proposition is heading toward "no".

### Analysis 10 — status at the checkpoint

**Paused, not finished.** Student chose to switch (option B) rather than
finish this as a third null (the assignment's requirement 3 asks for a
significant, explainable shift). Chart/placebo/write-up not built.

**Exploratory lead, parked for later (student's request):** per *day*, movers
show up on Fri–Sun just as often after moving in; per *message*, all 6 movers'
weekend share fell more than their control's (clean movers −0.11, −0.14,
−0.16). Candidate story: *after moving in, people still turn up on weekends
but write less on those days.* **Data-suggested** — found on the same data it
would be tested on, n = 3 clean movers — so only valid as a lead to check on
other data (e.g. pliable-tiger once post-move data exists), not a conclusion.

---

# Analysis 11 — does starting a job move chat time from daytime to evening?

Pivot inside the same ~3h session (~1h30 left at the start of this cycle).
Setup (windows, control rule, per-day measure, clean vs. confounded shown
separately, placebo test) carried over from Analysis 10.

## Stage 1 — Question (done)

- **Event:** starting a job. (Option C — a member moving abroad — dropped:
  the move was within Europe, so no timezone shift.)
- **Mechanism (student):** once working, fewer daytime messages on workdays,
  more in the evening, fewer late at night.
- **Student asked whether this is boring, and proposed flipping it** to
  "activity increases while everyone should be working." Discussed: *boring*
  = the static "chat quiets during work hours"; this is a within-person
  shift at a dated event, which is exactly requirement 3. Analysis 8's own
  finding (daytime share higher post-lockdown — hybrid-work phone use) makes
  the outcome genuinely open. Flipping a believed prediction to get a better
  story = prediction-shopping. **Student kept the original prediction.**
- **Framing, two outcomes, both a story:** *Does starting a job push your chat
  time from daytime to the evening — or do people just keep chatting during
  work, as Analysis 8 hinted?*
- **Primary statistic (one, to avoid multiple testing):** per active weekday,
  share of that day's messages sent **09:00–17:00**, averaged per period;
  *(starter after − before) − (control after − before)*. **Prediction:
  negative.** Other hours shown in the chart, not tested.
- **Weekdays = Mon–Fri** (differs from Analysis 10's Fri–Sun weekend on
  purpose — a job runs Mon–Fri). Weekends kept as a later internal check.
- **Falsification:** the weekday hour-of-day shape stays the same.
- **Origin:** lived experience, job dates from memory (month precision),
  prediction stated before looking.

## Stage 2 — Data (done, reused from Analysis 10)

| alias | job start | status |
|---|---|---|
| striking-rail | 2023-01 | clean |
| pliable-tiger | 2024-09 | clean |
| vibrant-barracuda | 2022-09 | clean |
| animated-elk | 2022-09 | confounded — city move same month |
| humorous-stingray | 2023-09 | confounded — move-in/city move 2023-07 |
| effervescent-penguin | 2024-09 | confounded — move-in 2024-07 |
| rib-tickling-curlew | 2025-09 | confounded — move-in 2025-11 |
| fluffy-beaver | 2020-09 | **dropped** — export starts 2020-08-12 |
| hypnotic-rabbit | before 2020 | no event — control |

- Filter Mon–Fri; `hour` 0–23; `work` = 09:00 ≤ t < 17:00; ±6-month windows
  around the 1st of the job month.
- **Control = authors with no event of their own (job, move-in, city move)
  inside that starter's window.** Same-month starters (barracuda/elk,
  tiger/penguin) are automatically excluded from each other's controls.

## Stage 3 — Shape (done)

- **First look:** `notebooks/analysis/job-start-hour-first-look.png` (scratch).
- **Per-day work-hours (09–17) share, difference vs. control:**

  | starter | status | ±6 (primary) | ±12 (sensitivity) |
  |---|---|---|---|
  | striking-rail | clean | −0.06 | −0.06 |
  | pliable-tiger | clean | +0.12 | +0.02 |
  | vibrant-barracuda | clean | **−0.12** | −0.01 |
  | animated-elk | ⚠ | +0.13 | +0.06 |
  | humorous-stingray | ⚠ | +0.00 | −0.03 |
  | effervescent-penguin | ⚠ | +0.16 | −0.07 |
  | rib-tickling-curlew | ⚠ | −0.06 | −0.12 |

- **Student's reading:** vibrant-barracuda clearly shifts toward the evening,
  less active during the day; others show no real difference. Assistant
  corrected curlew: the tall 13:00 bar after the start is a few busy
  lunchtimes; per day, curlew's work share *fell* (0.59 → 0.48).
- **Thin after-windows:** pliable-tiger 25 and effervescent-penguin 28 active
  weekdays after the start (of ~130). Student asked about widening the window;
  flagged that choosing the window after looking is a forking path. **Decision:
  ±6 stays primary (fixed before looking); ±12 reported as sensitivity.**
  ±12 flips the count (5/7 negative, all small) and barracuda's shift
  disappears — it seems to fade after the first months. The window is the
  parameter that changes the conclusion; it goes in the write-up.
- **Buckets for the chart, agreed:** night 00–06, morning 06–09, **work
  09–17**, evening 17–24. The work bucket *is* the primary statistic, so
  bucketing changes the display, not the result.
- **Student's provisional answer:** "people keep chatting during work" — only
  barracuda shows a clear evening shift.

## Stage 4 — Encoding (done)

- **Sketched two families** (`sketch-A-job-buckets-per-starter.png`,
  `sketch-B-job-work-share-difference.png`, scratch):
  - **A — distributions:** 7 small multiples, 4 buckets, share of an active
    weekday's messages, before vs. after, control as black ticks.
  - **B — categories (the stretch):** sorted dot plot of each starter's
    difference-vs-control in work-hours share, zero line, ±12 as diamonds.
- **Student chose A** — shows best what we're visualising, and meets the
  assignment's "every option on x / likelihood on y". B answers the
  two-outcome question faster but shows a difference, not a distribution.
- **The single comparison (student):** the message distribution before and
  after starting the job, compared with the control.
- **Axes:** x = 4 categorical buckets (equally spaced but unequal in hours:
  6/3/8/7h — to be stated on the chart); y = share, 0–1.
- **Aggregation (student):** an average hides skewness and can be pulled by
  outliers. Added: the per-day average protects against loud days, but a
  1-message day weighs as much as a 50-message day, and the bar hides how
  many active days sit behind it (tiger 25, penguin 28 after).
- **Parameter that changes the conclusion:** the window (±6 vs ±12) —
  **student: report it in the write-up, not on the chart.**

### Build

- **Notebook:** `notebooks/analysis/08-job-start-hour-of-day.ipynb` — runs
  top to bottom; reproduces the Stage 3 numbers exactly.
- **Images:** `job-start-hour-of-day-v1.png` (pre-critique),
  `job-start-hour-of-day-v2.png` (4 buckets), `-v2b-two-buckets.png`,
  **`job-start-hour-of-day-final.png`** (v2b + title), `job-start-placebo-null.png`.
  Scratch, not part of the deliverable: `job-start-hour-first-look.png`,
  `sketch-A-…`, `sketch-B-…`, `move-in-weekday-first-look.png`.

## Stage 5 — Critique (done)

- **First impression (student):** the blue bars — far left (barracuda, tall
  evening) and second from right (penguin, tall work-hours): the two most
  extreme cases, in opposite directions. That *is* the message.
- **Where the opposite would have shown:** clarified together — if the
  prediction held, every panel's after-work bar sits below the before-work
  bar with the control ticks flat.
- **Grouping:** two rows, clean on top / confounded below. Student asked about
  also fading the confounded panels; flagged that it reads as hiding data
  (they're the cases against the original prediction) → rows only.
- **Colour:** only the work-hours bucket in colour (student's call).
- **Buckets:** student proposed work vs. non-work. Flagged: it collapses the
  distribution to one number (non-work = 1 − work) and loses where the
  non-work messages go. Built both; **student chose v2b** (two buckets). v2
  kept in the notebook.
- **Active-days counts:** left off the chart (student's call).
- **Title:** student saw that "users text more in the evening" isn't
  supported; deferred until after the test, then set: **"People do not change
  the moment of texting after getting a job."**

## Stage 6 — Verification (test run; interview questions still open)

- **Null:** placebo job starts — every month where a person's ±6 window fits
  in the export and holds none of their real events (25–49 per person); same
  statistic. Combined: mean over starters vs. 2,000 draws of one placebo per
  starter. One-sided (prediction was negative). Seed 42.
- **Result:**

  | | real | share of placebos at least as negative |
  |---|---|---|
  | vibrant-barracuda | −0.12 | 14% |
  | striking-rail | −0.05 | 16% |
  | pliable-tiger | +0.12 | 88% |
  | animated-elk ⚠ | +0.13 | 90% |
  | humorous-stingray ⚠ | +0.00 | 55% |
  | effervescent-penguin ⚠ | +0.16 | 100% |
  | rib-tickling-curlew ⚠ | −0.06 | 20% |
  | **clean (3), mean** | **−0.02** | **p = 0.30** |
  | **all (7), mean** | **+0.03** | **p = 0.73** |

- Even barracuda's clear shift happens in ~1 of 7 random months for them —
  consistent with it vanishing at ±12.
- **Not yet discussed with the student (Stage 6 interview):** how many
  comparisons were looked at this session (Analyses 10 and 11, two
  measures, two windows), and a held-out slice. Not recorded in goad as done.

### Verdict on the Analysis 11 proposition

**Not supported — the second of the two pre-agreed outcomes.** Starting a job
did not measurably push weekday chatting out of 09:00–17:00: the real job
starts don't stand out from ordinary months (clean p = 0.30, all p = 0.73).
Student's claim: *people do not change the moment of texting after getting a
job.* Against the assignment: requirement 3 asks for a *significant* shift;
this is a well-tested *absence* of one, framed as a two-outcome question
before looking — worth checking with the teacher whether that counts.

### Write-up

- `notebooks/analysis/findings-job-start-hour-of-day.md`.

### Stage 6 — interview completed (supersedes "questions still open" above)

- **Null (student):** neither outcome true — no shift, neither less nor more
  chatting during work hours. Refined together: the null is *the change at
  the real start is no bigger than at a random month* — exactly what the
  placebo months measure. "Keep chatting during work" coincides with the
  null, which is why the two-outcome framing holds up.
- **Comparisons:** student counted 2×2×6 = 24+; ~30 in total (A10 6×2, A11
  7×2, pooled tests, the weekend lead). Many comparisons don't weaken a null.
  **The Analysis 10 weekend lead is the most striking of ~30 looks, found
  after looking → exploratory only**, to be tested on new data.
- **Third variable (student):** the general decline in volume. Share measure
  + same-month control → no direct bias, but fewer messages = noisier shares =
  **less power to detect a real effect**. Also possible: the "before" period
  (internship/thesis) was already work-like.
- **Held-out slice — weekends** (student asked to run it): for
  `vibrant-barracuda`, the only one who shifted, the weekend 09–17 share
  dropped **even more** (−0.19 vs. control; 19/18 weekend days) → their weekday
  shift is not job-specific. Weekend differences across starters range
  −0.19 to +0.26, confirming ±0.12 is ordinary variation.
- **Verdict (student):** no clear distribution shift across the group chat
  after these tests; no significant story on these topics.
- **Title:** "People do not change…" claims proof of no effect; with n = 7 and
  noisy shares only *no detectable change* is defensible. Wording left to the
  student.

---

# Analysis 12 — does moving abroad make someone start more conversations?

New goad cycle, ~1h budget (extendable if results warrant; a second hour is
reserved for another data find). Follows four nulls (Analyses 8–11). Student's
standard, stated before choosing: a chart needs a story a reader gets quickly,
with data behind it — a well-tested null (Analysis 11) is strong analysis but
weak storytelling.

## Stage 1 — Question (done)

- **Event:** `humorous-stingray` moved abroad, 07/2023 (within Europe — no
  timezone shift). It is a **package**: moving abroad + moving in with a
  partner (same event as Analysis 9's city move and Analysis 10's move-in) +
  a new job at 09/2023. The claim is about the package, not one component.
- **Origin: data-suggested.** Student's prediction was *fewer* messages after
  the move (less contact). A scratch first look
  (`notebooks/analysis/move-abroad-first-look.png`, throwaway script, not the
  deliverable) showed the opposite:

  | ±6 months | before | after | ratio |
  |---|---|---|---|
  | stingray, messages/day | 1.09 | 1.89 | ×1.74 |
  | control (6 people), per person-day | 1.17 | 1.23 | ×1.06 |
  | stingray, days with 0 messages | 73% | 57% | |

  Relative ×1.64 (±12 months: ×1.88). The jump is guaranteed to be there —
  it is where it was found — so it is **not** the evidence. Evidence must come
  from measures not yet looked at.
- **Mechanism (student):** WhatsApp replaces seeing the group in person, so
  Stingray proactively shares more.
- **Proposition (student):** after moving abroad, a **larger share of
  Stingray's messages start a conversation** compared with before, by more
  than the control's change over the same months.
  - Conversation start = first message after **>12h** of group silence
    (student's choice; >24h gave only 93 starts in all of 2023 for the whole
    group — too few).
- **Falsifier (option a):** the start share does not rise relative to the
  control. If it *falls*, the answer is "others ask, Stingray replies" — the
  competing mechanism, also reportable. Student first named both mechanisms in
  the punchline; flagged that "both" can't lose, student chose (a).
- **Boring version (student):** "Stingray sends more messages after the move"
  — true, no why.
- **Arc:** expected fewer messages (less contact) → activity went up instead
  → because Stingray shared more about the new life.
- **Audience:** teacher, classmates, the friend group.
- **Scope (1h):** keep the per-day count distribution (main chart, req. 1–3),
  placebo months (significance), conversation-start share (mechanism), other
  job starters' messages/day (reference table). Dropped unless time is left:
  media messages, fade-over-time.

## Stage 2 — Data (done)

- **One row = one message. Claim unit = one person** (`humorous-stingray`,
  n = 1). The claim describes Stingray only; significance has to come from
  within-person comparison (placebo months), not from other people.
- **Features (agreed):**

  | feature | definition |
  |---|---|
  | `is_start` | message after >12h of silence in the whole chat |
  | `period` | before / after, ±6 months around 2023-07-01 (±12 as check) |
  | `role` | Stingray / control (no own life event inside the window; 6 people at ±6m) |
  | `daily_count` | messages per person per calendar day, zero days kept (a real 0, not missing) |
  | placebo months | Stingray windows holding none of their real events |
  | job-starter reference | other 6 starters' messages/day, relative to control |

- **Mechanism measure: (ii) Stingray's share of all the group's conversation
  starts** ("who opens the chat"). Rejected: (i) starts / Stingray's messages
  — dilution: more messages inside conversations can hide more starts;
  (iii) starts per day — rises with volume alone.
- **Note:** a busier Stingray opens more conversations by chance, so the start
  share is read against Stingray's share of *all* messages in the same window.

## Stage 3 — Shape (done)

- **Feature that matters: `daily_count`** (main chart). Shape: most days at 0,
  drops off fast, long thin tail up to 52. Median is 0 in both periods and the
  mean is pulled by loud days → **report the full distribution.**
- **Every value on the x-axis**, tail capped at a **10+** bar (req. 1).
- **Extreme value:** Stingray's 52-message day, 2023-06-25 (before window).
  Student read the day (`data/processed/stingray-2023-06-25.txt`, git-ignored):
  a group trip to Groningen with an overnight stay, Stingray sent many
  pictures. Real, not an error, not about the move → **kept.**
- **Conversation starts are few:** whole group 111 / 98 starts per ±6m
  window; Stingray's count will be small (one start ≈ 1pp). **Main window
  changed to ±12 months** (group 215 / 190 starts). Trade-off accepted: the
  control shrinks from 6 to 4 people (`animated-elk`, `vibrant-barracuda` have
  2022-09 events inside ±12m).

## Stage 4 — Encoding (interrupted → reframe)

- **Sketches** (scratch): `sketch-A-move-abroad-daily-count.png`
  (distributions, ±12m before/after, Stingray vs control) and
  `sketch-B-move-abroad-timeline.png` (time, monthly messages/day, 3-month
  rolling, whole export).
- **Sketch B changed the story (student spotted it):** Stingray was *above*
  the control in early 2022, far *below* it mid-2022 → spring 2023, rising
  from ~April 2023 — before the move. The ±12 "before" window is mostly a dip.
- **Student's context:** internship in Switzerland (H2 2022) → back in NL to
  finish the thesis (early 2023 — still quiet while physically close) → moved
  to Austria with partner (07/2023) → job (09/2023). Stingray was already
  abroad before the "move", so "less contact after moving away" never applied.
- Student chose to **reframe and continue** (over placebo-first or stopping).

## Stage 1 (reframed) — Question (done)

- **Event:** one transition, student life → settled life abroad (thesis end,
  move to Austria, partner, job). Cannot be separated; claim is about the
  transition.
- **Simpler explanation adopted (student):** the dip was the studies, not the
  distance.
- **Proposition:** after settling abroad (07/2023–06/2024), Stingray's
  messages per day **relative to the control** are back at their
  pre-internship level (**01–06/2022**). **Not true if** clearly below (no
  full recovery) or clearly above (something new after the move).
- **Expectation stated before looking (student):** back at the early-2022
  level. The early-2022 comparison has not been looked at yet.
- **Punchline (student kept this wording):** Stingray went quiet during the
  internship and thesis, not because of the distance. Once settled abroad,
  they came back to their old level — further away than ever. Distance didn't
  reduce the chatting; studying did.
- **Dropped:** conversation-start share (the earlier mechanism measure).
- Origin still data-suggested (the jump was seen first); the baseline test is
  the new evidence.

## Stage 2 (reframed) — Data (done)

- One row = one message; claim unit = Stingray (n = 1).
- **Features:** `daily_count` (zeros kept); `phase` = **baseline 01–06/2022 ·
  dip 07/2022–06/2023 · settled 07/2023–06/2024**; `role` = Stingray / control
  (no own event 01/2022–06/2024: `pliable-tiger`, `rib-tickling-curlew`,
  `fluffy-beaver`, `hypnotic-rabbit`); `relative_level` = Stingray's
  messages/day ÷ control's, per phase.
- **Season:** full settled year kept against a Jan–Jun baseline (student's
  choice) — mismatch reported as a limitation.
- **Margin, set before looking (student): 25%** — settled relative level
  within 0.75–1.25× the baseline relative level counts as "back at the old
  level". Stage 6 checks this against Stingray's ordinary half-year swings.
- Transparency: Sketch B already gave a rough impression of early 2022; the
  formal comparison is not yet computed.

## Stage 3 (reframed) — Shape (done)

- Carried over: long-tailed `daily_count`, full distribution, every value with
  a 10+ cap. The Groningen day (2023-06-25) now sits in the **dip** phase —
  it no longer touches baseline vs. settled.
- **Main measure (student): mean messages per day**, relative to the control
  — captures how much Stingray says, but loud days can pull it. **Check:**
  share of days posting at all (robust to loud days).
- Units: baseline 181 days, settled 366 days, one person.

## Stage 4 (reframed) — Encoding (done)

- **Sketches:** A (distributions, before/after — outdated, "before" was
  mostly the dip), B (time, whole export), **C (distributions, three phases)**
  — `sketch-C-move-abroad-three-phases.png` (scratch).
- **Student's reading of C:** settled-phase busier days (≈4–6 messages)
  return close to the baseline, but there are still more zero days than at
  baseline (~61% vs ~50%; dip ~72%) → a **partial recovery**. Corrected: the
  chart shows messages per day, not conversation length (student's
  assumption: more messages ≈ longer conversations — untested).
- **Chosen:** C as the main chart.
- **Single comparison:** baseline vs. settled in strong colours; dip lighter
  ("what happened in between"); control as a smaller reference.
- **Parameter to check:** the baseline window — also run with 2021 + H1 2022.
- Student wants to try an **area-style chart** instead of grouped bars.

## Analysis 12 — parked (2026-10-04), before Critique/Verification

**Status:** Stages 1–4 done (after one reframe). Stage 5 (critique) and
Stage 6 (verification: 25% margin, placebo / half-year swings, longer
baseline, active-day check, job-starter reference) **not run**. No notebook
or final chart yet — only scratch sketches.

**Conclusions so far (descriptive, untested):**
- The original question ("moving abroad → more texting") fell apart on
  context: Stingray was already abroad (internship) before the move, so the
  "jump" after 07/2023 is mostly **recovery from a study-period dip**.
- Reframed story: quiet during internship/thesis — even while back in NL —
  then back up once settled abroad. **By eye the recovery is partial:**
  busier days return to baseline, zero days don't (≈50% → 72% → 61%).
- Whether that counts as "back at the old level" (within 25%, set before
  looking) is **not yet tested**.
- Limits already known: n = 1, one person's life phase; data-suggested
  origin; season mismatch (Jan–Jun baseline vs full settled year); control is
  4 people; the transition bundles thesis end, move, partner and job.

**Student's doubt:** whether an n = 1 life-phase story makes a strong
distribution chart for the assignment — the shift is visible but partial,
and the story leans on context only the group knows.

**Next steps if resumed:**
1. Stage 5 critique on a proper build of C (baseline vs settled emphasised;
   dip faded; control as ticks). Try an **area / filled-step version** —
   note that a smooth area suggests values *between* counts (2.5 messages)
   that can't exist; a filled step keeps the counts discrete.
2. Stage 6: 25% margin test on mean messages/day relative to control;
   Stingray's ordinary half-year-to-half-year swings as the yardstick;
   longer baseline (2021 + H1 2022); active-day share as check; other job
   starters as reference.
3. Write-up only if Stage 6 supports a clear story.

**Parked candidates for the next cycle (student's ideas + suggestions):**
- **Response time around the friends' weekend** (student) — see chat
  discussion; repeats yearly, so n = number of weekends, not 1.
- **Response time, city vs non-city movers** (student).
- Hour-of-day on NL match days vs ordinary days (suggested).
- Message length (`n_words`) around a fixed event (suggested).

**Update (2026-10-05):** stays parked. Student judged the n = 1, partial-
recovery story not strong enough for the assignment; moved on to the parked
candidate "response time around the friends' weekend" (Analysis 13).

---

# Analysis 13 — do we reply faster around the friends' weekend?

New goad cycle (2026-10-05), started from the parked candidate "response time
around the friends' weekend". Chosen over "response time, city vs non-city
movers". Same 9-person chat; event dates from Analysis 7 (2024-04-19..21,
2025-09-19..21).

## Stage 1 — Question (done)

- **Suspicion (student):** at the friends' weekend, or just before, we have a
  reason to reply faster ("where are you?", "do you want a drink?", planning,
  sharing photos). Being physically close means quicker replies.
- **Measure, sharpened in discussion:** the student first proposed the gap
  between consecutive messages, then time per 5–10 messages. Both restate
  message *volume*, which Analysis 7 already showed rises around the weekend.
  **Chosen: reply gap** = minutes from a message to the next message by a
  *different* author. Gaps over **6 hours** = new conversation, not a slow
  reply (student's threshold; to be checked for sensitivity later).
- **Boring result (student):** people reply faster in the evening and off
  work. Because the weekend is Fri–Sun, the **comparison is other Fridays–
  Sundays only**, so the weekend-vs-workday effect is not what gets measured.
- **Proposition:** the median reply gap during the weekend and the 3 days
  around it is **at least 20% lower** than on comparable days (other Fri–Sun)
  outside that range. **No** if the decrease is under 20%.
- **Scope:** only the two friends' weekends. Birthdays and the Groningen day
  were considered and left out of this hypothesis.
- **Origin:** lived experience. The reply-gap measure has not been looked at;
  Analysis 7 looked at volume over the same weekends.
- **Punchline (student):** reply time drops sharply around group events,
  because there is something to coordinate.
- **Student's own worry:** few messages during the weekend itself means a
  small sample and high variance (6 event days in total).
- **Open for Stage 2:** the "3 days around" window includes Mon–Thu days
  while the comparison is Fri–Sun only; whether birthday/Groningen days are
  also removed from the comparison days.

## Stage 2 — Data (done)

- **One row = one message** (timestamp, author, text); photos and stickers
  count as messages.
- **Features (agreed):**
  | Feature | Definition |
  |---|---|
  | `reply_gap_min` | minutes from the *last* message of the previous author's run to the first message by a different author; empty when the same author continues |
  | `new_conversation` | gap > 6 h → not a reply, dropped |
  | `weekend` | Fri–Sun block, named after its Friday; a reply belongs to the day it was sent |
  | `event` | `2024` / `2025` / `none` |
- **Window — student chose option A:** only the friends' weekend itself
  (Fri–Sun) vs other Fri–Sun blocks. The "3 days around" part of the Stage 1
  proposition is dropped (Mon–Thu days would face the weekday/weekend
  comparison again). **Proposition now:** the event weekend's median reply gap
  is ≥ 20% lower than on ordinary Fri–Sun weekends.
- **Claim unit (agreed):** one value per weekend = its median reply gap.
  Ordinary weekends form the reference distribution; the 2 event weekends are
  placed in it. Row unit (gap) ≠ claim unit (weekend); n = 2 events. Whether
  this is also the chart is a Stage 4 decision.
- **Missingness:** quiet weekends have few reply gaps under 6 h. Student set a
  **minimum of 2 reply gaps** per weekend (low → sensitivity check in Stage 6).
- **Still open:** whether birthday and Groningen days are removed from the
  ordinary weekends.
- **Build:** `notebooks/analysis/09-friends-weekend-reply-gap.ipynb` (reply-gap
  step + Stage 3 shape plots, event weekends not shown).
- **Transparency:** the prototype script printed both event weekends' medians;
  the assistant has seen them, the student chooses when to see them.

## Stage 3 — Shape (done)

- **Reply gaps (ordinary weekends, 6,328):** long-tailed; half within
  **4 minutes**, 23% at 0 minutes, mean 28.5 → **median per weekend** stays.
- **Timestamps are minute-only** (raw export `HH:MM`, no seconds). A % threshold
  is unrealistic at 1–2 minute medians (10% would only be easier to pass, not
  more realistic). **Test changed to a rank (student):** *both* friends'
  weekends have a lower median reply gap than **80% of the comparable
  weekends**.
- **Comparable weekends (student): ordinary Fri–Sun with ≥ 50 reply gaps**
  (≈ 21). Reason: busy weekends have more rapid back-and-forth, and the
  friends' weekends were the 1st and 5th busiest of 289 — "faster than equally
  busy weekends" is the stricter test of "faster because we're together".
- **Birthday and Groningen days** removed from the comparison weekends.
- **Units:** 2 event weekends vs ≈ 21 comparable → the claim is
  **descriptive** (student agrees n = 2 is too small for proof).
- **For Stage 6:** remaining volume difference (events ≈ 150 / 110 gaps vs
  comparison ≥ 50); the 6 h and ≥ 50 cut-offs.
- **Chart preference (student):** show *all* reply gaps, not two medians —
  a **cumulative distribution (ECDF)**, friends' weekends vs comparable
  weekends. The rank of the medians stays the test.
- **Open:** tie rule for equal (whole-minute) medians.

## Stage 4 — Encoding (done)

- **Settled before the sketches:** birthdays (all 9, day/month against
  aliases) — a birthday on Fri/Sat/Sun removes that whole weekend; Groningen
  weekend 2023-06-23..25 removed; **a tie counts against the claim** (assumed
  from the student's "good point", to confirm). → **20 comparable weekends**
  (1 dropped for a birthday).
- **Rank test (pre-agreed: both weekends faster than 80%):**
  | Weekend | Median gap | Reply gaps | Comparable weekends beaten |
  |---|---|---|---|
  | 2024 | 1 min | 153 | 50% |
  | 2025 | 3 min | 109 | 25% |
  → **fails for both** — the proposition is not supported on its own test.
- **Sketches** (scratch, `notebooks/analysis/`):
  - A `sketch-A-reply-gap-ecdf.png` — ECDF of all reply gaps (student's
    preference). 2024 ≈ the comparable weekends; 2025 *slower*.
  - B `sketch-B-reply-gap-rank.png` — one dot per weekend; ties overlap, so
    20 weekends look like ~9 dots (fix if chosen).
  - C `sketch-C-reply-gap-over-time.png` — median per weekend over time; the
    busy (≥ 50) weekends are almost all fast, all years.
- **Student's reading:** in A the friends' weekends don't stand out (2024 on
  top of the comparable weekends, 2025 slower); in C many busy weekends have a
  low median. **"The null is true"** — story: fast on the friends' weekend,
  but no faster than any busy weekend.
- **Chart (student):** the ECDF doesn't meet "likelihood of each option" →
  **probability mass function** (whole minutes, so not a smooth density).
  Long tail → **option C**: one bar per minute 0–10, then 11–20, 21–60,
  61–360, height = share of replies **per minute** (corrected for bin width).
  Comparison: each friends' weekend vs the comparable weekends.
- **Parameter:** the ≥ 50 busy line. Student's first reason (raise it so the
  friends' weekend becomes an outlier) named as parameter-shopping — and likely
  backwards, since C shows the busiest weekends are the fastest. Reframed as a
  fairness check (events had 153 / 109 gaps). **≥ 50 stays primary;** Stage 6
  checks ≥ 30 and ≥ 75.
- **Build choice (assistant, for critique):** comparable weekends shown as the
  average of the per-weekend distributions (each weekend one vote, as in
  notebook 08), not pooled gaps (busy weekends would count more).

## Stage 5 — Critique (done)

- **Draft:** `notebooks/analysis/draft-reply-gap-pmf.png` — two panels
  (2024, 2025), grey = comparable weekends, colour = friends' weekend.
- **First glance (student):** the friends' weekends "lose" at 0 minutes, 2025
  more than 2024; 2024 has more replies at 1 minute. Student read both as
  slower overall — **corrected:** 2024 is mixed (≈ equal share within 1
  minute; its median beat 50% of comparables); only 2025 is clearly slower.
- **Grouping:** two panels invite 2024 vs 2025, which is not the point →
  **one graph**, both friends' weekends (kept separate) against the rest.
- **Colour / the null:** the message ("no difference") isn't visible in the
  data shown → show the **spread of the comparable weekends**, so a reader can
  see whether the friends' weekends fall inside ordinary variation.
- **Delete:** duplicate legend and duplicate panel.
- **Claim:** student said "a bit slower"; the pre-registered claim is **"not
  faster"**. "Slower" is data-suggested and only holds for 2025.
  **Falsifying region:** friends' bars clearly above the comparable range at
  0–1 minutes.
- **Revision v2:** `notebooks/analysis/draft-reply-gap-pmf-v2.png` — one
  panel; grey bars = comparable average; grey whisker = middle 80% of the
  comparable weekends (10th–90th percentile, matching the 80% test); 2024 and
  2025 as dots; one legend; divider before the wider bins. Title is a
  placeholder; wording is the student's.

## Stage 6 — Verification (done)

- **Busy-line sensitivity (agreed in Stage 4; ≥ 50 primary):**
  | Busy line | Comparable weekends | 2024 beats | 2025 beats |
  |---|---|---|---|
  | ≥ 30 | 67 | 69% | 43% |
  | ≥ 50 | 20 | 50% | 25% |
  | ≥ 75 | 10 | 30% | 10% |
  Never reaches 80% → the "not faster" result holds at every line. The
  higher the line, the *less* special the friends' weekends look (busier =
  faster), as expected in Stage 4.
- **Null (student):** reply time on the friends' weekend is the same as on
  other busy weekends. Each comparable weekend works as a placebo (as the
  placebo months in Analysis 11): under the null, beating ≥ 80% happens ~20%
  of the time per weekend, ~4% for both.
- **Comparisons:** ~5 measures, 4 sketches, 3 busy lines, 14 bins × 2 weekends
  on the final chart. The **2024 1-minute dot** above the 80% band is a side
  note, not a finding: with 28 dots, ~5–6 outside the band are expected by
  chance (student: "many weekends are just as fast").
- **Third variable — volume:** the friends' weekends were still busier than
  the comparison weekends; busier = faster, so this works *for* the claim and
  still doesn't produce it — it can't explain the null away.
- **Limitations:** n = 2 weekends; 2024 accident (worried messages) not
  checked; no held-out slice (the planning weeks before would need a Mon–Thu
  comparison); minute-only timestamps.

### Verdict on the Analysis 13 proposition

**Not supported.** Both friends' weekends fail the rank test set before
looking (2024 beats 50%, 2025 beats 25% of 20 equally busy weekends; 80%
needed), and this holds at every busy line. Student's verdict: the friends'
weekend brings more messages (Analysis 7), but **no faster replies than other
busy weekends**. With n = 2, "not detectably faster" is the defensible
wording, not "significantly". Chart title confirmed by the student.

### Write-up

- `notebooks/analysis/findings-friends-weekend-reply-gap.md` (draft
  suggested by the assistant, for the student to edit).
- Final chart: `notebooks/analysis/friends-weekend-reply-gap-final.png`
  (the v2 design; title kept as proposed).

---

# Analysis 14 — do city people start their day later in the chat?

New goad cycle (2026-10-05, evening), from the parked candidate "city vs
non-city". Student will ask the teacher tomorrow whether a well-tested null
counts for requirement 3.

## Stage 1 — Question (done)

- **Design (student chose option a):** a **group comparison**, city vs
  non-city people — not the move as an event. Flagged: a group comparison is
  not a shift at a moment in time, so on its own it doesn't meet requirement 3.
- **Suspicion (student, lived experience):** city friends live a different
  life, with more weekday evening activities; their messages come later in
  the day. Two mechanisms separated — (3) later start of work → later first
  message (morning shifts) vs (5) more evening activities → later replies.
  **Chosen: mechanism 3.**
- **Measure:** time of a person's **first message of the day**, **weekdays
  only** (Mon–Fri), only on days that person sent **≥ 5 messages** (a quiet
  day makes anyone's first message look late).
- **Groups:** city = `striking-rail` (the student), `animated-elk`,
  `effervescent-penguin`, `humorous-stingray` (abroad, counts as city) + the
  person already city-based before the export; non-city = the other 4.
- **Boring result (student):** the daily distribution only shows that people
  sleep at night and work by day.
- **Proposition:** at least **4 of the 5** city people have a later typical
  first-message time than **all 4** non-city people. Otherwise: no.
- **Origin:** lived experience; the student is one of the city people, so the
  impression includes their own behaviour (kept in, student's choice).
- **Punchline (student):** city friends are online at different times than
  their non-city friends.
- **Not pursued:** election message length — already tested in Analysis 3
  (not supported for the group; only `pliable-tiger`, likely volume).

## Stage 2 — Data (done)

- **One row = one message; claim unit = person** (5 city vs 4 non-city), one
  number per person. Days are repeated measurements of the same person.
- **Features (agreed):**
  | Feature | Definition |
  |---|---|
  | `day` | calendar day starting at **06:00** (student) — a message before 06:00 belongs to the previous day, so a late night doesn't count as an early start |
  | `first_message_time` | per person per weekday (Mon–Fri): time of their first message after 06:00 |
  | `messages_that_day` | that person's own count; only days with **≥ 5** count |
  | `typical_start` | per person: median `first_message_time` |
  | `is_city` | city / non-city |
- **Groups (student; move dates corrected vs Analysis 9's log):**
  city = `striking-rail` (Sep 2021), `pliable-tiger` (city before the
  export; moved in with partner 2026, stayed), `animated-elk` (Sep 2022),
  `humorous-stingray` (Jul 2023, abroad), `effervescent-penguin` (**Jul
  2023**; moved to another city in 2025, stays city). Non-city =
  `vibrant-barracuda`, `rib-tickling-curlew`, `fluffy-beaver`,
  `hypnotic-rabbit`.
  **Correction to Analysis 9's log:** Rail Sep 2021 (not Aug), Stingray
  Jul 2023 (not Sep).
- **Period:** Aug 2023 – Sep 2026 (everyone in their final group). Student
  first chose option B (leave Penguin out), which was written for a wrong 2025
  Penguin date; after the correction: all 5 vs 4, original test kept.
- **Missingness:** a person needs **≥ 20 qualifying weekdays** to count.

## Stage 3 — Shape (done)

- **Counts only were looked at** (no city/non-city split). Qualifying
  weekdays (≥ 5 messages) after Aug 2023 are rare (312 person-days).
  `hypnotic-rabbit` (6) and `striking-rail` (19) fall under 20 days; student
  kept ≥ 5 messages and 20 days.
- **Two humps** in the pooled first-message time: morning (08–12) and
  evening (18–20). Student's reading: an evening first message comes from a
  day without the phone, or a conversation that only starts in the evening —
  two kinds of days. The overall median (12:30) falls in the dip.
- **Student: keep only morning days** (first message before 18:00); per
  person the median over those.
- **Validity (limitation):** this measures first appearance in the chat, not
  waking up or starting work. Student holds it's a fair proxy, since many
  messages are sent before lunch.
- With morning days only, `animated-elk` also drops (15 days) → **3 vs 3**.
  Options shown with their chance of "yes" by luck (random order): 3v3 all
  later 5%; 5v3 (15 days) "4 of 5" 7%; 5v3 "all 5" 2%.
- **Final test (student, option a):** all 3 city people (`pliable-tiger`,
  `humorous-stingray`, `effervescent-penguin`) have a later median morning
  first-message time than all 3 non-city people (`vibrant-barracuda`,
  `fluffy-beaver`, `rib-tickling-curlew`). 5% by chance. The student is no
  longer in the test.

## Stage 4 — Encoding (done)

- **Test result (pre-agreed: all 3 city later than all 3 non-city): fails.**
  Median morning first message: `humorous-stingray` 10:09 (city),
  `effervescent-penguin` 10:21 (city), `vibrant-barracuda` 10:55,
  `fluffy-beaver` 11:31, `pliable-tiger` 12:07 (city),
  `rib-tickling-curlew` 12:15. Two of three city people are the *earliest*.
- **Sketches** (scratch, `notebooks/analysis/`):
  - A `sketch-A-city-first-message-hours.png` — share of morning days per
    hour, per group (each person one vote). City higher at 08–09.
  - B `sketch-B-city-first-message-people.png` — one dot per person (the
    test).
  - C `sketch-C-city-first-message-vs-volume.png` — start time vs messages
    per active weekday; no clear pattern with 6 dots.
- **Student's reading:** A goes against the expectation — city people have
  more first messages at 08–09. New explanation (student): non-city people
  work early and have no time to text; city people text before work. B: also
  against; `pliable-tiger` is late, but is known to always go to bed and get
  up late (a personal trait, not the city).
- **Student wants to treat "city people start earlier" as a finding —
  pushed back:** it is data-suggested, and the same mechanism ("city people
  start work later") now explains *both* directions, so it can't lose. At
  most a lead for data that didn't suggest it.
- **Chart: A** (note: A pools 3 people per group; the test is per person).
- **Naming fix:** "morning days" was a misnomer — the 18:00 cutoff sits in
  the dip between the two humps, so it splits **daytime** from **evening**
  days. A stricter cutoff (e.g. 12:00) only as a Stage 6 sensitivity check,
  since the results are now seen.
- **Parameter to check (student):** the 20-day minimum (brings back
  `striking-rail` and `animated-elk`, 15 days each).

## Stage 5 — Critique (done)

- **First glance (student):** city more active earlier — 08 and 09 stand out
  for city; 12, 15, 16 also higher for city.
- **Grouping:** only 3 people per group, so hour-to-hour jumps may be noise.
- **Colour:** purple too pale → stronger colour. **Delete/shorten:** x-axis
  label; "morning days" → "daytime days".
- **Claim (student):** city people have more time to text before work.
  **Pushed back again:** that's a mechanism the chart can't show, and "city
  earlier" is data-suggested. For the *pre-agreed* claim the falsifying
  region was city bars higher at 12–17 (city later) — that didn't happen →
  defensible claim: **city people do not start later**.
- **Agreed held-out check for "city earlier":** the movers *before* their
  move, when they were non-city.
- **Revision:** `notebooks/analysis/city-first-message-v2.png` — Sketch A with
  orange for city, shorter x label; title is a placeholder.

## Stage 6 — Verification (done)

- **Held-out check, rule fixed before running:** if the movers were already
  earlier than the non-city people before moving → a trait, not the city.
  | Mover | Before move | After (Aug 2023+) | Non-city, same months before |
  |---|---|---|---|
  | `humorous-stingray` | 11:06 | 10:09 | Beaver 11:06, Curlew 11:48, Barracuda 13:51 |
  | `effervescent-penguin` | 11:40 | 10:21 | same |
  | `animated-elk` | 10:46 | 10:07 (15 days) | Beaver 10:36, Curlew 12:13 |
  Not earlier than everyone before; ~1 h earlier after → by the rule the
  lead holds up on data that didn't suggest it.
- **Caveats:** non-city people also shift between periods (`vibrant-barracuda`
  13:51 → 10:55, 26 days, possibly their 2022 job). The move is a **package**
  with new jobs (Stingray new job abroad Sep 2023, Penguin job Sep 2024) →
  city can't be separated from job / life stage.
- **Sensitivity:** 15 days → 4 of 5 city people earliest, `pliable-tiger`
  late; "city later" fails everywhere. 12:00 cutoff leaves 3 people —
  uninformative.
- **Comparisons:** ~10 decisions this cycle (student first estimated 3–4).

### Verdict on the Analysis 14 proposition

**Not supported — reversed.** City people do not start their day later in
the chat; if anything earlier. The "earlier" lead is data-suggested but
held up on the movers' before-move data (~1 h earlier after moving). Student
explains it from experience (city people can start work later, so have time
before work) — **that reason is untested** (no work-start data) and the
shift is confounded with new jobs.

### Correction after a first look at the move itself (scratch, 2026-10-05)

`notebooks/analysis/move-city-first-look.png` (throwaway script; `striking-rail`
left out to keep their data unseen). Before vs **the full after-move period**,
with the non-city friends over the same months:
| Mover | Before → after | Non-city, same split | Net vs non-city |
|---|---|---|---|
| `humorous-stingray` | 11:06 → 10:09 | 11:48 → 11:12 | ≈ −20 min |
| `effervescent-penguin` | 11:40 → 10:27 | 11:48 → 11:12 | ≈ −37 min |
| `animated-elk` | 10:46 → **11:16** | 11:28 → 11:37 | ≈ **+21 min** (later) |
The Stage 6 "all three ≈ 1 h earlier" leaned on Aug 2023+ only; Elk's 10:07
was 15 days. Over Elk's full after-period they are *later*. And the non-city
friends also moved ≈ 35 min earlier after Jul 2023. **The lead weakens to:
2 of 3 movers ≈ 20–40 min earlier than the general drift; 1 later.**

---

# Analysis 15 — is animated-elk the fastest to reply? (reply-time distribution)

Started 2026-10-06 as a curiosity, after reviewing which distribution families
Analyses 1–14 assumed (Poisson in 4, negative binomial in 9; everything else
shuffle/placebo/ECDF without a family). Notebook:
`notebooks/analysis/11-reply-speed-elk.ipynb` (runs top to bottom).

## Data type and shape (student decisions)

- **Reply** = a message directly after someone else's message (same as Analysis 13).
- **Continuous, recorded in whole minutes** (the export has no seconds). Student
  first chose +0.5 min, then **spreading** (a 0-minute gap is not zero time).
  **Correction (assistant):** both timestamps are rounded down, so a recorded
  gap of k minutes is really in (k−1, k+1) — spread over that range, not over
  [k, k+1). The uniform version created a fake spike at 1–2 min that no family
  could fit; the earlier "1–3 min excess" was mostly that artefact.
- **Two processes:** replies within a conversation vs new conversations (second
  bump around 10³ min). **Student: cut at the dip**, not the 6 h from Analysis 13.
  Dip = **401 min** with the corrected spreading (392 with the uniform one;
  6 h holds ~the same share). Per author the dip is unstable (252–716 min, 3
  authors without one) → one pooled cut.
- **Not independent:** consecutive log gaps correlate 0.38; nights slower
  (median 12 vs 5 min) → all uncertainty from resampling **days**, never gaps.

## Family

- **Predicted before fitting:** not exponential (median 3 vs mean 26 min; an
  exponential has median ≈ 0.69 × mean).
- With the start fixed at 0 (a free start degenerates on the tied values —
  positive log-likelihoods, nonsense winners beta/gamma): **lognormal wins**
  (KS 0.055), Weibull 0.090, gamma 0.129, **exponential 0.395** (prediction held).
- Per author σ ≈ 2 for everyone → authors differ in *location*, not spread.
  animated-elk fits worst; the misfit is an excess of very fast replies, the
  same shape for everyone but strongest for elk. A **mix of two lognormals**
  (fast + slow) fits better for 7 of 9: elk 67% fast (median 0.5 min) / 33% slow
  (24 min); rest 56% / 44% (0.8 / 30 min). The slow process is about the same —
  the difference is *how often* someone replies in fast mode.

## Per author (corrected spreading, 1,000 day resamples)

| author | typical (geo. mean) | 95% CI | within 6 min |
|---|---|---|---|
| animated-elk | **2.0 min** | 1.7–2.3 | **70%** (66–73%) |
| humorous-stingray | 3.4 | 2.9–4.0 | 58% |
| fluffy-beaver | 3.5 | 3.0–4.1 | 58% |
| others | 3.7–4.9 | | 51–57% |

- elk is **fastest of all 9 in 1,000/1,000 resamples** on both measures.
- **Threshold sweep** (1, 2, 3, 6, 10, 30, 60 min): elk rank 1 at every
  threshold; lead ~12 points at 1–6 min, shrinking to 2–4 at 30–60 min.

## Tests (fixed before looking)

1. **Quiet-period test** (student: 60 min silence; ≥ 1.5× general share) — split:
   - first to reply after silence: 18.9% vs general 14.8% → **1.27×** (CI 1.16–1.39)
     → **not met**;
   - speed advantage after silence **3.3×** (2.5 vs 8.2 min) vs 2.3× within
     conversations → **held up**, the "only fast in lively conversations"
     explanation predicted the opposite.
   - Student verdict: one half held up (speed), the other not (being first more
     often). Not moved after the fact.
2. **Shuffle test** (teacher's feedback; student: share within 6 min, shuffle
   author labels within each day, 2,000×, seed 42, threshold **10%**):
   shuffled mean 62%, highest 65%; real **70% never reached (p < 0.001)**.
   Within conversations: same. Shuffling everything: 58% → the step to 62% is
   **context** (elk is active on faster days); ~8 points remain that are elk's own.

## Chart (student choices)

- Form: densities of both groups on one log axis (V1-2), not the decomposed
  panels; the model bar moved to a separate supporting chart.
- Title: *"The group's fastest thumbs belong to the only one with the 🔔 on"* —
  two true facts side by side, no causal claim. Subtitle carries the numbers.
  Earlier "makes sure he responds quickest" dropped: causal, not tested.
- Direct labels instead of a legend; footnote as bullets incl. the shuffle test
  and "notifications always on: known from the group, not measured".
- Files: `reply-speed-elk-vs-rest-draft.png` (main),
  `reply-speed-model-share.png` (supporting, not for the reader).

## Leads (data-suggested, not tested)

- **The notification bell** explains the pattern from lived experience
  (elk is the only one with notifications always on). The data shows faster,
  not why. A test would need a bell-off period or another person turning it on.

## Stage 5 — Critique (done)

- **First impression (student):** orange is higher at the start — the message.
- **Claim (student):** animated-elk is faster in responding. Sharpened to the
  title's ranking + scope: faster than *each* of the other 8, within
  conversations, 2020–2026.
- **Falsification (student):** one of the others, or the others as a whole, just
  as fast. "As a whole" = curves overlapping left of the 6-min line; "one of the
  others" was invisible on the pooled chart → range of the 8 added.
- **Not shown (student):** replies after 401 min — defensible, a new
  conversation can start. Also not on the chart: the failed half of the
  quiet-period test (claim is speed only) → belongs in the write-up.
- **Changes applied:** data lines lighter (student keeps them — a model never
  fits every bin; the ~1 min peak is the rounding grid); "each of the other 8:
  51–58%" under the annotation; footnote bullet "fastest of all 9 in 1,000/1,000
  day resamples and at every threshold 1–60 min"; subtitle removed (numbers were
  duplicated in the annotation, and it coloured the rest orange).

## Open

- **Revisit earlier distribution charts** with the family in mind (plan agreed
  2026-10-06, order to be chosen): 13 (geometric mean per weekend removes the
  tie rule; corrected spreading), 4 (Poisson SEs too narrow → negative binomial
  or week bootstrap), 3 (message length on a log scale), 12 (overlay
  Poisson/negative binomial on the daily counts), 8 (per-day PMF), 9 (CIs from
  the fitted negative binomial).
