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

## Stage 5 — Critique (in progress)
