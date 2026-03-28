# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Production build
npm run build

# Run tests
npm run test

# Lint
npm run lint

# Reset database
npm run db:reset
```

## Architecture

UIGen is a three-panel AI-powered React component generator: a chat panel on the left, and a tabbed preview/code editor on the right.

### Core Data Flow

1. User sends a message in the **Chat** panel
2. `POST /api/chat` streams a response from Claude via the Vercel AI SDK (`streamText`)
3. Claude calls tools (`str_replace_editor`, `file_manager`) to manipulate a **virtual file system**
4. Tool results update `FileSystemContext`, which triggers a re-render of the **Preview** iframe
5. The preview iframe transpiles JSX → ES5 via Babel standalone and loads React from esm.sh CDN
6. On completion, the project (messages + serialized VFS) is saved to the database

### Key Abstractions

- **`VirtualFileSystem`** (`src/lib/file-system.ts`) — in-memory Map-based FS, serialized as JSON and persisted in the `Project.data` column
- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — bridges AI tool calls to React state; defines the tool handlers Claude calls
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — manages messages and streaming state
- **`PreviewFrame`** (`src/components/preview/PreviewFrame.tsx`) — sandboxed iframe; builds an import map so `@/` aliases resolve to virtual FS files
- **`provider.ts`** (`src/lib/provider.ts`) — returns a real `AnthropicProvider` or a `MockLanguageModel` if `ANTHROPIC_API_KEY` is absent

### AI Tools

Claude is given two tools at `/api/chat`:
- `str_replace_editor` — view, create, str_replace, or insert within virtual files
- `file_manager` — rename or delete virtual files

Every generated project must have `/App.jsx` as its entry point (enforced in the system prompt at `src/lib/prompts/generation.tsx`).

### Auth & Persistence

- JWT sessions stored in HTTP-only cookies (via `jose`); passwords hashed with `bcrypt`
- Anonymous users can generate components without logging in; their work is tracked in `sessionStorage` (`src/lib/anon-work-tracker.ts`) and migrated to a named project on sign-in
- Database: SQLite via Prisma; schema has `User` and `Project` (messages + VFS data stored as JSON strings)
- Middleware (`src/middleware.ts`) guards `/api/projects` and `/api/filesystem` routes

### Environment

Requires `ANTHROPIC_API_KEY` in `.env`. Without it the app falls back to a mock provider that generates static example components.
