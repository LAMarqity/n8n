# 1. Build image
FROM --platform=linux/amd64 n8nio/n8n:latest AS builder

USER root

# Installera nödvändiga verktyg och beroenden
RUN apk add --no-cache python3 py3-pip tesseract-ocr poppler-utils \
    build-base cmake ninja py3-setuptools py3-wheel py3-opencv

WORKDIR /src
COPY . /src

# Använd PNPM för att installera projektets beroenden
RUN --mount=type=cache,id=pnpm-store,target=/root/.local/share/pnpm/store \
    --mount=type=cache,id=pnpm-metadata,target=/root/.cache/pnpm/metadata \
    DOCKER_BUILD=true pnpm install --frozen-lockfile

# Konfigurera PNPM för globala installationer
ENV PNPM_HOME=/home/node/.pnpm
ENV PATH=$PNPM_HOME:$PATH

RUN pnpm add -g turbo && pnpm build


RUN jq 'del(.pnpm.patchedDependencies)' package.json > package.json.tmp && mv package.json.tmp package.json
RUN node .github/scripts/trim-fe-packageJson.js
RUN find . -type f -name "*.ts" -o -name "*.js.map" -o -name "*.vue" -o -name "tsconfig.json" -o -name "*.tsbuildinfo" | xargs rm -rf

RUN mkdir /compiled
RUN NODE_ENV=production DOCKER_BUILD=true pnpm --filter=n8n --prod --no-optional deploy /compiled

# 2. Clean image
FROM n8nio/n8n:latest
ENV NODE_ENV=production

ARG N8N_RELEASE_TYPE=dev
ENV N8N_RELEASE_TYPE=${N8N_RELEASE_TYPE}

WORKDIR /home/node
COPY --from=builder /compiled /usr/local/lib/node_modules/n8n
COPY docker/images/n8n/docker-entrypoint.sh /

ARG TARGETPLATFORM
ARG LAUNCHER_VERSION=1.1.0
COPY docker/images/n8n/n8n-task-runners.json /etc/n8n-task-runners.json

RUN if [[ "$TARGETPLATFORM" = "linux/amd64" ]]; then export ARCH_NAME="amd64"; \
    elif [[ "$TARGETPLATFORM" = "linux/arm64" ]]; then export ARCH_NAME="arm64"; fi && \
    mkdir /launcher-temp && \
    cd /launcher-temp && \
    wget https://github.com/n8n-io/task-runner-launcher/releases/download/${LAUNCHER_VERSION}/task-runner-launcher-${LAUNCHER_VERSION}-linux-${ARCH_NAME}.tar.gz && \
    wget https://github.com/n8n-io/task-runner-launcher/releases/download/${LAUNCHER_VERSION}/task-runner-launcher-${LAUNCHER_VERSION}-linux-${ARCH_NAME}.tar.gz.sha256 && \
    echo "$(cat task-runner-launcher-${LAUNCHER_VERSION}-linux-${ARCH_NAME}.tar.gz.sha256) task-runner-launcher-${LAUNCHER_VERSION}-linux-${ARCH_NAME}.tar.gz" > checksum.sha256 && \
    sha256sum -c checksum.sha256 && \
    tar xvf task-runner-launcher-${LAUNCHER_VERSION}-linux-${ARCH_NAME}.tar.gz --directory=/usr/local/bin && \
    cd - && rm -r /launcher-temp

RUN cd /usr/local/lib/node_modules/n8n && npm rebuild sqlite3 && \
    cd - && ln -s /usr/local/lib/node_modules/n8n/bin/n8n /usr/local/bin/n8n && \
    mkdir .n8n && chown node:node .n8n

ENV SHELL /bin/sh
USER node
ENTRYPOINT ["tini", "--", "/docker-entrypoint.sh"]

ENV PATH="/opt/venv/bin:$PATH"
