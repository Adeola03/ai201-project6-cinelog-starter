# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used an AI coding assistant (Claude) at several points, always verifying its output
against the actual code:

- **Orientation.** I had it summarize `models.py` and `services/collection_service.py`
  — responsibilities, main functions, and dependencies — before reading the review
  comments, then confirmed the summaries against the files (e.g. that `add_to_collection`
  checks film existence *then* uniqueness before committing).
- **Understanding the dedup pattern (Comment 2).** I asked it to walk through
  `add_to_collection()` step by step and say what it returns when the film doesn't exist
  vs. when it's a duplicate. I then wrote the `add_to_watchlist` check myself from that
  understanding rather than having it generate the code.
- **Test structure (Comment 3).** I asked what pattern each test in `test_collection.py`
  follows and what a new test needs (fixtures + `pytest.raises`), then mirrored it.
- **Stress-testing the design arguments (Comments 4 and 5).** After drafting both
  positions I asked the AI to argue the *opposite* side — "what would a careful reviewer
  say against this?" For Comment 4 it raised privacy-by-design / GDPR Art. 25, the
  "asymmetric harm" of an un-undoable public exposure, and the risk that an unchosen
  default edges toward a dark pattern. Those were stronger than my first draft, which
  only argued the social-engagement upside, so I revised the response to concede the
  asymmetric-harm point explicitly and add concrete mitigations (informed default at
  add-time/onboarding, per-entry `public` toggle, revisit if semantics change). For
  Comment 5 the counterargument (alphabetical aids findability in long lists) was one I
  had already addressed, so I kept my position and folded in the "optional `sort=title`
  param" concession.
- **Commit-message check.** I had it sanity-check my `git log --oneline` against the
  Conventional Commits spec (prefixes, imperative mood, one logical change each), then
  verified against the spec myself.

All final reasoning, code, and commit messages are my own; where an AI summary of the
code disagreed with the code, I trusted the code.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` to match the project's `verb_to_noun` convention
(mirrors `add_to_collection` in the collection service). Updated the one call site
in `routes/watchlist/watchlist.py` — both the `import` statement (line 8) and the
call inside `add_film` (line 32).

**How I verified:** Before editing I ran a project-wide search
(`grep -rn "save_to_watchlist"`) to enumerate every reference — it returned exactly
three hits: the definition plus the import and call in the route. After renaming I
re-ran the same search and got zero matches, confirming no call site was missed.
The full suite (`pytest tests/ -v`) still passes (4 passed), so nothing that
imports the service broke.

## Comment 2 — Deduplication
**What I did:** `add_to_watchlist()` previously inserted a new `WatchlistEntry`
unconditionally, so calling it twice created duplicate rows. I mirrored the pattern
already used by `add_to_collection()` in the collection service: after the
film-existence check and before inserting, query for an existing entry with the same
`(user_id, film_id)` and raise a domain-specific error if one is found. I added an
`AlreadyInWatchlistError` exception class (parallel to the collection service's
`AlreadyInCollectionError`) rather than reusing the collection error, so callers can
distinguish which list rejected the add.

**How I verified:** I read `add_to_collection()` first and traced its flow: it calls
`db.session.get(Film, film_id)` (returns `None` for a missing film → `FilmNotFoundError`),
then `CollectionEntry.query.filter_by(user_id, film_id).first()` — if that returns a
row it raises `AlreadyInCollectionError` and never reaches the insert/commit; only a
brand-new pair gets added. I wrote the watchlist check the same way. `pytest tests/ -v`
still passes, and the new duplicate behavior is covered by the test added in Comment 3.
I also wired the error through the route (`routes/watchlist/watchlist.py`) so it returns
HTTP `409 Conflict` instead of a `500` — again matching how the collection endpoint
surfaces `AlreadyInCollectionError` — and confirmed it with a Flask test-client smoke
run (second add of the same film → `409`).

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`. I used
`test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py` as my
model and wrote the direct equivalent, `test_add_to_watchlist_nonexistent_film_raises`:
same `app` / `sample_user` fixtures, the same `"00000000-...-000000000000"` fake id,
and the same `pytest.raises(FilmNotFoundError)` assertion. I copied the fixture block
verbatim from `test_collection.py` so the two test modules stay structurally identical.
I also added `test_add_to_watchlist_duplicate_raises` (modeled on
`test_add_to_collection_duplicate_raises`) to lock in the Comment 2 dedup behavior —
it asserts the second add raises `AlreadyInWatchlistError` and that only one row
exists afterward.

**How I verified:** `pytest tests/test_watchlist.py -v` → 2 passed. Full suite
`pytest tests/ -v` → 6 passed, confirming the new module coexists with the existing
collection tests and nothing regressed.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default for a new `WatchlistEntry`, but
make it an *informed* default — surface it at add-time and at onboarding — and keep
the per-entry `public` flag so any single entry can be made private.

**Reasoning:** CineLog is a social film platform; its core loop is discovery through
other people's taste. A watchlist ("films I want to watch") is one of the most useful
social signals on the platform — it lets friends recommend, plan a watch-together, or
gift a film. Defaults decide behavior: the overwhelming majority of users never open
settings, so an opt-in (`public=False`) default would leave nearly every watchlist
private-by-inertia and starve the discovery feed, which undercuts the product's reason
to exist. I'm also deliberately drawing a line between the two lists: a *collection*
(films watched + rated) is a behavioral record and far more revealing, whereas a
watchlist is aspirational intent. If any list is safe to default public, it's the
watchlist — so the default is defensible precisely because of what a watchlist *is*.

**Tradeoff acknowledged:** The opposite choice — `public=False`, privacy-by-default —
is the safer, least-surprise stance, and I'm accepting a real cost by not taking it.
The harm is asymmetric: an accidentally-public watchlist is an exposure the user can't
undo, while an accidentally-private one is a one-toggle fix — that asymmetry genuinely
favors private, and privacy-by-design (e.g. GDPR Art. 25) leans the same way. There's
also a fair objection that engagement won from a default the user never actively chose
edges toward a dark pattern. I'm accepting these because CineLog's value proposition is
social and a watchlist is low-sensitivity intent — but I mitigate the real risks rather
than dismiss them: (1) the `public` boolean is per-entry, so nothing is all-or-nothing;
(2) the default must be disclosed clearly at the point of adding a film and in
onboarding, so it's an informed default, not a hidden one; and (3) if watchlists ever
take on more sensitive semantics, this default should be revisited and likely flipped.

## Comment 5 — Sort order
**My position:** I agree with the maintainer and implemented it: `get_watchlist()`
now orders by `date_added` descending (newest first) instead of alphabetically by
title. Changed in `services/watchlist_service.py`.

**Reasoning:** Two concrete reasons beyond preference. (1) Consistency —
`get_collection()` already returns entries newest-first (`date_added.desc()`). Having
the two list endpoints on the same platform sort by different keys is a surprising
inconsistency: it makes the API harder to reason about and forces the UI to special-case
each list. Matching the established collection behavior is the least-astonishing choice.
(2) Intent freshness — a watchlist is a queue of things the user wants to watch, and the
film they just added is the strongest signal of what they want *now*. Newest-first puts
that at the top with no scrolling. Alphabetical ties an item's position to its title,
which is arbitrary relative to the user's intent ("Amélie" sits at the top forever) and
gets worse as the list grows.

**Engagement with reviewer's point:** The maintainer preferred date-added, and I not
only adopted it but grounded *why* it's correct rather than just deferring. I also
weighed the strongest case for the existing alphabetical order: it's stable and
predictable, and it makes a specific title easy to find in a long list. I don't think
that justifies keeping it as the server default — findability in a large list is better
served by client-side sort controls and search than by fixing the default sort to
titles — but it's a real benefit, which is why I'd support exposing an optional
`sort=title` query parameter later rather than removing alphabetical as an option
entirely.

## Comment 6 — Rebase
**What conflicted:** Two things, only one of which git surfaced as a textual conflict.

1. `.gitignore` (checkout collision). `main` already tracks a `.gitignore` (a superset
   of the one I had created — it also ignores `.pytest_cache/`). My locally-created
   `.gitignore` was still untracked, so `git rebase origin/main` would have aborted with
   "untracked working tree files would be overwritten by checkout." I removed my untracked
   copy before rebasing and let main's tracked version take over — no reason to keep a
   redundant duplicate, which is also why the branch has no separate `.gitignore` commit.

2. `WatchlistEntry` / UUID (the real, semantic conflict). `main` refactored film IDs from
   integer to UUID: `Film.id` and `CollectionEntry.film_id` are now `db.String(36)`. Main's
   `models.py` predates the watchlist feature, so it has **no** `WatchlistEntry` class at all.
   Because none of my branch commits actually edited `models.py` (the model happened to sit in
   the fork's base commit), the rebase applied cleanly with *no* git-reported conflict — but it
   silently left me on main's `models.py`, dropping `WatchlistEntry` entirely. My watchlist
   service still did `from models import WatchlistEntry`, so the code was broken even though git
   reported success.

**How I resolved it:** After the rebase I re-added `WatchlistEntry` to `models.py`, placed
after `CollectionEntry` and matching the post-refactor schema — `film_id` is now
`db.Column(db.String(36), db.ForeignKey("film.id"))` instead of `db.Integer` (I kept
`public` defaulting to `True` per Comment 4). I also updated the now-stale integer references
in my watchlist code: the `film_id (int) — pre-refactor` docstring in
`services/watchlist_service.py` became `film_id (str): UUID of the film`, and the route's
request-body comment in `routes/watchlist/watchlist.py` changed from `{ "film_id": <int> }`
to `<str>` (UUID). Committed as a single `fix:` for the UUID migration.

**How I verified no conflict remains:**
- `grep "^class " models.py` shows `User`, `Film`, `CollectionEntry`, `WatchlistEntry`, and
  every `film_id` column is `db.String(36)` — no integer IDs left.
- Before the fix, `pytest tests/` failed with `ImportError: cannot import name
  'WatchlistEntry'`; after re-adding the model the suite passes, confirming the import and the
  UUID round-trip (fixtures create UUID film IDs and the service accepts them).
- While verifying end-to-end with the Flask test client I also caught a latent bug the
  refactor exposed: `get_watchlist()` reads `entry.film`, but `WatchlistEntry` had no
  relationship to `Film` (only `CollectionEntry` did, via a backref), so viewing a watchlist
  returned a `500`. I added `film = db.relationship("Film")` to `WatchlistEntry` and added a
  `test_get_watchlist_returns_newest_first` test so it stays fixed.
- `git log --oneline --merges origin/main..HEAD` prints nothing — the branch is linear with no
  merge commits — and `git log --oneline origin/main..HEAD` shows my work replayed directly on
  top of the refactored `main`.

## PR Description

### What this adds
A **watchlist** feature: films a user wants to watch later, kept separate from their
collection (films already watched). It adds a `WatchlistEntry` model and two endpoints:

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/watchlist/<user_id>/add` | Add a film (`{ "film_id": "<uuid>" }`) to the user's watchlist |
| `GET`  | `/watchlist/<user_id>` | Return the user's watchlist, newest-added first |

Behavior mirrors the existing collection feature: a missing `film_id` in the body → `400`,
an unknown film → `404`, a film already on the watchlist → `409`, and a successful add → `201`.

### Design decisions
1. **Default visibility — `public=True`.** New watchlist entries default to public. CineLog
   is a social platform whose value is discovery through other people's taste, and a
   watchlist is low-sensitivity *intent* (not a behavioral record like a collection), so it
   is the safest list to surface by default. The tradeoff — privacy-by-default would be the
   least-surprising, lower-risk choice — is real; I mitigate it by keeping a per-entry
   `public` flag and requiring the default to be disclosed at add-time. (Full argument in
   the Comment 4 section above.)
2. **Sort order — newest-added first.** `get_watchlist()` orders by `date_added` descending
   rather than alphabetically, both for consistency with `get_collection()` and because the
   most recently added film is the freshest signal of what the user wants to watch now.
   (Full argument in the Comment 5 section above.)

### How to test manually
```bash
# 1. Install and run (serves http://localhost:5000, creates cinelog.db)
pip install -r requirements.txt
python app.py

# 2. In a second terminal, seed a user and two films, capturing their UUIDs
python - <<'PY'
from app import create_app, db
from models import User, Film
app = create_app()
with app.app_context():
    u  = User(username="ada", email="ada@example.com")
    f1 = Film(title="Amelie", year=2001)
    f2 = Film(title="Zodiac", year=2007)
    db.session.add_all([u, f1, f2]); db.session.commit()
    print("USER_ID =", u.id); print("FILM_1 =", f1.id); print("FILM_2 =", f2.id)
PY
# export the printed values, e.g. USER_ID=... FILM_1=... FILM_2=...

# 3. Add two films (FILM_1 first, then FILM_2)
curl -X POST localhost:5000/watchlist/$USER_ID/add \
     -H "Content-Type: application/json" -d "{\"film_id\":\"$FILM_1\"}"
curl -X POST localhost:5000/watchlist/$USER_ID/add \
     -H "Content-Type: application/json" -d "{\"film_id\":\"$FILM_2\"}"

# 4. View the watchlist: FILM_2 (added last) appears FIRST, and each entry has "public": true
curl localhost:5000/watchlist/$USER_ID

# 5. Deduplication: re-adding FILM_1 returns HTTP 409
curl -i -X POST localhost:5000/watchlist/$USER_ID/add \
     -H "Content-Type: application/json" -d "{\"film_id\":\"$FILM_1\"}"

# 6. Unknown film returns 404; empty body returns 400
curl -i -X POST localhost:5000/watchlist/$USER_ID/add \
     -H "Content-Type: application/json" -d "{\"film_id\":\"does-not-exist\"}"
curl -i -X POST localhost:5000/watchlist/$USER_ID/add \
     -H "Content-Type: application/json" -d "{}"
```
Run the automated tests with `pytest tests/ -v`.

### Commit history
Final `git log --oneline origin/main..HEAD` after interactive rebase (8 commits,
conventional format, no merge commits). Short hashes are illustrative — the tip hash
changes with the final amend/force-push; run `git log --oneline` for live values or see
the attached screenshot.

```
docs: add pr-response.md with review responses and design decisions
fix: add WatchlistEntry model with UUID film_id after rebasing on main
fix: sort watchlist by date added (newest first) for consistency with collection
test: add tests for nonexistent film and duplicate in add_to_watchlist
fix: add deduplication check to prevent duplicate watchlist entries
fix: rename save_to_watchlist to add_to_watchlist per naming convention
fix: update film retrieval method to use db.session.get in collection and watchlist services
feat: add watchlist service and endpoints
```

