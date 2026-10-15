# 🩺 PCOS Risk Assessment Platform

A full-stack web app that helps doctors manage their patients and run **PCOS (Polycystic Ovary Syndrome) risk assessments**.
A **Go** backend handles auth, patients and analysis history, and calls a separate **Python ML service** that serves a trained scikit-learn model.

![Go](https://img.shields.io/badge/Go-00ADD8?style=flat-square&logo=go&logoColor=white)
![Gin](https://img.shields.io/badge/Gin-008ECF?style=flat-square&logo=go&logoColor=white)
![GORM](https://img.shields.io/badge/GORM-00ADD8?style=flat-square&logo=go&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-F7931E?style=flat-square&logo=scikitlearn&logoColor=white)
![Next.js](https://img.shields.io/badge/Next.js-000000?style=flat-square&logo=nextdotjs&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)

---

## ✨ Features

- **Doctor accounts** — sign up / log in with **bcrypt**-hashed passwords and **JWT** (24h expiry)
- **Auth middleware** — every patient & analysis route is protected by a Gin JWT middleware
- **Patient management** — create and list patients, scoped to the logged-in doctor
- **PCOS risk prediction** — 13 clinical/lifestyle inputs are sent to the ML service, which returns a probability and a `high` / `low` risk label
- **Analysis history** — every prediction is stored in PostgreSQL with the raw inputs as `JSONB`
- **Resilient service-to-service calls** — HTTP client with timeout and **retry with backoff** when the ML service is cold-starting
- **DB connection retries** on startup, so the backend waits for Postgres inside Docker

---

## 🏗️ Architecture

```
┌──────────────┐   REST + JWT   ┌──────────────────┐   HTTP /predict   ┌──────────────────┐
│  Next.js UI  │ ─────────────▶ │  Go backend      │ ────────────────▶ │  FastAPI ML      │
│  (port 3000) │                │  Gin + GORM      │                   │  service (8005)  │
└──────────────┘                │  (port 8080)     │                   │  scikit-learn    │
                                └────────┬─────────┘                   └──────────────────┘
                                         │ GORM
                                         ▼
                                ┌──────────────────┐
                                │  PostgreSQL 16   │
                                └──────────────────┘
```

### Backend layout (Go)

Each domain follows the same `routes → controller → process (validate + service) → response` structure:

```
backend/
├── main.go              # Gin setup, CORS, route mounting, auto-migrations
├── database/            # DB connection (with retries) + GORM models
│   └── model/           # User, Patient, PcosAnalysis
├── login/               # sign-up, login, email check, bcrypt utils
├── logout/
├── patient/             # list / create patients, run analysis
├── middleware/          # JWT generation + auth middleware
└── response/            # common API response format
```

---

## 🔌 API

Base URL: `http://localhost:8080`

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET`  | `/pcos/v1/verifyEmail` | – | Check whether an email is already registered |
| `POST` | `/pcos/v1/signUp/` | – | Register a doctor |
| `POST` | `/pcos/v1/login/` | – | Log in, returns a JWT |
| `GET`  | `/pcos/patient/list/` | JWT | List the doctor's patients |
| `POST` | `/pcos/patient/create/` | JWT | Add a patient |
| `POST` | `/pcos/patient/analysis/` | JWT | Run a PCOS risk prediction and save the result |
| `GET`  | `/pcos/v1/logout/` | JWT | Log out |

Protected routes expect the token in the `Authorization` header.

**ML service** (`http://localhost:8005`)

| Method | Endpoint | Description |
|---|---|---|
| `GET` / `HEAD` | `/health` | Health check |
| `POST` | `/predict` | Returns `{ "probability": 0.82, "risk_label": "high" }` |

---

## 🚀 Running locally

### With Docker (recommended)

```bash
git clone https://github.com/arushi-sh/PCOS-App.git
cd PCOS-App
docker compose up --build
```

| Service | URL |
|---|---|
| Frontend | http://localhost:3000 |
| Backend | http://localhost:8080 |
| ML service | http://localhost:8005 |
| PostgreSQL | localhost:5555 |

### Without Docker

```bash
# 1. Backend
cd backend
cp .env.example .env      # fill in your DB credentials
go run main.go

# 2. ML service
cd ml-service
pip install -r requirements.txt
uvicorn src.app:app --port 8005

# 3. Frontend
cd frontend
npm install
npm run dev
```

---

## 🧰 Tech stack

| Layer | Tech |
|---|---|
| Backend | Go 1.24, Gin, GORM, golang-jwt, bcrypt |
| ML service | Python, FastAPI, scikit-learn, pandas, joblib |
| Database | PostgreSQL 16 |
| Frontend | Next.js, React, TypeScript, Tailwind CSS |
| DevOps | Docker, Docker Compose |
