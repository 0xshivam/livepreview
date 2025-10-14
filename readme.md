Perfect! What you’ve pasted is essentially a **complete guide to setting up Sanity Visual Editing (inline/live preview) with Next.js App Router** using the new `next-sanity` v11 tools. Here’s a **summary of how it works** and how to integrate it into your project:

---

## 1️⃣ Core Concepts

1. **Draft Mode**

   * Next.js draft mode lets authorized users view **draft content**.
   * API routes enable turning draft mode on/off (`/api/draft-mode/enable`).

2. **VisualEditing Component**

   * Provided by `next-sanity/visual-editing`.
   * Adds overlays to your app so editors can click content and jump to Studio.
   * Should be rendered in your root layout when draft mode is active.

3. **Live Content (`SanityLive`)**

   * `defineLive` + `SanityLive` handles **real-time updates** in your frontend.
   * Automatically switches between draft and published content.
   * Replaces manual `client.fetch` calls with `sanityFetch`.

4. **Stega Encoding**

   * Encodes metadata in your page HTML so the Studio knows which content maps to which document.
   * Used for overlays and editing links.

---

## 2️⃣ Project Structure (Next.js App Router)

```
src/
├─ app/
│  ├─ [slug]/page.tsx         # Your dynamic pages
│  ├─ layout.tsx              # Root layout with <SanityLive> and <VisualEditing>
│  └─ api/
│     └─ draft-mode/enable/route.ts
├─ components/
│  └─ DisableDraftMode.tsx    # Optional UI to disable draft mode
├─ sanity/
│  ├─ client.ts               # Sanity client with stega
│  └─ live.ts                 # defineLive wrapper
└─ .env                       # SANITY & NEXT_PUBLIC variables
```

---

## 3️⃣ Key Implementation Steps

1. **Install dependencies**

```bash
npm install @sanity/client next-sanity
```

2. **Configure `.env`**

```env
NEXT_PUBLIC_SANITY_PROJECT_ID=yourProjectID
NEXT_PUBLIC_SANITY_DATASET=production
NEXT_PUBLIC_SANITY_STUDIO_URL=https://your-studio.sanity.studio
SANITY_VIEWER_TOKEN=yourViewerToken
SANITY_STUDIO_PREVIEW_ORIGIN=https://your-frontend.com
```

3. **Sanity client with Stega**

```ts
import { createClient } from "next-sanity";

export const client = createClient({
  projectId: process.env.NEXT_PUBLIC_SANITY_PROJECT_ID,
  dataset: process.env.NEXT_PUBLIC_SANITY_DATASET,
  apiVersion: "2024-12-01",
  useCdn: true,
  token: process.env.SANITY_VIEWER_TOKEN,
  stega: { studioUrl: process.env.NEXT_PUBLIC_SANITY_STUDIO_URL },
});
```

4. **Enable Draft Mode API**

```ts
import { defineEnableDraftMode } from "next-sanity/draft-mode";
import { client } from "@/sanity/client";

export const { GET } = defineEnableDraftMode({
  client: client.withConfig({ token: process.env.SANITY_VIEWER_TOKEN }),
});
```

5. **Visual Editing in Layout**

```tsx
import { VisualEditing } from "next-sanity/visual-editing";
import { draftMode } from "next/headers";
import { SanityLive } from "@/sanity/live";

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <SanityLive />
        {(await draftMode()).isEnabled && <VisualEditing />}
      </body>
    </html>
  );
}
```

6. **Live Fetch Pages**

```ts
import { defineQuery } from "next-sanity";
import { sanityFetch } from "@/sanity/live";

const query = defineQuery(`*[_type == "page" && slug.current == $slug][0]{title}`);

export default async function Page({ params }: { params: { slug: string } }) {
  const { data } = await sanityFetch({ query, params });
  return <h1>{data.title}</h1>;
}
```

---

## 4️⃣ Optional Extras

* **DisableDraftMode** component for content editors outside of Presentation tool.
* **Presentation Tool in Studio**: Embed your Next.js pages in Studio for editors to see **live inline editing**.
* **Live Content API**: Use for true real-time updates when editing multiple sections or blocks.

---

✅ **Result**

* Editors can **click content in your frontend**, jump to Studio, and edit it.
* Changes are reflected **immediately** in the page using `SanityLive`.
* Draft/published switching is automatic.
* Overlays make editing intuitive without leaving the page.

---

If you want, I can **write a full working example for your `[slug]/page.tsx` that supports BlockQuote, TextBlock, and GSAP animations with live visual editing enabled**.

Do you want me to do that?
