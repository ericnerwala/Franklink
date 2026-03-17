# Franklink

Franklink is an iMessage-based AI career and networking assistant. It ingests inbound messages from Photon, routes direct messages through a FastAPI + LangGraph application, and handles managed group chats, match introductions, and proactive background jobs on top of Supabase, Azure OpenAI, Zep, Kafka, and Composio.

## Why we built it

Professional networking is still too manual, too cold, and too badly timed for the moments when people actually need help. Most students and early-career builders do not need a giant social graph. They need the right introduction, resource, or small group at the exact moment they are preparing for an interview, looking for collaborators, planning an event, or trying to learn from peers with adjacent experience.

Franklink is built to turn the messaging channel people already use into a higher-leverage networking layer. Instead of asking users to maintain another profile or browse a directory, Franklink uses conversational context, inbox signals, shared interests, location, and demand/value matching to help people meet the right person faster.

## Problem it solves

Franklink is designed around a few concrete failures in traditional networking products:

- Useful people are hard to discover at the moment of need.
- Existing professional networks are optimized for broadcasting, not warm introductions or small-group coordination.
- Valuable context lives in scattered places like texts, inboxes, schools, events, and calendars instead of one structured profile.
- Group chats form organically, but they rarely get help with expansion, follow-up, scheduling, or keeping momentum.
- A generic chatbot can answer questions, but it usually cannot coordinate introductions, context-aware outreach, and group formation inside the communication tools people already use.

## Core use cases

- A student wants a mock interviewer, mentor, recruiter insight, or referral path for a specific role.
- A founder or builder wants collaborators for a project, hackathon, or startup idea.
- A user wants relevant opportunities or resources based on their interests, school, inbox activity, or event context.
- An existing group chat wants Frank to find and invite one more person who fits the group goal.
- A conversation has gone quiet and needs a useful follow-up, summary, or scheduling nudge instead of more manual coordination.
- A user wants their networking context to improve over time as Franklink learns from onboarding, email signals, location updates, and prior conversations.

## What the app does

- Runs direct-message flows for onboarding, recommendation, networking, updates, and general chat.
- Creates and manages networking introductions, including handshake flows and group chat creation.
- Handles explicit `Frank` invocations inside group chats for group chat networking and opinion-style responses.
- Pulls email context through Composio/Gmail to improve onboarding and matching context.
- Runs background workers for group chat summaries, follow-ups, daily email extraction, proactive outreach, and location-related maintenance, with profile synthesis scheduled from the API process.
- Supports two inbound modes:
  - direct Photon listener/webhook handling
  - Kafka-backed ingest plus worker consumption for higher-throughput processing

## High-level architecture

Core services in this repository:

- `app/main.py`: FastAPI entrypoint, Photon webhook endpoints, Kafka producer/consumer startup, diagnostics, and payment webhooks.
- `app/orchestrator.py`: main message orchestration for direct messages.
- `app/agents/interaction/`: routing and task selection across onboarding, networking, recommendations, and general chat.
- `app/groupchat/`: managed group chat recording, routing, networking, summaries, and follow-up flows.
- `app/proactive/`: daily email extraction, location updates, and proactive outreach workers.
- `app/database/`: Supabase-backed data access layer.
- `infrastructure/aws/ecs/`: current ECS deployment assets and task definitions.

## Main dependencies

- FastAPI and Uvicorn
- LangGraph
- Azure OpenAI
- Supabase
- Photon
- Kafka / aiokafka
- Redis
- Zep
- Composio
- Stripe

## Repository layout

```text
app/                  Application code
docs/                 Runtime and testing documentation
infrastructure/       Deployment configs and scripts
support/scripts/      SQL migrations, smoke tests, and utility scripts
support/openspec/     Change proposals and project specs
```

## Local development

### Prerequisites

- Python 3.11+ recommended
- Access to required external services and credentials:
  - Photon
  - Supabase
  - Azure OpenAI
  - resources Supabase database
  - Redis
  - Zep
  - Composio
  - Stripe
- Docker Desktop if you want the Kafka local stack

### Environment

This repo expects a `.env` file at the project root. There is no committed `.env.example` in this checkout, so create the file manually with the settings required by [`app/config.py`](/Users/eric/Downloads/Franklink-iMessage/app/config.py).

At minimum, local app startup requires values for:

- `PHOTON_SERVER_URL`
- `PHOTON_DEFAULT_NUMBER`
- `AZURE_OPENAI_API_KEY`
- `AZURE_OPENAI_ENDPOINT`
- `AZURE_OPENAI_DEPLOYMENT_NAME`
- `AZURE_OPENAI_REASONING_DEPLOYMENT_NAME`
- `SUPABASE_URL`
- `SUPABASE_KEY`
- `RESOURCES_SUPABASE_URL`
- `RESOURCES_SUPABASE_KEY`

Depending on which features you enable, you will also need Redis, Kafka, Zep, Composio, and Stripe settings.

### Run the API directly

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

By default, the app serves on `http://127.0.0.1:8000`.

Useful endpoints:

- `GET /health`
- `POST /webhook/photon`
- `POST /send-message`
- `POST /send-poll`

### Run the Kafka-based local stack

The included `docker-compose.yml` starts:

- Zookeeper
- Kafka
- `photon-ingest` service running the FastAPI app in Kafka ingest mode
- `frank-worker` service running the FastAPI app in Kafka consumer mode
- `background-workers` supervisor

Start it with:

```bash
docker compose up --build
```

In this mode:

- `photon-ingest` accepts inbound Photon traffic and publishes to Kafka
- `frank-worker` consumes Kafka events and processes conversations
- `background-workers` runs the looping worker processes

## Background jobs and workers

The supervisor in [`app/workers/background_supervisor.py`](/Users/eric/Downloads/Franklink-iMessage/app/workers/background_supervisor.py) launches:

- group chat summary worker
- group chat follow-up worker
- daily email worker
- proactive outreach worker

Additional worker entrypoints also exist for other job types, including location updates, and the API startup also schedules periodic profile synthesis work.

## Documentation

- [`docs/KAFKA_CONCURRENCY_TEST.md`](/Users/eric/Downloads/Franklink-iMessage/docs/KAFKA_CONCURRENCY_TEST.md): local and container-based Kafka load testing
- [`docs/EMAIL_EXTRACTION_FUNCTION.md`](/Users/eric/Downloads/Franklink-iMessage/docs/EMAIL_EXTRACTION_FUNCTION.md): email extraction pipeline notes
- [`support/docs/LANGGRAPH_ARCHITECTURE.md`](/Users/eric/Downloads/Franklink-iMessage/support/docs/LANGGRAPH_ARCHITECTURE.md): current DM graph architecture
- [`infrastructure/aws/ecs/README.md`](/Users/eric/Downloads/Franklink-iMessage/infrastructure/aws/ecs/README.md): current AWS ECS deployment workflow

## Notes

- App Runner documentation in this repository is legacy; the current deployment path is ECS.
- `support/openspec/` contains in-progress and historical design changes, so not every document there reflects currently deployed behavior.
