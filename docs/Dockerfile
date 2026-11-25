# ==============================================================================
# GastroCore ERP • Dockerfile Multi-Stage de Produção (DevOps APONTI 2026)
# Builder Node 20 Alpine -> Runner Nginx Alpine SPA (< 60 MB, Non-Root)
# ==============================================================================

# STAGE 1: Build da SPA Vite + React 19
FROM node:20-alpine AS builder

WORKDIR /app

# Variáveis de build do Vite (injetadas via --build-arg no CI/Compose)
ARG VITE_APP_ENV=homologacao
ARG VITE_SUPABASE_URL=""
ARG VITE_SUPABASE_ANON_KEY=""

ENV VITE_APP_ENV=${VITE_APP_ENV}
ENV VITE_SUPABASE_URL=${VITE_SUPABASE_URL}
ENV VITE_SUPABASE_ANON_KEY=${VITE_SUPABASE_ANON_KEY}

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run lint && npm run build

# ==============================================================================
# STAGE 2: Runner Nginx Alpine (Somente estáticos compilados + SPA Fallback)
# ==============================================================================
FROM nginx:1.27-alpine AS runner

# Configura Nginx na porta 8080 (não-privilegiada) com fallback SPA e gzip
RUN cat <<'EOF' > /etc/nginx/conf.d/default.conf
server {
  listen 8080;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;

  gzip on;
  gzip_types text/plain text/css application/javascript application/json image/svg+xml;

  location /assets/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
  }

  location / {
    try_files $uri $uri/ /index.html;
  }
}
EOF

COPY --from=builder /app/dist /usr/share/nginx/html

# Permissões para executar como usuário sem privilégios de root (nginx UID 101)
RUN chown -R nginx:nginx /usr/share/nginx/html /var/cache/nginx /var/log/nginx /etc/nginx/conf.d && \
    touch /var/run/nginx.pid && chown nginx:nginx /var/run/nginx.pid

USER nginx

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/ || exit 1

CMD ["nginx", "-g", "daemon off;"]
