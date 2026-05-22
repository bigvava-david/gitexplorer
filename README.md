# GitStarExplorer

A single static HTML page that shows the top 12 trending GitHub repositories across four time windows (Day, Week, Month, Year) and three definitions of "trending." No backend, no build step, no dependencies — open `index.html` in a browser and you're done.

## Why

The GitHub REST API doesn't expose "stars gained in a period" directly, which is what users usually mean when they ask "what's trending this week." The closest you can get from the public REST API is repos created (or pushed) in a window, sorted by total stars — useful but not the same answer. GitStarExplorer offers both: GitHub Search API queries for the proxy answers, plus an OSSInsight-backed mode for the real one.

## Modes

The mode toggle below the time tabs picks how "trending" is defined:

- **New** — repos *created* in the window, ranked by total stars. Best for finding recent launches and brand-new projects.
- **Active** — repos *recently pushed* (i.e. with new commits) in the window, ranked by total stars. Useful for finding established projects that are being actively shipped on right now.
- **Trending** — repos by *stars gained* in the window. Backed by [ossinsight.io](https://ossinsight.io), which tracks star events over time. This is the closest to what most people mean by "trending."
- **Sleeper** — established repos gaining stars fast (high velocity, but *not* newly created in the window). Combines OSSInsight trending data with GitHub "created in window" filtering.

New and Active hit `api.github.com/search/repositories` directly. Trending and Sleeper hit `api.ossinsight.io/v1/trends/repos/`.

## Time windows

Day, Week, Month, Year — pill buttons at the top.

OSSInsight's trending endpoint doesn't expose a yearly window, so the Year tab is disabled while Trending mode is active. If you switch to Trending while on Year, the app snaps to Month.

## Running it

It's one HTML file. Three options:

```
# 1. Just open it
open index.html

# 2. Local dev server (any static server works)
python3 -m http.server 8000
# then visit http://localhost:8000

# 3. Deploy to any static host
# Drop index.html into GitHub Pages, Netlify, Vercel, S3, etc.
```

There's no build step. The whole app — HTML, CSS, JS — lives in `index.html`.

## GitHub rate limits & token

Unauthenticated, the GitHub Search API allows 60 requests per hour per IP. That's shared with anything else on your network using the same IP, so you can hit it sooner than you'd think. With a personal access token, it goes up to 5,000/hour.

To add one: open the **Settings** disclosure at the bottom of the page, paste a `ghp_...` token, click Save. The token is stored only in your browser's `localStorage` and only sent to `api.github.com` as a `Bearer` header. It's never sent to OSSInsight or any other service.

Create a token at [github.com/settings/tokens?type=beta](https://github.com/settings/tokens?type=beta). It requires **no scopes** — the search endpoint needs no permissions; the token just identifies you for the higher quota.

If your token is rejected, the UI surfaces a 401 error and a hint to check Settings. If you hit the rate limit, the UI tells you how many minutes until reset.

## OSSInsight (Trending mode)

OSSInsight is a public service run by PingCAP that aggregates GitHub event data. Their `trends/repos` endpoint returns repositories ranked by stars gained in a given period. It's free, no auth required, and rate-limited to 600 requests per hour per IP.

The app maps OSSInsight's response shape (`data.rows[].{ repo_name, stars, forks, primary_language, description }`) into the same card format used for the GitHub-backed modes, so the UI is identical regardless of source.

If OSSInsight is unavailable, only Trending mode breaks — New and Active continue to work.

## Caching

Every `(window × mode)` combination is cached for 30 minutes in `localStorage`. After the first request, switching between cached views is instant. Cache keys include both the window and the mode, so the three modes don't trample each other.

To force a fresh fetch, clear `localStorage` for the page (DevTools → Application → Local Storage). A manual refresh button is on the short-list of nice-to-haves.

## What's actually in the file

A bit over 500 lines of HTML/CSS/JavaScript. Roughly:

- ~250 lines of CSS (single dark theme, GitHub-ish palette)
- ~50 lines of HTML (header, time tabs, mode toggle, results grid, settings disclosure, footer)
- ~250 lines of JS (URL builders for both APIs, fetch + cache, OSSInsight response adapter, render functions, event handlers)

Vanilla JS, no framework. The entire app is one IIFE. Cards are rendered by string-templating into `innerHTML` (with HTML-escaped content).

## Caveats

- **"New" doesn't find sleeper hits.** A two-year-old repo that picked up 5,000 stars this week won't appear in New mode — it shows repos created in the window, not stars gained in the window. Use Trending mode for that question.
- **"Day" window in New mode is often thin.** Very few brand-new repos accumulate enough stars in 24 hours to rank highly. The 12-result grid will sometimes show repos with single-digit stars on the Day/New view.
- **Trending depends on a third party.** OSSInsight is stable and has been running for years, but if their service breaks, Trending mode breaks until it's fixed. New and Active are unaffected.
- **No language filter yet.** Currently shows all languages. A language dropdown is the most obvious next addition.
- **No URL state.** Filters aren't reflected in the URL, so a particular view isn't shareable or bookmarkable.

## Tech stack

HTML, CSS, vanilla JavaScript. One file. Zero dependencies, zero build configuration. Renders in any modern browser.

## Data sources

- [GitHub Search API](https://docs.github.com/rest/search/search#search-repositories) — for New and Active modes
- [OSSInsight Public API](https://ossinsight.io/docs/api/list-trending-repos) — for Trending mode
