# Self-Hosted Stremio Stack

A high-performance, self-hosted backend for Stremio with Real-Debrid. Uses **Comet** as the addon, **Jackett** for indexer management, **PostgreSQL** for persistent caching, and built-in scrapers (Torrentio, Zilean) for maximum content coverage.

## Features

* **Fast:** Background scraper pre-caches popular content. Most searches resolve from local cache in under 3 seconds.
* **Secure:** No ports exposed by default. PostgreSQL isolated in an internal network with no internet access.
* **Multi-source:** Three scraping layers (Jackett indexers + Torrentio + Zilean) working together for broad coverage.
* **Smart caching:** PostgreSQL stores torrent metadata permanently. Repeated searches are instant.
* **Captcha bypass:** FlareSolverr included for Cloudflare-protected indexers (like 1337x).

## Prerequisites

* **Docker** & **Docker Compose** installed.
* **Real-Debrid** API Key.
* **(Optional)** A reverse proxy (Caddy, Nginx, Traefik) or Tailscale for external access.

## Deployment

### 1. Clone and Configure

```bash
git clone https://github.com/ImJustDoingMyPart/stremio-stack.git
cd stremio-stack
cp .env.example .env
```

Edit `.env` with your settings. You'll need to start the stack once to get the Jackett API Key, then update `.env` and restart.

### 2. Create the proxy network

If you don't already have a shared proxy network:

```bash
docker network create proxy_net
```

If you use Caddy or another reverse proxy, connect it to this same network.

### 3. Start the Stack

```bash
docker compose up -d
```

### 4. Configure Jackett

1. Access Jackett (via your proxy or `http://YOUR-IP:9117` if ports are uncommented).
2. **Set FlareSolverr URL** in Settings: `http://flaresolverr:8191`
3. **Add indexers.** Recommended starting set:
   - **General:** 1337x, The Pirate Bay, TheRARBG, TorrentGalaxyClone, EZTV, YTS, Knaben, LimeTorrents
   - **Anime:** Anime Tosho
   - **Spanish content:** MejorTorrent, DonTorrent
4. Copy the **API Key** from Jackett's top-right corner into your `.env` file.
5. Restart Comet: `docker compose restart comet`

> **Tip:** Avoid adding too many indexers. 8-12 well-chosen ones beat 50 slow ones. The `INDEXER_MANAGER_TIMEOUT` (default 12s) will drop slow indexers from results automatically.

### 5. Configure Comet

1. Access the Comet configuration page (`/configure`).
2. Enter your **Real-Debrid API Key**.
3. Enable **Show Cached Only** for a clean streaming experience.
4. Click **Install** to add it to Stremio.

## Architecture

```
                    ┌─────────────────────────────────────┐
                    │           proxy_net (internet)       │
                    │                                      │
                    │  ┌───────┐  ┌───────┐  ┌──────────┐│
  Stremio ──────────┼─►│ Comet │  │Jackett│  │FlareSolvr││
                    │  └──┬────┘  └──┬────┘  └──────────┘│
                    └─────┼──────────┼────────────────────┘
                          │          │
                    ┌─────┼──────────┼────────────────────┐
                    │     │  internal (no internet)        │
                    │     ▼          ▼                     │
                    │  ┌──────────────────┐               │
                    │  │   PostgreSQL     │               │
                    │  └──────────────────┘               │
                    └─────────────────────────────────────┘
```

* **proxy_net:** External network with internet access. Connects to your reverse proxy.
* **internal:** Isolated network. PostgreSQL lives here with no internet access.
* Comet, Jackett, and FlareSolverr bridge both networks.

## How It Works

Comet pulls torrents from **three sources simultaneously**:

| Source | Type | Speed | Coverage |
|--------|------|-------|----------|
| **Jackett** | Your configured indexers | Depends on indexer | Customizable |
| **Torrentio** | Pre-crawled database | Fast | Massive (mainstream) |
| **Zilean** | DMM hashlist index | Fast | Large (debrid-focused) |

The **background scraper** runs every hour, pre-caching results from all three sources into PostgreSQL. When you search in Stremio, Comet checks the local cache first — if it's fresh enough, you get results in under a second without waiting for any external query.

Content you search manually gets **demand priority**, so it stays fresh in the cache for longer.

## Chromecast / Android TV / iOS

Stremio on these devices requires HTTPS. The easiest way without opening router ports is **Tailscale Serve**:

```bash
# Expose Comet privately to your Tailnet
tailscale serve --bg --https=443 http://localhost:8000
```

Use the generated URL (e.g., `https://my-server.tailnet.ts.net`) as your `PUBLIC_BASE_URL` in `.env`, restart, and reconfigure the addon.

> **Serve** = private to your Tailnet (recommended). **Funnel** = public internet.

## Performance Tuning

All performance variables have sensible defaults in `docker-compose.yml`. Override them in `.env` only if needed:

| Variable | Default | Description |
|----------|---------|-------------|
| `FASTAPI_WORKERS` | 2 | Gunicorn workers (increase for high concurrency) |
| `INDEXER_MANAGER_TIMEOUT` | 12s | Max wait per Jackett indexer |
| `BACKGROUND_SCRAPER_ENABLED` | True | Pre-cache popular content |
| `BACKGROUND_SCRAPER_CONCURRENT_WORKERS` | 2 | Parallel scraping workers |
| `SCRAPE_JACKETT` | both | Jackett scraping mode |
| `SCRAPE_TORRENTIO` | both | Torrentio scraping mode |
| `SCRAPE_ZILEAN` | both | Zilean scraping mode |
| `TORRENT_CACHE_TTL` | -1 | Torrent cache lifetime (-1 = permanent) |
| `LIVE_TORRENT_CACHE_TTL` | 86400 | Seconds before triggering a fresh search |
| `DEBRID_CACHE_TTL` | 86400 | Debrid availability cache (1 day) |

Scraper modes: `live` (on-demand only), `background` (pre-cache only), `both`, `false` (disabled).

## Project Structure

* `docker-compose.yml` — Stack definition with performance defaults.
* `.env.example` — Template for environment variables.
* `Caddyfile` — Optional reverse proxy configuration example.

## Contributing

Feel free to fork this repository and submit pull requests.
