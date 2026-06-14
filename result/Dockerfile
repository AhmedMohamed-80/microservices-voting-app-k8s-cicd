FROM node:18-slim

RUN apt-get update && \
    apt-get install -y --no-install-recommends curl tini && \
    rm -rf /var/lib/apt/lists/* && \
    groupadd -g 1001 appgroup && \
    useradd -u 1001 -g appgroup -s /bin/false -M appuser

WORKDIR /usr/local/app

COPY package*.json ./
RUN npm ci && \
    npm cache clean --force && \
    mv /usr/local/app/node_modules /node_modules

COPY --chown=appuser:appgroup . .

USER appuser

ENV PORT=80
EXPOSE 80
HEALTHCHECK --interval=10s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:${PORT}/ || exit 1
  
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "server.js"]
