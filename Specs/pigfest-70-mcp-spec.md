# Project Spec: Collaborative MCP Server with Google Drive Sync

## Overview

This project provides a self-contained Docker container that serves two purposes:

1. **Google Drive Sync** — Keeps a local folder in the container continuously synchronised with a shared Google Drive folder, so collaborators can add, edit, and remove files via their Google Drive account and those changes automatically become available inside the container.
2. **MCP Server** — Exposes an MCP (Model Context Protocol) endpoint over the network that allows Claude (or any MCP-compatible AI client) to read and write files in the synced folder and execute arbitrary bash commands. Access is optionally restricted via a bearer token.

The container is designed to be deployed via a Docker Compose file in Portainer. The image is automatically built and published to GitHub Container Registry via GitHub Actions, so the Portainer deployment simply pulls the latest image without needing any local build step.

---

## Goals

- Non-technical collaborators only need a Google account and access to a shared Google Drive folder. They never interact with the server or any tooling directly.
- The person running the server only needs to fill in a handful of environment variables and spin up the container in Portainer.
- Claude Cowork (or any MCP client) connects to the server using a URL and an optional bearer token — nothing else to install or configure on the client side.
- The whole system should be robust enough for a test/collaborative environment without being overly complex.

---

## Architecture Summary

```
Google Drive (shared folder)
        ↕  rclone bidirectional sync (scheduled)
Local folder inside Docker container (/data)
        ↕  fastMCP reads/writes files + runs bash
MCP Server (network port, bearer token auth)
        ↕  HTTP/SSE
Claude Cowork (or any MCP client)
```

---

## Components

### 1. rclone — Google Drive Sync

rclone is a mature, well-supported command-line tool for syncing files between local storage and cloud providers including Google Drive. It will be installed inside the container and configured to sync a specific Google Drive folder to a local directory at `/data`.

**Authentication**

rclone supports authenticating with Google Drive using OAuth2. Since this container runs headlessly (no browser), the recommended approach is to authenticate once on a local machine using the rclone interactive setup, which produces a token file. This token file is then passed into the container as an environment variable or mounted as a secret. The container will write it to the appropriate rclone config location on startup.

Alternatively, a Google Service Account JSON key can be used. This is cleaner for server deployments — you create a service account in Google Cloud Console, grant it access to the shared Drive folder, and pass the JSON key as an environment variable. The shared Drive folder simply needs to be shared with the service account's email address.

The service account approach is recommended because:
- No interactive login step required
- No token expiry issues
- Easy to revoke access without affecting personal accounts

**Folder Targeting**

The specific Google Drive folder to sync is identified by its folder ID (the long string visible in the Google Drive URL when you open the folder). This is passed as an environment variable. The container will sync the contents of this folder to `/data` inside the container.

**Sync Strategy**

rclone will run in bidirectional sync mode on a schedule. A simple loop or cron job inside the container will trigger a sync every configurable number of seconds (defaulting to around 30 seconds). This means:
- Files added or changed in Google Drive appear locally within ~30 seconds
- Files added or changed locally are pushed to Google Drive within ~30 seconds

This is the simplest and most reliable approach. Google Drive does not provide native push notifications in a way that is straightforward to consume without building additional webhook infrastructure. Polling at 30-second intervals is well within Google Drive API rate limits for personal and small team use.

**Conflict Handling**

rclone's bisync mode handles conflicts by keeping both versions and renaming one. For the purposes of this project (small team, test environment), the default conflict behaviour is acceptable. It should be documented that two people should avoid editing the same file simultaneously.

---

### 2. fastMCP — MCP Server

fastMCP is a Python library that makes it straightforward to build MCP-compliant servers. It handles the MCP protocol, tool registration, bearer token authentication, and network exposure out of the box, removing the need for custom middleware.

**What the MCP server exposes**

The server exposes a set of tools that Claude (or any MCP client) can invoke:

- **Run bash command** — Executes any bash command inside the container and returns stdout and stderr. No restrictions on which commands can be run, since this is a controlled test environment. Errors are caught and returned as structured responses rather than crashing the server.
- **Read file** — Reads the contents of a file within `/data` and returns them as text.
- **Write file** — Writes or overwrites a file within `/data` with provided content.
- **List directory** — Lists files and folders within a given path inside `/data`.
- **Delete file** — Deletes a specified file within `/data`.

These tools give Claude full ability to manage the synced folder contents and run any supporting commands needed during a collaborative session.

**Error Handling**

All tool implementations will wrap execution in try/except blocks. If a bash command fails, the stderr output and exit code are returned in the response so Claude can understand what went wrong and adjust. The server itself will not crash on tool errors — it will return a structured error message to the client.

**Bearer Token Authentication**

fastMCP supports bearer token authentication natively. An optional bearer token is passed as an environment variable. If set, any MCP client connecting to the server must include the token in its Authorization header. If the environment variable is left empty, the server runs without authentication (suitable for a fully private/internal network).

**Network Exposure**

The MCP server listens on a configurable port (defaulting to 8000) on all network interfaces inside the container, so it is accessible from outside the container via the host machine's IP or domain. The Docker Compose file maps this port to the host.

The MCP URL that clients (e.g. Claude Cowork) will use will be in the format:
`http://<your-server-ip>:<port>/sse`

---

### 3. Logging

Logging is intentionally minimal. The container writes basic operational logs — startup confirmation, sync events, errors — to a log file at `/logs/app.log`. This directory is mounted as a Docker volume so logs persist across container restarts and can be inspected via Portainer. No complex log aggregation or rotation is required for this project.

---

### 4. Docker Container

The container is built on a lightweight base image (e.g. a slim Python image). On top of this:
- rclone is installed
- Python dependencies including fastMCP are installed
- A startup script runs on container launch that:
  1. Writes the rclone config from the provided environment variables
  2. Performs an initial sync to populate `/data` before the MCP server starts
  3. Starts the background sync loop
  4. Starts the fastMCP server

The `/data` folder and `/logs` folder are both exposed as Docker volumes so their contents persist if the container is restarted.

---

### 5. Docker Compose Configuration

The project provides a `docker-compose.yml` file for use in Portainer. The user only needs to paste this file into Portainer's stack deployment and fill in the environment variable values.

**Environment Variables**

| Variable | Description |
|---|---|
| `GDRIVE_FOLDER_ID` | The Google Drive folder ID to sync |
| `GDRIVE_SERVICE_ACCOUNT_JSON` | The full contents of the Google Service Account JSON key, or a path to a mounted secret |
| `SYNC_INTERVAL_SECONDS` | How often rclone syncs (default: 30) |
| `MCP_PORT` | The port the MCP server listens on (default: 8000) |
| `MCP_BEARER_TOKEN` | Optional bearer token for MCP access. Leave empty for no auth. |

**Volumes**

Two named volumes are defined:
- `mcp_data` — mounted at `/data`, holds the synced Google Drive files
- `mcp_logs` — mounted at `/logs`, holds log files

---

### 6. GitHub Actions — Automated Build and Publish

A GitHub Actions workflow is included in the repository that triggers on every push to the `main` branch. The workflow:

1. Checks out the repository
2. Logs in to GitHub Container Registry (ghcr.io) using the repository's built-in `GITHUB_TOKEN` secret — no manual secret setup required
3. Builds the Docker image
4. Tags it as `latest` and with the commit SHA
5. Pushes it to `ghcr.io/<your-github-username>/<repo-name>`

The Docker Compose file references the `latest` tag from ghcr.io, so Portainer always pulls the most recently built image. To update the running container, you simply pull the new image and recreate the stack in Portainer.

The GitHub Container Registry package visibility should be set to **public** (or the Portainer host authenticated to ghcr.io) so the image can be pulled without credentials.

---

## Setup Steps (High Level)

1. **Google Cloud** — Create a Google Cloud project, enable the Drive API, create a Service Account, download the JSON key, and share the target Google Drive folder with the service account's email address.
2. **Google Drive** — Create the shared folder, share it with your collaborator's Google account (standard Drive sharing), and share it with the service account email.
3. **GitHub** — Fork or clone this repository. The GitHub Actions workflow will automatically build and publish the image on your first push.
4. **Portainer** — Create a new stack, paste in the `docker-compose.yml`, fill in the environment variables, and deploy.
5. **Claude Cowork** — Add a new MCP connector using the server URL and bearer token.
6. **Collaborator onboarding** — Share the Google Drive folder link with your collaborator. That's all they need. They add/edit files in Drive and those changes flow through automatically.

---

## Limitations and Considerations

- **Sync delay** — There is an inherent delay of up to the sync interval (default 30 seconds) between a file being changed in Google Drive and it being available to Claude via the MCP server. This is acceptable for most collaborative workflows.
- **Simultaneous edits** — If two people edit the same file at the same time, rclone's conflict resolution will preserve both versions. The team should be aware of this and avoid working on the same file simultaneously.
- **Security** — The MCP server exposes arbitrary bash execution. It should not be exposed on a public IP without the bearer token set, and ideally should sit behind a firewall or VPN. For a test/internal environment this is acceptable.
- **Google Drive API quotas** — For small teams syncing a modest number of files, the default Google Drive API quotas are more than sufficient. No special quota increases should be needed.
- **Binary files** — rclone handles binary files (images, PDFs, Office documents etc.) without issue. The MCP file read tool is intended for text files; binary files would need to be handled via bash commands if manipulation is needed.
