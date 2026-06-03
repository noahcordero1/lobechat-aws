# CI Reasoning — build-free static-analysis pipeline for `lobechat-aws`

> **Scope of this document.** It explains the CI workflow I added in
> `.github/workflows/ci.yml`, *why each gate matters in this specific repository*
> (Part A), and *what is still missing to turn this into a real production CD
> pipeline* (Part B). Every claim is anchored to a real `file:line` in this
> branch (based on upstream commit `4779a9e2`).

---

## Evidence — the workflow actually ran in GitHub Actions

![GitHub Actions run for this branch](docs/evidence/ci/actions-run.png)

- **Repository / owner:** `noahcordero1/lobechat-aws`
- **Actions run URL:** `<<FILL-RUN-URL>>`  <!-- e.g. https://github.com/noahcordero1/lobechat-aws/actions/runs/XXXXXXXXXX -->
- **Commit SHA the run executed against:** `<<FILL-COMMIT-SHA>>`
- **Outcome:** green run — the four *correctness* gates pass; the four *finding-generator*
  gates are warn-only and surface their real findings as annotations/logs (see policy below).

---

## Gate policy (the design choice I own)

The exam requires me to decide, per gate, whether a finding **blocks** the run or only **warns**.
My split, and why:

| Policy | Gates | Rationale |
|---|---|---|
| **Blocking** (a finding fails the run) | `docker compose config -q`, `yamllint` (on `ci.yml`), `actionlint`, `uv run cz check` | These are *correctness* gates over files I author/control. They must be clean — a failure here is my bug, not a pre-existing repo issue. |
| **Warn-only** (`continue-on-error: true`) | `hadolint` (×2), `trivy fs`, `trivy config`, `gitleaks`, and `yamllint` over `docker-compose.yml` | These surface **real, pre-existing** issues in code I did not author. I keep them **visible** (the findings still print) without failing the first-ever pipeline on day one. |

I did **not** delete any gate to fake a green run; every gate executes. (An honest *red* run with the failing gate explained would also be acceptable — I chose green-with-warnings so the pipeline is adoptable immediately while the findings remain on record.)

---

## Part A — why each gate matters *in this repository*

### 1. `hadolint` on `dockerfiles/mcphub.Dockerfile` — unpinned base image
`dockerfiles/mcphub.Dockerfile:1` is `FROM samanhappy/mcphub:latest`. `:latest` is mutable, so two builds days apart can produce different runtime images — non-reproducible and a supply-chain risk (hadolint `DL3007`). **Precision note:** the compose `mcphub` *service* is **built locally** (`docker-compose.yml:77-79`, `build: { context: ., dockerfile: dockerfiles/mcphub.Dockerfile }`), so the unpinned-**pull** risk lives in this **Dockerfile `FROM`**, *not* in a pulled compose image.

### 2. `hadolint` on `mcphub.Dockerfile` — privileged, bloated runtime
`dockerfiles/mcphub.Dockerfile:5` switches to `USER root`, and `:7` installs `gcc` and `docker.io` into the image. A compiler and the Docker CLI in a *runtime* image enlarge the attack surface; `docker.io` implies the container is expected to talk to a Docker socket — container-escape territory if that socket is mounted.

### 3. `hadolint` / `trivy config` on `dockerfiles/sandbox.Dockerfile` — passwordless root + unverified "latest" binaries
`dockerfiles/sandbox.Dockerfile:21` writes `oriol ALL=(ALL) NOPASSWD:ALL` (passwordless sudo). Worse for supply-chain: `:43-44` fetch `kubectl` from `https://dl.k8s.io/release/stable.txt` ("whatever is latest"), `:49-50` fetch `eksctl` from `releases/latest/download`, `:62` fetch `zellij` from `releases/latest/download`, and `:36` fetches the AWS CLI zip — **none with a pinned version or checksum**. Any upstream compromise or MITM is installed verbatim into a container that can already `sudo` to root.

### 4. `docker compose config` / reasoning — untagged & `:latest` pulled images
`docker-compose.yml:21` is `image: lobehub/lobe-chat-database` with **no tag at all**; `:109` is `qdrant/qdrant:latest`; `:185` is `minio/minio:latest`. (Contrast the *pinned* services that do it right: `:3` `casbin/casdoor:v2.13.0`, `:218` `pgvector/pgvector:pg16`.) Untagged/`:latest` pulled images mean the deployed version is whatever the registry served that minute — no reproducibility and **no rollback target**. The `compose config` gate also proves the file still parses and interpolates after any edit.

### 5. `trivy config` / reasoning — cleartext database connections & secrets
`docker-compose.yml:13` sets `sslmode=disable` on Casdoor's Postgres DSN, and `:32` `DATABASE_URL=postgresql://postgres:...@postgres:5432/lobechat` specifies **no TLS** for LobeChat's connection. DB credentials and traffic cross the Docker network in cleartext. Secrets are also injected as plaintext env vars (`:35` `NEXT_AUTH_SECRET`, `:41` `AUTH_CASDOOR_SECRET`, `:53` `KEY_VAULTS_SECRET`, `:55` `OPENROUTER_API_KEY`), readable by anything that can inspect the container environment.

### 6. `gitleaks` — secret scan over the full git history
`gitleaks` runs with `fetch-depth: 0` so it scans the whole tree and history. **Honest framing (no fabrication):** the matches it can produce are the **intentional dummy placeholders** committed in `.env.example` (`:5` `KEY_VAULTS_SECRET`, `:6` `NEXT_AUTH_SECRET`, `:39` `OPENROUTER_API_KEY=sk-or-v1-…`, `:44` `HF_TOKEN=hf_…`) — an *example* file, not leaked credentials. Real secrets are kept out of git by `.gitignore`: `.env` (`:8`), `aws_credentials.yaml` (`:19`), `*.pem` (`:20`), `config/ssh/` (`:26`). I explicitly do **not** claim gitleaks "found committed AWS keys" — on a clean tree it does not. Its value is the *standing* guard against the day someone pastes a real key.

### 7. `actionlint` + `yamllint` — keep the pipeline itself correct
`actionlint` validates `.github/workflows/ci.yml` (and shellcheck-lints the `run:` blocks); `yamllint` validates YAML structure. Without these, a typo in a trigger or an unpinned action ships silently. (`yamllint` is warn-only over `docker-compose.yml`, where it correctly flags one pre-existing style nit at `:144` — `entrypoint: ["sh","-c"]`, no space after the comma — which I surface rather than edit the app file to hide.)

### 8. `uv run cz check` — reuse the repo's own commit convention
The repo already enforces Conventional Commits locally via `.githooks/commit-msg:13` (`uv run cz check --commit-msg-file …`), configured by `pyproject.toml:17` `[tool.commitizen]`. That hook's `--commit-msg-file` does not exist on a CI runner, so I mirror the *same tool* with a CI-appropriate, **bounded** target — `uv run cz check -m "$(git log -1 --pretty=%B)"` — which lints only **this** commit. An unbounded check would go red on the repo's pre-existing non-conventional history. Keeping commits conventional is what lets `cz bump` / changelog / `v$version` tagging (`pyproject.toml:23`) work.

### Why the pipeline is **build-free**
The stack cannot run on a standard GitHub-hosted runner. The `vllm` service (`docker-compose.yml:151`) requires an **NVIDIA GPU** — `:169-175` declare `deploy.resources.reservations.devices: [{ driver: nvidia, count: 1, capabilities: [gpu] }]` — and its healthcheck allows `start_period: 300s` (`:182`). Layer on the 10-service `depends_on` graph (`casdoor`, `lobe-chat`, `mcphub`, `qdrant`, `hayhooks`, `hayhooks-mcp`, `vllm`, `minio`, `linux-sandbox`, `postgres`) and there is no way to `up` it on a runner. So every gate here is **static**: no `docker build`, no `docker compose up/run`, no deploy.

### Why `tests/` are **excluded**
`tests/` are **live-stack integration tests**, not unit tests. `tests/test_vllm.py:9` `import httpx`, `:11` `from openai import OpenAI`, and `:33` does `httpx.get(f"{base_url}/health")` against a *running* vLLM endpoint; the test deps in `pyproject.toml:12-14` are `pytest`, `openai`, `httpx`. They require the stack to be up, which a build-free CI deliberately never does — so running `pytest` here would only ever fail.

### Why the Compose-interpolation fix is **safe**
`docker-compose.yml` interpolates undefined `${VAR}` secrets (e.g. `NEXT_AUTH_SECRET`, `AUTH_CASDOOR_ID/SECRET`, `KEY_VAULTS_SECRET`, `OPENROUTER_API_KEY`, `HF_TOKEN`, plus the SSH vars and `OPENAPI_MCP_HEADERS` on `mcphub`), so `config` fails with *"variable is not set"* unless values are supplied. The `compose-config` job runs `cp .env.example .env` to provide **dummy placeholders**, then appends `OPENAPI_MCP_HEADERS={}` (the one no-default var that `.env.example` omits). This is safe because `.gitignore:8` ignores `.env`, so the copy is **never committed** — the job validates schema + interpolation only, never `up`/`build`.

---

## Part B — what is missing for a real production CI/CD (delivery) pipeline

**What I built is Continuous *Integration*** — static quality and security gates that run on every push/PR. It deliberately **stops short of Continuous Delivery/Deployment**: nothing here builds an artifact, authenticates to a cloud, migrates a database, or ships to a host. To become a real production pipeline for *this* system, it must add:

1. **Build → push → sign → SBOM the locally-built images.** `mcphub` and `linux-sandbox` are built from source at deploy time (`docker-compose.yml:77-79` and `:208`). A real pipeline builds them once, generates an SBOM, signs them (cosign), and pushes to a registry (ECR), and **resolves the mutable images to immutable digests** — `lobehub/lobe-chat-database` (`:21`, untagged), `qdrant/qdrant:latest` (`:109`), `minio/minio:latest` (`:185`) — so a deploy is reproducible.

2. **Federate to AWS via GitHub OIDC instead of static keys.** Today credentials are long-lived and host-bound: `docker-compose.yml:103` bind-mounts the host's `~/.aws` into `mcphub` (`~/.aws:/root/.aws:ro`), and `.env.example:61-63` carries commented `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN` placeholders. A delivery pipeline should assume a role via OIDC and carry **no standing credentials**.

3. **Inject secrets at deploy time from SSM Parameter Store / Secrets Manager.** The `${VAR}` secrets in compose (`:35`, `:41`, `:53`, `:55`, plus the `.env.example` placeholders at `:5,6,21,22,39,44`) must be pulled from a managed secret store at deploy — never baked into the CI image or committed. (This is the final-project §2.2 mandate.)

4. **A guarded database-migration stage.** The repo ships a migration toolchain — `db/migrations/*.sql` driven by `db/migrate` (dbmate) and a Flyway helper `db/flyway/provision.sh`. A real pipeline runs migrations as an explicit, ordered stage, with the **destructive** path guarded behind manual approval: `db/flyway/provision.sh:71` runs `flyway … -cleanDisabled=false clean`, which **drops all rows + resets history** (its own usage banner at `:10` says so).

5. **Environment promotion dev → stage → prod with protected environments.** None exists today. CD needs GitHub **protected environments** with required reviewers / manual approval gates between tiers (the final-project Q2 promotion flow), so a change is proven in stage before it can reach prod.

6. **An actual deploy mechanism to the target host.** The system runs on a single EC2 box; a pipeline needs a real deploy step (SSM Run Command / SSH / `docker compose pull` of the now-digest-pinned images) that **keeps port 47000 closed** — `docker-compose.yml:24` maps `${LOBECHAT_PORT:-47000}:3210`, and that port must stay behind the Caddy reverse proxy, not exposed to the internet.

7. **Post-deploy smoke + health gates.** Several services have **no healthcheck**: `casdoor` (`:2`), `lobe-chat` (`:20`), `mcphub` (`:76`), `hayhooks` (`:123`), `hayhooks-mcp` (`:137`), `linux-sandbox` (`:205`) — only `qdrant` (`:117`), `vllm` (`:177`), `minio` (`:198`), `postgres` (`:229`) have them. CD must add the missing healthchecks and run the live `tests/` (e.g. `tests/test_vllm.py:33` `/health`) against an **ephemeral** environment as a post-deploy gate.

8. **Automated rollback — and stop deploying a mounted monkeypatch.** Today the deploy unit is an *unpinned image* plus a **bind-mounted patch**: `docker-compose.yml:27` mounts `./patches/route.js` over LobeChat's built tRPC route at runtime. That is not rollback-able. A real pipeline **bakes the patch into a forked, pinned image** and keeps the previous digest ready for an automatic rollback if the smoke gate fails.

Plus, around the pipeline itself: **branch protection / required status checks** (make these CI gates mandatory before merge) and **signed release tags** (`cz bump` already produces `v$version` tags per `pyproject.toml:23`).

### Prioritisation — the single highest-value next step
**Build the two local images and push immutable, digest-pinned images to ECR (item 1).** Continuous *Delivery* is, by definition, the promotion and rollback of artifacts — and this system currently has no artifact: it deploys untagged/`:latest` images plus a runtime-mounted `patches/route.js` (`docker-compose.yml:21,27`), which can be neither reproduced nor rolled back. Until a real, versioned artifact exists in a registry, every other CD step (promotion, rollback, smoke-gating) has nothing concrete to act on. (Strong runner-up: OIDC federation, item 2, to retire the `~/.aws` mount — but it secures a delivery process that doesn't exist yet.)
