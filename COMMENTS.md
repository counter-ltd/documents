# Comment style guide

How we comment code in the Counter `core` repo. The goal is simple: someone new
should be able to read a file top to bottom and understand not just *what* it
does but *why* it does it that way. We comment generously and we comment to
teach.

This guide is the house standard. New code follows it; when you touch old code,
nudge it toward this style.

---

## The three layers

Every source file gets up to three layers of comment. Use the ones that earn
their place; don't pad.

### 1. File header

A `/** … */` block at the very top of the file describing what this file is
responsible for in the system. Plain language. Enough that a reader knows
whether they're in the right place before reading a single line of code.

```ts
/**
 * Signing and verification for the JSON Web Tokens that power login.
 *
 * Two kinds of token are in play. The access token is short-lived and gets
 * sent on every request to prove who you are. The refresh token lives much
 * longer and has exactly one job: handing it in gets you a fresh access token.
 */
```

### 2. TSDoc on exported symbols

Every exported function, type, interface, and non-obvious constant gets a
`/** … */` block. Lead with a one-line summary of what it's for, then add
`@param` / `@returns` when they tell the reader something the types don't.

```ts
/**
 * Mint a signed refresh token tied to a specific login session.
 *
 * @param userId     Who the token belongs to.
 * @param sessionId  The session this token can refresh, so we can revoke one
 *                   session without touching the user's others.
 * @returns          The signed JWT, ready to store and hand back later.
 */
```

Don't restate the type signature in prose. `@param userId The user id` adds
nothing. `@param userId Who the token will authenticate` adds intent.

### 3. Inline comments for the *why*

Use `//` inside function bodies to explain reasoning that the code itself can't:
the constraint that forced this approach, the bug this guards against, the
non-obvious ordering, the thing that bit us last time.

```ts
// We look the config up through a function call instead of reading it once at
// the top of the file. On Cloudflare Workers the secrets don't exist until the
// first request arrives, so grabbing them at import time would read undefined.
const cfg = () => loadServerEnv();
```

---

## Voice

Write like an engineer explaining the code to a teammate sitting next to them.
Conversational, direct, and aimed at teaching.

- **Explain the why, never narrate the what.** `// loop over users` on a line
  that loops over users is noise. `// oldest-first so the thread reads top down`
  earns its place.
- **Plain words over ceremony.** "so that", "because", "otherwise" beat
  "in order to facilitate".
- **It's fine to be a little human.** "this one bit us in prod" is a perfectly
  good comment.

---

## De-AI checklist

These are the tells that make a comment read as machine-generated. Avoid them.

- **No em-dashes (`—`).** Use a comma, parentheses, "so", or two sentences.
- **No filler openers:** "Note that", "Importantly", "It's worth noting",
  "Keep in mind", "Of course".
- **No hedging stacks:** "this might possibly potentially". Say it or don't.
- **No grader voice.** Don't write comments that sound like they're justifying
  the code to a reviewer. Write them for the next person maintaining it.
- **No restating the obvious** just to have a comment on every line.

---

## Mechanics

- TypeScript / Svelte `<script>`: `/** */` for headers and TSDoc, `//` inline.
- Svelte markup: `<!-- … -->`, used sparingly to label non-obvious blocks.
- CSS: `/* … */`, only where a rule's purpose isn't self-evident.
- Keep `// --- section ---` dividers in long files; they're good signposting.
- Wrap comment prose at roughly the same width as the surrounding code.

---

## What not to comment

- Don't document the changelog or release-notes content here or in code; that
  lives in `CHANGELOG.md` and stays out of source comments.
- Don't comment generated files, build output, or vendored code.
- Don't leave commented-out code. Delete it; git remembers.
