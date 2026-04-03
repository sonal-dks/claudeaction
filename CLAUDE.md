# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UIGen** — an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates React/JSX files into a virtual file system, rendered live in an iframe.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server (Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all tests (Vitest)
npm run db:reset     # Reset database
```

All Next.js commands require `NODE_OPTIONS='--require ./node-compat.cjs'` (already set in npm scripts via a Node compatibility shim).

To run a single test file:
```bash
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx
```

**Environment:** Set `ANTHROPIC_API_KEY` in `.env`. If empty, the app falls back to a `MockLanguageModel` that returns static demo components — useful for development without API costs.

## Architecture

### Virtual File System
All generated code lives in memory (`VirtualFileSystem` class in `src/lib/file-system.ts`). Nothing is written to disk. Files are serialized as JSON and persisted in the `Project.data` column in SQLite.

### AI Generation Flow
```
User message → ChatInterface → Vercel AI SDK useChat → POST /api/chat
                                                              ↓
                                              Claude with two tools:
                                              - str_replace_editor (create/view/edit files)
                                              - file_manager (rename/delete files)
                                                              ↓
                                              Tool calls handled in FileSystemProvider
                                                              ↓
                                              PreviewFrame re-renders iframe
```

- `src/app/api/chat/route.ts` — main AI endpoint, streams responses
- `src/lib/provider.ts` — returns Claude or MockLanguageModel
- `src/lib/prompts/generation.tsx` — system prompt; instructs Claude to create `/App.jsx` as entry point and use Tailwind for styling
- `src/lib/tools/` — tool definitions for `str_replace_editor` and `file_manager`

### Live Preview
`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) renders in a sandboxed iframe. `src/lib/transform/jsx-transformer.ts` compiles JSX via Babel and builds an import map using esm.sh for external dependencies.

### State Management
Two React Contexts (no external state library):
- `ChatProvider` (`src/lib/contexts/chat-context.tsx`) — chat messages, AI interaction via Vercel AI SDK
- `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) — virtual FS state, tool call handling

### Auth
JWT in httpOnly cookies (7-day sessions). Anonymous use is supported — users can generate components without signing in. Registered users get project persistence. Logic in `src/lib/auth.ts`; middleware in `src/middleware.ts`.

### Database
SQLite via Prisma. Two models: `User` and `Project`. `Project.messages` stores chat history as JSON; `Project.data` stores the serialized virtual file system.

## Database Schema

The schema is defined in `prisma/schema.prisma`. Reference it for all questions about data structure. Two models:

- **User**: `id` (cuid), `email` (unique), `password`, `createdAt`, `updatedAt`, `projects[]`
- **Project**: `id` (cuid), `name`, `userId` (nullable — anonymous projects have no owner), `messages` (JSON string, chat history), `data` (JSON string, serialized VirtualFileSystem), `createdAt`, `updatedAt`, `user` (optional relation, cascade delete on user removal)

## Key Conventions

- Components use `@/` path alias (maps to `src/`)
- Claude is instructed to create `/App.jsx` as the component entry point
- Tailwind CSS v4 for all styling; no hardcoded inline styles
- UI primitives from Radix UI (in `src/components/ui/`)
- Tests use Vitest + React Testing Library; test files live in `__tests__/` subdirectories next to the components they test
