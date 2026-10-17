# Using svebcomponents/atproto in Cloudless

Use the published ATProto packages for post rendering, account interactions, and shared data handling. Keep the extension-specific integration small and explicit.

This plan is grounded in [source revision ca0ffb9](https://github.com/svebcomponents/atproto/tree/ca0ffb9efb6f842828486337b74bdaae1ca1fa59). Names described as proposed are design work, not APIs that already ship. No upstream changes have been made as part of this document.

## Existing primitives and how to use them

| Public surface | Cloudless use | Constraints |
| --- | --- | --- |
| `<atproto-comments>` | Render the post, reactions, discussion, and composer inside a packaged extension iframe | Consume the published custom element; do not import its private Svelte source or duplicate its markup |
| `thread`, `show-root` | Identify the real thread and display its root | Showing the root still includes a comment-section layout; compact / collapsed presentation is proposed work |
| `threadData`, `fetchedAt`, `stale-time` | Hand the component a real prefetched snapshot and its freshness boundary | `threadData` is a JavaScript property, not a JSON HTML attribute; never include credentials |
| `fetchCommentTree`, `getPostThread` | Prefetch actual thread snapshots with bounded depth; load discussion when expanded | The helpers already accept an injected fetch and abort signal |
| `normalizeThread`, `CommentTree`, `PostView` | Share the package's normalization and data types across the worker and UI | A feed item is not a complete thread; preserve the distinction between unloaded and absent replies |
| `parseThreadRef`, `resolveThreadUri` | Resolve supported viewer URLs and AT URIs consistently | Prefer canonical post URIs as cache and deduplication keys |
| `segmentRichText`, `viewerPostUrl`, `viewerProfileUrl`, `viewerTagUrl` | Keep facets and outbound links consistent wherever the host needs them | The component already uses these helpers for its own rendering |
| `ServiceClient` and its exported request/result types | Retain one bridge API contract for sessions, replies, reactions, and viewer state | Its fetch hook is useful, but storage, sign-in, and session restoration also need abstraction |
| `--atproto-comments-*` and `::part(...)` | Match the extension's theme through the supported styling surface | Layout behavior should use a public component option rather than hidden DOM manipulation |
| Component events and `revalidate()` | React to loading, revalidation, posting, errors, and session changes | Current custom-event details are typed `unknown`; validate the documented payloads before consuming them |
| `service`, `appview`, `viewer`, `viewer-name` | Preserve replaceable bridge, read endpoint, and outbound viewer configuration | These are different responsibilities; routing every public read through the auth service is unnecessary |
| `live` | Reuse live discussion refresh when a user opens a thread | Collapsed feed cards should not each open a live stream |

Sources: [component API](https://github.com/svebcomponents/atproto/blob/ca0ffb9efb6f842828486337b74bdaae1ca1fa59/components/atproto-comments/README.md), [client exports](https://github.com/svebcomponents/atproto/blob/ca0ffb9efb6f842828486337b74bdaae1ca1fa59/packages/atproto-client/src/index.ts), and [fetch helpers](https://github.com/svebcomponents/atproto/blob/ca0ffb9efb6f842828486337b74bdaae1ca1fa59/packages/atproto-client/src/fetchThread.ts).

## What to add upstream

### A replaceable session client

Introduce a structural interface for the session and action methods the component needs, implemented by the existing website client and an extension adapter. Keep normal website use working with its existing defaults.

The component currently creates `new ServiceClient(service)` internally. That client reads page `localStorage`, opens a browser popup, and uses browser events during sign-in. Passing a different fetch function alone cannot replace those behaviors. Also, session restoration currently depends on either a stored page token or a same-origin service; an extension-owned session needs an explicit restoration path.

The proposed seam should cover session restoration, sign-in, sign-out, session changes, replies, reactions, and viewer reaction state. It should not require the extension adapter to subclass a class with private token state. Keep message serialization, Chrome APIs, and credential storage in Cloudless; keep the bridge's request and response handling reusable in the client package.

Account changes must reach every mounted card. The component should keep ownership of its optimistic interaction state and clear or reconcile it when the session changes. The host supplies public session information, never raw tokens.

Public AppView reads and live discussion events are separate concerns. Preserve those existing paths; a session-client hook should not turn into an unrestricted fetch proxy.

### A reusable post presentation

Start with the real component and its `show-root` option for the first integration spike. Add a supported compact post presentation with collapsed replies upstream when the feed requires it. Preserve the existing action handling, composer, accessibility, and thread rendering across presentations.

Use the current CSS custom properties and parts for appearance. Do not hide arbitrary internal sections with selectors, patch the shadow DOM, or recreate a second post renderer in Cloudless. If a separate post custom element becomes the cleaner public API, extract shared rendering and behavior upstream first.

There are two concrete data gaps to address:

- **Root moderation.** `RootSummary` currently omits labels, and the component deliberately renders its root with an empty label array because its original host is a blog showing its own post. Arbitrary replacement posts need their labels preserved and the chosen moderation policy applied to the root as well as replies.
- **Embeds.** The current post wire types and normalized model do not retain media embeds. Extend the shared model and renderer when adding images, quoted posts, or other embeds. Until then, keep the supported post scope explicit and offer an appropriate viewer link for unsupported content.

Sources: [normalization](https://github.com/svebcomponents/atproto/blob/ca0ffb9efb6f842828486337b74bdaae1ca1fa59/packages/atproto-client/src/commentTree.ts) and [root presentation](https://github.com/svebcomponents/atproto/blob/ca0ffb9efb6f842828486337b74bdaae1ca1fa59/components/atproto-comments/src/index.svelte#L215).

### Feed access through the existing bridge

Extend the hosted bridge with bounded, cursor-based feed access and the necessary OAuth read / proxy permissions. Retain the existing server-side OAuth client, session stores, grant revocation, rate limits, and reply / reaction handlers.

Support two source kinds: Following, served by `app.bsky.feed.getTimeline`, and a feed generator identified by an AT URI, served by `app.bsky.feed.getFeed`. Discover is a feed-generator selection with URI `at://did:plc:z72i7hdynmk6r22z27h6tvur/app.bsky.feed.generator/whats-hot`. Other custom feeds use the same API with their own URI.

The bridge performs personalized feed reads using the signed-in account, forwards explicitly selected language preferences, and returns the cursor and any opaque feed context alongside the posts. Resolve feed metadata through the AppView; a selected feed URI must not become an arbitrary network URL to proxy. Keep OAuth grants limited to the selected capabilities.

Bluesky's client performs additional filtering, caching, and recommendation feedback. Fetching the personalized generator is supported, but it does not guarantee an identical stream to the first-party app. Preserve opaque context for future accurate feedback. If feedback is implemented, report actual visible impressions and explicit interactions; prefetched posts are not viewed posts. [Feed API guide](https://bsky.network/docs/bluesky-api/viewing-feeds/#feed-generators), [current feed client](https://github.com/bluesky-social/social-app/blob/main/src/lib/api/feed/custom.ts), and [feed selection](https://github.com/bluesky-social/social-app/blob/main/src/lib/constants.ts).

Add explicit extension-client sign-in and session binding. Do not reuse `x.com` as the account's trusted embedding origin or accept absent / opaque origins as authorization. Keep existing website consumers working.

Feed data will need shared normalization beyond today's thread-only helpers. Put broadly useful post and feed types and normalization in `atproto.client`. Preserve labels, embeds, the generator's returned order, cursor / feed context, and relevant viewer state; keep Cloudless's replacement queue and X-slot assignments outside the package. Never construct a fake complete thread by discarding a feed post's replies or moderation data.

Preserve the user's existing Bluesky moderation and visibility decisions. A later public thread read must not override authenticated eligibility or viewer state, or reintroduce a post excluded from the replacement queue. Keep those decisions separate from Cloudless's user-selected X filters.

## Data flow and lifecycle

1. The worker fetches a page from the selected Discover, Following, or custom feed through the bridge and assigns supported, eligible posts to replacement slots. Queue and cursor state is isolated per account, source URI / kind, and language settings and reset when those change.
2. Prefetch the real thread snapshot only for cards that are about to be shown. Use the exported fetch helpers and conservative reply depth, then pass `threadData` and its fetch timestamp into the component.
3. The component uses its own rendering, reaction state, and optimistic reply behavior. Its injected session client delegates authorized operations to the worker, which calls the hosted bridge.
4. Opening replies loads the actual discussion. Enable the component's live refresh only for an active discussion, and release it when collapsed, hidden, or unmounted.
5. On slot removal or recycling, unmount the frame and release its requests and subscriptions. Cache snapshots and deduplicate in-flight reads by thread and endpoint configuration.

Keep the iframe population bounded to the visible area and a small prefetch margin. Each iframe has a document and runtime cost. The package is a standalone custom-element bundle that includes its own Svelte runtime; verify the actual emitted package size and memory use instead of assuming a native Svelte import or shared runtime across frames.

The composer uses a dialog inside the frame. It cannot escape the frame's bounds, so the host needs a validated resize / expansion protocol that keeps the composer usable and preserves scroll position. Pin an active composer instead of recycling its frame. If the host needs composer lifecycle notifications to do that, add a public event upstream rather than inspecting private shadow DOM.

The existing SSE bridge already multiplexes its upstream event source. Reuse that behavior. A foreground thread frame can own its `EventSource`; do not depend on an indefinitely alive Manifest V3 worker to host a continuous stream.

## SvelteKit and SSR

Use SvelteKit's static output for the extension's packaged pages. Load and register the published custom-element runtime in the browser, and use its supplied framework types for template integration. At this source revision, `@svebcomponents/atproto.comments/svelte` is a type-only entry, not a native Svelte component export.

The component's `threadData` channel is useful for runtime prefetching as well as SSR. Using it in an extension does not require an HTTP rendering server. Bind account data and thread snapshots only after the extension starts; browser APIs must stay out of build-time rendering.

The package's `./ssr` entry and svebcomponents' SvelteKit integration can improve first paint on hosted public documentation pages. Use public example content there, and keep account credentials out of serialized properties. The extension's runtime feed does not require a separate rendering server.

## What stays in Cloudless

- The X DOM adapter, slot tracking, scroll stability, and extension iframe lifecycle.
- Browser permissions, trusted storage, extension messaging, and sign-in UI wiring.
- User-selected filters, classification providers, and replacement scheduling.
- Product branding and the reveal-original / correct-classification controls.

ATProto packages should not acquire X selectors, classification-provider credentials, or a hard dependency on Chrome extension APIs. Consume upstream releases through public exports, record compatible versions, and use a temporary fork only when an explicit, documented development need requires one.

Keep dependency attribution in technical documentation and license notices. Feed cards identify the post's Bluesky origin and where interactions go; their controls stay focused on the reader's actions.

## Acceptance checks for the first implementation

- A card mounts the published custom element and uses its public styling API.
- A fresh preloaded snapshot renders immediately without an unnecessary second read; a stale snapshot revalidates through the supported path.
- Root moderation labels survive normalization and are enforced in feed presentation. Unsupported embeds have an explicit fallback.
- Discover and Following use their respective authenticated endpoints. Switching account, feed, or language cannot reuse another source's cursor or queued posts, and ranked feeds keep their returned order.
- Sign-in and account changes reach all mounted cards without exposing credentials to the page or component properties.
- Existing likes / reposts restore correctly, toggles can be undone, and reply root / parent references remain correct through the component's own composer.
- Expanding replies preserves thread semantics; collapsing or recycling a card releases live connections and pending work.
- The composer fits its frame, drafts survive normal UI updates, and scrolling stays stable with a bounded number of frames.
- Equivalent website use still works with the default client and existing hosted bridge contract.
