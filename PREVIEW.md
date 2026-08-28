# founding.dev preview stack

This fork adds three files to upstream WeKan. Nothing else is modified, so
rebasing on upstream stays cheap.

| File | Purpose |
|---|---|
| `Dockerfile.preview` | The dev image founding.dev pulls on import. Meteor toolchain + `node_modules` + a warm `.meteor/local`, running `meteor run` so edits hot reload. |
| `docker-compose.preview.yml` | The preview stack. founding.dev looks for this exact filename and lets it override every other compose file in the repo. |
| `Dockerfile.prod` | Production image that **builds from source**. |

## Why Dockerfile.prod exists

Upstream's `Dockerfile` does not build the working tree — it downloads a
prebuilt WeKan release `.zip` and unpacks it. That is correct for shipping a
pinned release and destructive for us: a customer who edits this template and
presses Publish would get stock WeKan, because their changes are not in that
zip. No error, no warning. `Dockerfile.prod` builds the tree that was actually
edited.

## Measured (local, arm64, Docker 28.0.4)

| What | Result |
|---|---|
| Boot to real WeKan UI, warm volumes | **~15s** |
| Edit `imports/i18n/data/en.i18n.json` → served in the client bundle | **18s** |
| Preview image size | 11 GB — too large, being slimmed |

## Four things that will bite the next person

1. **`--port 8080`, never `--port 0.0.0.0:8080`.** Meteor accepts the host:port
   form, but Meteor 3.5's Rspack dev server then derives its own HMR port from
   it, gets `NaN`, and dies with `ERR_SOCKET_BAD_PORT` before finishing its first
   compilation. A bare port binds all interfaces inside a container anyway.

2. **Do not set `PORT` alongside the `--port` flag.** Same failure.

3. **`WRITABLE_PATH` is required.** `server/00checkStartup.js` exits the process
   when it is unset or unwritable — and it does so *after* Meteor starts serving,
   so the symptom is an HTTP **200** rendering Meteor's error page. A health check
   that only looks at the status code will call that healthy.

4. **A bind mount shadows everything the image baked under it.** `node_modules`
   and `.meteor/local` must both be re-exposed as named volumes or the container
   throws away the warm cache and recompiles from nothing — slower than having no
   preview image at all.

## Reproduce

```bash
docker build -f Dockerfile.preview -t wekan-preview:local .
PREVIEW_IMAGE=wekan-preview:local PREVIEW_HOST_PORT=8099 \
  docker compose -f docker-compose.preview.yml up
# curl localhost:8099 → <title>Wekan</title>   (NOT "Meteor App - Error")
```

`down -v` between runs: named volumes seed from the image exactly once, so a new
image against old volumes silently runs the old dependencies.
