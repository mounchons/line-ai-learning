# Bridge for @line/line-bot-mcp-server, which only speaks MCP over stdio.
# n8n's MCP Client Tool only supports HTTP Streamable / SSE, so supergateway
# spawns the LINE server as a child process and exposes it at :8000/mcp.
FROM node:22-alpine

# create_rich_menu renders its menu image with Marp + Puppeteer, so the image
# needs a browser. NPM_CONFIG_IGNORE_SCRIPTS below skips Puppeteer's Chromium
# download, so point it at Alpine's package instead. Without this the tool
# fails with "Could not find Chrome".
# font-noto-thai: rich menu labels are Thai; without it they render as tofu.
RUN apk add --no-cache \
      chromium \
      nss \
      freetype \
      harfbuzz \
      ca-certificates \
      ttf-freefont \
      font-noto \
      font-noto-thai

ENV PUPPETEER_SKIP_DOWNLOAD=true \
    PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser \
    CHROME_PATH=/usr/bin/chromium-browser

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
