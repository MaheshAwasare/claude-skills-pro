---
name: algolia-search
description: Production Algolia setup — index design, attribute configuration, ranking and relevance, faceting, secured API keys (per-user filters), search-as-you-type UI patterns, and the cost discipline that prevents bill shocks. Use when adding search to a SaaS product or replacing Elasticsearch with managed search.
---

# Algolia Search

Algolia gets you to "search bar that feels instant" in a day. The risks: misconfigured indices that bill 10x what you expected, leaked search keys, and ranking that returns garbage because you never read the relevance docs.

## When to use

- SaaS product needing in-app search (docs, CRM, product catalog).
- E-commerce with faceting (filter by brand, price range, category).
- Search-as-you-type UX over O(thousands–millions) of records.
- Replacing Elasticsearch when you don't want to operate ES.

## When NOT to use

- Hyperscale (>100M records) where Algolia's pricing doesn't make sense — Typesense / Meilisearch self-hosted, or OpenSearch.
- Full-text indexing of >10MB documents per record — Algolia's per-record limit will fight you.
- You need vector / semantic search as the primary mode — pgvector / Pinecone / Weaviate.

## Index design

One **index** per logical search context. For a SaaS app:
- `prod_articles`, `prod_users`, `prod_projects` — separate indices.
- Replicate per env: `staging_articles`, `prod_articles`. Never share.
- For sort variants of the same data, use **virtual replicas** (cheaper than full replicas).

**Don't** stuff multiple entity types into one index "to search everything." Use multi-index search at query time.

## Record shape

Records are JSON objects, max ~10KB each (soft limit; hard 100KB).

```json
{
  "objectID": "article_abc123",
  "title": "Razorpay subscriptions",
  "excerpt": "Setting up auto-debit mandates...",
  "content_searchable": "long body trimmed to 5KB...",
  "tags": ["razorpay", "payments", "india"],
  "author": "mahesh",
  "published_at_unix": 1714838400,
  "popularity": 42
}
```

Key tricks:
- **`objectID`** is your primary key — match it to your DB ID for upserts.
- **Trim long content** — index a search-friendly excerpt, not the full doc.
- **Numeric `_unix` timestamps** for `customRanking` (Algolia ranks numerics, not strings).
- **Pre-compute popularity / boost signals** — don't expect Algolia to know which articles are good.

## Attribute configuration (the bit nobody reads)

In Algolia dashboard or via API, configure per index:

| Setting | What it controls | Recommended |
|---|---|---|
| `searchableAttributes` | Which fields are searched | List in priority order: `title`, `tags`, `content_searchable` |
| `attributesForFaceting` | Which fields support filters | `searchable(tags)`, `author`, `category` |
| `attributesToRetrieve` | Which fields client gets back | Only what UI needs (saves bandwidth) |
| `customRanking` | Tie-breaker after textual relevance | `["desc(popularity)", "desc(published_at_unix)"]` |
| `ranking` | Algolia's full ranking pipeline | Default is good; rarely change |

```ts
import algoliasearch from "algoliasearch";
const admin = algoliasearch(APP_ID, ADMIN_KEY);
const index = admin.initIndex("prod_articles");

await index.setSettings({
  searchableAttributes: ["title", "tags", "content_searchable"],
  attributesForFaceting: ["searchable(tags)", "author"],
  attributesToRetrieve: ["title", "excerpt", "tags", "objectID"],
  customRanking: ["desc(popularity)", "desc(published_at_unix)"],
  highlightPreTag: "<mark>",
  highlightPostTag: "</mark>",
});
```

**Commit settings to code, not the dashboard.** Dashboard edits aren't versioned and disappear on env rebuild.

## Indexing pipeline

```ts
// Whenever an article changes in your DB
async function indexArticle(article: Article) {
  await index.saveObject({
    objectID: article.id,
    title: article.title,
    excerpt: article.excerpt,
    content_searchable: article.body.slice(0, 5000),
    tags: article.tags,
    author: article.authorSlug,
    published_at_unix: Math.floor(article.publishedAt.getTime() / 1000),
    popularity: article.viewCount,
  });
}

// Bulk: full reindex via temp index then atomic move
async function fullReindex(articles: Article[]) {
  const tmp = admin.initIndex("prod_articles_tmp");
  await tmp.saveObjects(articles.map(toRecord), { autoGenerateObjectIDIfNotExist: false });
  await admin.copyIndex("prod_articles_tmp", "prod_articles", {
    scope: ["records"],                               // settings stay
  });
  await tmp.delete();
}
```

`copyIndex` is the standard pattern for re-indexing without serving 0 results during the rebuild.

## Secured API keys (the way to filter per user)

NEVER ship the Admin Key to the browser. The Search-Only Key is fine to ship — but if results need user-scoped filtering, use **secured API keys**.

```ts
// Server-side
import { generateSecuredApiKey } from "algoliasearch";

function tokenForUser(userId: string, orgId: string): string {
  return generateSecuredApiKey(SEARCH_KEY, {
    filters: `org_id:${orgId} AND (visibility:public OR allowed_users:${userId})`,
    validUntil: Math.floor(Date.now() / 1000) + 60 * 60,    // 1h
    userToken: userId,                                       // for analytics
  });
}
```

The token is verified server-side by Algolia at query time. Even if a user inspects network requests, they can't broaden the filter — Algolia rejects.

## Search UI

```ts
import { liteClient } from "algoliasearch/lite";

const search = liteClient(APP_ID, securedKeyFromServer);
const { hits } = await search.searchSingleIndex({
  indexName: "prod_articles",
  searchParams: {
    query: "razorpay",
    hitsPerPage: 10,
    facetFilters: [["tags:payments", "tags:india"]],         // OR within array
    attributesToHighlight: ["title", "excerpt"],
  },
});
```

For React, use `instantsearch.js`/`react-instantsearch` for the standard UI patterns (SearchBox, Hits, RefinementList, Pagination) — saves a week.

## Cost discipline

Algolia bills on:
- **Records** stored.
- **Operations** (each search counts; reindex of 10k records = 10k operations).

Footguns:
- **Reindexing whole index on every change** — bills explode. Use partial updates (`saveObject` for one record).
- **Per-user "saved searches" run on each page load** — caching helps; Algolia doesn't dedupe.
- **Indexing every keystroke as a "search"** — debounce client-side (200ms) before firing.
- **Replicas duplicating storage** — virtual replicas for sort orders, not full replicas.

Set spending alerts in dashboard. Track the **search/record ratio** — if above 20:1/month, indexing is fine; if 100:1, look for accidental hot loops.

## Anti-patterns

- **Admin Key in browser** — full account compromise. Search-Only or Secured key only.
- **Filter via post-processing in client code** — leaks data through the network layer. Use `filters` or secured key.
- **Full content in records** — wasteful storage; truncate to a search-friendly excerpt.
- **No `objectID` mapping to your DB** — upserts become inserts; index drifts.
- **Settings in dashboard only** — not version-controlled; disappear on env rebuild.
- **One index for users + projects + articles** — ranking can't be tuned per type.
- **Re-indexing the world on every save** — partial updates exist for a reason.
- **No facet UI when records have facetable attributes** — UX gap; users can't refine.
- **Forgetting `searchable()` on faceted fields** — facets don't appear in search-as-you-type results.
- **Synthetic search load tests against prod index** — bills your account. Use a separate `loadtest_*` index.

## Verify it worked

- [ ] First letter typed shows results in < 50ms (regional p95).
- [ ] Admin Key never appears in browser network tab; only Search-Only or Secured key.
- [ ] Cross-tenant search: user in org A cannot see records from org B (verify by inspecting raw query → Algolia logs).
- [ ] Index settings are committed to code, applied via deploy (not dashboard).
- [ ] Full reindex via tmp index → copy → delete; users see no downtime.
- [ ] Partial update: changing one article in DB re-indexes only that record (not the world).
- [ ] Facets render with counts; clicking a facet filters results.
- [ ] Highlight markers (`<mark>`) appear in `_highlightResult` for matched terms.
- [ ] Spending alert configured at expected monthly threshold.
- [ ] Search/record op ratio is reasonable (<30:1 typically).
