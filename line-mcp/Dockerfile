# Bridge for @line/line-bot-mcp-server, which only speaks MCP over stdio.
# n8n's MCP Client Tool only supports HTTP Streamable / SSE, so supergateway
# spawns the LINE server as a child process and exposes it at :8000/mcp.
FROM node:22-alpine

# Same as LINE's recommended client config: skip npm lifecycle scripts.
ENV NPM_CONFIG_IGNORE_SCRIPTS=true \
    NPM_CONFIG_UPDATE_NOTIFIER=false

# Pinned so a rebuild never pulls a different server silently.
RUN npm install -g supergateway@3.4.3 @line/line-bot-mcp-server@0.5.0 \
 && npm cache clean --force

USER node
EXPOSE 8000

ENTRYPOINT ["supergateway"]
# Stateful: one LINE server process per MCP session (n8n opens a session per
# tool call run), cleaned up after 10 minutes idle.
CMD ["--stdio", "line-bot-mcp-server", \
     "--outputTransport", "streamableHttp", \
     "--streamableHttpPath", "/mcp", \
     "--port", "8000", \
     "--stateful", "--sessionTimeout", "600000"]
