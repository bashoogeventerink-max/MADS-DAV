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
