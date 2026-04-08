# Media List Generation Guide

How to research, vet, and generate curated movie/TV show lists with TMDB IDs.

---

## Output Format

Each list is a JSON object:

```json
{
  "name": "Top TV Docuseries",
  "owner": "mediacircle",
  "profile": "MediaCircle AI researches and selects the best of the best",
  "description": "Bingeable TV Series Documentaries",
  "image": "https://imagedelivery.net/...",
  "circleType": "movies_and_tv",
  "tags": ["binge", "tv", "series"],
  "priority": 3,
  "items": [
    { "type": "tv", "tmdb_id": 64439 },
    { "type": "movie", "tmdb_id": 550 }
  ]
}
```

### Field Reference

| Field | Type | Notes |
|-------|------|-------|
| `name` | string | Short display title for the list |
| `owner` | string | Always `"mediacircle"` |
| `profile` | string | Tagline for the curator identity |
| `description` | string | One-line description of the list theme |
| `image` | string | Cloudflare Images URL for the list cover art |
| `circleType` | string | Always `"movies_and_tv"` |
| `tags` | string[] | 2–5 lowercase tags for discoverability |
| `priority` | number | Sort order (lower = higher priority) |
| `items` | object[] | Ordered array of `{ type, tmdb_id }` |

`type` is either `"tv"` or `"movie"`.

---

## Step-by-Step Workflow

### Step 1 — Define Criteria

Before researching, nail down:
- **Media type**: movies, TV series, or mixed
- **Genre/theme**: e.g. docuseries, true crime, sci-fi thrillers
- **Quality bar**: critic score threshold, award recognition, cultural impact
- **Time range**: all-time classics, last 5 years, a specific decade
- **Audience**: general, niche, family, etc.
- **List size**: typically 20–50 items

Document criteria before starting research so vetting is consistent.

---

### Step 2 — Research Phase

Use web search to build a candidate pool. Cast wide — you will narrow later.

**Sources to consult:**
- Rotten Tomatoes (critic + audience scores)
- IMDb Top lists and genre charts
- Metacritic
- Letterboxd (for film, community rankings)
- Wikipedia "List of..." articles for genre/era
- Streaming platform editorial lists (Netflix, HBO, etc.)
- Award databases: Emmy, Oscar, BAFTA, Sundance

**Target 1.5–2× the final list size** as candidates before vetting.

---

### Step 3 — Vetting

For each candidate, verify:
- [ ] Matches the defined criteria (genre, type, era)
- [ ] Meets quality bar (e.g. RT ≥ 80%, IMDb ≥ 7.5, or award nominated)
- [ ] Actually available/notable (not obscure or unreleased)
- [ ] No duplicates or alternate versions already in the list
- [ ] Mixed lists: check for reasonable movie/TV balance

Rank and trim to final list size.

---

### Step 4 — TMDB ID Lookup

The TMDB API key is stored in **OneCLI's vault** (not in `.env`). Access is via the OneCLI proxy at `127.0.0.1:10255`, which acts as a MITM SSL proxy that injects the TMDB key into outbound requests automatically. This works in both Claude Code CLI sessions and inside agent containers.

The proxy credentials and CA cert are established during OneCLI setup. The CA cert is cached at `/tmp/onecli-ca.pem`.

#### Search for a TV show

```bash
curl -s --proxy "http://x:ONECLI_KEY@127.0.0.1:10255" --cacert /tmp/onecli-ca.pem \
  "https://api.themoviedb.org/3/search/tv?query=SHOW+NAME&language=en-US&page=1"
```

#### Search for a movie

```bash
curl -s --proxy "http://x:ONECLI_KEY@127.0.0.1:10255" --cacert /tmp/onecli-ca.pem \
  "https://api.themoviedb.org/3/search/movie?query=MOVIE+NAME&language=en-US&page=1"
```

> **Note:** The `ONECLI_KEY` is the `aoc_...` token from the OneCLI setup. It is stored in `settings.local.json` in the allowed Bash commands from prior sessions. Direct calls to `api.themoviedb.org` without the proxy will return 401.

The `id` field in the first result is the TMDB ID. Always verify the result matches by checking `name` (TV) or `title` (movie) and `first_air_date` / `release_date`.

#### Verify a TV show by ID

```bash
curl -s --proxy "http://x:ONECLI_KEY@127.0.0.1:10255" --cacert /tmp/onecli-ca.pem \
  "https://api.themoviedb.org/3/tv/TMDB_ID"
```

#### Verify a movie by ID

```bash
curl -s --proxy "http://x:ONECLI_KEY@127.0.0.1:10255" --cacert /tmp/onecli-ca.pem \
  "https://api.themoviedb.org/3/movie/TMDB_ID"
```

#### Disambiguation

When search returns multiple results (common titles, remakes, international versions):
- Match on `original_name` / `original_title`
- Match on `first_air_date` year or `release_date` year
- For remakes, confirm the correct version by checking `overview` or fetching full details

---

### Step 5 — Assemble JSON

Once all TMDB IDs are confirmed:

1. Build the `items` array in ranked order (best/most relevant first)
2. Fill in all metadata fields per the field reference above
3. Assign `priority` relative to other lists in the same collection (lower number = shown first)
4. Choose 2–5 tags that match the theme and audience

---

## TMDB API Quick Reference

Base URL: `https://api.themoviedb.org/3`

| Endpoint | Purpose |
|----------|---------|
| `GET /search/tv?query=` | Search TV shows by name |
| `GET /search/movie?query=` | Search movies by name |
| `GET /tv/{id}` | Get full TV show details |
| `GET /movie/{id}` | Get full movie details |
| `GET /tv/{id}/external_ids` | Get IMDb ID and other external IDs |
| `GET /discover/tv` | Filter TV by genre, date, rating |
| `GET /discover/movie` | Filter movies by genre, date, rating |
| `GET /genre/tv/list` | List all TV genre IDs |
| `GET /genre/movie/list` | List all movie genre IDs |

### Useful `discover` parameters

| Param | Example | Notes |
|-------|---------|-------|
| `with_genres` | `99` | Genre ID (99 = Documentary) |
| `sort_by` | `vote_average.desc` | Sort order |
| `vote_count.gte` | `100` | Minimum vote count (avoids noise) |
| `vote_average.gte` | `7.5` | Minimum rating |
| `first_air_date.gte` | `2015-01-01` | Earliest air date |
| `language` | `en-US` | Response language |

### Common genre IDs

**TV:** Action & Adventure: 10759 · Animation: 16 · Comedy: 35 · Crime: 80 · Documentary: 99 · Drama: 18 · Kids: 10762 · Mystery: 9648 · News: 10763 · Reality: 10764 · Sci-Fi & Fantasy: 10765 · Talk: 10767 · War & Politics: 10768 · Western: 37

**Movies:** Action: 28 · Adventure: 12 · Animation: 16 · Comedy: 35 · Crime: 80 · Documentary: 99 · Drama: 18 · Horror: 27 · Romance: 10749 · Science Fiction: 878 · Thriller: 53 · War: 10752

---

## Tips

- **Search URL-encode** spaces as `+` or `%20` in query params
- **Batch lookups**: loop through your vetted title list with search calls, capture IDs, verify each
- **Popular titles with common names** (e.g. "The Office"): always check year to pick the right version
- **Anime**: TMDB sometimes lists under original Japanese title — try both English and Japanese name
- **Miniseries vs series**: TMDB type is always `"tv"` regardless of episode count
- **Specials and stand-up**: use `"tv"` for stand-up specials listed as TV on TMDB, `"movie"` if listed under movies
