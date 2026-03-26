# YouTube Watch Later Client — Claude Code Plan

## Project Summary

A custom YouTube Watch Later client with a better UX than YouTube's native implementation. Key goals:
- All common actions (delete, tag, mark watched, reorder) accessible without hidden menus
- Bulk operation support on all actions
- Built-in YouTube iframe player with watch time tracking
- Full sort, filter, search, and custom tagging on the list
- Drag-to-reorder queue

---

## Stack

| Layer | Choice | Reason |
|---|---|---|
| Framework | Next.js 14 (App Router) | Native Vercel deployment, built-in API routes for server-side OAuth token handling |
| Styling | Tailwind CSS | Utility-first, consistent with other projects |
| Auth | NextAuth.js (Google/YouTube OAuth2 provider) | Handles token refresh, session management, Google OAuth flow |
| Database | Supabase (Postgres) | Stores local metadata: watch progress, tags, custom position, resume points |
| Drag & Drop | dnd-kit | Already familiar from MTG project |
| Player | YouTube IFrame Player API | JS event hooks required for watch time tracking (not bare iframe) |
| Deployment | Vercel | Zero-config Next.js deployment |

---

## Environment Variables Needed

```env
# NextAuth
NEXTAUTH_URL=https://your-vercel-domain.vercel.app
NEXTAUTH_SECRET=<generate with: openssl rand -base64 32>

# Google OAuth (from Google Cloud Console)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
```

### Google Cloud Console Setup (do before scaffolding)
1. Create a project at console.cloud.google.com
2. Enable **YouTube Data API v3**
3. Create OAuth 2.0 credentials (Web application type)
4. Add authorized redirect URI: `https://your-domain.vercel.app/api/auth/callback/google`
5. Add OAuth scopes:
   - `https://www.googleapis.com/auth/youtube`
   - `https://www.googleapis.com/auth/youtube.readonly`

---

## Supabase Schema

Run these in the Supabase SQL editor to initialize:

```sql
-- Cached YouTube Watch Later items
create table videos (
  id              text primary key,         -- YouTube video ID (e.g. "dQw4w9WgXcQ")
  yt_item_id      text not null,            -- YouTube playlistItem ID (required for deletion via API)
  title           text not null,
  channel         text not null,
  channel_id      text,
  duration_sec    int,
  thumbnail       text,
  added_at        timestamptz,
  position        int not null default 0,   -- local drag-to-reorder position
  synced_at       timestamptz default now()
);

-- Local user metadata about each video
create table video_meta (
  video_id          text primary key references videos(id) on delete cascade,
  watched_sec       int not null default 0,      -- cumulative watch time in seconds
  last_position_sec int not null default 0,      -- resume point (last playback position)
  fully_watched     boolean not null default false,
  tags              text[] not null default '{}',
  notes             text,
  updated_at        timestamptz default now()
);

-- Index for common filter/sort operations
create index on videos(channel);
create index on videos(added_at);
create index on video_meta(tags) using gin;
```

---

## Project File Structure

```
/
├── app/
│   ├── layout.tsx                  # Root layout, SessionProvider wrapper
│   ├── page.tsx                    # Redirect to /queue or /login
│   ├── login/
│   │   └── page.tsx                # Sign in with Google button
│   └── queue/
│       └── page.tsx                # Main app view (list + player)
│
├── components/
│   ├── VideoList/
│   │   ├── VideoList.tsx           # dnd-kit sortable list container
│   │   ├── VideoRow.tsx            # Single video row with inline actions
│   │   ├── BulkToolbar.tsx         # Appears on selection: delete/tag/mark/move
│   │   └── VideoListFilters.tsx    # Sort / filter / search bar
│   ├── Player/
│   │   ├── PlayerPanel.tsx         # Right-panel player container
│   │   ├── YouTubePlayer.tsx       # IFrame API wrapper with event hooks
│   │   └── WatchTimeTracker.tsx    # setInterval polling + Supabase flush
│   └── Tags/
│       └── TagEditor.tsx           # Inline tag add/remove UI
│
├── api/
│   ├── auth/
│   │   └── [...nextauth]/route.ts  # NextAuth handler
│   ├── queue/
│   │   ├── route.ts                # GET: fetch & sync WL from YouTube API
│   │   └── [itemId]/route.ts       # DELETE: remove item from YouTube WL
│   └── sync/
│       └── route.ts                # POST: full re-sync YouTube → Supabase
│
├── lib/
│   ├── youtube.ts                  # YouTube Data API v3 client helpers
│   ├── supabase.ts                 # Supabase client (server + browser)
│   └── sync.ts                    # Diff logic: YouTube list vs local DB
│
└── types/
    └── index.ts                    # Shared TypeScript types
```

---

## Core Logic: Sync Strategy

On every app load (or manual refresh), run a sync:

```
1. Fetch all items from YouTube WL playlist (playlistItems.list, playlistId="WL")
2. Fetch all video IDs currently in local Supabase videos table
3. Diff:
   - Items in YouTube but not local → INSERT into videos, INSERT into video_meta
   - Items in local but not YouTube → mark as removed (or delete, TBD)
   - Items in both → update title/thumbnail/duration if stale (check synced_at)
4. Preserve local position, tags, watch progress — never overwrite these from YouTube
```

Sync runs server-side in `/api/queue` so the YouTube access token stays off the client.

---

## Core Logic: Watch Time Tracking

Uses the [YouTube IFrame Player API](https://developers.google.com/youtube/iframe_api_reference), not a bare `<iframe>`.

```
1. Player fires onStateChange event
2. On YT.PlayerState.PLAYING → start setInterval polling getCurrentTime() every 5 seconds
3. Each tick:
   a. Compute delta since last tick
   b. Accumulate into local state (watched_sec, last_position_sec)
4. On PAUSED, ENDED, or video change → flush accumulated values to Supabase video_meta
5. On ENDED → set fully_watched = true if watched_sec >= duration_sec * 0.9
```

Flush is debounced/batched — don't write to Supabase on every tick, only on pause/end/unload.

---

## Core Logic: YouTube API Quota Awareness

YouTube Data API v3 has a **10,000 unit/day** quota per project.

| Operation | Cost |
|---|---|
| `playlistItems.list` (100 items) | 1 unit |
| `playlistItems.delete` (1 item) | 50 units |
| `videos.list` (50 items, for duration/details) | 1 unit |

**Implications:**
- Bulk delete of 200 items = 10,000 units = full daily quota. Surface this as a warning in the UI when bulk-deleting large batches.
- Cache video details in Supabase aggressively — only re-fetch from YouTube when `synced_at` is stale (e.g., >24h).
- Sync on load but don't re-sync on every navigation — use a stale-while-revalidate pattern.

---

## UI Layout

```
┌─────────────────────────────────────────────────────────────┐
│  🎬 WatchLater   [Search...]  [Sort ▾]  [Filter ▾]  [Sync] │
├──────────────────────────────────┬──────────────────────────┤
│                                  │                          │
│  ┌ BulkToolbar (on selection) ┐  │   ▶ YouTube Player      │
│  │ Delete  Tag  Mark Watched  │  │                          │
│  └────────────────────────────┘  │   Now Playing: Title     │
│                                  │   Channel Name           │
│  ☐ ⠿ [thumb] Title              │                          │
│        Channel • 12:34           │   ⏱ Watched: 4m 32s     │
│        ▓▓▓░░░░░ 40%              │   ↩ Resume from 3:21?   │
│        🏷 linux  sysadmin  [+]   │                          │
│                                  │                          │
│  ☐ ⠿ [thumb] Title              │                          │
│        Channel • 8:02            │                          │
│        ░░░░░░░░ 0%               │                          │
│        🏷 [+]                    │                          │
│                                  │                          │
│  (drag handle ⠿ visible on hover)│                          │
└──────────────────────────────────┴──────────────────────────┘
```

**Video row actions (always visible, no hidden menus):**
- Checkbox for bulk select
- Drag handle (hover)
- ▶ Play button
- 🗑 Delete from WL (single item, with confirm)
- 🏷 Tag editor (inline popover)
- ✓ Mark as watched toggle
- ↑ Move to top

**Bulk toolbar (appears when ≥1 item selected):**
- Delete selected
- Tag selected (apply/remove tag to all)
- Mark selected as watched
- Move selected to top of queue

---

## Sort / Filter Options

**Sort:**
- Date added (default, desc)
- Duration (asc/desc)
- Channel (alpha)
- Watch progress (least watched first)
- Custom (drag order)

**Filter:**
- By watch progress: Unwatched / In Progress (1–89%) / Watched (≥90%)
- By channel (multi-select from channels present in list)
- By tag (multi-select)

**Search:** Client-side fuzzy match on title and channel name.

---

## Known Constraints / Gotchas

1. **Watch Later is a special playlist** — `playlistId = "WL"` only works for the authenticated user's own session. The YouTube API does not let you read someone else's WL.
2. **Reorder does not sync back to YouTube** — YouTube's API does not support reordering the Watch Later playlist specifically (only regular playlists). Custom order is local-only in Supabase.
3. **Video duration requires a second API call** — `playlistItems.list` does not return duration. You must call `videos.list` with the video IDs to get `contentDetails.duration` (ISO 8601 format, needs parsing). Batch this in groups of 50 to minimize quota usage.
4. **NextAuth token refresh** — Google OAuth tokens expire in 1 hour. Configure NextAuth to refresh the access token using the refresh token in the `jwt` callback, and store the refreshed token back in the session.
5. **IFrame API requires global `onYouTubeIframeAPIReady` callback** — Wire this up carefully in a `useEffect` with proper cleanup to avoid double-loading the script.

---

## Phase Plan

### Phase 1 — Auth + Sync
- [ ] Next.js project init with Tailwind
- [ ] NextAuth with Google provider, YouTube scopes
- [ ] `/api/queue` route: fetch WL from YouTube, sync into Supabase
- [ ] Login page → redirect to `/queue` after auth
- [ ] Display raw synced list (no styling yet)

### Phase 2 — List UI
- [ ] `VideoRow` component with thumbnail, title, channel, duration
- [ ] Watch progress bar (from `video_meta.watched_sec` / `videos.duration_sec`)
- [ ] Tag display + inline `TagEditor`
- [ ] Sort and filter controls
- [ ] Client-side search
- [ ] Checkbox multi-select
- [ ] `BulkToolbar` component

### Phase 3 — Player + Watch Tracking
- [ ] `YouTubePlayer` wrapper using IFrame API
- [ ] `WatchTimeTracker` with interval polling and Supabase flush
- [ ] Resume prompt when `last_position_sec > 0`
- [ ] Mark fully watched on completion

### Phase 4 — Mutations
- [ ] Single delete (API route → YouTube `playlistItems.delete` + Supabase remove)
- [ ] Bulk delete with quota warning
- [ ] Move to top (update `position` in Supabase)
- [ ] dnd-kit drag-to-reorder with position persistence

### Phase 5 — Polish
- [ ] Quota usage indicator in header
- [ ] Loading skeletons
- [ ] Error states (API down, quota exceeded)
- [ ] Keyboard shortcuts (space to play/pause, `d` to delete selected, etc.)
- [ ] Mobile responsive layout

---

## Dependencies to Install

```bash
npm install next-auth @auth/supabase-adapter
npm install @supabase/supabase-js
npm install @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities
npm install date-fns          # ISO 8601 duration parsing + date formatting
npm install clsx tailwind-merge
```
