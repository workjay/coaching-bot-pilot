# Coaching Bot Pilot — Local setup guide

This guide is written for anyone setting up the pilot on their own computer. **Docker** is used for the databases so you do not have to install PostgreSQL or Qdrant manually.

## What you need installed first

1. **Docker Desktop** — [Install Docker Desktop](https://www.docker.com/products/docker-desktop/) for Windows or Mac, start it, and wait until it shows “running”.
2. **Node.js** (LTS, e.g. v20 or v22) — [Download Node.js](https://nodejs.org/). This runs the coaching bot API.

---

## Step 1 — Get a free Gemini API key (for a quick test)

This pilot uses Google’s **Gemini** API for chat and embeddings. Google offers a **free API key** with usage limits, which is enough to try the pilot.

1. Open **[Google AI Studio — API keys](https://aistudio.google.com/api-keys)** in your browser.
2. Sign in with your Google account.
3. Create an **API key** and copy it. You will paste it into `.env` in a later step.

Keep this key private (do not share it or commit it to git).

---

## Step 2 — Start PostgreSQL with Docker

The app stores chat history and knowledge-base metadata in **PostgreSQL**. The example configuration uses:

- User: `postgres`
- Password: `postgres`
- Database name: `coaching_bot_pilot`
- Port: `5432`

### Option A — You do **not** have PostgreSQL yet (recommended)

In a terminal, from any folder you like, run:

```bash
docker run -d \
  --name coaching-postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=coaching_bot_pilot \
  -p 5432:5432 \
  -v coaching_postgres_data:/var/lib/postgresql/data \
  postgres:16
```

- **`-d`** runs the database in the background.
- Data is kept in a Docker volume named `coaching_postgres_data` so it survives restarts.

### Option B — You **already** have PostgreSQL installed

You can keep using your existing server instead of Docker:

1. Create a database named **`coaching_bot_pilot`** (and a user/password if you do not use the default `postgres` user).
2. If your PostgreSQL already uses **port 5432**, either:
   - stop the local PostgreSQL service while you use this project, **or**
   - run the Docker command above on another host port, for example **`-p 5433:5432`**, and set `DATABASE_URL` to use port `5433` (see Step 5).

Your `DATABASE_URL` must point at that database. Example if everything is on localhost with the same names as `.env.example`:

```text
postgresql://postgres:postgres@localhost:5432/coaching_bot_pilot?schema=public
```

Change user, password, host, port, or database name to match your setup.

---

## Step 3 — Start Qdrant (vector database) with Docker

Qdrant holds **vector embeddings** for the knowledge base search.

1. Pull the image (once):

```bash
docker pull qdrant/qdrant
```

2. Run Qdrant:

**macOS / Linux** — persists data in a folder `qdrant_storage` in your current directory:

```bash
docker run -d \
  --name coaching-qdrant \
  -p 6333:6333 \
  -p 6334:6334 \
  -v "$(pwd)/qdrant_storage:/qdrant/storage:z" \
  qdrant/qdrant
```

**Windows** — bind mounts can be awkward; use a **named Docker volume** instead:

```bash
docker run -d --name coaching-qdrant -p 6333:6333 -p 6334:6334 -v qdrant_storage:/qdrant/storage qdrant/qdrant
```

- **6333** — HTTP API (the app uses this by default).
- **6334** — gRPC (optional for this pilot).

For local Qdrant you usually **do not** need an API key; leave `QDRANT_API_KEY` empty in `.env` unless you have secured Qdrant.

---

## Step 4 — Create `.env` from `.env.example` and add your Gemini key

The file **`.env.example`** lists every setting the app needs, with sensible defaults copied from the project template. **`GEMINI_API_KEY`** is intentionally an empty string there — you fill it in your own **`.env`** (never commit real keys to git).

1. Open a terminal in the **project folder** (`coaching-bot-pilot`).

2. **Create `.env` by copying the example file** (pick the command for your system):

   **macOS / Linux:**

   ```bash
   cp .env.example .env
   ```

   **Windows (Command Prompt):**

   ```bat
   copy .env.example .env
   ```

   **Windows (PowerShell):**

   ```powershell
   Copy-Item .env.example .env
   ```

3. **Add your Gemini API key** (from Step 1):
   - Open **`.env`** in any text editor (Notepad, VS Code, etc.).
   - Find the line `GEMINI_API_KEY=""`.
   - Paste your key **between the quotes**, for example: `GEMINI_API_KEY="your-key-here"`.
   - Save the file.

4. **Adjust other values only if needed:**
   - **`DATABASE_URL`** — keep the default if you used **Option A** in Step 2 on port `5432`; otherwise set it to match your PostgreSQL (Step 2, Option B).

Other variables (Gemini model names, RAG settings, Qdrant host/port) can stay as copied from `.env.example` for a first test.

---

## Step 5 — Install dependencies and database tables

In the project folder:

```bash
npm install
npx prisma migrate deploy
```

(`migrate deploy` applies the existing migrations to your PostgreSQL database.)

---

## Step 6 — Run the app

```bash
npm run start:dev
```

The API should listen on the port in `.env` (default **3000**). Check the terminal for the exact URL.

---

## Quick checklist

| Item | How |
|------|-----|
| Docker | Docker Desktop running |
| PostgreSQL | Docker container **or** existing install with DB `coaching_bot_pilot` |
| Qdrant | Docker on ports **6333** / **6334** |
| Gemini | API key from [aistudio.google.com/api-keys](https://aistudio.google.com/api-keys) in `.env` |
| App | `npm install` → `npx prisma migrate deploy` → `npm run start:dev` |

---

## Stopping and removing the Docker containers (optional)

```bash
docker stop coaching-postgres coaching-qdrant
```

To remove them and start fresh (this deletes container config; named volumes like `coaching_postgres_data` keep data until you remove them with `docker volume rm`):

```bash
docker rm coaching-postgres coaching-qdrant
```

---

## Troubleshooting

- **“Port already in use”** for `5432`, `6333`, or `3000` — another program is using that port. Stop that program, or change the port in Docker (`-p`) and/or in `.env`.
- **Database connection errors** — confirm PostgreSQL is running (`docker ps`), and that `DATABASE_URL` matches user, password, database name, host, and port.
- **Qdrant connection errors** — confirm the Qdrant container is running and `QDRANT_HOST` / `QDRANT_PORT` in `.env` match (default `localhost` and `6333`).

For official Qdrant Docker details, see the [Qdrant documentation](https://qdrant.tech/documentation/guides/installation/).
