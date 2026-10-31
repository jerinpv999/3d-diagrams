# ArchFlow 3D — static app served by nginx.
#
# Stage 1 downloads the exact library versions the app uses from npm and
# rewrites index.html to load them locally, so the container works offline
# (no CDN needed at runtime). Fonts still come from Google Fonts when
# reachable and fall back to system fonts otherwise.
#
#   docker build -t archflow3d .
#   docker run --rm -p 8080:80 archflow3d
#   open http://localhost:8080

# ---------- stage 1: vendor the JavaScript libraries ----------
FROM node:20-alpine AS vendor
WORKDIR /build

RUN npm init -y >/dev/null \
 && npm install --no-audit --no-fund \
      react@18.3.1 react-dom@18.3.1 htm@3.1.1 three@0.147.0

COPY . /site

# index.source.html keeps the original (CDN) page, so "Download project"
# from a running container still gives you a portable project.
RUN mkdir -p /site/vendor \
 && cp /site/index.html /site/index.source.html \
 && cp node_modules/react/umd/react.production.min.js            /site/vendor/ \
 && cp node_modules/react-dom/umd/react-dom.production.min.js    /site/vendor/ \
 && cp node_modules/htm/dist/htm.umd.js                          /site/vendor/ \
 && cp node_modules/three/build/three.min.js                     /site/vendor/ \
 && cp node_modules/three/examples/js/controls/OrbitControls.js  /site/vendor/ \
 && sed -i \
      -e 's#https://cdnjs.cloudflare.com/ajax/libs/react/18.3.1/umd/react.production.min.js#vendor/react.production.min.js#' \
      -e 's#https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.3.1/umd/react-dom.production.min.js#vendor/react-dom.production.min.js#' \
      -e 's#https://cdn.jsdelivr.net/npm/htm@3.1.1/dist/htm.umd.js#vendor/htm.umd.js#' \
      -e 's#https://cdn.jsdelivr.net/npm/three@0.147.0/build/three.min.js#vendor/three.min.js#' \
      -e 's#https://cdn.jsdelivr.net/npm/three@0.147.0/examples/js/controls/OrbitControls.js#vendor/OrbitControls.js#' \
      /site/index.html \
 && ! grep -q 'src="https://' /site/index.html

# ---------- stage 2: serve with nginx ----------
FROM nginx:1.27-alpine

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=vendor /site /usr/share/nginx/html

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://127.0.0.1/healthz >/dev/null || exit 1
