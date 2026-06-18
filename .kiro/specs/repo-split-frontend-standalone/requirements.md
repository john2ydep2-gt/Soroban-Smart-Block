# Requirements Document

## Introduction

PERO-J currently lives in a single monorepo containing three components: a React/Vite/TypeScript frontend (`frontend/`), a Node.js indexer and Express REST API (`indexer/`), and Rust/Soroban smart contracts (`contracts/`). This feature splits the monorepo into two independent repositories:

- **pero-j-frontend** — contains only the `frontend/` directory and must build, develop, and deploy independently.
- **pero-j-backend** — contains the `indexer/` and `contracts/` directories along with shared project-level files.

After the split, the frontend must be configurable to point at any deployed instance of the backend REST API without code changes, and each repository must be self-contained with its own documentation, environment template, CI-ready build instructions, and version control history.

## Glossary

- **Frontend_Repo**: The new standalone repository containing only the `frontend/` directory contents (`pero-j-frontend`).
- **Backend_Repo**: The new repository containing the `indexer/` and `contracts/` directories plus shared project files (`pero-j-backend`).
- **Vite_Proxy**: The development-time reverse proxy configured in `vite.config.ts` that forwards `/api` requests to the indexer during local development.
- **API_Base_URL**: The runtime-configurable URL prefix used by the frontend to construct REST API requests (e.g., `https://api.example.com`).
- **VITE_API_BASE_URL**: The Vite environment variable that supplies the API_Base_URL at build time and development time.
- **api.ts**: The TypeScript module at `frontend/src/api.ts` that constructs and dispatches all HTTP requests to the backend REST API.
- **env.example**: A committed template file listing all required environment variables with placeholder values and explanatory comments, not containing real secrets.
- **Standalone_Build**: A `tsc && vite build` invocation that succeeds to completion and produces a deployable `dist/` artifact without requiring any sibling directories from the original monorepo.
- **Root_Cargo_toml**: The Cargo workspace manifest at the monorepo root (`Cargo.toml`) that declares the `contracts/explorer` member.
- **Root_Makefile**: The `Makefile` at the monorepo root containing targets for all three components.
- **Shared_Docs**: Project-level files (`README.md`, `stellar.toml`, `BUDGET.md`, `ROADMAP.md`, `MANIFEST.md`, `TEAM.md`, `LICENSE`) currently at the monorepo root.

---

## Requirements

### Requirement 1: File Partitioning — Frontend_Repo Contents

**User Story:** As a frontend developer, I want a repository that contains only the frontend source code and its direct configuration, so that I can clone, install, and work on the UI without pulling in Rust toolchains or backend Node.js code.

#### Acceptance Criteria

1. THE Frontend_Repo SHALL contain all files currently under `frontend/` (`src/`, `index.html`, `package.json`, `tsconfig.json`, `vite.config.ts`) promoted to the repository root.
2. THE Frontend_Repo SHALL contain a `.gitignore` file that excludes `node_modules/`, `dist/`, and `.env`.
3. THE Frontend_Repo SHALL contain an `env.example` file listing `VITE_API_BASE_URL` with a placeholder value and an explanatory comment.
4. THE Frontend_Repo SHALL contain a `README.md` describing frontend-specific setup, the `VITE_API_BASE_URL` variable, and the `npm run dev`, `npm run build`, and `npm run preview` commands.
5. THE Frontend_Repo SHALL NOT contain `indexer/`, `contracts/`, `Cargo.toml`, or any Rust-specific files.
6. THE Frontend_Repo SHALL NOT contain the Root_Makefile; WHERE a convenience script is desired, THE Frontend_Repo SHALL provide npm scripts in `package.json` as the sole build interface.

---

### Requirement 2: File Partitioning — Backend_Repo Contents

**User Story:** As a backend developer, I want a repository that contains the indexer and contracts alongside the shared project documentation, so that I can run the full data pipeline and deploy contracts without the frontend source code.

#### Acceptance Criteria

1. THE Backend_Repo SHALL contain the `indexer/` directory with all its source files and `package.json` preserved at the same relative path.
2. THE Backend_Repo SHALL contain the `contracts/` directory and the Root_Cargo_toml (renamed to the repository-root `Cargo.toml`) with the `contracts/explorer` workspace member intact.
3. THE Backend_Repo SHALL contain all Shared_Docs files (`README.md`, `stellar.toml`, `BUDGET.md`, `ROADMAP.md`, `MANIFEST.md`, `TEAM.md`, `LICENSE`) at the repository root.
4. THE Backend_Repo SHALL contain a root `.env.example` equivalent to the current monorepo `.env.example`, covering all indexer and contract variables (`SOROBAN_RPC_URL`, `HORIZON_URL`, `NETWORK_PASSPHRASE`, `DATABASE_URL`, `START_LEDGER`, `POLL_MS`, `PORT`, `EXPLORER_CONTRACT_ID`).
5. THE Backend_Repo SHALL contain a Root_Makefile retaining only the contract and indexer targets (`build`, `test`, `optimize`, `deploy`, `indexer-install`, `indexer`, `install`, `clean`), with the `frontend`, `frontend-install`, `frontend-build`, and `dev` targets removed.
6. THE Backend_Repo SHALL contain a `.gitignore` that excludes `target/`, `*.wasm`, `node_modules/`, `indexer/node_modules/`, and `.env`.

---

### Requirement 3: Frontend API Configuration via Environment Variable

**User Story:** As a developer deploying the frontend, I want to configure the backend API URL through an environment variable, so that I can point the same built artifact at development, staging, or production backends without modifying source code.

#### Acceptance Criteria

1. THE api.ts SHALL read the API_Base_URL from the `VITE_API_BASE_URL` environment variable using `import.meta.env.VITE_API_BASE_URL`.
2. WHEN `VITE_API_BASE_URL` is set to a non-empty string, THE api.ts SHALL use that value as the prefix for all API requests instead of the hardcoded `/api` string.
3. WHEN `VITE_API_BASE_URL` is not set or is empty, THE api.ts SHALL fall back to `/api` so that the Vite_Proxy continues to work during local development.
4. THE Frontend_Repo SHALL document in its `README.md` that `VITE_API_BASE_URL` must be set to the full origin and path prefix of the deployed backend API (e.g., `https://api.example.com`) for production builds.
5. IF a `VITE_API_BASE_URL` value contains a trailing slash, THEN THE api.ts SHALL strip the trailing slash before constructing request URLs, so that paths like `/api/events` are not duplicated.

---

### Requirement 4: Frontend Standalone Build

**User Story:** As a CI pipeline, I want to run `npm run build` inside the Frontend_Repo and receive a deployable `dist/` artifact, so that the frontend can be built and deployed independently of any backend code.

#### Acceptance Criteria

1. WHEN `npm run build` is executed in the Frontend_Repo root with `VITE_API_BASE_URL` set to a valid URL, THE Standalone_Build SHALL complete without errors and produce a `dist/` directory containing at minimum `index.html`.
2. WHEN `npm run build` is executed in the Frontend_Repo root without `VITE_API_BASE_URL` set, THE Standalone_Build SHALL complete without errors, falling back to the `/api` prefix.
3. THE Frontend_Repo `package.json` `build` script SHALL remain `tsc && vite build`, preserving the existing TypeScript type-check step before bundling.
4. THE Frontend_Repo SHALL NOT require any files outside its own directory to be present for the build to succeed.
5. WHEN `npm run dev` is executed in the Frontend_Repo root, THE Vite_Proxy SHALL forward `/api/*` requests to `http://localhost:3001` to preserve the local development workflow.

---

### Requirement 5: Vite Configuration Preservation and CORS Handling

**User Story:** As a developer, I want the Vite dev server configuration to continue working after the split, and I want to understand any CORS requirements that arise when the frontend is served from a different origin than the backend in production.

#### Acceptance Criteria

1. THE `vite.config.ts` in the Frontend_Repo SHALL retain the `server.proxy` configuration mapping `/api` to `http://localhost:3001`.
2. WHERE `VITE_API_BASE_URL` points to a different origin than the frontend host, THE Backend_Repo `README.md` SHALL document that the indexer's Express server must be configured with an appropriate CORS policy to allow requests from the frontend's origin.
3. THE Frontend_Repo `README.md` SHALL note that the Vite_Proxy eliminates CORS concerns during local development but that production deployments require CORS configuration on the backend.

---

### Requirement 6: Git History and Repository Initialisation

**User Story:** As a repository maintainer, I want each new repository to be properly initialised with git history options documented, so that the split is performed cleanly and traceably.

#### Acceptance Criteria

1. THE Frontend_Repo `README.md` SHALL document the recommended method to initialise the repository from the monorepo using `git filter-repo --subdirectory-filter frontend` to preserve commit history for the `frontend/` subtree.
2. THE Backend_Repo `README.md` SHALL document the recommended method to initialise the repository from the monorepo using `git filter-repo` to retain history for `indexer/`, `contracts/`, and root-level files.
3. THE Frontend_Repo `README.md` SHALL note that after `git filter-repo`, all file paths will be at the repository root (i.e., `frontend/src/` becomes `src/`), matching the new directory layout.
4. IF `git filter-repo` is not available, THEN the documentation SHALL provide an alternative initialisation path using `git clone` followed by manual removal of unneeded directories and a fresh initial commit.

---

### Requirement 7: Shared Documentation Handling

**User Story:** As a project maintainer, I want the shared project-level documents to live in the Backend_Repo as the canonical source, with the Frontend_Repo README linking back to them, so that there is a single source of truth for project-wide information.

#### Acceptance Criteria

1. THE Backend_Repo SHALL be the canonical location for `stellar.toml`, `BUDGET.md`, `ROADMAP.md`, `MANIFEST.md`, `TEAM.md`, and `LICENSE`.
2. THE Frontend_Repo `README.md` SHALL include a reference to the Backend_Repo repository URL for full project documentation, architecture details, and the REST API specification.
3. THE Frontend_Repo SHALL include its own copy of `LICENSE` so that the frontend repository is independently licensed without requiring a cross-repository reference.
4. THE Backend_Repo `README.md` SHALL be updated to reflect the two-repository structure, replacing the monorepo Quick Start section with separate setup instructions for each repository and noting that the frontend is in a separate repository.
