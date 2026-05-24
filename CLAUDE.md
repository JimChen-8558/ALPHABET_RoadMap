# ALPHABET_RoadMap

A two-file static web app for tracking game product release schedules. Traditional Chinese (`zh-TW`) UI, backed by Firebase Realtime Database via REST. No build system, no dependencies, no framework — just open the HTML in a browser.

## Files

- `index.html` — Public dashboard (dark theme). Tabs: 總覽 / 法規遊戲 / H5 遊戲 / 全部產品.
- `admin.html` — Password-protected editor (light theme). Edits status per product and writes back to Firebase.
- `README.md` — Title only.

There are no other source files, configs, or scripts. Both HTML files are self-contained (inline `<style>` and `<script>`).

## Data model

### Firebase
- URL: `https://alphabet-roadmap-b24b1-default-rtdb.asia-southeast1.firebasedatabase.app`
- Hardcoded in both files as `FB_URL`. No auth — all reads/writes hit the REST endpoint directly via `fetch`.
- Two top-level keys:
  - `sections` — keyed by section id (`law`, `h5`). Each holds `{id, label, order, products:[…]}`.
  - `status` — keyed by product id. Each holds the six dimension statuses plus `_name`, `_month`, `_launch`, `_notes`, `_updatedAt`.

### Sections (categories)
- `law` — 法規遊戲 (regulated). Platforms: `D27`, `APOLLO`.
- `h5` — H5 遊戲. Platforms: `主流` (mainstream), `彩金` (jackpot).

### Product record (in `sections.<id>.products[]`)
```
{id, yr, mo, pl, nm, la, gm:[…]}
```
- `id` — string id (e.g. `l16`, `h202`). New ids are `secId[0] + Date.now()` from the admin modal.
- `yr`, `mo` — release year / month (1–12). Quarter is `Math.ceil(mo/3)`.
- `pl` — platform tag.
- `nm` — product name.
- `la` — launch ETA (free text, e.g. `2025/11月`).
- `gm` — sub-games (string array).

### Status record (in `status.<productId>`)
```
{planning, math, art, program, qa, cert,
 _name, _month, _launch, _notes, _updatedAt}
```
- The six dimension keys map 1:1 to `DIMS` in both files:
  - `planning` 企劃, `math` 機率, `art` 美術, `program` 程式, `qa` 品保, `cert` 送驗
- Each dimension value is one of: `''` (none), `on-time`, `delayed`, `early`.
- `_name`, `_month`, `_launch` are admin overrides that win over the values baked into the product record. The frontend reads `(fbData[id]&&fbData[id]._name)||p.nm` — always preserve this fallback chain.
- `_notes` — free-text note. `_updatedAt` — `Date.now()` ms.

### Overall status (derived)
`ov(id)` / `overall(id)` in the two files compute one of `on-time | delayed | early | none` from the six dimensions: any `delayed` wins, else any `early`, else all set → `on-time`, else `none`.

## Conventions to preserve when editing

1. **Keep `DIMS`, `S` (color/cls/label maps), and the `DEFAULT_SECTIONS` shape in sync between `index.html` and `admin.html`.** They are duplicated by design — there is no shared module. If you add a dimension or status, update both files.
2. **Both files poll Firebase every 5 seconds** (`setInterval` in `startListening`). The poll is guarded so it does not stomp on user interaction:
   - `index.html`: tracks `userTouching` (scroll / touch / mousedown) with a 60s timer.
   - `admin.html`: checks `document.activeElement` against `input | textarea | select`.
   Keep these guards — re-rendering during a touch or while a field is focused will drop user input.
3. **Writes use REST `PATCH` for partial status updates and `PUT` for whole-section rewrites.** See `fbPatch` / `fbSet` in `admin.html`. After a write, the admin file updates the DOM in place (pill colour, timestamp) rather than calling `render()` — this avoids losing focus on the editing field.
4. **Admin password** lives in `admin.html` as `CORRECT = 'alpha.net'`, persisted via `sessionStorage[PW_KEY]`. There is no real auth — Firebase is open. Do not invent a security story; if the user asks about auth, point out that the password is client-side only and the DB is public.
5. **Language is Traditional Chinese.** UI strings, labels, and comments are in zh-TW. Match the existing tone when adding strings; don't translate existing text to English unless asked.
6. **Styling tokens** live in `:root` in each file (different palettes — dark for `index`, light for `admin`). Use the existing CSS custom properties (`--gold`, `--green`, `--red`, `--blue`, `--grey`, `--law`, `--h5`) rather than introducing new colors.
7. **No frameworks.** Vanilla JS with global `window.fn = …` handlers used inline from `onclick=`. Stay in this style — do not introduce React, modules, or build steps.

## Running

There is nothing to build or install. Open `index.html` (or `admin.html`) directly in a browser, or serve the directory with any static server (e.g. `python3 -m http.server`). Firebase is reachable over the public internet from anywhere; offline use will show skeletons and "連線中…" indefinitely.

## Git workflow

- The user's working branch for documentation changes is `claude/claude-md-docs-8ginI`.
- Commit history shows the repo is maintained mostly via "Add files via upload" commits from the GitHub web UI — granular, descriptive commit messages from this end are still preferred when committing programmatically.
- Do not create PRs unless explicitly asked.
