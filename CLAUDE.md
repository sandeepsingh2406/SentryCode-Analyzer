# Project Context for Claude Code

## Overall context
- This project is called SentryCode — an AI agent that analyzes Python code for security vulnerabilities
- Instructions and tooling need to work on Windows, macOS, and Linux
- The project deploys locally with npm and uv run, as a single Docker container, to Azure Container Apps, and to GCP Cloud Run
- There is a `.env` file in the project root (not committed) with `OPENAI_API_KEY` and `SEMGREP_APP_TOKEN`

## Project Overview
SentryCode is a web application for analyzing Python code for security vulnerabilities using AI-powered analysis with OpenAI and Semgrep.

## Architecture
- **Frontend**: Next.js (React) with TypeScript, Tailwind CSS
  - Located in `frontend/`
  - Runs on port 3000 in development
  - Built as a static export for production
- **Backend**: FastAPI with Python 3.12
  - Located in `backend/`
  - Runs on port 8000
  - Uses the OpenAI Agents SDK with a Semgrep MCP server

## Key Technical Decisions

### Docker Deployment
- Simplified from a multi-stage supervisor approach to single-stage deployment
- Frontend is built as a static export (`next export`) and served directly by FastAPI
- Single container exposes port 8000 for both API and static files
- Optimized for Google Cloud Run and Azure Container Apps

### MCP Version Pinning
- **Issue**: MCP library updated from 1.12.2 to 1.12.3, and FastMCP stopped accepting a `version` parameter in its constructor
- **Solution**: Pin MCP to version 1.12.2 in `pyproject.toml` and use `uvx --with mcp==1.12.2` when launching semgrep-mcp
- **Reason**: semgrep-mcp v0.4.1 still passes the `version` parameter, causing a `TypeError` with MCP 1.12.3+

## Development Setup

### Environment Variables
Required in `.env` file:
- `OPENAI_API_KEY` - For OpenAI API access
- `SEMGREP_APP_TOKEN` - For Semgrep analysis

### Local Development
```bash
# Backend
cd backend
uv run server.py

# Frontend (in separate terminal)
cd frontend
npm run dev
```

### Docker Commands
```bash
# Build
docker build -t sentrycode .

# Run with env file
docker run --rm -d --name sentrycode -p 8000:8000 --env-file .env sentrycode

# Logs
docker logs sentrycode

# Stop
docker stop sentrycode
```

## Important Implementation Details

1. **Static File Serving**: FastAPI serves the Next.js static export from the `static` directory. The `/health` endpoint must be defined before mounting static files to avoid route conflicts.
2. **API Routes**: All API endpoints are under the `/api/` prefix (e.g., `/api/analyze`).
3. **Frontend Configuration**:
   - `next.config.ts` uses `output: 'export'` for static generation
   - `trailingSlash: true` for proper routing
   - `images.unoptimized: true` for static export compatibility

## Known Issues & Workarounds

1. **MCP Version Compatibility**: Must use MCP 1.12.2 until semgrep-mcp is updated to remove the `version` parameter from FastMCP initialization.

## Testing & Quality
- Run `npm run lint` in frontend for linting
- Run `npm run typecheck` in frontend for type checking
- Backend uses `uv` for dependency management

## Future Considerations
- Monitor semgrep-mcp updates for compatibility with MCP 1.12.3+
- Consider adding automated tests
- May need to adjust Docker health check timeout for cloud deployments

## Cloud Deployment

### Deployment Strategy
1. **Azure**
   - Azure Container Apps (serverless container platform)
   - Azure Container Registry for image storage
   - Azure Key Vault for secrets management
2. **GCP**
   - Google Cloud Run (equivalent to Azure Container Apps)
   - Artifact Registry for container images
   - Secret Manager for environment variables
3. **Infrastructure as Code**
   - Terraform with workspaces to manage both clouds
   - Modular design for reusable components
   - Clear separation between Azure and GCP configurations

### Current Status
- Azure deployment completed — application successfully deployed to Azure Container Apps
- Docker image optimized — multi-stage build with ARM64→AMD64 cross-compilation for cloud compatibility
- Terraform deployment pipeline working for Azure
- CORS and API routing resolved — frontend uses relative URLs in production, localhost in development
- MCP server memory issue resolved (see below)
- GCP Cloud Run deployment not yet started

### MCP Server Memory Issue — Resolved
**Issue**: Semgrep MCP server was getting SIGKILL (-9) on Azure when loading the rule registry.
- `list_tools` worked but `semgrep_scan` failed with exit code -9
- Process killed right after "Loading rules from registry..."
- **Root cause**: Insufficient memory allocation (1.0Gi)
- **Solution**: Increased container memory from 1.0Gi to 2.0Gi and CPU from 0.5 to 1.0
- **Verified**: Works on both Azure Container Apps and Azure Container Instances with 2GB RAM

**Key lesson**: Semgrep rule registry loading is memory-intensive and requires at least 2GB RAM in cloud environments.

### Key Deployment Lessons Learned
1. **Terraform Docker Provider Limitations**:
   - Doesn't automatically detect source code changes
   - Must use `terraform taint` to force rebuilds when code changes
   - Using unique image tags can help avoid caching issues
2. **Frontend API URL Configuration**:
   - Next.js static export needs relative URLs for same-domain deployment
   - Environment-based logic: localhost in dev, relative URLs in production
   - CORS must allow wildcard origins when serving frontend from same domain
3. **Azure Container Apps Behavior**:
   - FQDN changes with each new revision (--0000001, --0000002, etc.)
   - Apps scale to zero automatically, saving costs
   - Logs accessible via `az containerapp logs show`
4. **Cross-Platform Docker Builds**:
   - M1 Mac builds ARM64 by default, Azure needs AMD64
   - Solution: `platform = "linux/amd64"` in Terraform Docker build
   - Slight performance penalty on M1 Macs but ensures compatibility
