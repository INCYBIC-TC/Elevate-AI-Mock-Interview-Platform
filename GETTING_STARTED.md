# 🚀 Elevate – Getting Started Guide (For Juniors)

> Companion doc to the main `Readme.md`. This file focuses on **exact API endpoints** and a **step-by-step build order** so you can go from an empty folder to a working MERN app without guessing.

---

# 1. Prerequisites

Install these before touching the code:

| Tool | Check version | Notes |
| --- | --- | --- |
| Node.js (LTS) | `node -v` | v18+ recommended |
| npm | `npm -v` | comes with Node |
| Git | `git -v` | for cloning + version control |
| MongoDB Atlas account | — | free tier is enough |
| VS Code | — | + ESLint, Prettier extensions |
| Postman | — | to test API endpoints before wiring up the frontend |

---

# 2. Project Setup (Day 1)

```bash
# 1. Clone the repo
git clone https://github.com/INCYBIC-TC/Elevate-AI-Mock-Interview-Platform.git
cd Elevate-AI-Mock-Interview-Platform

# 2. Create the two app folders (per the recommended structure)
mkdir backend frontend

# 3. Set up backend
cd backend
npm init -y
npm install express mongoose dotenv cors bcryptjs jsonwebtoken
npm install --save-dev nodemon

# 4. Set up frontend (Vite + React)
cd ../frontend
npm create vite@latest . -- --template react
npm install
npm install axios react-router-dom zustand tailwindcss
```

Add `.gitignore` in both `backend/` and `frontend/`:
```
node_modules/
.env
dist/
```

Add scripts to `backend/package.json`:
```json
"scripts": {
  "dev": "nodemon server.js",
  "start": "node server.js"
}
```

---

# 3. Environment Variables

**`backend/.env`**
```env
PORT=5000
MONGO_URI=your_mongodb_atlas_connection_string
JWT_SECRET=some_long_random_string
AI_API_KEY=your_gemini_or_openai_key
NODE_ENV=development
```

**`frontend/.env`**
```env
VITE_API_BASE_URL=http://localhost:5000/api
```

⚠️ Never commit `.env` files. Each junior should create their own MongoDB Atlas free cluster and Gemini/OpenAI API key for local dev.

---

# 4. Folder Structure to Create

```
backend/
├── config/
│   └── db.js                # mongoose connection
├── controllers/
│   ├── authController.js
│   ├── sessionController.js
│   └── evaluationController.js
├── middleware/
│   ├── authMiddleware.js     # verifies JWT
│   └── errorMiddleware.js
├── models/
│   ├── User.js
│   ├── InterviewSession.js
│   └── QuestionTurn.js
├── routes/
│   ├── authRoutes.js
│   ├── sessionRoutes.js
│   └── evaluationRoutes.js
├── services/
│   └── aiService.js          # calls Gemini/OpenAI
├── utils/
│   └── generateToken.js
└── server.js

frontend/src/
├── api/
│   └── axiosClient.js
├── pages/
│   ├── Login.jsx
│   ├── Register.jsx
│   ├── Dashboard.jsx
│   └── InterviewRoom.jsx
├── components/
├── hooks/
├── store/
│   └── authStore.js          # zustand
├── layouts/
└── App.jsx
```

---

# 5. Full API Specification

All routes are prefixed with `/api`. Protected routes require:
```
Authorization: Bearer <jwt_token>
```

## 5.1 Auth Routes — `/api/auth`

### Register
```
POST /api/auth/register
```
**Body**
```json
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "SecurePass123",
  "targetRole": "Frontend Developer"
}
```
**Response `201`**
```json
{
  "user": {
    "id": "665f1c2e...",
    "username": "john_doe",
    "email": "john@example.com",
    "targetRole": "Frontend Developer"
  },
  "token": "eyJhbGciOi..."
}
```
**Errors:** `400` missing fields · `409` email already exists

### Login
```
POST /api/auth/login
```
**Body**
```json
{ "email": "john@example.com", "password": "SecurePass123" }
```
**Response `200`** — same shape as register response
**Errors:** `401` invalid credentials

### Get Current User
```
GET /api/auth/me
```
🔒 Protected. Returns the logged-in user's profile (no password hash).

---

## 5.2 Interview Session Routes — `/api/sessions`

### Create Session
```
POST /api/sessions
```
🔒 Protected
**Body**
```json
{
  "role": "Frontend Developer",
  "experienceLevel": "Fresher",
  "techStack": ["React", "JavaScript", "CSS"],
  "interviewType": "Technical"
}
```
**Response `201`**
```json
{
  "sessionId": "665f2b90...",
  "role": "Frontend Developer",
  "status": "in-progress",
  "createdAt": "2026-07-27T10:00:00Z",
  "firstQuestion": "Tell me about yourself."
}
```

### Get All Sessions (history)
```
GET /api/sessions?page=1&limit=10
```
🔒 Protected. Supports pagination for the dashboard history list.
**Response `200`**
```json
{
  "sessions": [
    { "sessionId": "665f2b90...", "role": "Frontend Developer", "overallScore": 8, "status": "completed", "createdAt": "..." }
  ],
  "totalPages": 3,
  "currentPage": 1
}
```

### Get Single Session Detail
```
GET /api/sessions/:id
```
🔒 Protected. Returns session metadata + all question turns for that session (used on the "review" page).

### Delete a Session (optional, nice-to-have)
```
DELETE /api/sessions/:id
```
🔒 Protected.

---

## 5.3 Evaluation Routes — `/api/evaluation`

### Submit an Answer / Get Next Question
```
POST /api/evaluation/submit-turn
```
🔒 Protected
**Body**
```json
{
  "sessionId": "665f2b90...",
  "questionAsked": "Tell me about yourself.",
  "userAnswerTranscript": "I'm a frontend developer with 6 months of React experience..."
}
```
**What happens internally:**
1. Middleware verifies the JWT and loads the session.
2. Controller builds a prompt with: current question, candidate's answer, and (optionally) prior Q&A turns for context.
3. `aiService.js` calls Gemini/OpenAI and forces JSON-only output.
4. Result is saved as a new `QuestionTurn` document.
5. Response is sent back to the frontend.

**Response `200`**
```json
{
  "score": 8,
  "strengths": ["Clear communication", "Good project explanation"],
  "weaknesses": ["Needs more technical depth"],
  "nextQuestion": "Explain how REST APIs work."
}
```
**Errors:** `400` missing sessionId/answer · `502` AI provider failed (should fall back gracefully, e.g. return a generic next question)

### Complete a Session
```
POST /api/evaluation/complete/:sessionId
```
🔒 Protected. Marks session as `completed`, computes `overallScore` (average of all turn scores), sets `completedAt`.
**Response `200`**
```json
{ "sessionId": "665f2b90...", "status": "completed", "overallScore": 7.5 }
```

---

## 5.4 Analytics Routes — `/api/analytics` (Phase 5)

### Get Dashboard Summary
```
GET /api/analytics/summary
```
🔒 Protected. Returns aggregate stats for the logged-in user (used with Recharts on the dashboard).
**Response `200`**
```json
{
  "totalInterviews": 12,
  "averageScore": 7.2,
  "communicationScore": 8,
  "technicalScore": 6.5,
  "confidenceScore": 7,
  "scoreTrend": [
    { "date": "2026-07-01", "score": 6 },
    { "date": "2026-07-15", "score": 7.5 }
  ]
}
```

---

## 5.5 Health Check (good practice)
```
GET /api/health
```
No auth. Returns `{ "status": "ok" }`. Useful once you deploy, so Render/Railway's uptime monitor has something to ping.

---

# 6. Step-by-Step Build Order (map to Phases in main README)

Work through these in order — don't skip ahead to the AI features before auth + CRUD are solid. Each junior should be able to demo the checkbox items at the end of their week.

### ✅ Step 1 — Backend Skeleton
- [ ] `server.js` — Express app, connect MongoDB via `config/db.js`, mount routes
- [ ] `GET /api/health` returns 200
- [ ] Test with Postman that the server boots and connects to Atlas

### ✅ Step 2 — Auth
- [ ] `User` model (username, email, passwordHash, targetRole)
- [ ] `POST /api/auth/register` — hash password with bcrypt, save user, return JWT
- [ ] `POST /api/auth/login` — compare password, return JWT
- [ ] `authMiddleware.js` — verifies `Authorization: Bearer <token>` header
- [ ] `GET /api/auth/me` protected route works in Postman

### ✅ Step 3 — Frontend Auth Pages
- [ ] Register + Login forms (React + Tailwind)
- [ ] Store JWT in Zustand store (in-memory + persisted safely, not localStorage inside artifacts — but real apps can use localStorage/cookies)
- [ ] Protected route wrapper (`<ProtectedRoute>`) that redirects to `/login` if no token

### ✅ Step 4 — Interview Sessions CRUD
- [ ] `InterviewSession` model
- [ ] `POST /api/sessions`, `GET /api/sessions`, `GET /api/sessions/:id`
- [ ] Dashboard page lists past sessions (call `GET /api/sessions`)
- [ ] "Start New Interview" button calls `POST /api/sessions`

### ✅ Step 5 — AI Evaluation Engine
- [ ] `QuestionTurn` model
- [ ] `aiService.js` — builds the prompt, calls Gemini/OpenAI, forces JSON output
- [ ] `POST /api/evaluation/submit-turn` — saves turn, returns feedback + next question
- [ ] `POST /api/evaluation/complete/:sessionId` — computes overall score
- [ ] Interview Room page: shows question → text input → submit → shows feedback → next question

### ✅ Step 6 — Analytics & Polish
- [ ] `GET /api/analytics/summary` — aggregation pipeline over `QuestionTurn`/`InterviewSession`
- [ ] Recharts radar/bar chart on dashboard
- [ ] Framer Motion page transitions

### ✅ Step 7 — Deployment
- [ ] Backend → Render/Railway (set env vars in dashboard, not committed)
- [ ] Frontend → Vercel (set `VITE_API_BASE_URL` to deployed backend URL)
- [ ] Update CORS on backend to allow the deployed frontend origin only

---

# 7. Testing Checklist Before Calling a Feature "Done"

- [ ] Tested the endpoint in Postman with valid input
- [ ] Tested with missing/invalid input (does it return a proper error, not a crash?)
- [ ] Tested protected routes without a token (should return `401`)
- [ ] Frontend shows a loading state while waiting for the API
- [ ] Frontend shows a friendly error message on failure

---

# 8. Suggested Git Workflow for the Team

```bash
main            # stable, deployable
 └── dev        # integration branch
      ├── feature/auth
      ├── feature/sessions
      ├── feature/ai-evaluation
      └── feature/analytics-dashboard
```

- One feature branch per person/task, PR into `dev`
- Require at least one reviewer per PR (great habit even in a learning project)
- Merge `dev` → `main` only after a feature set is tested end-to-end

---

Refer back to the main `Readme.md` for architecture diagrams, schema definitions, and the full learning resource list.
