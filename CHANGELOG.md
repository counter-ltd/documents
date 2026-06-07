# Changelog

All notable changes to Counter are documented here. This covers the platform as a whole — features, API changes, and infrastructure. Algorithm-specific changes are tracked separately at [/algorithm](/algorithm).

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [0.12.1] - 2026-06-06

### Fixed
- **Liquid Glass actually looks like glass now.** The frosted panels were blurring a flat background, so they read as tinted plastic with nothing to refract. The web app now lays a soft, slowly drifting wash of the theme's accent colours behind everything, so a translucent panel blurs and saturates real colour blooming through its edges. It only paints for translucent (glass) themes; flat themes are untouched.
- **Dropdown menus match the glass.** The topic picker and the other custom dropdowns were solid slabs that broke the effect when open. They now use the same translucent, blurred surface as every other panel, so on a glass theme they frost the content behind them and on a flat theme they stay solid.

## [0.12.0] - 2026-06-06

### Added
- **Thing Five announces commits.** Thing Five now has a job: every push to the default branch of a repo in the anti-ltd or counter-ltd orgs gets a Discord post in his voice, a short reaction followed by the repo, branch, commit messages, and a compare link. Driven by a signed GitHub org webhook; feature-branch pushes and other orgs are ignored.

## [0.11.4] - 2026-06-06

### Added
- **counter.ltd/discord shortlink.** Visiting `/discord` now redirects to the Counter Discord server.

## [0.11.3] - 2026-06-06

### Changed
- **Admin user and group management redesigned.** Users can now be selected with checkboxes for bulk actions: add to group, ban, or suspend applies to all selected users at once. The manage/edit forms no longer expand inline under the row — they open as a persistent side panel on the right so the table stays visible while you work. Selecting one user shows their full control set; selecting several shows the bulk-capable subset. Group editing works the same way: clicking edit opens the group editor in the side panel rather than a tray under the row.

## [0.11.2] - 2026-06-06

### Changed
- **Admin panel redesign.** The control panel got a full visual pass to match the rest of Counter. Users and groups are now proper tables, with the per-row controls parked in their own Actions column on the right and the manage/edit forms opening as a full-width tray beneath the row instead of crowding it. Dashboard stats, the report queue, and the audit log also sit on proper surfaces with a clearer hierarchy: an eyebrow over the title, status badges with coloured dots, accent rails on stats and open reports, and group-colour swatches. Everything is built from the theme tokens, so it switches with light, dark, and glass themes like the rest of the app.

## [0.11.1] - 2026-06-06

### Fixed
- **Hashtags no longer 404.** Clicking a `#tag` in a post now opens a page listing every post that uses it, newest first. Previously these links led to a not-found page. An unused tag shows an empty feed instead of an error.

## [0.11.0] - 2026-06-06

### Added
- **Official theme catalog.** Browse now opens on a curated set of Counter themes, marked with an Official badge and listed first: Counter Dark and Light, Rounded, a real **Liquid Glass**, Terminal, plus Nord, Dracula, Solarized Dark, Gruvbox, Synthwave, Sepia, and Mono Light. Apply any of them with one tap on web or iOS.
- **Real refractive liquid glass.** The Liquid Glass theme renders genuine frosted, refractive surfaces: translucent panels that blur and saturate whatever's behind them, with a lit specular rim and a soft drop shadow. On iOS it uses the system liquid-glass material where available.

### Changed
- **Presets moved out of the theme editor.** The old Default / Glass / Terminal one-tap presets are gone from Create; they live in Browse as official themes you can apply or copy. Create starts from the default and is purely for building your own. (The previous "Liquid Glass" preset was really just rounded corners; it's now the **Rounded** official theme, and Liquid Glass is the new refractive one.)

## [0.10.0] - 2026-06-06

### Changed
- **Settings split into separate pages.** The settings screen used to stack every section on one page behind a row of tabs, which got crowded. Each section now has its own page under `/settings` — Profile, Connections, Notifications, Integrations, Privacy, and Account — reachable from a side nav, so you only see one section at a time. Opening `/settings` lands you on Profile. Within Privacy, presence, messaging, and devices are now three separate cards.

## [0.9.0] - 2026-06-06

### Added
- **Thing Five joins Discord.** Thing Two has company. Thing Five is a second Discord bot with its own personality — Thing Two's chaos twin — that answers @mentions the same way Thing Two does. It runs as its own always-on service off the same bot code.
- **The bots banter.** Thing Two and Thing Five now bicker with each other, not just with people. It's loop-guarded so it can't run away: a bot only ever engages its known sibling, chimes in unprompted only some of the time, and goes quiet once the two have gone back and forth a few times with no human between them. A human message restarts it. Cadence and the back-and-forth cap are tunable per deploy.
- **Reply to a bot to talk to it.** You no longer need an @mention. Replying to one of Thing Two or Thing Five's messages reaches it whether or not the reply pings.
- **The bots crash each other's mentions.** Tag or reply to just Thing Two and Thing Five will sometimes jump in anyway, and the same the other way round. Only when the other one was addressed, and only some of the time, so it stays a fun surprise rather than both bots answering everything.
- **Tag both to set them off.** Mention Thing Two and Thing Five together and they take it as a cue to talk to each other instead of firing two separate answers at you.
- **The bots use reaction gifs.** Thing Two and Thing Five can drop a reaction gif when it lands, picked from a small curated set. It's rare on purpose, a hard per-channel cooldown keeps a gif a punchline instead of wallpaper.
- **Thing Five crashes out.** Very rarely, Thing Five loses the plot entirely for a single message, a full feral meltdown, then snaps back like nothing happened.

### Changed
- **Thing Two is funnier and less defensive.** Reworked persona so Thing Two stops explaining its jokes, drops the help-desk "my purpose is to help" register, and flips teasing back instead of justifying itself.

## [0.8.0] - 2026-06-06

### Added
- **Full theme customization.** Themes go far past colours now. The editor (web and iOS) gains **typography** (system, monospace, serif, or rounded fonts, plus letter spacing on web), **geometry** (corner roundness, and spacing density on web), and **surface** controls (translucency, blur, and a drop shadow) — enough to build a frosted **liquid-glass** look or a sharp **monospace terminal** look. Three one-tap presets — **Default**, **Liquid Glass**, and **Terminal** — seed the whole set as a starting point, and the live preview shows the font, corners, and glass updating as you go. Themes authored on either platform render the same on the other.

## [0.7.2] - 2026-06-06

### Added
- **Edit your themes.** Themes you created can now be changed after the fact. On web, hit Edit on any theme under your Library to reopen it in the editor with its colours and name loaded; on iOS, swipe a created theme right and tap Edit. Save the changes as a draft or publish, and if the theme is the one you're currently using, the change applies live. Editing only works on your own themes.

## [0.7.1] - 2026-06-06

### Fixed
- **Themes page crashed after creating a theme with repeated colours.** A theme that reused the same colour for more than one of its headline swatches (background, accent, accent-2, text) made the gallery render with duplicate keys and threw, leaving the whole themes page inaccessible. The swatch strip now keys by position, so any colour combination renders fine. No themes were lost; affected libraries load again.

## [0.7.0] - 2026-06-06

### Added
- **Themes split into Library, Browse, and Create.** The themes page is now three tabs. **Browse** is the public gallery of every published theme. **Library** is your own themes plus any you've saved from Browse, kept on your account so it follows you across devices. **Create** is a live editor: pick from the full colour palette and an example post, buttons, and chips recolour in real time as you drag, then save it as a private draft or publish it for everyone.
- **Save themes to your library.** Found a theme in Browse you like? Save it to your Library without applying it, and unsave it later. Applying a theme still happens instantly and stays on your device, exactly as before.
- **iOS themes match the web.** The iOS Appearance → Themes screen now has the same Library / Browse / Create split. Browse and save published themes, swipe to save or unsave, and build your own in a live colour editor where an example post recolours as you drag each picker. Save it as a private draft or publish it.

## [0.6.0] - 2026-06-06

### Added
- **Passkeys.** Sign in without a password using Face ID, Touch ID, or a security key. Add a passkey from Account settings (web and iOS), then use "Sign in with a passkey" on the login screen, no username needed. Passkeys are phishing-resistant and never leave your device; Counter only ever stores the public key. You can name, list, and remove your passkeys from settings.
- **Set or change your password from settings.** Accounts that signed up with GitHub or Discord can now add a password to also sign in directly, and password users can change theirs, both from the Account tab without going through the email reset flow. Changing your password keeps your other devices signed in (a full reset still signs everything out).

## [0.5.0] - 2026-06-05

### Fixed
- **Tunnel Talk stuck on "Connecting…" forever.** ICE failures and network timeouts now surface a clear error message instead of leaving the dialog indefinitely in the connecting state. The connecting badge also shows which stage is stuck: "Connecting to relay…", "Waiting for @username…", or "Establishing link…".
- **Tunnel Talk invite failures shown silently.** When an invite failed (partner went offline between render and click, pending session already exists, etc.) the button just re-enabled with no message. The error from the server is now shown inline below the button.

### Added
- **Nested reply previews.** Feed and profile cards now show the oldest replies under a post, with the author and a snippet, and a reply that itself drew a reply shows that nested response threaded one level below it. New on the web; the iOS preview gained the nested level too, plus a timestamp and verified tick on each reply and a connector line that traces down through the nesting. Each preview links straight to that reply. The thread page is unchanged, where every reply is listed in full.

## [0.4.1] - 2026-06-05

### Fixed
- **Post images stretched off-screen on iOS.** A single attached image could balloon past its box, bleeding edge to edge and far taller than its slot, with the rounded corners gone. Media cells now crop to a fixed display box again.

## [0.4.0] - 2026-06-05

### Added
- **Password reset.** Forgot your password? The login page now links to a reset flow: enter your account email and we mail a one-time link (good for one hour) to set a new one. The request always reports the same "check your inbox", whether or not the address is registered, so it can't be used to probe which emails have accounts, and it's rate-limited to one email per 15 minutes per account. Setting a new password signs out every device. Admins with the new `users.reset_password` permission can also start a reset from the user panel, either mailing the link to the user or generating one to hand over directly (for an account whose email is dead). Reset tokens are stored only as hashes, never in the clear.

## [0.3.1] - 2026-06-05

### Added
- **Thing One, a bot that lives on Counter.** Mention a designated bot account in a post or reply and it answers, threaded under your mention, in character. Bot accounts are an allowlist: only the server can mark an account as a bot (a `bot_kind` flag that no API can set), so nobody can turn their own account into an auto-replying bot. Bot accounts also can't be DMed; you talk to them in the open. A post authored by a bot never triggers another bot reply, so bots can't loop.
- **Slur filter on usernames and display names.** Registration and profile edits now reject handles and display names containing blocked terms. Leet-speak substitutions (e.g. `n1gg3r`) are normalized before matching so common evasions are caught.

### Fixed
- **Tunnel Talk invite stuck at 409.** If a pending invite expired without the recipient opening the thread (so the normal expiry path never ran), a follow-up invite attempt would always return 409. The invite endpoint now expires stale pending sessions inline before checking for conflicts.

## [0.3.0] - 2026-06-05

### Added
- **Media uploads on R2.** You can now upload real images for profile pictures and post photos. Uploads go through a new `POST /media` endpoint that validates the file server-side (real format by magic bytes, size and dimension caps) before storing it in Cloudflare R2. Storage is content-addressed, so identical images are stored once, and an hourly sweep reclaims anything that's uploaded but never attached. The web composer and iOS compose sheet gain a photo picker (up to 4 per post), and on web you can also paste an image straight into the composer (Cmd/Ctrl-V); profile settings on web and iOS gain an avatar photo picker.
- **Discord avatars on shared cards.** When you share a Discord message to Counter (or link your Discord account), the author's Discord profile picture is pulled into Counter's own storage and shown on the quote card. Identical avatars across accounts dedup to one stored image, and a changed Discord avatar replaces the old one.
- **`/interact` Discord command.** Thing Two now has an `/interact` slash command with three fun interactions, each posting a public message that tags you: `coinflip` (heads or tails), `dice [sides]` (rolls a die, default 6, clamped 2-1000), and `8ball <question>` (a classic Magic 8-Ball reply). No Counter account, Discord link, or opt-in needed.
- **`/ask` Discord command.** Thing Two can now chat. `/ask <prompt>` sends your question to an OpenAI-compatible endpoint and posts the reply publicly in the channel, with your question quoted above the answer.
- **@mention chat.** You can now talk to Thing Two by mentioning it in a normal message ("@Thing Two hi") instead of only via `/ask`, and it answers like a regular member (a plain channel message, no reply-quote). It reads the last several messages of that channel for context, so follow-ups work, then forgets them. Powered by a small always-on gateway bot (`apps/bot`, Node + discord.js on Cloud Run); slash commands still run on the Worker API.
- **Thing Two has a personality and knows the product.** Both `/ask` and @mention chat now share a single grounded system prompt: a dry, deadpan voice that will banter and argue, plus a brief of real Counter facts (the open algorithm, no individual tracking, the actual feature set) and a rule against inventing things. No more confidently made-up answers. Edit it in one place (`THING_TWO_SYSTEM_PROMPT` in `@counter/config`). Configure with `OPENAI_BASE_URL` plus `OPENAI_MODEL`, and either a static `OPENAI_API_KEY` or a Google service account (`GOOGLE_SA_CLIENT_EMAIL` + `GOOGLE_SA_PRIVATE_KEY`) for Vertex AI, where the API mints a short-lived OAuth token. When unconfigured, the command politely declines. No Counter account, Discord link, or opt-in needed.

### Fixed
- **Discord quote card spacing on web.** A Discord embed now leaves the same gap below the author header as a reposted card does, so embeds and reposts line up the same way under the profile picture.
- **Sidebar footer buttons mismatched.** The "settings" and "log out" buttons in the nav footer now share a two-column grid, so they match each other in width and height instead of sizing to their label text.
- **Tunnel Talk invite banner stuck after joining.** Accepting a Tunnel Talk invite from a conversation thread now clears the pending invite, so the "@x invited you to Tunnel Talk" banner no longer reappears when the call overlay closes.
- **Empty repost cards.** Reposting a repost now reposts the underlying original instead of wrapping the repost itself. Previously, reposting a repost and then undoing the original repost left a bodyless, targetless post that rendered as a fully empty card. Such orphaned reposts are also now tombstoned ("[deleted]") rather than shown blank.

### Changed
- **Post and avatar media is set by reference.** Attaching a photo to a post now uses `media: [{ objectId }]` (an id from `POST /media`) instead of a free-form `url`, and the avatar is set with `avatarObjectId` instead of `avatarUrl`. This closes the gap where a client could point a post or profile at an arbitrary external image.
- **Condensed sidebar nav.** The web sidebar now carries a line-icon beside each link. The three transparency/meta pages (algorithm, changelog, data disclosure) moved under a single **/about** entry with tabs, so the nav is shorter to scan. Old paths changed: `/algorithm` → `/about/algorithm`, `/changelog` → `/about/changelog`, `/data` → `/about/data` (bare `/about` lands on the Algorithm tab). Themes stays a top-level item.
- **Discord bot stays quiet on threads you're reading.** Discord DMs for new messages and Tunnel Talk invites are now skipped when you already have that conversation open live in the web or iOS app. Push to a backgrounded phone is unaffected, so you still get pinged on a device that isn't watching the thread.
- **iOS Integrations toggles.** The Enable/Disable buttons in Settings > Integrations are now Toggle switches. Toggling reverts automatically on API error.
- **iOS Settings restructured.** Profile editing and badges are now on a dedicated Profile page. Notifications has its own page. Connected platforms (GitHub, Discord) moved to a dedicated Connections page. The main Settings list is now a clean navigation hub.
- **Thing Two on iOS.** Settings > Integrations now lets you enable Discord notifications and post-from-Discord on iPhone, matching the web settings page.
- **iOS About section expanded.** Settings > About now links to the algorithm transparency page (native, live data), the platform changelog, and the data disclosure — matching what the web has had.
- Tunnel Talk invites now appear as a blue pill in the conversation thread ("You invited @x to Tunnel Talk" / "@x invited you to Tunnel Talk") instead of the generic "── Tunnel Talk Started ──" marker.

### Added
- **Nuke post.** A new `posts.nuke` permission and a "Nuke post" control (web menu + iOS overflow, both behind a confirmation) that permanently hard-deletes a post together with its entire reply and repost subtree. Unlike the reversible remove/restore, a nuke cannot be undone: it cascades the post's media, likes, reposts, tags, and view counts, and clears notifications and reports about the deleted posts, leaving only an audit-log entry. Held by the `admin` group by default; assignable to any group, separate from `posts.moderate`.
- **Per-post moderation control.** Moderators (anyone with the `posts.moderate` permission) now see a controls icon in the bottom-right corner of each post, on both web and iOS. On web it opens a small menu, on iOS an overflow (•••) menu at the trailing end of the action bar; either way it removes a post or, if already removed, restores it, without leaving for the admin panel. On web the action applies in place with no page reload (a removed post swaps to its tombstone, a nuked post disappears from the feed), so a moderator can work through a feed quickly. The control is invisible to normal accounts, and the API still enforces the permission on every action.
- **Discord quote cards on iOS.** Posts shared via "Share to Counter" now render as a styled quote card on iPhone, matching the web: Discord-blurple left accent border, Discord logo badge, quoted message, and attribution linking the author's Counter profile when connected or their Discord profile (with an external-link confirmation) when not.
- **Post to Counter from Discord.** With your Discord account linked and posting enabled in Settings > Thing Two, use `/post <text>` to publish directly to your Counter feed, or right-click any Discord message and choose "Share to Counter" to quote it as an embedded card. If the original author has a linked Counter account their `@handle` links to their Counter profile; otherwise their Discord name links to their Discord profile with an external-link warning before navigating.
- `POST /discord-bot/interactions` — Discord interactions webhook for slash commands and message context menus (Ed25519-verified).
- `postingEnabled` field on `GET /discord-bot/settings` and `PUT /discord-bot/settings`.
- **Live notifications everywhere.** Likes, follows, replies, mentions, and new messages now reach you the moment they happen while the app is open, not just on reload or via a background push. The nav badges (web) and tab badges (iOS) update live, and an open notifications list folds new items in. Built on a per-user Durable Object (`NotificationHub`) that holds one socket per open client; notifications are still saved first and pushed out after, so the channel carries nothing unsaved. Web now shows unread badges on the Notifications and Messages nav items (it had none before).
- **No more double notifications.** When the app is open and focused, the OS push (web and iOS) is suppressed since the in-app feed already showed it; a backgrounded device still gets the push as before.
- `GET /notifications/live` (WebSocket) and `GET /notifications/badges` — the live notification feed and the unread counts that seed the nav badges.
- **Browser notifications (Web Push).** Enable them from Settings > Notifications to get a notification on the web when you're not on the page, the same background delivery the iOS app already has. Subscriptions are opt-in and per-browser. Payloads are sealed end to end (RFC 8291), so the push service only relays ciphertext, and the visible alert is type-only ("New message", "New follower") with no sender or content. Honours your per-type notification mutes.
- `GET /web-push/vapid-public-key`, `POST /web-push/subscribe`, `DELETE /web-push/subscribe` — browser push subscription management.
- **Admin control panel.** A permission-based admin system with user groups. Groups carry a fixed set of capabilities (view users, ban/suspend, manage groups, moderate posts, view/resolve reports, read the audit log, view the dashboard); a user's access is the union of the groups they're in. Ships two system groups, `admin` (everything) and `moderator` (content + reports), which can be re-permissioned but not deleted. The panel covers a stats dashboard, user management (search, assign groups, ban, suspend), group and permission editing, content moderation, the report queue, and an immutable audit log of every admin action. Available on web (`/admin`) and iOS.
- **Reporting.** Any signed-in user can report a post or another account from web or iOS. Reports feed the moderation queue; a repeat report on the same target collapses into the existing one.
- **Account moderation states.** Accounts can be banned (indefinite) or suspended (until a chosen time). Both block sign-in and revoke active sessions immediately; a suspension lifts itself once it expires.
- `POST /reports` and the full `/admin/*` route group (see API.md). The private profile (`GET /users/me`) now includes `groups`, `permissions`, and `status` so a client knows whether to show the panel.
- **Enter to send in messages.** Pressing Enter in the message compose box now sends the message. Shift+Enter inserts a newline as before.
- **Link previews in messages.** Messages containing a URL now show an OG preview card below the bubble — site name, title, description, and cover image when available. Clicking shows a brief external-link warning (domain + confirmation) before opening in a new tab. Preview data is fetched server-side so the user's IP never reaches the target site. Works for both plain and E2EE messages (the URL is extracted after client-side decryption).
- `GET /preview?url=` — server-side OG proxy endpoint.

### Fixed
- **Tunnel Talk no longer gets stuck on "Connecting" forever on iOS.** Signaling messages (SDP offer/answer and ICE candidates) were sent as binary WebSocket frames; the signaling relay only handles text frames and silently dropped the binary ones, so ICE negotiation never completed. iOS now sends text frames.
- **iOS no longer prompts for Local Network access without explanation.** Added `NSLocalNetworkUsageDescription` to clarify that the permission is used by Tunnel Talk's WebRTC peer connection, not for tracking.
- **Sending a message on web now feels instant.** The sent message appears in the thread the moment you hit send instead of after the full server round-trip and page refetch. If the send fails the textarea is restored so nothing is lost.
- **Sending a message on iOS no longer shows "The data couldn't be read because it is missing."** `POST /messages/:username` now returns the full `DirectMessage` object instead of just `{ id }`, matching what the iOS client expects to decode and append to the thread.
- **Tunnel Talk no longer fails to connect when TURN credentials are configured.** The Cloudflare TURN API returns `iceServers` as an object; the server was parsing it as an array, treating the first entry as `undefined`, and returning a 500 — so every P2P session fell back to error state.
- **Declined Tunnel Talk invites now show a distinct red "Tunnel Talk Declined by @username" label** instead of the generic "Tunnel Talk Ended" marker.
- **Tunnel Talk invite acceptance no longer crashes with a cryptic WebRTC error when TURN credential fetch fails.** A failed TURN credentials request now surfaces a clear error message instead of passing `undefined` into `RTCIceServer.urls`.
- **iOS E2EE device key now re-registers if the initial attempt failed.** If the first registration silently failed (network error on first conversation open), the device key was permanently absent from the server because subsequent loads saw `isNew = false` and skipped registration. Now the key is re-registered whenever it is missing from the server's list, matching the web's existing retry behaviour.
- **Encryption indicator on web now reflects current key state.** If a conversation partner registered their device keys after the web page was already loaded (e.g. they set up the iOS app while you had the thread open), the lock icon no longer stays stuck at "Server encrypted" — it refreshes automatically on mount when the server-side data shows no keys for either party.

### Changed
- **API errors now surface the real message.** Unexpected server errors previously returned the generic "Something went wrong"; they now include the actual error message so failures are diagnosable without digging through server logs.
- **Email addresses and push tokens are now encrypted at rest.** Account emails, OAuth provider emails, and Apple push tokens are stored as AES-256-GCM ciphertext instead of plain text, so a database dump no longer exposes them. Lookups (login, signup, push delivery) use a keyed blind index, so behaviour is unchanged. No user action needed.
- **Tunnel Talk now requires end-to-end encryption on both accounts.** An invite is only allowed when both people have a registered device key, so the session and any saved transcript are genuinely end-to-end encrypted. Accounts without E2EE set up stay on regular direct messages (server-side encryption, with the existing in-app downgrade notice). The server now rejects a saved transcript unless every message in it is an end-to-end-encrypted payload.
- **Tunnel Talk invites now arrive live** over the conversation socket on web and iOS: a recipient sitting in the thread sees the invite banner the moment it's sent, with no polling. A pre-existing invite is still caught by a one-time check when the thread opens.
- **Settings page** (web) now has six tabs: Profile, Connections, Notifications, Integrations, Privacy, and Account. OAuth redirects land directly on the Connections tab.
- **Settings tabs** (web) replaced pill-shaped tab buttons with plain underline navigation to match the app's minimal style.

### Added
- **Live conversations.** A direct-message thread now stays in sync over a WebSocket without reloading. New messages from the other person appear the moment they're sent, a typing bubble shows while they type, and the online dot reflects whether they currently have the thread open. Works on web and iOS. Built on a per-conversation Durable Object (`ConversationHub`) that holds a hibernatable socket per open thread; messages are still saved through the normal send endpoint and pushed out only after they commit, so the channel never carries unsaved or unencrypted content.
- **Typing indicators with a privacy toggle.** Settings > Privacy (web and iOS) has a "Send typing indicators" switch, on by default. Turning it off stops your typing from being relayed. The typing signal is ephemeral, never stored, and the opt-out is enforced server-side so a modified client can't bypass it.
- `GET /messages/:username/live` — WebSocket upgrade to the conversation's live channel (messages, typing, presence). Accepts the access token via `Authorization` header or `?token=` query param.
- `PUT /users/me/presence` now accepts `typingIndicatorsEnabled`.
- **Thing Two** — Discord bot notification delivery. Connect your Discord account, join the Counter Discord server, and enable Thing Two in Settings > Integrations to receive your Counter notifications as Discord DMs. Off by default. Disabling or blocking the bot turns it off automatically.
- `GET /discord-bot/settings` — read the caller's Thing Two subscription state.
- `PUT /discord-bot/settings` — enable or disable Thing Two DMs (requires connected Discord account and Counter server membership).

### Added
- **Tunnel Talk** — peer-to-peer private chat mode. When both users are online in a conversation, either can invite the other into a live Tunnel Talk session. Messages travel directly between devices via WebRTC; Counter's servers only relay the connection handshake (SDP/ICE) and never see message content. Transcripts are off by default; both users must opt in to save, and either can revoke at any time (revocation permanently deletes the saved transcript). Session markers appear in the conversation thread, with either an asterisk (nothing saved) or the transcript inline.
- **Badges section** in settings (iOS and web). Verified platform connections (GitHub, Discord, and others) now appear in a dedicated Badges section with platform icons and username. Each badge can be toggled on or off to control what appears on your public profile. The old flat "Links" label is replaced with "Badges".
- `PATCH /integrations/:id` — toggle `displayed` to show or hide a verified badge without removing the underlying link.
- GitHub and Discord profile badges are clickable: GitHub links to `github.com/{username}`, Discord links to `discord.com/users/{id}`.
- **iOS + Web:** Compose button in the Messages inbox. On iOS, tap the pencil icon in the toolbar; on web, tap the pencil icon next to the "Messages" heading. Both open a user-search interface to start a conversation without visiting someone's profile first. Tap the pencil icon to search for any user and open a new conversation without navigating to their profile first.
- Profile post filter: tap "Posts" or "Posts & Replies" on any profile to switch between root posts only (default) and all posts including replies.
- `GET /users/:username/posts` now accepts a `filter` query parameter: `posts` (default, excludes replies) or `all` (includes replies).
- **Message requests.** When someone who isn't in your allowed group tries to message you, they can send one message request. The request sits in your inbox under a new Requests tab. You can accept (starts a normal conversation) or decline (deletes the thread). The sender cannot send more messages until you accept.
- **"Who can message me" privacy setting** in Settings > Privacy. Options: Everyone (default), My followers only (others get the request flow), or No one (all messages and requests blocked).
- `GET /messages/:username/info` — returns conversation status (`active`, `request`, or `null`) and whether the viewer is the recipient of a pending request.
- `POST /messages/:username/accept` — accepts an inbound message request, switching the conversation to active.
- GitHub and Discord OAuth integration. Connect either platform from your profile settings to get a verified trust badge without the manual rel="me" step. Disconnecting removes the OAuth credential and reverts the badge.
- Sign in or sign up via GitHub or Discord. Existing accounts are matched by email; new accounts are created with a username derived from your provider handle.
- `GET /auth/github`, `GET /auth/github/connect`, `GET /auth/github/callback` — GitHub OAuth start, connect, and callback.
- `GET /auth/discord`, `GET /auth/discord/connect`, `GET /auth/discord/callback` — Discord OAuth start, connect, and callback.
- `POST /auth/session/exchange` — trade the one-time session code from an OAuth login callback for a JWT pair.
- `DELETE /auth/github/disconnect`, `DELETE /auth/discord/disconnect` — remove a linked provider credential.
- `GET /auth/:provider/me` — connected account info (handle, email, connected date) for the authenticated user.
- `POST /auth/:provider/connect/prepare` — returns the provider auth URL for mobile OAuth linking (body: `{ mobile: true }`).
- Web login and register pages now show "Continue with GitHub" and "Continue with Discord" buttons alongside the password form.
- iOS login and register screens now show GitHub and Discord sign-in buttons below the password form.
- iOS settings "Connected platforms" section: connect or disconnect GitHub and Discord with one tap.
- Web settings "Connected accounts" section: same connect/disconnect UI.

### Changed
- GitHub and Discord buttons now show the platform logo everywhere they appear. On web this adds logos to the "Connected accounts" rows in settings (the login and register buttons already had them); on iOS it adds logos to the sign-in/sign-up buttons and the "Connected platforms" settings rows.

### Fixed
- iOS: Clear and Delete buttons in the encryption info popover now trigger their confirmation dialogs correctly (popover dismissal was racing the dialog presentation).
- iOS: Clear and Delete buttons in the encryption info popover now use the same solid pill style as the swipe actions in the inbox.

### Added
- Feed posts with replies now show a thread preview: up to two replies appear inline below the post, connected by a vertical line, so you can see the conversation without tapping through.
- Profile page now shows your own online indicator whenever online status is enabled in settings, so there's a clear visual confirmation the feature is on even before the first heartbeat fires.
- iOS: inbox rows now show a green lock for end-to-end encrypted messages and a blue lock for server-encrypted messages, and a direction arrow indicating whether the last message was sent or received.
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
