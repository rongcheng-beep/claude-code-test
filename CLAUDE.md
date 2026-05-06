# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Common Commands

```bash
npm run setup       # First-time setup: install deps, generate Prisma client, run migrations
npm run dev         # Dev server with Turbopack
npm run build       # Production build
npm run lint        # ESLint
npm run test        # Vitest (jsdom environment)
npm run db:reset    # Force-reset the SQLite database
```

Single test: `npx vitest run <test-file-path>`

Dev and build scripts inject `NODE_OPTIONS='--require ./node-compat.cjs'` to patch `globalThis.localStorage/sessionStorage` for Node 25+ SSR compatibility.

## Environment

Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. If omitted, the app falls back to a mock LLM provider that replays a 4-step canned response — useful for UI development without spending tokens.

## Architecture

**UIGen** is a Next.js 15 (App Router) app that lets users describe React components in chat; Claude generates them in real time with a live preview.

### Key data flow

1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. API deserializes the virtual file system from the request body, injects the system prompt with an Anthropic ephemeral cache header, and calls `streamText()` via Vercel AI SDK with two tools: `str_replace_editor` and `file_manager`
3. Claude's tool calls (and text deltas) are streamed back to the frontend
4. `ChatContext` (`src/lib/contexts/chat-context.tsx`) receives the stream and dispatches tool calls to `FileSystemContext`
5. `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) mutates the in-memory `VirtualFileSystem` and increments `refreshTrigger`
6. `PreviewFrame` detects the trigger change, re-transforms all files via Babel, rebuilds an ES import map, and re-renders in a sandboxed `<iframe>`
7. On stream finish, if the user is authenticated, the API saves the updated messages + serialized file system to Prisma

### Virtual file system (`src/lib/file-system.ts`)

All generated code lives here — nothing ever writes to disk. The class wraps a `Map<path, FileNode>` and handles path normalization, recursive deletes, and renames. It serializes to a JSON `Record` for SQLite persistence and is sent in full on every `/api/chat` request (stateless API).

### AI tools (`src/lib/tools/`)

- `str-replace.ts` — four operations: `view` (with optional line range), `create`, `str_replace` (targeted patch), `insert` (line-based)
- `file-manager.ts` — `rename` and `delete` (recursive for directories)

The system prompt lives in `src/lib/prompts/generation.tsx`. It instructs Claude to use `@/` import aliases and the Shadcn/ui component library. Prompt caching (Anthropic ephemeral cache) is applied to the system message to reduce repeated-token cost.

### Preview (`src/components/preview/PreviewFrame.tsx`)

Pipeline: TSX/JSX → Babel standalone → blob URLs → ES import map → `<iframe srcdoc>`.

- CSS imports are stripped and injected as `<style>` tags
- Third-party imports (not `.`, `/`, or `@/`) resolve to `esm.sh/<package>`
- `@/` aliases resolve to blob URLs of the corresponding virtual file
- Entry point detection order: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx` → `/src/App.tsx` → first `.jsx/.tsx` found
- The iframe runs with `allow-scripts allow-same-origin allow-forms` sandbox attributes

### Auth (`src/lib/auth.ts`, `src/actions/index.ts`)

JWT sessions (jose, HS256, 7-day expiry) stored in an HTTP-only cookie. Anonymous use is fully supported — projects are only persisted to SQLite when the user is authenticated. `src/lib/anon-work-tracker.ts` tracks in-progress anonymous edits.

### Database schema

Two models in `prisma/schema.prisma` (SQLite, `prisma/dev.db`):
- `User` — id (cuid), email (unique), bcrypt password
- `Project` — id, name, userId (nullable, cascade delete), `messages` (JSON string), `data` (JSON string of serialized VirtualFileSystem)

Generated client outputs to `src/generated/prisma` (not the default location).

### State management

- `FileSystemContext` — virtual FS instance, selected file path, tool call handler, refresh trigger
- `ChatContext` — wraps Vercel AI SDK `useChat`, serializes the FS on each submit, routes incoming tool calls to `FileSystemContext`

### Path alias

`@/*` maps to `src/*` (configured in `tsconfig.json` and `components.json`).

## Tech stack

| Layer | Choice |
|-------|--------|
| Framework | Next.js 15, React 19 |
| Styling | Tailwind CSS v4, Radix UI primitives, Shadcn/ui |
| Editor | Monaco Editor |
| AI | Anthropic Claude via Vercel AI SDK (`streamText`) |
| DB | SQLite + Prisma ORM (client at `src/generated/prisma`) |
| Tests | Vitest + @testing-library/react (jsdom) |
