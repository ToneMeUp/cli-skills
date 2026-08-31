# Illumify — the storefront theme contract

What a valid Illumify **storefront theme** is, how it declares itself, and which of the platform's
rules fail silently. Load this before editing `illumify.config.ts`, adding a page, writing routing, or
reading the injected global.

## What you are building

A **theme**: an ordinary static site — HTML plus assets, built with anything or written by hand.
Illumify serves it as its own browser document under a same-origin prefix. There is no host page
around it, no iframe, and no framework of ours in the way.

- The router is yours. Real URLs, real `pushState`, any library or none.
- CSS is yours; nothing leaks in or out.
- Data calls are same-origin, so there is no CORS to fight — and, as the CSP section below shows,
  no third-party API you can call either. `illumify skills get data` owns the data surface.

The boundary is the **API, not the pages**. You own every pixel. The server owns prices,
availability, and who is looking.

## Source structure

These are Illumify rules, not a React style guide:

- **Mirror the route list.** Put one page module in `src/pages/` for each route declared in
  `illumify.config.ts`, whatever those routes are served by. The module is the unit somebody edits,
  replaces or deletes later — and "somebody" is usually a coding agent working from a description of
  one page, not a person who has read the file. A route that lives inside a shared component cannot
  be handed over that way.

  **This holds when every route declares the same `entry`.** Client-side routing is the normal case,
  not an exception to the rule: several routes serving one built `index.html` is expected, and it
  changes nothing about where the code for each of them belongs. The routing table and the file
  layout answer different questions.

  Routes also match exactly and never fall through, so mirroring keeps a route without its module,
  and a module nobody declared, visible before a preview rather than as a silent 404.
- **Keep catalogue access in one module.** Put the four `@illumify/sdk` catalogue reads and the
  pager in `src/catalog/`, then have pages call that module. Its `null` money,
  `recommendedOfferId` and `409`-restart rules are sharp; copies in components eventually render the
  wrong offer or restart pagination incorrectly, only for particular facilities or cursors.
- **Read the runtime boundary once.** Read the frozen `window.IllumifyStorefront` at the application
  edge and pass what pages need explicitly. Reading an ambient global in many components hides the
  runtime dependency and makes a preview-only difference look like a page bug.

The default scaffold remains deliberately empty: these rules describe where the owner's pages and
catalogue code go once they exist; they do not choose a page, route or feature.

### A theme is a name holding immutable revisions, and uploading is not publishing

A theme carries the display name you chose and a stack of **immutable revisions**. Uploading appends
one; nothing is ever replaced or overwritten. What a shopper is served is decided by an **assignment**
on a storefront site, and moving an assignment is an administrative act performed in the ERP by whoever
owns the site. No command in this CLI can move one.

**But "uploaded never means live" is retired — changed 2026-08-28.** An assignment points at a theme,
with an *optional* pin to a particular revision. Unpinned, which is the default, it serves the newest
revision on every request. So if a site already serves this theme, appending a revision to it is live
immediately, with no assignment moved.

So do not report either way from the fact of a successful upload. `illumify upload` prints which of the
two happened — and says "cannot tell" rather than guessing when it cannot. Report what that line says.
See `illumify skills get deploy`.

### The site is not yours to create, and neither is its slug

A **storefront site** lives in the ERP. A person creates it there and sets its slug, its access
policy, its customer links, and which theme it serves. Your project builds a theme and does nothing
else to the site.

So nothing in this project names the site. `illumify link` asks the ERP and records the answer in
`illumify.link.json`, which is committed. "Not linked to a storefront site" is an expected state, not
a broken one — ask the owner for the **profit centre id**. The ERP resolves the root company and slug; do not guess either.

## `illumify.config.ts`

The one file that is ours and yours at once. It default-exports an object whose only required member
is `pages`:

```ts
export default {
  pages: [
    { route: "/", entry: "index.html" },
    { route: "/about", entry: "about.html" },
  ],
};
```

| Field | Rules |
| --- | --- |
| `pages[].route` | Starts with `/`. A trailing slash is stripped; `/` alone is the home page, and **one page must declare it**. At most 256 characters, at most 256 pages. **Case is preserved and matching is case-insensitive**, so two routes differing only in case are one route and declaring both is an error. Rejected: `//`, `.` and `..` segments, `\`, `?`, `#`, `%`, control characters, and anything under the eight reserved first segments below |
| `pages[].entry` | Path of an `.html` or `.htm` file **in the build output** (`dist/`), not in `src/`. Relative, no `..`. Several routes may share one entry |
| `previewImage` | Optional. A path in the build output, recorded on the theme as a thumbnail. It must exist in the output or the upload is refused |
| `build.script` | Optional. The `package.json` script that produces the static site. Defaults to `build:app` |
| `build.outDir` | Optional. Where that script writes. Defaults to `dist` |

**There is no `type`, no `slug`, no `targets` array, no `access` level and no `themeKey`**, and none
of them was renamed. A config still declaring `targets` is refused with a message saying so rather
than half-read.

**Why `access` is gone.** Whether a shopper must be identified is the *site's* access policy, set in
the ERP by whoever owns it. A file in your project has no say in it, so the field does not exist
rather than being accepted and ignored. What your page *can* read is which kind of shopper it is
serving right now — `shopperContext`, below.

**Why `themeKey` is not yours to write.** `illumify build` derives one from this file plus every byte
of the build output, and the *server* derives its own for every revision it stores — a value you cannot
compute and should never compare with the local one. Neither is a field for you to set. What is worth
knowing is the consequence: editing this file changes the theme's content identity, so a rebuild is a
new revision even when every page byte is the same.

## Route matching is EXACT, and nothing falls through

**This is the single thing about this platform that differs from every other static host, and getting
it wrong is a 404 on a page you can see working in your framework's dev server.**

The server normalises the request path and looks it up in `pages` with a case-insensitive **equality**
comparison. Not longest-prefix. So:

```ts
pages: [{ route: "/", entry: "index.html" }]
```

serves `/` and **nothing else**. `/products`, `/products/7` and `/about` are all 404 — declaring `/`
does not make them reach it.

**There is no fallthrough of any kind.** A miss does not fall back to a parent route, and it does not
fall through to a *file* either: a request for `{document}/assets/app.js` is a 404, because a theme's
files are served from a different base entirely (see below). Both halves matter — the second one used
to work on the retired platform and does not now.

A single-page app that routes client-side therefore needs **one entry per route it owns**:

```ts
pages: [
  { route: "/",         entry: "index.html" },   // the same built file
  { route: "/products", entry: "index.html" },   // answers all three
  { route: "/about",    entry: "index.html" },
]
```

That is legal and normal: one built `index.html` can answer any number of routes while your router
decides what to render. What it cannot do is answer a route nobody declared.

**A route with a path parameter has no shape you can declare.** `/products/:id` is not something the
manifest expresses, so a detail page addressed as `/products/7` needs either a declared route per id
(no) or a query string on a declared route (`/product?sku=7`), which is what to build. Query strings
are not part of route matching, so one declaration covers every value.

**A query string is welcome everywhere a route is.** `pages[].route` itself must not contain `?` — the
declaration is the path — but every runtime helper takes one: `routeUrl("/product?sku=7")` and
`navigate("/product?sku=7")` resolve the path against the document base and carry the query (and a
`#fragment`) through untouched, and `data-illumify-storefront-route="/product?sku=7"` produces an
`href` with the query on it. Nothing throws and nothing strips it. Read `location.search` on the way
in, exactly as you would anywhere else.

### Eight reserved first segments

`api` · `_auth` · `c` · `_account` · `preview` · `admin` · `internal` · `management`

A page declared under one of these is refused at upload, **and** a document request under one is
404ed before the manifest is even consulted — so such a page is unreachable twice over. `api` is your
own catalog API, `_auth` is the sign-in endpoints, and `c`, `_account` and `preview` are the prefixes
a document is served under when a customer link, a signed-in account or a hosted preview supplies the
authority. The other three the platform holds back.

Only a whole *first* segment is reserved: `/apiary`, `/account` (no underscore), `/previews` and
`/docs/api` are all fine.

## What the server injects, and where

Before any of your scripts run, the server writes two things into the document, in this order,
**immediately after the `<head …>` open tag** — ahead of your `<meta charset>` and every script you
wrote:

1. an inline `<script>` that builds `window.IllumifyStorefront`;
2. a `<base href>` pointing at **the theme's asset root**.

So a page's first inline script already sees the global, and there is nothing to wait for and no
marker comment to leave in place.

### `window.IllumifyStorefront` — the only supported browser API

Frozen, and versioned by `version`. Its full declaration ships in your project at
`src/storefront.d.ts`; read that file, it is the contract.

```ts
window.IllumifyStorefront = {
  version: 2,
  config: {
    accessMode,        // "Anonymous" | "CustomerLink" | "SignedInAccount" | "Preview"
    documentBaseUrl,   // absolute — where this document is mounted
    apiBaseUrl,        // absolute — the API for THIS authority
    assetBaseUrl,      // absolute — this theme's own files. NOT product media
    shopperContext,    // "Anonymous" | "Identified" — NOT a pricing signal
    themeKey,          // absent under `illumify dev`
  },
  navigate,            // (route: string) => void — a full document load
  routeUrl,            // (route: string) => URL  — the same URL, without going there
  login,               // (() => void) | null
  logout,              // (() => void | Promise<void>) | null
  getCart,             // (options?) => Promise<Cart>            — ABSENT under Preview
  replaceCart,         // (body, options?) => Promise<Cart>      — ABSENT under Preview
  prepareCheckout,     // (body, options?) => Promise<Prepared>  — ABSENT under Preview
  checkout,            // (body, options?) => Promise<Result>    — ABSENT under Preview
};
```

Four properties are worth stating rather than discovering:

- **It is frozen**, so you cannot patch or wrap any of them.
- **`version` is `2`.** Version 1 was the six above the line; 2 added the four cart and checkout
  methods and nothing else. Branch on the number rather than feature-detecting a method, because a
  version-2 runtime under `Preview` genuinely has none of the four.
- **`themeKey` is absent under `illumify dev`**, because nothing has been uploaded when previewing
  and a made-up 64-character key would name nothing while reading as an answer. Do not depend on it.
  When it *is* present it is the key of the **revision** being served, which the server minted — not
  the key `illumify build` printed, and not stable across uploads of the same theme.
- **The four cart methods are absent — not `null` — under `accessMode: "Preview"`.** A hosted preview
  cannot write a cart or place an order, by construction. Branch on `accessMode` and render a
  read-only cart; do not call and catch.

**There is a second, underscored global on the page — `window.__ILLUMIFY_STOREFRONT__` — and a theme
MUST NOT read it.** It is how the document renderer hands its configuration to its own bootstrap and
how it detects its own prior injection: internal, unversioned, and free to change without notice.
`IllumifyStorefront` is built from it and is the contract. It is deliberately not declared in
`src/storefront.d.ts`, because a typed global is indistinguishable from a supported one.

### The same theme is served under four different bases

**Build every data call from the injected `apiBaseUrl`. A hand-built path silently serves one
audience the wrong data.**

Which authority a document was served under decides what `apiBaseUrl` and `documentBaseUrl` contain,
and your page cannot tell which it got except by reading `accessMode`:

| `accessMode` | The document is at | Its API is at |
| --- | --- | --- |
| `Anonymous` | `/storefront/{slug}` | `/storefront/{slug}/api` |
| `CustomerLink` | `/storefront/{slug}/c/{token}` | `/storefront/{slug}/c/{token}/api` |
| `SignedInAccount` | `/storefront/{slug}/_account` | `/storefront/{slug}/_account/api` |
| `Preview` | `/storefront/{slug}/preview/{capability}` | `/storefront/{slug}/preview/{capability}/api` |

(Those paths sit under a per-environment service prefix; the whole absolute URL arrives in the
injected config, which is why you never assemble one.)

**A customer link's token in the URL is the whole of its authority.** There is no identity cookie and
no shopper session anywhere in this contract — the token is resolved on every document and every
catalog request. (The one cookie the platform does set is the **anonymous cart's** lookup token,
described with the cart below. It is scoped to this storefront, `HttpOnly`, and finds a cart; it
carries no identity and never changes which shopper the server resolves.) So a URL you recomposed from a slug drops the token, and the server silently answers a
public shopper's catalogue where it meant an identified one. Same prices-look-fine, wrong-audience
failure for `_account` and `preview`.

All four endpoints exist under all four authorities with identical shapes, so a theme is written once
and never branches on `accessMode` for data.

### `<base href>` is the ASSET base, not the document base

This is the one consequence that catches people who have shipped to a static host before, and it is
the opposite of what they expect.

`<base href>` points at the theme's asset root. So `assets/app.js` written inside `pages/about.html`
resolves to `{assetBase}/assets/app.js` — **not** `{assetBase}/pages/assets/app.js`. A bundler
emitting *document*-relative URLs for a nested entry writes `../assets/app.js`, which resolves above
the theme root and 404s. Keep `base: "./"` in the bundler config at the output root; `illumify build`
resolves every link this way and explains any that climb out.

Two things follow:

- **Derive a router base from `IllumifyStorefront.config.documentBaseUrl`, not from
  `document.baseURI`.** Here those are two different URLs, and `document.baseURI` is the asset root.
- **A relative `href` is not a route link.** It resolves against the asset root, not against your
  page tree. For a link to one of your own routes use `data-illumify-storefront-route` in static
  markup, or `IllumifyStorefront.routeUrl(route)` for anything you render — both below. The same trap
  catches `history.pushState` with a relative URL.

There is also a supported literal, `__ILLUMIFY_STOREFRONT_ASSET_BASE_URL__`, which the server
replaces with the asset base **in HTML documents only**. Do not hand it to a bundler as its `base`: it
would also be written into JS chunk preloads and CSS `url()`, where nothing replaces it, and the
result is a broken theme that looks correct in the one file anybody inspects.

## The three DOM hooks: add markup, write no JavaScript

```html
<a      data-illumify-storefront-route="/products">Products</a>
<button data-illumify-storefront-login>Sign in</button>
<button data-illumify-storefront-logout hidden>Sign out</button>
```

- **`data-illumify-storefront-route`** is rewritten to a real `href` at `DOMContentLoaded`, so
  ordinary click, copy-link and open-in-new-tab all keep whichever authority the shopper arrived
  with. **For markup that is in the document at first paint.** Anything rendered later needs the
  section below — this is the one hook that is not render-safe.
- **`data-illumify-storefront-login` / `-logout`** are wired by Illumify. No click handler, no
  redirect, no token, no endpoint. The platform owns the identity round trip and the return to the
  URL the shopper was on.

**The login and logout clicks are handled by delegation on the document**, not by a listener bound to
your elements, so a button rendered later by React, Vue or a template swap still works and there is
nothing to re-run after a render. **That delegation is registered for those two attributes only**, and
the difference between them and the route attribute has its own section below.

`login` and `logout` are **`null`, not absent**, when the platform has no URL for them:

- `login` is `null` on a preview document, on an already-signed-in (`_account`) document, and on a
  site that has sign-in disabled;
- `logout` is `null` unless the document is a customer-link or an account one;
- **both are `null` under `illumify dev`**, because the auth bridge is a deployed route.

So guard before calling (`IllumifyStorefront.login?.()`), and expect the attribute's click to be
handled and do nothing in those cases. **That is not a reason to build a sign-in of your own.**

The two calls are not symmetrical, which matters if you call them rather than using the attributes:
`login` navigates and returns nothing, while `logout` on a customer-link document is `async` — it
POSTs, throws a plain `Error` on a non-2xx, then navigates. The delegated click handler discards that
promise, so a failed logout from a button is an unhandled rejection and nothing on screen.

### Which of the two buttons shows is your page's only decision

`accessMode` tells you how the shopper arrived; `IllumifyStorefront.config.shopperContext` tells you
whether an identity resolved. Either will do for the toggle. What you must not do is build a page that
only works under one site policy — the policy is the seller's and they can change it without touching
your project. `getSession()` returns the site's `accessPolicy` if you want to hide a sign-in button
the site does not offer; `illumify skills get data` has it.

### `shopperContext: "Identified"` does not mean discounted

**Signing in is not a discount.** `"Identified"` means the server resolved an identity, not that it
matched a seller customer: an identified shopper whose facility licence matched nothing gets public
prices, and this page is never told which happened. **Never write copy promising a discount for
signing in** — it will be wrong for every shopper whose licence matched nothing, and that is a normal
outcome rather than a failure. The separate field that answers the pricing question is
`pricingContext` on a `getSession()` or catalog response; `illumify skills get data` owns it.

## Route anchors are rewritten once, and that is not what the login button does

**This is the most expensive thing in this file, because it produces a page that looks finished.** An
acceptance build shipped eight product cards whose every link was dead, and a screenshot would not have
caught it.

The two hooks sit side by side in the same markup and read as if they work the same way. They do not:

| | How it is wired | Survives a re-render? |
| --- | --- | --- |
| `data-illumify-storefront-login` / `-logout` | A **delegated click listener** on the document. Nothing is written into your element | **Yes.** The element can appear at any time |
| `data-illumify-storefront-route` | A **one-time sweep** that writes an `href` **into** each element | **No.** Only elements present when the sweep ran have one |

The sweep runs **once**: at `DOMContentLoaded` if the document is still loading, otherwise immediately.
There is no `MutationObserver`, no second pass, and no way to ask for one. So an anchor a framework
draws when its fetch resolves — a grid, a filtered list, a modal, a paginated page two — carries the
attribute and **no `href`**. It is not a link. Clicking it does nothing, and:

- **no console error**, no warning, and nothing in the network tab;
- the element still looks like a link if your CSS styles it as one;
- the attribute is right there in the DOM inspector, so it reads as wired.

**The sweep fails the same silent way on a bad value.** If the attribute names a route it cannot
resolve — one that escapes this storefront — the sweep *removes* the element's `href` rather than
reporting anything. So a dead anchor means either "rendered too late" or "that route is not mine", and
the page looks identical in both cases. `routeUrl` throws for the second, which is how you tell.

### What to do, by which situation

**In markup that is in the document at first paint — keep using the attribute.** It is the right shape
there, it needs no JavaScript, and nothing below improves on it.

```html
<a data-illumify-storefront-route="/products">Products</a>
```

**For anything you render yourself — build the `href` from `routeUrl` and keep the anchor an anchor.**

```jsx
const url = window.IllumifyStorefront.routeUrl(`/product?sku=${item.skuId}`);

<a
  href={url.toString()}
  onClick={e => { e.preventDefault(); history.pushState(null, "", url); rerender(); }}
>
  {item.name}
</a>
```

**The `href` alone is a complete answer.** Drop the `onClick` and a click is an ordinary navigation:
the server serves whichever `pages` entry that route declares, and your app boots there. Add the
`onClick` only if you have a client-side router to hand the transition to.

The `href` is not optional decoration, though, and an `onClick` is no substitute for it: it is what
gives you middle-click and ⌘-click to a new tab, "copy link address", the status-bar preview on hover,
and an anchor a screen reader announces as a link. A `<div onClick>` has none of them.

**`routeUrl` returns a `URL`, not a string** — hence `.toString()`. JSX will accept the object and
stringify it, so a theme that forgets is not visibly broken; write the `.toString()` anyway, because
the next reader of that line will otherwise expect a string from the type.

It is the same function the sweep uses and the same one `navigate` uses, so the `href` you get is the
one the attribute would have produced: the shopper's authority preserved, a route outside this
storefront thrown on rather than linked to. **Do not check whether `routeUrl` exists before calling
it, and do not write a fallback** — a theme runs on the deployed runtime, and `illumify dev` injects
that same runtime's bootstrap.

If you do write the `onClick`, two things about the transition are worth knowing:

- **`IllumifyStorefront.navigate(route)` is a full document load** — `location.assign` under the hood.
  It is the programmatic equivalent of following the link, so reach for it from a button, after a form
  submit, or anywhere you have no anchor. It is not the no-reload path, and using it inside `onClick`
  makes the `preventDefault` pointless.
- **Pass a `URL` to `pushState`, not a bare path.** A relative URL there resolves against
  `document.baseURI`, which on this platform is the *asset* root — so `pushState(null, "", "product")`
  quietly moves the address bar into your asset tree. `routeUrl(route)` is absolute and correct:
  `history.pushState(null, "", routeUrl(route))`.

**Do not reach for a `MutationObserver` to re-run the rewrite.** It was considered and rejected: the
attribute is the right shape for static markup, and watching the whole document on every DOM change to
support a usage it was never meant for costs every render. `routeUrl` is the sanctioned answer.

## The cart and the checkout, and the adapter is the only way in

A cart and a real checkout exist. A shopper fills a cart, presses buy, and genuine Sales Orders land in
the seller's ERP for approval — so **the theme owns the entire cart experience**: the lines, the
quantity controls, the facility columns or selector, the guest form, the confirmation, the retry states
and the accessibility. Illumify injects no cart DOM, no shared component and no required layout. There
is nothing to match and nothing to extend.

Three routes sit under the same `apiBaseUrl` as the catalog reads:

```text
{apiBaseUrl}/cart              GET, PUT
{apiBaseUrl}/checkout/prepare  POST
{apiBaseUrl}/checkout          POST
```

**Do not call them with your own `fetch`.** A write is accepted only when it carries an
`X-Illumify-Storefront-Request` header at the value the server compares *and* an `Origin` matching the
storefront's own configured origin. The browser supplies the second; the first is the adapter's, and a
`fetch` you write does not send it — so what you get is `403 InvalidStorefrontWrite`, which reads as
"this storefront does not allow carts" and is really "you went round the adapter".

Nor is copying the header the fix. Its name and its value are the platform's, unversioned, and free to
move; a theme that hard-codes them works until the day it does not, and then fails as a `403` on a
checkout in front of a real buyer. The adapter exists so that value is never yours to track.

So the four methods on `window.IllumifyStorefront` are the whole surface, and they preserve whichever
authority the document was served under — anonymous, customer link, or signed-in account. There is no
authority to pass and none to get wrong. `src/storefront.d.ts` in your project carries every field of
every request and response; read it, it is the contract.

### The cart is replaced, never edited

`PUT` is a **full replace**. There is no add, no increment, no remove and no patch: you send the cart
you want to exist.

- **Read, apply, write.** Removing a line means sending the array without it. An empty
  `selectionList` empties the cart.
- **A `GET` creates nothing.** A shopper who has added nothing has no cart, and reading does not make
  one.
- **`cartVersion` guards the write.** It is an opaque rowversion, `null` on the first `PUT`, and a
  mismatch is `409 CartVersionConflict` — with the current cart attached, so the repair is to re-apply
  the shopper's intent to *that* and write again, never to retry the same body.
- **A line is `{ offerId, quantity, observedOfferRevision }`**, plus `guestAllocationKey` on an
  anonymous line. That is all a theme may send. No SKU, no batch, no UOM, no conversion, no money —
  the server re-derives every one of them from `offerId` and re-prices at cart time, again at prepare,
  and a third time inside the checkout transaction. A price your page sends is not rejected; it is not
  read.
- **At most 100 unique offers**, quantities greater than zero with at most four decimal places.

**The anonymous cart is found by a cookie, and that is the one cookie in this contract.** Storefront-
scoped, `HttpOnly`, and only the hash of its token is stored. It belongs to the browser profile rather
than the tab, so **two tabs deliberately share one anonymous cart** — which is why `409
CartVersionConflict` is a normal thing to handle and not an edge case. A different profile, a private
window, a cleared cookie, a different site or a different authority is a different cart.

### `409 PricingChanged` is an instruction, not a failure

Send the `offerRevision` your page actually displayed as `observedOfferRevision`. If it has moved, the
write is refused with `409 PricingChanged`, **nothing is mutated**, and the refreshed cart comes back
on the error.

This is the same shape of instruction as `409 CatalogRevisionChanged` on the catalog, and a theme that
treats either as an error is broken in the same way: show the shopper the new number and let them
decide. Do not retry with the stale revision, and never send a remembered or hand-made revision — a
stale one that happens to match is the one case the server cannot catch.

### Checkout is two calls, and one checkout is several orders

`prepareCheckout` writes nothing and places no order. It returns a protected token good for **15
minutes** and the destinations this cart is allowed to ship to; `checkout` resubmits everything and
refuses if any of it has moved.

**`checkout` returns `orderList`, never a single order.** The cart is grouped by (seller source,
destination facility), because that is what has to be picked and driven somewhere — so a page that
reads `orderList[0]` shows one shipment of several. The fan-out is all-or-nothing: if one group fails,
the whole transaction rolls back.

`idempotencyKey` is yours to generate, and it is what makes a retry safe. Generate it **once per
checkout the shopper started**, not per network request: the same key with the same body returns the
same orders, and a fresh key on a retry places a second order.

**A `503` can mean the orders were placed.** When the orders commit but the post-commit approval
workflow does not activate, the response is `503` with `finalizationStatus: "Pending"` — and because
the status is not 2xx, the call *rejects*. The orders are real. Catch it, look for `orderList` on the
error's `response`, and if it is there show the confirmation and retry by the same `idempotencyKey`;
do not send the shopper back to the cart to buy again.

After a successful checkout, `getCart()` returns that cart's **frozen snapshot** — not re-priced and not
an error — so a shopper who refreshes their confirmation page still sees what they ordered. Writing to a
checked-out cart is `409 CartCheckedOut`, so a shopper who wants to buy again starts a fresh cart with a
`null` `cartVersion`. Reading a cart that never existed or has expired is an empty cart, not an error.

### Two kinds of buyer, and the difference decides what you build

- **A known customer** — arriving by customer link or signed-in account — sees their negotiated
  pricing and can send **one order to several of their own facilities**. Each cart line carries the
  facility its offer was priced under, so a multi-facility cart is one cart with differently allocated
  lines, not several carts. Their lines must **not** carry a `guestAllocationKey`.
- **A guest** — a licensed business nobody has set up yet — enters their contact details, licence
  number and address at checkout, and **the customer record is created on the spot**. Guests are
  always priced at public list pricing, even when their licence turns out to match an existing
  customer, so never render a guest checkout as a way to reach a discount. Their lines **must** carry
  a `guestAllocationKey`: it is your own label, one per destination, and `(offerId,
  guestAllocationKey)` is the line identity, so the same offer allocated to two destinations is two
  lines. With one guest destination the allocation map may be omitted; with more than one, every line
  must be allocated and every destination used, because the server will not infer one.

### The guest destination's State is an id you were given, not a string you compose

A guest destination is one entry in `guestFacilityList`, and every field on it is text you collected
— company name, contact, licence number, address — **except one**. `stateId` is a **number**, and it
is the id of a row in the ERP's own state table. There is no spelling of `"CA"` or `"California"`
that the server will accept.

The list comes back on the session you already read:

```ts
const { stateList } = await getSession();
// stateList: Array<{ id: number; name: string; shortName: string }>
```

`id` is what you submit, `name` is what a shopper reads, `shortName` is the two-letter postal
abbreviation. The list is **US states only, ordered by name**, so render it in the order it arrives
and it is already alphabetical.

**Do not go looking for a state endpoint.** `CompanyAccess/GetStateList` exists in the ERP, and it
is not the CSP that stops you calling it — the ERP is the same origin as your document, so
`connect-src 'self'` permits the request. What stops you is authority: it is an authenticated
management operation, outside the route families the storefront's anonymous allowance covers, and a
shopper carries no credential for it. The retired preauth Storefront endpoint is gone. The session is
the only source, and it is a call your page already makes.

**Never hardcode an id.** The numbers are database rows, not a standard, and they are not guaranteed
to match between environments. A theme that reads `stateList` and posts the selected entry's `id`
back is correct everywhere; one that ships `stateId: 12` because that is what a dropdown showed
during development is wrong in every environment but the one it was built against, and nothing will
say so — the order is simply filed against the wrong state.

**Submit the same id in both phases.** `prepareCheckout` and `checkout` each carry the full
`guestFacilityList`, so the chosen `stateId` goes in both. `checkout` resubmits everything and
refuses if any of it has moved, so changing the State between the two calls invalidates the prepared
token rather than quietly updating the destination: collect the address once, then send the same
values twice.

**An empty `stateList` is the contract, not a gap.** A signed-in or customer-link shopper gets `[]`,
because they check out against facilities the seller already allowed them and cannot submit a guest
destination at all. Read its length the way you read `facilityList`'s: empty means *do not render
this field*, not *the data failed to load*. Anonymous preview sessions do return the list, so a
hosted preview exercises the real form.

**One caveat, as of 2026-08-31.** The field arrives with `Main` `77a2ae0312`, which is now on the
QA branch — but a branch carrying it and an environment running it are two things, so until the
deployment you are pointed at has picked it up, **treat an absent `stateList` exactly as you treat an
empty one**. That is the same check you already need for signed-in shoppers, so it costs nothing.

**`@illumify/sdk` has not published the type yet**, so `session.stateList` does not type-check
against the SDK's `CatalogSession`. Read it through a narrowed local type until the SDK catches up,
rather than widening it with a cast you will forget to remove. `illumify dev --fixtures` serves the
list now, on the anonymous branch only.

### A shopper with more than one facility has no facility until your page picks one

**A theme with no facility selector shows a multi-facility shopper a catalogue with no prices and no
way to add anything, and nothing on the page says why.**

One eligible facility is selected for the shopper. **Two or more and the server selects none** — an
arbitrarily chosen facility would mean arbitrary prices. Until the browser supplies a validated
`customerFacilityKey`, every offer's money and availability come back suppressed and every cart write
is refused with `400 FacilitySelectionRequired`. `illumify skills get data` has the exact fields and
the reason they are indistinguishable from an empty catalogue.

So: read `getSession().facilityList` before you draw anything priced. More than one entry means a
selector is not a feature you might add later — it is the thing that makes the page work. You may hold
several validated keys at once and render them as columns; each column is its own priced read, and
each cart line keeps the facility its offer was priced under.

## The Content Security Policy, measured

Every theme document is served with this policy, and it is stricter than most people assume:

```text
default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';
img-src 'self' data: blob:; font-src 'self' data:; media-src 'self' data: blob:;
connect-src 'self'; manifest-src 'self'; worker-src 'none'; frame-src 'none';
frame-ancestors 'none'; object-src 'none'; form-action 'self'; base-uri 'self'
```

Read as instructions:

| You want to | Allowed? |
| --- | --- |
| `fetch` / `EventSource` / WebSocket to a third-party API | **No** — `connect-src 'self'` |
| Load a script or stylesheet from a CDN | **No** — `'self'` only. It does not load at all |
| Inline `<script>` and `<style>` in your own document | Yes |
| An external image, font or video by `https` URL | **No**. `data:` and `blob:` are allowed |
| Embed an `<iframe>` — a video, a map, a widget | **No** — `frame-src 'none'` |
| A web worker or service worker | **No** — `worker-src 'none'` |
| `<object>` / `<embed>` | **No** — `object-src 'none'` |
| Submit a form to another origin | **No** — `form-action 'self'` |
| Write your own `<base>` tag | **No** — `base-uri 'self'`, and the server already injected one |

So: **ship every asset inside the theme, and plan for a page whose only data source is the catalog
API.** A dependency that phones home, loads a font from a CDN, or embeds a third-party widget cannot
work here — that is a design constraint to check before you plan a feature, not a bug to debug after.

Product images are the exception that is not an exception: they arrive from the catalog as complete
**same-origin** URLs, so `img-src 'self'` covers them. See `illumify skills get data`.

**`illumify dev` sends no CSP.** That is the widest dev/production divergence in the toolchain: a
third-party fetch or a CDN `<script>` works locally and fails deployed, with the browser console the
only place the reason appears. Do not conclude from a working preview that an external dependency is
allowed.

## Two different file-type rules, and they bite differently

**1. The archive extension allowlist — one bad file refuses the whole upload.**

Accepted: `html htm css js mjs json map svg png jpg jpeg webp gif ico woff woff2 ttf otf eot txt xml
pdf`.

Anything else fails the entire theme ZIP — one stray `.wasm`, `.avif`, `.webmanifest`, `LICENSE`,
`_headers` or `.mp4` and nothing uploads. A file with **no extension at all** is refused too.
`illumify build` says so before the archive goes up the wire.

**2. The served content-type table — has gaps, and they are fatal for the file.**

The server types an asset by extension from a fixed list. `.map`, `.txt` and `.xml` are **allowed into
the archive but have no content type**, so they come back as `application/octet-stream`. A browser
will not execute a module script or apply a stylesheet delivered that way. `illumify build` warns and
names every such file, and `illumify dev` reproduces the gaps on purpose so a file that will not work
deployed does not work locally either.

HTML is not in the asset table and does not need to be: pages are served by the document route as
`text/html; charset=utf-8`.

Other limits worth knowing: **2 MB per HTML document**, 50 MB per file, 2,000 files, 100 MB
compressed / 500 MB expanded per theme. No symlinks, no ZIP64, no encrypted entries, no
case-colliding paths.

## Project layout

```
AGENTS.md / CLAUDE.md   how the platform behaves; reference, not a brief
README.md               for the human
illumify.config.ts      your pages
illumify.link.json      storefront site plus optional remembered theme name, per environment.
                        Written by `link`/`upload`; only `displayName` is deliberately editable
vite.config.ts          one platform-owned line: base: "./"
index.html              your HTML entry
src/storefront.d.ts     the injected global's types. Read it; do not restate it
.env                    local settings, gitignored. Generated — see `illumify skills get deploy`
skills/illumify/        a discovery stub; the real content comes from `illumify skills get`
src/                    entirely yours. A plain scaffold has only an entry point that renders a
                        placeholder — nothing here decides what your pages contain
.illumify/              build output: the ZIP and a readable copy of the manifest. Not uploaded
vendor/                 the packed @illumify/sdk only when --sdk points at a local checkout
```

**`@illumify/sdk`'s own `README.md` is in `node_modules/@illumify/sdk/README.md`** and is the
reference for the data API. These skills do not restate it.

## Showing the catalogue: cards, a table, or the spreadsheet

Three answers, and which one is right depends on what the owner asked for — not on what looks
impressive.

**Cards, by default.** A catalogue of products with an image, a name and a price is a card grid.
Reach for this unless you were told otherwise. It needs no library.

**A plain HTML `<table>` when they asked for a table.** Order forms, price lists, anything read
row by row. Write the table yourself: `<table>` with real `<thead>`/`<tbody>`, and CSS. A dependency
here buys nothing a browser does not already do, and it costs every shopper the download.

**`@illumify/react-data-grid` only when they said "spreadsheet".** Frozen columns, cell editing,
fill handles, virtualised scrolling over thousands of rows — a grid someone works *in*, not a table
they read. That is the one case where the weight is worth paying.

```bash
npm i @illumify/react-data-grid @mui/material @mui/icons-material @emotion/react @emotion/styled
```

**The peers are not optional and not installed for you.** The package declares `react`,
`@mui/material`, `@mui/icons-material`, `@emotion/react` and `@emotion/styled` as peer dependencies
so a project that already has MUI does not end up with two copies — two Emotion instances mean two
style contexts, and the symptom is styles silently not applying. A scaffolded Illumify project has
none of them, so install all five or the grid will not render.

**Know what you are spending.** The package unpacks to about 860 KB before its dependencies, and it
brings `@tanstack/react-virtual`, `zustand`, `dayjs` and two `@atlaskit/pragmatic-drag-and-drop`
packages, on top of the MUI and Emotion the peers require. On a storefront that is a real cost to a
shopper on a phone, and it is the reason this is the third answer rather than the first.

If you are unsure which the owner meant, ask. "Table" and "spreadsheet" are different requests, and
guessing the heavy one is the expensive mistake.

## What you must not do

- **Do not declare only `/` and expect deep routes to reach it.** Matching is exact, and there is no
  fallthrough to a parent route or to a file.
- **Do not build a URL by hand — not an API path, not a route, not an asset path.** They arrive at
  runtime on `IllumifyStorefront.config`, and one theme serves whichever site it is assigned to under
  whichever authority the shopper arrived with. `routeUrl(route)` is how you get a route's URL; it is
  not "by hand", it is the platform computing it.
- **Do not put `data-illumify-storefront-route` on an anchor your code renders after first paint and
  expect a link.** The rewrite runs once. Set `href` from `routeUrl(route)` instead — this one fails
  with no error at all, and the page looks finished.
- **Do not read `window.__ILLUMIFY_STOREFRONT__`.** It is private and unversioned.
- **Do not `fetch` `cart`, `checkout/prepare` or `checkout` yourself, and do not copy the adapter's
  header to make it work.** Omitting it is `403 InvalidStorefrontWrite`, which reads as a refusal by the
  storefront rather than by the route you skipped; hard-coding it pins your theme to an unversioned
  internal value. Use `replaceCart`, `prepareCheckout` and `checkout`.
- **Do not send money, a SKU, a batch, a UOM or a conversion to the cart.** A line is `offerId`,
  `quantity` and the `offerRevision` you displayed. Everything else is re-derived and re-priced.
- **Do not treat `409 PricingChanged` or `409 CartVersionConflict` as errors.** Both carry the current
  cart and both are instructions. Do not retry either with the body that was refused.
- **Do not draw a priced page for a multi-facility shopper without a facility selector.** They get no
  prices and no Add, with nothing saying why.
- **Do not read `orderList[0]` as "the order".** One checkout produces one order per (source,
  destination) group.
- **Do not re-place an order after a `503`.** If the error's `response` carries `orderList`, the orders
  are committed; retry by the same `idempotencyKey`.
- **Do not remove `base: "./"`** from the bundler config, and do not write an absolute asset base:
  the real one contains a hash of the theme's own content and cannot be known while that content is
  being built. In particular, do not restore `base: process.env.ILLUMIFY_ASSET_BASE ?? "/"`: that
  variable is never set. A root-relative build can look fine locally while its scripts resolve to the
  origin root and the deployed document is blank.
- **Do not put a route in `pages` whose `entry` your build does not emit.** `illumify build` fails,
  and the server would refuse the upload anyway.
- **Do not design around a third-party API, a CDN script, an external image or an `<iframe>`.** The
  CSP forbids all four. Read `illumify skills get data` before planning a feature, not after.
- **Do not promise a discount for signing in.** Identified is not the same question as priced.
- **Do not put a credential in this project as an agent.** `.env` may contain the owner's
  `ILLUMIFY_API_KEY` for the CLI's supported lookup, and its clearly marked `ILLUMIFY_DEV_API_KEY`
  slot is for the dev proxy; `.env` is gitignored. Neither key belongs in a theme bundle. If you are an
  agent: never ask for either key, read either value, or accept one pasted to you — see
  `illumify skills get deploy`.
- **Do not report an upload as a deployment.** Uploading creates a theme; a shopper reaches whatever
  the site's assignment names, and only an administrator can change that.
