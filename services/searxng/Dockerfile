FROM searxng/searxng:latest

# Copy configuration files directly into the image
COPY settings.yml /etc/searxng/settings.yml
COPY limiter.toml /etc/searxng/limiter.toml

# Ensure correct permissions
USER root
RUN chmod 644 /etc/searxng/settings.yml /etc/searxng/limiter.toml && \
    chown searxng:searxng /etc/searxng/settings.yml /etc/searxng/limiter.toml && \
    chown -R searxng:searxng /etc/ssl/certs

USER searxng
