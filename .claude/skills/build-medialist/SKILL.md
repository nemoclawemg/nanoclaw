---
name: build-medialist
description: Research, vet, and generate a curated MediaCircle movie/TV list with verified TMDB IDs. Use when asked to create, build, or generate a media list based on a theme, genre, era, or other criteria.
---

Build a curated MediaCircle list by following these steps. Full field reference and API docs are in `docs/MEDIALIST-GENERATION.md`.

## Step 1 — Clarify criteria

Before researching, confirm with the user:
- **Media type**: movies, TV, or mixed
- **Theme/genre**: era, genre, mood, award type, etc.
- **Quality bar**: all-time greats, cult favorites, mainstream hits, etc.
- **List size**: default to 25–35 items if not specified

If the request is clear enough, proceed without asking.

## Step 2 — Research candidates

Use `WebSearch` and `WebFetch` to build a candidate pool (1.5–2× the target size). Consult:
- Rotten Tomatoes genre/era lists
- IMDb Top 250 and genre charts
- AFI 100 lists (for films)
- Emmy and Oscar winner databases
- Wikipedia "List of..." articles
- Streaming platform editorial lists

Vet each candidate: confirm it matches the criteria, meets a quality bar (RT ≥ 80% or IMDb ≥ 7.5 or award recognition), and is not a duplicate.

## Step 3 — Look up TMDB IDs

TMDB credentials are stored in OneCLI and injected automatically via the proxy at `127.0.0.1:10255`. The CA cert is at `/tmp/onecli-ca.pem`.

```bash
PROXY="http://x:ONECLI_KEY@127.0.0.1:10255"
CACERT="/tmp/onecli-ca.pem"
```

Get the `ONECLI_KEY` from `.claude/settings.local.json` — look for the `aoc_...` token in the allowed curl commands.

**Search TV:**
```bash
curl -s --proxy "$PROXY" --cacert "$CACERT" \
  "https://api.themoviedb.org/3/search/tv?query=SHOW+NAME&language=en-US&page=1"
```

**Search movie:**
```bash
curl -s --proxy "$PROXY" --cacert "$CACERT" \
  "https://api.themoviedb.org/3/search/movie?query=MOVIE+NAME&language=en-US&page=1"
```

The `id` field in the first result is the TMDB ID. Always verify `name`/`title` and `first_air_date`/`release_date` match. Batch lookups in a shell loop for efficiency.

**Verify by ID:**
```bash
curl -s --proxy "$PROXY" --cacert "$CACERT" "https://api.themoviedb.org/3/tv/TMDB_ID"
curl -s --proxy "$PROXY" --cacert "$CACERT" "https://api.themoviedb.org/3/movie/TMDB_ID"
```

## Step 4 — Assemble JSON

Output format:

```json
{
  "name": "List Display Name",
  "owner": "mediacircle",
  "profile": "MediaCircle AI researches and selects the best of the best",
  "description": "One-line theme description",
  "image": "",
  "circleType": "movies_and_tv",
  "tags": ["tag1", "tag2", "tag3"],
  "priority": 1,
  "items": [
    { "type": "tv", "tmdb_id": 12345 },
    { "type": "movie", "tmdb_id": 67890 }
  ]
}
```

- `owner` is always `"mediacircle"`
- `image` is a Cloudflare Images URL — leave blank if not provided
- `tags`: 2–5 lowercase strings
- `priority`: lower number = shown first; ask the user if placement matters
- Items ordered best/most relevant first

## Step 5 — Save and deliver

Save the file to `data/lists/<descriptive-slug>.json`.

Always scp the file to the user's local machine without asking:
```bash
scp -i ~/.ssh/id_ed25519 data/lists/<file>.json ericgould@192.168.4.114:/tmp/
```

## Notes

- Run all TMDB lookups without asking for permission — the API key is already configured
- For ambiguous titles (remakes, international versions), match on year and `original_title`
- Miniseries and stand-up specials on TMDB are `"tv"` unless TMDB categorizes them as movies
- See `docs/MEDIALIST-GENERATION.md` for full TMDB endpoint reference and genre ID table
