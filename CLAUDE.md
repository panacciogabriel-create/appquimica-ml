# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

**AppQuimica — Panel ML** is a single-page dashboard for a Mercado Libre
(Argentina) seller store. It lets the seller view their listings, unanswered
buyer questions, recent sales, and compare competitor prices. It also drafts
answers to buyer questions using the Anthropic (Claude) API.

The entire application is **one self-contained file**: `index.html`. There is
no build step, no package manager, no framework, no backend of its own. All
HTML, CSS, and JavaScript live inline in that file, and it talks directly to
external APIs from the browser.

## Repository layout

```
index.html   The whole app: inline <style>, markup, and inline <script>
README.md     One-line placeholder
CLAUDE.md     This file
```

## Running it

Open `index.html` directly in a browser, or serve the directory statically:

```bash
python3 -m http.server 8000   # then visit http://localhost:8000
```

There is nothing to install, build, compile, or test. No lint config, no CI.

> Note: API calls to Mercado Libre may be subject to CORS from a `file://`
> origin; serving over `http://localhost` is more reliable when testing live
> data.

## Architecture

Two screens toggled by CSS `display`, no router:

- **`#login-screen`** — a "connect your store" card. `entrarDirecto()` simply
  hides the login and shows the dashboard (credentials are already hardcoded;
  see Security below). It then starts the token timer and loads data.
- **`#dashboard`** — a top bar, a `.nav-tabs` row, and five `.section` panels.
  `showTab(name, event)` switches the active tab/section and lazy-loads each
  tab's data on first view (tracked via the `loaded` map).

The five tabs and their loader functions:

| Tab            | Section id           | Loader               |
| -------------- | -------------------- | -------------------- |
| Resumen        | `sec-resumen`        | `cargarResumen()` (loaded on entry) |
| Publicaciones  | `sec-publicaciones`  | `cargarPublicaciones()` |
| Preguntas      | `sec-preguntas`      | `cargarPreguntas()`  |
| Ventas         | `sec-ventas`         | `cargarVentas()`     |
| Competencia    | `sec-competencia`    | `buscarCompetencia()` (on search) |

### Data flow

- `mlFetch(path)` is the single helper for authenticated Mercado Libre API
  calls (`https://api.mercadolibre.com{path}` with a `Bearer` token). Use it
  for any new ML endpoint rather than calling `fetch` directly.
- Endpoints in use: `/users/me`, `/users/{id}/items/search`, `/items/{id}`,
  `/questions/search`, `/answers`, `/orders/search`, `/sites/MLA/search`.
- `generarIA(id, texto)` calls the Anthropic Messages API
  (`https://api.anthropic.com/v1/messages`) to draft a buyer-question reply in
  Spanish, then fills the corresponding response input.
- Token lifecycle: `iniciarTimerToken()` counts down `tokenExpiry` every 30s
  and calls `renovarToken()` (OAuth `refresh_token` grant) when it is nearly
  expired.

### UI feedback

- `toast(msg, ms)` shows a transient bottom-right notification.
- Loading states use the `.loading` + `.spinner` markup; empty states use
  `.empty` + `.empty-ico`.

## Conventions

- **Language:** All UI copy, comments, function names, and toasts are in
  **Spanish** (`lang="es"`, `es-AR` locale for number/date formatting via
  `toLocaleString('es-AR')`). Keep new user-facing text in Spanish and match
  the existing es-AR formatting.
- **Styling:** Plain CSS with a design-token color palette defined in
  `:root` (`--amarillo` yellow accent, `--azul*` dark blues, status colors
  `--verde`/`--rojo`/`--naranja`). Reuse these variables and the existing
  component classes (`.card`, `.stat-card`, `.item-row`, `.badge`, etc.)
  rather than introducing new ad-hoc colors. Fonts: `Syne` (headings) and
  `DM Sans` (body), loaded from Google Fonts.
- **JS style:** Vanilla ES modules-free script, `async`/`await`, direct
  `document.getElementById` / template-literal `innerHTML` rendering. No
  build tooling — keep it dependency-free and inline.
- **No framework:** Do not introduce React/Vue/bundlers/npm unless the user
  explicitly asks; the project's value is being a single portable file.

## Security note (important)

`index.html` currently contains **hardcoded live secrets** in the inline
script: `APP_ID`, `CLIENT_SECRET`, `ACCESS_TOKEN`, `REFRESH_TOKEN`, and the
Mercado Libre `USER_ID`. Anthropic API calls are also made directly from the
browser, which would expose an API key if one is added client-side.

- Do **not** add new secrets to client-side code.
- If you touch this area, flag to the user that committing these credentials
  is a risk (they should be rotated and moved behind a backend/proxy), rather
  than silently propagating the pattern.
- Treat any real token values as sensitive: do not echo them into commits,
  logs, or PR descriptions.

## Making changes

- Edit `index.html` in place; keep additions consistent with the existing
  inline structure and Spanish naming.
- After a change, sanity-check by opening the file in a browser and clicking
  through the affected tab.
- There are no automated tests; verification is manual.
