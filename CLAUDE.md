# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install          # Install dependencies
npm run dev          # Start dev server at http://localhost:3000
npm run build        # Production build
npm run preview      # Preview production build locally
```

There is no test runner or linter configured in this project.

**Required environment variable**: create `.env.local` with `GEMINI_API_KEY=<your-key>`. Vite injects it at build time as both `process.env.API_KEY` and `process.env.GEMINI_API_KEY` (see `vite.config.ts`).

## Architecture

"Past Forward" is a single-page React + TypeScript app (Vite) that uploads a user photo and uses the Gemini API to regenerate it across six 1990s German youth subcultures.

### App state machine (`App.tsx`)

The entire app state lives in one component with four sequential states:

```
idle → image-uploaded → generating → results-shown
```

- `idle`: intro animation with ghost polaroids, file upload prompt
- `image-uploaded`: shows the uploaded photo, awaits confirmation
- `generating`: fires API calls and renders polaroids as they resolve
- `results-shown`: all done; album download and reset available

### Image generation flow

`STYLES` (defined at the top of `App.tsx`) is the single source of truth for what subcultures exist — each entry has a `name` and a detailed `prompt`. **Adding a new subculture means adding an entry here.**

Generation runs with a concurrency limit of 2 using a worker-queue pattern (`Array(2).fill(null).map(async () => { while (queue.length) ... })`). Each style's result is stored in `generatedImages` keyed by `style.name` with a `pending | done | error` status.

### `services/geminiService.ts`

Wraps `@google/genai` with two resilience layers:
1. **Retry on 500/INTERNAL**: up to 3 attempts with exponential backoff (1s, 2s, 4s)
2. **Fallback prompt**: if the model returns text instead of an image (typically due to content policy in certain regions), it retries with a simpler generic prompt via `getFallbackPrompt()`

The model used is `gemini-2.5-flash-image`. All images are passed and returned as `data:image/...;base64,...` strings — nothing is uploaded to any server.

### `components/PolaroidCard.tsx`

The main visual component. Behaviour varies by context:
- **Desktop**: wrapped in `DraggableCardContainer` / `DraggableCardBody` — cards are freely draggable. **Shake-to-regenerate** is implemented by detecting high velocity with a direction reversal during drag (`dotProduct < 0`), with a 2-second cooldown.
- **Mobile** (`isMobile` prop): rendered as a plain `div` in a vertical scroll list; regeneration is a button overlay instead.

Photo-developing animation: when a new image loads, the card starts in sepia/dark mode and transitions to full colour over 4 seconds (`isDeveloped` state toggled 200ms after `onLoad`).

### `lib/albumUtils.ts`

Canvas-based renderer that composites all six generated images onto a 2480×3508px (A4-like) parchment background with polaroid frames, random slight rotations, drop shadows, and handwritten captions. Returns a JPEG data URL at 90% quality for download.

### `components/ui/draggable-card.tsx`

Low-level framer-motion primitive providing 3D tilt (rotateX/Y via spring) and a radial glare effect on hover, plus full drag support with physics-based bounce on release.

### Styling

- **Tailwind CSS** is loaded via CDN (`<script src="https://cdn.tailwindcss.com">`) in `index.html` — there is no PostCSS or Tailwind config file.
- **Google Fonts** (Caveat 700, Permanent Marker, Roboto) are loaded in `index.html`. The classes `font-caveat` and `font-permanent-marker` are defined as inline `<style>` rules there, not as Tailwind plugins.
- Use `cn()` from `lib/utils.ts` (clsx + tailwind-merge) for conditional class composition.
- Path alias `@` resolves to the repo root (e.g., `@/lib/utils`).
