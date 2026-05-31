# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

"打了吗" (dalema) — a joke WeChat mini-program for a friend group to track daily "activity" (手冲打卡), with monthly leaderboards crowning the "飞机王" 👑.

The app is intentionally crude/silly in tone. Code naming should match this vibe: `joinSquad`, `startBattle`, `fireOne`, etc.

## Tech stack

| Layer | Choice | Notes |
|-------|--------|-------|
| Frontend | **uni-app (Vue3)** | Cross-platform escape hatch. Primary target: WeChat mini-program (体验版, no review needed). Fallback: compile to H5 and serve as a webpage if WeChat blocks it. |
| Backend | **Node.js + Koa2** | REST API, no auth layer |
| Database | **SQLite** (+ Prisma ORM) | Small data volume, zero deployment overhead |
| State (FE) | **Pinia** | Current user, active squad, squad list |

## Key architecture decisions

1. **No authentication.** Users generate a UUID client-side on first launch, pick a nickname, and pass `user_id` with every request. No passwords, no WeChat login, no tokens.
2. **Many-to-many users ↔ squads.** A user can belong to multiple squads (via `user_groups` join table). Check-ins are scoped to a specific squad (`user_id + group_id`). Each squad's leaderboard is independent.
3. **Squads are the core container.** Everything happens inside a squad. A user must create or join one before they can check in. Squad membership is by 6-digit numeric invite code.

## Data model (SQLite via Prisma)

```
users (id: UUID PK, nickname)
groups (id: auto PK, name, invite_code: unique 6-digit, owner_id, max_daily)
user_groups (user_id + group_id: composite PK)   -- many-to-many
checkins (id: auto PK, user_id, group_id, note?, checkin_at)
monthly_kings (id: auto PK, group_id, user_id, year, month, count)
```

## Backend

```
backend/
├── src/
│   ├── app.js              # Koa entry
│   ├── config/             # DB path, port, etc.
│   ├── models/             # Prisma schema
│   ├── controllers/        # Request handlers
│   ├── services/           # Business logic
│   ├── middlewares/        # error handling, etc.
│   ├── routes/             # Route definitions
│   └── jobs/               # node-cron: monthly king settlement
├── data/                   # SQLite file lives here
└── prisma/schema.prisma
```

Common backend commands:

```bash
cd backend
npm run dev          # start dev server (nodemon)
npx prisma generate  # regenerate Prisma client after schema change
npx prisma migrate dev --name <name>  # create migration
npx prisma studio    # visual DB browser
```

## Frontend (uni-app)

```bash
cd frontend
npm run dev:mp-weixin   # dev with Weixin mini-program target
npm run build:mp-weixin # build for Weixin
npm run dev:h5           # dev with H5 target (browser)
npm run build:h5         # build H5 static files
```

API calls use `uni.request`, NOT `fetch`/`axios` — this is critical for WeChat mini-program compatibility.

## API conventions

All endpoints return `{ code: 0, message: "success", data: {} }`. No auth tokens. Every request that needs identity passes `user_id` in the body or query string.

Key endpoints:
- `POST /api/user` — register or retrieve user by UUID
- `POST /api/group`, `POST /api/group/join` — create/join squad
- `POST /api/checkin` — record a check-in to a squad
- `GET /api/rank/monthly`, `GET /api/rank/kings` — leaderboards
