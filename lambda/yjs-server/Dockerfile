FROM public.ecr.aws/docker/library/node:24.15.0-alpine3.23@sha256:d1b3b4da11eefd5941e7f0b9cf17783fc99d9c6fc34884a665f40a06dbdfc94f

WORKDIR /app

COPY --chown=node:node --chmod=0644 yjs-server/package.json yjs-server/package-lock.json ./
RUN npm ci --omit=dev

COPY --chown=node:node --chmod=0644 shared/space-permissions.js shared/space-membership.js shared/business-roles.js shared/business-role-catalog.json ./shared/
COPY --chown=node:node --chmod=0644 shared/policies/space.cedar ./shared/policies/
COPY --chown=node:node --chmod=0644 yjs-server/*.js ./yjs-server/

RUN chmod 0755 /app/shared /app/shared/policies /app/yjs-server

WORKDIR /app/yjs-server

USER node

EXPOSE 1234

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --spider -q http://localhost:1234/livez || exit 1

CMD ["node", "server.js"]
