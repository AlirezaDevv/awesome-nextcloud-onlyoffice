# Self-Hosted Nextcloud & ONLYOFFICE with Docker

This repository provides a complete, secure, and robust setup for deploying a self-hosted cloud suite using Docker Compose. It includes:

* **Nextcloud**: The powerful, open-source, self-hosted productivity platform.
* **ONLYOFFICE Docs**: A feature-rich office suite for editing documents, spreadsheets, and presentations.
* **MariaDB**: A reliable open-source database for Nextcloud.
* **Nginx Proxy Manager**: An easy-to-use reverse proxy for handling SSL certificates and routing traffic.

This guide will walk you through setting up a secure, HTTPS-enabled instance of Nextcloud fully integrated with ONLYOFFICE.

## Architecture

The setup uses a reverse proxy architecture. All incoming web traffic on ports 80 and 443 is handled by Nginx Proxy Manager (NPM). NPM then routes the traffic internally to the correct service (either Nextcloud or ONLYOFFICE) based on the requested domain name. NPM also automatically obtains and renews SSL certificates from Let's Encrypt.

`User Browser` -> `Internet` -> `Your Router (Ports 80/443)` -> `Server` -> `Nginx Proxy Manager` -> `(Nextcloud or ONLYOFFICE)`

## Prerequisites

Before you begin, ensure you have the following:

* A server (PC, VPS, or dedicated server) running a Docker-supported OS (e.g., Ubuntu, Debian, Windows).
* **Docker** and **Docker Compose** installed on the server.
* A **domain name** that you own (e.g., `yourdomain.com`).
* Access to your domain's **DNS management panel**.
* Access to your network **router's admin panel** for port forwarding.

## 🚀 Setup Guide

### Step 1: Clone the Repository

Clone this repository to your server.

```bash
git clone [https://github.com/your-username/your-repo-name.git](https://github.com/your-username/your-repo-name.git)
cd your-repo-name
```

### Step 2: Configure Environment Variables

Create your environment configuration file by copying the example file.

```bash
cp .env.example .env
```

Now, open the `.env` file with a text editor (`nano .env`) and fill in your own secure passwords and domain names.

### Step 3: DNS Setup

Log in to your DNS provider's panel and create **two `A` records**:

1.  **For Nextcloud:**
    * **Type:** `A`
    * **Name/Host:** `nextcloud` (or your preferred subdomain)
    * **Value/Points to:** Your server's **public IP address**.

2.  **For ONLYOFFICE:**
    * **Type:** `A`
    * **Name/Host:** `onlyoffice` (or your preferred subdomain)
    * **Value/Points to:** Your server's **public IP address**.

> **Note:** If your provider offers a proxy service (like Cloudflare or ArvanCloud's orange cloud), **enable it** for both records. This protects your server's IP and often simplifies SSL.

### Step 4: Port Forwarding & Firewall

You must allow public traffic to reach Nginx Proxy Manager.

1.  **Router:** In your router's admin panel, set up port forwarding rules to forward incoming **TCP** traffic on ports **80** and **443** to the **local IP address** of your server.
2.  **Server Firewall:** On your server's OS firewall (e.g., Windows Defender Firewall, `ufw` on Linux), create rules to allow incoming TCP traffic on ports **80** and **443**.

### Step 5: Launch the Stack

Start all the services using Docker Compose.

```bash
docker-compose up -d
```

### Step 6: Configure Nginx Proxy Manager (NPM)

1.  Open your browser and navigate to the NPM admin panel: `http://<your_server_local_ip>:8181`
2.  Log in with the default credentials and change your password immediately:
    * **Email:** `admin@example.com`
    * **Password:** `changeme`
3.  Go to **Hosts** -> **Proxy Hosts** and click **"Add Proxy Host"**.

#### Create the Nextcloud Proxy Host:
* **Details Tab:**
    * `Domain Names`: Your Nextcloud domain (e.g., `nextcloud.yourdomain.com`).
    * `Scheme`: `http`
    * `Forward Hostname / IP`: `app` (the service name in `docker-compose.yml`).
    * `Forward Port`: `80`
* **SSL Tab:**
    * `SSL Certificate`: Select **"Request a new SSL Certificate"**.
    * Enable **"Force SSL"**.
    * Agree to the Let's Encrypt ToS and click **Save**.

#### Create the ONLYOFFICE Proxy Host:
* Click **"Add Proxy Host"** again.
* **Details Tab:**
    * `Domain Names`: Your ONLYOFFICE domain (e.g., `onlyoffice.yourdomain.com`).
    * `Scheme`: `http`
    * `Forward Hostname / IP`: `onlyoffice`
    * `Forward Port`: `80`
* **SSL Tab:**
    * Do the same as above: Request a new certificate and enable "Force SSL".
* **Advanced Tab:**
    * Paste the following code into the text box. This is **critical** to fix the "Download Failed" error.
      ```nginx
      location / {
          proxy_pass http://onlyoffice:80;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
          proxy_set_header Host $host;
          proxy_set_header X-Forwarded-Proto https;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      }
      ```
* Click **Save**.

### Step 7: Final Nextcloud Configuration

1.  Navigate to your Nextcloud instance at `https://nextcloud.yourdomain.com`.
2.  Log in with the admin credentials you set in the `.env` file.
3.  Install the **ONLYOFFICE** connector app from the Nextcloud Apps store.
4.  Go to **Settings** -> **Administration** -> **ONLYOFFICE**.
5.  Configure the addresses as follows:
    * `ONLYOFFICE Docs address`: `https://onlyoffice.yourdomain.com/`
    * Expand **"Advanced server settings"**:
    * `...address for internal requests from the server`: `http://onlyoffice/`
    * `...address for internal requests from ONLYOFFICE Docs`: `http://app/`
6.  Click **Save**.

**Congratulations! Your self-hosted cloud suite is now ready!**

## Troubleshooting

* **502 Bad Gateway:** Ensure all containers are running (`docker-compose ps`). Check the Nginx Proxy Manager logs (`docker-compose logs npm`).
* **SSL "Internal Error":** Your server cannot be reached from the internet on **port 80**. Check your port forwarding and firewall rules. Use an online port checker tool to verify.
* **ONLYOFFICE "Download Failed":** You missed adding the code to the **Advanced** tab for the ONLYOFFICE proxy host in NPM.
* **Can't access from inside my network (Hairpin NAT):** If you are on the same local network as the server, your router might not support NAT Loopback. For testing, you can edit the `hosts` file on your client computer to point the domains directly to the server's local IP.
