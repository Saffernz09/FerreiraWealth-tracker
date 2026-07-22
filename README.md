# Ferreira Wealth OS

A single-file household wealth tracker: net worth snapshots, FIRE maths, projections,
goals, cash flow, budget, and a property decision engine. Everything lives in
`index.html` — host it anywhere static (GitHub Pages works out of the box) and it
syncs through Supabase.

## How data & privacy work

- **Your real numbers live in Supabase**, behind email + password logins and
  row-level security. Each household's data is a single row only its members can
  read or write. Nothing you log is ever written into this repository.
- **The source file only contains fictional seed values** and the Supabase
  *publishable* (anon) key, which is designed to be public. Keep the `service_role`
  key secret and never put it in this repo.
- **Friends can't see your data.** When someone signs up they start from the
  generic starter values and create (or join) their *own* household. The invite
  code only joins the household of the person who shares it.
- **Demo mode**: the login screen has an "Explore the demo" button that loads a
  fully fictional household with 14 months of generated history. It runs entirely
  in memory — nothing is saved, synced, or shared — so anyone can poke around
  before creating an account.

### Supabase security checklist (worth re-verifying occasionally)

1. RLS is **enabled** on `households`, `household_members`, and `household_data`.
2. Policies only allow members of a household to select/update that household's row.
3. The `create_household` / `join_household` functions are `security definer` and
   validate their inputs.
4. Email confirmation is on (Supabase Auth → Providers → Email).

## Keeping the free Supabase project awake

Supabase pauses free projects after a period of no activity. The GitHub Action in
`.github/workflows/keep-supabase-alive.yml` writes a row to a dedicated keepalive
table twice a week (Mondays and Thursdays), which is unambiguous database activity.
You can also trigger it manually from the Actions tab ("Keep Supabase awake" →
Run workflow).

An earlier version of this workflow just read from `household_data` with the
public anon key. Row-level security correctly blocks that from returning
anything, and Supabase's auto-pause detector apparently doesn't count a blocked
read as real usage — the project got flagged for pausing even though the
workflow ran successfully every time. Writing a row sidesteps the ambiguity.

**One-time setup** (a few minutes, only needs doing once):

1. In the Supabase dashboard, open **SQL Editor** and run:
   ```sql
   create table if not exists public._keepalive (
     id int primary key default 1,
     pinged_at timestamptz not null default now()
   );
   alter table public._keepalive enable row level security;
   -- No policies are added on purpose: with RLS on and zero policies, neither
   -- the anon key nor logged-in users can read or write this table at all.
   -- Only the service_role key (which bypasses RLS by design) can touch it.
   ```
2. In **Project Settings → API**, copy the **service_role** key (marked *secret*
   — this is different from the publishable/anon key already in `index.html`,
   and must never be put in this repo or any client-side file).
3. In this GitHub repo, go to **Settings → Secrets and variables → Actions →
   New repository secret**, name it `SUPABASE_SERVICE_ROLE_KEY`, and paste the
   key as the value.
4. From the **Actions** tab, run "Keep Supabase awake" once manually to confirm
   it succeeds.

One caveat: GitHub pauses *scheduled workflows* in repos with no pushes for 60
days — it emails you first, and one click ("keep workflow enabled") or any commit
re-arms it.

## Sharing it with friends

Send them the hosted URL. They can:

1. Tap **Explore the demo** to look around with fictional data, then
2. **Create an account**, create their own household, and start logging their own
   snapshots — the app moulds to their situation:
   - **Single or couple**: the household type on the **Assumptions** page adds or
     removes the second partner everywhere (switching hides, never deletes).
   - Partner names, birth dates, the child fund's name, and both cities are all
     editable on the same page.
   - **Budget scenarios** are fully theirs: rename, add or delete scenario columns
     and budget rows to compare any set of income situations.
   - Every goal target and date, and every projection assumption, is editable.

## Development

No build step. Edit `index.html`, open it in a browser, done. The Supabase URL and
publishable key are the two constants at the top of the `<script>` block.
