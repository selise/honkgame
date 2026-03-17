# Repository state (current)

This repo is a **static HTML5 canvas game** with a **hybrid SELISE Blocks integration** for auth and leaderboard storage.

## What runs

- **Entrypoint**: `index.html`
- **Runtime config**: `settings.json` (loaded at runtime)
- **Static assets**: `assets/` (images + sounds)
- **Static hosting**: nginx container (see `Dockerfile` + `docker-compose.yml`)

### Local dev / run

- **Recommended**: run via Docker/nginx so `fetch('settings.json')` works.

```bash
docker compose up --build
# open http://localhost:8080
```

Notes:
- Opening `index.html` via `file://` can cause `settings.json` to fail loading (browser restrictions). The game will fall back to inlined defaults.

## Key files

- `index.html`: all UI + game logic + integrations (single page, large inline `<script>`)
- `settings.json`: gameplay tunables (bird/physics/traffic/audio/gameplay/environment/food)
- `assets/`: sprites + audio files referenced by `index.html`
- `Dockerfile`: nginx image copying `index.html`, `settings.json`, `assets/`
- `docker-compose.yml`: maps `8080:80`
- `.env.dev`: SELISE Blocks config values used for dev workflows (currently not wired into the static page)
- `selise-mcp-server.js`: local MCP proxy helper for SELISE Blocks automation
- `llm-docs/`: AI documentation bundle (mostly oriented around Selise Blocks React apps; treat as reference material)

## External services used by the game (hybrid)

### SELISE Blocks Data Gateway (GraphQL)

- **Endpoint**: `https://api.seliseblocks.com/data/v1/gateway`
- **Header**: `x-blocks-key: <BLOCKS_API.key>`

Current GraphQL operations visible in `index.html`:
- **Users**:
  - `getHonkGameUsers` (existence check by `Email`)
  - `insertHonkGameUser` (creates a user record with `Email`, `PasswordHash`, `Username`, `CreatedAt`)
- **Leaderboard**:
  - `getHonkGameLeaderboards` (top 10 by `Score`)
  - `insertHonkGameLeaderboard` (submits score entry)

### SELISE Blocks IDP (authentication)

- **Base URL**: `https://api.seliseblocks.com/idp/v1`
- **Header**: `x-blocks-key: <IDP_API.key>`

Current flows visible in `index.html`:
- **Login**: password grant via `POST /Authentication/Token` (stores JWT access/refresh tokens)
- **Token refresh**: refresh_token grant via `POST /Authentication/Token`
- **User info**: `GET /Authentication/GetUserInfo` (Bearer access token)
- **Logout**: `POST /Authentication/Logout` (invalidates refresh token server-side)

## Client-side persistence

`index.html` uses `localStorage` for:
- **Auth**: `pooppatrol_auth` (stores `user`, `accessToken`, `refreshToken`)
- **Player name**: `pooppatrol_player_name`
- **High score**: `pooppatrol_highscore_horiz`

## Runtime configuration surface (`settings.json`)

`index.html` defines default `settings` and then merges values from `settings.json` (when successfully fetched).

Top-level keys currently used:
- `bird`: `speed`, `size`, `margin`
- `physics`: `poopGravity`, `poopInitialVy`, `poopDetectionRadius`
- `traffic`: lane setup, speeds, spawn timing, difficulty pacing
- `audio`: `honkVolume`, honk chances
- `gameplay`: noise meter tuning
- `environment`: scroll speeds, counts, silhouette sizing, background selection
- `food`: spawn timing, speed range, size, gravity increase

## Containerization

- `Dockerfile` is a simple nginx static site image:
  - Copies `index.html`, `settings.json`, and `assets/` into `/usr/share/nginx/html/`
- `docker-compose.yml` builds the image and serves it on `localhost:8080`

## What’s “in progress” / non-standard

- **No JS build tooling**: there’s no `package.json`/bundler detected; the game is plain static HTML/CSS/JS.
- **Selise Blocks React docs included**: `llm-docs/` contains patterns for a different stack (React/TS). In this repo, it functions as guidance/reference rather than the implemented architecture.

## Risk notes (demo stance)

You indicated this is **ok as a demo**, so this section is informational rather than “must fix now.”

- **Keys/tokens in repo**:
  - `.env.dev` contains a `VITE_X_BLOCKS_KEY` and related values.
  - `index.html` currently hardcodes Blocks keys inside `BLOCKS_API` and `IDP_API`.
  - `selise-mcp-server.js` contains a fallback `SELIS_BLOCKS_TOKEN` value if the env var is unset.
- **Auth stored in `localStorage`**: convenient for demos, but it increases exposure to token theft if any XSS is introduced. (The code does use `escapeHtml()` in some UI rendering, which helps for those specific insertions.)

