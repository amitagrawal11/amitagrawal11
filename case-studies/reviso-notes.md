# Reviso Notes — Case Study

A privacy-first, offline-capable markdown notes PWA built for the AI era. Paste raw markdown from ChatGPT, Claude, or Cursor — it just renders.

**Stack:** React 19 · TypeScript (strict) · Vite 6 · Mantine v7 · Supabase · vite-plugin-pwa · Playwright · Vitest

---

## The problem

Most of us now document everything in markdown, because that's what AI tools output. But existing notes apps weren't built for that workflow:

- **Notion / Evernote / Apple Notes** — don't speak markdown natively; you re-format what the AI already formatted.
- **Obsidian** — desktop-first, sync is paid or DIY, no one-tap mobile install.
- **Bear / Craft** — paywalled, Apple-only.
- **Gists / VS Code** — preserve markdown but have no real reading experience.

I wanted a notes app that is:

1. **Markdown-native** — paste AI output verbatim, it just works.
2. **Free, no signup to try** — open the URL and start writing.
3. **Cross-device** — phone, tablet, laptop; installable as a PWA.
4. **Synced online, local when offline** — never blocked by the network.
5. **Kindle-like to read** — calm typography, read mode, scroll-spy TOC, dark mode.

---

## The solution

Two parallel route trees from the same app:

- **`/demo`** — localStorage-backed, zero signup. The landing page links straight in.
- **`/`** — Supabase auth, Postgres + RLS + realtime sync.

Both share every component. A single `Repo` interface with two implementations (`noteRepositoryMock` for localStorage, `noteRepositorySupabase` for the cloud) is the only swap. Components call `useRepo()` and never know which backend they're talking to — which means **the offline story is the same as the demo story**.

---

## Markdown built for AI output

- `@uiw/react-md-editor` (nohighlight build) + `remark-gfm` + `remark-breaks` — full GFM.
- `rehype-highlight` with auto-detect for code blocks.
- **Mermaid** renders natively from fenced ` ```mermaid ` blocks.
- Auto-linked headings, scroll-spy TOC, zebra tables, copy-code buttons.

Paste a 4000-word ChatGPT response and read it like a published article.

---

## Kindle-like reading

- **Read mode** (`⌘.`) hides all chrome — only the content well remains.
- Tightened heading rhythm, hairline rules, anchor `scroll-margin-top`.
- Dual themes with WCAG AA contrast audit.
- **No FOUC** — inline pre-paint script sets the theme before the first frame.
- **CLS: 0.39 → ~0** via skeleton reservations.

---

## Offline + sync

| Scenario | Behavior |
|---|---|
| Online + signed in | Optimistic local update → Supabase write → realtime broadcast |
| Offline + signed in | Local update lands immediately; reconnect syncs |
| No account (demo) | localStorage only, fully functional |
| Installed PWA | Home-screen icon, standalone window, offline-capable |

IDs are generated client-side with `crypto.randomUUID()` — navigation to a new note never races the network.

---

## Architecture highlights

- **Swappable `Repo` interface** — one contract, two backends, zero conditional logic in components.
- **Mode-aware routing** — `useModePath()` prefixes `/demo` so one `<Link>` works in both trees.
- **Aggressive lazy-loading** — every page, AppLayout, DnD, dialogs, markdown editor, mermaid, command palette, and `@supabase/supabase-js` (~120 KB gz) live in separate chunks. Markdown + dialogs prefetch on hover.
- **LCP preload** — inline script injects a per-theme WebP preload only on `/`; runtime `<picture>` matches the srcset so nothing re-fetches.
- **Inline boot splash** in `#root` paints during HTML parse.

---

## Engineering practices

| Area | Approach |
|---|---|
| Type safety | Strict TS, `noUncheckedIndexedAccess`, enforced `import type` |
| Testing | Vitest at ~92% line coverage; Playwright e2e split desktop + mobile |
| Code layout | Feature folders (`features/notes`, `features/authentication`, …) + `shared/` |
| Accessibility | Lighthouse 100, WCAG audit, skip-to-content, safe-area + `100dvh` |
| PWA | NetworkFirst with 3s timeout, `globPatterns: []` to avoid stale shells |

---

## Results

- Landing critical chunk: **< 200 KB gz**
- CLS: **~0** (from 0.39)
- Lighthouse a11y: **100**
- No render-blocking stylesheets

---

## Worth a look in the code

- [`src/features/notes/repository/`](src/features/notes/repository/) — swappable repo + optimistic mutations
- [`src/app/App.tsx`](src/app/App.tsx) — two route trees, lazy pages, `RealRoot` auth gate
- [`index.html`](index.html) — pre-paint theme script, LCP preload, inline boot splash
- [`vite.config.ts`](vite.config.ts) — PWA + runtime caching
- [`scripts/screenshot-landing.ts`](scripts/screenshot-landing.ts) — Playwright + Sharp screenshot pipeline

---

## Try it

- **Demo (no signup):** `/demo`
- **Install as PWA:** Chrome/Safari → "Add to Home Screen"

---

## Takeaways

1. **Lazy-loading is a discipline.** One stray static import of Supabase puts 120 KB gz back on the landing chunk.
2. **Client-side IDs beat tmp-id swaps.** `crypto.randomUUID()` before the network call means navigation never races the server — and offline-created notes are stable forever.
3. **A `Repo` interface pays for itself.** Two backends, zero conditional logic, free demo *and* offline modes.
4. **Reading is a feature.** Treating the viewer as its own designed surface — not "the editor minus the toolbar" — is what makes notes worth coming back to.
