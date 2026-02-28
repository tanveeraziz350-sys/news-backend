# replit.md

## Overview

This is a **News Manager** application — a role-based content management system for news articles. It supports four user roles (superadmin, admin, editor, reporter) with a workflow pipeline centered on the Super Admin. The workflow is: reporters submit news → super admin reviews and selects media to forward to editor → editor downloads, edits, and re-uploads → super admin publishes to website. The app also includes user management for admins and superadmins.

The project follows a full-stack TypeScript monorepo pattern with a React frontend and Express backend, sharing types and schemas through a `shared/` directory.

## User Preferences

Preferred communication style: Simple, everyday language.

## System Architecture

### Directory Structure
- `client/` — React SPA (Vite-based)
- `server/` — Express API server
- `shared/` — Shared schemas and types (Drizzle ORM + Zod)
- `migrations/` — Drizzle-generated database migrations
- `uploads/` — File uploads directory (images/videos for news articles)
- `script/` — Build scripts

### Frontend Architecture
- **Framework**: React with TypeScript, bundled by Vite
- **Routing**: `wouter` (lightweight client-side router)
- **State/Data Fetching**: TanStack React Query for server state
- **UI Components**: shadcn/ui (new-york style) built on Radix UI primitives with Tailwind CSS
- **Styling**: Tailwind CSS with CSS variables for theming (light/dark mode support)
- **Auth**: JWT tokens stored in localStorage, managed via React Context (`AuthProvider`)
- **API Layer**: Custom `authFetch`/`authJsonFetch` helpers in `client/src/lib/api.ts` that automatically attach Bearer tokens

### Role-Based Pages
- **Super Admin**: Unified Dashboard (`/superadmin`) with tabs: Overview, All News, Submit, Users
- **Reporter**: Submit News (`/submit`), My Submissions (`/my-news`)
- **Editor**: Assigned Articles (`/pending`), Completed (`/reviewed`)
- **Admin**: Approved News (`/approved`), Published News (`/published`), Manage Users (`/users`)

### Backend Architecture
- **Runtime**: Node.js with Express
- **Language**: TypeScript (executed via `tsx` in dev, compiled via esbuild for production)
- **Authentication**: JWT-based (bcryptjs for password hashing, jsonwebtoken for token generation/verification)
- **Role Middleware**: `authMiddleware` validates tokens, `roleMiddleware` restricts endpoints by user role
- **File Uploads**: Multer with disk storage in `uploads/` directory, served as static files. Supports multiple files: up to 10 images, 5 videos, and 5 audio files per article.
- **API Pattern**: RESTful JSON API under `/api/` prefix
- **News Workflow**: Reporter submits (status: pending) → Super Admin selects media and forwards to editor (status: forwarded_to_editor) → Editor downloads, edits, re-uploads (status: edited) → Super Admin publishes (status: published)
- **Super Admin Workflow**: Sees all pending submissions, can select which images/videos/audios to forward to editor, can also directly publish. After editor returns edited article, super admin reviews and publishes.
- **Editor Workflow**: Receives forwarded articles, downloads for offline editing, re-uploads edited version back to super admin. No approve/reject capability — only edit and send back.
- **Database Seeding**: Automatic seeding on first run with default admin/editor/reporter accounts

### Database
- **Database**: PostgreSQL (required via `DATABASE_URL` environment variable)
- **ORM**: Drizzle ORM with `drizzle-kit` for schema management
- **Schema Push**: Use `npm run db:push` to sync schema to database (no migration files needed for development)
- **Tables**:
  - `users` — id (UUID), name, email (unique), password (hashed), role (enum: admin/editor/reporter)
  - `news` — id (UUID), title, description, images (text[]), videos (text[]), audios (text[]), status (enum: pending/approved/rejected/published/forwarded_to_editor/edited), createdBy (FK to users), timestamps
- **Enums**: `user_role` and `news_status` are PostgreSQL enums

### Build & Deployment
- **Dev**: `npm run dev` — runs Express + Vite dev server with HMR
- **Build**: `npm run build` — Vite builds client to `dist/public/`, esbuild bundles server to `dist/index.cjs`
- **Production**: `npm start` — serves the built app from `dist/`
- **Type Check**: `npm run check`

### Key Design Decisions
1. **JWT over Sessions**: Stateless auth using Bearer tokens rather than server-side sessions. Simpler to scale, tokens stored in localStorage on the client.
2. **Shared Schema**: Drizzle schema in `shared/schema.ts` provides both database types and Zod validation schemas, ensuring type safety across the full stack.
3. **File Upload to Disk**: News articles support image and video uploads via Multer, stored directly on the filesystem (not cloud storage). Maximum file size: 100MB per file, with upload progress tracking via XMLHttpRequest on the frontend. HTTP server timeout set to 10 minutes for uploads.
4. **Role-Based Access Control**: Three distinct roles with separate page views and API middleware enforcement.
5. **Forgot Password**: Email OTP-based password reset via Gmail SMTP (nodemailer). OTPs stored in `password_resets` table, expire in 10 minutes.
6. **Auto-Delete Published News**: Published articles are automatically deleted after 24 hours. A cleanup scheduler runs every 30 minutes, removing expired articles and their associated media files from disk.
7. **Video Download**: Visitors can download news videos via the `GET /api/public/download-video?url=...` endpoint with proper download headers.
8. **Build Output**: Vite builds to `dist/public/` then the build script moves files to `dist/client/` to avoid conflict with Replit's deployment `publicDir` setting. `server/static.ts` looks for `dist/client` first, then falls back to `dist/public`.
9. **Publish Email Notification**: When Super Admin publishes an article (via `/publish` or `/direct-publish`), a Gmail notification is sent to the reporter with article title, publish time, and a direct link to their backup page.
10. **Reporter Backup Page**: Public page at `/article-backup/:id` — professional newspaper-style backup of a published article with all media, byline, and Print/Download PDF button. No login required.

## External Dependencies

### Required Services
- **PostgreSQL Database**: Connected via `DATABASE_URL` environment variable. Used for all persistent data storage.

### Environment Variables
- `DATABASE_URL` — PostgreSQL connection string (required)
- `SESSION_SECRET` — Used as JWT signing secret (required)

### Key NPM Packages
- **drizzle-orm** + **drizzle-kit** — Database ORM and schema tooling
- **pg** — PostgreSQL client
- **jsonwebtoken** + **bcryptjs** — Authentication
- **multer** — File upload handling
- **zod** + **drizzle-zod** — Runtime validation
- **@tanstack/react-query** — Client-side data fetching
- **wouter** — Client-side routing
- **shadcn/ui** components (Radix UI + Tailwind CSS)
- **cors** — Cross-origin resource sharing middleware