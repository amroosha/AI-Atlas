# DevOps and Python Tooling Reference

Linux-oriented reference for Docker, `uv`, `pipreqs`, and Cookiecutter. Commands assume a Bash shell and a recent tool release.

## Docker

Docker packages applications and their runtime dependencies into isolated containers. The Docker daemon must be running before using the CLI.

### Quick Reference

| Task | Command |
| --- | --- |
| Verify installation | `docker version` |
| Run a shell in a disposable container | `docker run --rm -it ubuntu:24.04 bash` |
| List running containers | `docker ps` |
| List all containers | `docker ps -a` |
| Build an image | `docker build -t app:local .` |
| View logs | `docker logs -f CONTAINER` |
| Open a shell in a running container | `docker exec -it CONTAINER sh` |
| Start a Compose application | `docker compose up -d --build` |
| Remove stopped containers | `docker container prune` |

### Installation and Context

Check the client, daemon, storage driver, and active context before troubleshooting.

```bash
docker version
docker info
docker context ls
docker context show
docker system df
```

Use a non-root Docker context where possible. On a standard Linux installation, the current user can be added to the `docker` group, after which a new login session is required.

```bash
sudo usermod -aG docker "$USER"
newgrp docker
docker run --rm hello-world
```

### Images

Images are immutable layers used to create containers. Tags are mutable pointers, so pin production dependencies by digest when reproducibility is required.

```bash
docker pull python:3.12-slim
docker images
docker image ls --filter=reference='python*'
docker image inspect python:3.12-slim
docker history python:3.12-slim
docker tag python:3.12-slim registry.example.com/team/python:3.12-slim
docker push registry.example.com/team/python:3.12-slim
```

Build from a `Dockerfile`, pass build arguments explicitly, and inspect the resulting image.

```bash
docker build --pull --progress=plain -t atlas-api:dev .
docker build --build-arg APP_VERSION=dev -t atlas-api:dev .
docker image inspect atlas-api:dev --format '{{.Id}} {{.Size}}'
docker save atlas-api:dev | gzip > atlas-api-dev.tar.gz
docker load < atlas-api-dev.tar.gz
```

### Container Lifecycle

`docker run` creates and starts a container. `start`, `stop`, `restart`, and `rm` operate on an existing container.

```bash
docker run -d --name atlas-api -p 8080:8000 --restart unless-stopped atlas-api:dev
docker ps
docker ps -a
docker inspect atlas-api
docker logs --tail=100 -f atlas-api
docker stop atlas-api
docker start atlas-api
docker restart atlas-api
docker rm atlas-api
```

Use resource limits and environment files for repeatable local runs.

```bash
docker run -d --name atlas-api \
	--env-file .env \
	--cpus=2 --memory=1g \
	-p 127.0.0.1:8080:8000 \
	atlas-api:dev
docker stats atlas-api
docker top atlas-api
```

### Executing Commands and Copying Files

`exec` runs a process in a running container. It does not create a new container.

```bash
docker exec atlas-api python --version
docker exec -it atlas-api sh
docker exec -u root atlas-api sh
docker cp atlas-api:/app/logs/app.log ./app.log
docker cp ./config.yaml atlas-api:/app/config.yaml
docker diff atlas-api
```

### Volumes and Bind Mounts

Named volumes are managed by Docker and are suitable for persistent service data. Bind mounts expose host paths and are useful for source code during development.

```bash
docker volume create atlas-db-data
docker volume ls
docker volume inspect atlas-db-data
docker run -d --name postgres \
	-e POSTGRES_PASSWORD=localdev \
	-v atlas-db-data:/var/lib/postgresql/data \
	postgres:16
docker run --rm -v "$PWD":/workspace -w /workspace alpine ls -la
docker volume rm atlas-db-data
```

Do not remove a volume containing data until it has been backed up.

```bash
docker run --rm -v atlas-db-data:/data -v "$PWD":/backup alpine \
	tar czf /backup/atlas-db-data.tgz -C /data .
```

### Networking

Containers on the same user-defined network resolve each other by container name. Published ports are for access from the host or external networks.

```bash
docker network create atlas-net
docker network ls
docker run -d --name redis --network atlas-net redis:7-alpine
docker run --rm --network atlas-net busybox nslookup redis
docker network inspect atlas-net
docker network connect atlas-net atlas-api
docker network disconnect atlas-net atlas-api
docker network rm atlas-net
```

### Dockerfile

This example uses a small runtime image, a non-root user, and a health check.

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
		PYTHONUNBUFFERED=1

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt \
		&& useradd --create-home appuser

COPY . .
RUN chown -R appuser:appuser /app
USER appuser

EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')"
CMD ["python", "-m", "app"]
```

### CMD and ENTRYPOINT

`ENTRYPOINT` defines the executable that normally cannot be replaced, while `CMD` provides default arguments or a default command. Prefer exec form so the application receives signals directly and can shut down cleanly.

| Instruction | Purpose | Runtime override |
| --- | --- | --- |
| `ENTRYPOINT ["python", "-m", "app"]` | Fixed executable | `docker run --entrypoint sh IMAGE` |
| `CMD ["python", "-m", "app"]` | Default executable and arguments | `docker run IMAGE other-command` |
| `ENTRYPOINT ["python", "-m", "app"]` plus `CMD ["--port", "8000"]` | Fixed executable with default arguments | `docker run IMAGE --port 9000` |

The shell form starts through `/bin/sh -c`; this can prevent signals such as `SIGTERM` from reaching the application process. The JSON array is the exec form.

```dockerfile
# Fixed executable; arguments supplied by CMD can be replaced at runtime.
ENTRYPOINT ["python", "-m", "app"]
CMD ["--host", "0.0.0.0", "--port", "8000"]
```

Build and exercise the image with its defaults, changed arguments, and a completely different entrypoint.

```bash
docker build -t atlas-api:dev .
docker run --rm -p 8000:8000 atlas-api:dev
docker run --rm -p 9000:9000 atlas-api:dev --port 9000
docker run --rm -it --entrypoint sh atlas-api:dev
```

Use `CMD` alone for images intended to run different commands. Use `ENTRYPOINT` when the image represents one executable, such as a service or CLI, and use `CMD` for defaults that operators may change.

Build and test the image locally.

```bash
docker build --pull -t atlas-api:dev .
docker run --rm -p 8000:8000 atlas-api:dev
curl --fail http://127.0.0.1:8000/health
```

### Multi-Platform Builds

Docker identifies platforms with an operating system, architecture, and optional variant. Common Linux targets are `linux/amd64` for x86-64 servers, `linux/arm64` for 64-bit ARM servers and Apple Silicon, and `linux/arm/v7` for 32-bit ARM devices.

Inspect the host and create a Buildx builder capable of producing images for multiple platforms.

```bash
uname -m
docker buildx version
docker buildx ls
docker buildx create --name atlas-builder --driver docker-container --use
docker buildx inspect --bootstrap
```

Build for one platform and load the result into the local Docker image store. `--load` supports one platform at a time.

```bash
docker buildx build \
	--platform linux/amd64 \
	--tag atlas-api:amd64 \
	--load \
	.
docker image inspect atlas-api:amd64 --format '{{.Os}}/{{.Architecture}}'
docker run --rm --platform linux/amd64 atlas-api:amd64 python --version
```

Build and publish a multi-platform manifest list. Each platform gets its own image, and the registry tag points clients to the correct variant.

```bash
docker login registry.example.com
docker buildx build \
	--platform linux/amd64,linux/arm64,linux/arm/v7 \
	--tag registry.example.com/team/atlas-api:1.0.0 \
	--push \
	.
docker buildx imagetools inspect registry.example.com/team/atlas-api:1.0.0
docker pull registry.example.com/team/atlas-api:1.0.0
```

Use `--platform` during a build to select the target platform and `TARGETARCH` or `TARGETPLATFORM` when a Dockerfile must download architecture-specific artifacts.

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.23 AS builder
ARG TARGETOS
ARG TARGETARCH
WORKDIR /src
COPY . .
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /out/app .

FROM alpine:3.20
COPY --from=builder /out/app /usr/local/bin/app
ENTRYPOINT ["/usr/local/bin/app"]
```

For builds that execute target-architecture binaries, install emulation support or use native builders. Native ARM and AMD64 builders usually provide better performance than emulation.

```bash
docker run --privileged --rm tonistiigi/binfmt --install all
docker buildx build --platform linux/amd64,linux/arm64 --push -t registry.example.com/team/atlas-api:latest .
docker buildx create --name atlas-multi --append ssh://arm-builder
docker buildx inspect atlas-multi --bootstrap
```

Keep dependencies and base images available for every target, and test each published variant explicitly.

```bash
docker pull --platform linux/amd64 registry.example.com/team/atlas-api:1.0.0
docker pull --platform linux/arm64 registry.example.com/team/atlas-api:1.0.0
docker run --rm --platform linux/amd64 registry.example.com/team/atlas-api:1.0.0 --version
docker run --rm --platform linux/arm64 registry.example.com/team/atlas-api:1.0.0 --version
```

### Docker Compose

Compose defines related services, networks, and volumes in one YAML file. Service names become DNS names on the default Compose network.

```yaml
services:
	api:
		build: .
		ports:
			- "8000:8000"
		environment:
			DATABASE_URL: postgresql://app:localdev@db:5432/app
		depends_on:
			db:
				condition: service_healthy

	db:
		image: postgres:16-alpine
		environment:
			POSTGRES_DB: app
			POSTGRES_USER: app
			POSTGRES_PASSWORD: localdev
		volumes:
			- postgres-data:/var/lib/postgresql/data
		healthcheck:
			test: ["CMD-SHELL", "pg_isready -U app -d app"]
			interval: 5s
			timeout: 5s
			retries: 10

volumes:
	postgres-data:
```

Run, inspect, and tear down the Compose project.

```bash
docker compose config
docker compose up -d --build
docker compose ps
docker compose logs -f api
docker compose exec api python --version
docker compose restart api
docker compose down
docker compose down --volumes
```

Use profiles for optional services and a separate override file for local-only settings.

```bash
docker compose --profile tools up -d
docker compose -f compose.yaml -f compose.override.yaml up -d
```

### Cleanup

Prune only resources that are no longer referenced. The volume command can delete persistent data.

```bash
docker container prune
docker image prune
docker image prune -a
docker volume prune
docker network prune
docker system prune
docker system prune --all --volumes
```

Review disk usage before and after cleanup.

```bash
docker system df
docker system df --verbose
```

## uv

`uv` is a fast Python package and project manager implemented in Rust. It uses a global cache, resolves dependencies once into a lockfile, and creates isolated environments for projects.

### Quick Reference

| Task | Command |
| --- | --- |
| Install `uv` | `curl -LsSf https://astral.sh/uv/install.sh | sh` |
| Create a project | `uv init my-project` |
| Add a dependency | `uv add requests` |
| Install the lockfile | `uv sync` |
| Run a project command | `uv run python -m app` |
| Run a one-off tool | `uvx ruff check .` |
| Manage a virtual environment | `uv venv` |
| Use the pip-compatible interface | `uv pip install -r requirements.txt` |
| Compile pinned requirements | `uv pip compile pyproject.toml -o requirements.txt` |

### Installation and Python Versions

Install with the official installer or a package manager, then verify the executable.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv --version
uv self update
```

Install and select Python versions independently of the system Python.

```bash
uv python install 3.12
uv python list
uv python find 3.12
uv python pin 3.12
python --version
```

### Project Workflow

Initialize an application or library. `uv` creates `pyproject.toml`, a source layout where requested, and manages `.venv` and `uv.lock`.

```bash
uv init atlas-api
cd atlas-api
uv init --lib
uv add fastapi uvicorn
uv add --dev pytest ruff
uv sync
uv run python -c "import fastapi; print(fastapi.__version__)"
```

A minimal project configuration can declare the Python range and dependencies explicitly.

```toml
[project]
name = "atlas-api"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
	"fastapi>=0.115,<1",
	"uvicorn[standard]>=0.30,<1",
]

[dependency-groups]
dev = [
	"pytest>=8,<9",
	"ruff>=0.6,<1",
]
```

Keep the lockfile under version control for applications. Update selected dependencies deliberately.

```bash
uv lock
uv lock --check
uv sync --locked
uv add 'httpx>=0.27'
uv remove httpx
uv sync --dev
uv sync --no-dev
```

Run tools through the project environment so the command uses locked dependencies.

```bash
uv run pytest
uv run ruff check .
uv run python -m compileall src
uv run --group docs mkdocs build
```

### Environments and the Pip Interface

Use the pip-compatible commands when working with an existing `requirements.txt` workflow or when a project does not use a `pyproject.toml`.

```bash
uv venv --python 3.12 .venv
source .venv/bin/activate
uv pip install requests
uv pip install -r requirements.txt
uv pip install -e '.[dev]'
uv pip list
uv pip show requests
uv pip freeze > requirements-lock.txt
```

Resolve and compile repeatable requirements without installing them.

```bash
uv pip compile pyproject.toml --extra dev -o requirements.txt
uv pip compile requirements.in --universal -o requirements.txt
uv pip sync requirements.txt
uv pip check
```

### Tools with `uvx`

`uvx` is an alias for `uv tool run`. It creates an isolated temporary environment and makes a CLI available without adding it to the project.

```bash
uvx ruff check .
uvx --from 'httpie>=3,<4' http GET https://example.com
uv tool install ruff
uv tool list
uv tool run ruff --version
uv tool upgrade ruff
uv tool uninstall ruff
```

### Sources, Caches, and Performance

Use alternate indexes and cache controls explicitly in automation.

```bash
uv add --index https://pypi.org/simple requests
UV_INDEX_URL=https://pypi.org/simple uv sync
uv cache dir
uv cache clean
uv cache prune
```

`uv` is fast because it uses a Rust implementation, parallel downloads, a content-addressed global cache, and a resolver that can reuse previously computed artifacts. It does not eliminate network, build, or resolver work when inputs change.

```bash
time uv sync --locked
time uv sync --refresh
uv sync --offline
uv sync --no-cache
```

For CI, cache the directory reported by `uv cache dir`, use `uv sync --locked`, and avoid silently changing the lockfile.

```bash
export UV_CACHE_DIR="$HOME/.cache/uv"
uv cache dir
uv sync --locked --no-dev
```

## pipreqs

`pipreqs` scans Python source imports and writes a minimal `requirements.txt`. It is useful for recovering direct dependencies from an existing codebase; it does not replace lockfiles or dependency review.

### Quick Reference

| Task | Command |
| --- | --- |
| Install | `python -m pip install pipreqs` |
| Scan the current project | `pipreqs .` |
| Choose an output file | `pipreqs . --force --savepath requirements.txt` |
| Scan a source directory | `pipreqs src --force` |
| Include package versions | `pipreqs . --use-local --force` |
| Ignore directories | `pipreqs . --ignore .venv,tests --force` |
| Inspect the generated file | `python -m pip install -r requirements.txt` |

### Generate Requirements

Run from the repository root and exclude virtual environments, generated files, and tests when they are not deployment dependencies.

```bash
python -m pip install pipreqs
pipreqs . --force --savepath requirements.txt --ignore .venv,venv,tests
cat requirements.txt
python -m pip install -r requirements.txt
```

Use local imports and a different encoding when the source tree requires it.

```bash
pipreqs src --force --use-local --encoding=utf-8
pipreqs . --force --savepath requirements-dev.txt --ignore .git,.venv,build,dist
```

Review import-to-distribution mappings manually. Python import names and package names are not always identical, and dynamic imports are not reliably discoverable by static scanning.

```bash
python -m pip install -r requirements.txt
python -m pip check
python -c "import pathlib; print(pathlib.Path('requirements.txt').read_text())"
```

## Cookiecutter

Cookiecutter generates projects from Jinja2 templates. Template variables are collected from `cookiecutter.json` and substituted into directory names, filenames, and file contents.

### Quick Reference

| Task | Command |
| --- | --- |
| Install | `uv tool install cookiecutter` |
| Generate from a Git template | `cookiecutter https://github.com/audreyr/cookiecutter-pypackage.git` |
| Generate from a local template | `cookiecutter ./templates/python-service` |
| Set a variable | `cookiecutter TEMPLATE project_name=atlas-api` |
| Preview without writing | `cookiecutter TEMPLATE --no-input --output-dir /tmp/preview` |
| Reuse cached answers | `cookiecutter TEMPLATE --replay` |
| Store answers | `cookiecutter TEMPLATE --replay-file answers.json` |

### Install and Use a Template

Install the CLI in an isolated tool environment, then render a template into a target directory.

```bash
uv tool install cookiecutter
cookiecutter --version
cookiecutter https://github.com/audreyr/cookiecutter-pypackage.git
cookiecutter ./templates/python-service --output-dir ./generated
```

### Minimal Template

Create a template with a JSON context file and Jinja2 placeholders in both paths and contents.

```bash
mkdir -p templates/python-service/{{cookiecutter.project_slug}}
cat > templates/python-service/cookiecutter.json <<'EOF'
{
	"project_name": "Atlas Service",
	"project_slug": "atlas_service",
	"python_version": "3.12"
}
EOF
printf '# {{ cookiecutter.project_name }}\n\nRequires Python {{ cookiecutter.python_version }}.\n' > templates/python-service/{{cookiecutter.project_slug}}/README.md
cookiecutter ./templates/python-service --no-input --output-dir ./generated
```

For a version-controlled template, place hooks under `hooks/` and keep generated projects free of template-only files.

```bash
find templates/python-service -maxdepth 3 -type f -print
cookiecutter ./templates/python-service --no-input project_slug=payments_service
git diff -- generated/
```

### Reproducible Generation

Use `--no-input` with explicit values in automation and save the answer file when an interactive template should be repeatable.

```bash
cookiecutter ./templates/python-service \
	--no-input \
	project_name='Payments Service' \
	project_slug=payments_service \
	python_version=3.12 \
	--output-dir ./generated

cookiecutter ./templates/python-service --replay
cookiecutter ./templates/python-service --replay-file answers.json
```

Generated code should be reviewed and tested like handwritten code; templating does not validate the resulting project.
