
Nice — I’ll walk you through a complete, practical step-by-step integration of **Sanity visual editing + live preview** into a Next.js **App Router** project (draft mode, `next-sanity` live helpers, stega, and the Presentation tool). I’ll give runnable code snippets you can drop into your project and explain why each piece is needed. I’ll assume the project root is `src/` like your structure.

I looked up the latest `next-sanity` / Sanity visual-editing recommendations and Next.js Draft Mode docs while preparing this to make sure the steps match current best practice. ([Sanity.io][1])

---

# Quick overview (what we’ll create)

1. `.env` with required vars.
2. `src/sanity/client.ts` — Sanity client with `stega` & tokens.
3. `src/sanity/live.ts` — `defineLive` wrapper + `sanityFetch` helper.
4. `src/app/api/draft-mode/enable/route.ts` — API route to enable Draft Mode (Presentation tool uses this).
5. `src/app/layout.tsx` (root layout) — Render `<SanityLive />` and `<VisualEditing />` when Draft Mode is enabled.
6. `src/app/[slug]/page.tsx` — Example dynamic page using `sanityFetch` + `defineQuery`.
7. `src/components/DisableDraftMode.tsx` — small helper to turn Draft Mode off.
8. Sanity Studio: configure Presentation tool and `studioUrl` so Studio ↔ Frontend know about each other.

Each step includes code + explanation.

---

## 0) Prereqs & packages

Run in your project:

```bash
npm install next-sanity @sanity/client @portabletext/react @sanity/image-url
# or
yarn add next-sanity @sanity/client @portabletext/react @sanity/image-url
```

`next-sanity` bundles helpers for visual editing + draft-mode integration. ([Sanity.io][2])

---

## 1) Environment variables (`.env`)

Create `.env` (or `.env.local`) at project root:

```
NEXT_PUBLIC_SANITY_PROJECT_ID=yourProjectID
NEXT_PUBLIC_SANITY_DATASET=production
NEXT_PUBLIC_SANITY_STUDIO_URL=https://your-studio.sanity.studio
SANITY_VIEWER_TOKEN=yourViewerToken
SANITY_STUDIO_PREVIEW_ORIGIN=https://your-frontend.com
```

Notes:

* `SANITY_VIEWER_TOKEN` should be a token with read access to drafts (viewer token). Keep it server-side (do not expose in client bundle).
* `NEXT_PUBLIC_SANITY_STUDIO_URL` is used for the stega links that let VisualEditing open the Studio at the right document. ([Sanity.io][1])

---

## 2) `src/sanity/client.ts` — create Sanity client with stega

Create `src/sanity/client.ts`:

```ts
// src/sanity/client.ts
import { createClient } from "next-sanity";

export const client = createClient({
  projectId: process.env.NEXT_PUBLIC_SANITY_PROJECT_ID!,
  dataset: process.env.NEXT_PUBLIC_SANITY_DATASET || "production",
  apiVersion: "2024-12-01", // pin to a stable date
  useCdn: true,
  token: process.env.SANITY_VIEWER_TOKEN, // server-side token if used
  stega: {
    // This is what embeds editing metadata into HTML for VisualEditing
    studioUrl: process.env.NEXT_PUBLIC_SANITY_STUDIO_URL,
  },
});
```

Why: `stega` adds hidden metadata into HTML so Studio can find the matching doc for an overlay. `next-sanity` client supports this option and it’s required for Visual Editing overlays. ([npm][3])

---

## 3) `src/sanity/live.ts` — defineLive and sanityFetch helper

Create `src/sanity/live.ts`:

```ts
// src/sanity/live.ts
import { defineLive, defineQuery } from "next-sanity";
import { client } from "./client";

export const live = defineLive({
  // serverToken must be able to read drafts (used server-side)
  serverToken: process.env.SANITY_VIEWER_TOKEN,
  // Optional: browserToken improves client-side live experience
  // browserToken: process.env.NEXT_PUBLIC_SANITY_BROWSER_TOKEN,
  client,
});

export const sanityFetch = live.sanityFetch;
export const defineSanityQuery = defineQuery;
```

Why & notes:

* `defineLive` wires up Sanity's live preview features and exposes a `SanityLive` component you can render from your layout.
* `sanityFetch` wraps queries in a way that automatically returns draft or published content depending on Draft Mode / live updates. This replaces manual `client.fetch` in pages. ([GitHub][4])

---

## 4) Draft Mode API: `src/app/api/draft-mode/enable/route.ts`

Create the API route that Presentation tool (or Studio) will hit to enable Draft Mode:

```ts
// src/app/api/draft-mode/enable/route.ts
import { defineEnableDraftMode } from "next-sanity/draft-mode";
import { client } from "@/sanity/client";

export const { GET } = defineEnableDraftMode({
  client: client.withConfig({
    token: process.env.SANITY_VIEWER_TOKEN,
  }),
});
```

Explanation:

* `defineEnableDraftMode` returns a `GET` handler that will enable Next.js Draft Mode and set cookies that cause server rendering to fetch drafts. Presentation tool in Studio calls this route when you click "Open in frontend" or "presentation". Official docs show this pattern. ([Sanity.io][1])

(Optional) create a route to **disable**:

```ts
// src/app/api/draft-mode/disable/route.ts
import { draftMode } from "next/headers";
export async function GET() {
  draftMode().disable();
  return new Response("draft mode disabled");
}
```

---

## 5) Root layout: render `SanityLive` and `VisualEditing`

Update `src/app/layout.tsx` (or `/app/layout.tsx`) to include `SanityLive` and `VisualEditing`. Example:

```tsx
// src/app/layout.tsx
import React from "react";
import { SanityLive } from "@/sanity/live";
import { VisualEditing } from "next-sanity/visual-editing";
import { draftMode } from "next/headers";

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const isDraft = (await draftMode()).isEnabled;
  return (
    <html lang="en">
      <body>
        {children}
        <SanityLive />
        {isDraft && <VisualEditing />}
      </body>
    </html>
  );
}
```

Why:

* `SanityLive` handles the client-side wiring for live updates and listens to changes.
* `VisualEditing` adds overlays + clickable areas to open the Studio for inline editing when Draft Mode is enabled. The `draftMode()` check prevents VisualEditing in production public view. ([Sanity.io][1])

---

## 6) Example page: `src/app/[slug]/page.tsx`

This shows server-rendered page using `sanityFetch` and `defineQuery`.

```tsx
// src/app/[slug]/page.tsx
import React from "react";
import { sanityFetch, defineSanityQuery } from "@/sanity/live";

const query = defineSanityQuery(`
  *[_type == "page" && slug.current == $slug][0]{
    _id,
    title,
    content[] // whatever fields you have
  }
`);

export default async function Page({ params }: { params: { slug: string } }) {
  const { data } = await sanityFetch({ query, params });
  if (!data) return <div>Not found</div>;

  // Render content — use PortableText for blocks if needed
  return (
    <main>
      <h1 data-sanity-field={data._id}>{data.title}</h1>
      <div>{/* render data.content here with PortableText */}</div>
    </main>
  );
}
```

Notes:

* `sanityFetch` will pull draft content if Draft Mode is enabled (server cookie) and it will manage cache/revalidation behavior for you. You don't need to call `client.fetch` yourself. ([GitHub][5])
* Add `data-sanity-field` attributes where helpful; `next-sanity` stega will instrument the page so overlays know which doc/field is which.

---

## 7) DisableDraftMode component (optional)

Small client component to turn Draft Mode off:

```tsx
// src/components/DisableDraftMode.tsx
"use client";
import React from "react";

export default function DisableDraftMode() {
  async function handleClick() {
    await fetch("/api/draft-mode/disable");
    // reload to reflect public content
    location.reload();
  }
  return <button onClick={handleClick}>Exit Draft Mode</button>;
}
```

---

## 8) Sanity Studio: Presentation tool + preview config

In your Studio (e.g., `studio/`), ensure you add the Presentation tool and configure the preview origin:

```ts
// studio/sanity.config.ts (excerpt)
import { presentationTool } from "sanity/presentation";

export default defineConfig({
  // ... existing config
  plugins: [
    presentationTool({
      // Frontend URL that Presentation tool will call to enable Draft Mode
      previewUrls: [
        {
          name: "Next App",
          // ensure this matches SANITY_STUDIO_PREVIEW_ORIGIN or NEXT_PUBLIC_STUDIO_URL settings
          url: process.env.SANITY_STUDIO_PREVIEW_ORIGIN || "https://your-frontend.com",
          // path to the route that Presentation tool will call to enable draft mode
          // Presentation will call: `${url}/api/draft-mode/enable?slug=...`
        },
      ],
    }),
    // ...other plugins
  ],
});
```

Why:

* The Presentation tool uses your preview URLs to call the `/api/draft-mode/enable` route and then open the frontend URL with Draft Mode enabled. That’s how editors click “Open preview” from Studio and see drafts in the real site. ([focusreactive.com][6])

---

## 9) Stega (HTML embedding) — what you need to know

* The `client` with `stega` (we added `studioUrl`) will cause `next-sanity` to embed metadata in the page HTML so Visual Editing overlays can open Studio at the exact document/field. You don’t have to implement stega wiring manually—the `next-sanity` client + `VisualEditing` do the heavy lifting. ([npm][3])

---

## 10) Security & deployment notes

* **Server tokens** (viewer token) must be server-only. Use `process.env` and `client.withConfig({ token })` in server code. Do **not** expose `SANITY_VIEWER_TOKEN` to client bundles. ([GitHub][4])
* Confirm your hosting config accepts the Draft Mode cookies. Vercel/most hosts support this pattern out of the box. Next.js docs explain Draft Mode behavior. ([Next.js][7])

---

## 11) Testing flow (how editors will use it)

1. Log into Sanity Studio.
2. Open Presentation tool → choose your frontend preview URL. The Presentation tool will call `/api/draft-mode/enable` on your frontend. ([focusreactive.com][6])
3. Frontend loads in Draft Mode and `VisualEditing` overlays appear. Click content on the page → it opens the Studio document for inline editing.
4. Edits save in Studio; `SanityLive` picks up changes and updates the page in real time.

---

## 12) Troubleshooting tips

* **No overlays visible**: ensure Draft Mode is enabled and `VisualEditing` is rendered (check `draftMode().isEnabled`). Confirm `NEXT_PUBLIC_SANITY_STUDIO_URL` is correct and matches Studio origin. ([Sanity.io][1])
* **Draft content not showing**: verify your `defineEnableDraftMode` route uses a server token that can read drafts. Verify cookies are set (inspect response `Set-Cookie` for `__prerender_bypass`). See Next.js Draft Mode docs. ([Next.js][7])
* **Live updates not appearing**: ensure `SanityLive` is present and `defineLive` was configured with `serverToken`/`browserToken` as needed. ([GitHub][4])

---

## 13) Extra: Block types, PortableText & GSAP animations

* For block content (Portable Text), use `@portabletext/react` in your page to render portable text and pass the same fields you fetched with `sanityFetch`.
* If you want GSAP animations tied to editable elements, add consistent DOM attributes (e.g., `data-sanity-id`) so overlays don’t interfere — place your animation initialization in a client component and avoid mutating the DOM that Visual Editing expects.

---

## TL;DR — Minimal set of files to drop in

* `.env.local` (vars above)
* `src/sanity/client.ts` (stega)
* `src/sanity/live.ts` (`defineLive`, `sanityFetch`)
* `src/app/api/draft-mode/enable/route.ts` (enable)
* `src/app/layout.tsx` (SanityLive + VisualEditing)
* `src/app/[slug]/page.tsx` (example page using `sanityFetch`)
* Studio: add `presentationTool` and set preview URL.

Official docs & references I used while preparing this so you can dive deeper: Sanity Visual Editing guide, `next-sanity` README, and Next.js Draft Mode docs. ([Sanity.io][1])

---

If you want I can:

* Create a **full ready-to-run repo** structure (all files filled in) you can `git clone` (I’ll paste everything in the chat).
* Or generate the exact `page.tsx` that supports **BlockQuote**, **TextBlock** and includes a small GSAP scroll animation in a client component, wired to `sanityFetch` — ready for you to paste.

Which of those do you want next? (I’ll generate the files right here in chat.)

[1]: https://www.sanity.io/docs/visual-editing/visual-editing-with-next-js-app-router?utm_source=chatgpt.com "Visual Editing with Next.js App Router"
[2]: https://www.sanity.io/plugins/next-sanity?utm_source=chatgpt.com "The official Sanity toolkit for Next.js | Sanity.io plugin"
[3]: https://www.npmjs.com/package/next-sanity/v/8.5.5?utm_source=chatgpt.com "next-sanity"
[4]: https://github.com/sanity-io/next-sanity?utm_source=chatgpt.com "Sanity toolkit for Next.js"
[5]: https://raw.githubusercontent.com/sanity-io/next-sanity/main/packages/next-sanity/README.md?utm_source=chatgpt.com "https://raw.githubusercontent.com/sanity-io/next-s..."
[6]: https://focusreactive.com/sanity-visual-editing-review/?utm_source=chatgpt.com "A Deep Dive into Sanity's Visual Editing and Presentation ..."
[7]: https://nextjs.org/docs/app/guides/draft-mode?utm_source=chatgpt.com "Guides: Draft Mode"
