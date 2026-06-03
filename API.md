# Counter API

The API is the product. Every client — web, iOS, macOS, third-party — talks to the same endpoints. No client holds special privilege over another.

**Base URL:** `https://api.counter.ltd/v1`  
**Format:** JSON  
**Auth:** Bearer token (JWT)

---

## Authentication

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/register` | Create account |
| POST | `/auth/login` | Login, returns token |
| POST | `/auth/logout` | Invalidate token |
| POST | `/auth/refresh` | Refresh access token |
| DELETE | `/auth/account` | Permanently delete account and all data |

---

## Users

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/users/:username` | Get public profile |
| PATCH | `/users/me` | Update own profile |
| GET | `/users/me` | Get own profile (authenticated) |
| GET | `/users/:username/posts` | Get posts by user |
| GET | `/users/:username/followers` | Get followers list |
| GET | `/users/:username/following` | Get following list |
| POST | `/users/:username/follow` | Follow a user |
| DELETE | `/users/:username/follow` | Unfollow a user |

---

## Posts

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/posts` | Get feed (authenticated) |
| GET | `/posts/public` | Get public feed (no auth required) |
| POST | `/posts` | Create a post |
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
| GET | `/tags/:tag` | Get posts with tag |
| GET | `/tags/trending` | Get trending tags |

---

## Notifications

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/notifications` | Get notifications |
| POST | `/notifications/read` | Mark all as read |
| PATCH | `/notifications/:id/read` | Mark one as read |

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

All optional. Linked at user discretion.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/integrations/github` | Link GitHub account |
| DELETE | `/integrations/github` | Unlink GitHub account |
| GET | `/integrations/github/:username` | Get public GitHub data for profile |
| POST | `/integrations/:platform` | Link external platform |
| DELETE | `/integrations/:platform` | Unlink external platform |

Supported platforms: `github` `bandcamp` `soundcloud` `letterboxd` `goodreads` `strava` `itch`

---

## Themes

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/themes` | Get public themes (browsable, no auth) |
| GET | `/themes/:id` | Get a single theme |
| POST | `/themes` | Publish a theme |
| DELETE | `/themes/:id` | Delete own theme |
| GET | `/themes/posts/:postId` | Get theme embedded in a post |

Themes are flat JSON objects of CSS variables. Validated server-side for structure, never executed.

---

## Algorithm

The algorithm is open source. This endpoint exposes its current state publicly.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/algorithm` | Current ranking weights and parameters |
| GET | `/algorithm/changelog` | Full public changelog of every change |

---

## Cross-posting

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/crosspost` | Publish a post to connected external platforms |
| GET | `/crosspost/connections` | Get connected external accounts |
| POST | `/crosspost/connect/:platform` | Connect external platform |
| DELETE | `/crosspost/connect/:platform` | Disconnect external platform |

Supported: `threads` `twitter` `mastodon` `bluesky`

---

## Admin (self-hosted instances only)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/admin/stats` | Instance stats |
| POST | `/admin/users/:id/suspend` | Suspend a user |
| DELETE | `/admin/users/:id/suspend` | Unsuspend a user |
| DELETE | `/admin/posts/:id` | Remove a post |

---

## Notes

- All public content is readable without authentication
- Pagination uses cursor-based `?after=` and `?limit=` params
- Rate limits are documented in response headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- All endpoints return consistent error shapes: `{ error: { code, message } }`
