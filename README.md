# 🚀 Self-Hosted Stremio Stack

A high-performance, secure, and clean self-hosted backend for Stremio.

This stack replaces the traditional Prowlarr setup with **Jackett** for better handling of public indexers, uses **PostgreSQL** for persistent caching, and **Comet** as the high-speed addon. It is designed to be deployed behind a Reverse Proxy (like Caddy) or via Tailscale, keeping all ports closed to the public internet for maximum security.

## ✨ Features

* **⚡ Blazing Fast:** Optimized configuration to serve cached results in <2 seconds.
* **🔒 Secure:** No ports exposed to the host by default. All traffic is routed through an internal Docker network.
* **🧠 Smart Caching:** Uses PostgreSQL to remember search results, making repeated queries instant.
* **🧹 Clean UI:** Configured to filter out raw P2P links and only show "Real-Debrid Cached" results (requires Real-Debrid).
* **🛡️ Captcha Solver:** Includes FlareSolverr to bypass Cloudflare protection on supported indexers.

## 🛠️ Prerequisites

* **Docker** & **Docker Compose** installed.
* **Real-Debrid** API Key (Required for the "Cached Only" experience).
* **(Optional)** A Reverse Proxy (Caddy, Nginx, Traefik) OR Tailscale (for external access).

## 🚀 Deployment

### 1. Clone and Configure
Clone this repository and create your environment file:

```bash
# Copy the example file
cp .env.example .env
Edit the .env file with your preferred settings. Note: You will need to start the stack once to generate the Jackett API Key, then update the .env file and restart.
```

### 2. Start the Stack
```bash
docker-compose up -d
```
### 3. Final Configuration (Critical Steps)
#### A. Jackett Configuration (The Performance Secret)
1. Access Jackett (e.g., http://localhost:9117 or via your proxy).

2. Add Indexers: This stack does not come with pre-configured indexers. You must add them manually.

> ⚠️ Performance Tip: * Do not add 50 indexers blindly. This will slow down your search significantly.

3. Test manually: Add a few public indexers and use the "Test" button or perform a manual search.

> ⚠️ The 4-Second Rule: If an indexer takes more than 3-4 seconds to return results, delete it. Comet waits for the slowest indexer. A lean list of fast indexers is better than a huge list of slow ones.

4. Copy API Key: Grab the API Key from the top-right corner of the Jackett dashboard and paste it into your .env file (JACKETT_API_KEY).

5. Restart the stack: docker-compose restart comet.

#### B. Comet Setup
1. Access the Comet configuration page (e.g., http://localhost:8000/configure or your domain).

2. Real-Debrid API Key: Enter your RD token.

3. Max Results: Set to 10 or 15 (prevents infinite scrolling).

4. Show Cached Only: ✅ CHECK THIS. This ensures you only see content that is ready to stream instantly.

5. Deduplicate Streams: ✅ CHECK THIS to keep your list clean.

6. Click Install to add it to Stremio.

## 📺 How to use on Chromecast / Android TV / iOS
Stremio Web and devices like Chromecast often require a valid HTTPS connection to load external addons. Local HTTP addresses (like http://192.168.1.x:8000) often fail or get blocked by Mixed Content policies.

The easiest way to make this work without opening ports on your router is using Tailscale Funnel.

A. Step-by-Step for Tailscale Users:
1. Install Tailscale on your server.

2. Enable Funnel in your Tailscale Admin Console (Access Controls).

3. Run the following command on your server to expose the Comet container publicly via a secure Tailscale URL:

```Bash
# Assuming Comet is running on port 8000 internally
sudo tailscale funnel --bg --https=443 http://localhost:8000
```

4. Tailscale will generate a public URL (e.g., https://my-server.tailnet-name.ts.net). Use that URL as your BASE_URL in the .env file.

5. Restart Comet.

6. Configure the addon using the new HTTPS URL. Now it will work perfectly on Chromecast, iOS, and Smart TVs outside your network.

## 📂 Project Structure
* docker-compose.yml: Main stack definition.

* .env.example: Template for environment variables.

* Caddyfile: (Optional) Configuration example if you prefer using Caddy instead of Tailscale.

## ⚠️ Notes
* **Security** The ports section in docker-compose.yml is commented out by default. This is intentional. Traffic should flow through the internal Docker network (proxy_net) to your Reverse Proxy or Tailscale. If you need direct IP access, uncomment the ports in the YAML file.

* **Logs**: If searches are slow, check the Jackett logs. Identify which indexer is timing out and remove it.

## 🤝 Contributing
Feel free to fork this repository and submit pull requests to improve the configuration or documentation.