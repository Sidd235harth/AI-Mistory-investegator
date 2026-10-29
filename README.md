# AI Mystery Room

A solo detective game: provide a scenario, inspect the generated evidence, keep notes, and accuse a suspect.

## Requirements

- Node.js 20 or newer
- MongoDB (local or hosted)
- Optional: Google Gemini API key for the AI provider integration

## Start locally

1. Copy `.env.example` to `.env` and set `MONGODB_URI` and a private `JWT_SECRET`.
2. Install dependencies with `npm install`.
3. Run `npm run dev` from the repository root.
4. Open `http://localhost:5173`. The API health route is `http://localhost:5000/api/health`.

The default provider is `AI_PROVIDER=mock`, which generates a deterministic sample mystery without a Gemini key. Client and server start together. MongoDB must be reachable before the API starts.

## Commands

- `npm run dev` — run client and API in development
- `npm run build` — build shared schemas, server, and client
- `npm run start` — start the server build with `NODE_ENV=production`
- `npm run typecheck` — type-check all workspaces
- `npm run lint` — run ESLint
- `npm test` — run server and client test suites
- `npm run smoke --workspace server` — run the live API smoke flow using `API_BASE_URL`

## Environment

See `.env.example` for server, Gemini, rate-limit, storage, and client settings. `AI_PROVIDER=mock` selects the deterministic fixture; `AI_PROVIDER=gemini` enables structured Gemini text and image generation. Set Gemini model identifiers to models enabled for your Google AI account. Scenario inputs are untrusted prompt data and are delimited in model prompts.

Run `npm run gemini:check --workspace server` after configuring the API key and text model to make a real structured blueprint call and print its schema summary.

## Architecture

- `shared/` — shared Zod request and AI schemas
- `server/` — Express API, Mongoose models, auth, job pipeline, scoring, and case DTOs
- `client/` — React/Vite investigation game

Unresolved case and evidence responses go through explicit public DTO mappers. Hidden blueprint and evidence clue fields are excluded by Mongoose selection defaults and only loaded for server-side resolution. The API uses JWT auth in an HTTP-only cookie with an in-memory bearer token for the active client session.

Mock mode uses illustrative remote image URLs. Generated Gemini images and CCTV stills use the configured storage adapter: local disk for development or an S3-compatible private bucket for production. S3 records store UUID-only object keys, while evidence reads generate fresh presigned URLs.

## Deployment

The production topology uses a static Vite client on Vercel, the long-running Express API on Render, MongoDB Atlas, and a private S3-compatible bucket (Cloudflare R2 or AWS S3) for generated assets.

### 1. Create the MongoDB Atlas database

1. Create an Atlas project and cluster, then create a database user with access limited to the application database.
2. Add the Render service's outbound network access to the Atlas network access list. Use the narrowest network rule compatible with your Render plan and organization policy.
3. Copy the application connection string into Render's `MONGODB_URI` setting and set the application database name. Keep the URI private.

### 2. Create a private R2 or S3 bucket

1. Create a private bucket for generated case assets. Do not enable public bucket access or bucket listing.
2. Create credentials limited to object read, write, and delete in that bucket.
3. Set the bucket name, region, credentials, and `STORAGE_DRIVER` in Render. For R2, set `S3_ENDPOINT` to the account's S3 API endpoint and use the region required by the R2 S3 API. For AWS S3, leave the custom endpoint unset and select the bucket's AWS region.
4. If the bucket requires browser CORS rules, allow `GET` and `HEAD` from the Vercel client origin. The API signs individual object reads; the app does not use bucket listing or public object URLs.

### 3. Deploy the API on Render

1. Connect the repository to Render and create the service from the root `render.yaml` blueprint.
2. In the Render dashboard, add the Atlas URI, a unique random JWT secret of at least 32 characters, the deployed Vercel origin, Gemini credentials and model IDs, and the S3 settings.
3. Set the AI provider and review the generation limits. Keep the generation kill switch disabled until external settings are ready.
4. Deploy. Render builds the workspace and starts the server with production mode; its health check uses `/api/health`.

### 4. Deploy the client on Vercel

1. Import the repository in Vercel and set the project Root Directory to `client`, which applies `client/vercel.json` SPA rewrites.
2. Add `VITE_API_BASE_URL` pointing to the Render API's `/api` base path.
3. Deploy the static Vite build. Copy the deployed site's exact HTTPS origin into Render's `CLIENT_ORIGIN`. For preview deployments that should use the API, add each approved preview origin as a comma-separated entry.

### Environment variable reference

Set server variables in Render and client build variables in Vercel. The table lists names and descriptions only.

| Name | Description |
| --- | --- |
| `NODE_ENV` | Server runtime environment; production enables secure cross-site cookies. |
| `PORT` | HTTP port supplied by the hosting platform. |
| `CLIENT_ORIGIN` | Comma-separated allow-list of client origins. |
| `MONGODB_URI` | MongoDB Atlas application connection string. |
| `JWT_SECRET` | Private signing secret; production requires at least 32 characters. |
| `JWT_EXPIRES_IN` | JWT lifetime. |
| `AI_PROVIDER` | AI provider selection (`mock` or `gemini`). |
| `GEMINI_API_KEY` | Private Gemini API credential. |
| `GEMINI_TEXT_MODEL` | Gemini text model identifier. |
| `GEMINI_IMAGE_MODEL` | Gemini image model identifier. |
| `GEMINI_IMAGE_CONCURRENCY` | Maximum concurrent Gemini image requests. |
| `GENERATION_ENABLED` | Global generation kill switch. |
| `MAX_CASES_PER_USER_PER_DAY` | Per-user rolling 24-hour case cap. |
| `MAX_CONCURRENT_JOBS_PER_USER` | Per-user concurrent generation cap. |
| `MAX_GENERATIONS_PER_DAY_GLOBAL` | Global rolling 24-hour generation cap. |
| `STORAGE_DRIVER` | Asset storage selection (`local` or `s3`). |
| `S3_ENDPOINT` | Optional custom S3-compatible API endpoint, used for R2. |
| `S3_BUCKET` | Private generated-assets bucket name. |
| `S3_REGION` | S3-compatible bucket region. |
| `S3_ACCESS_KEY_ID` | Private bucket access key. |
| `S3_SECRET_ACCESS_KEY` | Private bucket secret key. |
| `VITE_API_BASE_URL` | Render API base URL compiled into the client. |
| `API_BASE_URL` | Running API base URL used by the manual smoke script. |

### Cross-domain authentication cookies

Vercel and Render use different domains. In production the API sets its auth cookie as `HttpOnly`, `Secure`, and `SameSite=None`; the Vite client sends credentialed requests. Both sites must use HTTPS, and the exact Vercel origin must be allowed by `CLIENT_ORIGIN`. The JWT response remains available to the client for its in-memory bearer session; do not store it in browser local storage.

### Post-deploy checklist

- [ ] Confirm `/api/health` returns HTTP 200 and reports the database as connected.
- [ ] Confirm the Vercel home page and a React Router deep link load directly.
- [ ] Confirm the browser receives the secure auth cookie and credentialed API calls succeed.
- [ ] Run `npm run smoke --workspace server` with `API_BASE_URL` set to the deployed API base; verify the smoke-created case is deleted.
- [ ] Generate a case and verify image/CCTV assets load from fresh signed URLs.
- [ ] Delete a generated case and confirm its database records and bucket objects are removed.
- [ ] Confirm the production JWT secret, Gemini credential, Atlas URI, and bucket credentials exist only in hosting dashboards.
- [ ] Review generation limits and the kill switch before sharing the site.

## Manual QA checklist

- [ ] Start MongoDB, run `npm run dev`, and confirm `/api/health` returns HTTP 200 with the database connected.
- [ ] In mock mode, create Easy, Medium, and Hard cases; verify progress stages and case opening.
- [ ] Submit an unsafe seed and confirm a readable rejection with a retry path.
- [ ] Open Chats, Photographs, CCTV, Suspect board, and Field notes at desktop and 360px widths.
- [ ] Scrub, step, play, and pause CCTV frames; confirm per-frame timestamps and a visible motion log.
- [ ] Save notes and suspect marks, reload the case, and confirm they persist.
- [ ] Request all three hints and confirm another request is rejected.
- [ ] Make an incorrect accusation, then a correct one; separately exhaust attempts and verify both verdict paths.
- [ ] While a case is unresolved, inspect every GET response in the Network panel and confirm there are no solution fields, clue IDs, red-herring flags, or unrevealed hints.
- [ ] Resolve a case and verify the verdict, clue evidence links, score breakdown, and history entry.
- [ ] Use keyboard only in the accusation dialog; confirm focus stays in the dialog, Escape closes it, and images have useful alt text.
- [ ] Delete a case with generated assets and confirm case data, evidence, attempts, jobs, and stored assets are removed.
- [ ] With configured Gemini settings, generate a case and confirm images load through the selected storage adapter.
#   A I - M i s t o r y - i n v e s t e g a t o r  
 