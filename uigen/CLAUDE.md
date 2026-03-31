# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Code Style

Use comments sparingly. Only write comments for complex code.

## Commands

```bash
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run start        # Start production server
npm run test         # Run tests with Vitest
npm run lint         # Run ESLint
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset database (prisma migrate reset --force)
```

Run a single test file: `npx vitest run <path-to-test-file>`

## Environment

Requires `ANTHROPIC_API_KEY` in `.env`. Without it, the app falls back to a `MockLanguageModel` with static component templates (Counter, ContactForm, Card).

Database: SQLite at `./prisma/dev.db`. Prisma client is generated to `src/generated/prisma`. The schema at `prisma/schema.prisma` is the source of truth for all data structures — reference it whenever you need to understand stored data.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat interface; Claude generates them using tool calls that manipulate a **virtual file system**, and the result is rendered in a live iframe preview.

### Data Flow

```
User chat message
  → ChatContext (useChat from @ai-sdk/react)
  → POST /api/chat (streams via Vercel AI SDK)
  → Claude claude-haiku-4-5 with tool use
  → Tool calls: str_replace_editor / file_manager
  → FileSystemContext updates VirtualFileSystem
  → PreviewFrame regenerates iframe with Babel-transpiled output
```

### Key Subsystems

The core subsystems are: **VirtualFileSystem** (`src/lib/file-system.ts`) — in-memory file CRUD serialized to SQLite for persistence; **AI Tools** (`src/lib/tools/`) — `str_replace_editor` for view/create/edit and `file_manager` for rename/delete on the virtual FS; **JSX Transformer** (`src/lib/transform/jsx-transformer.ts`) — Babel-based runtime transpiler that resolves `@/` aliases, builds esm.sh import maps, generates blob URLs per file, and injects Tailwind via CDN; **System Prompt** (`src/lib/prompts/generation.tsx`) — instructs Claude to use `/App.jsx` as the entry point with Tailwind and `@/` aliases; **AI Provider** (`src/lib/provider.ts`) — Anthropic Claude with up to 40 tool steps, falling back to `MockLanguageModel` without an API key; **Preview** (`src/components/preview/PreviewFrame.tsx`) — sandboxed iframe rendering a full HTML document from the virtual FS; and **Auth** (`src/lib/auth.ts`) — JWT sessions in HTTP-only cookies with bcrypt password hashing, enforced by `src/middleware.ts`.

### UI Layout

`src/app/main-content.tsx` sets up a resizable three-panel layout:
1. **Left** — Chat interface (`src/components/chat/`)
2. **Center** — Monaco code editor + virtual file tree (`src/components/editor/`)
3. **Right** — Live preview iframe (`src/components/preview/`)

State is managed through two React contexts: `ChatContext` and `FileSystemContext`.

### Path Aliases

`@/*` maps to `./src/*` (configured in `tsconfig.json`).