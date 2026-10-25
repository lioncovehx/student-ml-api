# MLOps Release Pipeline

An MLOps course assignment on CI/CD. The app itself is a tiny Flask "prediction" API: it returns the input times two, standing in for a real model. The work is in the release process around it: branch protection, pull requests, CI, versioned Docker images published to GHCR, and rollback.

## The API

| Method | Path | Returns |
|---|---|---|
| GET | `/health` | Status, app version, model version |
| POST | `/predict` | `{"input": n, "prediction": n * 2}` |

```bash
curl -X POST http://localhost:5000/predict -H 'Content-Type: application/json' -d '{"value":21}'
{"input":21,"prediction":42}
```

A missing field, a non-numeric value or a body that isn't a JSON object gets a 400. The model version is tracked separately from the app version, so you can tell a model change from a code change.

## How a change gets released

```
feature branch -> pull request -> CI -> squash merge -> version tag -> release
```

- `main` is protected. Every change has to come through a pull request with both CI checks passing, and that applies to admins too. I tried pushing directly to check that it really gets rejected.
- **`ci.yml`** runs on pull requests. It runs the tests, builds the Docker image, starts the container and checks `/health`. It never publishes anything.
- **`release.yml`** runs when a `v*.*.*` tag is pushed. It runs the tests again, takes the version from the tag (`v1.1.0` becomes `1.1.0`), fails if that doesn't match the `VERSION` file, then builds the image and pushes it to GHCR.
- Every image is tagged with its version, `latest` and the commit hash. It also has the commit stored in an OCI label, so you can trace a running container back to its source.

## Releases

| Version | Image tags | Pull request |
|---|---|---|
| 1.0.0 | `1.0.0`, `fa0bcc7` | #1 |
| 1.1.0 | `1.1.0`, `latest`, `a85bb7c` | #2 |

```bash
docker run -d --name ml-api -p 5000:5000 ghcr.io/lioncovehx/student-ml-api:1.1.0
```

To roll back, run the older tag. There's nothing to rebuild:

```bash
docker rm -f ml-api
docker run -d --name ml-api -p 5000:5000 ghcr.io/lioncovehx/student-ml-api:1.0.0
```

## Things that broke along the way

1. **Tests passed locally but not in CI.** CI failed with `ModuleNotFoundError: No module named 'app'`. Locally I was running `python -m pytest`, which adds the current folder to the import path. CI ran plain `pytest`, which doesn't. Adding a `pytest.ini` with `pythonpath` fixed it.
2. **A failing test blocks the merge.** I broke a test on purpose to check this. The PR went red, the merge button was disabled, and the Docker job didn't run, because it only runs after the tests pass.
3. **The Docker build failed even though all the tests passed.** I added `COPY tests/` to the Dockerfile, but `tests/` is in `.dockerignore`, so the build failed. This showed that the build check blocks a PR on its own, even with green tests.
4. **The container ran but nothing could reach it.** I bound the app to `127.0.0.1` instead of `0.0.0.0`. Inside a container, that's the container's own loopback, so requests coming in through the published port never reached the app. `docker ps` looked fine, and the only clue was in `docker logs`.
5. **A test passed against the wrong container.** While reproducing problem 4, the new container couldn't start because its port was already taken by an old one. My `curl` then got a healthy response from the old container. Now I check which container is actually answering.

## Files

| File | What it is |
|---|---|
| `app.py` | The Flask app |
| `tests/test_app.py` | Five pytest tests |
| `Dockerfile` | Pinned base image, deps installed before source is copied, runs on gunicorn |
| `VERSION` | The one place the version number is set |
| `.github/workflows/` | `ci.yml` and `release.yml` |
| `REPORT.md`, `VIVA.md` | Assignment write-up and viva answers |

## Run locally

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/pytest
.venv/bin/python app.py
```
