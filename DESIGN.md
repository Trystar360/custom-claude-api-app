# Atlas — Personal Claude Studio

**A bring-your-own-key AI workstation. One Claude API key, every Claude capability, your rules.**

Design blueprint · v1 · 2026-06-28

---

## 1. The one-paragraph vision

Atlas is a single web app you open in any browser on desktop or phone, paste your own Anthropic API key into once, and from then on own outright. It is not a wrapper around someone else's chat product — it is *your* studio. Every message, persona, prompt, and saved file lives on your device. There is no backend, no account, no telemetry, no monthly fee on top of the API. You pay Anthropic for tokens and nothing else. Built Claude-first but behind a provider abstraction, so the day you want to add another model you change one file, not the whole app.

The design goal is a tool that is *feature-packed yet quiet*: power is available but never in the way. A first-time user sees a clean chat box. A power user finds personas, a prompt library, file analysis, multi-step automations, token accounting, and deep customization a tap away.

---

## 2. Principles

The whole design hangs on six rules.

**Your key never leaves your device.** The key is held in browser storage (optionally encrypted behind a passphrase) and sent only to `api.anthropic.com`. No proxy, no logging, no third party. This is the single most important promise and it shapes every later decision.

**Local-first, always.** Conversations, personas, prompts, and settings persist in IndexedDB. The app works offline for everything except the actual model call. Export and import are first-class, not afterthoughts — your data is a file you own.

**Progressive disclosure.** The surface is calm. Depth lives one layer down — a slide-out panel, a command palette, a long-press. Nobody is forced to learn the whole tool to send a message.

**Customizable to the bone.** Theme, density, fonts, default model, default system prompt, temperature, token budgets, keyboard shortcuts, which panels show — all user-settable and all exportable as a single settings file.

**Honest about cost and state.** Every response shows tokens in/out and an estimated price. Streaming is visible. Errors are plain-language, never a raw stack trace.

**One codebase, two shapes.** A responsive layout that becomes a real mobile app (installable PWA) on a phone and a multi-pane workstation on a wide screen — not a desktop site crammed onto glass.

---

## 3. Who it's for and the jobs it does

Atlas is for an individual who already has (or will get) an Anthropic API key and wants a personal, private, deeply tunable Claude tool rather than a consumer subscription. Across a day it covers four jobs, which map directly to the four feature pillars:

1. **Talk to a model that already knows my preferences** — chat with saved personas / system prompts.
2. **Fire off things I do repeatedly without retyping them** — a prompt and tool library.
3. **Work with my documents and files** — upload, analyze, summarize, generate.
4. **Set things to run on their own** — automations and scheduled or chained tasks.

---

## 4. Architecture at a glance

Atlas is a static single-page application. Three real files ship:

- `index.html` — the entire app: markup, styles, and logic in one self-contained file for portability. (In a maintained build this splits into modules; the prototype keeps it single-file so you can email it to yourself and it just runs.)
- `manifest.webmanifest` — makes it installable on a phone home screen or desktop dock.
- `sw.js` — a service worker that caches the shell so the app opens instantly and offline.

```
┌──────────────────────── Browser (your device) ────────────────────────┐
│                                                                        │
│  UI layer ── Chat · Personas · Library · Files · Automations · Settings│
│      │                                                                 │
│  State store (in-memory, reactive)                                     │
│      │                                                                 │
│  ┌───┴────────────┐     ┌──────────────────┐     ┌──────────────────┐  │
│  │ Persistence    │     │ Provider layer   │     │ Crypto / vault   │  │
│  │ IndexedDB +    │     │ Claude adapter   │     │ key storage,     │  │
│  │ localStorage   │     │ (+ future: GPT,  │     │ optional         │  │
│  │ (chats, prefs) │     │  Gemini, local)  │     │ passphrase       │  │
│  └────────────────┘     └────────┬─────────┘     └──────────────────┘  │
│                                  │                                     │
└──────────────────────────────────┼─────────────────────────────────────┘
                                   │ HTTPS, x-api-key,
                                   │ anthropic-dangerous-direct-browser-access: true
                                   ▼
                       api.anthropic.com /v1/messages  (streaming SSE)
```

**The provider layer is the extensibility hinge.** Every model call goes through one interface — `send(messages, options) → stream of events`. The Claude adapter implements it against `/v1/messages`. Adding OpenAI or a local Ollama endpoint later means writing one more adapter that emits the same event shape; nothing in the UI changes. That is what "Claude-first, extensible" buys you.

---

## 5. The four feature pillars, fleshed out

### Pillar 1 — Chat + Personas

The core surface. A streaming conversation with Markdown and code rendering, copy-per-block, edit-and-resend, regenerate, and branch (fork a conversation at any message to explore an alternative without losing the original).

A **persona** is a saved bundle: a name, an icon, a system prompt, a default model, a temperature, and optional starter prompts. Switching persona reframes the whole conversation. Your stated preferences — plainspoken thought-partner, strict analytical lens, the Russellian and scriptural framings — become a persona you select once rather than retype. Personas are exportable and shareable as small JSON files.

Per-conversation overrides sit in a slide-out inspector: model, temperature, max tokens, system prompt, and a token/cost meter for that thread.

### Pillar 2 — Prompt & Tool Library

A searchable, taggable collection of reusable prompts. Each entry supports `{{variables}}` so a single "competitive teardown" prompt can be filled with a company name at call time via a quick form. Entries can be pinned, grouped into folders, and launched straight into a new chat or inserted into the current one.

"Tools" here are saved *parameterized actions* — a prompt plus a target persona plus default variable values, runnable from a command palette in two keystrokes. A starter set ships (summarize, rewrite, extract action items, explain like I'm rusty, steelman, translate), all editable.

### Pillar 3 — Docs + Files

Drag a file onto the chat or the Files tab. Text, Markdown, CSV, and code are read and attached inline. PDFs and images use Claude's native document/vision support. CSVs get a quick table preview. Outputs that are clearly documents (a drafted report, a generated table) get a one-click "save as file" so the result leaves the chat as a real `.md`, `.txt`, or `.csv`. A lightweight file shelf keeps recently used attachments around so you can reapply them to a new question without re-uploading.

### Pillar 4 — Automations

Two kinds. **Chains**: an ordered series of prompts where each step's output feeds the next (e.g., outline → draft → critique → revise), runnable on demand against an input. **Schedules**: a saved prompt or chain set to run on a cadence ("every morning, summarize this feed and write me three bullets"). In a pure-browser build, schedules run while the app is open and notify via the Notifications API; a documented optional path uses the platform's scheduled-task runner for true background execution. Every automation run is logged with its output and cost so nothing happens invisibly.

---

## 6. Data model

Six small record types, all local:

- **Conversation** — id, title, personaId, createdAt, model, messages[], tokenTotals.
- **Message** — id, role, content (blocks), attachments[], usage, parentId (for branching).
- **Persona** — id, name, icon, systemPrompt, model, temperature, starters[].
- **PromptTemplate** — id, title, body (with `{{vars}}`), tags[], folderId.
- **Automation** — id, type (chain|schedule), steps[], cadence, lastRun, runLog[].
- **Settings** — theme, density, font, defaults, shortcuts, providerConfigs[].

Everything serializes to one JSON export ("Atlas backup") and restores from it. That single file is your account.

---

## 7. UI / UX

**Wide screen (desktop):** three zones — a left rail of conversations and personas, a center conversation, and a right inspector (settings for this thread, token meter, attachments). The inspector collapses to reclaim space.

**Phone:** a single column with a bottom tab bar (Chat · Library · Files · Automate · Settings). The left rail becomes a slide-over; the inspector becomes a bottom sheet. Touch targets are large, the composer sticks to the bottom above the keyboard, and long-press opens message actions.

**Cross-cutting:** a command palette (Ctrl/Cmd-K) is the fast path to everything — switch persona, insert a prompt, run a tool, jump to a chat. Theme supports light, dark, and system, with an accent color and a density toggle. Type scale and font are user-set for readability.

---

## 8. Security & privacy

The key is the crown jewel. It is stored locally and, if the user sets a passphrase, encrypted at rest with WebCrypto (AES-GCM, key derived via PBKDF2) so a casual look at browser storage reveals nothing usable. It is transmitted only to Anthropic over HTTPS. The app makes no other network calls — no analytics, no fonts-from-a-CDN that could phone home in the hardened build. A visible reminder explains the real risk of the browser pattern: anyone with physical access to an unlocked device-and-app can use the key, so the passphrase lock and a "clear key on exit" option both exist. Export files containing the key are clearly labeled and the key can be excluded from a backup.

---

## 9. Build roadmap

**v1 (this session)** — installable PWA shell; bring-your-own Claude key with local storage; streaming chat with Markdown/code; personas with the user's preference-personas preloaded; prompt library with variables; file attach (text/CSV/image/PDF); settings with theme, density, model, and defaults; token + cost meter; full JSON export/import. Single self-contained file.

**v2** — passphrase-encrypted vault; conversation branching; chain automations; richer file shelf; per-message cost breakdown; keyboard-shortcut editor.

**v3** — scheduled automations via the platform task runner; second provider adapter (OpenAI-compatible) to prove the abstraction; shareable persona/prompt packs; optional sync via a user-chosen file location.

---

## 10. Why this shape is right

The bring-your-own-key, no-backend design is not a limitation dressed up as a virtue — it is the only architecture that keeps the core promise literally true. There is no server that *could* leak your data because there is no server. The provider abstraction means Claude-first never becomes Claude-only by accident. And the progressive-disclosure UI is what lets the same artifact be both a clean chat box for a quick question and a full workstation for a long working session, on both a laptop and a phone, without two codebases. The prototype that follows implements the v1 row in full.
