# 💬 World Tech VI AI Chat Support
An AI-powered customer support chat widget for World Tech VI, a Saint Croix electronics retailer — built with Next.js and streamed responses from OpenAI's GPT-4o-mini.
---
## 🛠️ Tech Stack
- Next.js 14 (App Router)
- React 18, Material UI (MUI) 5, Emotion
- OpenAI Node SDK v4 (`gpt-4o-mini`, streaming completions)
- ESLint (`eslint-config-next`)
- `next/font` (Google Inter)
---
## ✅ Prerequisites
- Node.js 18.17 or later
- npm (or yarn/pnpm/bun)
- An [OpenAI API key](https://platform.openai.com/api-keys) with access to `gpt-4o-mini`
---
## 🚀 Setup
### 1. Clone the repo
```bash
git clone git@github.com:xAdhm/World-Tech-VI-AI-Chat-Support.git
cd World-Tech-VI-AI-Chat-Support
```
### 2. Install dependencies
```bash
npm install
```
### 3. Configure environment variables
Create a file named `.env.local` in the root of the project:
```
OPENAI_API_KEY=sk-your-openai-api-key
```
The OpenAI SDK (`new OpenAI()`) reads this variable automatically server-side — no extra config needed. Never commit this file to GitHub — it's already in `.gitignore`.
### 4. Start the server
```bash
npm run dev
```
The app runs on **port 3000**. Open [http://localhost:3000](http://localhost:3000) to see the chat UI.

Other scripts: `npm run build` (production build), `npm run start` (serve production build), `npm run lint` (ESLint).
---
## 📡 API Endpoints
### 💬 `POST /api/chat`
Streams a chat completion from OpenAI back to the client as plain text. The handler injects a fixed system prompt (World Tech VI's support persona, store address, hours, etc. — see `app/api/chat/route.js`) ahead of whatever conversation history is sent.

**Auth required:** None. The endpoint is unauthenticated; it relies solely on the server-side `OPENAI_API_KEY`. There is no rate limiting, so treat this as a known gap before deploying publicly.

**Request body:** a raw JSON array of message objects (not wrapped in an object key):
```json
[
  { "role": "user", "content": "Where is the store located?" }
]
```

| Field | Type | Description |
|---|---|---|
| `role` | string | `"user"` or `"assistant"` |
| `content` | string | The message text |

**Response:** `200 OK`. The body is a streamed sequence of UTF-8 text chunks (raw assistant reply text, not JSON/SSE-wrapped). The client reads it with a `ReadableStream` reader and concatenates chunks until the stream closes.

> ⚠️ The request body is a **bare array**, not an object — don't wrap it in `{ "messages": [...] }`.

**Possible errors:**

| Scenario | Behavior |
|---|---|
| Missing/invalid `OPENAI_API_KEY` | The OpenAI SDK call throws before streaming starts; Next.js returns a generic `500` since the route has no top-level try/catch around `openai.chat.completions.create`. |
| Malformed request body (not valid JSON / not an array) | `req.json()` throws or the spread into `messages` fails, surfacing as a `500`. |
| Error mid-stream (e.g. OpenAI connection drop) | Caught inside the `ReadableStream`'s `start()`; the stream is errored out via `controller.error(err)`, which the client read loop does not explicitly handle. |
---
## 🗃️ Data Models
This project has no database — nothing is persisted server-side. The only structured shape in play is the in-memory chat message used by both the frontend state and the API payload:
```js
{ role: 'user' | 'assistant' | 'system', content: string }
```
---
## 📝 Notes for Developers
- All chat messages use the shape `{ role, content }` — there's no separate request/response wrapper object
- **Dead/misleading code:** `app/page.js` sets an `Authorization: Bearer ${process.env.OPENAI_API_KEY}` header on the client-side `fetch('/api/chat')` call. This does nothing — Next.js only exposes env vars prefixed `NEXT_PUBLIC_` to the browser, so this always evaluates to `Bearer undefined`. It's also unnecessary, since `app/api/chat/route.js` instantiates `new OpenAI()` server-side, which reads `OPENAI_API_KEY` directly from the server environment. Safe to remove
- **No input validation:** the `/api/chat` route trusts the request body to be a well-formed array of `{ role, content }` objects
- **No auth/rate limiting:** anyone with the deployed URL can call `/api/chat` and consume your OpenAI quota. Add auth or rate limiting before going to production
- **System prompt is hardcoded:** store info (address, hours) and persona instructions live as a template literal in `app/api/chat/route.js`. Update it there if business details change — there's no CMS or config file backing it
- **Typing animation is client-side only:** the "typewriter" effect in `app/page.js` replays the already-fully-received response character by character via `setInterval`; it does not render tokens as they actually stream in from the server
- **Single page, no routing:** the entire UI lives in `app/page.js`; there are no other routes besides the `/api/chat` API route
- The `.env.local` file must be present with a valid `OPENAI_API_KEY` or all AI calls will fail
