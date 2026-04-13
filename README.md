# Pigfest 70 MCP Server

A scaffolded repository for a Dockerised MCP server that syncs a shared Google Drive folder into `/data` and exposes file + shell tools to an MCP-compatible client.

## Source of truth

The initial project requirements live in:

- `Specs/pigfest-70-mcp-spec.md`

## What this repo currently includes

- sensible Git hygiene via `.gitignore`
- a starter `docker-compose.yml`
- `.env.example` for deployment configuration
- a GitHub Actions workflow to publish a container image to GHCR
- a basic project layout ready for implementation

## Planned components

- `rclone`-based Google Drive sync loop
- Python MCP server exposing file and bash tools
- Docker image for Portainer deployment
- GHCR publishing on push to `main`

## Quick start

1. Copy `.env.example` to `.env` and fill in your values.
2. Implement the application runtime under `src/` and `scripts/`.
3. Push to GitHub.
4. Let GitHub Actions publish `ghcr.io/<github-user>/<repo>:latest`.
5. Deploy using `docker-compose.yml` in Portainer.

## Notes

- This repository was initialised from the project spec and prepared for GitHub + container publishing.
- If you want, I can now build out the actual MCP server, Dockerfile, and startup scripts next.
