# Feedback on the friends'-weekend time-series chart

Feedback received on `friends-weekend-timeseries.png` (week 2 time-series
assignment, built from `05-friends-weekend-activity.ipynb`, Analysis 7 in
`analysis-log.md`): daily message volume aligned on the weekend, 2024 and
2025 overlaid. Logged here to pick back up later, not acted on yet.

## 1. The story is a good find

Feedback: the friends' weekend is a strong story to tell from this chat.

## 2. The data and visualisation back up the story

Feedback: the chart supports the claim it makes. No redesign requested.

## 3. Improve the shuffle test so it guards against spurious correlation

Feedback: the current shuffle test reads as "does any other stretch of the
chat reach the same average messages per day as the weekend?" That is a test
of the absolute **level**. It does not test the **increase**. Suggested
instead:

- Measure the weekend's **increase** in messages per day relative to its own
  local baseline: the **4–5 weeks before and the 4–5 weeks after** the
  weekend.
- Do the same for randomly placed fake "weekends" across the chat: compute
  each one's increase over its own 4–5-week before/after baseline.
- Repeat roughly **10,000 times** and count how often a fake weekend shows
  a **similar or higher** increase than the real one. That share is the
  evidence for how much of the shift the friends' weekend explains.

**What the current test actually does (from Stage 6, for comparison):**
it excludes every day inside either weekend's 30-before/during/30-after
window, takes a 3-day sliding-window median over the remaining 2091 baseline
days, and checks how many reach the real during-median of 80/day (result
0/2091). The after-window version found 68/2087 (~3%) reaching 20.5/day.
Both compare **levels**. Neither subtracts a local baseline. So a busy
period of the chat could match the weekend without containing any real
"jump", and a quiet period with a big jump could be missed.

**To think about when revisiting (open, not decided):**
- **Increase as a difference or a ratio?** "During minus local baseline"
  and "during divided by local baseline" behave differently when the
  baseline is near zero (ratios explode on quiet stretches). Decide which
  matches the claim you want to defend.
- **Mean or median** for the during-window and the baseline? Stage 6 used
  medians because of the spiky daily counts. Keep it consistent or justify
  the switch.
- **4 or 5 weeks?** The current sensitivity check already uses 30-day
  before/after windows, which is close to 4 weeks. Pick one up front
  rather than after seeing results (the multiple-comparisons point from
  Stage 6).
- **Should the baseline exclude the real weekends?** If a fake weekend's
  before/after window overlaps a real weekend, the real spike inflates
  its baseline and makes it look less impressive. Decide whether to
  exclude those placements or the real weekend days.
- **10,000 draws vs. ~2,000 possible positions.** There are only about
  2,000 distinct start days in the chat. 10,000 random draws will
  therefore reuse positions (sampling with replacement), which mostly
  stabilises the estimate rather than adding new information. Think about
  whether that's what you want, or whether an exhaustive pass over every
  possible position is the cleaner version of the same idea.
- **Two real weekends, one number?** Test each weekend's increase
  separately (2024's during-median was 91 and 2025's was 53, which are
  quite different), or pool them? If pooled, a fake should probably also
  be a pair of windows.
- **Reporting:** what share of fake weekends counts as "rare" for you, and
  how will you phrase it on the chart footnote and in
  `findings-friends-weekend.md` (which currently quote the 0/2091 and
  68/2087 level-based results)?
