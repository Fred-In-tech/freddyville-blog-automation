# Architecture

Deeper notes on how the daily blog agent is put together, and why it is shaped this way.

Interactive map: [architecture map](https://fred-in-tech.github.io/freddyville-blog-automation/map/)

## The design constraint: no infrastructure

The agent has no server, no database, no queue and no message bus. It has a schedule, a markdown spec, a git repo and a set of CLI tools. Everything it needs to know about previous runs it recovers by reading the repo.

That is a deliberate trade. An unattended writer that keeps its own state gets to drift: its idea of what has been published diverges from what is actually live, and the failure is invisible until a duplicate ships. Deriving state from the repo means the run is idempotent in the ways that matter — a missed day, a re-run, or a run from a different machine all produce the same behaviour, because the input is the current contents of `main`.

The cost is that every run pays to read a large data file. That is cheap compared to a wrong publish.

## Run sequence

```
schedule (weekday 9am)
  → read run spec
  → news sweep, biased toward a concrete Houston angle
  → read the existing archive, reject covered topics and taken slugs
  → draft 700–1100 words
  → brand-voice gate (banned list) → rewrite on hit
  → AEO structure pass (direct answer → H2 sections → bullets)
  → image: search → download → VIEW → reject mismatch → sips optimize into repo
  → insert post object at index 0 of POSTS
  → verify the module still parses and matches the existing shape
  → commit (data file + image together) → push main → deploy
```

Each stage can fail closed. The image stage loops back to search on a mismatch rather than accepting the best available. The verify stage blocks the commit rather than pushing a file that might not parse. Nothing is half-published, because a post is only real once the array entry and its image land in the same commit.

## The content store is one array

The site keeps every post in a single module exporting `POSTS` — an array of `{ slug, title, img, cat, date, read, excerpt, body }`, newest first, currently around 470KB across 82 posts. `body` is a backtick template literal in a lightweight markdown dialect: blank-line-separated paragraphs, `## ` headings, `**bold**`, `- ` bullets and `[text](url)` links.

This looks primitive next to a CMS. It buys three things that matter more than an admin UI here:

1. **The publish operation is a file edit.** The agent's write step is "insert an object at index 0", which is something a text-editing agent does reliably. No API client, no auth, no schema migration, no partially-created draft in someone's dashboard.
2. **Content is reviewable in a diff.** Every published post is a git commit with a `Blog:` prefix. What changed, when, and what the agent wrote are all visible in `git log` — the archive doubles as an audit trail of the automation.
3. **The site is static.** `/blog/[slug]` derives its params from the same array via `generateStaticParams`, so one insert produces a rendered route, a sitemap URL and a JSON-LD block with no other edits. There is no runtime fetch to fail.

Ordering has one piece of business logic on top of newest-first: a small pinned cluster of posts surfaces directly behind the newest post while it is seasonally relevant. The agent does not need to know about this — it only ever writes index 0.

## Brand voice as data

The voice rules live in the site's constants as a `VOICE` object with three pillars, a preferred list, and a banned list of 14 hype words and phrases. The banned list is the important half. Generic LLM marketing prose converges hard on exactly those words, so treating them as a compile error is a cheap, checkable proxy for "does this read like our company wrote it".

Two properties make this work better than a style paragraph in the prompt:

- It is **binary**. A word is present or it is not. There is no judgement call for the agent to rationalise its way through.
- It lives **in the codebase, not in the prompt**. The site and the agent read the same source. Changing the voice is a code change with a diff, and every downstream surface picks it up.

## Why the image is looked at

Image sourcing is where unattended publishing quietly fails. An agent can construct a plausible search query, extract a plausible CDN URL and download a real photo, and still end up with a stock desk shot on a post about a convention centre expansion. Every step succeeded; the output is wrong.

So the pipeline separates *fetching* from *accepting*. The candidate is downloaded to a scratch path, viewed, and judged against the actual topic — wrong subject or wrong city means go back and search again. Only an accepted image is re-encoded (fixed width, fixed JPEG quality) into the repo at a path derived from the slug, which is also what keeps a post and its hero from ever drifting apart.

The general rule this encodes: when a step's success condition is perceptual, verify it perceptually. An exit code of 0 from `curl` tells you a file arrived, not that it is the right file.

## Writing for answer engines

The post format is built backwards from how an AI answer engine consumes a page:

- **Direct answer first.** One to two sentences answering the implied question, before any setup. That block is what gets lifted.
- **H2 sections with self-contained claims.** A section that only makes sense after reading the previous one cannot be quoted.
- **At least one bulleted list.** Lists survive extraction; long paragraphs do not.
- **Real, linked sources.** Claims trace to URLs from the research step, which is both an honesty constraint and a citability one.

The site cooperates: each post emits `BlogPosting` schema (with ISO 8601 dates, parsed by string so a timezone can never shift the day), pillar posts can additionally carry `Service` and `FAQPage` schema, the sitemap enumerates every post, and `robots.js` names the major AI crawlers with an explicit allow rather than relying on a default.

## Failure modes worth naming

- **Silent renderer loss.** The body renderer originally handled only `**bold**`, so bullets and markdown links across the whole archive shipped as literal text — every internal link the agent wrote was inert. Nothing errored. The lesson: an authoring format and its renderer are one contract, and a gap between them fails silently at scale.
- **Schema dates discarded.** Human-formatted dates are invalid `schema.org` Date values, so Google dropped them entirely and the posts lost their freshness signal. An invalid value is worse than an absent one; the parser returns `undefined` on an unrecognised format so callers can omit the key rather than emit a wrong date.
- **Fabrication.** The hardest failure to detect after the fact, which is why it is blocked at the research step rather than reviewed at the end.

## What this repo is

A public architecture showcase of a private production codebase. Paths and structure are shown; the source, credentials and client-confidential material stay private.
