-- PACE App — Supabase Database Schema
-- Run this in your Supabase SQL editor
-- ============================================
 
-- Enable PostGIS for geo queries (nearby athletes)
create extension if not exists postgis;
 
-- ── PROFILES ──────────────────────────────────
create table profiles (
  id          uuid references auth.users primary key,
  username    text unique not null,
  full_name   text not null,
  avatar_url  text,
  bio         text,
  location    text,
  sports      text[] default '{}',
  level       text,
  goals       text[] default '{}',
  lat         float,
  lng         float,
  geo         geography(point, 4326),
  strava_id   bigint unique,
  created_at  timestamptz default now(),
  updated_at  timestamptz default now()
);
 
alter table profiles enable row level security;
 
create policy "Profiles are viewable by everyone"
  on profiles for select using (true);
 
create policy "Users can update their own profile"
  on profiles for update using (auth.uid() = id);
 
-- ── ACTIVITIES ────────────────────────────────
create table activities (
  id               uuid primary key default gen_random_uuid(),
  user_id          uuid references profiles(id) on delete cascade not null,
  sport            text not null,
  title            text not null,
  description      text,
  distance_km      float,
  duration_seconds integer,
  elevation_m      float,
  avg_hr           integer,
  avg_pace         integer,  -- seconds per km
  avg_speed        float,    -- km/h
  avg_power        integer,  -- watts
  location         text,
  route_polyline   text,
  strava_id        bigint unique,
  created_at       timestamptz default now()
);
 
alter table activities enable row level security;
 
create policy "Activities viewable by everyone"
  on activities for select using (true);
 
create policy "Users can insert own activities"
  on activities for insert with check (auth.uid() = user_id);
 
create policy "Users can update own activities"
  on activities for update using (auth.uid() = user_id);
 
create policy "Users can delete own activities"
  on activities for delete using (auth.uid() = user_id);
 
-- ── LIKES ─────────────────────────────────────
create table likes (
  user_id     uuid references profiles(id) on delete cascade,
  activity_id uuid references activities(id) on delete cascade,
  created_at  timestamptz default now(),
  primary key (user_id, activity_id)
);
 
alter table likes enable row level security;
 
create policy "Likes viewable by everyone" on likes for select using (true);
create policy "Users can like"   on likes for insert with check (auth.uid() = user_id);
create policy "Users can unlike" on likes for delete using (auth.uid() = user_id);
 
-- ── COMMENTS ──────────────────────────────────
create table comments (
  id          uuid primary key default gen_random_uuid(),
  activity_id uuid references activities(id) on delete cascade not null,
  user_id     uuid references profiles(id) on delete cascade not null,
  content     text not null,
  created_at  timestamptz default now()
);
 
alter table comments enable row level security;
 
create policy "Comments viewable by everyone" on comments for select using (true);
create policy "Users can comment" on comments for insert with check (auth.uid() = user_id);
create policy "Users can delete own comments" on comments for delete using (auth.uid() = user_id);
 
-- ── FOLLOWS ───────────────────────────────────
create table follows (
  follower_id  uuid references profiles(id) on delete cascade,
  following_id uuid references profiles(id) on delete cascade,
  created_at   timestamptz default now(),
  primary key (follower_id, following_id)
);
 
alter table follows enable row level security;
 
create policy "Follows viewable by everyone" on follows for select using (true);
create policy "Users can follow"   on follows for insert with check (auth.uid() = follower_id);
create policy "Users can unfollow" on follows for delete using (auth.uid() = follower_id);
 
-- ── EVENTS ────────────────────────────────────
create table events (
  id          uuid primary key default gen_random_uuid(),
  name        text not null,
  sport       text not null,
  date        date not null,
  location    text not null,
  country     text not null,
  lat         float,
  lng         float,
  distance_km float,
  description text,
  url         text,
  created_at  timestamptz default now()
);
 
create table event_attendees (
  user_id    uuid references profiles(id) on delete cascade,
  event_id   uuid references events(id) on delete cascade,
  created_at timestamptz default now(),
  primary key (user_id, event_id)
);
 
alter table events enable row level security;
alter table event_attendees enable row level security;
 
create policy "Events viewable by everyone" on events for select using (true);
create policy "Attendees viewable by everyone" on event_attendees for select using (true);
create policy "Users can join events"  on event_attendees for insert with check (auth.uid() = user_id);
create policy "Users can leave events" on event_attendees for delete using (auth.uid() = user_id);
 
-- ── VIEWS (for counts) ────────────────────────
create view activity_feed as
  select
    a.*,
    p.username, p.full_name, p.avatar_url, p.sports as profile_sports,
    count(distinct l.user_id) as likes_count,
    count(distinct c.id)      as comments_count
  from activities a
  join profiles p on a.user_id = p.id
  left join likes l on l.activity_id = a.id
  left join comments c on c.activity_id = a.id
  group by a.id, p.id;
 
-- ── NEARBY FUNCTION ───────────────────────────
create or replace function nearby_athletes(
  user_lat float, user_lng float, radius_km float default 10
)
returns table (
  id text, username text, full_name text,
  avatar_url text, sports text[], distance_km float
) language sql as $$
  select
    p.id::text, p.username, p.full_name,
    p.avatar_url, p.sports,
    round((st_distance(
      p.geo, st_point(user_lng, user_lat)::geography
    ) / 1000)::numeric, 1)::float as distance_km
  from profiles p
  where p.geo is not null
    and st_dwithin(p.geo, st_point(user_lng, user_lat)::geography, radius_km * 1000)
  order by distance_km asc
  limit 20;
$$;
 
