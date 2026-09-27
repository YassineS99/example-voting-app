# PR Smoke Test — Vote Service

## What this check verifies

This adds a GitHub Actions workflow (`.github/workflows/vote-smoke-test.yml`) that runs
automatically on every pull request against this repository. It answers one specific
question that a successful `docker build` alone cannot answer:

> **Does the resulting `vote` service image actually start and serve traffic?**

A Docker image can build without errors while the process inside it still fails to
start, crashes on boot, or never binds to its port. This check catches that class of
problem before merge.

### What it does, step by step

1. Checks out the code proposed in the pull request (not a fixed baseline).
2. Builds the existing `./vote/Dockerfile` using `./vote` as the build context — no
   custom or replacement Dockerfile is used.
3. Starts a container from the newly built image, publishing container port `80` to
   host port `8080`.
4. Retries an HTTP GET request to `http://localhost:8080/` for up to ~30 seconds
   (15 attempts, 2 seconds apart), to allow for normal application startup time.
5. Requires an HTTP `200` response. A connection failure, timeout, or any non-200
   status causes the job to fail.
6. On failure, dumps the container's logs (`docker logs`) before removing it, so the
   failure is diagnosable directly from the failed CI run.
7. Always removes the test container afterward, whether the check passed or failed.

### What it does *not* check

This is a smoke test, not a full integration test. It only verifies that the `vote`
service's root page (`/`) responds with HTTP 200. It does not submit a vote, and does
not exercise Redis or any other part of the application stack.

## How to rerun it

- **Automatically:** the workflow runs on every pull request (on open, new commits,
  and reopen) targeting this repository. No action needed.
- **Manually, from a PR:** open the "Checks" tab on the pull request, select
  "Vote Image Smoke Test," and use "Re-run all jobs."
- **Locally**, without GitHub Actions:
```bash
  docker build -t vote-app:test ./vote
  docker run -d --name vote-test -p 8080:80 vote-app:test
  for i in $(seq 1 15); do
    status=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/ || echo "000")
    [ "$status" = "200" ] && echo "OK on attempt $i" && break
    echo "Attempt $i: $status, retrying..."
    sleep 2
  done
  docker logs vote-test   # if you need to inspect it
  docker rm -f vote-test
```

## Limitation

The smoke test only checks that the container serves an HTTP 200 on `/`. It doesn't
verify the page's actual content, doesn't check that voting works end-to-end, and
doesn't exercise the Redis dependency at all — a container that returns 200 for an
unrelated or broken reason (e.g. a generic error page misconfigured to return 200)
would still pass this check.

## Tested against

- **Baseline commit:** `63e9150ca17af4ed05880d4245e486481f73fcb4`
- **Runner:** GitHub-hosted `ubuntu-24.04`, Actions runner version `2.337.0`
- **Actions used:** `actions/checkout@v4`
- **Git:** `2.55.0`
- **Docker:** as provided by the GitHub-hosted `ubuntu-24.04` runner image (no custom
  installation required)
- **Workflow permissions:** `contents: read` only — no secrets, no image publishing,
  no changes to external infrastructure.

Verified passing pull request: **PR #1** in this fork
(`YassineS99/example-voting-app`, `assessment-ci` → `main`).