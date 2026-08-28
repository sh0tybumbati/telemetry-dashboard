# Telemetry

Anonymous gameplay data, and a local dashboard that reads it. Built to take more
games over time, so the schema is game-agnostic and Silicon Tycoon is one entry.

Nothing is collected until you complete the setup below. With no URL configured
the game makes zero network calls.

## What is collected

Per run: the business model, difficulty, start era, and the company name. Every
8 in-game weeks: balance, year, node, mean line yield, cleanroom class, line
count. Milestones: first line built, contract signed, delivery (with lateness
and penalty), IPO, and the week the balance first goes negative.

A random id is stored in the browser so return visits can be counted. It is
linked to nothing and there is no fingerprinting. Players can opt out by calling
`optOut()` from `js/telemetry.js`.

The company name is free text and is collected deliberately. It is the one field
a player could type something personal into.

## Setup

1. Create a project at supabase.com. From Settings → API, copy the project URL
   and the `anon` key.

2. In the SQL editor, run these three files in order:

   ```
   sql/schema.sql
   sql/views-common.sql
   sql/panels-silicon-tycoon.sql
   ```

3. Register the game:

   ```sql
   insert into games values ('silicon-tycoon', 'Silicon Tycoon', 'week');
   ```

4. Under Authentication → Users, add a user for yourself. That login is what the
   dashboard signs in with.

5. Fill in `js/telemetryConfig.js`:

   ```js
   url:     'https://<project-ref>.supabase.co',
   anonKey: '<anon key>',
   enabled: true,
   ```

   The anon key is safe to commit. RLS grants it INSERT only, so it can write
   events and cannot read a single row back. The `service_role` key must never
   appear in this repo or in the dashboard.

6. Open `dashboard/index.html` in a browser and sign in. Credentials are stored
   in that browser, not in the repo.

If the browser blocks the request from `file://`, serve the folder instead:
`python3 -m http.server -d dashboard`.

## Verifying

```
npm run verify:schema      # runs the SQL against a real Postgres (pglite)
npm run verify:telemetry   # exercises the client with stubbed browser globals
npm run verify:dashboard   # drives the dashboard in a browser against a mock API
```

`verify:schema` proves the constraints reject bad values, that the anon key can
write but not read, and that the dashboard's role can read every view. Grants on
views are separate from grants on tables, and missing them was a real bug caught
by that harness.

## Adding a game

1. Add a row to `games`.
2. Add a `sql/panels-<game>.sql` with views named `p_<prefix>_*`, and grant
   select on them to `authenticated`.
3. Add an entry to the `GAMES` registry at the top of `dashboard/index.html`.

Game-specific fields live in the `meta` and `props` jsonb columns, so no
migration is needed. Add a guarded CHECK constraint for the new game's values,
following `runs_silicon_tycoon_meta` in `sql/schema.sql`.

## Notes

There is deliberately no foreign key from `events.run_id` to `runs.run_id`.
Transport is lossy: adblockers and closed tabs mean a `runs` insert can fail
while a later event succeeds. An FK would reject that real data.

Abandonment is derived from the last snapshot rather than sent as an event. The
game is multi-page, so every navigation looks identical to a tab close.

The player cannot go bankrupt. There is no loss condition, and a negative balance
only raises the chance of vulture investors. `insolvent` and `recovered` record
the crossings instead. `bankrupt` is a legal event name so a future loss
condition needs no migration.

Expect undercounting. itch serves the game in a cross-origin iframe, so requests
are third-party and a real share of players block them. Safari and Firefox also
partition that iframe's localStorage, which weakens the return-visit numbers.
Treat everything as relative, not as a headcount.
