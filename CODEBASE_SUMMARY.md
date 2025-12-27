# Codebase Summary

## Project Overview

**Smart Link Management System** - A full-stack application built on Cloudflare's edge platform for creating and managing intelligent short links with geo-routing capabilities and AI-powered destination monitoring.

### Core Capabilities
- Create short links with country-specific routing
- Track link clicks with geolocation analytics
- Automatically evaluate destination URLs using AI to detect product availability
- Monitor link health and identify broken/unavailable destinations

---

## Architecture

### Monorepo Structure

```
full-stack-on-cloudlare-starter-repo/
├── apps/
│   ├── data-service/          # Backend Cloudflare Worker
│   └── user-application/      # Frontend React + Worker
└── packages/
    └── data-ops/              # Shared database utilities
```

### Applications

#### 1. **data-service** (Backend Worker)
- **Purpose**: Link routing service with AI-powered destination evaluation
- **Entry Point**: `apps/data-service/src/index.ts`
- **Responsibilities**:
  - Handle link redirects based on geo-location
  - Process link click events via queues
  - Run AI workflows to evaluate destination URLs
  - Serve HTTP routes via Hono.js

#### 2. **user-application** (Frontend + API)
- **Purpose**: User-facing dashboard and tRPC API
- **Entry Points**:
  - Frontend: `apps/user-application/src/main.tsx`
  - Worker: `apps/user-application/worker/index.ts`
- **Responsibilities**:
  - Serve React SPA with TanStack Router
  - Provide tRPC API for link management
  - Handle authentication via Better Auth
  - Display analytics and link management UI

#### 3. **data-ops** (Shared Package)
- **Purpose**: Shared database queries, schemas, and utilities
- **Exports**:
  - Database connection (`database`)
  - Query functions (`queries/*`)
  - Zod schemas (`zod-schema/*`)
  - Better Auth configuration (`auth`)

---

## Tech Stack

### Frontend
- **React 19** - UI framework
- **TanStack Router** - Type-safe file-based routing
- **TanStack Query** - Data fetching and caching
- **Vite** - Build tool
- **Tailwind CSS** - Styling
- **shadcn/ui** - UI component library
- **Framer Motion** - Animations

### Backend
- **Cloudflare Workers** - Edge compute platform
- **Hono.js** - HTTP routing framework
- **tRPC** - Type-safe API layer
- **Drizzle ORM** - Database toolkit
- **Zod** - Schema validation

### Database & Storage
- **Cloudflare D1** - SQLite database
- **Cloudflare KV** - Key-value cache store

### AI & Automation
- **Cloudflare Workers AI** - LLM inference (Llama 3.3)
- **Cloudflare Browser** - Puppeteer for rendering pages
- **Cloudflare Workflows** - Orchestrated multi-step tasks
- **Cloudflare Queues** - Async message processing

### Authentication
- **Better Auth** - Authentication library with Stripe integration

---

## Key Features

### 1. Smart Links with Geo-Routing
- Create short links with multiple destination URLs
- Route users to different destinations based on their country
- Fallback to default URL for unmapped countries
- KV cache layer for fast link lookups

### 2. Link Click Tracking
- Capture click events with geolocation data (lat/long)
- Track country, timestamp, and destination
- Process clicks asynchronously via queues
- Store analytics in D1 database

### 3. AI-Powered Destination Evaluation
- Automated workflow to evaluate destination URLs
- Uses Puppeteer to render pages and extract content
- AI analysis to determine product availability status:
  - `AVAILABLE_PRODUCT` - Product is available
  - `NOT_AVAILABLE_PRODUCT` - Product out of stock/discontinued
  - `UNKNOWN_STATUS` - Status cannot be determined
- Stores evaluation results with reasoning

### 4. Analytics Dashboard
- View link performance metrics
- Track clicks by country and time
- Identify problematic destinations
- Manage link configurations

---

## Data Flow

### Link Click Flow
```
1. User visits short link (e.g., example.com/abc123)
   ↓
2. data-service receives request
   ↓
3. Checks KV cache for link info
   ↓
4. If not cached, queries D1 database
   ↓
5. Determines destination based on country (from Cloudflare headers)
   ↓
6. Sends click event to queue
   ↓
7. Redirects user to destination
   ↓
8. Queue consumer processes event and saves to database
```

### AI Evaluation Workflow
```
1. Workflow triggered with linkId, destinationUrl, accountId
   ↓
2. Step 1: Collect destination page data
   - Launch Puppeteer browser
   - Navigate to URL and wait for network idle
   - Extract body text and HTML
   ↓
3. Step 2: AI analysis
   - Send page content to Workers AI (Llama 3.3)
   - Get structured response with status and reason
   ↓
4. Step 3: Save evaluation
   - Store result in destination_evaluations table
   - Return evaluation ID
```

---

## Database Schema

### Tables

#### **links**
Stores smart link configurations
```typescript
{
  linkId: string (PK)           // Unique 10-char nanoid
  accountId: string             // Owner account ID
  name: string                  // Display name
  destinations: text (JSON)     // { "default": "url", "US": "url", ... }
  created: numeric (timestamp)
  updated: numeric (timestamp)
}
```

#### **link_clicks**
Tracks individual click events
```typescript
{
  id: string                    // Link ID
  accountId: string             // Owner account ID
  country: string               // ISO country code
  destination: string           // URL user was redirected to
  clickedTime: numeric          // Timestamp
  latitude: real                // User latitude
  longitude: real               // User longitude
}
```
**Indexes**: `id`, `clicked_time`, `account_id`

#### **destination_evaluations**
AI evaluation results
```typescript
{
  id: text (PK)                 // UUID v4
  linkId: string                // Associated link
  accountId: string             // Owner account ID
  destinationUrl: string        // URL that was evaluated
  status: string                // AVAILABLE_PRODUCT | NOT_AVAILABLE_PRODUCT | UNKNOWN_STATUS
  reason: string                // AI-generated explanation
  createdAt: numeric
}
```
**Indexes**: `(account_id, created_at)`

---

## API Structure (tRPC)

### Links Router (`apps/user-application/worker/trpc/routers/links.ts`)

#### Queries
- `linkList({ offset?: number })` - Get paginated list of links
- `getLink({ linkId: string })` - Get single link details
- `activeLinks()` - Get recently active links (last hour)
- `totalLinkClickLastHour()` - Count of clicks in last hour
- `last24HourClicks()` - Clicks comparison with previous period
- `last30DaysClicks()` - Total clicks in last 30 days
- `clicksByCountry()` - Click distribution by country

#### Mutations
- `createLink({ name, destinations })` - Create new smart link
- `updateLinkName({ linkId, name })` - Update link name
- `updateLinkDestinations({ linkId, destinations })` - Update routing config

### Evaluations Router (`apps/user-application/worker/trpc/routers/evaluations.ts`)

#### Queries
- `problematicDestinations()` - Get destinations flagged as unavailable
- `recentEvaluations({ createdBefore?: string })` - Get evaluation history

---

## Cloudflare Services Configuration

### Worker Bindings (data-service)

```jsonc
{
  "VIRTUAL_BROWSER": Browser binding,
  "AI": Workers AI binding,
  "DB": D1 Database,
  "CACHE": KV Namespace,
  "QUEUE": Queue producer (smart-links-data-queue-stage),
  "DESTINATION_EVALUATION_WORKFLOW": Workflow binding
}
```

### Queue Configuration
- **Main Queue**: `smart-links-data-queue-stage`
- **Dead Letter Queue**: `smart-links-data-dead-letter-stage`
- **Consumer**: Processes `LINK_CLICK` events

### Workflow Configuration
- **Name**: `destination-evaluation-workflow`
- **Class**: `DestinationEvaluationWorkflow`
- **Binding**: `DESTINATION_EVALUATION_WORKFLOW`

---

## Development Workflow

### Setup

```bash
# Install dependencies
pnpm install

# Generate Drizzle types (if schema changed)
cd packages/data-ops
pnpm run pull  # Pull schema from remote D1
pnpm run build # Build the package
```

### Running Development Servers

```bash
# Terminal 1: Frontend (port 3000)
pnpm run dev-frontend

# Terminal 2: Data Service
pnpm run dev-data-service
```

### Building & Deploying

```bash
# Build data-ops package
pnpm run build-package

# Deploy data-service
cd apps/data-service
pnpm run deploy

# Deploy user-application
cd apps/user-application
pnpm run build
pnpm run deploy
```

### Environment Variables

**data-ops** (for Drizzle):
- `CLOUDFLARE_ACCOUNT_ID`
- `CLOUDFLARE_DATABASE_ID`
- `CLOUDFLARE_D1_TOKEN`

**user-application** (frontend):
- `VITE_BACKEND_HOST` - Domain for short links

---

## File Structure

### apps/data-service
```
src/
├── index.ts                          # Worker entrypoint
├── hono/
│   └── app.ts                        # HTTP routes (link redirects)
├── helpers/
│   ├── ai-destination-checker.ts    # AI evaluation logic
│   ├── browser-render.ts            # Puppeteer page rendering
│   └── route-ops.ts                 # Link lookup & routing logic
├── queue-handlers/
│   └── link-clicks.ts               # Queue message processor
└── workflows/
    └── destination-evalutation-workflow.ts  # AI evaluation workflow

service-bindings.d.ts                 # Type definitions
wrangler.jsonc                        # Worker configuration
```

### apps/user-application
```
src/
├── main.tsx                          # React entry point
├── router.tsx                        # Router setup
├── components/                       # React components
│   ├── auth/                         # Authentication UI
│   ├── dashboard/                    # Analytics dashboard
│   ├── link/                         # Link management
│   └── ui/                           # shadcn components
└── routes/                           # TanStack Router routes
    ├── __root.tsx
    ├── index.tsx                     # Landing page
    └── app/_authed/                  # Protected routes
        ├── index.tsx                 # Dashboard
        ├── create.tsx                # Create link
        ├── links.tsx                 # Link list
        ├── link.$id.tsx              # Link detail
        └── evaluations.tsx           # Evaluations list

worker/
├── index.ts                          # Worker entrypoint
└── trpc/
    ├── context.ts                    # tRPC context
    ├── router.ts                     # Root router
    ├── trpc-instance.ts              # tRPC setup
    └── routers/
        ├── links.ts                  # Links endpoints
        ├── evaluations.ts            # Evaluations endpoints
        └── dummy-data.ts             # Mock data

wrangler.jsonc                        # Worker configuration
vite.config.ts                        # Vite configuration
```

### packages/data-ops
```
src/
├── db/
│   └── database.ts                   # Database initialization
├── drizzle-out/
│   ├── schema.ts                     # Drizzle schema (generated)
│   └── relations.ts                  # Table relations
├── queries/
│   ├── links.ts                      # Link CRUD operations
│   └── evaluations.ts                # Evaluation queries
├── zod/
│   ├── links.ts                      # Link validation schemas
│   └── queue.ts                      # Queue message schemas
└── better-auth/
    └── auth.ts                       # Better Auth config

drizzle.config.ts                     # Drizzle Kit config
```

---

## Key Code Paths

### 1. Creating a Link

**Frontend** → `apps/user-application/src/routes/app/_authed/create.tsx`
```typescript
createMutation.mutate({
  name: "My Link",
  destinations: { default: "https://example.com" }
})
```

**API** → `apps/user-application/worker/trpc/routers/links.ts`
```typescript
createLink: t.procedure.input(createLinkSchema).mutation(...)
```

**Database** → `packages/data-ops/src/queries/links.ts`
```typescript
export async function createLink(data) {
  const id = nanoid(10);  // Generate short ID
  await db.insert(links).values({ linkId: id, ... });
  return id;
}
```

### 2. Handling Link Clicks

**HTTP Request** → `apps/data-service/src/hono/app.ts`
```typescript
App.get('/:id', async (c) => {
  const linkInfo = await getRoutingDestinations(c.env, id);
  const destination = getDestinationForCountry(linkInfo, country);
  c.env.QUEUE.send({ type: 'LINK_CLICK', data: {...} });
  return c.redirect(destination);
})
```

**Routing Logic** → `apps/data-service/src/helpers/route-ops.ts`
```typescript
// 1. Check KV cache
const cached = await env.CACHE.get(id);
if (cached) return JSON.parse(cached);

// 2. Query database
const linkInfo = await getLink(id);

// 3. Cache result
await env.CACHE.put(id, JSON.stringify(linkInfo), { expirationTtl: 86400 });
```

**Queue Consumer** → `apps/data-service/src/index.ts`
```typescript
async queue(batch: MessageBatch) {
  for (const message of batch.messages) {
    if (event.type === 'LINK_CLICK') {
      await handleLinkClick(this.env, event);
    }
  }
}
```

**Database Insert** → `packages/data-ops/src/queries/links.ts`
```typescript
export async function addLinkClick(info) {
  await db.insert(linkClicks).values({
    id: info.id,
    country: info.country,
    destination: info.destination,
    latitude: info.latitude,
    longitude: info.longitude,
    clickedTime: info.timestamp
  });
}
```

### 3. AI Destination Evaluation

**Workflow Trigger** (via Cloudflare Dashboard or API)
```json
{
  "linkId": "abc123",
  "destinationUrl": "https://example.com/product",
  "accountId": "user123"
}
```

**Workflow** → `apps/data-service/src/workflows/destination-evalutation-workflow.ts`

**Step 1: Render Page** → `apps/data-service/src/helpers/browser-render.ts`
```typescript
const browser = await puppeteer.launch(env.VIRTUAL_BROWSER);
const page = await browser.newPage();
await page.goto(destinationUrl);
await page.waitForNetworkIdle();
const bodyText = await page.$eval('body', el => el.innerText);
```

**Step 2: AI Analysis** → `apps/data-service/src/helpers/ai-destination-checker.ts`
```typescript
const result = await generateObject({
  model: workersAi('@cf/meta/llama-3.3-70b-instruct-fp8-fast'),
  prompt: `Analyze this page and determine product availability...`,
  schema: z.object({
    pageStatus: z.object({
      status: z.enum(['AVAILABLE_PRODUCT', 'NOT_AVAILABLE_PRODUCT', 'UNKNOWN_STATUS']),
      statusReason: z.string()
    })
  })
});
```

**Step 3: Save Result** → `packages/data-ops/src/queries/evaluations.ts`
```typescript
export async function addEvaluation(data) {
  const id = uuidv4();
  await db.insert(destinationEvaluations).values({
    id, linkId, accountId, destinationUrl, status, reason
  });
  return id;
}
```

---

## Important Notes

### Parameter Naming Convention
When triggering workflows, use **camelCase** for parameters:
```json
{
  "linkId": "...",       // ✅ Correct
  "accountId": "...",    // ✅ Correct
  "destinationUrl": "..."
}
```

**NOT** lowercase:
```json
{
  "linkid": "...",       // ❌ Wrong
  "accountid": "..."     // ❌ Wrong
}
```

### Database Initialization
All Workers must call `initDatabase(env.DB)` before using database queries:
```typescript
initDatabase(this.env.DB);  // Required before any DB operations
```

### Remote Bindings
The data-service uses `--x-remote-bindings` flag in development to access remote Cloudflare resources (D1, KV, Queues, etc.) instead of local mocks.

### Link ID Format
- Generated using `nanoid(10)` - 10 character URL-safe IDs
- Example: `V1StGXR8_Z`

### Destination Format
Stored as JSON string in database:
```json
{
  "default": "https://example.com",
  "US": "https://example.com/us",
  "GB": "https://example.com/uk"
}
```

---

## Development Commands Reference

```bash
# Package builds
pnpm run build-package              # Build data-ops

# Development servers
pnpm run dev-frontend               # Start user-application:3000
pnpm run dev-data-service           # Start data-service worker

# Database operations (in packages/data-ops)
pnpm run pull                       # Pull schema from D1
pnpm run generate                   # Generate migrations
pnpm run migrate                    # Run migrations
pnpm run studio                     # Open Drizzle Studio

# Type generation
pnpm run cf-typegen                 # Generate Cloudflare types

# Deployment
pnpm run deploy                     # Deploy to Cloudflare
```

---

## Common Issues & Solutions

### Database Not Initialized Error
**Problem**: "Database not initialized" error
**Solution**: Ensure `initDatabase(env.DB)` is called in Worker constructor or before queries

### Workflow Parameter Error
**Problem**: Undefined values in workflow payload
**Solution**: Use camelCase parameter names (`linkId`, not `linkid`)

### KV Cache Invalidation
**Problem**: Stale link data after updates
**Solution**: KV cache has 24-hour TTL, or manually delete key after updates

### Queue Processing Errors
**Problem**: Messages stuck in queue
**Solution**: Check dead letter queue, validate message schema with `QueueMessageSchema`

---

## Additional Resources

- [Cloudflare Workers Docs](https://developers.cloudflare.com/workers/)
- [Cloudflare Workflows Docs](https://developers.cloudflare.com/workflows/)
- [Drizzle ORM Docs](https://orm.drizzle.team/)
- [tRPC Docs](https://trpc.io/docs)
- [TanStack Router Docs](https://tanstack.com/router/latest)
