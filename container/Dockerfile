# NanoClaw Agent Container
# Runs Claude Agent SDK in isolated Linux VM with browser automation

FROM node:22-slim

# Install system dependencies for Chromium
RUN apt-get update && apt-get install -y \
    chromium \
    fonts-liberation \
    fonts-noto-color-emoji \
    libgbm1 \
    libnss3 \
    libatk-bridge2.0-0 \
    libgtk-3-0 \
    libx11-xcb1 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    libasound2 \
    libpangocairo-1.0-0 \
    libcups2 \
    libdrm2 \
    libxshmfence1 \
    curl \
    git \
    openssh-client \
    && rm -rf /var/lib/apt/lists/*

# Set Chromium path for agent-browser
ENV AGENT_BROWSER_EXECUTABLE_PATH=/usr/bin/chromium
ENV PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH=/usr/bin/chromium

# Install agent-browser and claude-code globally
RUN npm install -g agent-browser @anthropic-ai/claude-code

# Create app directory
WORKDIR /app

# Copy package files first for better caching
COPY agent-runner/package*.json ./

# Install dependencies
RUN npm install

# Copy source code
COPY agent-runner/ ./

# Build TypeScript
RUN npm run build

# Copy utility scripts
COPY scripts/ /usr/local/bin/
RUN chmod +x /usr/local/bin/*.sh 2>/dev/null; \
    # Create symlinks without .sh extension for convenience
    for f in /usr/local/bin/*.sh; do ln -sf "$f" "${f%.sh}" 2>/dev/null || true; done

# Create workspace directories
RUN mkdir -p /workspace/group /workspace/global /workspace/extra /workspace/ipc/messages /workspace/ipc/tasks /workspace/ipc/input /workspace/ipc/images

# Create entrypoint script
# Sources env from mounted /workspace/env-dir/env if it exists (workaround for Apple Container -i bug)
# Stdin is buffered to /tmp then piped (Apple Container requires EOF to flush stdin pipe)
# Follow-up messages arrive via IPC files in /workspace/ipc/input/
RUN printf '#!/bin/bash\nset -e\n[ -f /workspace/env-dir/env ] && export $(cat /workspace/env-dir/env | xargs)\n# Set up SSH keys from mounted read-only directory\nif [ -d /workspace/ssh-keys ]; then\n  mkdir -p ~/.ssh\n  cp /workspace/ssh-keys/id_* ~/.ssh/ 2>/dev/null || true\n  [ -f /workspace/ssh-keys/known_hosts ] && cp /workspace/ssh-keys/known_hosts ~/.ssh/\n  chmod 700 ~/.ssh\n  chmod 600 ~/.ssh/id_* 2>/dev/null || true\n  [ -f ~/.ssh/known_hosts ] && chmod 644 ~/.ssh/known_hosts\n  # Add GitHub to known_hosts if not already there\n  grep -q github.com ~/.ssh/known_hosts 2>/dev/null || ssh-keyscan -t ed25519 github.com >> ~/.ssh/known_hosts 2>/dev/null\nfi\ncd /app && npx tsc --outDir /tmp/dist 2>&1 >&2\nln -s /app/node_modules /tmp/dist/node_modules\nchmod -R a-w /tmp/dist\ncat > /tmp/input.json\nnode /tmp/dist/index.js < /tmp/input.json\n' > /app/entrypoint.sh && chmod +x /app/entrypoint.sh

# Set ownership to node user (non-root) for writable directories
RUN chown -R node:node /workspace

# Switch to non-root user (required for --dangerously-skip-permissions)
USER node

# Set working directory to group workspace
WORKDIR /workspace/group

# Entry point reads JSON from stdin, outputs JSON to stdout
ENTRYPOINT ["/app/entrypoint.sh"]
