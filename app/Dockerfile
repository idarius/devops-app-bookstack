# Pin version linuxserver BookStack
ARG BOOKSTACK_VERSION=25.12.3
FROM lscr.io/linuxserver/bookstack:version-v${BOOKSTACK_VERSION}

# applique les correctifs Alpine (openssl/libssl/libcrypto)
RUN apk update && apk upgrade --no-cache

ARG BUILD_SHA=dev
LABEL org.opencontainers.image.revision=$BUILD_SHA

# Trace build si besoin de debug
RUN echo "${BUILD_SHA}" > /build_sha.txt

# Ajoute la CSS dans le dossier public de BookStack
COPY src/rncp.css /app/www/public/rncp.css

# Injecte le CSS dans un layout Blade si pas déjà présent
RUN set -e; \
  TARGET="$(grep -Rsl "</head>" /app/www/resources/views/layouts 2>/dev/null | head -n 1 || true)"; \
  if [ -n "$TARGET" ]; then \
    echo "Patching layout: $TARGET"; \
    grep -q "rncp.css" "$TARGET" || sed -i 's#</head>#  <link rel="stylesheet" href="/rncp.css" />\n</head>#' "$TARGET"; \
  else \
    echo "No layout found under /app/www/resources/views/layouts containing </head> (skip)"; \
  fi

# Vider le cache des vues
RUN php /app/www/artisan view:clear || true
