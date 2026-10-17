# Cloudless development

Cloudless is a real-world showcase for `svebcomponents/atproto`. Follow the
integration boundaries in `docs/atproto-integration.md` when implementing it.

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
- Preserve the existing checkout for quick read-only work. Use managed worktrees
  when isolation is needed; archive task-owned worktrees when no longer needed.
