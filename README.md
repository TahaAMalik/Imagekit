# ImageSite

A full-stack social media application for sharing images and videos. Built with a **FastAPI** backend, **Streamlit** frontend, **SQLite** database, and **ImageKit** for media storage and delivery.

---

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Code Components](#code-components)
- [Key Use Cases](#key-use-cases)
- [Application Flow](#application-flow)
- [Running Locally](#running-locally)

---

## Overview

ImageSite lets users register, log in, upload images or videos with captions, browse a shared feed, and delete their own posts. Media files are stored and served via ImageKit's CDN, while user and post metadata are persisted in a local SQLite database.

---

## Project Structure

```
ImageSite/
├── app/
│   ├── app.py          # FastAPI application — routes and request handling
│   ├── db.py           # Database models and session management
│   ├── images.py       # ImageKit client initialization
│   ├── schemas.py      # Pydantic schemas for request/response validation
│   └── users.py        # Authentication logic using FastAPI Users
├── frontend.py         # Streamlit frontend — all UI pages
├── main.py             # Uvicorn entry point
├── .env                # Environment variables (not committed)
└── requirements.txt    # Python dependencies
```

---

## Code Components

### `app/app.py` — API Routes

The core FastAPI application. Defines all HTTP endpoints and wires together authentication, database, and ImageKit dependencies.

| Endpoint | Method | Description |
|---|---|---|
| `/auth/jwt/login` | POST | Log in and receive a JWT token |
| `/auth/register` | POST | Register a new user account |
| `/upload` | POST | Upload an image or video with an optional caption |
| `/feed` | GET | Retrieve all posts in reverse-chronological order |
| `/posts/{post_id}` | DELETE | Delete a post (owner only) |

### `app/db.py` — Database Models

Uses **SQLAlchemy** with an async SQLite connection (`aiosqlite`). Defines two models:

- **`User`** — Extends `SQLAlchemyBaseUserTableUUID` from FastAPI Users. Stores authentication credentials and links to posts.
- **`Post`** — Stores a post's UUID, owner user ID, caption, ImageKit URL, file type (`image` or `video`), file name, and creation timestamp.

### `app/images.py` — ImageKit Client

Initializes the `AsyncImageKit` client using the private key from environment variables. Also exposes the `url_endpoint` used for constructing transformed media URLs on the frontend.

### `app/schemas.py` — Pydantic Schemas

Defines the data shapes for API requests and responses:

- `PostCreate` / `PostResponse` — Post data structures
- `UserRead` / `UserCreate` / `UserUpdate` — User data structures, extending FastAPI Users' base schemas

### `app/users.py` — Authentication

Configures **FastAPI Users** with:

- JWT authentication via `BearerTransport` and `JWTStrategy`
- A `UserManager` handling registration events and password resets
- Exports `current_active_user` dependency used to protect routes

### `frontend.py` — Streamlit UI

A single-file frontend with three pages:

- **Login / Sign Up page** — Handles both registration and login using the API. Stores the JWT token and user info in Streamlit session state.
- **Feed page** — Fetches and renders all posts. Displays images with optional caption overlays using ImageKit URL transformations. Owners see a delete button on their own posts.
- **Upload page** — File uploader that sends the selected media and caption to the `/upload` endpoint.

### `main.py` — Entry Point

Starts the Uvicorn ASGI server programmatically, running `app.app:app` on port `8000` with hot reload enabled.

---

## Key Use Cases

**User Registration and Login**
A new visitor registers with an email and password. On success, they can immediately log in to receive a JWT token, which is stored in the session and attached to all subsequent API requests as a Bearer token.

**Uploading Media**
An authenticated user selects an image or video from their device, optionally adds a caption, and submits. The backend writes the file to a temporary location, uploads it to ImageKit, then stores the resulting URL and metadata in the database as a new Post.

**Browsing the Feed**
Any authenticated user can view the feed, which shows all posts from all users sorted newest first. Each post displays the author's email, the upload date, and the media rendered via ImageKit's CDN. Caption text is overlaid directly onto images using ImageKit's text transformation API.

**Deleting a Post**
A user can delete any post they own. The delete button is only rendered on the frontend for posts where `is_owner` is true, and ownership is also enforced on the backend before the record is removed.

---

## Application Flow

```
User (Browser)
     │
     ▼
┌─────────────┐        JWT Token         ┌──────────────────┐
│  Streamlit  │ ◄─────────────────────── │   FastAPI        │
│  frontend   │ ──── HTTP Requests ────► │   Backend        │
│  :8501      │                          │   :8000          │
└─────────────┘                          └────────┬─────────┘
                                                  │
                          ┌───────────────────────┤
                          │                       │
                          ▼                       ▼
                 ┌─────────────────┐    ┌──────────────────┐
                 │  SQLite DB      │    │  ImageKit CDN    │
                 │  (users, posts) │    │  (media storage) │
                 └─────────────────┘    └──────────────────┘
```

**Upload sequence:**

1. User selects a file in Streamlit and clicks **Share**
2. Streamlit POSTs the file to `/upload` with the JWT token in the `Authorization` header
3. FastAPI validates the token and identifies the current user
4. The file is written to a temporary file on disk
5. The temp file is read into memory and uploaded to ImageKit via the SDK
6. ImageKit returns a permanent CDN URL
7. FastAPI saves a new `Post` record (URL, file type, caption, user ID) to SQLite
8. The temp file is deleted
9. Streamlit reruns and the new post appears in the feed

---

## Running Locally

### Prerequisites

- Python 3.10+
- An [ImageKit](https://imagekit.io) account (free tier works)

### 1. Clone the repository

```bash
git clone https://github.com/your-username/ImageSite.git
cd ImageSite
```

### 2. Create and activate a virtual environment

```bash
python -m venv .venv

# Windows
.venv\Scripts\Activate.ps1

# macOS / Linux
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install fastapi uvicorn sqlalchemy aiosqlite fastapi-users[sqlalchemy] \
            imagekitio streamlit requests python-dotenv python-multipart
```

### 4. Configure environment variables

Create a `.env` file in the project root:

```env
IMAGEKIT_PRIVATE_KEY=your_private_key_here
IMAGEKIT_PUBLIC_KEY=your_public_key_here
IMAGEKIT_URL_ENDPOINT=https://ik.imagekit.io/your_imagekit_id
```

You can find these values in the **Developer** section of your ImageKit dashboard.

### 5. Start the backend

```bash
uvicorn app.app:app --reload
```

The API will be available at `http://localhost:8000`.
Interactive API docs are at `http://localhost:8000/docs`.

### 6. Start the frontend

Open a second terminal, activate the virtual environment again, then run:

```bash
streamlit run frontend.py
```

The UI will open at `http://localhost:8501`.

### 7. Create an account and start posting

Navigate to `http://localhost:8501`, register with an email and password, then log in. You can now upload images or videos and see them appear in the feed.

---

## Notes

- The SQLite database file (`test.db`) is created automatically on first run in the project root.
- JWT secret key length should be at least 32 characters in production to avoid security warnings from the `jwt` library.
- The `.env` file contains sensitive credentials and should never be committed to version control. Add it to `.gitignore`.
