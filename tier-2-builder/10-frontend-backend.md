# Frontend and Backend: Knowing Where Your Code Runs

## What is it?

Every web application splits its logic between two worlds: the client (browser, or frontend) and the server (backend). The client renders UI and handles user interaction. The server handles data, business logic, authentication, and anything that needs to stay secure or private.

Think of it like a restaurant. The frontend is the dining room — what the customer sees and interacts with. The backend is the kitchen — where the real work happens, where recipes are protected, where raw ingredients become meals. A good restaurant keeps these spaces separate. A confused waiter might try to cook in the dining room. A confused kitchen might hand a customer raw ingredients.

In web development terms:

- **Frontend** runs in the user's browser. It includes HTML structure, CSS styling, JavaScript behavior, and any UI framework (React, Vue, Svelte, etc.).
- **Backend** runs on a server. It processes requests, queries databases, authenticates users, and returns data to the frontend.

The boundary between them is where HTTP lives. The frontend makes HTTP requests; the backend sends HTTP responses.

## Why it matters for vibe coding

Without understanding this boundary, you will produce code that either breaks silently, leaks secrets, or confuses AI tools to the point where they generate根本无法运行的应用.

Here are the specific failure modes:

**AI puts server-only code in client components.** Next.js, React Server Components, and similar frameworks have explicit rules about what can run where. AI will often generate code that imports Node.js-only modules (`fs`, `path`, database clients) inside client components. This runs fine in development but breaks in production because client code never runs on the server.

**AI doesn't understand `useEffect` runs on the client.** AI often generates code that expects server-side data to be available synchronously during the initial render, not understanding that `useEffect` runs after hydration on the client. This causes "flicker" — the UI loads with empty data, then jumps as data arrives.

**AI blurs the line between API routes and frontend code.** In Next.js, AI will sometimes put database queries inside page components instead of API routes, or call API routes directly from server components when they should be imported directly. These are fundamentally different patterns with different performance characteristics.

**AI generates form handling that sends data to the wrong place.** Without understanding the client/server split, AI might generate code that stores sensitive config in the frontend or sends authentication tokens to the wrong endpoint.

You will not always catch these issues immediately. The code might work in development (where client and server run together locally) and only break in deployment. Understanding the boundary is what lets you verify what AI produces.

## The 20% you need to know

### The Rendering Strategies

How and where your application generates HTML determines its performance and user experience profile. Four strategies dominate:

**Server-Side Rendering (SSR)** — The server generates HTML on every request. The browser receives a complete page. Use SSR when:
- Content changes frequently (news sites, social feeds)
- Content is personalized per user (dashboards, account pages)
- SEO matters and content must be crawlable immediately
- You need access to server-side resources (databases, secrets) at render time

SSR in Next.js looks like:
```javascript
// app/page.tsx — runs on server every request
export default async function Page() {
  const data = await db.query('SELECT * FROM posts');
  return <PostList posts={data} />;
}
```

**Client-Side Rendering (CSR)** — The server sends a minimal HTML shell. The browser downloads JavaScript, executes it, and renders content. Use CSR when:
- Content is behind authentication (user-specific dashboards)
- Content updates frequently without page reloads (real-time apps)
- The page is highly interactive after load (complex forms, games)
- SEO is not critical (authenticated apps, internal tools)

CSR in React looks like:
```javascript
// Runs in browser
function PostList() {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    fetch('/api/posts').then(r => r.json()).then(setPosts);
  }, []);

  return posts.map(p => <Post key={p.id} {...p} />);
}
```

**Static Site Generation (SSG)** — HTML is generated at build time, then served as static files. Use SSG when:
- Content rarely changes (marketing pages, documentation, blogs)
- You want maximum performance (CDN caching, no server computation)
- Build time is acceptable (content updates via rebuild, not real-time)

SSG in Next.js:
```javascript
// app/page.tsx — runs at build time
export async function generateStaticParams() {
  const posts = await db.query('SELECT id FROM posts');
  return posts.map(p => ({ id: p.id.toString() }));
}

export default async function Page({ params }) {
  const post = await db.query('SELECT * FROM posts WHERE id = ?', [params.id]);
  return <PostView post={post} />;
}
```

**Incremental Static Regeneration (ISR)** — Static pages are generated at build time but can be regenerated in the background without a full rebuild. Use ISR when:
- You want static-like performance but content updates occasionally
- You have many pages and can't rebuild all of them when content changes
- Content is product/user data that changes hourly or daily, not every minute

ISR in Next.js:
```javascript
export const revalidate = 3600; // Regenerate at most once per hour

export default async function Page() {
  const data = await fetch('https://api.example.com/data', {
    next: { revalidate: 3600 }
  });
  return <Dashboard data={data} />;
}
```

Decision tree: Is the content public and rarely changes? Use SSG. Does it change frequently or is it personalized? Use SSR. Does it require real-time updates and heavy interactivity? Use CSR with an API. Do you have lots of content that changes occasionally? Use ISR.

### API Contracts

The frontend and backend communicate through APIs. An API contract defines what data the frontend can request, what format it arrives in, and what the backend will do with it.

A well-defined contract is explicit about:
- **What endpoints exist** (GET /api/posts, POST /api/posts, DELETE /api/posts/123)
- **What data each endpoint expects** (query params, body JSON, headers)
- **What data each endpoint returns** (response shape, error formats)
- **What errors mean** (404 means not found, 401 means unauthorized, 500 means server error)

Without a contract, frontend and backend drift apart. The backend changes a response shape; the frontend wasn't notified; the UI breaks. This is especially common in vibe coding where you're iterating quickly.

A simple API contract example:
```json
// GET /api/posts
// Response:
{
  "posts": [
    { "id": 1, "title": "Hello", "author": { "id": 1, "name": "Alice" } }
  ]
}

// POST /api/posts
// Request body:
{ "title": "Hello", "content": "World" }
// Response: created post object with id

// Error response shape:
{ "error": "VALIDATION_ERROR", "message": "Title is required" }
```

### The BFF Pattern

Backend for Frontend (BFF) is an architectural pattern where the frontend does not call the core API directly. Instead, it calls a layer that's specifically designed for the frontend's needs.

Without BFF: The frontend calls the core API, which returns generic data. The frontend then transforms, filters, and reshapes that data for its own use. This means the frontend contains transformation logic that duplicates backend concerns.

With BFF: The frontend calls an API that's tailored to it. The BFF aggregates data from multiple sources, formats it for the frontend's needs, and handles any frontend-specific concerns (composing a social feed from user data + post data + like data, formatted for the mobile or web client).

In Next.js, API routes can serve as a lightweight BFF:
```javascript
// app/api/feed/route.ts — BFF for the feed page
export async function GET() {
  const [user, posts, suggestions] = await Promise.all([
    getCurrentUser(),
    getFeedPosts(),
    getSuggestedUsers()
  ]);

  return Response.json({
    user: { id: user.id, name: user.name, avatar: user.avatarUrl },
    posts: posts.map(p => ({
      id: p.id,
      title: p.title,
      authorName: p.author.name,
      likeCount: p.likes.length // BFF aggregates, not raw data
    })),
    suggestions: suggestions.slice(0, 5)
  });
}
```

The frontend just renders the data. It never has to know about the underlying services.

### Hydration

Hydration is the process where the server sends pre-rendered HTML to the browser, then the browser takes over by attaching JavaScript event handlers to that static HTML. The page appears immediately (from the HTML) but becomes interactive once JavaScript finishes "hydrating" the components.

This distinction matters because:

1. During SSR, components run on the server and produce HTML. They cannot have event handlers (no clicks, no forms) during this phase.
2. When the HTML arrives at the browser, the user sees content quickly — but interactive elements don't work yet.
3. After JavaScript loads, hydration attaches event handlers. Now the page is truly interactive.

Without understanding hydration, you will be confused when:
- A "click handler" doesn't work on the first render
- An element exists in the DOM but you can't interact with it
- "Flash of unstyled content" happens during hydration
- State seems to reset when hydration completes

In React/Next.js, you see hydration issues most often with:
- Custom event handlers on server components
- Browser-only APIs (`window`, `localStorage`, `document`) used in components that render on server
- Third-party libraries that expect to run in a browser environment

## Hands-on Exercise

**Build a page that demonstrates all four rendering strategies.**

You will create a Next.js app with four pages, each demonstrating a different rendering strategy. You will verify the output by checking the HTML source and watching network requests.

**Time: 15 minutes**

1. Create a new Next.js app:
```bash
npx create-next-app@latest rendering-demo --typescript --app --no-tailwind --no-src-dir --import-alias "@/*"
cd rendering-demo
npm run dev
```

2. Create four pages. Edit `app/page.tsx` to be a navigation hub:
```javascript
// app/page.tsx
import Link from 'next/link';
export default function Home() {
  return (
    <div style={{ padding: '2rem' }}>
      <h1>Rendering Strategies Demo</h1>
      <nav>
        <ul>
          <li><Link href="/ssr">SSR Page</Link></li>
          <li><Link href="/csr">CSR Page</Link></li>
          <li><Link href="/ssg">SSG Page</Link></li>
          <li><Link href="/isr">ISR Page</Link></li>
        </ul>
      </nav>
    </div>
  );
}
```

3. Create the SSR page:
```javascript
// app/ssr/page.tsx
export const dynamic = 'force-dynamic'; // always SSR

export default async function SSRPage() {
  const time = new Date().toISOString();
  return (
    <div style={{ padding: '2rem' }}>
      <h1>Server-Side Rendering</h1>
      <p>This page renders on every request.</p>
      <p>Server time: {time}</p>
    </div>
  );
}
```

4. Create the CSR page:
```javascript
// app/csr/page.tsx
'use client';
import { useState, useEffect } from 'react';

export default function CSRPage() {
  const [time, setTime] = useState<string>('loading...');

  useEffect(() => {
    setTime(new Date().toISOString());
  }, []);

  return (
    <div style={{ padding: '2rem' }}>
      <h1>Client-Side Rendering</h1>
      <p>This page renders on the client after JavaScript loads.</p>
      <p>Client time: {time}</p>
    </div>
  );
}
```

5. Create the SSG page:
```javascript
// app/ssg/page.tsx
export const dynamic = 'force-static'; // always SSG

export default function SSGPage() {
  const buildTime = new Date().toISOString();
  return (
    <div style={{ padding: '2rem' }}>
      <h1>Static Site Generation</h1>
      <p>This page was built at:</p>
      <p>{buildTime}</p>
    </div>
  );
}
```

6. Create the ISR page:
```javascript
// app/isr/page.tsx
export const revalidate = 10; // Revalidate every 10 seconds

export default async function ISRPage() {
  const time = new Date().toISOString();
  return (
    <div style={{ padding: '2rem' }}>
      <h1>Incremental Static Regeneration</h1>
      <p>This page is statically generated but revalidates.</p>
      <p>Current time: {time}</p>
    </div>
  );
}
```

7. Test the difference between SSR and SSG:
   - For SSG page, refresh — the timestamp does not change (it's baked in at build)
   - For SSR page, refresh — the timestamp updates (it's generated on every request)
   - Check "View Source" on the SSG page — you see the full HTML
   - Check "View Source" on the CSR page — you see a loading state or empty shell (the real content isn't in the HTML)

## Common mistakes

**Mistake 1: Importing server-only modules in client components.**

What happens: You import `fs` or a database client in a component marked with `'use client'`. In development, this works. In production, the build fails or the module is undefined at runtime.

Why it happens: Next.js and other frameworks isolate client and server code, but only during builds or at runtime — not always in development mode.

How to fix: Keep server-only code in server components or API routes. If you need to share code between server and client, extract the shared logic into a separate module that has no server dependencies. Use the `server-only` npm package to enforce this at build time:
```javascript
// Add to any module that should never run on the client
import 'server-only';
```

**Mistake 2: Expecting useEffect to run before first render.**

What happens: You put data fetching in `useEffect` and the UI shows nothing on first load, then "jumps" when data arrives.

Why it happens: `useEffect` runs after the component renders and after the browser paints. It cannot prevent the initial render.

How to fix: If you need data before render, either use SSR (fetch data on the server in a server component) or use a loading state that you design into the UI intentionally. Never assume data is available synchronously in a client component.

**Mistake 3: Sending sensitive data to the client because "it just needs to be in the page."**

What happens: You put API keys, database URLs, or user tokens in a component and ship it to the browser. Attackers find them in the HTML source.

Why it happens: The developer thought of the page as one unit, not two separate environments.

How to fix: Keep sensitive operations on the server. If the page needs data from an API, call the API from a server component — the secrets never leave the server. If you need to call an external API from the client, use a server-side proxy (an API route) so the secret stays hidden.

**Mistake 4: Calling API routes from server components instead of calling the underlying logic directly.**

What happens: A server component fetches from `/api/posts` instead of calling the database directly. This adds unnecessary HTTP overhead and confuses the architecture.

Why it happens: The developer thinks "all data fetching goes through API routes."

How to fix: Server components can query databases directly. API routes are for external clients (mobile apps, third parties, or the SPA fallback). If you're writing a Next.js server component and you have direct access to the database, use it.

**Mistake 5: Not handling hydration mismatches.**

What happens: The server renders one thing (e.g., today's date), the client renders something different (a second later). React throws a hydration error or the UI flickers.

Why it happens: Code generates different output on server vs. client — random values, `Date.now()`, or non-deterministic data.

How to fix: Ensure components produce identical output on server and client. Use `suppressHydrationWarning` sparingly. Move time-sensitive or random data to `useEffect` so the server and client both render empty initially.

## AI-specific pitfalls

**AI puts server-only code in client components.** This is the most common AI failure in this space. AI will generate imports like `import { db } from '@/lib/db'` inside a `'use client'` component because it doesn't track the server/client boundary. When reviewing AI output, look for any import of database clients, file system modules, or environment variables that should stay server-side.

**AI generates API routes that are insecure by default.** AI often creates API routes that trust client input without validation, expose database error messages directly, or don't check authentication. Always review API route code for input validation and authorization checks.

**AI doesn't specify rendering strategies in Next.js App Router.** AI will generate Next.js pages without specifying `dynamic` or `revalidate`, leading to confusing behavior where you can't tell if a page is SSR, SSG, or ISR just by looking at it. Enforce explicit declaration of rendering strategy.

**AI assumes `useEffect` runs synchronously.** AI will generate patterns where `useEffect` sets state that is immediately read in the same render cycle, or it will omit `useEffect` and call async code directly in the component body. Both cause bugs. Make sure AI's async data fetching uses `useEffect` with proper dependency arrays and loading states.

**AI blurs the BFF concept.** When you ask AI to "fetch some data for the frontend," it will often call external APIs directly from client components instead of creating a server-side aggregation layer. For anything beyond simple fetching, ask AI to create an API route that handles the aggregation.

## Quick Reference

| Strategy | Renders | When to use | Performance |
|---|---|---|---|
| **SSR** | On every request | Dynamic, personalized, SEO-critical | Medium — server compute on each request |
| **CSR** | In browser | Behind auth, real-time, highly interactive | Slow initial load, fast after |
| **SSG** | At build time | Static, rarely changes, high traffic | Fastest — pure CDN |
| **ISR** | At build + periodic | Content that updates occasionally | Fast + fresh |

**Hydration checklist:**
- Did this component render on server? It cannot have event handlers during SSR.
- Am I using `useEffect` for data? It won't run until after first render.
- Am I using browser APIs (`window`, `document`)? Wrap in `typeof window !== 'undefined'` or move to `useEffect`.
- Is the output deterministic? Random values and `Date.now()` cause hydration mismatches.

**Server vs. Client decision tree:**
1. Does this need secrets, database, or server-only resources? → Server component or API route
2. Does this need user interaction (clicks, forms, hover)? → Client component
3. Does this need to be crawlable by search engines? → Server component
4. Is this behind authentication? → Server component (fetch data), client handles display
5. Does this update without page reload? → Client component with its own data fetching

## Go deeper

- [Next.js Rendering Modes Documentation](https://nextjs.org/docs/app/building-your-application/data-fetching) (verified 2026-04) — Official docs covering SSR, SSG, ISR, and how to choose.
- [React Server Components](https://react.dev/reference/rsc) (verified 2026-04) — Official React docs on how server and client components work together.
- [MDN: How the Web Works](https://developer.mozilla.org/en-US/docs/Learn/Getting_started_with_the_web/How_the_Web_works) (verified 2026-04) — The foundational mental model for client/server communication.
- [Backend-for-Frontend Pattern](https://shopify.engineering/bff-architecture) (verified 2026-04) — Shopify's engineering blog on why they use BFF and how it works at scale.
- [Web.dev: Rendering on the Web](https://web.dev/articles/rendering-on-the-web) (verified 2026-04) — Google's guide comparing rendering strategies with performance implications.