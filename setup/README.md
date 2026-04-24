# Run Questor with Docker

This guide explains how to run the **prebuilt** Questor image from **GitHub Container Registry (GHCR)**. You do **not** need the application source code or a compiler—only Docker, a Compose file, and a small environment file.

---

## What you run

| | |
|--|--|
| **Image** | `ghcr.io/questor-ai/questor-app:latest` — [package on GHCR](https://github.com/Questor-AI/container-access/pkgs/container/questor-app) ([Questor-AI](https://github.com/Questor-AI) org). The image is **connected to** the [`container-access`](https://github.com/Questor-AI/container-access) repo so you can grant outside collaborators access to the package via that repository. |
| **Typical use** | Web portal (browser UI) for security reports and scans, with data stored in a Docker volume |

Because the image is in a **private** package, each person or machine must authenticate to **`ghcr.io`** with credentials that can pull `questor-ai/questor-app` before `docker pull`.

---

## Requirements

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2: `docker compose` command)
- A **GitHub account** granted access to the Questor-AI container package (your administrator adds you), and a [**personal access token (classic)**](https://github.com/settings/tokens) with at least the **`read:packages`** scope (used only for `docker login ghcr.io`, not inside `.env` unless your team says otherwise)

### Windows: Docker CLI from WSL

On **Windows**, if you run these steps from a **Linux terminal in WSL** (common with **Docker Desktop** and the WSL 2 backend), a fresh install sometimes leaves your Linux user without permission to use the Docker socket (`permission denied` when you run `docker`). Add your user to the `docker` group and apply it in the current shell:

```bash
sudo usermod -aG docker $USER && newgrp docker
```

If issues remain, **restart the WSL distribution** (or sign out and back in) so the group change is picked up everywhere.

---

## 1. Set up a folder

Create a directory on your computer and use it for all commands below (for example `~/questor-run`). You will place two files there:

- `docker-compose.yml` — downloaded in the next step  
- `.env` — created in step 3  

---

## 2. Download the Compose file

The Compose file lives in the [Questor-AI/docs](https://github.com/Questor-AI/docs) repository: [`setup/docker-compose.yml`](https://github.com/Questor-AI/docs/blob/main/setup/docker-compose.yml).

Download it using either option:

- **Option A — Browser:** Open the **raw** [`docker-compose.yml`](https://raw.githubusercontent.com/Questor-AI/docs/main/setup/docker-compose.yml) (plain text). Use **Save As** / **Save Page As** and save it as `docker-compose.yml` in the folder you created in step 1. (If the file opens in the tab, use your browser’s save command, e.g. **File → Save As**.)

- **Option B — Command line** (downloads straight to `docker-compose.yml`):

  ```bash
  curl -fsSL -o docker-compose.yml https://raw.githubusercontent.com/Questor-AI/docs/main/setup/docker-compose.yml
  ```

Proceed when you have the **`docker-compose.yml`** in your folder.

---

## 3. Create your `.env` file

In the **same folder** as `docker-compose.yml`, create a file named **`.env`** (leading dot, no extension).

For the setup in this guide, **only two variables are required** in `.env`: **`HOST_REPO_PATH`** and **`LLM_API_KEY`**. Everything else below is optional.

**Minimal `.env` (required variables only):**

```bash
HOST_REPO_PATH=/absolute/path/to/your/source
LLM_API_KEY=your-key-here
```

**Optional — OpenAI-compatible endpoint** (vLLM, LM Studio, OpenAI-compatible proxy, etc.). Omit these lines to use the app’s built-in defaults (Anthropic cloud when `LLM_PROVIDER` is unset):

```bash
HOST_REPO_PATH=/absolute/path/to/your/source
LLM_API_KEY=your-key-here
LLM_PROVIDER=openai_compatible
LLM_BASE_URL=http://your-endpoint:11434
LLM_MODEL=your-server-model-id
```

**Optional — explicit Anthropic cloud base** (only if you override; default is already correct):

```bash
HOST_REPO_PATH=/absolute/path/to/your/source
LLM_API_KEY=your-key-here
LLM_PROVIDER=anthropic
LLM_BASE_URL=https://api.anthropic.com/v1
LLM_MODEL=claude-sonnet-4-20250514
```

### Required variables

| Variable | Description |
|----------|-------------|
| **`HOST_REPO_PATH`** | **Absolute path** on your machine to the **source code** you want to scan. The container mounts it read-only at `/repo`. Example: `/Users/you/projects/my-app` (macOS/Linux) or `C:\path\to\my-app` (Windows—use the path format your Docker setup expects in `.env`). |
| **`LLM_API_KEY`** | API key for the LLM used by scan validation and related features. |

### Optional: LLM provider and endpoint

All of the following are **optional**. If you omit them, the agent uses its defaults (including default provider, base URL, and model). The published image reads these from the environment **inside the container** (Compose loads your `.env`).

| Variable | Default / behavior | Valid values and format |
|----------|-------------------|-------------------------|
| **`LLM_PROVIDER`** | **`anthropic`** when unset | **`anthropic`** or **`openai_compatible`**. Matched **exactly** (lowercase, underscores); any other value fails at runtime with an “unknown LLM provider” error. |
| **`LLM_BASE_URL`** | Anthropic: **`https://api.anthropic.com/v1`** when unset. **`openai_compatible`**: **`http://localhost:8080`** when unset (usually overridden). | **No automatic URL rewriting** — the value is used verbatim. **Anthropic:** the app posts to **`{LLM_BASE_URL}/messages`** (`x-api-key`). For Anthropic’s cloud, use **`https://api.anthropic.com/v1`** (the default). A value like **`https://api.anthropic.com/`** (no `v1` path) is not equivalent and can fail. **OpenAI-compatible:** the app posts to **`{LLM_BASE_URL}/v1/chat/completions`** (`Authorization: Bearer`). Use the server **origin** without a trailing **`/v1`** on the base, or the path becomes **`/v1/v1/chat/completions`**. |
| **`LLM_MODEL`** | Built-in default per provider | Any string your API accepts as `model` (e.g. Anthropic: `claude-sonnet-4-20250514`; OpenAI-compatible: `gpt-4o`, a Hugging Face id such as `mistralai/Devstral-Small-2-24B-Instruct-2512`, or whatever your server documents). |
| **`LLM_HTTP_TIMEOUT_SECS`** | `30` (minimum effective `10`) | Caps how long a single LLM HTTP call may block (connect + response body). |

**Docker networking:** If you set **`LLM_BASE_URL`**, it is resolved **from inside the container**. If the LLM runs on the same machine as Docker, `http://127.0.0.1:…` usually points at the **container itself**, not your host. Use a host-reachable name or IP (for example `http://host.docker.internal:11434` on Docker Desktop for Mac/Windows, or your LAN IP) unless the model server is in the same Compose stack on a service name.

### Optional: portal and scan behavior

| Variable | Default | Description |
|----------|---------|-------------|
| **`PORTAL_PORT`** | `8080` | Host port for the web UI (maps to the app’s port 80 inside the container). |
| **`SCAN_VALIDATE`** | on | Set to `0`, `false`, `no`, or `off` to run scans **without** LLM validation (`--no-validate`). Use only when your administrator approves offline or CI-only runs. |
| **`PROVE_FIX_TIMEOUT_SECS`** | `60` | Used by the **portal API** when it runs the **`prove-fix-json`** helper (generate / prove-fix flows from the web UI). Hard limit in seconds for that subprocess; if the run exceeds it, the portal returns a timeout and kills the child. Raise it (for example `300` or `1200`) if your administrator expects long-running fixes. |
| **`PROVE_FIX_SCANNER_LOG`** | `info` | Log verbosity for the **`prove-fix-json`** helper when the portal runs generate / prove-fix from the UI. Use common levels such as `error`, `warn`, `info`, or `debug` for more detail while troubleshooting. Your administrator may supply a richer filter string if needed. |

Your administrator may provide more variables (timeouts, logging, etc.). Use whatever they supply; names are documented in any **`env.example`** they share.

---

## 4. Log in to GHCR to access the private image

The QuestorAI container is private, so you must sign in to **`ghcr.io`** with a GitHub user or token that has permission to pull the image.

Create a token with **`read:packages`** (see [GitHub: Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)). For **fine-grained** tokens, grant read access to **Packages** for the Questor-AI org (or this repository) as your org allows.

Use **one** of the following login methods (you do **not** need both). In each case, use your **GitHub username** for `-u`, not your email.

**Option A — Personal access token (PAT)**  
Create a PAT with **`read:packages`** as above, then:

```bash
echo YOUR_GITHUB_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

Replace **`YOUR_GITHUB_USERNAME`** and **`YOUR_GITHUB_TOKEN`** with your account and token.

**Option B — GitHub CLI**  
If you use [GitHub CLI](https://cli.github.com/) and are already signed in with `gh auth login` (with scopes that can read packages—add `read:packages` to your `gh auth` scopes if pulls still fail), you can pipe the CLI’s token instead of managing a PAT by hand:

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
| `permission denied` when running `docker` in **WSL** | See **Windows: Docker CLI from WSL** under Requirements (add your Linux user to the `docker` group). |
| `docker pull` denied / 403 | Run `docker login ghcr.io` with a user and PAT that can **`read:packages`** for the Questor-AI org package. Confirm your GitHub account was granted access to the private image. |
| Error about `HOST_REPO_PATH` | Set it to a real **absolute** path on the host; the folder must exist before you start the container. |
| Port already in use | Set `PORTAL_PORT` in `.env` to a free port (for example `8081`). |
| Scan or LLM errors | Confirm required `HOST_REPO_PATH` and `LLM_API_KEY` are set. If you use optional LLM settings, confirm `LLM_PROVIDER`, `LLM_BASE_URL`, and `LLM_MODEL` match your endpoint (see **Optional: LLM provider and endpoint**). Typos in `LLM_PROVIDER` cause an unknown-provider error. For **Anthropic**, `LLM_BASE_URL` must be the Messages API root (typically **`https://api.anthropic.com/v1`**), not only **`https://api.anthropic.com/`**. For **OpenAI-compatible**, do not put **`/v1`** on the base URL alone or requests hit **`/v1/v1/chat/completions`**. The portal **Settings → LLM connection test** uses the same paths as the Rust agent (no rewriting). For offline use, your administrator may allow `SCAN_VALIDATE=0`. |

For questions about keys, network policy, or which tag to use, contact whoever provided the image or this Compose file.
