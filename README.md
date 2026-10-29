# Project 2: Containerize and Run Locally

This project contains a multi-stage web image, a Docker Compose stack
with Nginx and Redis, and a locally invocable AWS SAM function.

## Prerequisites

- Docker running with Docker Compose available.
- AWS SAM CLI installed inside Ubuntu WSL.
- Git installed.
- Internet access for initial image downloads and builds.
- Host port 8080 available.

No AWS account, credentials, or cloud deployment is required.
Run the following commands from the root of the cloned repository.

## 1. Validate the configuration

```bash
grep -c "^FROM" Dockerfile
docker compose config --services
```

Expected: 2 stages and services named cache and web.

## 2. Build the web image

```bash
docker build -t demo:multi .
```

Expected: a successful build.

## 3. Start and check both services

```bash
docker compose up -d --build
docker compose ps
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080
curl -s http://localhost:8080
docker compose exec cache redis-cli ping
```

Expected:
- web is running and cache is healthy.
- The HTTP status is 200.
- The webpage contains T02 demo.
- Redis returns PONG.

## 4. Stop the stack

```bash
docker compose down
```

Expected: both containers and the Compose network are removed.

## 5. Invoke the function locally

```bash
cd local-fn
sam local invoke HelloWorldFunction
sam local invoke HelloWorldFunction -e events/event.json
cd ..
```

Expected: both invocations return statusCode 200 with a hello world body.
The first invocation may download the runtime image.

The course handout notes that a trailing "Failed to get next invocation"
message can appear during emulator shutdown after the successful response.
Other errors or missing successful responses require investigation.

## 6. Reproduce the image-size comparison

```bash
docker build -f Dockerfile.single -t demo:single .
docker build -t demo:multi .
docker images | grep -E 'REPOSITORY|demo'
```

Recorded measurements are in evidence/t02-image-sizes.txt.
The design explanation is in RATIONALE.md.

## 7. Inspect and test the dependency

```bash
docker compose up -d
docker compose logs web
docker compose exec web sh -c 'nginx -v'
docker compose exec cache redis-cli ping
docker inspect $(docker compose ps -q web) --format '{{.State.Status}} started={{.State.StartedAt}} restarts={{.RestartCount}}'
docker compose stop cache
docker compose ps -a
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080
docker compose down
```

Expected: Redis stops, but web continues returning HTTP 200.
The static webpage does not use Redis at runtime.
The healthchecked dependency controls startup readiness.

## WSL troubleshooting observed during development

On my Docker Desktop and WSL VirtioProxy setup, SAM failed to start its
container with a "/forwards/expose returned unexpected status: 500" error.

Ordinary Docker port publishing worked. However, briefly binding and
closing the port in Ubuntu before starting Docker reproduced the failure.

Setting the following in the Windows user's .wslconfig file, then fully
shutting down WSL and restarting Docker Desktop, resolved the failure:

```ini
[wsl2]
localhostForwarding=false
```

Both SAM invocations and the Compose HTTP and Redis checks passed afterward.

This is an environment-specific workaround, not a prerequisite for everyone.
It affects automatic Windows localhost access to servers running directly
inside WSL. Preserve any existing configuration when applying it.
To reverse it, restore localhostForwarding=true and restart WSL and Docker.

Reference:
https://learn.microsoft.com/en-us/windows/wsl/wsl-config
