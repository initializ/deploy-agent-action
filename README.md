# deploy-agent-action

GitHub Actions to deploy **forge**, **claude-agent**, and **strands** agents to the
initializ platform. Two composite actions:

| Action | Path | What it does |
| --- | --- | --- |
| **build-governed-image** | `initializ/deploy-agent-action/build-governed-image@v1` | Builds a claude-agent/strands agent's own image, generates the governance Dockerfile inline, wraps + pushes the **governed** image. |
| **deploy** | `initializ/deploy-agent-action/deploy@v1` | Installs the released `initializ` CLI and deploys a pushed image (`initializ agent deploy`). Identical for all three frameworks. |

The **framework is carried by `agent.type` in `initializ-deploy.yaml`** (`forge` |
`claude-agent` | `strands`), so the deploy step never branches on it. Only the
*build* differs — see below.

## Prerequisites

To deploy to the initializ platform you need:

1. **A platform access token** — mint one in the console or via `api-next POST /tokens`
   (role `developer` or `workspace_admin`). Store it as a GitHub **secret**
   (`INITIALIZ_TOKEN`); an **org** secret if several repos deploy.
2. **Your platform endpoints** — `api-url` (agent-builder ingress) and `auth-url`
   (api-next ingress) for your environment. There is no default; you always name them.
3. **Org / workspace ids** — `INITIALIZ_ORG_ID`, and `INITIALIZ_WORKSPACE_ID` unless
   `agent.workspace` is set in the spec.
4. **A container registry you can push to** (GHCR or ECR — see below) that the
   platform's cluster can pull from.
5. **`claude-agent` / `strands` only:** read access to the governance pkg image
   (`ghcr.io/initializ/claude-agent-pkg` / `strands-pkg`) so `build-governed-image`
   can wrap your image. Pin its `:vX.Y.Z` to pin the SDK.
6. An `initializ-deploy.yaml` in the agent repo (and `forge.yaml` for forge agents).

## Where the image is pushed (registry)

`build-governed-image` pushes to whatever the **`image`** input names, using either
its own login (`registry`/`username`/`password`) or a `docker login` you did in an
earlier step. So the same action targets any registry — see the examples:

| Registry | How you authenticate | `image` looks like | Example |
| --- | --- | --- | --- |
| **GitHub (GHCR)** | `GITHUB_TOKEN` with `packages: write` (the action logs in) | `ghcr.io/<org>/<repo>` | [`examples/strands.yml`](examples/strands.yml) |
| **AWS ECR** | `aws-actions/configure-aws-credentials` + `amazon-ecr-login` **before** the action (leave `password` empty) | `<acct>.dkr.ecr.<region>.amazonaws.com/<repo>` | [`examples/ecr.yml`](examples/ecr.yml) |

> ECR does not auto-create repositories — create the ECR repo once (console / Terraform /
> `aws ecr create-repository`) before the first push.

## Per-framework flow

| | forge | claude-agent | strands |
| --- | --- | --- | --- |
| Build | `forge build` (image + `.forge-output`) | `docker build` your Dockerfile | `docker build` your Dockerfile |
| Govern | — (built-in) | wrap with `claude-agent-pkg` + `NODE_OPTIONS --import` | wrap with `strands-pkg` wheel (`.pth` auto-activation) |
| Deploy | `deploy` action | `deploy` action | `deploy` action |

- **forge** → skip `build-governed-image`; build with your forge toolchain, then `deploy`. See [`examples/forge.yml`](examples/forge.yml).
- **claude-agent** → [`examples/claude-agent.yml`](examples/claude-agent.yml).
- **strands** → [`examples/strands.yml`](examples/strands.yml).

## How the governance wrap works (build-governed-image)

1. Build the agent's **own, unchanged** Dockerfile → push `…:base-<tag>`.
2. **Generate** the governance Dockerfile inline (the SDK does not ship one):
   - **claude-agent**: `FROM claude-agent-pkg` → copy `/opt/initializ` → `ENV NODE_OPTIONS="--import /opt/initializ/dist/register.js"` → `USER 1000`.
   - **strands**: `FROM strands-pkg` → install the prebuilt `initializ_strands` wheel into the app's site-packages; its shipped `.pth` auto-activates governance (audit + PDP + model-gateway egress) on every interpreter start → `USER 1000`.
3. Build the governed image `FROM …:base-<tag>` → push `…:<tag>`.

> **Why push the base first?** The governed build does `FROM ${AGENT_IMAGE}`. The
> container/remote buildx driver most CI runners use resolves `FROM` from the
> **registry**, not the local store — so the base has to be pushed before the wrap
> build can reference it. This action does that for you.

## Inputs

### build-governed-image
| input | default | notes |
| --- | --- | --- |
| `framework` | — | `claude-agent` \| `strands` (required) |
| `image` | — | target repo, e.g. `ghcr.io/org/agent` (required) |
| `tag` | — | governed tag, e.g. `${{ github.sha }}` (required) |
| `context` | `.` | build context |
| `app-dockerfile` | `Dockerfile` | the agent's own Dockerfile |
| `pkg-image` | per-framework | **pin an immutable `:vX.Y.Z`** — a mutable tag is buildx-cached and can ship a stale SDK |
| `strands-extra` | `anthropic` | `anthropic` (Messages API) \| `openai` (Responses API) — match the model gateway backend |
| `governed-cmd` | — | JSON array, e.g. `'["node","dist/worker/index.js"]'`; set only if the base declares no CMD |
| `platform` | `linux/amd64` | match the cluster |
| `registry`/`username`/`password` | `ghcr.io` | optional login (needs `write:packages` + `read:packages`) |

Outputs: `ref` (governed), `base-ref`.

### deploy
Config resolves as **explicit input > ambient `INITIALIZ_*` env > baked default**, so a
caller that sets `INITIALIZ_*` at the job level (e.g. from an **org secret**) passes only `image`.

| input | default | notes |
| --- | --- | --- |
| `image` | — | pushed image ref (required) |
| `spec-file` | `initializ-deploy.yaml` | carries `agent.type` + tenancy |
| `token` | `$INITIALIZ_TOKEN` | platform access token; **falls back to the ambient env** |
| `org-id` | `$INITIALIZ_ORG_ID` | falls back to the ambient env |
| `workspace-id` | `$INITIALIZ_WORKSPACE_ID` | or `agent.workspace` in the spec |
| `api-url` | — (required) | your agent-builder ingress; or set `$INITIALIZ_API_URL` |
| `auth-url` | — (required) | your api-next ingress (whoami/verify); or set `$INITIALIZ_AUTH_URL` |
| `cli-version` | `latest` | CLI release tag |
| `wait` / `timeout` | `true` / `10m` | poll the rollout |
| `tags` | — | newline-separated `key=value`, stamped as `agent.initializ.ai/<key>` |

## What the action provides vs. what you supply
- **You always name your environment.** `api-url`/`auth-url` are **required** (via input or `INITIALIZ_API_URL`/`INITIALIZ_AUTH_URL`) — the action ships no default endpoint, so it can never point at the wrong platform.
- **`INITIALIZ_TOKEN` cannot be baked into the action** (a committed secret would be a leaked credential, and an action's own secrets aren't exposed to callers — GitHub requires the secret to come from the *caller's* workflow). Set it **once as a GitHub organization secret** (visible to your agent repos) and expose it at the job level; then the deploy step needs no token input.

### Minimal setup (set once at the org)
1. **Org secret** `INITIALIZ_TOKEN` (Settings → Secrets → Actions → New organization secret; scope to the agent repos).
2. **Org variables** `INITIALIZ_ORG_ID` (and `INITIALIZ_WORKSPACE_ID` unless it's in the spec). URLs only if you're not on the test default.
3. In the caller workflow, surface them as job env once:
   ```yaml
   jobs:
     deploy:
       env:
         INITIALIZ_TOKEN: ${{ secrets.INITIALIZ_TOKEN }}
         INITIALIZ_ORG_ID: ${{ vars.INITIALIZ_ORG_ID }}
         INITIALIZ_WORKSPACE_ID: ${{ vars.INITIALIZ_WORKSPACE_ID }}
   ```
   Then the deploy step is just `with: { image: ... }`.
- Registry creds for build/push (`GITHUB_TOKEN` suffices for ghcr with `packages: write`).

## Caveats
- The generated governance Dockerfile is **version-coupled to the SDK** — pin `pkg-image` and bump it when you take a new SDK.
- `strands-extra` must match the gateway backend (`anthropic`/`openai`).
- `governed-cmd` / non-root user are the agent-specific lines; override `governed-cmd` when your base sets no CMD.
