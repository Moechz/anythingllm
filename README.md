# anythingllm-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [AnythingLLM](https://github.com/Mintplex-Labs/anything-llm).

## What this is

A four-file TOS Docker application archive (`anythingllmdocker.tar.gz`) that installs
AnythingLLM on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `anythingllmdocker`)
- `anythingllmdocker.lang` — 23-language superset (TOS requires 14)
- `anythingllmdocker.svg` — application icon (274 bytes, 2 SVG elements)
- `docker-compose.yml` — one service, image `mintplexlabs/anythingllm:1.19.0`

## How it works

| | |
|---|---|
| Container image | `mintplexlabs/anythingllm:1.19.0` (Docker Hub, used unmodified) |
| Open | `http://<NAS-IP>:8042/` (container port 3001) |
| Runs as | non-root, `user: "1000:1000"` |
| Persistent data | `/app/server/storage` → `/Volume1/DockerAppData/anythingllmdocker/storage` |
| Logs | `docker logs anythingllmdocker` (stdout/stderr) |

`/app/server/storage` (mounted from `DockerAppData/anythingllmdocker/storage`) holds the SQLite database, uploaded documents, vector cache, chat history and settings.

## Design highlights

- **Non-root.** `user: "1000:1000"`; verified on real hardware (the container reports uid 1000).
- **Fixed image tag, never `:latest`**, Docker Hub only, no `privileged`, no `network_mode: host`, and no Docker socket, no `cap_add`.
- **Health check** probing the service on container port 3001.
- **Deterministic build**: GNU tar with uid/gid 0 and zeroed timestamps, gzip without a timestamp, plus the review-standards self-check above.
- **Already non-root.** The image runs as uid 1000 (`anythingllm`); the package pins the same user and points `STORAGE_DIR` at the mounted volume (without it the bundled collector crashes on start).
- **No bundled model.** The package stays small in scope: connect Ollama or a cloud API in Settings.
- **One volume, one port.** Only the storage folder is persistent and only port 3001 is published.

## Runtime file manifest

Everything the application writes on the NAS lives inside its own data
directory; the container-internal `/tmp` is discarded with the container. The
table lists what is created and how it behaves. Nothing else is written.

| Path | Purpose | Created when | Growth bound | Lifecycle |
|---|---|---|---|---|
| `/Volume1/DockerAppData/anythingllmdocker/storage` | AnythingLLM application data (everything the container writes outside /tmp) | on first start / on demand | bounded by your own data | persistent; kept on uninstall unless you ask to delete it |

## Build

```bash
scripts/build.sh                  # -> out/anythingllmdocker.tar.gz + .sha256
scripts/build.sh 1.19.0-2   # packaging iteration bump
scripts/build.sh 1.19.0-1 aarch64
```

The build stages the four files, substitutes the version/platform placeholders,
removes CRLF/BOM/AppleDouble, then runs
`tools/verify.py`, a review-standards self-check (archive layout, JSON validity,
required fields, 14+ language coverage, version consistency, Docker-Hub-only
images, fixed tag, reserved/recommended host ports, non-root user, per-service
healthcheck/TZ/restart, data path below the application data root, never the
`/Volume*` wildcard, comment-free sequence blocks, `x-app-meta` position, no
literal secrets and the SVG size/element limits).

## Local verification

```bash
./TOSAppSelfTestingTool -c out/anythingllmdocker.tar.gz   # official spec check
bash test/install-and-verify.sh                  # on-device E2E (needs root)
```

> Docker applications cannot be side-loaded through the App Center's manual
> install page (it accepts `.deb` only); the official self-testing tool's `-i`
> mode or `docker compose` are the only ways to install one for testing.

## Store submission checklist

1. Public GitHub repo (code + README only).
2. Release **tag `v1.19.0-1`**, assets `anythingllmdocker.tar.gz` +
   `anythingllmdocker.tar.gz.sha256` (Docker asset naming: `<app_id>.tar.gz`, **no
   platform suffix**).
3. Developer platform → Add Application: ID `anythingllmdocker`, package type Docker,
   repository URL, architecture `x86_64`.
4. Version Management → Add Version `1.19.0-1` → automated validation → review.

## License / branding

AnythingLLM is distributed by its upstream authors under **MIT**. The container image is the official image from Docker Hub (`mintplexlabs/anythingllm:1.19.0`) and is used unmodified; no upstream source code is redistributed here. This packaging is an independent community submission by Moechz and is not affiliated with the upstream project or TerraMaster.

## Limitations

- No model is bundled: without an Ollama endpoint or API key the chat has nothing to answer with.
- The web UI is plain HTTP; use a TLS reverse proxy before exposing it to the internet.
- The image is ~2 GB and startup pulls model metadata from the internet, so the first start is slower on a disconnected NAS.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
