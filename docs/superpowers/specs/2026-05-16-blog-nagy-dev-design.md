# blog.nagy.dev Design Spec

## Context

Building a personal dev blog at nagy.dev with a modern tech stack. The goal is a performant, maintainable blog with interactive features (comments, newsletter, contact form) while using Sanity CMS for content management.

The architecture is **single-service**: one Next.js app serves the blog UI, the embedded Sanity Studio, and the API (as Route Handlers). Postgres holds dynamic data (comments, views, newsletter, contact messages); Sanity holds posts and taxonomy. No separate API service.

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend & API | Next.js 16, React 19, Tailwind CSS v4, shadcn/ui |
| CMS | Sanity (hosted) + embedded Studio at `/studio` |
| Content rendering | `@portabletext/react` + Shiki (build-time syntax highlighting) |
| Database | PostgreSQL 16 |
| ORM | Prisma |
| Validation | Zod |
| Deployment | Docker Compose |

> **Auth**: Sanity Studio uses Sanity's built-in login. No NextAuth or custom auth layer for `/studio`. Comments, newsletter, and contact are anonymous (no auth at all).

## Project Structure

```
blog.nagy.dev/
├── client/                              # the only application
│   ├── app/
│   │   ├── page.tsx                     # Home
│   │   ├── layout.tsx
│   │   ├── blog/
│   │   │   ├── page.tsx                 # Blog listing (paginated)
│   │   │   ├── [slug]/page.tsx          # Single post
│   │   │   ├── category/[category]/page.tsx
│   │   │   └── tag/[tag]/page.tsx
│   │   ├── contact/page.tsx
│   │   ├── studio/[[...index]]/page.tsx # Embedded Sanity Studio
│   │   └── api/
│   │       ├── comments/
│   │       │   ├── route.ts             # POST
│   │       │   └── [postId]/route.ts    # GET
│   │       ├── newsletter/
│   │       │   ├── route.ts             # POST
│   │       │   └── [email]/route.ts     # DELETE
│   │       ├── views/[postId]/route.ts  # POST + GET
│   │       └── contact/route.ts         # POST
│   ├── components/
│   │   ├── blog/
│   │   │   ├── PostCard.tsx
│   │   │   ├── PostHeader.tsx
│   │   │   ├── PortableTextRenderer.tsx # PortableText + Shiki code blocks
│   │   │   ├── TableOfContents.tsx
│   │   │   └── Pagination.tsx
│   │   ├── CommentSection.tsx
│   │   ├── NewsletterForm.tsx
│   │   └── ContactForm.tsx
│   ├── lib/
│   │   ├── prisma.ts                    # Prisma client singleton
│   │   ├── sanity.ts                    # Sanity client
│   │   ├── ratelimit.ts                 # In-memory token bucket
│   │   └── validation.ts                # Zod schemas
│   ├── prisma/
│   │   └── schema.prisma
│   └── sanity/
│       ├── schemas/
│       └── sanity.config.ts
│
├── docker-compose.yml
├── .env.example
└── README.md
```

## Data Models

### Sanity (Blog Content)

```typescript
// Post
{
  _id: string                // stable UUID — used to key comments and views
  title: string
  slug: { current: string }  // editable; not used as a foreign key
  excerpt: string
  body: PortableText
  coverImage: Image          // also serves as OG image
  author: Reference<Author>
  categories: Reference<Category>[]
  tags: Reference<Tag>[]
  publishedAt: datetime
  readingTime: number
}

// Category
{ name: string, slug: { current: string }, description: string }

// Tag
{ name: string, slug: { current: string } }

// Author
{ name: string, bio: string, avatar: Image }
```

### PostgreSQL (Interactive Features)

Comments and views are keyed by Sanity document `_id`, **not slug** — slugs are editable in Sanity, IDs are stable. Renaming a slug must not orphan comments or reset view counts.

```prisma
model Comment {
  id          Int      @id @default(autoincrement())
  postId      String   // Sanity document _id
  authorName  String
  authorEmail String
  content     String
  createdAt   DateTime @default(now())
  @@index([postId, createdAt])
}

model NewsletterSubscriber {
  id           Int      @id @default(autoincrement())
  email        String   @unique
  subscribedAt DateTime @default(now())
  isActive     Boolean  @default(true)
}

model PostView {
  postId    String @id           // Sanity document _id as PK
  viewCount Int    @default(0)
}

model ContactMessage {
  id        Int      @id @default(autoincrement())
  name      String
  email     String
  subject   String
  message   String
  createdAt DateTime @default(now())
  isRead    Boolean  @default(false)
}
```

## API Endpoints

All endpoints are Next.js Route Handlers under `client/app/api/*`. Same-origin — no CORS, no separate service.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/comments` | Submit a comment (auto-approved, honeypot + rate-limited) |
| GET | `/api/comments/:postId` | Get comments for a post (by Sanity `_id`) |
| POST | `/api/newsletter` | Subscribe email |
| DELETE | `/api/newsletter/:email` | Unsubscribe |
| POST | `/api/views/:postId` | Increment view count (atomic) |
| GET | `/api/views/:postId` | Get view count |
| POST | `/api/contact` | Submit contact form |

### View count integrity

- **Server**: `prisma.postView.upsert({ ..., update: { viewCount: { increment: 1 } }, create: { postId, viewCount: 1 } })`. Single atomic SQL statement; no read-modify-write race.
- **Client**: `<PostView />` component sets a `sessionStorage` flag after its first POST per post per session, so refreshes and re-mounts don't inflate counts.

## Spam Protection (v1)

Three layers, all required:

1. **Honeypot field**. Every public form (comment, newsletter, contact) includes a hidden `<input name="website">` styled `display:none`. Real users never fill it; most bots do. If the field is non-empty, the server returns `200 OK` and silently discards the payload — so the bot doesn't know to retry.

2. **Per-IP rate limiting** (`lib/ratelimit.ts`): in-memory token bucket keyed by `x-forwarded-for`. Default limits:
   - Comments: 5 / minute / IP
   - Newsletter: 3 / hour / IP
   - Contact: 2 / hour / IP

   Over-limit requests return `429`. In-memory is sufficient for a single-instance deploy; swap for Upstash/Redis if we ever scale horizontally.

3. **Strict Zod validation** (`lib/validation.ts`): every payload is parsed against a schema before touching Prisma. Email is lowercased and trimmed; `content` has min/max lengths; `postId` must match Sanity's ID format.

## Rendering Pipeline

- **PortableText**: `@portabletext/react` renders Sanity's portable text. A custom serializer for `code` blocks invokes Shiki.
- **Syntax highlighting**: **Shiki at build time** (inside a Server Component, no runtime cost). Zero JavaScript shipped to the client for highlighting; output is pre-rendered HTML with inline styles.
- **OG images**: each post's `coverImage` from Sanity is used directly as the `og:image` via Next.js's `generateMetadata` per route. No dynamic `ImageResponse` rendering in v1.
- **Image loader for Sanity CDN**: TBD during implementation. Either a custom Next.js loader that builds Sanity CDN URLs with `?w=&auto=format`, or just use Sanity's URL builder and skip `next/image` optimization for post images. Decide once we see real post layouts.

## Docker Setup

```yaml
services:
  client:
    build: ./client
    ports:
      - "3000:3000"
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/blog
      - SANITY_PROJECT_ID=${SANITY_PROJECT_ID}
      - SANITY_DATASET=${SANITY_DATASET}
      - SANITY_API_TOKEN=${SANITY_API_TOKEN}

  db:
    image: postgres:16
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=blog
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Two services only. No `version:` key (deprecated in Compose v2).

## Environment Variables

```bash
# .env.example

# Sanity
SANITY_PROJECT_ID=your_project_id
SANITY_DATASET=production
SANITY_API_TOKEN=your_token   # for Studio write operations

# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/blog
```

No `NEXT_PUBLIC_API_URL` — the API is same-origin.

## Blog Features

- **Listing page**: paginated posts, filter by category/tag
- **Post page**: reading time, published date, table of contents, comments, view count
- **Categories & Tags**: filter posts by category or tag
- **Comments**: anonymous (name + email), auto-approved, honeypot + rate-limited
- **Newsletter**: email subscription form, stored in Postgres
- **Contact form**: name, email, subject, message
- **Sanity Studio**: embedded at `/studio` (auth via Sanity)

## Future Considerations (Not in v1)

- RSS feed (`/feed.xml`)
- Sitemap generation
- Email notifications for new comments
- Unsubscribe tokens for newsletter (random `unsubscribeToken` column, used in email links instead of exposing the integer PK)
- Admin panel for comment moderation
- Search functionality
- Dynamic OG image generation (`ImageResponse`) if `coverImage`-as-OG turns out to be too plain
- Decide on Sanity CDN image loader strategy (custom Next loader vs. Sanity URL builder)

## Verification

1. `docker compose up` starts exactly two services: `client` and `db`.
2. `http://localhost:3000` loads home; `/studio` loads Sanity Studio and prompts for Sanity login.
3. Create a test post in Studio → appears on `/blog` and at `/blog/[slug]`.
4. Submit a comment → row appears in `Comment` keyed by Sanity `_id`.
5. **Spam test — honeypot**: POST to `/api/comments` with `website` field populated → returns `200`, no DB row.
6. **Spam test — rate limit**: submit 6 comments / minute from the same IP → 6th returns `429`.
7. **Slug rename test**: rename a post's slug in Sanity → existing comments still appear on the renamed post (proves `_id` keying).
8. **View debounce**: refresh a post page 10× in one session → `viewCount` increments by exactly 1.
9. **View atomicity**: simulate concurrent requests with `xargs -P` curl loop → final `viewCount` equals total requests (no lost updates).
10. Subscribe to newsletter twice with the same email → second returns conflict; one row in DB.
11. Submit contact form → row appears in `ContactMessage`.
