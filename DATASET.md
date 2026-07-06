# Dataset Understanding — GH Archive

Notes from inspecting a real sample (`2024-01-15-15.json.gz`, one hour).

## What it is
GH Archive (https://www.gharchive.org/) records the **public GitHub event
timeline**. Files are **hourly**, gzip-compressed, **newline-delimited JSON**
(one JSON object per line = one event).

- URL pattern: `https://data.gharchive.org/YYYY-MM-DD-H.json.gz`  (H = 0..23)
- Master range download: loop over dates/hours (see `1_profile/profiling.txt`).

## Schema (top-level keys of every event)
```
id, type, actor, repo, payload, public, created_at
```
- `type` — event kind (PushEvent, PullRequestEvent, …)
- `actor.login` — the GitHub user
- `repo.id`, `repo.name` — the repository (`owner/name`)
- `payload` — event-specific detail (e.g. PushEvent has `.size` = #commits)
- `created_at` — ISO timestamp (`2024-01-15T15:00:00Z`)

## Measured volume (one hour, 2024-01-15 15:00 UTC)
| Metric | Value |
|--------|-------|
| Compressed size | 130 MB |
| Uncompressed size | ~1.0 GB |
| Events (rows) | 266,871 |

Extrapolated:
| Span | Compressed | Uncompressed | Events |
|------|-----------|--------------|--------|
| 1 hour | 130 MB | 1.0 GB | 0.27 M |
| 1 day | ~3.1 GB | ~24 GB | ~6.4 M |
| **~8 days** | **~25 GB** | **~190 GB** | **~51 M** ✅ meets floor |

**Row count is the binding constraint** (need > 50 M). ~8 full days clears both
the 5 GB and 50 M-row requirements. (Older years have fewer events/hour, so scale
the day-count up for 2021.)

## Event-type distribution (this hour)
```
169030 PushEvent          7217 PullRequestReviewEvent
 28616 CreateEvent        4311 PullRequestReviewCommentEvent
 18272 PullRequestEvent   4170 IssuesEvent
 11687 IssueCommentEvent  2373 ForkEvent
  9548 WatchEvent         1247 ReleaseEvent
  7556 DeleteEvent        + PublicEvent, CommitCommentEvent, MemberEvent, GollumEvent
```
PushEvent ≈ 63% of all events. WatchEvent = "starred". ForkEvent = fork.

## Sampling plan for our question (2021 → 2026)
Our question compares AI/ML vs general repo activity **over time**, so we don't
need 8 consecutive days — we need coverage *across* the window. Plan:
- Pick a fixed sample per month (e.g. the 1st–2nd of each month, all 24 hours)
  across 2021–2026, OR a full representative week per quarter.
- This still blows past the size floor while giving a clean monthly time series.
- Raw files go to a **GCS bucket** for Phase 3; only small aggregated CSVs come
  back for Phase 4 charts.

## How we'll classify AI/ML vs general
By repo name pattern (case-insensitive): `llm`, `gpt`, `pytorch`, `tensorflow`,
`langchain`, `diffus`, `transformer`, `huggingface`, `openai`, `neural`,
`deep-learning`, `machine-learning`, `agentic`, `rag-`, … (tune in Phase 1).
Everything else = "general". This is a heuristic; we document its limits in the
writeup.

## Initial data-engineering findings (one-hour sample)
Measured on `2024-01-15-15.json.gz` with jq/awk (see `1_profile/`).

**Cardinality (in a single hour):**
- distinct repos: **81,373**
- distinct actors: **66,135**
- 266,871 events total → most repos/actors appear only once or twice per hour.

**Bots dominate raw activity — a real data-quality issue.** Top actors:
```
19295 github-actions[bot]     9936 dependabot[bot]
17689 inse2233tto (spam)      3137 renovate[bot]
17660 ion561sdag  (spam)      3324 LMAO-armv8
```
The busiest "repos" are automated spam (`.../Projcts9`, 2000+ pushes/hour). So
**raw event counts overstate human activity** — we should either flag `[bot]`
actors / filter obvious spam, or report both raw and bot-excluded numbers. Note
this caveat in the poster.

**PushEvent detail:** avg **4.02 commits per push** (169,030 pushes, 679,685
commits this hour).

**AI/ML repos already look different (supports our question).** Event mix for
AI-repo events vs the overall stream:

| Event | AI-repo share | Overall share |
|-------|--------------:|--------------:|
| WatchEvent (star) | **16.4%** (363/2208) | 3.6% (9548/266871) |

AI/ML repos are **~4.6× more likely to be *starred*** than the average repo —
visible in a single hour. Stars (attention/adoption) may be a stronger AI-vs-
general signal than push volume. Worth a dedicated chart in Phase 4.
