FROM ghcr.io/constellatione/mastodon:v4.0.2-1
WORKDIR /opt/mastodon
EXPOSE 3000
ENTRYPOINT [ "bundle", "exec", "rails", "s", "-b", "::", "-p", "3000" ]