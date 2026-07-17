# HydroCore

HydroCore is a clean baseline for building water-treatment applications. This repository keeps the generic system foundation: authentication, RBAC, menu management, organization/user management, system configuration, generic data query plumbing, and the frontend shell.

This baseline does not include real water-treatment process models, device control algorithms, prediction algorithms, reports, or production plant data. Add those through separate OpenSpec/Comet changes.

## Repository Layout

- `hydrocore-be/`: Spring Boot backend.
- `hydrocore-fe/`: Vue 3 frontend.
- `openspec/`: OpenSpec and Comet change artifacts.
- `docs/`: product-level and cross-repository documentation.

The frontend and backend directories can keep their own Git history. This root repository is used for coordination docs and OpenSpec/Comet artifacts.

## Verify

```powershell
cd hydrocore-be
mvn -q -DskipTests compile
mvn -q test

cd ..\hydrocore-fe
pnpm install
pnpm run build
```

## Agent And IDE Policy

Use global Comet as the only agent workflow entry point for this project. OpenSpec and Comet artifacts live in this repository because they describe product changes. Local agent, IDE, and skill installations should come from the user's global environment.

Do not add repository-local `.agents/`, `.codex/`, `.claude/`, `.cursor/`, or `.github/skills/` copies unless a future change explicitly documents why they are required.

## Git Policy

No code has to be committed until a proper remote repository and branch strategy are ready. When Git is ready, keep one branch name per product change across root, backend, and frontend repositories.
