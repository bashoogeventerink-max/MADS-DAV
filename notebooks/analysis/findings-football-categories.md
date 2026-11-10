# Findings — football tournaments as the categories graph (Analysis 5)

Report text for `final_img/football-categories-final.png`. Draft of 2026-11-10,
built from the student's own three sections plus feedback; the student checks
it and moves it into the Google Doc. Numbers come from
`18-football-categories.ipynb`.

## What it shows

The graph shows that the group chat becomes more active during football
tournaments. Taking all four tournaments together, the group sent on average
7.2 messages per day in the 4 weeks before and the 4 weeks after a tournament,
while during the tournaments it sent 17.0 messages per day: 2.3× as many. The
orange bar in the middle of each tournament shows this increase.

## Design and text choices

I chose grey and orange so that the tournament weeks stand out against the
weeks around them, which is the goal of this graph. I placed the labels
directly on the bars of the first tournament instead of using a separate
legend, to reduce clutter. Because of similarity, the reader recognises the
orange and grey bars in the other tournaments without repeating the labels.
Before and after share the same grey because together they form the
comparison. Because of proximity, the three bars of each tournament touching
each other read as one tournament, while the repeated orange lets the eye
compare the tournaments with each other. I compare each tournament with the
4 weeks before and after it, for two reasons. First, a tournament takes roughly
4 weeks, so the periods are of similar length. Second, the chat slowly becomes
quieter over the years, so comparing a tournament with its own neighbouring
weeks is fairer than comparing it with the whole chat. I chose messages per
day because it is easier to relate to than messages per week or month.

(To check, student: the gestalt sentences are the assistant's wording.)

## Is it real?

Messages arrive in bursts: one message triggers replies, and a match night
brings dozens. A Poisson count assumes messages arrive independently at a
steady rate. In that case the variance of the daily count equals its mean.
Outside the tournaments, the daily count in this chat varies 27 times more
than that. Poisson error bars would therefore be far too narrow and make the
graph look more certain than it is. I therefore computed the error bars by
resampling whole weeks (2,000 times), so a burst stays inside its week. Each
bar shows the range in which 95% of those resampled averages fall.

I also looked at each tournament separately to see whether the effect holds
for every tournament. The World Cup 2026 was in the data from the start, but I
had left it out of my earlier analyses; it is now included. The effect is
clear for Euro 2020 and Euro 2024. For World Cup 2022 (2.1×, 95% range
0.9–4.6×) and World Cup 2026 (1.5×, 0.8–2.6×), no increase is also possible.
My claim is therefore about the tournaments together, not about each
tournament on its own: together the group sends 2.3× as many messages during a
tournament as around it, with a 95% range of 1.7–3.3×. That range stays above
1, so the increase is real.

What I cannot claim is that football alone causes the extra activity. In three
of the four tournaments, the 4 weeks after are busier than the 4 weeks before.
The World Cup 2022 "after" period also includes Christmas. Something else that
happens around the same time may also play a part.
