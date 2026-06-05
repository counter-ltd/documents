# Counter API

The API is the product. Every client — web, iOS, macOS, third-party — talks to the same endpoints. No client holds special privilege over another.

**Base URL:** `https://api.counter.ltd`  
**Format:** JSON  
**Auth:** Bearer token (JWT)

---

## Authentication

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/register` | Create account, returns token pair |
| POST | `/auth/login` | Login, returns token pair |
| POST | `/auth/logout` | Revoke a refresh token |
| POST | `/auth/refresh` | Trade a refresh token for a fresh pair |
| POST | `/auth/verify` | Redeem an email verification token (no auth) |
| POST | `/auth/verify/request` | Resend the verification email (authenticated) |
| DELETE | `/auth/account` | Permanently delete account and all data |
| GET | `/auth/keys` | List all device keys registered for the authenticated account |
| POST | `/auth/keys` | Register or upsert the E2EE key for one device (body: `{ deviceId, publicKey }`) |

Email verification is optional. It earns the ✦ verified badge and gates nothing.

---

## Users

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/users/:username` | Get public profile |
| GET | `/users/me` | Get own profile (authenticated); includes `presenceSettings` |
| PATCH | `/users/me` | Update own profile |
| GET | `/users/me/presence` | Get own presence settings |
| PUT | `/users/me/presence` | Update presence settings (`onlineStatusEnabled`, `onlineStatusVisibility`, `lastSeenEnabled`, `lastSeenVisibility`, `heartbeatIntervalSeconds`) |
| POST | `/users/me/heartbeat` | Record activity; sets `lastSeenAt = now()` |
| GET | `/users/:username/posts` | Get posts by user (includes reposts) |
| GET | `/users/:username/followers` | Get followers list |
| GET | `/users/:username/following` | Get following list |
| POST | `/users/:username/follow` | Follow a user |
| DELETE | `/users/:username/follow` | Unfollow a user |
| GET | `/users/:username/public-key` | Get all E2EE device keys for a user (`{ keys: [{deviceId, publicKey}] }`) |

---

## Posts

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/posts` | Get following feed (authenticated) |
| GET | `/posts/public` | Get ranked public feed (no auth required) |
| POST | `/posts` | Create a post (set `repostOf` for a quote-repost) |
| GET | `/posts/:id` | Get a single post (no auth required) |
| PATCH | `/posts/:id` | Edit a post |
| DELETE | `/posts/:id` | Delete a post |
| POST | `/posts/:id/like` | Like a post |
| DELETE | `/posts/:id/like` | Unlike a post |
| GET | `/posts/:id/likes` | Get likes on a post |
| POST | `/posts/:id/repost` | Repost |
| DELETE | `/posts/:id/repost` | Undo repost |
| GET | `/posts/:id/thread` | Get full thread context |

---

## Replies

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/posts/:id/replies` | Reply to a post |
| GET | `/posts/:id/replies` | Get replies to a post |

---

## Search

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/search/posts` | Search posts |
| GET | `/search/users` | Search users |
| GET | `/search/tags` | Search hashtags |

---

## Hashtags

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/tags/trending` | Get trending tags |
| GET | `/tags/:tag` | Get posts with tag |

---

## Topics

Topics are communities. Anyone can create one and creators auto-join.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/topics` | List all topics, sorted by member count |
| POST | `/topics` | Create a topic (creator auto-joins) |
| GET | `/topics/:slug` | Get a single topic with counts and viewer state |
| GET | `/topics/:slug/posts` | Get posts in a topic |
| POST | `/topics/:slug/join` | Join a topic |
| DELETE | `/topics/:slug/join` | Leave a topic |

---

## Messages

Private direct messages between two users. All endpoints require authentication.
Messages are end-to-end encrypted (E2EE): the server stores ciphertext and cannot read bodies.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/messages` | Inbox: all conversations with last-message preview and unread counts |
| GET | `/messages/:username` | Conversation thread with a user |
| POST | `/messages/:username` | Send an E2EE message (body must be a `v3:` multi-device ciphertext) |
| POST | `/messages/:username/read` | Mark all messages from a user as read |
| POST | `/messages/:username/screenshot` | Record a screenshot event in the transcript (conversation must exist) |
| DELETE | `/messages/:username/messages` | Clear the caller's view: stamps a per-user `clearedAt`; messages before that point are hidden for them only. Inserts a `kind: 'cleared'` event the partner sees. Returns the event as a `DirectMessage`. |
| DELETE | `/messages/:username` | Delete the conversation for the caller only: stamps a per-user `deletedAt` so it disappears from their inbox. The partner still sees the full thread. Inserts a `kind: 'deleted'` event. |

**E2EE format.** New sends must use the `v3:` multi-device format. The body is
`v3:<base64-encoded JSON>` where the JSON is an array of per-device copies:
`[{ "d": "<deviceId>", "b": "<v2-envelope>" }, ...]`. Each `v2` envelope is
`v2:<SPKI-base64 ephemeral public key>:<base64 IV>:<base64 ciphertext+tag>`,
independently encrypted for that device's P-256 key. The sender includes copies
for every device registered by the recipient (via `GET /users/:username/public-key`)
and by themselves (via `GET /auth/keys`), so all devices can decrypt sent messages.

If the recipient has no registered device keys, the client sends plaintext and
the server encrypts it with AES-256-GCM (v1) before storing. Both parties see
a notice in the UI. If the recipient has device keys, the body must be `v3:`;
the server rejects anything else.

`DirectMessage` objects include an `encrypted: boolean` field. When `true`, the
`body` is raw ciphertext (`v2:` or `v3:`) and the client decrypts it locally by
finding its device copy. When `false`, the server returned plaintext (legacy
messages predating E2EE).

`DirectMessage` objects include a `kind` field: `'message'` for normal messages; `'screenshot'` when the viewer took a screenshot; `'cleared'` when a participant cleared their history (per-user); `'deleted'` when a participant deleted the conversation (per-user). System event entries (`screenshot`, `cleared`, `deleted`) have an empty `body` and are excluded from unread counts and inbox last-message previews.

---

## Notifications

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/notifications` | Get notifications |
| POST | `/notifications/read` | Mark all as read |
| PATCH | `/notifications/:id/read` | Mark one as read |
| GET | `/notifications/preferences` | Get per-type toggles (likes, reposts, replies, follows, mentions, messages) |
| PUT | `/notifications/preferences` | Update toggles; a type set to false stops it in the inbox and push |

---

## Devices

Push tokens for native clients. Registration is opt-in: the client never
auto-uploads a token. Users register and remove devices from the Privacy
settings panel. A muted notification type is never delivered to a device, so
these only push what the user asked for in their preferences.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/devices` | List all registered devices for the authenticated user (excludes raw token) |
| POST | `/devices` | Register a push token (`{ token, platform, name? }`); returns `{ ok, id }` |
| DELETE | `/devices/by-id/:id` | Remove a device by UUID (used from the Privacy panel) |
| DELETE | `/devices/:token` | Remove a device by raw token (used on sign-out) |

---

## Insights

Available from post one. No follower gate. Ever.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/insights/posts/:id` | Insights for a single post |
| GET | `/insights/profile` | Aggregate profile insights |
| GET | `/insights/public` | Public aggregate platform stats |

---

## Integrations

Linked external accounts. All optional, linked at user discretion. Ownership is
proven with a `rel="me"` link-back; a verified link becomes a display-only trust
badge that gates nothing.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/integrations/me` | The caller's own links, verified or not (authenticated) |
| GET | `/integrations/:username` | A user's verified links (public) |
| POST | `/integrations` | Link a platform; `platform` and `url` in the body |
| POST | `/integrations/:id/verify` | Re-check the `rel="me"` link-back and set verified state |
| DELETE | `/integrations/:id` | Remove a link |

Supported platforms: `github` `bandcamp` `soundcloud` `letterboxd` `goodreads` `strava` `itch`

---

## Themes

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/themes` | Get public themes (browsable, no auth) |
| GET | `/themes/:id` | Get a single theme |
| POST | `/themes` | Publish a theme |
| DELETE | `/themes/:id` | Delete own theme |

Themes are flat JSON objects of CSS variables. Validated server-side for structure, never executed.

---

## Algorithm

The algorithm is open source. This endpoint exposes its current state publicly.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/algorithm` | Current ranking weights and parameters |
| GET | `/algorithm/changelog` | Full public changelog of every change |

---

## Notes

- All public content is readable without authentication
- Pagination uses cursor-based `?after=` and `?limit=` params
- Rate limits are reported in response headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, plus `Retry-After` on a 429
- All endpoints return consistent error shapes: `{ error: { code, message } }`
