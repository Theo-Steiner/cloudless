# Cloudless development

Build Cloudless on the public APIs of `svebcomponents/atproto`. Follow the
integration boundaries in `docs/atproto-integration.md` when implementing it.

- Describe the product through its behavior and concrete architecture. Keep
  user-facing copy focused on the feed and interactions, with dependency
  attribution in technical documentation and license notices.

- Consume the published ATProto packages through their public exports. Use
  the actual custom element for posts, threads, reactions, and composition.
- Reuse the client package's types, normalization, URI / rich-text helpers,
  and bridge API contract. Avoid parallel ATProto implementations in Cloudless.
- Implement reusable missing capabilities in the upstream ATProto packages;
  keep Chrome APIs, X selectors, classification, and replacement scheduling here.
- Use the hosted bridge for ATProto OAuth and account credentials. Browser
  sessions belong to the extension, never X's page storage or DOM.
- Keep proposed APIs clearly distinguished from available package exports.
  Do not import private package source or depend on shadow-DOM internals.
- Use SSR where it has a rendering benefit. Personalized feed snapshots arrive
  at runtime and must never be baked into the extension build.
- Check root-post moderation, unsupported embeds, shared account state, and
  bounded iframe / live-connection lifetimes when adding feed presentation.
- Use Spacecowboy's For You as the initial feed source:
  `at://did:plc:3guzzweuqraryl3rdkimjamk/app.bsky.feed.generator/for-you`.
  It and other custom generators use authenticated `getFeed`; Following uses
  `getTimeline`. Preserve source order and scope queues and cursors to the
  account, source, and language preferences.
- Preserve the existing checkout for quick read-only work. Use managed worktrees
  when isolation is needed; archive task-owned worktrees when no longer needed.
