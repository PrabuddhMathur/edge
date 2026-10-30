# edge

Caddy for the Hetzner server, on its own. It's the only container that faces the internet: it
gets the HTTPS certificates and hands each site's traffic to the app behind it.

| Site | App | Its Caddyfile |
|---|---|---|
| `prabuddh.in`, `www.`, `stats.` | the blog (`/srv/prabuddh-me`) | `docker/Caddyfile` in the blog's repository |
| `photos.`, `api.photos.`, `albums.` | Ente Photos (`/srv/ente`) | `caddy/Caddyfile` in `my-ente` |

**Why on its own:** while Caddy was part of the blog's Compose project, `docker compose down`
on the blog took every site on the server offline with it, Ente included. Now stopping an app
stops only that app, and Caddy answers its sites with an error until it's back.

**How the pieces meet:**

- **One Docker network, `edge`,** made once by hand. Caddy and each app's web-facing
  containers join it; databases stay on their app's own network. Because no Compose project
  owns it, no `docker compose down` removes it. Names on it are shared by every app, so two
  apps must never put a service with the same name on it (the blog uses `backend`, `frontend`
  and `umami`; Ente `museum` and `web`).
- **Each app keeps its own Caddyfile in its own repository.** `compose.yaml` mounts each app's
  folder at `/sites/<app>`, and `caddy/Caddyfile` imports `/sites/*/Caddyfile`. A change to the
  blog's routing still ships with the blog: its deploy validates the whole configuration and
  runs `caddy reload`.
- **Settings shared by the sites** (`DOMAIN`, `HSTS_SECONDS`, `ADMIN_ALLOWED_IPS`) live in this
  folder's `.env`. See `.env.example`.

## Setting it up

Run these as the `deploy` user (`sudo -iu deploy`), like everything else in `/srv`.

1. **The network, once per server.** IPv6 on, so Caddy sees IPv6 visitors' own addresses.
   Without it, Docker hands them all over from its own address: they'd share one comment and
   signup limit, and the admin allowlist would refuse your devices over IPv6 (the blog's
   decision E11). Docker picks the IPv6 range itself (Engine 27 or newer).

   ```sh
   docker network create --ipv6 edge
   docker network inspect edge --format '{{.EnableIPv6}}'   # true
   ```

2. **Each app's folder, before Caddy.** Caddy refuses to start while a folder it mounts is
   missing: `/srv/prabuddh-me` (cloned, and started once, so its uploads volume exists) and
   `/srv/ente`.

3. **This folder, at `/srv/edge`.**

   ```sh
   sudo mkdir -p /srv/edge && sudo chown deploy:deploy /srv/edge   # as your own user
   git clone <this repository> /srv/edge                           # as deploy
   cd /srv/edge
   cp .env.example .env && chmod 600 .env
   nano .env
   ```

   On a new server, start with `ADMIN_ALLOWED_IPS=127.0.0.1/32`, which closes the admin and
   the Umami dashboard to everyone until the blog's Tailscale guide replaces it.

4. **Start.** The DNS records of every site must already point at the server, or Let's
   Encrypt can't check them and Caddy keeps retrying.

   ```sh
   docker compose up -d
   docker compose logs -f caddy
   ```

## Day to day

- **A site's Caddyfile changed.** The blog's deploy reloads Caddy itself. For any other app,
  after `git pull` in its folder:

  ```sh
  docker exec caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
  docker exec caddy caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
  ```

  A configuration that doesn't validate is never loaded: the sites keep the old one.

- **Changing `.env`.** Check the new values in a throwaway container first: a typing mistake
  in `ADMIN_ALLOWED_IPS` would stop Caddy, and every site with it. Then recreate Caddy so it
  reads them (a plain `restart` keeps the old ones).

  ```sh
  docker compose run --rm --no-deps caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
  docker compose up -d
  ```

- **Locked out of the admin.** Put a `#` in front of `ADMIN_ALLOWED_IPS` in `.env` and run
  `docker compose up -d`. The admin is open to everyone until you set it back.

- **Upgrading Caddy.** Read the [release notes](https://github.com/caddyserver/caddy/releases),
  change the tag in `compose.yaml` and in the blog's CI (`.github/workflows/ci.yml`, which
  validates the blog's Caddyfile with it), then `docker compose up -d`.

## Adding a site

1. Give the app a folder with its `Caddyfile` in it and nothing secret (Caddy can read the
   whole folder), and put its web-facing containers on the `edge` network (`external: true`
   in its compose file), with names no other app uses.
2. Mount that folder in `compose.yaml` at `/sites/<app>`, the same way as the others.
3. `docker compose up -d`.
