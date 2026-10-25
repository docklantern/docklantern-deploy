# Match the version and digest used by the application repository.
# Build locally to apply available Alpine package security fixes.
FROM tecnativa/docker-socket-proxy:v0.5.0@sha256:1f5038b54f06c3e18422902cf00ba21803d1c97805aae032e5e6673d532d3459
USER root
RUN apk upgrade --no-cache
