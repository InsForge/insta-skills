# Framework deploy recipes

Copy the recipe for the app's framework **before writing a Dockerfile from scratch** — each one
has the four first-deploy traps already solved, so `insta deploy .` works on the first try.

## The four traps (why first deploys fail)

Every recipe below encodes these. If you hand-write a Dockerfile, get all four right:

1. **Bind `::` (dual-stack), never `0.0.0.0`-only or `127.0.0.1`.** InstaCloud's router and private
   network are **IPv6**. An app listening only on IPv4 boots "successfully" and then every request
   404s/502s forever. Node's `server.listen(port, '::')`. Next's standalone server does
   `server.listen(port, process.env.HOSTNAME || '0.0.0.0')`, and `0.0.0.0` binds **IPv4-only** —
   so set **`HOSTNAME=::`** (verified: binds dual-stack). `0.0.0.0` happens to work on Fly's
   IPv4-reachable proxy but fails on IPv6-only networks like Railway — `::` is safe on both.
2. **`EXPOSE <port>` in the Dockerfile.** `insta deploy` derives `--port` from the last `EXPOSE`;
   without it the service wires to 8080 and refuses every request. Keep `EXPOSE` == the listen port.
3. **`PORT` env == the exposed port.** Read `process.env.PORT` and default it to the same number you
   `EXPOSE`. (The platform injects `PORT`; a mismatch is the classic 502.)
4. **Build only what runs.** Multi-stage: build in one layer, copy just the runtime output into a
   slim final image. Keeps images small and start fast.

Credentials always arrive via injected env (`DATABASE_URL`, `BUCKET_NAME`, the `AWS_*` S3 bundle,
plus your `insta secrets set` values) — never bake them into the image.

## Next.js (App Router or Pages) — the common case

`next.config.mjs` **must** set standalone output:

```js
/** @type {import('next').NextConfig} */
export default { output: 'standalone' }
```

`Dockerfile`:

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json ./
RUN npm install
COPY . .
RUN npm run build
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production PORT=3000 HOSTNAME=::
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

Then: `insta deploy .` (port auto-derives from `EXPOSE 3000`). Route handlers read `process.env`
for `DATABASE_URL` / the S3 bundle. Pool Postgres at module scope; set `idleTimeoutMillis` under
Neon's suspend window so a scaled-to-zero DB doesn't leave a dead socket.

## Node/Express (API or full-stack, backend serves the built SPA)

```js
// bind '::' — dual-stack; PORT matches EXPOSE
app.listen(process.env.PORT || 3000, '::', () => console.log('up'))
```

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json ./
RUN npm install --omit=dev
COPY . .
ENV PORT=3000
EXPOSE 3000
CMD ["npm", "start"]
```

## Vite / static SPA (served by a tiny Node static server)

Build to `dist/`, serve it with a dual-stack static server so client routes fall back to
`index.html`. Simplest is a 15-line Express static server (bind `::`, `EXPOSE 3000`) using the
Node recipe above with `app.use(express.static('dist'))` + a `* → dist/index.html` fallback.
Deploy it as its own compute service; point it at the API via a build-time env var.

## Python / FastAPI (uvicorn)

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV PORT=8000
EXPOSE 8000
# --host :: binds dual-stack (IPv6 + mapped IPv4)
CMD ["sh","-c","uvicorn main:app --host :: --port ${PORT}"]
```

## After any deploy — verify (non-negotiable)

`curl` the printed URL's health path until it's 200 (cold start takes a few seconds). A 404/502
that never clears almost always means trap #1 (bound IPv4-only) or #3 (PORT≠EXPOSE) — check
`insta logs compute`, which prints the platform's "instance refused connection" hint.
