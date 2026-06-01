---
title: "Designing the Vithos Stack"
description: "The architectural choices behind Vithos: a type-safe React + Hono monorepo built for Cloudflare Workers."
date: 2026-05-31
draft: true
---

[Vithos](https://github.com/reybahl/vithos) is a type-safe, opinionated, full-stack template built specifically for Cloudflare Workers. I've tried out many frameworks and templates for building full-stack apps and finally landed on something that checks all my boxes: modern, type-safe, lightweight, and inexpensive at scale.

Every single architectural decision I made for Vithos was made deliberately. The purpose of this post is to walk through each of those decisions — why a specific tool or framework made more sense than the alternatives. Note that this template only contains things that most full-stack apps need, including database management and auth, but does not go as deep as state management or LLM providers.

I'll start with the application core, then the data layer, then what actually runs in production on Workers.

## The application core: Vite, Hono, TanStack Router, TanStack Query

### Vite + React + TanStack Router

For the frontend I went with a Vite + React SPA. Vite is fast, simple, and stays out of your way. HMR is instant, config is minimal, and `vite build` gives you static assets you can serve from anywhere.

I used to use Next.js for most full-stack projects. It's a nice framework with file-based routing, API routes, and, generally, a nice developer experience. But a lot of these things can easily be integrated into a Vite + React app without also needing to bring on SSR and other Next.js features that most authenticated apps don't really need. A plain SPA means no SSR runtime needed: just static files in production. I've found it to be simpler and faster to build and deploy. 

For routing, I went with TanStack Router. With React Router, search params and redirects are mostly untyped strings. TanStack Router validates and types them, which fits the rest of the stack. Auth is handled in `beforeLoad`: unauthenticated users get sent to sign-in with a redirect back, and signed-in users skip the auth pages.

### Hono as the API

For the API layer I went with Hono. It's extremely lightweight, the API surface is small and easy to learn, and the developer experience is awesome. The biggest draw is the type safety Hono RPC provides. Hono exports an `AppType` and the client uses `hc<AppType>()` to call the API. Request and response shapes are inferred from the server, so no OpenAPI spec or codegen step is needed. When you write something like `apiClient.api.counter.increment.$post()`, TypeScript already knows what comes back, so there's no typecasting you need to do at any layer. Validation on the server is just Zod via @hono/zod-validator. It supports many runtimes too, including Node, Bun, and CF workers.

Express and Fastify are proven, but Hono is the lighter-weight and more type-safe option, which was enough for me to pick it.

On the frontend, TanStack Query handles server state on top of the typed Hono client: caching, invalidation, loading and error states, mutations without wiring up `useEffect` hooks yourself. It's not client state management, which is handled by libraries like Zustand or Redux, but it's incredibly helpful for anything that hits the API.

## The data layer: Prisma + Kysely

For the database layer I split responsibilities. Prisma owns the schema and migrations (the tables needed for Better Auth also live in the Prisma schema). But for actually querying the database, I didn't want to use an ORM. Prisma Client, Drizzle, and similar tools add an extra layer of abstraction, making it more difficult to write more complex queries while maintaining type-safety.

That's where Kysely comes in. It's a typed SQL query builder, not an ORM. You write query-shaped code, TypeScript knows your table shapes (generated from the Prisma schema via `prisma-kysely`), and complex stuff like upserts and raw expressions stay readable. It's so close to SQL it feels almost like writing raw SQL with type-safety, which feels magical whenever I use it. This was an interesting (and somewhat unusual) choice, but for me, this has been the best of both worlds.

## Production deployment: Cloudflare Workers, Hyperdrive

### Why Workers

For production, everything is deployed to Cloudflare Workers. The SPA and API ship as one unit. The Worker serves static assets from the Vite build and routes `/api/*` to Hono, so there's no splitting frontend and backend across different hosts or worrying about credentialed CORS in prod. Everything runs at the edge, close to users, on a single origin.

What especially made this a no-brainer was the pricing. The Workers free tier is usable for many use cases, on the order of 100k requests per day, no egress fees, and commercial use being allowed. Compare that to Vercel's free Hobby plan, which is fine for personal sites, but it's non-commercial only, function and CPU limits are tighter, and upgrading is more expensive. For $5/month, Workers Paid includes 10M requests and far more CPU time, a big jump.

### Hyperdrive

[Hyperdrive](https://developers.cloudflare.com/hyperdrive/) (by Cloudflare) is a managed connection pool that sits between your Worker and your existing Postgres instance. You point Hyperdrive at your database’s connection string, bind it in `wrangler.toml`, and use `env.HYPERDRIVE.connectionString` in your app like a normal Postgres URL. Your drivers and query layer (Kysely + `pg`) stay the same.

Cloudflare splits the connection into two hops. Your Worker connects to a Hyperdrive endpoint at the edge, colocated with the Worker. Hyperdrive then routes the query over a warm, pooled connection to your origin database, with the pool kept in regions close to where Postgres actually runs. On a cold connection, Postgres costs you TCP, TLS, and auth round trips before you can even run a query; Hyperdrive’s pool reuses connections so you’re not paying that setup cost on every request. When the Worker finishes, your client connection is torn down, but Hyperdrive keeps the upstream connection open for the next invocation. With Hyperdrive, your app spends less time connecting, which matters on every authenticated API call. One practical note: point Hyperdrive at your database’s direct connection string, not a pooler URL. Hyperdrive already pools, and stacking poolers can cause issues.

### Database

Hyperdrive works with any Postgres: Neon, RDS, PlanetScale, self-hosted, etc. Vithos doesn't depend on a specific host: no code is specific to any one host. All you would need to configure is the DB connection string. I picked Neon as the default in the template mostly for developer experience: the free tier is generous enough to build and ship projects on, and database branching is a great feature since each PR can get its own isolated database without manual setup. The preview CI in the repo creates a branch per pull request, runs migrations against it, and tears it down when the PR closes.

I'm not particularly opinionated about where Postgres lives long term. Neon made sense as a nice default. At scale, with sustained traffic, specific compliance needs, existing infra, it could certainly make sense to evaluate something else.

### Auth

For auth I went with [Better Auth](https://www.better-auth.com). It's self-hosted, stores sessions in your Postgres database via the Kysely adapter, and supports email/password out of the box. It's open-source, extremely customizable, and avoids creating external dependencies on managed auth providers like Clerk or Auth0. On the frontend, Better Auth's React client handles sign-in and sign-up; on the server, routes under `/api/auth/*` are wired through Hono, and a session middleware loads the user on every request.

### CI/CD

The repo ships with GitHub Actions for deploy and optional PR previews. I believe in automating all deployment via GitHub Actions. This includes running database migrations, building the application, and pushing to production. I'm also a fan of feature branching: every PR gets its own preview URL and isolated database branch, so you can get as close as possible to replicating the production environment before merging, keeping the main branch stable and deployable.

## Monorepo tooling

Vithos is a pnpm workspace with Turborepo. It has several shared packages: `@acme/hono-app`, `@acme/auth`, `@acme/db`, `@acme/ui`. Let the web app, API, and Worker import the same code without publishing anything. Turborepo caches build, lint, and typecheck across packages so CI and local dev stay fast. Local Postgres runs via Docker Compose; TypeScript and ESLint configs live in a shared `tooling/` directory.

## Closing

If you're looking to start a new full-stack project on CF workers, give Vithos a try: [GitHub](https://github.com/reybahl/vithos). Always open to feedback and contributions!