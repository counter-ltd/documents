# Changelog

All notable changes to Counter are documented here. This covers the platform as a whole — features, API changes, and infrastructure. Algorithm-specific changes are tracked separately at [/algorithm](/algorithm).

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [Unreleased]

### Added
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
- Composer accepts an optional `topicId` prop; when pre-scoped (e.g. on a topic page) the topic is set via a hidden input rather than a selector
- Home page and feed page server loads now fetch topics in parallel to power the Composer selector
- Post serializer includes a `topic` field (id, slug, name) on every post that belongs to a topic

### Fixed
- Topic creation no longer relies on a client-side JS toggle; "New topic" is a plain link to `/topics/new`, so it works immediately on page load without hydration
- Form resubmission dialog on refresh after a failed topic create: the `/topics/new` form uses `use:enhance` so failures go through `fetch` and the URL stays clean
- Mobile viewport: added `maximum-scale=1, user-scalable=no` to prevent unwanted zoom on input focus
- Mobile layout: `overflow-x: hidden` on `html` and `body` prevents horizontal scroll bleed

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
