# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in chat, and the AI generates React code using Claude. Components are stored in a virtual file system (no disk writes) and previewed in real-time using an iframe with import maps and Babel transformation.

## Common Commands

### Development
```bash
npm run dev              # Start dev server with Turbopack
npm run dev:daemon       # Start dev server in background, log to logs.txt
```

### Testing
```bash
npm test                 # Run Vitest tests
npm test -- path/to/file.test.ts   # Run specific test file
```

### Database
```bash
npm run setup            # Install deps + generate Prisma client + run migrations
npm run db:reset         # Reset database (force)
npx prisma generate      # Regenerate Prisma client after schema changes
npx prisma migrate dev   # Create and apply new migration
```

### Build & Lint
```bash
npm run build            # Build for production
npm run lint             # Run ESLint
```

## Architecture

### Virtual File System
The core abstraction is `VirtualFileSystem` (src/lib/file-system.ts), which manages files in-memory without disk writes. It provides:
- File/directory CRUD operations with automatic parent directory creation
- Path normalization (always absolute paths starting with `/`)
- Serialization/deserialization for database persistence
- Operations: `createFile`, `updateFile`, `deleteFile`, `rename`, `readFile`, `exists`

The file system is provided via React Context (`FileSystemProvider` in src/lib/contexts/file-system-context.tsx) and synchronized between:
1. Client state (React Context)
2. Server-side generation (AI tool calls)
3. Database (serialized in `Project.data` JSON column)

### AI Integration Flow
1. User sends message → `POST /api/chat/route.ts`
2. Server reconstructs VirtualFileSystem from `files` parameter
3. AI SDK `streamText` with two tools:
   - `str_replace_editor`: Create/edit files via exact string replacement
   - `file_manager`: Rename/delete files and folders
4. Tools mutate server-side VirtualFileSystem instance
5. `onFinish` callback saves conversation messages and serialized file system to database
6. Client receives streaming response with tool calls
7. `ChatContext.handleToolCall()` applies same mutations to client-side VirtualFileSystem
8. UI re-renders to reflect file changes

### Preview System
Preview rendering (src/components/preview/PreviewFrame.tsx):
1. Watch file system changes via `refreshTrigger`
2. Transform all .jsx/.tsx files with Babel (src/lib/transform/jsx-transformer.ts)
3. Create blob URLs for each transformed module
4. Generate import map mapping file paths → blob URLs
5. Support `@/` alias for root-relative imports
6. Handle third-party imports via esm.sh CDN
7. Inject Tailwind CSS via CDN
8. Render iframe with import map + entry point (App.jsx or index.jsx)

### Authentication
JWT-based auth (src/lib/auth.ts):
- Sessions stored in httpOnly cookies with 7-day expiry
- `createSession()`, `getSession()`, `deleteSession()` for server actions
- Middleware (src/middleware.ts) protects routes requiring auth
- Anonymous users: projects not saved, files only in memory
- Authenticated users: projects persisted to database with userId

### Data Models
Reference `prisma/schema.prisma` for the complete database structure and relationships.

Key models:
- **User**: id, email, password (bcrypt hashed), timestamps
- **Project**: id, name, userId (nullable for anon), messages (JSON), data (JSON with serialized VirtualFileSystem), timestamps
- SQLite database, Prisma client generated to `src/generated/prisma`

### Provider Pattern
Language model selection (src/lib/provider.ts):
- Checks `ANTHROPIC_API_KEY` environment variable
- If present: uses `claude-haiku-4-5` via Anthropic SDK
- If absent: uses `MockLanguageModel` that returns static component templates
- Mock provider demonstrates the app without API costs

## Code Style

### Comments
Only add comments where the code is complex or non-obvious. Prefer self-documenting code with clear variable and function names over excessive comments.

## Key Patterns

### File System Operations
Always use VirtualFileSystem methods, never direct file I/O. All paths must be absolute (start with `/`).

### Tool Call Synchronization
When AI tools modify files, ensure both server-side (route.ts) and client-side (file-system-context.tsx) apply identical mutations.

### Import Resolution
The transformer supports:
- Absolute paths: `/components/Button.jsx`
- @/ alias: `@/components/Button.jsx`
- Relative imports: `./Button` from `/components/Form.jsx`
- Third-party packages: `lucide-react` → esm.sh CDN

### Database Serialization
When saving projects:
- Messages: JSON array (exclude system messages)
- Data: `fileSystem.serialize()` returns Record<string, FileNode>

### Testing
- Vitest with jsdom environment
- React Testing Library for components
- Tests colocated in `__tests__` directories
- Mock file system in tests by passing controlled VirtualFileSystem instance to FileSystemProvider
