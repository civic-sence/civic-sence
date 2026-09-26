# CivicSense

A focused social platform for **civic sense** and good deeds only.

Share the positive actions you take every day — helping others, keeping public spaces clean, following rules, protecting nature, showing kindness. Nothing else is allowed.

**Design:** Clean black & white, minimal, professional. No gradients, no clutter.

---

## Features

- **Civic-only posts** — Text, images, and videos about good deeds
- **Social Coins** — Earned from likes on your posts
- **Coins** — Bonus reward for every post (+2)
- **Follow system** — Follow people, view followers/following
- **Feed** — All posts or only people you follow
- **Direct messaging** — Chat with other users
- **Profiles** — Public profiles with stats
- **Auth** — Email/password via Supabase

---

## Tech Stack

| Layer        | Technology              |
|--------------|-------------------------|
| Frontend     | HTML + vanilla JS       |
| Backend      | Supabase (Auth, DB, Storage) |
| Styling      | Embedded CSS (B&W theme) |

No frameworks. No build step. Just open the files.

---

## Project Structure

```
CivicSense/
├── index.html          # Landing page
├── login.html          # Log in
├── signup.html         # Sign up
├── app.html            # Main feed
├── post.html           # Create post
├── account.html        # Profile (self & others)
├── followers.html      # Followers / Following
├── settings.html       # Edit profile + logout
├── chat.html           # Conversations & messaging
├── supabase.js         # Supabase client + helpers
└── README.md
```

---

## Setup

### 1. Create a Supabase project

1. Go to [supabase.com](https://supabase.com) and create a free project.
2. Open **Project Settings → API**.
3. Copy the **Project URL** and **anon public** key.

### 2. Configure the client

Open `supabase.js` and replace the placeholders:

```js
const SUPABASE_URL = 'https://YOUR_PROJECT_REF.supabase.co';
const SUPABASE_ANON_KEY = 'YOUR_ANON_KEY_HERE';
```

### 3. Run the database schema

Go to **SQL Editor → New query** and paste the entire contents of the SQL block below, then run it.

<details>
<summary>Click to expand full SQL schema</summary>

```sql
-- ============================================================
-- CivicSense Schema
-- ============================================================

-- Profiles (extends auth.users)
create table public.profiles (
  id uuid references auth.users on delete cascade primary key,
  username text unique not null,
  full_name text,
  avatar_url text,
  bio text,
  social_coins integer not null default 0,
  coins integer not null default 0,
  created_at timestamptz not null default now()
);

-- Posts
create table public.posts (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references public.profiles(id) on delete cascade,
  content text not null,
  media_url text,
  media_type text check (media_type in ('image', 'video')),
  likes_count integer not null default 0,
  created_at timestamptz not null default now()
);

-- Likes
create table public.likes (
  id uuid primary key default gen_random_uuid(),
  post_id uuid not null references public.posts(id) on delete cascade,
  user_id uuid not null references public.profiles(id) on delete cascade,
  created_at timestamptz not null default now(),
  unique (post_id, user_id)
);

-- Follows
create table public.follows (
  id uuid primary key default gen_random_uuid(),
  follower_id uuid not null references public.profiles(id) on delete cascade,
  following_id uuid not null references public.profiles(id) on delete cascade,
  created_at timestamptz not null default now(),
  unique (follower_id, following_id),
  check (follower_id <> following_id)
);

-- Messages
create table public.messages (
  id uuid primary key default gen_random_uuid(),
  sender_id uuid not null references public.profiles(id) on delete cascade,
  receiver_id uuid not null references public.profiles(id) on delete cascade,
  content text not null,
  read boolean not null default false,
  created_at timestamptz not null default now()
);

-- Indexes
create index posts_user_id_idx on public.posts(user_id);
create index posts_created_at_idx on public.posts(created_at desc);
create index likes_post_id_idx on public.likes(post_id);
create index likes_user_id_idx on public.likes(user_id);
create index follows_follower_id_idx on public.follows(follower_id);
create index follows_following_id_idx on public.follows(following_id);
create index messages_sender_id_idx on public.messages(sender_id);
create index messages_receiver_id_idx on public.messages(receiver_id);

-- ============================================================
-- Auto-create profile on signup
-- ============================================================
create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer set search_path = public
as $$
begin
  insert into public.profiles (id, username, full_name)
  values (
    new.id,
    coalesce(new.raw_user_meta_data->>'username', 'user_' || substr(new.id::text, 1, 8)),
    coalesce(new.raw_user_meta_data->>'full_name', '')
  );
  return new;
end;
$$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();

-- ============================================================
-- Like → update likes_count + Social Coins
-- ============================================================
create or replace function public.handle_like()
returns trigger
language plpgsql
security definer set search_path = public
as $$
declare
  post_owner uuid;
begin
  if tg_op = 'INSERT' then
    update public.posts
      set likes_count = likes_count + 1
      where id = new.post_id;

    select user_id into post_owner from public.posts where id = new.post_id;
    update public.profiles
      set social_coins = social_coins + 1
      where id = post_owner;

  elsif tg_op = 'DELETE' then
    update public.posts
      set likes_count = greatest(0, likes_count - 1)
      where id = old.post_id;

    select user_id into post_owner from public.posts where id = old.post_id;
    update public.profiles
      set social_coins = greatest(0, social_coins - 1)
      where id = post_owner;
  end if;
  return null;
end;
$$;

create trigger on_like_change
  after insert or delete on public.likes
  for each row execute procedure public.handle_like();

-- ============================================================
-- RPC: add coins (for posting reward)
-- ============================================================
create or replace function public.add_coins(p_user_id uuid, p_amount integer)
returns void
language plpgsql
security definer set search_path = public
as $$
begin
  update public.profiles
    set coins = coins + p_amount
    where id = p_user_id;
end;
$$;

-- ============================================================
-- Row Level Security
-- ============================================================
alter table public.profiles enable row level security;
alter table public.posts enable row level security;
alter table public.likes enable row level security;
alter table public.follows enable row level security;
alter table public.messages enable row level security;

-- Profiles
create policy "Public profiles are viewable by everyone"
  on public.profiles for select using (true);

create policy "Users can update own profile"
  on public.profiles for update using (auth.uid() = id);

create policy "Users can insert own profile"
  on public.profiles for insert with check (auth.uid() = id);

-- Posts
create policy "Posts are viewable by everyone"
  on public.posts for select using (true);

create policy "Users can create posts"
  on public.posts for insert with check (auth.uid() = user_id);

create policy "Users can delete own posts"
  on public.posts for delete using (auth.uid() = user_id);

-- Likes
create policy "Likes are viewable by everyone"
  on public.likes for select using (true);

create policy "Users can like"
  on public.likes for insert with check (auth.uid() = user_id);

create policy "Users can unlike"
  on public.likes for delete using (auth.uid() = user_id);

-- Follows
create policy "Follows are viewable by everyone"
  on public.follows for select using (true);

create policy "Users can follow"
  on public.follows for insert with check (auth.uid() = follower_id);

create policy "Users can unfollow"
  on public.follows for delete using (auth.uid() = follower_id);

-- Messages
create policy "Users can view their messages"
  on public.messages for select
  using (auth.uid() = sender_id or auth.uid() = receiver_id);

create policy "Users can send messages"
  on public.messages for insert
  with check (auth.uid() = sender_id);

create policy "Users can update own received messages (read status)"
  on public.messages for update
  using (auth.uid() = receiver_id);
```

</details>

### 4. Create Storage bucket for media

1. Go to **Storage → New bucket**.
2. Name: `media`
3. Set it to **Public**.
4. Add these policies (Storage → media → Policies):

```sql
-- Allow authenticated users to upload into their own folder
create policy "Authenticated users can upload"
on storage.objects for insert
to authenticated
with check (
  bucket_id = 'media'
  and auth.uid()::text = (storage.foldername(name))[1]
);

-- Allow public read access
create policy "Public read access"
on storage.objects for select
to public
using (bucket_id = 'media');
```

### 5. Run the app

Open `index.html` in a browser, or serve the folder with any static server:

```bash
# Example with Python
python -m http.server 3000

# Example with Node
npx serve .
```

Then visit `http://localhost:3000`.

---

## How Coins Work

| Action                    | Reward              |
|---------------------------|---------------------|
| Someone likes your post   | +1 Social Coin      |
| You create a post         | +2 Coins            |
| Unlike removes the Social Coin from the post owner | — |

Social Coins and Coins are stored on the `profiles` table and updated automatically by database triggers / RPC.

---

## Content Rules

Only posts about **civic sense** and good deeds are intended:

- Helping others
- Keeping public spaces clean
- Following traffic / social rules
- Protecting nature / animals
- Small acts of kindness
- Community improvement

The UI reminds users of this rule when creating a post.

---

## License

MIT — feel free to use and modify.

---

## Roadmap (ideas)

- [ ] Leaderboard by Social Coins
- [ ] Coin redemption / rewards
- [ ] Push notifications
- [ ] Better media compression
- [ ] Dark mode toggle
- [ ] Report / moderation tools
