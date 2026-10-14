# SentryCode

![SentryCode](assets/cyber.png)

AI-powered security analysis for Python code. SentryCode combines static analysis via [Semgrep](https://semgrep.dev) with an LLM-driven security researcher agent to find vulnerabilities, explain their impact, and suggest fixes — all from a simple web UI.

## Features

- 📁 Upload any Python file (or paste code) for analysis
- 🔍 Static analysis powered by Semgrep's `auto` ruleset
- 🤖 An AI agent reviews the Semgrep output and adds its own findings beyond what static rules catch
- 📊 Each issue is reported with a title, description, vulnerable snippet, suggested fix, CVSS score, and severity
- ⬇️ Results sorted by severity (highest CVSS first)

## Architecture

| Layer    | Stack                                               |
|----------|------------------------------------------------------|
| Frontend | Next.js (React), TypeScript, Tailwind CSS            |
| Backend  | FastAPI, Python 3.12                                 |
| Agent    | OpenAI Agents SDK                                     |
| Analysis | Semgrep via a Model Context Protocol (MCP) server     |

In production, the Next.js app is built as a static export and served directly by FastAPI from a single container — one process, one port.

```
├── frontend/   # Next.js UI
├── backend/    # FastAPI app + agent/MCP wiring
├── terraform/  # Infrastructure as code (Azure, GCP)
└── Dockerfile  # Single-container build (frontend + backend)
```

## Prerequisites

- [uv](https://docs.astral.sh/uv/) (Python package/dependency manager)
- [Node.js](https://nodejs.org/) 20+ and npm
- An [OpenAI API key](https://platform.openai.com/api-keys)
- A [Semgrep App token](https://semgrep.dev/login) (free tier works)

## Setup

1. Clone the repo:

   ```bash
   git clone https://github.com/sandeepsingh2406/sentrycode.git
   cd sentrycode
   ```

2. Create a `.env` file in the project root:

   ```bash
   OPENAI_API_KEY=sk-...
   SEMGREP_APP_TOKEN=...
   ```

## Running locally

**Backend** (in one terminal):

```bash
cd backend
uv run server.py
```

Runs on [http://localhost:8000](http://localhost:8000).

**Frontend** (in another terminal):

```bash
cd frontend
npm install
npm run dev
```

Runs on [http://localhost:3000](http://localhost:3000).

## Running with Docker

Build and run as a single container (serves both the API and the static frontend on port 8000):

```bash
docker build -t sentrycode .
docker run --rm -d --name sentrycode -p 8000:8000 --env-file .env sentrycode
```

Check it's healthy:

```bash
curl http://localhost:8000/health
```

View logs / stop:

```bash
docker logs sentrycode
docker stop sentrycode
```

## Cloud Deployment

Terraform configurations are provided for two serverless container platforms:

- **Azure Container Apps** — `terraform/azure/`
- **Google Cloud Run** — `terraform/gcp/`

Each is a standard Terraform workspace:

```bash
cd terraform/azure   # or terraform/gcp
terraform init
terraform apply \
  -var="openai_api_key=$OPENAI_API_KEY" \
  -var="semgrep_app_token=$SEMGREP_APP_TOKEN"
```

See the `variables.tf` in each directory for the full list of configurable inputs (region, resource naming, image tag, etc.).

## API

### `POST /api/analyze`

Analyze a Python source file for security vulnerabilities.

**Request:**

```json
{ "code": "import os\nos.system(user_input)" }
```

**Response:**

```json
{
  "summary": "Analyzed 42 characters of Python code. ...",
  "issues": [
    {
      "title": "OS Command Injection",
      "description": "...",
      "code": "os.system(user_input)",
      "fix": "Use subprocess with a list of args instead of shell=True / os.system",
      "cvss_score": 9.8,
      "severity": "critical"
    }
  ]
}
```

### `GET /health`

Returns `{"message": "Cybersecurity Analyzer API"}` — used for container health checks.

## Tech Stack

- [FastAPI](https://fastapi.tiangolo.com/) — backend API framework
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) — agent orchestration
- [Semgrep](https://semgrep.dev/) — static analysis engine, invoked via an MCP server
- [Next.js](https://nextjs.org/) — frontend framework
- [Tailwind CSS](https://tailwindcss.com/) — styling
- [Terraform](https://www.terraform.io/) — infrastructure as code for Azure and GCP

## License

MIT
