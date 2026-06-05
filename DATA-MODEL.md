# Counter Data Model

Postgres. Nothing clever. Every table is straightforward, every relationship is explicit, and every piece of data that touches a user is documented here. No hidden columns. No shadow tables. This file is part of the open source commitment.

---

## users

The core identity record. Passwords are never stored — hashed with bcrypt, immediately discarded.

```sql
users
  id                          uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  username                    text          UNIQUE NOT NULL
  display_name                text
  bio                         text
  avatar_url                  text
  email                       text          UNIQUE NOT NULL
  password_hash               text          NOT NULL
  verified                    boolean       DEFAULT false
  online_status_enabled       boolean       DEFAULT false  -- off by default; user must opt in
  online_status_visibility    text          DEFAULT 'everyone'  -- 'everyone' | 'followers' | 'mutualFollowers'
  last_seen_enabled           boolean       DEFAULT false
  last_seen_visibility        text          DEFAULT 'everyone'
  last_seen_at                timestamptz                  -- updated by POST /users/me/heartbeat
  heartbeat_interval_seconds  integer       DEFAULT 300    -- client fires heartbeat this often; server online window = interval + 30s
  created_at                  timestamptz   DEFAULT now()
  updated_at                  timestamptz   DEFAULT now()
```

`online_status_enabled` and `last_seen_enabled` are independent toggles. Each has its own visibility column that controls who can see that field. Both are off by default. `last_seen_at` is updated every time the client calls the heartbeat endpoint, regardless of toggle state, so re-enabling a feature shows accurate data immediately.

---

## device_keys

Per-device E2EE key pairs. One row per registered (user, device) pair. A user
can have multiple registered devices, each with its own P-256 key pair. Senders
encrypt a separate copy of each message for every device so all of them can
decrypt. The private key never leaves the device; only the public half is stored
here.

```sql
device_keys
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  device_id       text          NOT NULL  -- stable UUID generated on the client device
  public_key      text          NOT NULL  -- SPKI base64 P-256 public key
  created_at      timestamptz   DEFAULT now()
  UNIQUE (user_id, device_id)
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

## conversations

One row per pair of participants. Participant ids are stored in lexicographic
order (participantA < participantB) so a unique index on the pair prevents
duplicate conversation rows without a read-before-write.

```sql
conversations
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  participant_a   uuid          REFERENCES users(id) ON DELETE CASCADE
  participant_b   uuid          REFERENCES users(id) ON DELETE CASCADE
  last_message_at timestamptz   DEFAULT now()  -- drives inbox sort order
  created_at      timestamptz   DEFAULT now()
  UNIQUE (participant_a, participant_b)
```

---

## messages

Individual entries within a conversation. `body` is either plaintext (legacy),
AES-256-GCM server-encrypted ciphertext, or a `v3:` multi-device E2EE payload.
Plaintext is never written to the database in either encryption path.

`kind` distinguishes real messages from system events. Screenshot entries have
an empty body and are excluded from unread counts and inbox last-message previews.

```sql
messages
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  conversation_id uuid          REFERENCES conversations(id) ON DELETE CASCADE
  sender_id       uuid          REFERENCES users(id) ON DELETE CASCADE
  body            text          NOT NULL  -- plaintext, AES ciphertext, or v3: E2EE payload
  kind            text          NOT NULL DEFAULT 'message'  -- 'message' | 'screenshot'
  read            boolean       DEFAULT false  -- flipped when the recipient views the thread
  created_at      timestamptz   DEFAULT now()
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
  type            text          NOT NULL  -- 'like' | 'repost' | 'reply' | 'follow' | 'mention' | 'message'
  actor_id        uuid          REFERENCES users(id) ON DELETE CASCADE
  post_id         uuid          REFERENCES posts(id) ON DELETE SET NULL
  conversation_id uuid          REFERENCES conversations(id) ON DELETE CASCADE  -- set only for 'message'
  read            boolean       DEFAULT false
  created_at      timestamptz   DEFAULT now()
```

---

## notification_preferences

A user's muted notification types. The presence of a row means "don't send me
this type", on any channel. Absence means on, so every account defaults to
all-on with no per-user row to create up front.

```sql
notification_preferences
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  type            text          NOT NULL  -- a notifications.type value; row present = muted
  created_at      timestamptz   DEFAULT now()
  PRIMARY KEY (user_id, type)
```

---

## devices

Registered push devices, one row per APNs token. Registration is opt-in: the
client never auto-uploads a token; the user registers from Settings > Privacy >
Devices. `name` is a user-supplied label (e.g. "John's iPhone") so multiple
devices are distinguishable in the settings panel.

```sql
devices
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  platform        text          NOT NULL  -- 'ios'
  token           text          NOT NULL UNIQUE
  name            text          NULL      -- user-visible label; null for pre-name devices
  created_at      timestamptz   DEFAULT now()
  last_seen_at    timestamptz   DEFAULT now()
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

## Retention

The CSL requires us to state how long every category of data is kept. The rule
is simple: identity-linked data lives exactly as long as the account does, and
nothing is kept "just in case."

| Data | Purpose | Retention |
|------|---------|-----------|
| `users` | Identity, login | Life of the account. Deleted within 30 days of account deletion. |
| `sessions` | Keep you logged in across devices | Until logout or token expiry, whichever comes first. Wiped on account deletion. |
| `posts` | Your content | Life of the account, or until you delete the post. Removed within 30 days of account deletion. |
| `media` | Attachments on posts | Same as the parent post. Files purged from storage within 24 hours of deletion. |
| `follows`, `likes`, `reposts` | The social graph and interactions | Life of the account. Cascade-deleted with it. |
| `tags`, `post_tags` | Topic discovery | Tag rows are shared and persist; the link to your post is removed when the post is. |
| `conversations`, `messages` | Private direct messages | Life of the account. Cascade-deleted with it. Screenshot events are stored as message rows with `kind = 'screenshot'` and are covered by the same retention. |
| `notifications` | Telling you what happened | Life of the account. Cascade-deleted with it. |
| `notification_preferences` | Which notification types you've muted | Life of the account. Cascade-deleted with it. |
| `device_keys` | E2EE public key per registered device | Life of the account, or until the device re-registers with a new key (upsert replaces the old row). Cascade-deleted with the account. The private key is never stored here. |
| `devices` | Push delivery address (opaque APNs token) | Life of the account, or until you sign out on that device. Cascade-deleted with the account. No device or location data. |
| `themes`, `profile_themes` | Themes you published or applied | Life of the account. Cascade-deleted with it. |
| `integrations` | Public links to other platforms | Life of the account, or until you unlink. No private tokens stored. |
| `post_views` | Anonymous aggregate view counts | Retained indefinitely as counts only. Never tied to a person, so nothing identifying survives. |
| `algorithm_changelog` | Public record of ranking changes | Retained permanently. Contains no user data. |

If a category is not listed here, it is not collected. This table is updated
within 7 days of any change to data collection, as the CSL requires.

---

## Data Deletion

When a user deletes their account:

- `users` row is hard deleted
- All `posts`, `likes`, `reposts`, `follows`, `notifications`, `notification_preferences`, `devices`, `sessions`, `integrations`, `themes` cascade delete
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
