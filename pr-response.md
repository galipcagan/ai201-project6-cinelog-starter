# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI (Claude Code) throughout this project as a pair-programming and
research assistant, not as the source of the design decisions.

- **Orientation:** Before touching any review comment, I had it summarize
  `models.py`, `services/collection_service.py`, and `tests/test_collection.py`
  and explain what `add_to_collection()` returns/raises for a duplicate and a
  nonexistent film. I verified each claim against the actual source before
  relying on it (this is how I confirmed, for example, that `CollectionEntry`
  has a `UniqueConstraint` but `WatchlistEntry` does not — that fact drove
  Comment 2).
- **Implementation (Comments 1–3):** I wrote the rename, the deduplication
  guard, and the new test file myself, using `add_to_collection()` as the
  explicit pattern to mirror, per the project's own guidance not to have AI
  write the deduplication logic.
- **Comments 4 & 5 (design decisions):** These positions and the core
  reasoning are mine — I decided `public=False` for privacy reasons and
  `date_added desc` because recency of intent matters more than alphabetical
  lookup for a watchlist. Before I finalized either position, AI walked me
  through the tradeoffs of each option (e.g., that `public` is currently
  unenforced anywhere in the code, so the default only matters for future
  consumers and existing-entry exposure) so I could weigh them explicitly
  rather than guessing. I then had it help expand my stated position/reasoning
  into the full position/reasoning/tradeoff write-up required in this doc — the
  argument and the acknowledgment of the tradeoff are my own; AI's role was
  drafting and structuring, not originating the stance.
- **Rebase (Comment 6):** I used AI to dig into *why* the `models.py` conflict
  happened rather than just mechanically picking a side — comparing the diffs
  of the shared-ancestor commit, the UUID-refactor commit on `main`, and my own
  commits to establish that `main` had deleted the `WatchlistEntry` class as a
  side effect of the refactor. That understanding is what led to reintroducing
  the class with a migrated `String(36)` `film_id` instead of blindly resolving
  in favor of either side.
- **Commit hygiene:** Before finalizing, I had it review the final
  `git log --oneline` output against the conventional commits spec and confirm
  no commit bundles more than one logical change; I then checked that
  assessment myself against `CONTRIBUTING.md` before treating the history as
  final.

## Comment 1 — Rename
**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` so the watchlist service matches the project's
documented `verb_to_noun` naming convention (CONTRIBUTING.md: `add_to_collection`,
`remove_from_collection`, `get_collection`). "save" was the odd verb out — every
other write path in the codebase uses "add".

To find every call site I ran a project-wide search for `save_to_watchlist`
(`grep -r save_to_watchlist`, excluding `.venv/`). It returned three references:
the function definition in `services/watchlist_service.py`, and both the import
and the call in `routes/watchlist/watchlist.py`. I updated all three, then re-ran
the search to confirm zero remaining matches. I also updated the function's
docstring ("Save a film" → "Add a film") for consistency.

**How I verified:**
- `grep save_to_watchlist` returns no matches anywhere outside `.venv/`.
- `pytest tests/ -v` → all tests pass (the app imports the route module at
  startup, so a missed reference would surface as an ImportError).

## Comment 2 — Deduplication
**What I did:**
Added a deduplication guard to `add_to_watchlist()`, following the exact pattern
in `add_to_collection()` (`services/collection_service.py`). After the
film-exists check and before inserting, I query
`WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` and,
if a row already exists, raise a new `AlreadyInWatchlistError` instead of
silently creating a second entry. This mirrors `AlreadyInCollectionError` in the
collection service.

Unlike `CollectionEntry`, the `WatchlistEntry` model has **no** `UniqueConstraint`
on `(user_id, film_id)`, so without this check the endpoint would happily create
duplicate watchlist rows. I chose to match the collection service's
application-level check (raising a clear domain exception) rather than rely on a
DB constraint, so the behavior and error surface are identical to the collection
feature the reviewer already approved.

I also updated `routes/watchlist/watchlist.py` to catch the new exception and
return HTTP 409, and to catch `FilmNotFoundError` and return 404 — matching the
collection route exactly. Before this change the watchlist route wrapped neither,
so a duplicate or bad film_id would have returned a 500.

**How I verified:**
- Added `test_add_to_watchlist_duplicate_raises` (see Comment 3), which adds a
  film twice, asserts `AlreadyInWatchlistError` is raised, and confirms exactly
  one row exists in the DB afterward.
- `pytest tests/ -v` → all tests pass.

## Comment 3 — Missing test
**What I did:**
Created `tests/test_watchlist.py`, modeled directly on `tests/test_collection.py`.
I copied the same fixture structure (`app` with an in-memory SQLite DB,
`sample_user`, `sample_film`) and assertion style. The specifically requested
test is `test_add_to_watchlist_nonexistent_film_raises`, the equivalent of
`test_add_to_collection_nonexistent_film_raises`: it calls `add_to_watchlist`
with a film_id that doesn't exist and asserts `FilmNotFoundError` is raised.

Because CONTRIBUTING.md states a new service function should ship with happy-path,
duplicate/conflict, and nonexistent-ID tests, I also added
`test_add_to_watchlist_creates_entry` and `test_add_to_watchlist_duplicate_raises`
to cover the full minimum. (The nonexistent-film test originally used a fake
integer id, since film IDs were still integers on this branch at the time. It
was updated to a well-formed but nonexistent UUID string as part of the
Comment 6 rebase — see below.)

**How I verified:**
- `pytest tests/test_watchlist.py -v` → 3 passed.
- `pytest tests/ -v` → 7 passed (4 collection + 3 watchlist), nothing broken.

## Comment 4 — Default visibility
**My position:**
`WatchlistEntry.public` should default to `False`, not `True`. I changed
`models.py` accordingly (new watchlist entries are private unless a caller
explicitly opts in).

**Reasoning:**
My primary concern is privacy. A watchlist is a declaration of what a user
*intends* to watch — which can be more revealing than a collection of films
they've already watched. Users may be embarrassed by a guilty-pleasure pick,
not want their taste in an unfinished franchise judged, or simply not have
decided yet whether they want that information shared. Defaulting to
`public=True` exposes that by default, before the user has made any explicit
choice.

This is made worse by the current state of the codebase: there is no endpoint
anywhere that lets a user view or change their own visibility setting, and no
"public watchlists" browse feature exists yet either. Right now `public` is
inert metadata returned in every `to_dict()` — but the moment any feature reads
it to decide what to expose, a `True` default means every existing watchlist
entry created under the old default is public before the user ever
affirmatively agreed to that. Defaulting to `False` means privacy is the
resting state until a user (or a future UI toggle) explicitly opts in, which
is the safer failure mode for user data.

**Tradeoff acknowledged:**
CineLog is a *community* film tracking app, and defaulting to private cuts
against that — most users never change a default, so if a future feature
lets users discover what films their friends want to watch, a `False` default
will make that feature look far less populated than a `True` default would.
The maintainer's original `public=True` almost certainly optimized for that
kind of social discovery and engagement. I'm accepting that cost because I
think the privacy risk of over-sharing personal viewing intent by default
outweighs the benefit of maximizing default participation in a discovery
feature that doesn't exist yet — it's easier to grow visibility later (prompt
users to opt in) than to walk back an accidental public-by-default exposure
after the fact.

## Comment 5 — Sort order
**My position:**
`get_watchlist()` should sort by `date_added` descending (most recently added
first), matching the maintainer's preference and the pattern already used by
`get_collection()`. I changed `services/watchlist_service.py` accordingly and
removed the now-unnecessary `.join(Film)` that existed only to support the
alphabetical `Film.title.asc()` ordering.

**Reasoning:**
A watchlist is forward-looking — it reflects what a user is currently
interested in, not a static reference list. The films someone adds most
recently are the ones "everyone hasn't seen yet" and that they're most
actively excited about right now; that's the more useful signal to surface
first. Alphabetical order treats a film added five minutes ago the same as
one added eight months ago, which buries current intent under whatever letter
the title happens to start with.

**Engagement with reviewer's point:**
The alphabetical sort was my original implementation choice, reasoning that
it made scanning a long list for a specific title easier. I'm persuaded by
the maintainer's push toward `date_added` descending for two reasons: first,
it's the behavior collection already has, so a user (or another developer)
moving between the two features gets a consistent mental model instead of
having to remember that one feature sorts by recency and the other by title.
Second, and more importantly for a watchlist specifically, recency of intent
is more actionable than alphabetical position — a "want to watch" list is
usually consulted right after adding something new, not as a lookup table.
I do acknowledge the tradeoff: a long watchlist becomes harder to scan for one
specific known title without alphabetical order or a search feature. I think
that's an acceptable cost now, and a client-side or query-param sort/filter
option would be the right way to address it later rather than sacrificing the
more common "what's fresh" use case as the default.

## Comment 6 — Rebase
**What conflicted:**
I ran `git fetch origin` followed by `git rebase origin/main`. `origin/main`
had moved forward with `refactor: migrate film IDs from integer to UUID`
(`Film.id` and `CollectionEntry.film_id` changed from `Integer` to
`String(36)`) plus a `.gitignore` added independently on main.

Two conflicts surfaced:
1. **`.gitignore` (add/add):** both branches added a `.gitignore` independently
   while the PR was open. Textual conflict only — no semantic disagreement.
2. **`models.py` (content conflict) at the `fix: default watchlist entries to
   private` commit:** this was the real conflict. The shared ancestor commit
   already contained a `WatchlistEntry` class (with an `Integer` `film_id`,
   scaffolded before the watchlist feature branch's own commits), but main's
   UUID-refactor commit deleted that class entirely as a side effect of
   rewriting `models.py` — main has no watchlist feature, so from main's
   perspective the class was just unused code being cleared out. Meanwhile my
   branch's `fix: default watchlist entries to private (public=False)` commit
   only changed one line *inside* that class. Git couldn't 3-way-merge a
   single-line edit against a class that no longer existed on the `main` side,
   so it surfaced as a conflict rather than auto-deleting my change.

**How I resolved it:**
- For `.gitignore`, I merged both lists by hand (kept `.pytest_cache/` from
  main's version alongside `.venv/`, `venv/`, etc. from mine) so nothing either
  side needed got dropped.
- For `models.py`, I reintroduced the `WatchlistEntry` class into the
  post-refactor file, but migrated `film_id` from `db.Column(db.Integer, ...)`
  to `db.Column(db.String(36), db.ForeignKey("film.id"), nullable=False)` to
  match the new `Film.id` type, while keeping `public=False` (my Comment 4
  decision). Simply taking "my side" or "their side" of the conflict would
  have been wrong either way — main's side would have silently dropped the
  watchlist feature, and my side alone would have left `film_id` as an
  `Integer` foreign key pointing at a now-`String` primary key, which is a
  broken schema, not a resolved conflict.
- The conflict resolution only fixed the model. It didn't fix every place in
  the watchlist code that still assumed integer film IDs, so I went through
  the rest of the branch afterward and updated: the `add_to_watchlist()`
  docstring (`film_id (int)` → `film_id (str): UUID of the film.`), the
  `POST /watchlist/<user_id>/add` route docstring's example body
  (`"film_id": <int>` → `"film_id": "<uuid>"`), and the nonexistent-film test's
  fake id (`999999` → `"00000000-0000-0000-0000-000000000000"`, matching the
  pattern already used in `test_collection.py`). These landed as a follow-up
  `fix:` commit after the rebase completed.

**How I verified no conflict remains:**
- `git status` showed no unmerged paths after each `git add` + `git rebase
  --continue`, and the rebase reported "Successfully rebased and updated
  refs/heads/feature/watchlist."
- `git log --oneline --merges origin/main..HEAD` returns nothing — no merge
  commits, confirming a linear history on top of the rebased `main`.
- I then grepped the whole tree (outside `.venv/`) for `integer`, `int`, and
  `pre-refactor` to catch any leftover assumptions the textual merge wouldn't
  have flagged, which is how I found the three straggling references listed
  above.
- `pytest tests/ -v` → all 7 tests pass against the rebased models, including
  the updated UUID-based nonexistent-film test — this is the real proof the
  conflict is resolved semantically, not just textually, since a broken
  `film_id` foreign key type would have failed at table-creation or
  insert time, not just at merge time.

## Commit History

`git log --oneline` on `feature/watchlist` (rewritten via interactive rebase,
oldest first — 8 commits, no merge commits):

```
4a58a85 fix: update watchlist code and tests to use UUID film IDs after main refactor
1aaaa68 fix: sort watchlist by date added (newest first) instead of alphabetically
acca937 fix: default watchlist entries to private (public=False)
ab3b2fc test: add watchlist service tests for add, duplicate, and nonexistent film
c687bf8 fix: add deduplication check to prevent duplicate watchlist entries
184d048 fix: rename save_to_watchlist to add_to_watchlist per naming convention
ce5b05a fix: update film retrieval method to use db.session.get in collection and watchlist services
bafb23a feat: add watchlist service and endpoints for viewing and adding films
```

> **Note:** this is text output, not an image. Replace this block with an
> actual screenshot of `git log --oneline` before submitting — I wasn't able
> to capture a real screenshot in this environment.

## PR Description

**What this feature does:** Adds a watchlist to CineLog — a list of films a
user intends to watch, separate from their collection of films already
watched. New endpoints:

- `GET /watchlist/<user_id>` — returns the user's watchlist, sorted by most
  recently added first.
- `POST /watchlist/<user_id>/add` — adds a film to the watchlist. Body:
  `{ "film_id": "<uuid>" }`. Returns `201` on success, `404` if the film
  doesn't exist, `409` if the film is already on the user's watchlist.

**Design decisions made:**

1. **Default visibility (`public`):** New watchlist entries default to
   `public=False` (private), not `True`. Rationale: a watchlist reveals
   viewing *intent*, which can be more sensitive than a collection of films
   already watched, and there is currently no feature anywhere that lets a
   user review or change this setting — so privacy-by-default is the safer
   resting state until an explicit opt-in exists. Full reasoning and the
   tradeoff acknowledged (this weakens any future social-discovery feature)
   are in Comment 4 above.
2. **Sort order:** `GET /watchlist/<user_id>` returns entries sorted by
   `date_added` descending (most recent first), matching the convention
   already used by `get_collection()`, rather than alphabetically by title.
   Rationale: a watchlist is forward-looking, and recency of intent is a more
   useful signal than alphabetical position. Full reasoning and the
   acknowledged tradeoff (harder to scan for one known title in a long list)
   are in Comment 5 above.

**How to manually test:**

```bash
python app.py
```

1. Create a user and a film (no seed/admin endpoint exists for either in this
   starter, so insert directly via the Flask shell or a quick Python script
   using `models.User` / `models.Film`, then note their generated UUIDs).
2. Add a film to the watchlist:
   ```bash
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_uuid>"}'
   ```
   Expect `201` with the new entry, including `"public": false`.
3. View the watchlist:
   ```bash
   curl http://127.0.0.1:5000/watchlist/<user_id>
   ```
   Expect a list containing the film just added.
4. Add a second film, then a third — confirm the response from step 3 lists
   the most recently added film first (date-added descending).
5. Repeat step 2 with the same `film_id` — expect `409` and no new row
   created (deduplication).
6. Repeat step 2 with a `film_id` that doesn't exist (e.g.
   `00000000-0000-0000-0000-000000000000`) — expect `404`.
7. Run the automated suite: `pytest tests/ -v` — expect 7 passed (4 collection
   + 3 watchlist).
