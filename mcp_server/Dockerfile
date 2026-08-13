ARG BASE_IMAGE=python:3.12-slim-bookworm
FROM ${BASE_IMAGE}

WORKDIR /app

ARG PIP_INDEX_URL=https://pypi.org/simple

ENV PIP_INDEX_URL=${PIP_INDEX_URL} \
    PIP_DEFAULT_TIMEOUT=120 \
    PIP_RETRIES=5 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml README.md LICENSE ./
COPY src/ ./src/

RUN pip install --no-cache-dir --prefer-binary .

# Cloud defaults: bind streamable-http to all interfaces on the platform-supplied
# port. Override any of these on the platform side; stdio still works locally
# because pipx/uvx invocations don't inherit these.
ENV QUANTDINGER_MCP_TRANSPORT=streamable-http \
    QUANTDINGER_MCP_HOST=0.0.0.0 \
    QUANTDINGER_MCP_PORT=7800 \
    PYTHONUNBUFFERED=1

EXPOSE 7800

CMD ["quantdinger-mcp"]
