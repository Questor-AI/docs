# Run Questor with Docker

This guide explains how to run the **prebuilt** Questor image from **GitHub Container Registry (GHCR)**. You do **not** need the application source code or a compiler—only Docker, a Compose file, and a small environment file.

---

## What you run

| | |
|--|--|
| **Image** | `ghcr.io/questor-ai/questor-app:latest` on [GHCR](https://github.com/orgs/Questor-AI/packages/container/package/questor-app) (under the [Questor-AI](https://github.com/Questor-AI) organization) |
| **Typical use** | Web portal (browser UI) for security reports and scans, with data stored in a Docker volume |

Because the image is in a **private** package, each person or machine must authenticate to **`ghcr.io`** with credentials that can pull `questor-ai/questor-app` before `docker pull`.

---

## Requirements

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2: `docker compose` command)
- A **GitHub account** granted access to the Questor-AI container package (your administrator adds you), and a [**personal access token (classic)**](https://github.com/settings/tokens) with at least the **`read:packages`** scope (used only for `docker login ghcr.io`, not inside `.env` unless your team says otherwise)

---

## 1. Set up a folder

Create a directory on your computer and use it for all commands below (for example `~/questor-run`). You will place two files there:

- `docker-compose.yml` — downloaded in the next step  
- `.env` — created in step 3  

---

## 2. Download the Compose file

The Compose file lives in the [Questor-AI/docs](https://github.com/Questor-AI/docs) repository: [`setup/docker-compose.yml`](https://github.com/Questor-AI/docs/blob/main/setup/docker-compose.yml).

Download it using either option:

- **Option A — Browser:** Open the [**raw** `docker-compose.yml`](https://raw.githubusercontent.com/Questor-AI/docs/main/setup/docker-compose.yml) (plain text). Use **Save As** / **Save Page As** and save it as `docker-compose.yml` in the folder you created in step 1. (If the file opens in the tab, use your browser’s save command, e.g. **File → Save As**.)

- **Option B — Command line** (downloads straight to `docker-compose.yml`):

  ```bash
  curl -fsSL -o docker-compose.yml https://raw.githubusercontent.com/Questor-AI/docs/main/setup/docker-compose.yml
  ```

Proceed when you have the **`docker-compose.yml`** in your folder.

---

## 3. Create your `.env` file

In the **same folder** as `docker-compose.yml`, create a file named **`.env`** (leading dot, no extension).

**Example `.env` (adjust paths and secrets):**

```bash
HOST_REPO_PATH=/absolute/path/to/your/source
LLM_API_KEY=ABC123
```

### Required for normal use

| Variable | Description |
|----------|-------------|
| **`HOST_REPO_PATH`** | **Absolute path** on your machine to the **source code** you want to scan. The container mounts it read-only at `/repo`. Example: `/Users/you/projects/my-app` (macOS/Linux) or `C:\path\to\my-app` (Windows—use the path format your Docker setup expects in `.env`). |
| **`LLM_API_KEY`** | API key for the LLM provider used by the scanner and related features, if your deployment uses them. If you run fully offline or without validation, your administrator may tell you to disable validation instead—see optional settings below. |

### Optional Values

| Variable | Default | Description |
|----------|---------|-------------|
| **`PORTAL_PORT`** | `8080` | Host port for the web UI (maps to the app’s port 80 inside the container). |

Your administrator may provide more variables (timeouts, logging, etc.). Use whatever they supply; names are documented in any **`env.example`** they share.

---

## 4. Log in to GHCR to access the private image

The QuestorAI container is private, so you must sign in to **`ghcr.io`** with a GitHub user or token that has permission to pull the image.

Create a token with **`read:packages`** (see [GitHub: Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)), then run:

```bash
echo YOUR_GITHUB_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

Replace **`YOUR_GITHUB_USERNAME`** with your GitHub username and **`YOUR_GITHUB_TOKEN`** with the PAT. Alternatively, if you use the [GitHub CLI](https://cli.github.com/) (`gh auth login`), you can run:

```bash
echo "$(gh auth token)" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

---

## 5. Pull the image and start the portal

From the folder that contains **`docker-compose.yml`** and **`.env`**:

```bash
docker pull ghcr.io/questor-ai/questor-app:latest
docker compose --profile portal up -d
```

Open a browser: **http://127.0.0.1:8080** — or `http://127.0.0.1:<PORTAL_PORT>` if you set `PORTAL_PORT` in `.env`.

To stop:

```bash
docker compose --profile portal down
```

---

## Using a different image tag

The Compose file you were given may pin **`ghcr.io/questor-ai/questor-app:latest`**. If your administrator asks you to run a specific version, they may send an updated Compose file or ask you to change every `image:` line to the tag they specify (for example `ghcr.io/questor-ai/questor-app:1.0.3`).

---

## Troubleshooting

| Problem | What to try |
|---------|-------------|
| `docker pull` denied / 403 | Run `docker login ghcr.io` with a user and PAT that can **`read:packages`** for the Questor-AI org package. Confirm your GitHub account was granted access to the private image. |
| Error about `HOST_REPO_PATH` | Set it to a real **absolute** path on the host; the folder must exist before you start the container. |
| Port already in use | Set `PORTAL_PORT` in `.env` to a free port (for example `8081`). |
| Scan or LLM errors | Confirm `LLM_API_KEY` and any flags your administrator gave you; some environments use `SCAN_VALIDATE=0` for offline use—only if they told you to. |

For questions about keys, network policy, or which tag to use, contact whoever provided the image or this Compose file.
