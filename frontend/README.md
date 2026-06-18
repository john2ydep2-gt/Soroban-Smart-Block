# PERO-J Frontend

React + Vite + TypeScript frontend for the PERO-J Soroban block explorer.

## Setup

```bash
npm install
cp .env.example .env
# Edit .env and set VITE_API_BASE_URL
npm run dev
```

## Environment Variables

| Variable | Description | Default |
|---|---|---|
| `VITE_API_BASE_URL` | Full URL of the backend API (e.g. `https://api.example.com/api`) | `/api` (Vite dev proxy) |

Leave `VITE_API_BASE_URL` empty during local development — the Vite proxy will forward `/api` requests to `http://localhost:3001` automatically.

For production builds, set it to the full backend API URL before running `npm run build`.

## Scripts

| Command | Description |
|---|---|
| `npm run dev` | Start dev server at http://localhost:5173 |
| `npm run build` | TypeScript check + Vite production build → `dist/` |
| `npm run preview` | Preview the production build locally |

## Notes

- The Vite dev proxy (`/api → http://localhost:3001`) handles CORS during local development.
- In production, the backend Express server must allow CORS from the frontend's origin.
- Git history for this repo was split from the PERO-J monorepo using:
  ```bash
  git filter-repo --subdirectory-filter frontend
  ```
  After filtering, all paths are at the repo root (`frontend/src/` → `src/`).

## Full Project Docs

For architecture, REST API reference, contract docs, and deployment guides see the [pero-j-backend](https://github.com/your-org/pero-j-backend) repository.
