# Personal Secretary — Design Spec

Date: 2026-11-11
Status: Draft for review

## 1. Goal

A personal "secretary" app. The user jots short notes throughout the day
(mainly from LINE) and, whenever they want, asks the app to produce a daily
summary that they read and act on in a web app.

### Success criteria
- Sending a text message to the LINE OA saves it as a note within a second
  and gets a "จดแล้ว ✓" reply.
- Pressing "ขอสรุป" for a day produces: one overview paragraph plus items
  grouped into To-do / Appointment / Idea / Memo.
- To-do items can be ticked done in the web app; ticks survive regenerating
  the summary when the item text is unchanged.
- Past days can be browsed with their notes and summaries.

### Decisions already made (from brainstorming)
| Topic | Decision |
|---|---|
| Channels | Type notes via LINE OA; read summaries and history in the web app |
| Summary delivery | On demand only (user presses a button). No scheduler, no push messages |
| Summary format | Overview paragraph + categorized items |
| Users | Single user for now; data model scoped per user (RLS) so more users can be added later |
| Stack | Next.js (App Router) on Vercel, Supabase (Postgres + Auth), Claude API, React + MUI, TypeScript |
| Language | UI and summaries in Thai |

### Out of scope (v1)
Edit/delete notes in the web app, tags, search, image/audio/sticker notes
from LINE, pushing summaries to LINE, LINE Login (OAuth), multi-user signup.

## 2. Architecture

```
LINE chat ──webhook──▶ /api/line/webhook ──▶ Supabase (notes)
                                                ▲
Web (React + MUI) ──▶ Server Actions / API ─────┤
   [ขอสรุป] ──▶ /api/summaries ──▶ Claude ──────┘ (summaries, summary_items)
```

One Next.js app serves the UI, the LINE webhook and the summarize endpoint.
Secrets (LINE channel secret/token, Claude API key, Supabase service role key)
live only in server-side environment variables.

Units and responsibilities:
- `lib/time` — Bangkok day range calculation. No dependencies.
- `lib/line` — signature verification, reply helper, message parsing.
- `lib/linking` — generate/hash/verify link codes, attempt limiting.
- `lib/summarize` — build prompt, call Claude, validate output, merge into DB.
- `lib/supabase` — server client (service role) and request-scoped client (anon + session).
- `app/` — pages, server actions, route handlers (thin; call the libs above).

## 3. Data model (Supabase, RLS on every table)

All user-owned tables carry `user_id` and an RLS policy limiting rows to
`auth.uid() = user_id`. The LINE webhook uses the service role key
(there is no user session) and must always set `user_id` explicitly.

- `profiles`: `user_id` (PK, FK auth.users), `line_user_id` (unique, nullable),
  `timezone` (default `Asia/Bangkok`).
- `notes`: `id`, `user_id`, `content`, `source` (`line` | `web`), `created_at`.
- `summaries`: `id`, `user_id`, `date`, `overview`, `generated_at`,
  `note_count`. Unique `(user_id, date)`.
- `summary_items`: `id`, `summary_id`, `category`
  (`todo` | `appointment` | `idea` | `memo`), `text`, `due_at` (nullable),
  `done` (bool, default false).
- `line_link_codes`: `id`, `user_id`, `code_hash`, `expires_at`, `used_at`,
  `failed_attempts`.
- `line_events`: `event_id` (PK), `received_at` — webhook idempotency.

Rules:
- A "day" is 00:00–23:59 in the profile timezone (default Bangkok), never UTC.
- Raw notes are the source of truth; summaries can always be regenerated.

## 4. LINE webhook — `POST /api/line/webhook`

1. Verify `x-line-signature` (HMAC-SHA256 of the raw body with the channel
   secret). Invalid → 401, no side effects.
2. For each event: handle only `message` events of type `text`. Other message
   types get a short reply: "ตอนนี้รับเฉพาะข้อความตัวอักษรนะ".
3. Skip events whose `webhookEventId` is already in `line_events`
   (LINE retries); otherwise record it.
4. Look up `profiles` by `line_user_id`:
   - Linked → insert into `notes` (`source = line`), reply "จดแล้ว ✓".
   - Not linked → store nothing; reply with linking instructions.
   - Message matching `link <6 digits>` → run the linking flow below
     (checked before the normal note path).
5. Respond 200 quickly. Only one insert per event, so no queue is needed.

Only reply messages are used (no push), so no message quota is consumed.

### Account linking (one time)
1. User signs in on the web (Supabase Auth, email magic link).
2. Settings → "ผูก LINE" creates a 6-digit code (10-minute expiry, single use,
   stored hashed).
3. User sends `link 482913` to the LINE OA.
4. Webhook verifies the code (unexpired, unused, attempts below the limit),
   sets `profiles.line_user_id`, marks the code used, replies "ผูกบัญชีแล้ว 🎀".
5. Wrong codes increment a per-LINE-user and per-code attempt counter; after
   5 failures the code is invalidated.

## 5. Summarization — `POST /api/summaries { date }`

Requires a signed-in session.

1. Compute the Bangkok day range for `date`; load that day's notes in time
   order.
2. No notes → return "วันนี้ยังไม่มีโน้ต"; do not call Claude.
3. Call Claude with the notes (each with its timestamp so relative phrases like
   "พรุ่งนี้" resolve correctly), forcing structured output:
   ```ts
   {
     overview: string;
     items: { category: 'todo'|'appointment'|'idea'|'memo';
              text: string; due_at?: string }[];
   }
   ```
4. Validate with zod; on failure retry once, then fail.
5. In one transaction upsert `summaries` on `(user_id, date)`, replace
   `summary_items`, and carry over `done = true` for items whose `text` equals
   a previously done item.
6. Return the new summary.

Prompt: Thai, instructs the model to stay faithful to the notes, not invent
details, and keep the notes' original language. Notes are passed as data in a
delimited block; instructions inside notes are not followed.

Model: env `SUMMARY_MODEL`, default `claude-sonnet-5-5` (`claude-haiku-5-5` is
a cheaper alternative).

### Error handling and limits
- Claude timeout / rate limit / API error → friendly error; the existing
  summary is left untouched.
- Rate limit per user: at most 20 summaries per day and a 10-second cooldown.
- Input cap: if the day's notes exceed a size ceiling, drop the oldest first
  and report that truncation happened in the response.

## 6. Web app (React + MUI, mobile-first)

Pages:
- `/login` — email magic link.
- `/` (today) — multiline note composer (Ctrl/Cmd+Enter saves), today's notes,
  "ขอสรุป" button, summary card (overview + category groups with Chip colours;
  Checkbox toggles `done`).
- `/day/[date]` — history list of days with notes/summaries; opening a day
  shows its summary and original notes, with the option to regenerate.
- `/settings` — generate link code, show LINE link status, sign out.

Technical notes:
- MUI via `@mui/material-nextjs` (`AppRouterCacheProvider`); light/dark theme.
- Thai font: Noto Sans Thai (swappable).
- Loading: `LinearProgress`/Skeleton while summarizing; errors in `Snackbar`.
- `done` toggle: optimistic update via Server Action, rolled back on failure.
- Route protection through Supabase session middleware; webhook route is
  exempt (authenticated by LINE signature instead).

## 7. Security

- Service role key and LINE/Claude secrets are server-only env vars.
- RLS enforced on all tables; browser uses anon key + user session.
- LINE signature verified on the raw request body before any processing.
- Link codes hashed, short-lived, single-use, attempt-limited.
- Notes are untrusted input to the model (prompt-injection hygiene above).

## 8. Testing

- Vitest (pure logic): Bangkok day range, LINE signature verification, link
  code create/verify/expiry/attempt limit, `done` merge on regeneration,
  zod schema validation and retry.
- Route tests with mocked Claude and Supabase: webhook (valid/invalid
  signature, duplicate event, unlinked user, link command, non-text message)
  and `/api/summaries` (no notes, success, Claude failure keeps old summary,
  rate limit).
- Playwright smoke test: sign in → add note → request summary → tick a to-do.

## 9. Configuration

Environment variables: `NEXT_PUBLIC_SUPABASE_URL`,
`NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`,
`LINE_CHANNEL_SECRET`, `LINE_CHANNEL_ACCESS_TOKEN`, `ANTHROPIC_API_KEY`,
`SUMMARY_MODEL` (optional).

Manual setup required from the owner (not automatable here): create the LINE
Messaging API channel and set its webhook URL to the deployed
`/api/line/webhook`; create the Supabase project; create the Vercel project and
set the variables above.

## 10. Suggested build order

1. Next.js + MUI scaffold, Supabase schema + RLS, magic-link login.
2. Web notes: composer + today's list.
3. Summarization endpoint + summary card + `done` toggle.
4. History pages.
5. LINE webhook + account linking + settings page.
