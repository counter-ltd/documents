# Changelog

All notable changes to Counter are documented here. This covers the platform as a whole — features, API changes, and infrastructure. Algorithm-specific changes are tracked separately at [/algorithm](/algorithm).

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [Unreleased]

### Added
- iOS: opening a conversation now shows a centered "Decrypting" state with an animated lock icon and progress bar while messages are being fetched and decrypted, replacing the blank black screen.
- Online status and last-seen indicators on profiles. Both features are off by default and must be explicitly enabled in Settings > Privacy > Online Status. Each has its own independent visibility setting (everyone, followers, or followers you follow back). The heartbeat interval is configurable from 60 to 3600 seconds.
- `GET /users/me/presence` — read your own presence settings.
- `PUT /users/me/presence` — update online-status enabled/visibility, last-seen enabled/visibility, and heartbeat interval.
- `POST /users/me/heartbeat` — record activity; sets the `last_seen_at` timestamp used for online detection and last-seen display.
- `GET /users/me` and any user profile now include a `presence` field (`isOnline`, `lastSeenAt`) when the viewer has permission to see it.
- Settings > Privacy > Devices panel on iOS and web: see all registered push devices, register the current device by name, and remove devices you no longer use.
- `GET /devices` — list all registered devices for the authenticated user (id, platform, name, last seen; raw token excluded).
- `DELETE /devices/by-id/:id` — remove a device by UUID, used from the Privacy panel.
- `POST /devices` now accepts an optional `name` field and returns `{ ok, id }` so the client can identify "this device" in the list.
- iOS: when you screenshot a conversation, a notice is added to the transcript for both parties ("You took a screenshot" / "@user took a screenshot"). Uses `UIApplication.userDidTakeScreenshotNotification` — no special permission needed.
- `POST /messages/:username/screenshot` — records a screenshot event in the conversation transcript. Returns the new `DirectMessage` entry with `kind: 'screenshot'`. Requires the conversation to already exist.
- `DirectMessage` now includes a `kind` field: `'message'` for normal messages, `'screenshot'` for transcript events. Screenshot entries are excluded from unread counts and inbox last-message previews.
- Clear and Delete actions on conversations (iOS: swipe left on inbox row or use the lock popover; web: lock popover). Both are per-user: Clear hides your message history from that point, leaving the other party's view intact. Delete removes the conversation from your inbox only. The other party sees a system notice ("@user cleared their history" / "@user deleted the conversation") in the thread.
- `DELETE /messages/:username/messages` — per-user clear: stamps `clearedAt` for the caller, inserts a `kind: 'cleared'` transcript event, returns it as a `DirectMessage`.
- `DELETE /messages/:username` — per-user delete: stamps `deletedAt` for the caller so the conversation disappears from their inbox; inserts a `kind: 'deleted'` transcript event for the partner.
- `conversations` table: `participant_a_cleared_at`, `participant_b_cleared_at`, `participant_a_deleted_at`, `participant_b_deleted_at` columns for per-user clear and delete state.

### Changed
- iOS: conversation send bar uses the iOS 26 `glassEffect` capsule on the text field. The container background is removed so the glass floats over the message list. Falls back to `ultraThinMaterial` on iOS 17-25.
- iOS push registration is now opt-in. The app no longer auto-uploads the APNs token on sign-in; the user registers their device explicitly from Settings > Privacy > Devices.
- iOS: tapping the encryption lock in a conversation now shows a popover instead of a system alert.
- iOS: encrypted message previews in the conversations list now show a lock glyph and "Encrypted Message" instead of raw ciphertext.
- Web: encrypted message previews in the conversations list now show a green SVG lock glyph and "Encrypted message" instead of the raw emoji.
- Web: conversation header now shows a lock icon with a popover matching the iOS encryption indicator — green for full E2EE (with a per-device access table), yellow for single-device E2EE, orange for server-encrypted fallback.

## [0.2.0] - 2026-06-05

### Added
- iOS: lock icon in the conversation toolbar shows the current encryption level — green for full E2EE, yellow for single-device E2EE, orange for server-encrypted fallback. Tap it for a plain-language explanation.

### Changed
- iOS: the compose FAB is now hidden on the Search, Messages, and Notifications tabs. It only appears on Home and Profile.

### Added
- Server-encrypted fallback for direct messages: if either party hasn't registered a device yet, messages are still encrypted in storage (AES-256-GCM, server-side) rather than blocked. Both users see a notice in the conversation explaining that messages are server-encrypted, not end-to-end encrypted. Plaintext is never written to the database in either path.
- End-to-end encrypted direct messages now support multiple devices. Each registered device gets its own copy of every message (v3 format: one per-device `v2:` envelope per entry, wrapped in a `v3:` payload). Sent messages are now readable on all your registered Counter sessions, not just the one you sent from. Open Counter on any device once to register it before sending, so that device is included in future message copies.
- When only one device is registered, a soft warning is shown in the compose area: "Encrypted for this device only. Open Counter on your other devices to register them before sending."
- `GET /auth/keys` — list all device keys registered for the authenticated account.
- `POST /auth/keys` now accepts `{ deviceId, publicKey }` and upserts by device rather than overwriting a single account-level key.
- `GET /users/:username/public-key` now returns `{ keys: [{deviceId, publicKey}] }` (array) instead of a single nullable key.
- `device_keys` table: replaces the single `public_key` column on `users`; one row per (user, device) pair.
- End-to-end encrypted direct messages. Message bodies are encrypted on-device with ECDH P-256 + HKDF-SHA256 + AES-256-GCM before leaving the client; the server stores only ciphertext and cannot read any message. Per-message ephemeral keys give forward secrecy. Key pairs are generated automatically on first use and stored in `localStorage` (web) or the Keychain (iOS). Legacy messages sent before this change are still readable as before.
- `DirectMessage.encrypted` boolean field: when true the `body` is `v2:` or `v3:` ciphertext; the client decrypts it locally.
- iOS theming: a new Appearance section in Settings lets you pick a light or dark base and browse the public theme gallery. Applying a theme recolors the app instantly, and your choice is remembered on the device. The whole app is now dark or light based on your base choice, instead of being locked to dark.
- Custom notifications: a Notifications panel in Settings (web and iOS) lets you choose which activity types you're notified about — likes, reposts, replies, new followers, mentions, and direct messages. A type you turn off is created nowhere: no inbox entry and no push.
- Direct messages now generate a notification. New `message` notification type; tapping it opens the conversation.
- iOS push notifications (APNs): the app registers its device token, asks for permission, and delivers the same notifications you'd see in the inbox, respecting your per-type settings. Tapping a push opens the relevant thread, conversation, or profile. Tokens are removed on sign-out.
- `GET /notifications/preferences` and `PUT /notifications/preferences` — read and update the per-type toggles.
- `POST /devices` and `DELETE /devices/:token` — register and remove a push token for the signed-in account.
- `notification_preferences` and `devices` tables; `notifications.conversation_id` column for deep-linking message notifications.
- Native iOS app (`apps/ios/`): full SwiftUI client targeting iOS 17+. Ships feed, post threads, profiles, private messaging, notifications, search, compose, and settings. Multi-account switching mirrors the web's session model, with accounts stored in Keychain. Token refresh is handled automatically on 401 with concurrent-refresh deduplication via `TokenRefreshActor`.
- iOS Settings is now reachable from a gear button on the Profile tab. From there you can edit your profile (display name, bio, avatar), switch between signed-in accounts, add another account, sign out, and delete your account.

### Fixed
- iOS account deletion now actually deletes the account server-side (`DELETE /auth/account`) instead of only signing out locally.
- Message bodies are encrypted at rest with AES-256-GCM before storage; the database never holds plaintext
- "Message" button on user profiles (logged-in, non-self only) links directly to the conversation thread
- Private messaging: users can send direct messages to each other. `/messages` shows the inbox sorted by most recent activity with unread badges; `/messages/:username` shows a conversation thread with a send form. Messages are marked read automatically when the thread is opened.
- `GET /messages` — paginated inbox listing all conversations with last-message preview and unread counts
- `GET /messages/:username` — cursor-paginated message thread with a given user
- `POST /messages/:username` — send a message; creates the conversation on first use
- `POST /messages/:username/read` — mark all messages from a user as read
- Account switching: sign into multiple accounts and switch between them from the nav footer without logging out. The `▾` toggle opens a tray showing all stored accounts; clicking one switches instantly. "+ add account" links to the login or register page to add another.
- `/actions/switch-account` endpoint: POST with a `userId` to reorder the session list and activate that account on the next request.
- Login and register pages now accept `?add=1` to skip the "already signed in" redirect, allowing a second account to be added while the first is still active.
- Topics system: users can create, join, and leave topics (communities), each with a member count and post count
- `GET /topics` — list all topics sorted by member count, with viewer membership state
- `POST /topics` — create a new topic; creator auto-joins on creation
- `GET /topics/:slug` — fetch a single topic with counts and viewer state
- `GET /topics/:slug/posts` — cursor-paginated feed of posts in a topic
- `POST /topics/:slug/join` and `DELETE /topics/:slug/join` — join and leave endpoints
- `/topics` discover page: browse all topics, see member and post counts, join or leave inline
- `/topics/new` page: dedicated topic creation form, works without JavaScript
- `/topics/:slug` topic page: topic header with counts, join button, and a post feed scoped to the topic
- Composer topic selector: when composing from the feed or home page, a dropdown lets you post into any topic
- Topic badge on post cards: posts that belong to a topic show a linked `▦ TopicName` chip
- `posts.topic_id` column: nullable foreign key linking a post to a topic
- `/topics` added to the main navigation

### Changed
- Session storage redesigned from a single `counter_refresh` cookie to a `counter_accounts` cookie holding a JSON array of every signed-in account. The first entry is always the active one; switching reorders it. Refresh tokens stay server-only inside the httpOnly cookie.
- Logging out of the current account now activates the next stored account automatically (if one exists) rather than signing out completely.
- Account deletion removes only the deleted account from the local session list; any other stored accounts remain active.
- Likes, reposts, and follows now update instantly without a page reload; the button state and counts flip optimistically in the client and the server call happens in the background
- Composer accepts an optional `topicId` prop; when pre-scoped (e.g. on a topic page) the topic is set via a hidden input rather than a selector
- Home page and feed page server loads now fetch topics in parallel to power the Composer selector
- Post serializer includes a `topic` field (id, slug, name) on every post that belongs to a topic

### Fixed
- Plain reposts now appear on the reposter's profile. Previously only quote-reposts (with added text) showed up; tapping the repost button on another user's post would increment the count and fill the icon but leave the profile empty.
- iOS: feed and profile redesigned to a flat, edge-to-edge layout with hairline dividers. Post cards no longer float as rounded panels, and NavigationLink disclosure chevrons no longer appear inside rows.
- iOS: post author header is now two lines: display name, optional topic, and timestamp on the first; @username on the second. Eliminates text truncation that plagued the single-line layout.
- iOS: follow button no longer appears when viewing your own profile. The API client now sends the auth token on all requests when available, so public endpoints return correct viewer context (isSelf, isFollowing).
- iOS: feed title is now a tappable menu for switching between All, Following, and any topic feed. Sources load in parallel with the first post page.
- iOS: all text now uses a single system font; the monospaced variant on handles, timestamps, and counts is removed.
- iOS: post rows now show only the username, not the display name.
- iOS: topic shown inline in the post header as `username > TopicName` rather than a separate badge row.
- iOS: tapping a post now opens the thread correctly. The app was hitting `/posts/:id` (single post) instead of `/posts/:id/thread` (ancestors + post + replies), causing a decode failure.
- Topic creation no longer relies on a client-side JS toggle; "New topic" is a plain link to `/topics/new`, so it works immediately on page load without hydration
- Form resubmission dialog on refresh after a failed topic create: the `/topics/new` form uses `use:enhance` so failures go through `fetch` and the URL stays clean
- Mobile viewport: added `maximum-scale=1, user-scalable=no` to prevent unwanted zoom on input focus
- Mobile layout: `overflow-x: hidden` on `html` and `body` prevents horizontal scroll bleed
- Changelog page rendered blank when the CHANGELOG.md had multiple `### Added` or `### Changed` sections within one release; the parser now merges duplicate category headings into one

## [0.1.0] - 2026-06-04

### Added
- Public feed powered by open ranking algorithm with live weight transparency
- Algorithm transparency page — current weights, parameters, and full version history
- Account creation, log in, and log out, backed by revocable JWT sessions
- Post creation with plain text content
- Likes, reposts, and replies on posts
- User profiles with follower and following counts
- Follow and unfollow other users
- Topic tagging on posts and topic discovery page
- Search across posts, people, and tags
- Notifications for likes, reposts, replies, and new followers
- Per-post analytics (views, likes, reposts, replies) visible to everyone — no follower gate
- Theme customization: light/dark mode and accent color selection
- Community themes: browse, apply per-device, and publish your own as server-validated CSS-variable overrides
- Responsive layout with a mobile navigation drawer
- In-app changelog page, public and unauthenticated
- No-account read access — all posts and profiles are public by default
- Open algorithm: all ranking logic is public, version-controlled, and reflected live in the UI
- No individual tracking — post view counts are anonymous aggregates only
- Optional email verification — confirm your address to earn a ✦ verified badge; never required, gates nothing
- Verified trust badges on profiles: a transparent, itemized list of proven facts (verified email, linked accounts), display only, never used to gate features or influence ranking
- Linked accounts: connect external profiles (GitHub, Bandcamp, a personal site, and more) and verify ownership via a rel="me" link-back
- Account deletion: a permanent hard delete of your account and personal data, with a typed confirmation and a written confirmation once it's done
- Public data disclosure at /data — every category of data collected, its purpose, and its retention period, readable without an account and sourced from DATA-MODEL.md
- Source-available under the Counter Social License (CSL) v1.0, with the license notice on every source file
- "Built with Counter" attribution on every page, linking to the source — the transparency indicator required by the CSL
