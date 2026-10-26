# SPDX-License-Identifier: AGPL-3.0-or-later

FROM node:26-trixie-slim AS manifests

WORKDIR /usr/src/app

RUN npm install -g pnpm@11.27.0 && pnpm config set store-dir /pnpm/store

# Only the manifests, so a source change keeps the dependency layers cached. The pnpm
# store is a BuildKit cache, so a lockfile change only fetches the new packages.
# The deployed checkout strips "other" permissions (0750/0640); the runtime user is
# not the file owner, so everything copied into the image is made world-readable.
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY patches/ ./patches/
COPY fluxer_music/package.json ./fluxer_music/
COPY packages/music_protocol/package.json ./packages/music_protocol/
RUN chmod -R a+rX .

# Production node_modules for the runtime image; depends on the manifests only.
FROM manifests AS prod-deps
RUN --mount=type=cache,id=fluxer-music-pnpm,target=/pnpm/store \
	pnpm --filter @fluxer/music... install --frozen-lockfile && \
	pnpm --filter @fluxer/music deploy --legacy --prod --config.allow-unused-patches=true /out

FROM manifests AS build
RUN --mount=type=cache,id=fluxer-music-pnpm,target=/pnpm/store \
	pnpm --filter @fluxer/music... install --frozen-lockfile
COPY tsconfigs/ ./tsconfigs/
COPY packages/music_protocol/ ./packages/music_protocol/
COPY fluxer_music/ ./fluxer_music/
RUN pnpm --filter @fluxer/music run build

FROM node:26-trixie-slim

LABEL org.opencontainers.image.title="fluxer-music"
LABEL org.opencontainers.image.description="Fluxer music bot"
LABEL org.opencontainers.image.licenses="AGPL-3.0-or-later"

RUN apt-get update && apt-get install -y --no-install-recommends \
		ffmpeg \
		python3-venv \
		ca-certificates && \
	rm -rf /var/lib/apt/lists/*

# Fallback yt-dlp; the entrypoint tries to update it into /var/lib/ytdlp at start.
RUN python3 -m venv /opt/ytdlp && \
	/opt/ytdlp/bin/pip install --no-cache-dir 'yt-dlp[default]'

RUN useradd --uid 10001 --no-create-home --shell /usr/sbin/nologin music && \
	mkdir -p /var/lib/ytdlp /app && \
	chown 10001:10001 /var/lib/ytdlp

WORKDIR /app

COPY --from=prod-deps /out/node_modules ./node_modules
COPY --from=prod-deps /out/package.json ./package.json
COPY --from=build /usr/src/app/fluxer_music/dist ./dist
COPY --chmod=0755 fluxer_music/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh

ENV NODE_ENV=production
ENV HOME=/tmp
ENV PIP_CACHE_DIR=/tmp/pip-cache

USER 10001:10001

ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
