# Norsk Gjenbruksmarked – Store rebuild plan

Status: **plan ready, build blocked on 3 setup steps** (see "What I need from you").
Audit date: 2026-10-12. Source audited: Lovable project
"Norsk Gjenbruksmarked: Quality Renewed" (`d5d7713f-e169-4384-b099-7c26d48158b6`),
latest commit `b5cae0aaec832c9c9f26a8c4b190845ed94cf2af` ("Added 3D hero fade in", 2026-09-15).
That commit is the **rollback point**: nothing below goes live until it has been
built, checked, and published on purpose.

---

## 1. How the store works today

| Layer | What it is |
|---|---|
| Frontend | TanStack Start (React 19, SSR) + Tailwind 4 + shadcn/ui, built and hosted by Lovable |
| Products / stock / checkout | Shopify `802df2-2.myshopify.com`, read through the Storefront API (`src/lib/shopify.ts`) |
| Cart | Local cart in the browser (`src/lib/cart.tsx`), handed to Shopify with `cartCreate` on `/checkout` |
| Checkout | Own `/checkout` page (delivery/pickup, date, time window) → Shopify hosted checkout on the myshopify domain |
| Forms | Lovable Cloud (Supabase) tables + Gmail notification to the owner (Reply-To = customer) + Resend auto-reply from `team@norskgjenbruksmarked.no` |
| SEO | Per-route `head()`, JSON-LD on home/collections/products, dynamic `/sitemap.xml`, `robots.txt` |

### Forms that must keep working exactly as now

All live in `src/lib/form-submissions.functions.ts`. The rebuild will **not change
these server functions, their tables, or their email routing** – only the UI around them.

| Form | Page(s) | Saved to | Email |
|---|---|---|---|
| Kontakt | `/kontakt` | `contact_submissions` | Owner Gmail + customer auto-reply |
| Book visning | `/book-visning` | `viewing_bookings` | Owner Gmail + customer auto-reply |
| Selg til oss | `/selg-til-oss` | `sell_submissions` | Owner Gmail + customer auto-reply |
| Møbelrens | `/mobelrens` | `contact_submissions` (prefixed `[MØBELRENS]`) | Owner Gmail + customer auto-reply |
| Nyhetsbrev | Footer | email only | Owner Gmail |
| Venteliste | Sold-out product page | email only | Owner Gmail |

Image uploads go to the Supabase storage bucket `form-uploads` (signed links in the email).

---

## 2. Audit – what is holding the store back

Ordered by impact on sales and Google ranking.

1. **Search does not search the shop.** `/sok` queries a Supabase `products` table
   (`src/lib/products.ts` → `searchProducts`), not Shopify. Customers searching for
   a real sofa get stale or empty results.
2. **Collection pages are invisible to Google.** `CollectionPage.tsx` and `/sofaer`
   load products only in the browser (`useQuery`), so the server-rendered HTML that
   Google indexes has no products in it.
3. **Sold items are mixed in with available ones** and cannot be clicked
   (`pointer-events-none` on sold cards). The requirement is: available first,
   everything sold collected at the bottom.
4. **Old Shopify URLs have no redirects.** The domain used to serve the Shopify theme
   (`/products/...`, `/collections/...`, `/pages/...`). Those URLs are still in
   Google and on social media and now 404.
5. **Checkout leaks buyers.**
   - Payment opens in a **new tab** (`window.open(..., "_blank")`); pop-up blockers
     and confused mobile users drop off here.
   - The cart is **emptied before payment**, so anyone who goes back loses everything.
   - Three separate accordion steps for method → date → time before the customer even
     sees the payment page.
   - Unique items can be added with quantity > 1.
6. **Delivery messaging contradicts itself.** Announcement bar / hero / cart drawer say
   "Gratis levering i Bergen"; `src/lib/checkout.ts` charges 999 kr under 6 000 kr;
   the product page says "Rimelig levering". **You need to tell me which rule is true**
   (see section 6). The rebuild moves every fee and promise into one config file so
   it can never drift again.
7. **Two different collection implementations** (`/sofaer` has rich filters,
   every other category has a weaker copy in `CollectionPage.tsx`).
8. **Product page breadcrumb is wrong for anything that isn't a sofa** – it always
   links to `/sofa-og-lenestol`, even for office chairs (also in the JSON-LD).
9. Collections fetch max 250 products with no pagination (brand pages filter the
   whole catalog client-side).
10. Leftovers: a metafield debug table and an unused `ProductSpecs` component
    are still in `product.$slug.tsx` as dead code; "Møbelrens" in the menu points to
    `/kontakt` although `/mobelrens` exists; no search on mobile outside the menu.

---

## 3. The new store – what customers get

I could not open secundo.no from this environment (network policy), so the
comparison is against what Secundo is known for: refurbished, tested, deep-cleaned
furniture, delivery in their region, pickup point, and a "sell to us" flow with a
price offer within 24 h. The store already matches those basics. The plan below
is built to beat them on **finding, trusting and buying**.

### Finding (collections, search, navigation)
- **Available first, sold at the bottom** on every collection, brand and search page:
  a clear "Nylig solgt" divider, sold items greyed but clickable (social proof +
  "varsle meg om lignende").
- One shared, fast collection engine for all 20+ category, brand and campaign pages.
- Filters built from real data, with live counts: type, brand, price (two-handle
  slider + presets), condition, colour/material (from metafields/tags when present),
  width ranges, "På tilbud", "Nytt siste 14 dager", "Gratis levering-kvalifisert".
- Filters and sort live in the URL (shareable, back button works) – canonical
  stays clean so Google is not flooded with filter URLs.
- Sort: Anbefalt · Nyeste · Pris lav–høy · Pris høy–lav · Størst rabatt.
- Quick category chips with counts, grid density toggle, infinite scroll with a
  real "Viser 24 av 87".
- **Instant search** in the header (desktop) and a bottom nav search (mobile):
  results with photo and price while typing, plus categories and popular searches –
  powered by Shopify's own search, sold items ranked last.
- Mega menu with live product counts; app-like **bottom navigation on mobile**
  (Hjem · Kategorier · Søk · Favoritter · Kurv).

### Trusting (product page)
- Bigger gallery: thumbnails, hover zoom, full-screen swipe + pinch, image counter.
- Above the fold: condition, brand, key measurements (B × D × H), material, colour.
- Honest savings: "Nypris 24 990 kr – du sparer 16 000 kr (64 %)" from the
  `custom.new_price` metafield already in Shopify.
- "Unikt eksemplar – kun 1 stk" (real stock only – **no fake "12 people are viewing"
  counters**; those are misleading and hurt trust).
- Delivery estimate by postcode (uses the zone list already in `checkout.ts`).
- One-tap **"Book visning av denne"** and **"Still et spørsmål"** with the product
  pre-filled in the existing forms.
- Favourites (heart) on every card and product, `/favoritter` page.
- "Lignende" and "Passer sammen med" from Shopify recommendations, plus "Sist sett".
- Sold product page: clear "Solgt" state, similar available items first, waitlist.

### Buying (cart and checkout)
- "Kjøp nå" next to "Legg i handlekurv" – goes straight to the delivery step with
  only that item.
- Cart drawer: re-checks stock every time it opens (sold items flagged and removed
  with a friendly message), progress bar towards free delivery, quantity locked to
  what's actually in stock.
- **Checkout on one screen**: method tiles, the next 14 delivery dates as tappable
  chips, time windows, extras – all visible at once, sticky summary, one button.
- Payment in the **same tab**, cart kept until the items are confirmed sold
  (so going back never loses anything).
- All delivery/booking info still lands on the Shopify order as note + attributes,
  exactly like today.

### Feel
- Same brand (deep green `#16332b`, warm cream, gold accent, Poppins) but a tighter
  design system: one button style, one badge style, consistent spacing.
- Faster: Shopify CDN image sizes (`srcset`), lazy loading, preloaded hero, no
  page-transition fade that delays every navigation.

---

## 4. SEO plan

- Products rendered on the server for every collection page (fixes #2 above).
- Per page: unique title (≤ 60), description (≤ 155), canonical, Open Graph + Twitter
  image, `nb_NO` locale.
- JSON-LD: `Organization` + `WebSite` with search box on home; `BreadcrumbList`
  everywhere (correct category for each product); `ItemList` on collections;
  `Product` with `UsedCondition`, real availability (`SoldOut` when sold), brand,
  images, shipping details and 14-day return policy; `FAQPage` where FAQs exist.
- Sitemap: home, all info pages, all collections, all available products, with
  `lastmod`.
- **301 redirects** for legacy Shopify URLs (server middleware):

| Old URL | New URL |
|---|---|
| `/products/:handle` | `/product/:handle` |
| `/collections/all` | `/moblene` |
| `/collections/:handle` | matching category page from `collections-config.ts`, else `/moblene` |
| `/collections/:c/products/:handle` | `/product/:handle` |
| `/pages/contact`, `/pages/kontakt` | `/kontakt` |
| `/pages/om-oss`, `/pages/about-us` | `/om-oss` |
| `/search?q=` | `/sok?q=` |
| `/cart` | stays (existing route) |

---

## 5. Build phases

Each phase ends with: type-check + production build passing, all forms re-tested,
and a preview link for you before anything is published.

| # | Phase | Main outcome |
|---|---|---|
| 0 | Foundations | One Shopify data layer (paginated, typed), one store-config file (fees, USPs, contact), server-side loading, legacy redirects |
| 1 | Shell | New header + instant search + mega menu, mobile bottom nav, footer (same links), favourites |
| 2 | Collections | Shared collection engine, available-first / sold-at-bottom, filters in URL, quick view, SSR + ItemList |
| 3 | Product page | Gallery, buy box, savings, delivery estimate, book/ask shortcuts, recommendations, sold state, Product JSON-LD |
| 4 | Cart + checkout | Stock-aware cart, Kjøp nå, one-screen checkout, same-tab payment, keep cart until sold |
| 5 | Home | Search-first hero, "Nytt denne uken", "Nettopp solgt", categories with counts, brands, B2B, reviews, sell-to-us |
| 6 | SEO + speed | Sitemap, JSON-LD audit, image sizing, Lighthouse pass, 404/redirect check |
| 7 | Go live | Publish in Lovable to the custom domain, smoke-test every form and checkout on the live domain |

---

## 6. What I need from you

1. **Lovable plan.** The workspace is on the **Free** plan. Custom domains need a paid
   plan (Pro or higher), so the store can't go live on `norskgjenbruksmarked.no`
   without it, and Lovable's own agent only gets a few credits a day on Free.
2. **Connect the Lovable project to GitHub** (Lovable editor → GitHub icon →
   Connect → create repository under `norskgjenbruk888`). Lovable creates a new repo
   and keeps it in two-way sync. Then make sure the Claude GitHub app can access that
   repo. With this I write and test the code directly and push it; Lovable picks it up
   automatically and **no Lovable credits are used** for the build.
3. **Re-connect Shopify** in claude.ai → Settings → Connectors (it currently asks for
   sign-in). I need it to read your real tags, metafields, collections and shipping
   rates so the filters and delivery messages match the shop.
4. **Delivery rule:** which is true today –
   (a) free delivery everywhere in Bergen, or
   (b) free over 6 000 kr in the ≤ 55 km zone, otherwise 999 kr (+250 kr carry-in,
   +250 kr assembly), as in `src/lib/checkout.ts`?
5. *(Optional, makes testing much better)* allow these hosts in this Claude
   environment's network settings so I can test against real data:
   `802df2-2.myshopify.com`, `cdn.shopify.com`, `norskgjenbruksmarked.no`,
   `lovable.app`, `secundo.no`.

## 7. Rollback

Lovable keeps full history. If anything is wrong after publishing, restore commit
`b5cae0aaec832c9c9f26a8c4b190845ed94cf2af` in Lovable (or revert in the synced
GitHub repo) and publish again.
