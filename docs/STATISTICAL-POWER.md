# Statistical power: which of this tool's numbers can ever be proven

Snapshot 2026-08-23, one machine, 353 sessions and 61,798 API calls over the
window 2026-07-20 to 2026-08-22. Every figure below was computed from
`~/.claude/projects/*/*.jsonl` and the local telemetry ledgers. The standing
caveat at the foot of `docs/CLAIMS.md` applies to all of it: these are one
machine's numbers, offered as an order of magnitude and a method, never as
constants.

This document exists because the project's central promise, that a result is
either VERIFIED or honestly NOT_PROVEN, currently rests on guard conditions
(schema match, window match, a session floor, no parse skips) and not on any
comparison against noise. A verdict that cannot fail to noise is the same
shape of defect the backlog already records twice: a control that cannot
reach a verdict reads as a passing control.

## 1. The formula, stated once

For a two sample comparison at 80% power and alpha 0.05 two sided, the
required sample per arm is

```
n = 15.7 * (sd / delta)^2
```

Rearranged, the minimum detectable effect (MDE) for a cohort you already have
is

```
MDE = cv * sqrt(15.7 / n)          where cv = sd / mean
```

Both are used below. Nothing here needs a statistics package: the constant
15.7 is (1.96 + 0.84)^2 * 2, and `cv` comes out of the same counters
`measure_tokens.py` already reads.

## 2. The default metric is tighter than expected, and the session floor is too low

`DEFAULT_METRIC` is `first_request_median`. Measured across 353 sessions:

| Statistic | Value |
|---|---|
| Mean first request | 93,000 tokens |
| Median | 94,863 |
| Standard deviation | 16,626 |
| Coefficient of variation | **0.179** |

That dispersion is good news: first request is a well behaved metric. What it
exposes is the floor. `MIN_SESSIONS = 3` in `scripts/experiment.py` is the
only sample size guard in the proof engine.

| Sessions per arm | Minimum detectable effect at 80% power |
|---|---|
| **3 (today's floor)** | **40.9%** |
| 10 | 22.4% |
| 13 | 20.0% |
| 30 | 12.9% |
| 50 | 10.0% |
| 100 | 7.1% |
| 201 | 5.0% |

Read the first row plainly. An experiment closed at the floor can print
VERIFIED for a change of any size, including a 2% drift that a re-run would
reverse, because nothing in `build_record` compares the delta to the spread of
the cohorts it came from. The guards catch a broken comparison. They do not
catch a comparison that is fine but too small to see.

This is not a claim that past results are wrong. A 30 day baseline on an
active machine carries hundreds of sessions, which is well past the 5% row.
The defect is that the floor permits a verdict the data cannot support, not
that it has necessarily produced one.

## 3. Dispersion differs by three orders of magnitude across metric families

The same estate, measured at three different units of analysis:

| Unit | Metric | cv | Observations per day | Days per arm to detect 20% |
|---|---|---|---|---|
| Per API call | Context per call | 0.49 | 3,652 | 0.03 |
| Per API call | Latency per call | 1.30 | 3,652 | 0.2 |
| Per API call | Output per call | 1.36 | 3,652 | 0.2 |
| Per session | First request | 0.179 | 15.5 | 0.9 |
| Per day | Output tokens | 0.56 | 1 | 124 |
| Per day | Commits | 0.62 | 1 | 150 |
| Per day | Founder corrections | 0.96 | 1 | 365 |

The gap between the top and bottom of that table is four orders of magnitude
in time to an answer. It is the single most useful thing in this document,
and it generalises past token work: **choose the unit of analysis by its
dispersion and its arrival rate, before choosing the intervention.** A change
whose effect only lands on a daily aggregate cannot be evaluated by any
experiment a single operator can run. It should be decided by argument and
recorded as a decision, not dressed as a measurement.

## 4. Cross day correlations are confounded by volume, and the tool should say so

Two hypotheses were tested on 30 active days and both survived the raw
correlation, then collapsed under a control for how busy the day was.

| Hypothesis | Raw | Controlled | Verdict |
|---|---|---|---|
| Subagent fan out drives rework | r = 0.492, p = 0.004 (20,000 permutations) | partial r = 0.126 controlling for sessions | Refuted |
| Session count drives rework | r = 0.599, p = 0.0009 | r = -0.498, p = 0.026 per session, sign reversed | Refuted |

The mechanism is the same in both: a busy day carries more sessions, more
spawns, more tokens and more corrections, so any two of those correlate
strongly while explaining nothing. This matters directly to the advisor. Any
ranking of levers built from a cross day correlation, present or future,
inherits this confound, and the honest fix is cheap: normalise per session or
per call before correlating, and report the partial alongside the raw.

Caveat kept with the finding, because it cuts the other way: corrections are
logged when a human notices and says something, so a lower correction rate on
a busy day may mean fewer catches rather than fewer mistakes. The correct
reading is that the data does not support the fan out story, not that fan out
is proven safe.

## 5. The tool currently lints the smaller half of the problem

Direct decomposition of 24.04 billion tokens of context read across 61,798
calls in 143 sessions:

| Component | Tokens | Share |
|---|---|---|
| Fixed preamble floor (calls times the per session start context) | 6.27B | **26.1%** |
| Accumulation above the floor, inside sessions | 17.76B | **73.9%** |

`context_lint.py` measures the sources of the first row: CLAUDE.md, the memory
index, what loads at startup. Nothing in the toolkit measures the second row,
which is nearly three times larger. Median session length on this machine is
434 API calls, p90 is 995, and context grows roughly 725 tokens per call
inside a session (median across 108 sessions of 30 calls or more).

The arithmetic of the two levers, computed on the same 61,798 calls:

| Change | Reading saved | Mean context per call |
|---|---|---|
| Preamble from 101,517 to 45,000 | 14.5% | minus 55,104 |
| Cap sessions at 250 calls | 48.1% | minus 182,795 |
| Both | 62.6% | minus 237,899 |

Session length is a lever this project does not currently name, and on these
numbers it is the larger of the two.

## 6. Context has a measurable price in seconds, and the transcripts already carry it

Regressing time from a user message to the first assistant token on context
size, after removing the effect of reply length, over 3,636 turns:

```
0.98 seconds per 100,000 tokens of context      t = 7.68
```

R squared is only 0.016, because how long a reply is dominates how long it
takes. The coefficient itself is a 7.7 sigma result, and it is the coefficient
that matters: it converts every token this tool removes into seconds a person
gets back. Quintile view of the same 3,636 turns:

| Context quintile | Median context | Median seconds to first token |
|---|---|---|
| Q1 | 163,607 | 7.0 |
| Q2 | 256,672 | 7.5 |
| Q3 | 347,717 | 8.0 |
| Q4 | 453,007 | 9.2 |
| Q5 | 662,038 | 9.6 |

`profile.py` already walks message timestamps and computes gaps between
consecutive messages, but buckets all of them as idle time. Splitting the
user to assistant gap (model response) from the assistant to user gap (human
thinking) turns data the tool already parses into the one number a
non technical owner actually feels. Caveat to carry with any such feature: a
transcript gap includes network time and queueing, so it is response latency
as experienced, not model time, and it should be labelled that way.

## 7. Startup rent grows on its own

Median session start context by day, 353 sessions over 32 days, ordinary least
squares:

```
start_context = 78,773 + 748 * day       R^2 = 0.416   t = 4.62
```

That is the strongest relationship in the whole dataset. A deliberate pruning
round on 2026-08-12 is visible as a dip in the series and did not change the
slope. Startup rent is not a level to be set once and forgotten, it is a
quantity with a positive drift, and a tool that measures it should show the
trend and its slope rather than a single current reading.

## What this changes, proposed

Recorded as backlog entries D27 to D31 in `docs/BACKLOG-DEFECTS.md`. In short:

1. Compute the MDE from each cohort's own dispersion, print it beside every
   verdict, and downgrade to NOT_PROVEN when the observed delta is smaller
   than it. This is the load bearing one.
2. Raise the session floor, or make it a function of the effect the user is
   claiming rather than a constant 3.
3. Warn on total metrics (`output_total`, `input_total` and siblings) when the
   two cohorts differ materially in session count, since a total moves with
   volume independently of the treatment.
4. Extend the linter, or add a companion measure, to the within session
   accumulation term that is 73.9% of the cost.
5. Report the trend and slope of startup rent, not only its level.

## Reproducing this

Everything here comes from files Claude Code already writes. The per call
figures come from the `usage` object on each assistant message plus the
message timestamps beside it; the per session and per day figures come from
the same transcripts aggregated. No sampling, no modelling, no external
service. The permutation tests used 20,000 shuffles and the partial
correlations are the standard three variable form.

The reading to take away, if only one line survives: **run the power
calculation before the experiment, not after it.** A tool whose entire
premise is refusing to report savings it cannot prove should also refuse to
report savings it could not have detected.
