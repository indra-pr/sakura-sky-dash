# Shared leaderboard setup

Sakura Sky Dash can use Supabase for one leaderboard shared by every browser. Until these steps are completed, the game continues to use browser-local scores.

## 1. Create the database

Create a free Supabase project, open **SQL Editor**, and run:

```sql
create table public.sakura_sky_dash_scores (
  nickname_key text primary key,
  nickname text not null,
  score integer not null check (score >= 0 and score <= 100000),
  updated_at timestamptz not null default now(),
  constraint nickname_length check (char_length(nickname) between 1 and 16)
);

alter table public.sakura_sky_dash_scores enable row level security;

create policy "Anyone can read Sakura Sky Dash scores"
on public.sakura_sky_dash_scores
for select
to anon, authenticated
using (true);

create or replace function public.submit_sakura_score(player_name text, player_score integer)
returns void
language plpgsql
security definer
set search_path = public
as $$
declare
  clean_name text := trim(regexp_replace(player_name, '\s+', ' ', 'g'));
begin
  if char_length(clean_name) not between 1 and 16 then
    raise exception 'Nickname must contain 1 to 16 characters';
  end if;

  if player_score not between 0 and 100000 then
    raise exception 'Score is outside the allowed range';
  end if;

  insert into public.sakura_sky_dash_scores (nickname_key, nickname, score)
  values (lower(clean_name), clean_name, player_score)
  on conflict (nickname_key) do update
  set nickname = case
        when excluded.score > sakura_sky_dash_scores.score then excluded.nickname
        else sakura_sky_dash_scores.nickname
      end,
      score = greatest(sakura_sky_dash_scores.score, excluded.score),
      updated_at = case
        when excluded.score > sakura_sky_dash_scores.score then now()
        else sakura_sky_dash_scores.updated_at
      end;
end;
$$;

revoke all on function public.submit_sakura_score(text, integer) from public;
grant execute on function public.submit_sakura_score(text, integer) to anon, authenticated;
```

## 2. Connect the game

In both `index.html` and `sakura-sky-dash.html`, find `sharedScoreConfig` and insert the values from **Project Settings > API**:

```js
const sharedScoreConfig = {
  url: "https://YOUR_PROJECT.supabase.co",
  anonKey: "YOUR_PUBLIC_ANON_KEY"
};
```

The anon key is intended for browser use and is constrained by the database policies above. Never put the Supabase service-role key in HTML or commit it to GitHub.

## Behavior

- The top 20 shared personal-best scores are loaded when the game opens and when a player returns home.
- A nickname keeps only its highest submitted score.
- If Supabase is unavailable or not configured, the game falls back to scores stored in that browser.
- Because game logic runs in the browser, a determined user could submit a fabricated score. A cheat-resistant leaderboard requires a trusted game server that validates each run.
