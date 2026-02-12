# Campus Confessions

Anonymous Twitter-style confession board for college campuses. Built for a DevOps workshop — intentionally simple, no frameworks.

## Tech Stackdfddfd

- **Backend:** Node.js + Express (no frameworks beyond Express)
- **Database:** SQLite3 (file-based, `confessions.db` auto-created on first run)
- **Frontend:** Single HTML file with vanilla JS + CSS (no build step)
- **Containerization:** Docker (Alpine-based)

## Project Structure

```
├── server.js              # Express API server (all backend logic)
├── public/
│   └── index.html         # Entire frontend (HTML + CSS + JS in one file)
├── package.json           # Dependencies: express, sqlite3, cors
├── Dockerfile             # Production container (node:18-alpine)
├── .gitignore             # Ignores node_modules, confessions.db, .DS_Store
└── confessions.db         # SQLite database (auto-generated, gitignored)
```

## Running

```bash
npm install          # Install dependencies
npm run dev          # Development with auto-reload (nodemon)
npm start            # Production
docker build -t campus-confessions . && docker run -p 3000:3000 campus-confessions
```

Server runs on `http://localhost:3000` (or `PORT` env var).

## Database Schema

Three tables, all auto-created on startup:

```sql
posts (id, text, mood, likes, reposts, timestamp)
reactions (id, post_id, type, timestamp)          -- FK → posts.id
replies (id, post_id, text, timestamp)            -- FK → posts.id
```

- `mood` values: `none`, `love`, `happy`, `sad`, `angry`, `anxious`, `excited`
- `reaction` types: `love`, `haha`, `sad`, `angry`, `fire`
- `likes` and `reposts` are integer counters on the posts table (increment only)

## API Endpoints

All routes return JSON. Base path: `/api/posts`

| Method | Route                     | Body                    | Description                          |
|--------|---------------------------|-------------------------|--------------------------------------|
| GET    | `/api/posts`              | —                       | Get all posts (with reactions/reply counts). Query: `?sort=top` for trending |
| POST   | `/api/posts`              | `{ text, mood? }`       | Create post (280 char max)           |
| POST   | `/api/posts/:id/like`     | —                       | Increment like counter               |
| POST   | `/api/posts/:id/repost`   | —                       | Increment repost counter             |
| POST   | `/api/posts/:id/react`    | `{ type }`              | Add emoji reaction                   |
| GET    | `/api/posts/:id/replies`  | —                       | Get replies for a post               |
| POST   | `/api/posts/:id/reply`    | `{ text }`              | Add reply to a post                  |

### GET /api/posts response shape

```json
[
  {
    "id": 1,
    "text": "confession text",
    "mood": "love",
    "likes": 3,
    "reposts": 1,
    "timestamp": "2025-01-15 10:30:00",
    "reaction_count": 5,
    "reply_count": 2,
    "reactions": { "love": 3, "fire": 2 }
  }
]
```

## Frontend Architecture

Everything is in `public/index.html` — no build, no modules, no bundler.

**Layout:** Twitter-style timeline (600px centered column, sticky header, compose box, feed)

**Key sections in the HTML file:**
- `<style>` — All CSS (timeline layout, compose box, post cards, action bar, replies, toast)
- `<div class="timeline">` — Main app container
- `<script>` — All JS logic

**JS structure (inside `<script>`):**
- Config constants: `API`, `MAX`, `CIRCUMFERENCE`, emoji/mood maps
- Anonymous identity: `anonName(id)`, `anonHandle(id)`, `anonColor(id)` — deterministic from post ID
- Session state: `liked`, `reposted`, `bookmarked` (Sets), `reacted` (Map of Sets) — per-session only, no persistence
- Core functions: `loadPosts()`, `createPost()`, `doLike()`, `doRepost()`, `doBookmark()`, `react()`, `toggleReplies()`, `fetchReplies()`, `sendReply()`
- Helpers: `timeAgo()`, `esc()` (XSS escaping), `fmtCount()`, `toast()`

**Features:**
- Compose with mood tags, circular SVG character counter (280 limit), auto-resizing textarea
- For You / Trending tabs
- Post cards with: avatar, anonymous name/handle, timestamp, mood badge, text
- 5 emoji reaction buttons per post
- Twitter-style action bar: reply, repost, like (heart), bookmark
- Expandable reply threads with inline compose
- Toast notifications

## Design Decisions

- **No auth** — fully anonymous by design (workshop project)
- **No user accounts** — anonymous identity derived from post/reply ID
- **Likes/reposts are server-side counters** — reactions are stored individually in a separate table for per-type counts
- **Bookmarks are client-side only** — lost on page refresh (no user sessions)
- **Rate limiting not implemented** — acceptable for workshop scope
- **280 character limit** enforced both client-side (maxlength) and server-side
- **XSS protection** via `esc()` helper that uses `textContent`/`innerHTML` trick
- **Trending sort** ranks by `reactions + reposts + likes`
