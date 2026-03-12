# NihonGo! — Japanese Language Learning SaaS

## Context
Build a modern, Gen-Z targeted Japanese language learning SaaS for Indian users. Replaces the old Anki-clone plan with a full-featured platform combining Anki-grade SRS with Duolingo-grade UX, gamification, community features, and pop-culture appeal (anime/manga). MVP: Japanese only, free tier with auth, pre-built + user-created content.

## Tech Stack
- **Backend**: FastAPI + aiosqlite (raw SQL, no ORM) + pykakasi
- **Frontend**: Vite + React (TypeScript) + Tailwind CSS v4
- **Auth**: JWT (email/password + Google OAuth)
- **TTS**: Browser Web Speech API (ja-JP)
- **Key deps**: fastapi, uvicorn, aiosqlite, pykakasi, python-jose, passlib, httpx, pydantic-settings

## Existing Code to Reuse
- `srs/sm2.py` — Working SM-2 algorithm (port dataclasses → Pydantic)
- `srs/models.py` — CardSchedule, ReviewResult (convert to Pydantic)
- `db/schema.sql` — Base schema patterns (expand with users, gamification)
- `db/queries.py` — SQL query patterns (rewrite as async)

## Project Structure
```
anki-language-learning/
├── backend/
│   ├── main.py                 # FastAPI app + CORS + startup
│   ├── config.py               # pydantic-settings BaseSettings
│   ├── requirements.txt
│   ├── .env.example
│   ├── db/
│   │   ├── connection.py       # Async aiosqlite manager
│   │   ├── schema.sql          # Full DDL (8 tables)
│   │   ├── seed.sql            # Pre-built decks + achievements
│   │   └── queries/
│   │       ├── users.py, decks.py, cards.py, reviews.py, achievements.py
│   ├── auth/
│   │   ├── jwt.py, oauth.py, passwords.py, dependencies.py
│   ├── srs/
│   │   ├── sm2.py, models.py, scheduler.py
│   ├── routers/
│   │   ├── auth.py, decks.py, cards.py, study.py
│   │   ├── import_export.py, progress.py, community.py
│   ├── importers/
│   │   ├── apkg.py, csv_io.py
│   ├── japanese/
│   │   └── furigana.py         # pykakasi → <ruby> HTML
│   └── gamification/
│       ├── xp.py, streaks.py, achievements.py
├── frontend/
│   ├── package.json, vite.config.ts, tsconfig.json
│   └── src/
│       ├── main.tsx, App.tsx
│       ├── api/                # client.ts, auth.ts, decks.ts, cards.ts, study.ts, progress.ts
│       ├── components/         # Layout, StudyCard, RatingButtons, XPBar, StreakBadge, etc.
│       ├── pages/              # Dashboard, DeckDetail, Study, CardForm, Import, Progress, Community, Login, Register
│       ├── hooks/              # useAuth, useTTS, useKeyboard
│       ├── context/            # AuthContext
│       ├── styles/             # globals.css, study.css
│       └── utils/              # furigana.ts
```

## Database Schema (8 tables)
- **users**: id, email, password_hash, display_name, avatar_url, google_id, xp, streak_count, streak_last_date, current_level, created_at
- **decks**: id, creator_id (FK users), name, description, is_public, language, jlpt_level, card_count, timestamps
- **cards**: id, deck_id (FK decks), front, back, front_furigana, back_furigana, notes, source, timestamps — NO SRS state (moved to user_card_progress)
- **user_card_progress**: id, user_id, card_id, easiness_factor, interval, repetitions, next_review, last_review — UNIQUE(user_id, card_id). Created lazily at study time.
- **review_log**: id, user_id, card_id, quality, easiness_factor, interval, reviewed_at
- **achievements**: id, name, description, icon, xp_reward, condition_type, condition_value
- **user_achievements**: user_id, achievement_id, earned_at — PK(user_id, achievement_id)
- **user_deck_subscriptions**: user_id, deck_id, subscribed_at — PK(user_id, deck_id)

Key design: SRS state is per-user (user_card_progress), not per-card. This enables community deck sharing where each user tracks independent progress on the same cards.

## API Endpoints
| Method | Route | Purpose |
|--------|-------|---------|
| POST | /api/auth/register | Create account, return JWT |
| POST | /api/auth/login | Email/password login |
| POST | /api/auth/google | Google OAuth login |
| GET | /api/auth/me | Current user profile |
| GET/POST | /api/decks | List user's decks / create deck |
| GET/PUT/DELETE | /api/decks/{id} | Deck detail / update / delete |
| GET/POST | /api/decks/{id}/cards | List cards / create card (auto-furigana) |
| GET/PUT/DELETE | /api/cards/{id} | Card detail / update / delete |
| GET | /api/decks/{id}/study | Get study queue (due + new cards, max 20) |
| POST | /api/review | Submit rating → update SRS + XP + streak + achievements |
| GET | /api/progress | Dashboard stats (XP, streak, due, accuracy, JLPT progress) |
| GET | /api/leaderboard | Top 50 users by XP (weekly/alltime) |
| GET | /api/achievements | All achievements with earned status |
| GET | /api/community/decks | Browse public decks (search, filter, sort) |
| POST/DELETE | /api/community/decks/{id}/subscribe | Subscribe/unsubscribe to public deck |
| POST | /api/import/apkg | Upload .apkg file |
| POST | /api/import/csv | Upload CSV file |
| GET | /api/decks/{id}/export/csv | Download deck as CSV |

## Gamification
- **XP**: +10/review, +5 bonus for Easy, +25 for session complete
- **Levels**: Beginner(0) → Apprentice(100) → Scholar(500) → Sage(2000) → Sensei(5000) → Master(10000)
- **Streaks**: Consecutive days with ≥1 review. Day boundary = midnight IST (India target)
- **Achievements**: 9 seeded (First Steps, Dedicated Learner, Review Machine, Week Warrior, Month Master, Card Collector, Centurion, Deck Creator, Deck Master)
- **Leaderboard**: Weekly + all-time XP rankings

## Pre-built Content (seeded on first run)
1. **Hiragana & Katakana** (~96 cards, JLPT N5)
2. **JLPT N5 Vocabulary** (~100 words)
3. **Common Anime Phrases** (~50 phrases)
New users auto-subscribed to all 3 decks on registration.

## UI/UX Design
- **Dark mode default** — purple/indigo gradient palette, cyan/gold accents
- **Mobile-first** — bottom tab bar (mobile), left sidebar (desktop)
- **Card flip** — CSS 3D perspective transform (0.4s)
- **XP popup** — floating "+10 XP" animation on review
- **Skeleton loading** — no spinners
- **Keyboard shortcuts** — Space=flip, 1-4=ratings
- **TTS** — Web Speech API, ja-JP, rate=0.8

## Implementation Phases

### Phase 1: Scaffold
- Vite + React + Tailwind setup with proxy to FastAPI
- FastAPI app with CORS, config, directory structure
- Both running with hello-world endpoints

### Phase 2: Database + Auth
- Full schema (8 tables) + async aiosqlite connection manager
- JWT auth (register/login/google) + dependencies
- Login/Register React pages + AuthContext

### Phase 3: Decks + Cards + Furigana
- Deck/Card CRUD routers (async, user-scoped)
- pykakasi furigana generation on card create/update
- Dashboard, DeckDetail, CardForm React pages

### Phase 4: SM-2 + Study Session
- Port SM-2 to Pydantic models, async scheduler with user_card_progress
- Study queue endpoint (due + new cards via LEFT JOIN)
- Study page: card flip, rating buttons, keyboard shortcuts, TTS
- Review endpoint: update SRS + award XP

### Phase 5: Gamification
- XP, levels, streaks (IST timezone), achievement checking
- Progress page, leaderboard, achievement grid
- XP popup animation, streak badge

### Phase 6: Import/Export
- .apkg parser (ZIP → SQLite → cards), CSV import/export
- Upload UI with drag-and-drop

### Phase 7: Community
- Public deck browsing, search/filter, subscribe/unsubscribe
- Community page

### Phase 8: Seed Content
- seed.sql with system user, 3 decks, ~246 cards, 9 achievements
- Auto-subscribe new users

### Phase 9: Polish
- Mobile responsive layout, dark/light toggle
- Micro-animations, skeleton loading, error toasts, empty states

### Phase 10: Tests
- pytest + pytest-asyncio for SM-2, auth, CRUD, study, import, gamification

## Verification
- `cd backend && pip install -r requirements.txt && uvicorn main:app` — runs on :8000
- `cd frontend && npm install && npm run dev` — runs on :5173, proxies /api to :8000
- Register → land on dashboard with 3 pre-built decks → study Hiragana
- Create deck → add Japanese card → see furigana auto-generated
- Study session → rate cards → see XP popup, streak update, intervals change
- Import .apkg → cards appear with furigana → study them
- Check leaderboard → see your ranking
- Browse community decks → subscribe → study
