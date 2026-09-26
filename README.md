# AssistWalk

**Smart mobility assistance platform for visually impaired people**

AssistWalk combines computer vision, text recognition, and real-time communication to
help visually impaired people move around independently, while keeping their
companions informed whenever needed.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [AI Model — YOLOv8 Fine-tuning](#ai-model--yolov8-fine-tuning)
- [Prerequisites](#prerequisites)
- [Quick Start with Docker](#quick-start-with-docker)
- [Local Development Setup](#local-development-setup)
- [Environment Variables](#environment-variables)
- [Project Structure](#project-structure)
- [API Documentation](#api-documentation)
- [License](#license)
- [Authors](#authors)

---

## Project Overview

Visually impaired people face daily challenges detecting obstacles, reading documents,
or quickly alerting a relative in case of danger. AssistWalk addresses these
challenges through three complementary interfaces:

- a **mobile app** (Flutter) for the visually impaired user: real-time obstacle
  detection, text reading (OCR), and SOS alert triggering;
- a **web dashboard** (React) for companions: real-time map tracking and alert
  reception;
- an **admin panel** (React): user management, associations, and support tickets.

---

## Key Features

| Feature | Description |
|---|---|
| Obstacle detection | Fine-tuned YOLOv8 model, running in real time on the camera feed |
| Text recognition (OCR) | Reads documents and signs via Tesseract, EasyOCR, and Groq Vision API |
| SOS alerts | Instant notification to the companion via WebSocket, FCM push, and email |
| Real-time tracking | Geolocation displayed on a map on the companion's side |
| Account management | JWT authentication, roles (visually impaired / companion / admin), temporary password on account creation |

---

## Architecture

AssistWalk is built on a containerized microservices architecture, all exposed
behind a single Nginx gateway:

```
                         ┌────────────────────────────────────────┐
   Mobile / Browser ────▶│        Nginx Gateway (port 80)          │
                         └───────────────┬──────────────────────────┘
                                         │
        ┌───────────────┬───────────────┼───────────────┐
        │               │               │               │
   ┌──────────┐   ┌─────────────┐  ┌─────────────┐  ┌───────────┐
   │   web    │   │   backend   │  │  navigation  │  │    ocr    │
   │ (React)  │   │(Spring Boot)│  │(Flask+YOLOv8)│  │ (FastAPI) │
   └──────────┘   └──────┬──────┘  └─────────────┘  └─────┬─────┘
                          │                                 │
                    ┌─────▼─────┐                    ┌──────▼──────┐
                    │ PostgreSQL │                    │ Groq Vision │
                    └───────────┘                    │   API (ext) │
                                                       └─────────────┘
```

The Nginx gateway is the system's **single entry point**: it routes each request to
the right microservice based on the path prefix (`/api`, `/auth`, `/navigation`,
`/ocr`, `/ws`), which keeps client-side configuration (mobile and web) simple since
they only need to know one address.

| Service | Technology | Role |
|---|---|---|
| `gateway` | Nginx | Reverse proxy, single entry point |
| `web` | React + Vite | Companion / admin dashboard |
| `backend` | Spring Boot | Core API: auth, users, alerts, WebSocket |
| `backend-navigation` | Flask + YOLOv8 | Real-time obstacle detection |
| `ocr-service` | FastAPI | Text recognition (OCR) |
| `postgres` | PostgreSQL 16 | Relational database |

---

## Tech Stack

**Backend** — Spring Boot, PostgreSQL, JWT, WebSocket (STOMP)
**AI** — YOLOv8 (Ultralytics), Flask, OpenCV, FastAPI, Tesseract OCR, EasyOCR, Groq Vision API
**Web frontend** — React, Vite
**Mobile** — Flutter
**Infrastructure** — Docker, Docker Compose, Nginx
**Tools** — Git/GitHub, Postman, draw.io

---

## AI Model — YOLOv8 Fine-tuning

The obstacle detection model used by the `backend-navigation` service is a
**fine-tuned YOLOv8**, trained on a custom dataset targeting obstacle classes
relevant to visually impaired mobility, including:

- **stairs**
- **doors**
- additional urban obstacles

Fine-tuning significantly improved detection accuracy on these specific classes
compared to the base pre-trained YOLOv8 model, better adapting it to real-world
usage constraints (low viewing angles, variable lighting conditions, indoor and
outdoor environments).

---

## Prerequisites

- [Docker](https://www.docker.com/) and Docker Compose
- [Flutter](https://flutter.dev/) 3.x (for the mobile app)
- Java 17 and Python 3.11 (only required for local development without Docker)

---

## Quick Start with Docker

The entire stack (database, backend, AI microservices, web frontend, and gateway)
is containerized and runs with a single command.

### 1. Configure environment variables

```bash
cp .env.example .env
cp ocr/.env.example ocr/.env
```

Edit `.env` and `ocr/.env` to fill in your credentials (database, JWT secret, Groq
API key, etc.).

### 2. Start the full stack

```bash
cd docker
docker compose up -d --build
```

### 3. Verify

```bash
docker compose ps
```

All containers should show a `running` status (and `healthy` for `postgres` and
`gateway`).

### Accessing the services

| Service | URL |
|---|---|
| Web app | http://localhost (via the gateway) |
| Backend API | http://localhost/api |
| OCR service | http://localhost/ocr |
| Navigation service | http://localhost/navigation |

> Services are also exposed directly on their respective ports (bypassing the
> gateway): `web` (3000), `backend` (8081), `ocr-service` (8000),
> `backend-navigation` (5001), `postgres` (5432).

### Stopping the stack

```bash
docker compose down
```

To also remove volumes (full database reset):

```bash
docker compose down -v
```

### Running the mobile app

```bash
cd mobile
flutter pub get
flutter run --dart-define=GATEWAY_HOST=<YOUR_MACHINE_IP>
```

Replace `<YOUR_MACHINE_IP>` with the local IP address of the machine running
`docker compose` (the phone and the computer must be on the same network).

---

## Local Development Setup

For active backend development without rebuilding the Docker image on every change:

```bash
# 1. Environment variables
cp .env.example .env

# 2. Start only the database and AI microservices
cd docker
docker compose up -d postgres ocr-service backend-navigation

# 3. Run the backend locally
cd ../backend
./mvnw spring-boot:run

# 4. Run the web frontend locally
cd ../web
npm install
npm run dev

# 5. Run the mobile app
cd ../mobile
flutter run
```

---

## Environment Variables

| Variable | Description | Example |
|---|---|---|
| `POSTGRES_DB` | Database name | `assistwalk` |
| `POSTGRES_USER` | PostgreSQL user | `assistwalk_user` |
| `POSTGRES_PASSWORD` | PostgreSQL password | — |
| `JWT_SECRET` | JWT signing secret key | — |
| `JWT_EXPIRATION_MS` | Token validity duration (regular session) | `28800000` (8h) |
| `JWT_EXPIRATION_REMEMBER_MS` | Token validity duration ("remember me") | `2592000000` (30d) |
| `FRONTEND_URL` | Frontend URL, used for links in emails | `http://localhost` |
| `OCR_SERVICE_URL` | Internal OCR service URL (auto-resolved in Docker) | `http://ocr-service:8000` |

Full details of the variables required by the OCR service (Groq API key, etc.) are
listed in `ocr/.env.example`.

---

## Project Structure

```
assistwalk/
├── backend/                # Core API — Spring Boot
├── backend-navigation/     # Obstacle detection — Flask + YOLOv8
├── ocr/                    # Text recognition — FastAPI
├── web/                    # Companion / admin dashboard — React
├── mobile/                 # Mobile app — Flutter
├── docker/
│   ├── docker-compose.yml
│   └── gateway/             # Nginx reverse proxy configuration
├── docs/
│   └── api.md
├── .env.example
└── README.md
```

---

## API Documentation

Detailed API endpoint documentation is available in [docs/api.md](docs/api.md).

---

## License

Project developed as part of a final-year project (PFA) at ENSIAS.

---

## Authors

This project was developed by **Fatiha Khassil** and **Oumaima Lahkiar**, students
at **ENSIAS**.

- Supervised by: **Mrs. Widad Elouataoui**
