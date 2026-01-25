# Pin version linuxserver BookStack
ARG BOOKSTACK_VERSION=25.12.2
FROM lscr.io/linuxserver/bookstack:version-v${BOOKSTACK_VERSION}

ARG BUILD_SHA=dev
LABEL org.opencontainers.image.revision=$BUILD_SHA

# Trace build (utile pour debug)
RUN echo "${BUILD_SHA}" > /build_sha.txt

# 1) Ajoute la CSS dans le "public" BookStack (servi par l'app)
COPY src/rncp.css /app/www/public/rncp.css

# 2) Injecte <link rel="stylesheet" href="/rncp.css"> dans un layout Blade
# On cherche un fichier dans resources/views/layouts qui contient </head>
# et on patch si pas déjà patché.
RUN set -e; \
  TARGET="$(grep -Rsl "</head>" /app/www/resources/views/layouts 2>/dev/null | head -n 1 || true)"; \
  if [ -n "$TARGET" ]; then \
    echo "Patching layout: $TARGET"; \
    grep -q "rncp.css" "$TARGET" || sed -i 's#</head>#  <link rel="stylesheet" href="/rncp.css" />\n</head>#' "$TARGET"; \
  else \
    echo "No layout found under /app/www/resources/views/layouts containing </head> (skip)"; \
  fi

# 3) Clear view cache
RUN php /app/www/artisan view:clear || true
