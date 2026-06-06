# Counter Data Model

Postgres. Nothing clever. Every table is straightforward, every relationship is explicit, and every piece of data that touches a user is documented here. No hidden columns. No shadow tables. This file is part of the open source commitment.

---

## users

The core identity record. Passwords are hashed with PBKDF2-SHA256 (100k iterations, random salt) and the plaintext is immediately discarded. `password_hash` is null for OAuth-only accounts that signed up through GitHub or Discord.

`email` is encrypted at rest with AES-256-GCM (stored as `v1:<iv>:<ciphertext>`), so a database dump never exposes addresses in plain text. Because the ciphertext is randomised per write it can't be queried, so `email_index` holds a keyed blind index (HMAC-SHA256 of the lower-cased address); it carries the uniqueness constraint and every lookup at login, signup, and OAuth link.

```sql
users
  id                          uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  username                    text          UNIQUE NOT NULL
  display_name                text
  bio                         text
  avatar_url                  text                       -- served avatar URL (cached from the object)
  avatar_object_id            uuid          REFERENCES media_objects(id) ON DELETE SET NULL
  email                       text          NOT NULL   -- encrypted: v1:<iv>:<ciphertext>
  email_index                 text          UNIQUE NOT NULL  -- HMAC-SHA256 blind index, lookup key
  password_hash               text                       -- null for OAuth-only accounts
  verified                    boolean       DEFAULT false
  online_status_enabled       boolean       DEFAULT false  -- off by default; user must opt in
  online_status_visibility    text          DEFAULT 'everyone'  -- 'everyone' | 'followers' | 'mutualFollowers'
  last_seen_enabled           boolean       DEFAULT false
  last_seen_visibility        text          DEFAULT 'everyone'
  last_seen_at                timestamptz                  -- updated by POST /users/me/heartbeat
  heartbeat_interval_seconds  integer       DEFAULT 300    -- client fires heartbeat this often; server online window = interval + 30s
  messaging_privacy           text          DEFAULT 'everyone'  -- 'everyone' | 'followers' | 'nobody'
  typing_indicators_enabled   boolean       DEFAULT true   -- relay this user's typing over the conversation socket; opt-out
  status                      text          DEFAULT 'active' NOT NULL  -- 'active' | 'suspended' | 'banned'
  status_reason               text                       -- moderator's note, shown to the user on a blocked login
  suspended_until             timestamptz                -- suspension expiry; null for a ban or an active account
  bot_kind                    text                       -- null for humans; names a bot persona (e.g. 'thing_one') for server-designated bot accounts
  created_at                  timestamptz   DEFAULT now()
  updated_at                  timestamptz   DEFAULT now()
```

`status` is the moderation state, set from the admin panel. `'banned'` is indefinite; `'suspended'` lasts until `suspended_until`. Both block sign-in and have the account's sessions revoked when applied, so the block is immediate rather than waiting for the access token to expire. A suspension whose `suspended_until` has passed is flipped back to `'active'` on the next login attempt, so it lifts itself.

`online_status_enabled` and `last_seen_enabled` are independent toggles. Each has its own visibility column that controls who can see that field. Both are off by default. `last_seen_at` is updated every time the client calls the heartbeat endpoint, regardless of toggle state, so re-enabling a feature shows accurate data immediately.

`messaging_privacy` controls who can start a new conversation. `'everyone'` allows direct messages from anyone. `'followers'` restricts direct messages to accounts that follow the recipient; anyone else can still send one message request, which the recipient can accept or decline. `'nobody'` blocks all incoming messages and requests.

`bot_kind` is the bot allowlist. It is null for every normal account; a non-null value (e.g. `'thing_one'`) marks the account as a server-designated bot and names its persona. It is only ever set server-side (migration, seed, or direct SQL), never through any API surface, so a client cannot promote an arbitrary account to a bot. A bot replies when a post or reply @mentions it, and also when a new reply lands in a thread it is already part of (so it keeps talking without being re-tagged); the reply is authored by the bot and threaded under the post it answers. A bot post never triggers further bot replies (no loops) and a bot answers any given post at most once (no duplicates). Bot accounts also cannot be DMed: `POST /messages/:username` rejects them outright, regardless of `messaging_privacy`.

`typing_indicators_enabled` is on by default. When on, the live conversation channel (the `ConversationHub` Durable Object) relays this user's typing to the person they're chatting with. The typing signal is ephemeral, it lives only in the socket and is never written to any table, so it leaves no trail. The toggle is enforced server-side: the `GET /messages/:username/live` route reads the flag and the hub drops typing frames from a user who turned it off, so a patched client can't leak typing the user disabled.

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

## password_resets

Pending password-reset tokens, one row per outstanding request. Only the SHA-256 of a high-entropy random token is stored, never the token itself, so a database leak can't be replayed to seize an account. Issued by the public "forgot password" flow and by an admin with `users.reset_password`; redeeming sets a new `password_hash`, burns every token for the user, and wipes their sessions. Tokens expire after one hour.

```sql
password_resets
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  token_hash      text          NOT NULL
  expires_at      timestamptz   NOT NULL
  created_at      timestamptz   DEFAULT now()
```

---

## webauthn_credentials

Registered passkeys. Each row is an authentication factor, not message encryption (that's `device_keys`). Counter stores only the credential's public key and the signature counter, never anything that could impersonate the authenticator. The counter only moves forward; a value that goes backwards is the classic cloned-authenticator signal and fails verification. base64url strings come straight from the WebAuthn ceremony and are stored verbatim.

```sql
webauthn_credentials
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  credential_id   text          NOT NULL          -- base64url, unique
  public_key      text          NOT NULL          -- base64url COSE key
  counter         bigint        NOT NULL DEFAULT 0
  transports      text                            -- JSON array, nullable
  device_type     text                            -- 'singleDevice' | 'multiDevice'
  backed_up       boolean       NOT NULL DEFAULT false
  nickname        text                            -- user label
  created_at      timestamptz   DEFAULT now()
  last_used_at    timestamptz
```

---

## webauthn_challenges

Short-lived passkey ceremony nonces, one row per in-flight registration or authentication, deleted when the ceremony finishes. A challenge is a single-use nonce (useless without the matching private key), not a bearer credential, so it's stored in plaintext rather than hashed the way refresh tokens are. `user_id` is null for login challenges, where the account isn't known until the assertion resolves a credential.

```sql
webauthn_challenges
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  challenge       text          NOT NULL
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE   -- null for login
  ceremony        text          NOT NULL          -- 'registration' | 'authentication'
  expires_at      timestamptz   NOT NULL
  created_at      timestamptz   DEFAULT now()
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
  removed_by_admin boolean      DEFAULT false NOT NULL  -- true when a moderator removed it, vs the author self-deleting
  created_at      timestamptz   DEFAULT now()
  updated_at      timestamptz   DEFAULT now()
```

Both an author's self-delete and a moderator removal set `deleted`. `removed_by_admin` distinguishes the two, so the author can't quietly un-remove a moderated post and the moderation queue can show a post's state.

These are all soft-deletes: the row stays so replies and reposts pointing at it (`parent_id` / `repost_of`, both `SET NULL` on delete) keep their shape. The one exception is the `posts.nuke` admin action, which hard-`DELETE`s the post and the whole subtree reachable through those two edges in a single statement. That cascade clears the post's `media`, `likes`, `reposts`, `post_tags`, and `post_views`; the two referrers that would otherwise be orphaned (`notifications.post_id`, `reports` targeting the post) are deleted in the same statement. A nuke is irreversible and leaves only the `admin_audit_log` entry behind.

---

## media_objects

The physical layer of media storage: one row per unique blob in R2, keyed by
the sha256 of its bytes (which is also the R2 object key, `objects/{sha256}`).
Content-addressing means identical uploads dedup to a single object, so the same
avatar shared by many users, or the same image posted twice, is stored once.

```sql
media_objects
  id                  uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  sha256              text          NOT NULL UNIQUE   -- hex digest; also the R2 key
  mime_type           text          NOT NULL
  size_bytes          integer       NOT NULL
  width               integer
  height              integer
  ref_count           integer       NOT NULL DEFAULT 0   -- how many things point at this blob
  created_at          timestamptz   DEFAULT now()
  last_referenced_at  timestamptz   DEFAULT now()
```

`ref_count` tracks how many references hold the object alive: post `media` rows,
user avatars (`users.avatar_object_id`), and cached Discord profiles
(`discord_profiles.object_id`). It starts at 0 on upload and is incremented when
something attaches, decremented when it detaches (post deleted, avatar replaced).
An hourly sweep deletes any object that has been at `ref_count = 0` past a 24-hour
grace window, removing both the R2 bytes and the row. That single rule reclaims
both never-attached uploads and the old blobs left behind when an avatar changes.

---

## media

Attachments on posts. Images, video. The bytes live in R2 behind a
`media_objects` row; this table holds the served URL and per-attachment metadata.

```sql
media
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  post_id         uuid          REFERENCES posts(id) ON DELETE CASCADE
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  object_id       uuid          REFERENCES media_objects(id) ON DELETE SET NULL
  url             text          NOT NULL
  mime_type       text          NOT NULL
  width           integer
  height          integer
  size_bytes      integer
  alt_text        text
  created_at      timestamptz   DEFAULT now()
```

`object_id` links the physical blob so deleting the post can drop its refcount;
it's null only for legacy rows that predate R2 storage.

---

## discord_profiles

Per-Discord-account avatar cache, keyed by the Discord snowflake so there's
exactly one row per account. When a Discord message is shared (or an account is
linked), the author's avatar is fetched into a `media_objects` blob and pointed
at here. When their Discord `avatar_hash` changes, the new image is ingested and
the old object's refcount is released, so there are never duplicate account
photos and stale ones are reclaimed.

```sql
discord_profiles
  discord_user_id   text          PRIMARY KEY        -- Discord snowflake
  avatar_hash       text                             -- Discord's hash; null = default avatar
  object_id         uuid          REFERENCES media_objects(id) ON DELETE SET NULL
  username          text
  global_name       text
  updated_at        timestamptz   DEFAULT now()
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
  id                        uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  participant_a             uuid          REFERENCES users(id) ON DELETE CASCADE
  participant_b             uuid          REFERENCES users(id) ON DELETE CASCADE
  last_message_at           timestamptz   DEFAULT now()  -- drives inbox sort order
  created_at                timestamptz   DEFAULT now()
  participant_a_cleared_at  timestamptz                  -- per-user clear cutoff
  participant_b_cleared_at  timestamptz
  participant_a_deleted_at  timestamptz                  -- per-user soft delete
  participant_b_deleted_at  timestamptz
  status                    text          DEFAULT 'active'  -- 'active' | 'request'
  requested_by              uuid          REFERENCES users(id) ON DELETE SET NULL
  UNIQUE (participant_a, participant_b)
```

`status` is `'active'` for normal two-way conversations and `'request'` when the initiator sent one message and is waiting for the recipient to accept. `requested_by` identifies who sent the request; it is cleared to null when the conversation becomes active. The recipient accepts via `POST /messages/:username/accept` or declines via `DELETE /messages/:username`.

---

## messages

Individual entries within a conversation. `body` is either plaintext (legacy),
AES-256-GCM server-encrypted ciphertext, or a `v3:` multi-device E2EE payload.
Plaintext is never written to the database in either encryption path.

`kind` distinguishes real messages from system events. Screenshot entries have
an empty body and are excluded from unread counts and inbox last-message previews.

```sql
messages
  id                uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  conversation_id   uuid          REFERENCES conversations(id) ON DELETE CASCADE
  sender_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  body              text          NOT NULL  -- plaintext, AES ciphertext, or v3: E2EE payload
  kind              text          NOT NULL DEFAULT 'message'
    -- 'message' | 'screenshot' | 'cleared' | 'deleted'
    -- 'tunnel_started' | 'tunnel_ended' (Tunnel Talk session markers)
  tunnel_session_id uuid          REFERENCES tunnel_sessions(id) ON DELETE SET NULL
    -- non-null only for tunnel_started and tunnel_ended kinds; links the marker to its session
  read              boolean       DEFAULT false  -- flipped when the recipient views the thread
  created_at        timestamptz   DEFAULT now()
```

---

## tunnel_sessions

One row per Tunnel Talk invite or session. SDP and ICE candidates are never stored
here — they pass through the signaling Durable Object ephemerally and are
discarded once the WebRTC peer connection is established.

```sql
tunnel_sessions
  id                  uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  conversation_id     uuid          REFERENCES conversations(id) ON DELETE CASCADE
  initiator_id        uuid          REFERENCES users(id) ON DELETE SET NULL
  participant_id      uuid          REFERENCES users(id) ON DELETE SET NULL
  status              text          NOT NULL DEFAULT 'pending'
    -- 'pending' | 'active' | 'ended' | 'declined'
  initiator_consent   boolean       NOT NULL DEFAULT false
    -- whether the initiator opted in to transcript saving
  participant_consent boolean       NOT NULL DEFAULT false
  started_at          timestamptz              -- set when the WebRTC data channel opens
  ended_at            timestamptz              -- set when either peer ends the session
  created_at          timestamptz   DEFAULT now()
```

Session status flows: `pending` → `active` → `ended`, or `pending` → `declined`.
Invites expire after 60 seconds; the API sets `declined` when it detects expiry.

---

## tunnel_messages

Transcript rows uploaded after a session ends. Written only when both parties
consented (via `PUT /tunnel/:sessionId/consent`). Bodies are the same E2EE
ciphertext format as regular DMs — the server stores opaque bytes and cannot
decrypt them.

Revocation (`DELETE /tunnel/:sessionId/consent`) deletes all rows for the session
via the `ON DELETE CASCADE` on `tunnel_session_id`. This is the privacy guarantee:
revocation is atomic and complete.

```sql
tunnel_messages
  id                uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  tunnel_session_id uuid          REFERENCES tunnel_sessions(id) ON DELETE CASCADE
  sender_id         uuid          REFERENCES users(id) ON DELETE SET NULL
  body              text          NOT NULL  -- E2EE ciphertext, same format as messages.body
  sent_at           timestamptz   NOT NULL
    -- when the message was sent P2P, not when it was uploaded
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

`token` is encrypted at rest with AES-256-GCM (`v1:<iv>:<ciphertext>`), so a dump
can't be used to push to anyone's device; the APNs sender decrypts it just before
calling Apple. `token_index` is a keyed blind index (HMAC-SHA256 of the raw token)
that carries uniqueness and powers the upsert-on-reregister and delete-on-signout,
since the randomised ciphertext isn't queryable.

```sql
devices
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  platform        text          NOT NULL  -- 'ios'
  token           text          NOT NULL  -- encrypted: v1:<iv>:<ciphertext>
  token_index     text          NOT NULL UNIQUE  -- HMAC-SHA256 blind index, lookup key
  name            text          NULL      -- user-visible label; null for pre-name devices
  created_at      timestamptz   DEFAULT now()
  last_seen_at    timestamptz   DEFAULT now()
```

---

## web_push_subscriptions

Browser Web Push subscriptions, the web equivalent of `devices`. One row per
subscribed browser. Opt-in the same way: the user enables browser notifications
from Settings > Notifications, the browser hands over a `PushSubscription`, and
it's stored here.

`endpoint` (the push service URL that identifies the browser) is encrypted at
rest with AES-256-GCM, and `endpoint_index` is a keyed blind index over it that
carries uniqueness and powers the upsert-on-resubscribe and delete-on-unsubscribe.
`p256dh` and `auth` are the subscription's own public key and shared secret, the
RFC 8291 inputs the server needs to encrypt each payload; they're useless without
the matching endpoint, so they're stored as-is. Push payloads are sealed end to
end, so the push service only ever relays ciphertext.

```sql
web_push_subscriptions
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  endpoint        text          NOT NULL  -- encrypted: v1:<iv>:<ciphertext>
  endpoint_index  text          NOT NULL UNIQUE  -- HMAC-SHA256 blind index, lookup key
  p256dh          text          NOT NULL  -- base64url client public key (RFC 8291)
  auth            text          NOT NULL  -- base64url client auth secret (RFC 8291)
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

Flat JSON of CSS variables. Validated on write, never executed. `variables`
covers colours plus typography (`--font-design`, `--letter-spacing`), geometry
(`--radius`, `--density`), and surface treatment (`--surface-blur`,
`--surface-opacity`, `--surface-shadow`) — enough for glass and terminal looks.

```sql
themes
  id              uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  name            text          NOT NULL
  description     text
  variables       jsonb         NOT NULL  -- { "--color-bg": "#0d0d0d", "--surface-blur": "14px", ... }
  published       boolean       DEFAULT true
  official        boolean       DEFAULT false  -- Counter's curated catalog; set only by seed, never via API. Browse lists these first.
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
  platform        text          NOT NULL  -- 'github' | 'discord' | 'website' | 'bandcamp' | 'soundcloud' | 'letterboxd' | 'goodreads' | 'strava' | 'itch'
  platform_username text        NOT NULL
  platform_url    text
  verified        boolean       DEFAULT false
  displayed       boolean       DEFAULT true  -- user toggle: show/hide badge on public profile
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

## saved_themes

A user's library of themes saved from other people's galleries (their own authored themes are found via `themes.user_id`). Membership only: this records *which* themes are in a user's library, not which one is applied. Applying a theme stays on-device.

```sql
saved_themes
  user_id         uuid          REFERENCES users(id) ON DELETE CASCADE
  theme_id        uuid          REFERENCES themes(id) ON DELETE CASCADE
  created_at      timestamptz   DEFAULT now()
  PRIMARY KEY (user_id, theme_id)  -- a theme is saved at most once per user
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

## oauth_accounts

Linked OAuth credentials, one row per (user, provider) pair. Access and refresh tokens, and the provider email, are stored encrypted with AES-256-GCM using the same key as message bodies. `provider_email` is display-only and never looked up by, so it needs no blind index; it's decrypted in the route just before it's returned.

```sql
oauth_accounts
  id                uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  user_id           uuid          NOT NULL REFERENCES users(id) ON DELETE CASCADE
  provider          text          NOT NULL  -- 'github' | 'discord'
  provider_user_id  text          NOT NULL  -- numeric ID on the provider
  provider_username text                    -- login/handle, for display
  provider_email    text                    -- encrypted: v1:<iv>:<ciphertext>
  access_token      text          NOT NULL  -- encrypted: v1:<iv>:<ciphertext>
  refresh_token     text                    -- encrypted; null for providers that don't issue one
  created_at        timestamptz   DEFAULT now()
  updated_at        timestamptz   DEFAULT now()
  UNIQUE (provider, provider_user_id)
  UNIQUE (user_id, provider)
```

---

## oauth_states

Short-lived CSRF state tokens for in-flight OAuth flows. One row per pending redirect; consumed (deleted) in the callback.

```sql
oauth_states
  id          uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  state_hash  text          NOT NULL UNIQUE  -- SHA-256 of the random state param
  provider    text          NOT NULL
  action      text          NOT NULL  -- 'login' | 'connect'
  user_id     uuid          REFERENCES users(id) ON DELETE CASCADE  -- null for login flows
  expires_at  timestamptz   NOT NULL  -- 10 minutes after creation
  created_at  timestamptz   DEFAULT now()
```

---

## oauth_session_codes

One-time codes issued after a successful OAuth login callback. The client exchanges the code via `POST /auth/session/exchange` to receive a JWT pair. Tokens never travel through a redirect URL.

```sql
oauth_session_codes
  id          uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  code_hash   text          NOT NULL UNIQUE  -- SHA-256 of the raw code
  user_id     uuid          NOT NULL REFERENCES users(id) ON DELETE CASCADE
  expires_at  timestamptz   NOT NULL  -- 5 minutes after creation
  created_at  timestamptz   DEFAULT now()
```

---

## discord_bot_subscriptions

Thing Two Discord bot opt-in notification subscription. One row per user who has ever interacted with the toggle. Absence means the user has never enabled it and DM delivery is skipped. `in_guild` is a cached result of the Discord bot guild membership check.

```sql
discord_bot_subscriptions
  user_id          uuid          PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE
  enabled          boolean       DEFAULT false NOT NULL
  in_guild         boolean       DEFAULT false NOT NULL   -- cached from the bot guild API check
  guild_checked_at timestamptz                            -- when guild membership was last verified
  created_at       timestamptz   DEFAULT now()
  updated_at       timestamptz   DEFAULT now()
```

---

## groups

Permission groups for the admin system. A group carries a subset of the fixed permission catalogue (defined in `@counter/config`) in its `permissions` jsonb array. A user's effective permissions are the union across every group they belong to. `is_system` groups (`admin`, `moderator`) can be renamed and re-permissioned but never deleted, so a site can't lock itself out of administration. The `admin` group always resolves to the full permission set regardless of what's stored.

```sql
groups
  id           uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  slug         text          UNIQUE NOT NULL   -- lowercase letters/digits/hyphen
  name         text          NOT NULL
  description  text
  permissions  jsonb         NOT NULL          -- array of permission keys, e.g. ["users.ban","posts.moderate"]
  color        text                            -- badge colour token; null falls back to a default
  is_system    boolean       DEFAULT false NOT NULL  -- protected from deletion
  created_at   timestamptz   DEFAULT now()
  updated_at   timestamptz   DEFAULT now()
```

---

## user_groups

Group memberships. Many-to-many: a user can hold several groups and a group has many members. `assigned_by` records which admin granted the membership for the audit trail; it set-nulls if that admin's account is later deleted.

```sql
user_groups
  user_id      uuid          REFERENCES users(id)  ON DELETE CASCADE
  group_id     uuid          REFERENCES groups(id) ON DELETE CASCADE
  assigned_by  uuid          REFERENCES users(id)  ON DELETE SET NULL
  assigned_at  timestamptz   DEFAULT now()
  PRIMARY KEY (user_id, group_id)
```

---

## admin_audit_log

Append-only record of every privileged action an admin takes. Never updated or deleted in normal operation, so it's the trustworthy answer to "who did what". `actor_id` set-nulls (rather than cascades) so the log outlives the admin account that wrote it; `target_id` is plain text because the thing it points at (a post, a user, a group) may itself be gone by the time the entry is read.

```sql
admin_audit_log
  id           uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  actor_id     uuid          REFERENCES users(id) ON DELETE SET NULL
  action       text          NOT NULL          -- e.g. 'user.ban', 'group.update', 'post.remove'
  target_type  text                            -- 'user' | 'post' | 'group' | 'report'
  target_id    text                            -- id of the affected row (text: it may be deleted)
  summary      text          NOT NULL          -- one-line human description, rendered in the log
  metadata     jsonb                           -- optional structured before/after or context
  created_at   timestamptz   DEFAULT now()
```

---

## reports

User-submitted reports about a post or another user, feeding the moderation queue. `reporter_id` set-nulls so a report survives the reporter deleting their account; `resolved_by` records which moderator closed it.

```sql
reports
  id           uuid          PRIMARY KEY DEFAULT gen_random_uuid()
  reporter_id  uuid          REFERENCES users(id) ON DELETE SET NULL
  target_type  text          NOT NULL          -- 'post' | 'user'
  target_id    uuid          NOT NULL          -- id of the reported post or user
  reason       text          NOT NULL          -- 'spam' | 'harassment' | 'hate' | 'violence' | 'illegal' | 'other'
  detail       text                            -- optional free-text from the reporter
  status       text          DEFAULT 'open' NOT NULL  -- 'open' | 'resolved' | 'dismissed'
  resolved_by  uuid          REFERENCES users(id) ON DELETE SET NULL
  resolved_at  timestamptz
  created_at   timestamptz   DEFAULT now()
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
| `media_objects` | Physical blobs in R2 (dedup + refcount) | Kept while referenced. An hourly sweep deletes any object unreferenced for over 24 hours, removing the R2 bytes and the row. |
| `discord_profiles` | Cached Discord avatar per shared/linked account | Updated when a fresh avatar is ingested; the prior blob's refcount is released so it can be swept. |
| `follows`, `likes`, `reposts` | The social graph and interactions | Life of the account. Cascade-deleted with it. |
| `tags`, `post_tags` | Topic discovery | Tag rows are shared and persist; the link to your post is removed when the post is. |
| `conversations`, `messages` | Private direct messages | Life of the account. Cascade-deleted with it. Screenshot events are stored as message rows with `kind = 'screenshot'` and are covered by the same retention. |
| `notifications` | Telling you what happened | Life of the account. Cascade-deleted with it. |
| `notification_preferences` | Which notification types you've muted | Life of the account. Cascade-deleted with it. |
| `device_keys` | E2EE public key per registered device | Life of the account, or until the device re-registers with a new key (upsert replaces the old row). Cascade-deleted with the account. The private key is never stored here. |
| `webauthn_credentials` | Registered passkeys (public key + signature counter) | Life of the account, or until you remove the passkey from settings. Cascade-deleted with the account. No private key or biometric data is stored. |
| `webauthn_challenges` | Single-use passkey ceremony nonces | 5 minutes. Consumed (deleted) when the ceremony finishes; expired rows are harmless nonces. |
| `devices` | Push delivery address (opaque APNs token) | Life of the account, or until you sign out on that device. Cascade-deleted with the account. No device or location data. |
| `themes`, `profile_themes`, `saved_themes` | Themes you published, applied, or saved to your library | Life of the account. Cascade-deleted with it. |
| `integrations` | Public links to other platforms | Life of the account, or until you unlink. |
| `oauth_accounts` | Linked OAuth credentials (GitHub, Discord) | Life of the account, or until you disconnect the provider. Encrypted tokens are deleted with the row. |
| `oauth_states` | CSRF state for in-flight OAuth flows | 10 minutes. Consumed (deleted) in the callback; expired rows are cleaned up on next use. |
| `oauth_session_codes` | One-time login codes issued after OAuth callback | 5 minutes. Consumed (deleted) on exchange; expired rows are cleaned up on next use. |
| `post_views` | Anonymous aggregate view counts | Retained indefinitely as counts only. Never tied to a person, so nothing identifying survives. |
| `algorithm_changelog` | Public record of ranking changes | Retained permanently. Contains no user data. |

If a category is not listed here, it is not collected. This table is updated
within 7 days of any change to data collection, as the CSL requires.

---

## Data Deletion

When a user deletes their account:

- `users` row is hard deleted
- All `posts`, `likes`, `reposts`, `follows`, `notifications`, `notification_preferences`, `devices`, `sessions`, `password_resets`, `integrations`, `themes`, `saved_themes` cascade delete
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
