# Illumify — data through `@illumify/sdk`

Everything your page reads goes through `@illumify/sdk`: framework-agnostic, zero runtime
dependencies, plain `fetch`. Load this **before you plan a feature**, not before you write the fetch —
several of the constraints below decide whether a feature is possible at all.

**The package's own `README.md` is the API reference**, and it ships with the package:
`node_modules/@illumify/sdk/README.md`. Its TypeScript types are the contract, synchronised field by
field against the backend's generated response classes. This skill does not restate the signatures; it
carries what is true about the *platform* and what you would otherwise discover late.

## The whole surface

```ts
import {
  getSession, getItems, getItem, getFilters,   // the four catalog reads
  createItemPager,                             // walking the catalogue safely
  recommendedOffer, offerPrice,                // the offer model, made compile-checked
  IllumifyApiError,                            // every failure
  CatalogRevisionChangedError,                 // 409 — restart the walk
  CatalogCursorRejectedError,                  // 400 InvalidCursor — a new query is a new walk
} from "@illumify/sdk";
```

Four `GET`s, and that is the entire data contract:

```ts
getSession(options?):            Promise<CatalogSession>
getItems(query?, options?):      Promise<CatalogItemsResponse>
getItem(skuId, options?):        Promise<CatalogItemDetailResponse>
getFilters(query?, options?):    Promise<CatalogFiltersResponse>
```

There is no fifth and **no write of any kind in this package**. `options` is `{ signal }`, ordinary
cancellation.

**The cart and the checkout are not here, and they are not missing either.** They live on the injected
global as `getCart`, `replaceCart`, `prepareCheckout` and `checkout`, because a write has to carry a
request header and an `Origin` that only the platform's own injected adapter can supply — a package
that offered you `addToCart()` would be offering a `403`. `illumify skills get app` and
`src/storefront.d.ts` are the reference for those four. Everything below about offers, revisions and
prices is what you need in order to *call* them correctly.

**The package builds no URL and holds no credential.** Every read is appended to
`window.IllumifyStorefront.config.apiBaseUrl`, which already carries the request's authority. There is
no origin resolution, no environment detection, no path assembly, no authority branching, and no
`Authorization` header on any request in any environment — all four are enforced by checks in the
package's own build.

**It does not wrap `navigate`, `login` or `logout`.** Those are injected on the global and driven by
the `data-illumify-storefront-*` attributes with no JavaScript at all. `illumify skills get app` has
them.

## Read this before you plan anything

Nine facts. None of them is something the SDK can work around, and each one has a design behind it
that will otherwise be discovered after the page is built.

### 1. The same theme is served under four different bases

`apiBaseUrl` differs depending on how the shopper arrived — anonymous, a customer link
(`/c/{token}`), a signed-in account (`/_account`), or a hosted preview (`/preview/{capability}`).

**Build every call from the injected `apiBaseUrl`; a hand-built path silently serves one audience the
wrong data.** A customer link's token in the URL *is* its authority — there is no identity cookie and
no shopper session anywhere in this contract — so a URL you recomposed from a slug strips the token and
the server answers public pricing where it meant customer pricing. Nothing errors and nothing looks
wrong.

The SDK never composes one, which is why you cannot make this mistake through it. You can still make
it with a raw `fetch`, so do not write one.

All four **reads** exist under all four authorities with identical shapes, **including preview**. A
theme is written once and never branches on the authority for data. The cart and checkout writes are
the exception, and they are not in this package: they exist under three of the four, and a document
served with `accessMode: "Preview"` has **none of** the four cart methods on its runtime at all — so a
preview can read a catalogue and cannot write a cart. See `illumify skills get app`.

### 2. `409 CatalogRevisionChanged` is an instruction, not a failure

**The catalogue moved or the cursor is older than 15 minutes, and the walk restarts.** Cursors are
keyset positions pinned to a revision, not offsets, so there is no way to resume — and the server maps
"expired" and "the catalogue changed" to the *same* 409 on purpose, so you cannot tell which happened
and must not write a message claiming one.

What a theme must do is two things, and the second is the one that gets forgotten:

1. restart from **no cursor**;
2. **discard everything already accumulated** before appending the new first page.

Skip the second and you render the start of the catalogue twice, under a live shopper, at prices that
may have just changed — and nothing about the page looks broken.

**`createItemPager` exists for this.** It owns the accumulated list, so you render `pager.items` and
never append; it pins its query at construction, so a filter cannot change mid-walk; and it serialises
`loadMore()`, so a double-clicked button cannot fetch the same window twice.

```ts
const pager = createItemPager({ search: "gelato", sort: "priceAsc" });

button.onclick = async () => {
  const page = await pager.loadMore();
  render(pager.items);                       // never `items.push(...)`
  button.hidden = !page.hasMore;
  if (page.restarted) toast("Prices changed — the list was refreshed.");
};
```

It absorbs two restarts by default (`maxRestarts`) and then rethrows, at which point your own error
state is the right answer. A `404`, a `400 InvalidQuery` and a `503` are not restartable and pass
straight through.

`CatalogCursorRejectedError` (`400 InvalidCursor`) is a **different** failure: the cursor is not
stale, it belongs to a different walk. Almost always that means a filter, the search text or the sort
changed while the cursor was kept. A new query is a new walk — and a new pager.

Use `getItems` directly only for a one-shot read or when you are certain there is one page. It hands
you the raw `nextCursor`, and the two ways to lose with it are the two above.

**An empty catalogue is a valid answer.** `items: []` with `nextCursor: null` means this storefront has
no live-priced SKUs, not that anything failed. Render the empty state.

Do not branch page-exhaustion off `deliveryMode`. `"complete"` means the whole filtered catalogue came
back in one response and `"windowed"` means a keyset window, but a windowed response can be the last
one. The signal for "there is more" is a non-`null` `nextCursor`.

### 3. An offer is facility-neutral merchandise identity

The full identity is `site + skuId + batchId? + unitOfMeasureId + stockCompanyId`. Different UOMs,
item/batch grains and seller sources are separate offers; the buyer facility is not. One offer has one
`offerId`, `offerRevision`, price and `availableQuantity` whatever destination later receives it. Offers
no longer carry `customerFacilityKey`.

`offerId`, `offerRevision` and `stockCompanyKey` are opaque. Never parse or compose them. `offerId` is
stable commercial identity; `offerRevision` is context-bound change detection; the envelope `revision`
is the whole read model state used by cursors. A known customer's `customerFacilityKey` comes only from
`session.facilityList` and goes on each cart line byte-for-byte.

### 4. Missing money is `null` and never `0`

`unitPrice` is **gross**. `discountPercent` is resolved independently and stays separate — do not
collapse them. `effectiveUnitPrice` is the server-rounded display **net**. All three come from the
platform's shared pricing calculator, which owns commercial rounding: **display them, never recompute
them.** A net price you recalculate in JavaScript will disagree at the cent with the one an order
would be written at.

**A legitimate zero price is `0`. `null` means there is no price, and an offer with no price is
non-orderable.** So `unitPrice ?? 0` renders an unpriced item as free.

`offerPrice(offer)` makes that impossible to write by returning a discriminated union:

```ts
const price = offerPrice(offer);

if (price.kind === "unpriced") {
  label.textContent = "Call for pricing";           // price.orderabilityCode says why, when it can
} else {
  label.textContent = format(price.effectiveUnitPrice, session.currencyCode);
}
```

`msrp` and `availableQuantity` are `null` when absent on the same terms. `availableQuantity` is
advisory in that offer's own UOM and is **not reserved** — it can be gone by the time an order is
placed.

### 5. The recommended offer comes from `recommendedOfferId` — never from the lowest price

The server orders a SKU's offers and names one explicitly. **A theme must not infer the
recommendation** — not the cheapest, not `offers[0]`, not the largest pack. A guess agrees with the
server exactly often enough that nobody notices it is wrong.

`recommendedOffer(item)` matches the named id and returns `undefined` rather than falling back to
anything. `undefined` is worth rendering as "choose a size", not as a default. The two price sorts
(`priceAsc`, `priceDesc`) sort on this offer, so rendering it shows the number the list was ordered
by.

`item.available` is the server's own answer, computed as "at least one offer is orderable". **Render
off it and do not re-derive it.** A visible-but-unavailable SKU stays in the catalogue on purpose, so
render the unavailable state rather than filtering it out.

### 6. Signing in is not a discount

**`shopperContext: "Identified"` means an identity resolved, not that it matched a seller customer.**
The separate field is `pricingContext`, and it is the one that answers the pricing question:

| | |
| --- | --- |
| `shopperContext` | `"Anonymous"` or `"Identified"` — did the server resolve an identity |
| `pricingContext` | `"Public"` or `"Customer"` — which price basis it applied |

A shopper can be `"Identified"` on `"Public"` pricing, and that is a normal outcome. Licence matching
normalises the licence and its jurisdiction and then: **no match leaves an identified shopper on base
pricing, one match selects the seller's customer, and ambiguity fails closed.** There is no default
pricing segment to rescue them.

**A page promising a discount for signing in will be wrong for every shopper whose licence matched
nothing.** The honest wording is the platform's own: the licence lets the seller apply existing
customer pricing, if they have a record for it.

Two more consequences:

- **Do not gate a price display on either field.** Render whatever came back — the number is already
  resolved correctly for whoever asked, before and after a sign-in.
- **Do not try to tell which customer they matched.** There is no customer id, no segment id, no
  reason code and no price basis anywhere in these responses. The page is told what this shopper pays
  and is given no way to learn for whom. That is the design, it is enforced rather than incidental,
  and it is not something to expect relaxed.
- **Do not render an enrolment form of your own.** The platform owns first-login onboarding, including
  which buyers see it and when, and a second one would collect details nothing reads.

### 7. Every shopper failure is the same bare 404

The catalog API answers **`404` with an empty body** for every ownership failure, deliberately, so a
probe cannot tell them apart. Indistinguishable: an unknown or inactive slug, a storefront with no
assigned theme, a revoked or rotated customer-link token, an expired preview capability, and a `skuId`
outside this catalogue.

`error.body` is `undefined` and `error.code` is `undefined`. **Write handling that does not branch on
the cause, because it cannot.** The SDK's message names all five possibilities for whoever reads the
console.

The failures that *do* carry a code:

| | |
| --- | --- |
| `409 CatalogRevisionChanged` | restart the walk — fact 2 |
| `400 InvalidCursor` | the cursor belongs to a different walk — fact 2 |
| `400 InvalidQuery` | a malformed or non-positive id, an unknown sort, a repeated scalar parameter |
| `503 CatalogCapacityExceeded` | this storefront's catalogue is past the bounds the API will hydrate. A seller-side data condition, not a transient outage: retrying answers the same. Narrow the query |

Every failure throws `IllumifyApiError` with `status`, `code` and `body` — the server's body exactly as
received, reshaped by nothing. `status` is `0` when the request never got a response. The SDK **never**
touches `window.location`, never redirects and never retries; reacting to a failure is your page's
business.

### 8. Your page cannot call any third-party API

`connect-src 'self'`, and `script-src`/`img-src`/`font-src` are `'self'` too — so a CDN script does not
even load. The full policy, and what it means for a design, is in `illumify skills get app`. **Plan for
a page whose only data source is this SDK.**

`illumify dev` sends no CSP, so an external call works in a preview and fails deployed.

### 9. Facilities are per-line destinations over one catalogue

A known customer's `facilityList` contains opaque `{ customerFacilityKey, name }` choices. Read the
catalogue once without a facility parameter: pricing, availability, offer identity and revisions do not
vary by destination. Main ignores a stray `customerFacilityKey` query parameter, but do not send one.
Changing a selector is local UI state and must not restart a pager or refetch detail or filters.

Every known-customer cart line must carry one exact session key and no `guestAllocationKey`. Its identity
is `(offerId, customerFacilityKey)`, so the same offer may be sent to several facilities as separate
lines. Those quantities share the offer's one `availableQuantity`; exceeding the sum rejects the whole
replacement with `409 InsufficientInventory`. Key drafts by the compound identity or one destination
will silently overwrite another.

## Product media exists, and it arrives as a complete URL

**Product images are a contract field.** They come from real ERP document mappings, as same-origin
URLs served with immutable public caching:

```ts
item.primaryImage         // CatalogProductImage | null — also imageList[0] on a detail
detail.item.imageList     // the full list, detail endpoint only
```

Each carries `url`, `alt`, `contentType`, `width`, `height` and `contentHash`. **Render `url`
directly.** `width` and `height` are `null` when the ERP document carries no dimensions, so a layout
that needs them must handle their absence.

**A theme cannot construct one of these URLs**, and must not try: media is never inferred from a
filename, a `skuId`, or anything in the theme's own assets. `assetBaseUrl` serves **this theme's own
files** and never product media. If `primaryImage` is `null`, there is no image — render a placeholder
you shipped in the theme.

## Filtering and search

```ts
const { filters } = await getFilters();
const pager = createItemPager({ categoryIds: [filters.categories[0]!.id], sort: "nameAsc" });
```

- **OR within one group, AND across groups.** `categoryIds: [4, 9]` means either category;
  `categoryIds` plus `brandIds` means both must hold. An empty array is the same as an absent one: no
  constraint.
- Facet groups: `categories`, `brands`, `inventoryCategories`, `classes`, `tags`. Each value has
  `id`, a stable machine `key`, a display `name`, `itemCount` and `sortOrder`. Prefer `key` over
  `name` for anything you persist.
- **`tags` is empty for every storefront today** — the ERP has no authoritative SKU-tag relationship,
  so the facet is empty and a requested `tagId` matches nothing. A known limitation, not a bug to
  work around.
- Counts reflect the query you pass: pass the currently applied filters for co-varying counts, pass
  nothing for catalogue-wide ones.
- `getItems` already returns the same facets on `response.filters`, so a first paint needs one request
  rather than two. `getFilters` is for refreshing facets without refetching items.
- **`getFilters` rejects `cursor` and `sort`** with `400 InvalidQuery` rather than ignoring them.
  Neither member exists on its query type, so that refusal is a compile error instead of a runtime
  one.
- Sorts, and this is the whole accepted set: `default` · `nameAsc` · `nameDesc` · `priceAsc` ·
  `priceDesc`. `default` is deterministic name plus `skuId` — **there is no merchandising order and no
  "featured" sort to ask for.** Anything else is `400 InvalidQuery`.
- Search text is normalised server-side (Unicode FormKC, case, whitespace), so send the shopper's text
  unchanged.
- **Search is capped at 400 characters, and the cap is a refusal rather than a truncation** — a longer
  term is `400 InvalidQuery`, not a search on the first 400. It is measured **after** normalisation, so
  the number is not the length of what the shopper typed: whitespace collapses, and a composed sequence
  can grow. Cap your input at 400 and you are inside it either way; if you show a counter, count the
  raw text and treat the limit as approximate.

`getItem(skuId)` takes **no query parameters at all** — filters, search and sort are ignored there
rather than rejected. It adds `description` and the full `imageList` over what the list projects;
everything else is identical, so a detail view does not need the list response. A non-integer or
non-positive `skuId` throws a `RangeError` locally rather than reaching the server as a 404 that would
read as "no such product".

## `getSession()` — call it for the five things the global does not carry

`shopperContext` and `themeKey` are already on `window.IllumifyStorefront.config`, so a page that reads
only those does not need this call. (`slug` is **not** on the injected config — nothing there names the
site. It arrives here, and it is a label rather than something to build a URL from: every URL you need
is already on `config`.) These five are the reason to call:

| | |
| --- | --- |
| `pricingContext` | `"Public"` or `"Customer"` — the field to reason about prices with |
| `currencyCode` | ISO 4217 for every money value in the catalogue |
| `accessPolicy` | `customerLinkEnabled`, `signInEnabled`, `signInRequiresInvite` |
| `facilityList` | Known-customer cart destinations, each `{ customerFacilityKey, name }`; empty for a public shopper. See fact 9 |
| `stateList` | The US states a **guest** may name a destination in, each `{ id, name, shortName }`, ordered by name. Submit the chosen `id` as `guestFacilityList[n].stateId`. **Empty for a signed-in or customer-link shopper** — they cannot submit a guest destination — so read its length before rendering the field. `illumify skills get app` has the whole rule |

**`session.themeKey` is `null` under `illumify dev --fixtures`** — nothing has been uploaded, and a
made-up 64-hex value would name nothing while reading as an answer. The type says `string`, so a page
rendering it prints `null` with no compile error. `IllumifyStorefront.config.themeKey` is declared
optional and is the one to read.

`currencyCode` is **always `"USD"` today**, because the ERP has no authoritative per-storefront
currency source. Read it rather than hardcoding it, but do not build a currency switcher on it.

`accessPolicy` is how you decide whether to render a sign-in affordance at all. **Treat
`signInEnabled && signInRequiresInvite` as "sign-in is not available yet"**: invite administration and
claiming are not implemented, so that combination admits nobody and a login button there cannot
succeed.

## Caching: prices are not cacheable

`session` is served `private, no-store`. The list, detail and filters reads are `private, no-cache`
with weak ETags. **Nothing derived from a priced response belongs in `localStorage`,
`sessionStorage`, or a service worker** — and a service worker is forbidden by the CSP anyway. Prices
are resolved per request, for whoever asked.

## The cart is real, and it is not in this package

**A cart and a checkout exist**, and a checkout produces genuine Sales Orders in the seller's ERP. They
are reached through `getCart`, `replaceCart`, `prepareCheckout` and `checkout` on
`window.IllumifyStorefront` — not through this package, because a cart write must carry a request header
and an `Origin` only the platform's injected adapter supplies. `illumify skills get app` is the
reference, and `src/storefront.d.ts` in your project has every field.

What this package is for is getting the values those calls need **right**, and three of the facts above
are the ones a cart depends on:

- A cart line is `{ offerId, quantity, observedOfferRevision }` plus exactly one destination: an opaque
  session `customerFacilityKey` for a known customer, or your `guestAllocationKey` for an anonymous line. **A client-composed SKU, batch, UOM, conversion
  or price is never trusted**, so there is nothing to be gained by sending one. Hold the `offerId` and
  the `offerRevision` from the read the shopper actually saw (facts 3 and 5).
- A stale `observedOfferRevision` is refused with **`409 PricingChanged`**, nothing is mutated, and the
  refreshed cart arrives on the error. That is the same shape of instruction as `409
  CatalogRevisionChanged` (fact 2), and a theme that treats either as an error is broken in the same
  way: show the new number, do not retry the refused body.
- All destinations of one offer share stock, and line identity includes the destination key (fact 9).

Invites and buyer enrolment remain outside this phase.

## You never hold a credential

The SDK reads no credential — not from `import.meta.env`, not from `process.env`, not from a config
object you pass in. Anything the SDK could read, a bundler inlines as text into the built bundle, and
a credential the SDK can read is a credential that ships to every visitor.

**There is no `Authorization` header on any request this package makes, in any environment, ever.**
There is nothing to attach: customer-link authority is the URL, account authority is the browser's own
Illumify identity on `/_account`, and there is no identity cookie and no shopper session in this
contract at all. (The platform does set exactly one cookie — the **anonymous cart's** lookup token,
storefront-scoped and `HttpOnly`, stored server-side as a hash. It finds a cart and nothing else: it
carries no identity and never changes which shopper the server resolves. These four reads neither send
nor need it.) `fetch`'s own `same-origin` default is exactly right, and the package deliberately
sets no `credentials` option so nobody "fixes" it to `include`.

`illumify dev` attaches nothing on the shopper lane either, and that is deliberate: a bearer there is
at best ignored and at worst changes which shopper the server resolves — a preview showing different
prices than production, with nothing anywhere saying so.

**If you need to reach an Illumify service operation that is not one of these four reads, that is not
a storefront theme's job.** A deployed shopper holds no ERP session, so such a call cannot work for the
audience the site is for. `illumify dev` can credit one during development from the owner's
`ILLUMIFY_API_KEY` — the same key `upload` uses — **and `illumify skills get deploy` has that key's
rules, which matter**; do not read the value, do not fill it in, and do not weaken error handling you wrote for a
buyer in order to make your own preview work.

## Local development reads real data, unless you ask for fixtures

`illumify dev` proxies to whatever `ILLUMIFY_API_HOST` names. Nothing is faked and it is not a
sandbox: what you read is that environment's real catalogue. Pointed at a shared environment, that is
somebody's live data.

**`illumify dev --fixtures` is the other mode**, and it answers the catalogue plus an in-memory cart and
checkout inside the CLI process: no link, credential, or request leaving the machine. Reach for it to build
against shapes a live catalogue will not hand you on demand — `--fixtures empty` for the empty state,
`--fixtures restart` for the `409` walk restart in fact 2, **`--fixtures facilities` for fact 9**, and
`--shopper-context identified` for customer pricing (which really changes the numbers there, unlike
against a real environment).

**`--fixtures facilities` serves a known customer with three destinations over one fully priced
catalogue.** Offers contain no facility key. Use the same offer in two cart lines with different session
keys to exercise compound identity and shared stock. The fixture cart and checkout run in memory;
`pricing-change`, `low-stock`, `facility-removed` and `pending-checkout` expose recovery paths.

One thing to know while writing code against it: `session.themeKey` is `null`, and the type says
otherwise. `illumify skills get deploy` has the scenarios and the rest of the divergences.

## Nothing here has been seen on the wire yet

Stated plainly because the alternative is a false sense of measurement. The shopper routes are **not
yet proxied on the shared environments** — the catalog prefix answers a redirect, the signature of an
unrouted prefix — so no shape in this package has been exercised against a live storefront. Every wire
shape is synchronised against the backend's generated contracts and checked in both directions, and
every behavioural claim is read from the server's source. None of it is *measured*.

The cart and the checkout are one degree less proven still. Their contract is read from the server's own
source — routes, request and response shapes, error codes, per-line destinations and shared stock — and
the backend's own focused tests are green, but the cart slice's end-to-end cases were **stopped before
revalidation** rather than passed. So build against the shapes with confidence and treat the first live
run as the first live run.

What that means for you: the shapes are trustworthy and the plumbing is not proven. If a call answers a
redirect or a 404 that none of the causes above explain, suspect the route before you suspect your
code, and say what you ran rather than working around it.

## The SDK is not a security boundary

An endpoint is anonymous because of where it is mounted, not because of anything in this library. The
SDK is ergonomics and forward-compatibility. Do not reason about safety from the fact that a method
exists or does not.
