# Upstream Hermes image + the official Claude Code CLI, which the
# claude-subscription-directsdk plugin shells out to for every model turn.
# /opt/hermes is immutable at runtime, so the CLI has to be baked in here.
ARG HERMES_TAG=v2026.9.24
FROM nousresearch/hermes-agent:${HERMES_TAG}

# Pin the CLI so a rebuild doesn't silently change the model transport.
# Bump deliberately: docker compose build --build-arg CLAUDE_CODE_VERSION=x.y.z
ARG CLAUDE_CODE_VERSION=2.1.288
RUN npm install -g "@anthropic-ai/claude-code@${CLAUDE_CODE_VERSION}" \
    && npm cache clean --force \
    && claude --version

# Photon sidecar: upstream's baked node_modules is older than package-lock.json,
# so Hermes treats it as stale and mirrors the sidecar to /opt/data/photon —
# but the mirror omits send-format.mjs / stream-staleness.mjs and the sidecar
# crashes (ERR_MODULE_NOT_FOUND). Reinstalling from the lock here makes the
# deps current, so Hermes runs the sidecar in place from the complete tree.
RUN cd /opt/hermes/plugins/platforms/photon/sidecar \
    && npm ci --omit=dev \
    && npm cache clean --force
