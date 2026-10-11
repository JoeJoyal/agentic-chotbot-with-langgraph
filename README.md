# Agentic AI Chabot with RAG, HITL & CI/CD Deployment on AWS EC2 

An end-to-end guide for running a Streamlit-based agentic chatbot
locally and deploying it with Docker, Docker Hub, GitHub Actions, and an
AWS EC2 self-hosted runner.

> **Project note:** This README describes the architecture and
> deployment approach discussed for this project. Verify filenames,
> environment variables, tool implementations, and data paths against
> the current repository before production use. The example Dockerfile
> assumes the Streamlit entry point is `app.py` and the backend module
> is `backend.py`; adjust these if your repository differs.

## Table of Contents

1.  [Project Overview](#1-project-overview)
2.  [Goals and Features](#2-goals-and-features)
3.  [Technology Stack](#3-technology-stack)
4.  [Hardware and Infrastructure](#4-hardware-and-infrastructure)
5.  [Architecture](#5-architecture)
6.  [Application Workflow](#6-application-workflow)
7.  [CI/CD Workflow](#7-cicd-workflow)
8.  [Repository Structure](#8-repository-structure)
9.  [Prerequisites](#9-prerequisites)
10. [Local Setup](#10-local-setup)
11. [Run Locally](#11-run-locally)
12. [Docker Setup](#12-docker-setup)
13. [Run with Docker](#13-run-with-docker)
14. [Docker Hub Configuration](#14-docker-hub-configuration)
15. [AWS EC2 Setup](#15-aws-ec2-setup)
16. [Self-Hosted Runner](#16-self-hosted-runner)
17. [GitHub Actions Secrets](#17-github-actions-secrets)
18. [Deployment and Verification](#18-deployment-and-verification)
19. [Persistence and Rollback](#19-persistence-and-rollback)
20. [Security](#20-security)
21. [Troubleshooting](#21-troubleshooting)
22. [Operational Checklist](#22-operational-checklist)
23. [Future Improvements](#23-future-improvements)

## 1. Project Overview

This project packages an agentic chatbot in a Docker image and automates
its build and deployment with GitHub Actions.

The application layer uses Python and Streamlit. The backend may use
LangGraph and LangChain for agent orchestration. Depending on the
modules enabled in the repository, integrations may include an LLM
provider, Tavily web search, an OpenWeatherMap weather tool, LangSmith
tracing, a FAISS vector index, and SQLite checkpointing.

The pipeline is designed to:

1.  Validate the source code.
2.  Build a Docker image.
3.  Push the image to Docker Hub.
4.  Run deployment commands on an AWS EC2 self-hosted GitHub Actions
    runner.
5.  Pull and launch the commit-specific image.
6.  Check Streamlit's health endpoint.
7.  Attempt to restore the previous image if the new deployment fails
    its health check.

**Deployment model:** the current approach is a single-instance
stop-and-replace deployment with a best-effort rollback. It is not
zero-downtime blue-green deployment.

## 2. Goals and Features

### Goals

-   Make the application reproducible through containerization.
-   Validate code before image publishing.
-   Automate image build and push.
-   Deploy a known commit to EC2.
-   Keep credentials out of source code.
-   Check application health after deployment.
-   Provide a basic rollback path.

### Features

-   **Streamlit UI:** browser-based chat interface.
-   **Agent orchestration:** LangGraph/LangChain components, if enabled.
-   **LLM integration:** configured provider API.
-   **Optional tools:** web search, weather, calculator, stock, or other
    tools only if implemented and registered by the application.
-   **Optional retrieval:** FAISS or another configured retrieval
    component.
-   **Optional persistence:** SQLite conversation/checkpoint storage.
-   **Optional observability:** LangSmith tracing.
-   **Docker packaging:** reproducible application image.
-   **CI checks:** Python syntax, Ruff linting, and tests when present.
-   **Image publishing:** Docker Hub tags for `main` and the commit SHA.
-   **Automated deployment:** EC2 runner pulls and starts the image.
-   **Health verification:** checks `/_stcore/health` on port `8501`.
-   **Basic rollback:** attempts to restart the previous image after a
    failed health check.

Verify each optional feature against the actual source code before
describing it as enabled in production.

## 3. Technology Stack

  Component          Purpose
  ------------------ ---------------------------------------------------
  Python             Application language
  Streamlit          Web UI
  LangGraph          Agent workflow/state orchestration, if used
  LangChain          Model, tool, and retrieval integrations, if used
  LLM provider API   Language-model responses
  Tavily             Web search, if enabled
  OpenWeatherMap     Weather integration, if enabled
  FAISS              Local vector retrieval, if configured
  SQLite             Local checkpoint/state persistence, if configured
  LangSmith          Optional tracing
  Docker             Container packaging and runtime
  Docker Hub         Container registry
  GitHub Actions     CI/CD automation
  AWS EC2 Ubuntu     Deployment host and self-hosted runner
  Ruff               Python linting
  pytest             Test execution
  Git/GitHub         Version control and workflow triggers

Use tested and preferably pinned dependency versions in
`requirements.txt`.

## 4. Hardware and Infrastructure

### Development machine

A reasonable starting point for development is:

-   2--4 CPU cores.
-   8 GB RAM or more.
-   Sufficient disk for virtual environments, Docker images, documents,
    and indexes.
-   Windows, Linux, or macOS.
-   Network access for dependency installation and external APIs where
    required.

These are planning suggestions, not strict minimums or performance
guarantees. A GPU is generally not required when inference uses hosted
model APIs. Local model inference or local embedding/reranking may
require additional RAM, GPU memory, and storage.

### AWS EC2 host

Select the instance type after measuring workload requirements. Plan
for:

-   Ubuntu Server.
-   Docker Engine.
-   Disk space for images, logs, and persistent data.
-   Outbound network access to Docker Hub, GitHub runner services, and
    required APIs.
-   A registered self-hosted runner.
-   Security-group rules for intended administration and application
    traffic.
-   A host patching, backup, and recovery process.

### Ports

-   `8501/tcp`: Streamlit inside the container and optionally on the
    host.
-   `22/tcp`: SSH administration; restrict to trusted source addresses.
-   `443/tcp`: recommended HTTPS entry point when using a reverse proxy
    or load balancer.

Avoid exposing Docker sockets, databases, or runner credentials to the
network. Prefer HTTPS through a reverse proxy or load balancer rather
than exposing Streamlit directly.

## 5. Architecture

### Logical application architecture

``` mermaid
flowchart TD
    U[User Browser] -->|HTTP / HTTPS| UI[Streamlit UI :8501]
    UI --> B[Python Backend]
    B --> A[Agent Workflow / LangGraph]
    A --> L[LLM Provider API]
    A --> T[Configured Tools]
    A --> R[Retriever / FAISS]
    A --> DB[SQLite Checkpoint]
    A --> O[LangSmith Tracing]
```

This is a logical diagram. Optional components may not be enabled in
every code version.

### Deployment architecture

``` mermaid
flowchart TD
    DEV[Developer] -->|Push to main| GH[GitHub Repository]
    GH --> CI[GitHub Actions CI]
    CI -->|Success| BUILD[Build Docker Image]
    BUILD --> PUSH[Push Image to Docker Hub]
    PUSH --> RUNNER[EC2 Self-hosted Runner]
    RUNNER --> PULL[Pull Commit-specific Image]
    PULL --> START[Start Streamlit Container]
    START --> HEALTH{Health Check Passes?}
    HEALTH -->|Yes| LIVE[Application Running]
    HEALTH -->|No| ROLLBACK[Attempt Previous Image Rollback]
```

## 6. Application Workflow

A typical agentic-chat request lifecycle is:

1.  The user opens the Streamlit application.
2.  The user submits a message.
3.  The UI passes the message and conversation state to the backend.
4.  The backend invokes the configured agent graph or LLM logic.
5.  Depending on the request and implementation, the agent may call
    registered tools or a retriever.
6.  The backend returns a response.
7.  Streamlit displays the result.
8.  If configured, checkpointing and tracing record state or execution
    details.

The exact graph nodes, routing logic, tools, state schema, and error
handling must be confirmed in the code.

## 7. CI/CD Workflow

The pipeline has three jobs:

### Job 1 --- `integration`

-   Checkout the repository.
-   Set up Python 3.11.
-   Install Ruff, pytest, and project dependencies.
-   Run `python -m compileall -q .`.
-   Run `ruff check .`.
-   Run pytest when matching test files exist.

If linting or syntax checks fail, image publishing does not proceed.
Skipping tests when no tests exist is not the same as passing a
meaningful test suite.

### Job 2 --- `push_to_registry`

Runs after CI succeeds:

-   Validate Docker Hub secrets.
-   Configure Docker Buildx.
-   Log in to Docker Hub.
-   Build the Dockerfile.
-   Push tags:
    -   `username/image:main`
    -   `username/image:sha-<commit-sha>`
-   Use GitHub Actions build cache.

The commit-specific tag makes it possible to identify the image
associated with a source revision.

### Job 3 --- `continuous_deployment`

Runs after image publishing succeeds:

-   Execute on the EC2 self-hosted runner.
-   Validate required deployment secrets.
-   Log in to Docker Hub.
-   Pull the image for the current commit.
-   Start the container with runtime environment variables.
-   Poll Streamlit's `/_stcore/health` endpoint.
-   Print logs and attempt rollback if the new image fails health
    checks.
-   Prune dangling images.

The workflow replaces the container named `stapp`, so a brief
interruption is possible. Rollback is best-effort, not a guarantee.

## 8. Repository Structure

Example layout---adjust to match the actual repository:

``` text
agentic-chatbot/
├── .github/
│   └── workflows/
│       └── cicd.yaml
├── app.py
├── backend.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── .gitignore
├── README.md                 
└── .env                  # Local only; never commit
```

If your Streamlit entry point is named differently, update the
Dockerfile `CMD` and these instructions.

## 9. Prerequisites

Prepare:

1.  Git.
2.  Python supported by the project (the example CI uses Python 3.11).
3.  `venv` or Conda.
4.  Docker Desktop or Docker Engine.
5.  A GitHub repository with Actions enabled.
6.  A Docker Hub account and repository.
7.  An AWS account and Ubuntu EC2 instance.
8.  A GitHub self-hosted runner installed on EC2.
9.  API credentials for the services actually used by the code.

Check installations:

``` bash
git --version
python --version
docker --version
```

## 10. Local Setup

### 10.1 Clone the repository

``` bash
git clone https://github.com/JoeJoyal/agentic-chotbot-with-langgraph.git
cd agentic-chotbot-with-langgraph/
```

Replace both placeholders with your actual repository URL and folder.

### 10.2 Create a virtual environment

Windows PowerShell:

``` powershell
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

Linux/macOS:

``` bash
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

### 10.3 Install dependencies

``` bash
python -m pip install -r requirements.txt
```

If `requirements.txt` is missing, create it from the dependencies
actually used by the project and test the resulting environment.

### 10.4 Configure environment variables

Create a local `.env` file where the application expects it:

``` dotenv
# Populate only the variables used by your code.
OPENAI_API_KEY=
GOOGLE_API_KEY=
TAVILY_API_KEY=
OPENWEATHER_API_KEY=

LANGSMITH_TRACING=false
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
LANGSMITH_API_KEY=
LANGSMITH_PROJECT=agentic-chatbot-project
```

Use valid credentials locally. Keep `.env` out of version control. The
application must explicitly load the file (for example, with
`python-dotenv`) or receive variables from the shell; creating the file
alone does not automatically load it.

## 11. Run Locally

From the project root with the virtual environment active:

``` bash
streamlit run app.py
```

Replace `app.py` if the entry point has a different name. Open the local
URL printed by Streamlit, typically:

``` text
http://localhost:8501
```

Check that the UI loads, a basic chat request succeeds, configured tools
work, missing keys produce understandable errors, and any required local
data files can be found.

## 12. Docker Setup

Create a file named `Dockerfile` in the repository root. **Do not
include Markdown code fences in the actual Dockerfile.**

Example template:

``` dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

RUN python -m pip install --no-cache-dir --upgrade pip \
    && python -m pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8501

CMD ["streamlit", "run", "app.py", "--server.address=0.0.0.0", "--server.port=8501"]
```

Adapt this template:

-   Change `app.py` if the Streamlit entry point differs.
-   Add system libraries only if dependencies require them.
-   Ensure `requirements.txt` contains the imports used by the
    application.
-   Never bake API keys into the Dockerfile or image.
-   Do not copy local secrets into the image.

### `.dockerignore`

Starter example:

``` text
.git
.github
.venv
venv
__pycache__
*.py[cod]
.env
.env.*
!.env.example
```

Add local caches and generated data according to your needs. Do not
exclude required application files unless they are provided through a
deliberate runtime mount or another mechanism.

### Build the image

``` bash
docker build -t agentic-chatbot:local .
```

A successful build confirms that Docker can build the image; it does not
guarantee that runtime APIs, credentials, or data paths are correct.

## 13. Run with Docker

Pass credentials at runtime rather than during image build. Example
using PowerShell:

``` powershell
docker run --rm -it `
  -p 8501:8501 `
  --env-file .env `
  --name agentic-chatbot-local `
  agentic-chatbot:local
```

Open `http://localhost:8501`.

If the app depends on a local FAISS index, SQLite database, or uploaded
documents, mount the appropriate directories as persistent volumes.
Inspect the code to determine the actual paths before adding mounts.

Stop a foreground container with `Ctrl+C`. For a detached container,
use:

``` bash
docker stop agentic-chatbot-local
```

## 14. Docker Hub Configuration

1.  Sign in to Docker Hub.
2.  Create a repository, for example `agentic-chatbot`.
3.  Create an access token with the minimum permissions needed to push
    images.
4.  Add the token to GitHub Actions as `DOCKER_PASSWORD`.
5.  Set `DOCKER_USERNAME` to your Docker Hub username.
6.  Set `IMAGE_NAME` to the repository name only, for example
    `agentic-chatbot`.

The workflow builds references like:

``` text
<DOCKER_USERNAME>/<IMAGE_NAME>:main
<DOCKER_USERNAME>/<IMAGE_NAME>:sha-<GITHUB_COMMIT_SHA>
```

Never paste access tokens into chat, commit them to Git, or print them
in CI logs. Revoke any exposed token and replace it.

## 15. AWS EC2 Setup

### 15.1 Launch an instance

1.  Open the AWS console and navigate to EC2.
2.  Launch an Ubuntu Server instance.
3.  Choose an instance type and disk size based on workload testing.
4.  Configure a key pair for SSH administration.
5.  Attach an IAM instance profile if the host needs AWS API access.
6.  Configure a security group:
    -   SSH (`22`) only from trusted administration IPs.
    -   Application traffic only from intended clients or a secured
        front end.
    -   Avoid opening all ports to the internet.

### 15.2 Install Docker

Connect over SSH and follow Docker's installation instructions for your
Ubuntu release. A convenience installation method is:

``` bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker ubuntu
```

Reconnect or start a fresh login session, then verify:

``` bash
docker --version
docker ps
```

The convenience script is suitable for learning and some test
environments. For production, review the source and define a controlled
update process. Docker-group membership effectively grants root-level
privileges.

The EC2 host needs outbound access to Docker Hub, GitHub runner
services, and external APIs used by the application.

## 16. Self-Hosted Runner

In the GitHub repository:

1.  Open **Settings → Actions → Runners**.
2.  Select **New self-hosted runner**.
3.  Choose Linux and the correct architecture.
4.  Follow the exact download and configuration commands shown by
    GitHub.
5.  Start the runner and confirm it appears online.
6.  Configure it as a service using GitHub's current instructions if
    required.
7.  Confirm the runner account can run Docker commands.

The workflow uses `runs-on: self-hosted`, so an online runner with
matching labels is required. A self-hosted runner is a privileged
execution environment: limit repository access, review workflow changes,
and do not run untrusted pull-request code on a production runner.

## 17. GitHub Actions Secrets

Go to **Settings → Secrets and variables → Actions → New repository
secret**.

  Secret                  Purpose
  ----------------------- ----------------------------------
  `DOCKER_USERNAME`       Docker Hub username
  `DOCKER_PASSWORD`       Docker Hub access token
  `IMAGE_NAME`            Docker Hub repository name
  `OPENAI_API_KEY`        OpenAI API, if used
  `GOOGLE_API_KEY`        Google/Gemini API, if used
  `TAVILY_API_KEY`        Tavily search, if used
  `OPENWEATHER_API_KEY`   Weather tool, if used
  `LANGSMITH_TRACING`     Tracing toggle
  `LANGSMITH_ENDPOINT`    LangSmith endpoint
  `LANGSMITH_API_KEY`     LangSmith credential, if enabled
  `LANGSMITH_PROJECT`     LangSmith project label

The example workflow validates `GOOGLE_API_KEY`, `TAVILY_API_KEY`, and
`OPENWEATHER_API_KEY` as required. If any integration is optional,
update the validation logic to match the actual app.

The described workflow does not need `AWS_ACCESS_KEY_ID`,
`AWS_SECRET_ACCESS_KEY`, or `AWS_REGION` merely to run Docker commands
on EC2. Add them only if another job or component truly calls AWS APIs.
Prefer an EC2 instance role over long-lived AWS access keys where
possible. Do not grant `AmazonEC2FullAccess` automatically just to
deploy a Docker container.

## 18. Deployment and Verification

### 18.1 Push changes

Before staging files, inspect the status and make sure `.env`,
credentials, local databases, and private datasets are not included.

``` bash
git status
git add .
git commit -m "Update agentic chatbot"
git push origin main
```

### 18.2 Monitor GitHub Actions

Open the repository's **Actions** tab and inspect the workflow run.
Verify that:

1.  `integration` passes.
2.  `push_to_registry` builds and pushes the image.
3.  `continuous_deployment` executes on the EC2 self-hosted runner.

### 18.3 Verify Docker Hub

Confirm that the image has the expected `main` and `sha-<commit-sha>`
tags.

### 18.4 Verify the EC2 container

``` bash
docker ps
docker logs --tail 100 stapp
```

Check Streamlit's health endpoint from inside the container:

``` bash
docker exec stapp python -c \
"import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:8501/_stcore/health', timeout=5).read().decode())"
```

### 18.5 Open the app

If the security group intentionally allows direct access, use:

``` text
http://<EC2-PUBLIC-DNS-OR-IP>:8501
```

For production, prefer HTTPS through a reverse proxy or load balancer,
and keep the Streamlit port private where possible.

## 19. Persistence and Rollback

### Persistent data

When a container is removed, its writable layer is removed too. If the
application stores any of the following inside the container, they can
be lost during deployment:

-   FAISS index files.
-   SQLite checkpoint databases.
-   Uploaded PDFs.
-   User-generated content.
-   Other local state that must survive restarts.

Store important files in persistent Docker volumes or carefully managed
host directories mounted at the actual paths used by the application.
Back up data and test restoring it. Do not guess mount paths; inspect
the source and runtime configuration first.

### Rollback

The sample workflow remembers the previous image reference, starts the
new image, and checks health. If health verification fails, it attempts
to start the old image.

Rollback is best-effort and may fail if the old image is unavailable,
configuration has changed, the host lacks resources, or data/schema
changes are incompatible. Retain known-good images and test rollback
before relying on it.

## 20. Security

-   Never commit `.env` files or API credentials.
-   Revoke and replace any credential that has been exposed.
-   Store pipeline credentials in GitHub Actions secrets.
-   Use a Docker Hub token with minimum required permissions.
-   Prefer EC2 instance roles for AWS access.
-   Restrict SSH to trusted addresses.
-   Avoid exposing port `8501` publicly unless necessary.
-   Use HTTPS and an access-control layer for production.
-   Keep Ubuntu, Docker, Python dependencies, and actions updated.
-   Review third-party GitHub Actions and pin versions according to your
    supply-chain policy.
-   Do not run untrusted workflows on a production self-hosted runner.
-   Do not print secrets in logs.
-   Back up persistent data and monitor disk usage, logs, resource use,
    and API costs.

## 21. Troubleshooting

### Dockerfile parse error: ```` unknown instruction: ```dockerfile ````

**Cause:** Markdown fences were copied into the actual Dockerfile.

**Fix:** Remove the opening and closing triple-backtick lines. The
actual file must start with a Docker instruction such as
`FROM python:3.11-slim`.

### Docker Hub login fails

Check that `DOCKER_USERNAME` matches the account, `DOCKER_PASSWORD` is a
valid access token, the token has push permission, and the GitHub secret
was updated after rotation or revocation. Do not paste tokens into logs
or support messages.

### Ruff fails

Run locally:

``` bash
ruff check .
ruff check --fix .
ruff check .
```

Automatic fixes only handle supported issues. Review remaining warnings
rather than disabling every rule.

### Tests fail or are skipped

Confirm test files use names such as `test_example.py` or
`example_test.py`. If no tests are found, the sample workflow skips
pytest; this does not demonstrate that the application has adequate test
coverage.

### Streamlit health check fails

Inspect:

``` bash
docker logs --tail 100 stapp
```

Check the entry-point filename, dependencies, required environment
variables, Streamlit bind address (`0.0.0.0` inside the container), port
mapping, and startup errors.

### Container runs but the UI is unreachable

Check:

``` bash
docker ps
docker port stapp
```

Then inspect security-group rules, host firewall settings, network
routing, and the address used in the browser.

### FAISS or SQLite data disappears

The data may have been stored only in the old container's writable
layer. Configure persistent mounts at the real application paths and
restore from backup if necessary. Do not delete data directories while
troubleshooting.

### Self-hosted runner is offline

Check the runner service using the commands appropriate to your
installation, confirm it is online in GitHub, and verify Docker
permissions for the runner account.

### Local build succeeds but CI build fails

Compare Python versions, Dockerfile instructions, dependencies, required
files, and `.dockerignore` behavior. Confirm that all required source
files are committed.

## 22. Operational Checklist

-   [ ] Correct Streamlit entry point configured.
-   [ ] `.env` and credentials excluded from Git.
-   [ ] Docker image builds locally.
-   [ ] Syntax and Ruff checks pass.
-   [ ] Meaningful tests exist and pass.
-   [ ] Docker Hub secrets are configured.
-   [ ] Commit-specific image is published.
-   [ ] EC2 Docker and self-hosted runner are healthy.
-   [ ] Deployment completes successfully.
-   [ ] Streamlit health endpoint responds.
-   [ ] Required API integrations work in the deployed container.
-   [ ] FAISS, SQLite, and uploaded data have persistence where needed.
-   [ ] Network access is secured.
-   [ ] Backup and rollback procedures have been tested.

## 23. Future Improvements

-   **Blue-green deployment:** run the new version alongside the old
    one, verify it, and switch traffic only when healthy.
-   **HTTPS:** configure Nginx, Caddy, or an AWS load balancer with TLS.
-   **Monitoring:** add resource alerts, centralized logs, and
    deployment notifications.
-   **Testing:** add unit tests for routing, tools, retrieval, and error
    handling, plus integration tests for key user journeys.
-   **Image security:** scan images and dependencies for
    vulnerabilities.
-   **Immutable releases:** deploy by image digest and retain known-good
    versions.
-   **Secrets management:** use managed secrets and role-based access
    where appropriate.
-   **Data durability:** back up local indexes/databases and test
    recovery.
-   **Availability:** consider multiple instances and a load balancer if
    uptime requirements justify the complexity.
-   **Performance:** measure CPU, memory, latency, and API usage before
    resizing EC2.

------------------------------------------------------------------------

## Summary

The deployment pipeline connects source control, CI checks, container
image publishing, and EC2 deployment:

**Git push → CI → Docker build → Docker Hub push → EC2 pull → container
launch → health check → rollback attempt if needed.**
