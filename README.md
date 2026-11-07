# AI Customer Support Automation

An email support automation project for the fictional Harbor & Pine home and outdoor store. The intended system checks a Gmail inbox, categorizes incoming messages, drafts policy-based replies, and escalates cases that need a person. It includes a FastAPI backend, a Supabase/PostgreSQL schema, an admin dashboard, and evaluation examples.

> **Current status:** The repository contains the components for this workflow, but the checked-in files are not currently a complete runnable application. In particular, the database module is missing functions used by the backend, and the knowledge-base files are not in the location expected by the seed script. See [Known gaps](#known-gaps) before trying to run or deploy it.

## How it is intended to work

1. `app.poller` periodically asks Gmail for inbox messages that do not have the `ai-processed` label.
2. `app.pipeline` filters automated or self-sent messages, records the message and ticket, and formats the email thread.
3. `app.ai.classifier` chooses a support category and whether the message needs a human. `app.ai.agent` can use knowledge-base entries to draft a response.
4. `app.ai.decide` and `app.ai.rules` apply Python rules to choose whether to reply or escalate. Attachments and certain requests are escalated.
5. The Gmail client sends replies and uses the `needs-human` label for escalated conversations.
6. The FastAPI service exposes the dashboard API and serves the built React dashboard. The dashboard shows events and tickets and allows knowledge-base entries to be edited.

The AI categories and support policies are written for Harbor & Pine, including paddle boards, shelving, returns and warranty, shipping, and other questions.

## Repository layout

| Path | Purpose |
| --- | --- |
| `app/main.py` | FastAPI app, health endpoint, authenticated dashboard API, and static dashboard serving. |
| `app/config.py` | Loads environment-based application settings. |
| `app/gmail.py` | Gmail API client, labels, message/thread retrieval, reply sending, and email formatting helpers. |
| `app/poller.py` | Poll loop and one-time inbox polling function. |
| `app/pipeline.py` | Processes a message, applies guards, invokes the AI decision flow, and updates Gmail and database state. |
| `app/guards.py` | Rules for skipping messages the bot should not answer. |
| `app/ai/` | Classification, structured response schemas, policy decisions, and issue-resolution rules. |
| `app/db.py` | Database query functions currently present in the checkout; see [Known gaps](#known-gaps). |
| `supabase/migrations/` | SQL migration defining events, tickets, messages, and knowledge-base tables. |
| `scripts/get_refresh_token.py` | Gmail OAuth helper that obtains a refresh token. |
| `scripts/gmail_check.py` | Manual Gmail connection and reply check. |
| `scripts/seed_knowledge_base.py` | Reads knowledge-base Markdown files and inserts their sections into the database. |
| `evals/golden_emails.py` | Example customer emails and expected categories/actions. |
| `evals/run_evals.py` | Evaluates the AI decision flow against the example emails. |
| `playground/try_pipeline.py` | Small manual example for calling the decision flow. |
| `dashboard/` | React/Vite admin dashboard built and served by the Docker image. |
| `ai_coustmer_service/` | Separate Vite/React starter scaffold; it is not used by the root Dockerfile. |
| `next-app/` | Separate Next.js starter scaffold; it is not used by the root Dockerfile. |
| `pyproject.toml`, `uv.lock`, `.python-version` | Python project dependencies, lockfile, and Python version selection. |
| `Dockerfile`, `docker-compose.yml` | Build the dashboard and Python API image, then define the API and poller services. |
| `.env.example` | Names and example defaults for runtime configuration variables. |

Dependency directories and generated files such as `.venv/`, `node_modules/`, and build output are not part of the application source description above.

## Requirements

The project configuration indicates these tools and services are needed:

- Python 3.13 and [uv](https://docs.astral.sh/uv/)
- Node.js 22 or newer and npm, for the dashboard
- Docker and Docker Compose, for the container setup
- A Gmail account configured for OAuth 2.0 and the Gmail API
- A Supabase/PostgreSQL database
- An Anthropic API key for the configured Pydantic AI models

The detailed account and OAuth setup is covered by the linked project guide below.

## Configuration

Create a local environment file from the template:

```bash
cp .env.example .env
```

Set the values required by your environment in `.env`:

| Variable | Purpose |
| --- | --- |
| `GOOGLE_CLIENT_ID` | Google OAuth client ID for Gmail access. |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret. |
| `GOOGLE_REFRESH_TOKEN` | Refresh token for the Gmail account the poller uses. |
| `DATABASE_URL` | PostgreSQL connection string. |
| `ANTHROPIC_API_KEY` | API key used by Pydantic AI. |
| `CLASSIFIER_MODEL` | Model identifier used for message classification. |
| `AGENT_MODEL` | Model identifier used to draft support replies. |
| `POLL_INTERVAL_SECONDS` | Delay between inbox polling cycles. |
| `DASHBOARD_USERNAME` | Username required by the dashboard API. |
| `DASHBOARD_PASSWORD` | Password required by the dashboard API; the backend rejects login when this is empty. |
| `PYDANTIC_AI_NO_BANNER` | Pydantic AI environment option included in the template. |

`credentials.json` is used by the Gmail OAuth helper. Do not publish OAuth credentials, refresh tokens, database passwords, or API keys. The root `.dockerignore` excludes `.env` and `credentials.json` from the Docker build context; it does not prevent Git from tracking them. The inspected project root does not currently contain a root `.gitignore`, so check the files staged for GitHub before publishing.

## Database setup

The SQL migration in `supabase/migrations/` creates four tables:

- `events` for message-processing status and errors
- `tickets` for Gmail conversation-level support records
- `messages` for incoming and outgoing message bodies
- `knowledge_base` for sectioned policy and product information

Apply the migration to the Supabase project you configure, following the project guide. The backend expects to connect to PostgreSQL through `DATABASE_URL`.

## Running locally

The repository includes the following intended commands. **They may fail with the current files because of the known gaps below.**

Install Python dependencies:

```bash
uv sync
```

Start the FastAPI development server:

```bash
uv run fastapi dev app/main.py
```

The health endpoint is intended to be available at `http://localhost:8000/health`. The Vite dashboard development server can be started separately:

```bash
cd dashboard
npm install
npm run dev
```

The dashboard Vite configuration proxies `/api` requests to `http://127.0.0.1:8000`.

## Evaluation examples

The evaluation suite is defined in `evals/run_evals.py` and uses the examples in `evals/golden_emails.py`. The configured checks compare each decision's category and action with the expected values. The evaluation calls the AI pipeline, so it needs the model configuration and a working database/knowledge-base connection.

The source defines these npm scripts for the dashboard: `dev`, `build`, `lint`, `format`, `typecheck`, and `preview`.

## Docker deployment

The root `Dockerfile` builds `dashboard/` with Node.js 22, copies the resulting static files into a Python 3.13 image, and starts the FastAPI app. `docker-compose.yml` defines:

- `api`, bound to `127.0.0.1:8080` on the host
- `poller`, running `python -m app.poller`

Both services use `.env`, restart unless stopped, and share a built image. The API port is bound to localhost, consistent with accessing it through a local connection or SSH tunnel. The container setup is present, but the current database helper and knowledge-base path gaps need to be addressed before expecting the services to work end to end.

## Known gaps

These notes describe the files in this checkout, rather than assuming the missing pieces are present:

- `app/db.py` currently defines list and update functions, but does not define `connect`, `start_event`, `finish_event`, `fail_event`, `upsert_ticket`, `save_message`, `update_ticket`, `list_sections`, or `get_section`. The backend, pipeline, AI tools, and seed script call several of these functions.
- `scripts/seed_knowledge_base.py` looks for Markdown files in a top-level `knowledge_base/` directory. That directory is not present in the project root. Markdown knowledge-base files are currently located under `.venv/knowledge_base/`, which is a virtual-environment directory, and the Docker build does not copy them into the image.
- The `ai_coustmer_service/` and `next-app/` directories are starter frontends separate from `dashboard/`; only `dashboard/` is built in the root Dockerfile.
- The root directory has `.env` and `credentials.json` in the inspected copy. Confirm neither is staged or committed before publishing this project.
