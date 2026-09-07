# AGENTS.md

## What is this

A Breeze plugin for CopyManga (拷贝漫画). Bundled as a single CJS file via rspack, loaded by the Breeze app runtime. SDK is `breeze-plugin-kit`.

## Quick commands

```sh
pnpm install
pnpm build          # typecheck → version sync → manifest gen → rspack → brotli
pnpm dev            # dev server with hot reload (port 7878)
pnpm typecheck      # tsc --noEmit only
pnpm lint           # eslint .
pnpm format         # prettier . --check
```

No test suite. No CI config in repo.

## Build pipeline order

`pnpm build` runs sequentially:
1. `tsc --noEmit` (typecheck)
2. `generate-version.ts` — copies `PLUGIN_VERSION` from `src/config.ts` into `package.json`
3. `generate-manifest.ts` — generates `manifest.json` from `src/get-info.ts`
4. `rspack build` — bundles `src/index.ts` → `dist/breeze-plugin-copy-comic.bundle.cjs`
5. `generate-brotli.ts` — creates `.bundle.cjs.br` brotli-compressed copy

## Version source of truth

`PLUGIN_VERSION` in `src/config.ts:13`. All other version locations (`package.json`, `manifest.json`) are generated from it. **Edit only `src/config.ts`**, then run `pnpm build`.

## Code style

- **Prettier**: double quotes, semicolons, trailing commas, 100 char width (`.prettierrc`)
- **ESLint**: typescript-eslint recommended + prettier. `no-console` is off (console.log used for plugin runtime logging). `.cjs` files allow `require()`.
- **No comments** in code unless explicitly requested.

## Architecture

- **`src/index.ts`** — single entry point, exports default object with all plugin API functions (`getHomeRecommend`, `searchComic`, `getChapter`, `fetchImageBytes`, etc.)
- **`src/common.ts`** — shared helpers (`toStringMap`, `createImage`, `createActionItem`, etc.)
- **`src/config.ts`** — constants: plugin ID, version, API endpoints, rate limits
- **`src/types.ts`** — all TypeScript types (payloads, API responses)
- **`src/limiter.ts`** — rate limiter and concurrency limiter for API/image requests
- **`src/get-info.ts`** — builds plugin manifest info (duplicated in `manifest.json` via generation)
- **`build/console-location-loader.ts`** — rspack loader that injects file:line:col into `console.*` calls for debugging

## Key gotchas

- **rspack, not webpack.** Config is in `rspack.config.ts` + `rspack.shared.ts`. Uses `builtin:swc-loader` for TS.
- **Output is CJS (`commonjs2`)**, despite the project being ESM (`"type": "module"`). The bundle is consumed by a CJS plugin runtime.
- **`console-location-loader.cjs`** exists because rspack loaders must be CJS. It uses `tsx/cjs` to bridge.
- **API domain switching**: the plugin supports multiple API domains (国际服, 大陆专线, 热辣漫画, etc.) with anti-piracy fallback headers. See `API_DOMAIN_BASE_MAP` in `src/index.ts`.
- **Rate limiting is shared across invocations** via `cache` (plugin runtime KV store). Rate limit state keys are prefixed `copyComic:rateWindow:v1:`.
- **Chapter caching**: chapter lists and content are cached with 10-minute TTL (`CHAPTER_CACHE_TTL_MS` in `src/config.ts`).
- **`copymanga_mirror_flow.md`** documents an alternative mirror site API (AES-encrypted endpoints). Useful context if working on mirror site support.
