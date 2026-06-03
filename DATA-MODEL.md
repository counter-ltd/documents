# Counter Data Model

Postgres. Nothing clever. Every table is straightforward, every relationship is explicit, and every piece of data that touches a user is documented here. No hidden columns. No shadow tables. This file is part of the open source commitment.

---

## users

The core identity record. Passwords are never stored — hashed with bcrypt, immediately discarded.

```sql
users
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  username        text          UNIQUE NOT NULL
  display_name    text
  bio             text
  avatar_url      text
  email           text          UNIQUE NOT NULL
  password_hash   text          NOT NULL
  verified        boolean       DEFAULT false
  created_at      timestamptz   DEFAULT now()
  updated_at      timestamptz   DEFAULT now()
```

---

## sessions

JWT refresh token tracking. Invalidated on logout. All sessions for a user can be wiped on account deletion.

```sql
sessions
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  token_hash      text          NOT NULL
  created_at      timestamptz   DEFAULT now()
  expires_at      timestamptz   NOT NULL
  last_used_at    timestamptz
```

---

## posts

The main content unit. Supports plain text. Replies are posts with a parent_id. Reposts reference the original.

```sql
posts
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  body            text
  parent_id       uuid          REFERENCES posts(id) ON DELETE SET NULL
  repost_of       uuid          REFERENCES posts(id) ON DELETE SET NULL
  edited          boolean       DEFAULT false
  deleted         boolean       DEFAULT false
  created_at      timestamptz   DEFAULT now()
  updated_at      timestamptz   DEFAULT now()
```

---

## media

Attachments on posts. Images, video. Stored externally (S3-compatible), referenced here.

```sql
media
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  post_id         uuid          REFERENCES posts(id) ON DELETE CASCADE
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  url             text          NOT NULL
  mime_type       text          NOT NULL
  width           integer
  height          integer
  size_bytes      integer
  alt_text        text
  created_at      timestamptz   DEFAULT now()
```

---

## follows

Directed graph of who follows who.

```sql
follows
  follower_id     uuid          REFERENCES users(id) ON DELETE CASCADE
  following_id    uuid          REFERENCES users(id) ON DELETE CASCADE
  created_at      timestamptz   DEFAULT now()
  PRIMARY KEY (follower_id, following_id)
```

---

## likes

```sql
likes
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  post_id         uuid          REFERENCES posts(id) ON DELETE CASCADE
  created_at      timestamptz   DEFAULT now()
  PRIMARY KEY (user_id, post_id)
```

---

## reposts

Tracked separately from posts for clean querying, even though reposts are also posts.

```sql
reposts
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  post_id         uuid          REFERENCES posts(id) ON DELETE CASCADE
  created_at      timestamptz   DEFAULT now()
  PRIMARY KEY (user_id, post_id)
```

---

## tags

Hashtags extracted from post bodies at write time.

```sql
tags
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  name            text          UNIQUE NOT NULL
  created_at      timestamptz   DEFAULT now()

post_tags
  post_id         uuid          REFERENCES posts(id) ON DELETE CASCADE
  tag_id          uuid          REFERENCES tags(id) ON DELETE CASCADE
  PRIMARY KEY (post_id, tag_id)
```

---

## notifications

```sql
notifications
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  type            text          NOT NULL  -- 'like' | 'repost' | 'reply' | 'follow' | 'mention'
  actor_id        uuid          REFERENCES users(id) ON DELETE CASCADE
  post_id         uuid          REFERENCES posts(id) ON DELETE SET NULL
  read            boolean       DEFAULT false
  created_at      timestamptz   DEFAULT now()
```

---

## insights

Aggregate only. No individual user is ever identified in this table. A view is an anonymous tick.

```sql
post_views
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  post_id         uuid          REFERENCES posts(id) ON DELETE CASCADE
  viewed_at       timestamptz   DEFAULT now()
  referrer        text          -- 'feed' | 'profile' | 'search' | 'direct' | 'external'

-- No user_id. No IP. No session ID. A view is a count, not a person.
```

---

## themes

Flat JSON of CSS variables. Validated on write, never executed.

```sql
themes
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  name            text          NOT NULL
  description     text
  variables       jsonb         NOT NULL  -- { "--color-bg": "#0d0d0d", ... }
  published       boolean       DEFAULT true
  created_at      timestamptz   DEFAULT now()
  updated_at      timestamptz   DEFAULT now()
```

---

## integrations

External platform links. Only public data is stored — no OAuth tokens for reading private data.

```sql
integrations
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  platform        text          NOT NULL  -- 'github' | 'bandcamp' | 'soundcloud' | 'letterboxd' | 'goodreads' | 'strava' | 'itch'
  platform_username text        NOT NULL
  platform_url    text
  verified        boolean       DEFAULT false
  created_at      timestamptz   DEFAULT now()
  updated_at      timestamptz   DEFAULT now()
  UNIQUE (user_id, platform)
```

---

## profile_theme

The theme a user has currently applied. Stored server-side for profile rendering. Local overrides live on-device only.

```sql
profile_themes
  user_id         uuid          PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE
  theme_id        uuid          REFERENCES themes(id) ON DELETE SET NULL
  custom_variables jsonb        -- user's local overrides, synced only if user opts in
  updated_at      timestamptz   DEFAULT now()
```

---

## algorithm_changelog

Every change to ranking is recorded here. Publicly readable via the API.

```sql
algorithm_changelog
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  version         text          NOT NULL
  summary         text          NOT NULL
  detail          text
  changed_by      text          NOT NULL  -- github username of committer
  commit_hash     text          NOT NULL  -- links back to the repo
  deployed_at     timestamptz   DEFAULT now()
```

---

## Data Deletion

When a user deletes their account:

- `users` row is hard deleted
- All `posts`, `likes`, `reposts`, `follows`, `notifications`, `sessions`, `integrations`, `themes` cascade delete
- `post_views` rows for their posts are retained as anonymous aggregate counts — no personal data remains
- Media files are deleted from storage within 24 hours

No soft deletes on user data. When you leave, you're gone.

---

## What We Don't Store

- IP addresses
- Device fingerprints
- Behavioral event streams
- Ad targeting signals
- Session replay data
- Individual view records tied to a person

If it's not in this file, it's not in the database.
