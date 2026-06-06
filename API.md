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
| POST | `/auth/password-reset/request` | Request a reset link by email (no auth). Body: `{ email }`. Always returns `{ ok: true }`, whether or not the address matches an account. |
| POST | `/auth/password-reset/confirm` | Set a new password with a reset token (no auth). Body: `{ token, password }`. Logs the account out everywhere on success. |
| POST | `/auth/password` | Set or change the signed-in user's password (authenticated). Body: `{ currentPassword?, newPassword }`. `currentPassword` is required only if the account already has a password; OAuth-only accounts setting their first password omit it. Keeps all other sessions signed in. |
| DELETE | `/auth/account` | Permanently delete account and all data |
| GET | `/auth/keys` | List all device keys registered for the authenticated account |
| POST | `/auth/keys` | Register or upsert the E2EE key for one device (body: `{ deviceId, publicKey }`) |

Email verification is optional. It earns the ✦ verified badge and gates nothing.

### Passkeys (WebAuthn)

Passwordless, phishing-resistant sign-in. Registration runs under an authenticated session; authentication is public and discoverable (usernameless). The ceremony JSON is the standard `@simplewebauthn` shape.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/passkeys/register/options` | Begin enrolling a passkey (authenticated). Returns `PublicKeyCredentialCreationOptionsJSON`. |
| POST | `/auth/passkeys/register/verify` | Finish enrolling (authenticated). Body: `{ response, nickname? }`. |
| POST | `/auth/passkeys/authenticate/options` | Begin a passwordless login (no auth). Returns `PublicKeyCredentialRequestOptionsJSON`. |
| POST | `/auth/passkeys/authenticate/verify` | Finish login (no auth). Body: `{ response }`. Returns an `AuthResponse`. |
| GET | `/auth/passkeys` | List the caller's passkeys (authenticated). Returns `PasskeySummary[]`. |
| PATCH | `/auth/passkeys/:id` | Rename a passkey (authenticated). Body: `{ nickname }`. |
| DELETE | `/auth/passkeys/:id` | Remove a passkey (authenticated). |

Only the credential public key and signature counter are stored. A counter that fails to advance (cloned-authenticator signal) fails verification. Challenges are short-lived single-use nonces.

---

## OAuth

Connect GitHub or Discord to a Counter account for profile verification and future deep integrations. The same callback routes handle both flows:

- **login** — anonymous user; finds or creates a Counter account, issues a one-time session code, redirects to the web login callback.
- **connect** — authenticated user; links the provider to their existing account and auto-verifies the matching profile integration badge.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/auth/github` | Begin GitHub login/signup (anonymous) |
| GET | `/auth/github/connect` | Link GitHub to the current account (authenticated) |
| GET | `/auth/github/callback` | GitHub OAuth callback (handled by the API; redirects to web) |
| GET | `/auth/discord` | Begin Discord login/signup (anonymous) |
| GET | `/auth/discord/connect` | Link Discord to the current account (authenticated) |
| GET | `/auth/discord/callback` | Discord OAuth callback (handled by the API; redirects to web) |
| POST | `/auth/session/exchange` | Exchange a one-time session code for a JWT pair (body: `{ code }`) |
| POST | `/auth/:provider/connect/prepare` | Get the provider auth URL for mobile linking (body: `{ mobile: true }`; authenticated). Returns `{ authUrl }`. |
| DELETE | `/auth/github/disconnect` | Unlink GitHub (authenticated) |
| DELETE | `/auth/discord/disconnect` | Unlink Discord (authenticated) |
| GET | `/auth/:provider/me` | Connected account info for `github` or `discord` (authenticated) |

After a successful login callback the provider redirects to `{WEB_URL}/auth/callback?provider=github&code=SESSION_CODE`. The client POSTs that code to `/auth/session/exchange` to receive an `AuthResponse`. Tokens never appear in a URL.

Connecting a provider also auto-verifies the matching `/integrations` profile badge.

---

## Users

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/users/:username` | Get public profile |
| GET | `/users/me` | Get own profile (authenticated); includes `presenceSettings` |
| PATCH | `/users/me` | Update own profile. Set the avatar with `avatarObjectId` (from a prior `POST /media`), or `null` to clear it; omit the field to leave it unchanged. |
| GET | `/users/me/presence` | Get own presence and messaging-privacy settings |
| PUT | `/users/me/presence` | Update presence/privacy settings (`onlineStatusEnabled`, `onlineStatusVisibility`, `lastSeenEnabled`, `lastSeenVisibility`, `heartbeatIntervalSeconds`, `messagingPrivacy`, `typingIndicatorsEnabled`) |
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

Feed responses embed `topReplies` on each post: an array of up to two oldest direct replies, pre-serialized as full `Post` objects. Omitted when the post has no replies. Reply posts themselves never carry `topReplies`.
| POST | `/posts` | Create a post (set `repostOf` for a quote-repost). Attach photos with `media: [{ objectId, altText? }]` (up to 4), where `objectId` comes from a prior `POST /media`. |
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

**Bot replies.** A server-designated bot account (a user with `bot_kind` set) replies in two cases: when a post or reply @mentions it, and when a new reply lands in a thread the bot is already part of (so it keeps talking without being re-tagged). The reply is authored by the bot and threaded under the post it answers, generated in the background so it doesn't delay the response. Guards: a post authored by a bot never triggers further bot replies (no loops), and a bot answers any given post at most once (no duplicates).

---

## Media

Uploads go through the API, which validates the bytes and stores them in R2.
Objects are content-addressed (keyed by sha256), so identical uploads dedup to
one stored blob. The returned object id is what you attach to a post (`media[].objectId`)
or set as your avatar (`avatarObjectId` on `PATCH /users/me`).

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/media` | Upload an image (authenticated). Accepts a raw image body or a `multipart/form-data` `file` field. Validates format by magic bytes (jpeg/png/webp/gif), enforces the size and dimension caps, and stores it. Returns `{ id, url, mimeType, width, height, sizeBytes }`. |

The stored object starts unreferenced; an hourly garbage-collection sweep
deletes any object that has sat unreferenced past a grace window, so an upload
that's never attached to a post or avatar is reclaimed automatically.

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
| GET | `/messages` | Inbox: all conversations with last-message preview, unread counts, and `status`/`isInboundRequest` fields for request routing |
| GET | `/messages/:username` | Conversation thread with a user |
| GET | `/messages/:username/info` | Conversation status (`active`, `request`, or `null`) and whether this is an inbound request for the viewer |
| GET | `/messages/:username/live` | WebSocket upgrade to the conversation's live channel (the `ConversationHub` Durable Object). Pushes new messages, relays typing, and reports in-thread presence. Accepts `Authorization: Bearer` or `?token=` query param (browsers can't set headers on a WebSocket). Server-to-client frames: `message`, `presence`, `presence_state`, `tunnel_invite`; client-to-server: `typing` (dropped when the sender has typing indicators off). Returns a validation error when Durable Objects aren't bound (e.g. the Bun dev server). |
| POST | `/messages/:username` | Send a message. On first contact, checks recipient's `messagingPrivacy`; creates a request conversation when the sender isn't allowed to message directly. Blocked if the conversation is a pending request. Bot accounts (`bot_kind` set) reject all DMs with 403, regardless of `messagingPrivacy`. |
| POST | `/messages/:username/accept` | Accept an inbound message request; switches the conversation to `active`. Only callable by the recipient. |
| POST | `/messages/:username/read` | Mark all messages from a user as read |
| POST | `/messages/:username/screenshot` | Record a screenshot event in the transcript (conversation must exist) |
| DELETE | `/messages/:username/messages` | Clear the caller's view: stamps a per-user `clearedAt`; messages before that point are hidden for them only. Inserts a `kind: 'cleared'` event the partner sees. Returns the event as a `DirectMessage`. |
| DELETE | `/messages/:username` | Delete the conversation for the caller only: stamps a per-user `deletedAt` so it disappears from their inbox. The partner still sees the full thread. Inserts a `kind: 'deleted'` event. Used by the recipient to decline a request. |

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
| GET | `/notifications/badges` | Unread counts for the nav badges: `{ notifications, messages }`. Notifications excludes message-type so direct messages count only once. Seeds the badges on load; the live socket keeps them current. |
| GET | `/notifications/live` | WebSocket upgrade to the user's NotificationHub Durable Object. Pushes each new notification live (`{ type: 'notification', notification }`) so badges and lists update without a reload. Accepts `Authorization: Bearer` or `?token=` query param. One-way (server to client). Validation error when Durable Objects aren't bound (Bun dev server). |
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

`/devices` is for native (iOS) push tokens. Browser Web Push uses `/web-push`.

---

## Web Push

Browser push subscriptions, the web counterpart to `/devices`. Opt-in from the
notification settings: the browser creates a `PushSubscription`, the page posts
it here, and from then on notifications are delivered as Web Push. Payloads are
sealed with RFC 8291 (`aes128gcm`) so the push service only relays ciphertext,
and the visible text is type-only (no sender, no content). The endpoint is
encrypted at rest with a blind index, the same as native device tokens.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/web-push/vapid-public-key` | The base64url VAPID public key the browser needs to subscribe; `{ key }` is null when web push isn't configured |
| POST | `/web-push/subscribe` | Register/upsert a subscription (`{ endpoint, keys: { p256dh, auth } }`); returns `{ ok, id }` |
| DELETE | `/web-push/subscribe` | Remove a subscription (`{ endpoint }`), on unsubscribe or sign-out |

---

## Insights

Available from post one. No follower gate. Ever.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/insights/posts/:id` | Insights for a single post |
| GET | `/insights/profile` | Aggregate profile insights |
| GET | `/insights/public` | Public aggregate platform stats |

---

## Discord Bot (Thing Two)

Thing Two is Counter's Discord bot. Users who have connected their Discord account and are members of the Counter Discord server can opt in to receive their notifications as Discord DMs and to post to Counter directly from Discord. Both are off by default.

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/discord-bot/settings` | Required | Get the caller's current bot subscription state (`{ enabled, inGuild, guildCheckedAt, postingEnabled }`) |
| PUT | `/discord-bot/settings` | Required | Update settings; body: `{ enabled: boolean, postingEnabled?: boolean }`. Enabling `enabled` requires a connected Discord account and Counter server membership (400 if no Discord linked, 403 if not in guild). `postingEnabled` can be toggled independently of `enabled`. |
| POST | `/discord-bot/interactions` | Discord Ed25519 | Discord interactions webhook. Handles PING (type 1), `/post`, `/interact`, and `/ask` slash commands, and "Share to Counter" message context menu. Requests are authenticated by Ed25519 signature (verified against `DISCORD_PUBLIC_KEY`), not by Counter JWT. Register this URL as the Interactions Endpoint URL in the Discord developer portal. |

**Discord commands**

| Command | Type | Description |
|---------|------|-------------|
| `/post <content>` | Slash command | Publishes text as a Counter post on behalf of the invoker. Requires Discord linked + `postingEnabled`. |
| `/interact coinflip` | Slash command | Flips a coin and posts a public message tagging the invoker (`@user flipped a coin: heads`). No Counter account, link, or opt-in required. |
| `/interact dice [sides]` | Slash command | Rolls a die (default 6 sides, clamped 2-1000) and posts the result publicly, tagging the invoker. No Counter account, link, or opt-in required. |
| `/interact 8ball <question>` | Slash command | Answers a yes/no question with a classic Magic 8-Ball reply, posted publicly with the question and invoker tag. No Counter account, link, or opt-in required. |
| `/ask <prompt>` | Slash command | Sends the prompt to an OpenAI-compatible chat endpoint (`OPENAI_BASE_URL` + `OPENAI_MODEL`, authenticated with either a static `OPENAI_API_KEY` or a Google service account `GOOGLE_SA_CLIENT_EMAIL`/`GOOGLE_SA_PRIVATE_KEY` for Vertex AI) and posts the model's reply publicly. Uses a deferred response (Discord shows "thinking" until the answer arrives). No Counter account, link, or opt-in required; replies politely decline if chat isn't configured. |
| Share to Counter | Message context menu | Quotes the selected Discord message with attribution and posts it to Counter. If the original author has a linked Counter account, their `@handle` appears in the attribution; otherwise their Discord display name is used. Requires Discord linked + `postingEnabled`. |

`/post` and "Share to Counter" respond ephemerally (only visible to the invoker). `/interact` subcommands and `/ask` reply publicly so the whole channel sees the result.

**Setup**

1. Set `DISCORD_APP_ID` and `DISCORD_PUBLIC_KEY` in `.dev.vars` / wrangler secrets.
2. Register the Interactions Endpoint URL in the Discord developer portal.
3. Run `bun run apps/api/scripts/register-discord-commands.ts` once to register the commands (idempotent; safe to re-run).

---

## GitHub Webhook (Thing Five)

Thing Five announces pushes in Discord. A GitHub org webhook points at the endpoint below; verified default-branch pushes get a message in Five's voice (a short model-written quip, then a commit summary) posted to a fixed channel as Five.

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/github/webhook` | GitHub HMAC | GitHub webhook receiver. Verifies the `x-hub-signature-256` HMAC-SHA256 against `GITHUB_WEBHOOK_SECRET`, then on a `push` event to the repo's default branch (owner in `GITHUB_COMMIT_ORGS`) posts an announcement to `DISCORD_COMMIT_CHANNEL_ID` as Thing Five via `THING_FIVE_BOT_TOKEN`. Returns 501 if no secret is configured, 401 on a bad signature, and 2xx (ignored) for `ping`, non-`push` events, and pushes that don't qualify. |

**Setup**

1. Set `GITHUB_WEBHOOK_SECRET`, `THING_FIVE_BOT_TOKEN`, and `DISCORD_COMMIT_CHANNEL_ID` in `.dev.vars` / wrangler secrets (optionally `GITHUB_COMMIT_ORGS`, default `anti-ltd,counter-ltd`). Deploy Five's persona with `scripts/deploy-ask-prompt.sh` so the quip has a voice.
2. Add an org webhook in GitHub (Settings → Webhooks) for each org: payload URL `https://api.counter.ltd/github/webhook`, content type `application/json`, the same secret, and the "Pushes" event only.

---

## Integrations

Linked external accounts. All optional, linked at user discretion. Ownership is
proven with a `rel="me"` link-back; a verified link becomes a display-only trust
badge that gates nothing.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/integrations/me` | The caller's own links, verified or not (authenticated) |
| GET | `/integrations/:username` | A user's verified, displayed links (public) |
| POST | `/integrations` | Link a platform; `platform` and `url` in the body |
| POST | `/integrations/:id/verify` | Re-check the `rel="me"` link-back and set verified state |
| PATCH | `/integrations/:id` | Toggle `displayed` (show/hide badge on profile) |
| DELETE | `/integrations/:id` | Remove a link |

Supported platforms: `github` `discord` `website` `bandcamp` `soundcloud` `letterboxd` `goodreads` `strava` `itch`

The `Integration` response shape includes `id`, `platform`, `username`, `url`, `verified`, and `displayed`. The public `GET /:username` endpoint only returns rows where both `verified` and `displayed` are true.

---

## Themes

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/themes` | Get public themes (browsable, no auth) |
| GET | `/themes/library` | Your library: themes you created plus ones you saved (auth) |
| GET | `/themes/:id` | Get a single theme |
| POST | `/themes` | Create a theme (`published: false` for a private draft) |
| PATCH | `/themes/:id` | Edit own theme (partial: name, description, variables, published) |
| DELETE | `/themes/:id` | Delete own theme |
| POST | `/themes/:id/save` | Save a theme to your library (auth) |
| DELETE | `/themes/:id/save` | Remove a theme from your library (auth) |

Themes are flat JSON objects of CSS variables. Validated server-side for structure, never executed. The `variables` map spans more than colours: typography (`--font-design`, `--font`, `--letter-spacing`), geometry (`--radius`, `--density`), and surface treatment (`--surface-blur`, `--surface-opacity`, `--surface-shadow`). The API stores whatever well-formed `--*` tokens it's given; the clients decide which they consume. Each `Theme` carries an `official` flag (Counter's curated catalog, set only by the seed, never via the API); `GET /themes` lists official themes first. `/themes/library` returns `{ created, saved }`: themes you authored (drafts included) and published themes you saved from others. Saving is library membership only; which theme is *applied* stays on-device and is never sent to the server.

---

## Algorithm

The algorithm is open source. This endpoint exposes its current state publicly.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/algorithm` | Current ranking weights and parameters |
| GET | `/algorithm/changelog` | Full public changelog of every change |

---

## Tunnel Talk

Peer-to-peer ephemeral chat sessions. The server handles the invite lifecycle
and relays WebRTC signaling (SDP offer/answer, ICE candidates) during connection
setup. Once the WebRTC data channel opens, all message content flows directly
between devices and never reaches the server. All endpoints require authentication.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/tunnel/turn-credentials` | Short-lived TURN/STUN credentials for WebRTC NAT traversal. Falls back to public STUN when TURN keys aren't configured. |
| POST | `/tunnel/:username/invite` | Create a Tunnel Talk invite. Fails if the partner is offline, no active conversation exists, or either user already has an active session. Returns `{ sessionId }`. Also pushes the invite to the recipient's open conversation socket (a `tunnel_invite` `LiveSignal`) so their join banner appears live. |
| GET | `/tunnel/:username/pending` | Check for a pending incoming invite from the given user. Fetched once when the conversation view opens (to catch a pre-existing invite); live invites arrive over the conversation socket, so this is no longer polled. Auto-expires invites older than 60 seconds and returns `{ pending: false }`. Returns `{ pending: true, session: TunnelSession }` when one exists. |
| POST | `/tunnel/:sessionId/accept` | Accept a pending invite (recipient only). Sets status to `active`. |
| POST | `/tunnel/:sessionId/decline` | Decline a pending invite (recipient only). Sets status to `declined`. |
| DELETE | `/tunnel/:sessionId` | End an active or pending session (either party). Idempotent. |
| PUT | `/tunnel/:sessionId/consent` | Opt in to transcript saving for this session. The client starts buffering messages locally; transcript is only uploaded after the session ends. |
| DELETE | `/tunnel/:sessionId/consent` | Revoke transcript consent. Deletes all saved `tunnel_messages` for the session atomically. |
| POST | `/tunnel/:sessionId/transcript` | Upload a batch of E2EE message bodies after the session ends. Only callable by a consenting party after status is `ended`. Body: `{ messages: [{ body, sentAt }] }`. |
| GET | `/tunnel/:sessionId/signal` | WebSocket upgrade to the signaling Durable Object. Accepts `Authorization: Bearer` header or `?token=` query param. Allowed for `pending` and `active` sessions (the initiator connects while pending so they're on-channel when the participant accepts); rejected for `ended`/`declined`. |

**Session status lifecycle:** `pending` → `active` → `ended` (or `pending` → `declined`).

**Signaling protocol** (relayed through the DO, never stored):

| Message | Direction | Meaning |
|---------|-----------|---------|
| `{ type: 'offer', sdp }` | initiator → DO → participant | SDP offer |
| `{ type: 'answer', sdp }` | participant → DO → initiator | SDP answer |
| `{ type: 'ice', candidate }` | either → DO → other | Trickle ICE candidate |
| `{ type: 'peer_joined' }` | DO → client | The remote peer connected to signaling |
| `{ type: 'peer_left' }` | DO → client | The remote peer disconnected |

**Data channel protocol** (P2P only, never reaches the server):

| Message | Meaning |
|---------|---------|
| `{ type: 'message', body, tempId }` | E2EE ciphertext message |
| `{ type: 'delivered', tempId }` | Delivery acknowledgement |
| `{ type: 'consent', value }` | Transcript consent state change |
| `{ type: 'end' }` | Session end signal |

**Thread markers.** `GET /messages/:username` includes two new `kind` values for `DirectMessage`:
- `tunnel_started` — inserted when the invite is created; links to the session via `tunnelSessionId`
- `tunnel_ended` — inserted when the session ends or is declined

The response also includes a `tunnelSessions` map (`Record<string, TunnelSessionWithTranscript>`) so clients can render inline transcripts between the markers without a separate fetch.

---

## Link Preview

Server-side OG/meta proxy. Runs inside the Worker so the client avoids CORS
and the user's IP never reaches the target site. Auth is required to prevent
unauthenticated use of the Worker as an open proxy.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/preview?url=<encoded>` | Fetch OG preview data for a URL. Returns `{ url, title, description, image, siteName }`. 404 when metadata can't be fetched. Private/loopback hosts are rejected. |

---

## Reports

Any signed-in user can flag a post or another account for moderator review. A
repeat report from the same person on the same target collapses into the existing
open one rather than creating a duplicate.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/reports` | File a report. Body: `{ targetType: 'post' \| 'user', targetId, reason, detail? }`. `reason` is one of `spam`, `harassment`, `hate`, `violence`, `illegal`, `other`. Returns `{ ok, id }` (201), or `{ ok, id, duplicate: true }` when an open report already exists. |

---

## Admin

The control panel. Every endpoint requires authentication **and** a specific
permission, resolved as the union of the caller's group memberships. Missing the
permission returns 403. Permissions are a fixed catalogue (`dashboard.view`,
`users.view`, `users.manage_groups`, `users.ban`, `users.suspend`,
`users.reset_password`, `groups.view`,
`groups.manage`, `posts.moderate`, `posts.nuke`, `reports.view`,
`reports.resolve`, `audit.view`). The `admin` system group always holds all of
them.

Every state-changing action writes an immutable `admin_audit_log` entry.

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/admin/dashboard` | `dashboard.view` | Site-wide counts: users by status, new-last-7d, posts (total/removed), open reports, group count. |
| GET | `/admin/permissions` | `groups.view` | The permission catalogue with display metadata, for the group editor. |
| GET | `/admin/users` | `users.view` | Paginated user list. Query: `q` (username/display name search), `status`, `after`, `limit`. |
| GET | `/admin/users/:id` | `users.view` | One user's moderation detail, group memberships, and content counts. |
| POST | `/admin/users/:id/groups` | `users.manage_groups` | Add the user to a group. Body: `{ groupId }`. Idempotent. |
| DELETE | `/admin/users/:id/groups/:groupId` | `users.manage_groups` | Remove the user from a group. Idempotent. |
| POST | `/admin/users/:id/ban` | `users.ban` | Ban indefinitely and revoke sessions. Body: `{ reason? }`. Cannot ban yourself. |
| POST | `/admin/users/:id/unban` | `users.ban` | Lift a ban, returning the account to `active`. |
| POST | `/admin/users/:id/suspend` | `users.suspend` | Suspend until a time and revoke sessions. Body: `{ until (ISO, future), reason? }`. Auto-lifts at next login once expired. Cannot suspend yourself. |
| POST | `/admin/users/:id/unsuspend` | `users.suspend` | End a suspension early. |
| POST | `/admin/users/:id/password-reset` | `users.reset_password` | Start a password reset. Body: `{ delivery: 'email' \| 'link' }`. `email` mails the user the link; `link` returns `{ link }` in the response to hand over directly. |
| GET | `/admin/groups` | `groups.view` | All groups with permission sets and member counts. |
| GET | `/admin/groups/:id` | `groups.view` | One group. |
| POST | `/admin/groups` | `groups.manage` | Create a group. Body: `{ slug, name, description?, color?, permissions[] }`. |
| PATCH | `/admin/groups/:id` | `groups.manage` | Edit name/description/color/permissions/slug. System groups cannot be renamed by slug. |
| DELETE | `/admin/groups/:id` | `groups.manage` | Delete a group. System groups are protected. |
| GET | `/admin/posts/:id` | `posts.moderate` | Fetch one post in full for review, including removed ones, with author. |
| DELETE | `/admin/posts/:id` | `posts.moderate` | Remove a post (soft-delete + `removedByAdmin`). |
| POST | `/admin/posts/:id/restore` | `posts.moderate` | Restore a moderator-removed post. |
| DELETE | `/admin/posts/:id/nuke` | `posts.nuke` | Permanently hard-delete a post and its entire reply/repost subtree (cascading media, likes, reposts, tags, views; notifications and reports about them are also cleared). No restore. Returns `{ ok, deleted }` (total rows erased). |
| GET | `/admin/reports` | `reports.view` | The moderation queue. Query: `status` (defaults to `open`), `after`, `limit`. |
| POST | `/admin/reports/:id/resolve` | `reports.resolve` | Close a report. Body: `{ status: 'resolved' \| 'dismissed' }`. |
| GET | `/admin/audit` | `audit.view` | The admin action log, newest first. Query: `after`, `limit`. |

**Moderation enforcement.** A banned or actively-suspended account is blocked at
login (and on token refresh, since banning revokes refresh tokens). A suspension
whose end time has passed auto-reactivates the account on the next login attempt.

---

## Notes

- All public content is readable without authentication
- Pagination uses cursor-based `?after=` and `?limit=` params
- Rate limits are reported in response headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, plus `Retry-After` on a 429
- All endpoints return consistent error shapes: `{ error: { code, message } }`
