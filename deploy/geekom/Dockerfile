# Geekom build of upstream garmin_mcp @ 655efb8 with garminconnect 0.3.19.
# Same steps as the upstream ./Dockerfile, but with the base image, uv and every runtime dependency
# pinned to what the production image (garmin-mcp:655efb8f5639-gc0.3.19) actually contains.
# Build from the repository root:  docker build -f deploy/geekom/Dockerfile -t garmin-mcp:655efb8f5639-gc0.3.19-r .
FROM python:3.12.14-slim

WORKDIR /app

COPY --from=ghcr.io/astral-sh/uv:0.12.16 /uv /usr/local/bin/uv

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    UV_SYSTEM_PYTHON=1

COPY pyproject.toml README.md ./
COPY src/ ./src/
COPY deploy/geekom/constraints.txt /tmp/constraints.txt

RUN uv pip install -e . -c /tmp/constraints.txt && rm /tmp/constraints.txt

COPY tests/ ./tests/
COPY pytest.ini ./

# Token store; mount the persistent `garmin-tokens` volume here. Never bake tokens into the image.
RUN mkdir -p /root/.garminconnect && \
    chmod 700 /root/.garminconnect

ENTRYPOINT ["garmin-mcp"]
