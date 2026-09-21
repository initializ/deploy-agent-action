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
| input | default | notes |
| --- | --- | --- |
| `image` | — | pushed image ref (required) |
| `spec-file` | `initializ-deploy.yaml` | carries `agent.type` + tenancy |
| `token` | — | `INITIALIZ_TOKEN` (required, secret) |
| `org-id` | — | `INITIALIZ_ORG_ID` (required) |
| `workspace-id` | — | `INITIALIZ_WORKSPACE_ID` (or `agent.workspace` in the spec) |
| `api-url` / `auth-url` | — | agent-builder / api-next ingress (required) |
| `cli-version` | `latest` | CLI release tag |
| `wait` / `timeout` | `true` / `10m` | poll the rollout |
| `tags` | — | newline-separated `key=value`, stamped as `agent.initializ.ai/<key>` |

## Secrets / vars to set on the caller repo
- **Secret** `INITIALIZ_TOKEN` — platform access token (the only real secret).
- **Secrets/vars** `INITIALIZ_ORG_ID`, `INITIALIZ_WORKSPACE_ID`.
- **Vars** `INITIALIZ_API_URL`, `INITIALIZ_AUTH_URL`.
- Registry creds for build/push (`GITHUB_TOKEN` suffices for ghcr with `packages: write`).

## Caveats
- The generated governance Dockerfile is **version-coupled to the SDK** — pin `pkg-image` and bump it when you take a new SDK.
- `strands-extra` must match the gateway backend (`anthropic`/`openai`).
- `governed-cmd` / non-root user are the agent-specific lines; override `governed-cmd` when your base sets no CMD.
