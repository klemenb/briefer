FROM node:18-slim AS base

ENV TURBO_TELEMETRY_DISABLED=1

RUN apt-get update -y && apt-get install -y openssl curl tini && rm -rf /var/lib/apt/lists/*

# The web Dockerfile is copy-pasted into our main docs at /docs/handbook/deploying-with-docker.
# Make sure you update this Dockerfile, the Dockerfile in the web workspace and copy that over to Dockerfile in the docs.

FROM base AS builder
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
# RUN apk add --no-cache libc6-compat
# RUN apk update
# Set working directory
WORKDIR /app
RUN yarn global add turbo
COPY . .
RUN turbo prune @briefer/api --docker

# Add lockfile and package.json's of isolated subworkspace
FROM base AS installer
# RUN apk add --no-cache libc6-compat
# RUN apk update

WORKDIR /app

# First install dependencies (as they change less often)
COPY .gitignore .gitignore
COPY --from=builder /app/out/json/ .
COPY --from=builder /app/out/yarn.lock ./yarn.lock

RUN yarn install

# Build the project and its dependencies
COPY --from=builder /app/out/full/ .
COPY turbo.json turbo.json

# Uncomment and use build args to enable remote caching
# ARG TURBO_TEAM
# ENV TURBO_TEAM=$TURBO_TEAM

# ARG TURBO_TOKEN
# ENV TURBO_TOKEN=$TURBO_TOKEN

RUN yarn turbo run build --filter=@briefer/api...

FROM base AS runner
WORKDIR /app

# Don't run production as root
RUN groupadd -r expressjs && useradd -r -g expressjs expressjs \
    && mkdir -p /home/expressjs/Downloads \
    && chown -R expressjs:expressjs /home/expressjs

COPY --from=installer /app .
# change permissions to allow prisma to write to the filesystem
RUN chown -R expressjs:expressjs /app/node_modules/@prisma/engines

USER expressjs

# Use tini as the init system so that signals are forwarded to the node process
ENTRYPOINT ["tini", "--"]
CMD ["node", "--max-old-space-size=12000", "apps/api/dist/src/index.js"]
