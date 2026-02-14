# OpenClaw VPS Setup Guide with Telegram Bot

A complete step-by-step guide to set up OpenClaw on a VPS with Telegram integration.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Server Preparation](#step-1-server-preparation)
- [Step 2: Install Docker](#step-2-install-docker)
- [Step 3: Install PostgreSQL and Redis](#step-3-install-postgresql-and-redis)
- [Step 4: Create Telegram Bot](#step-4-create-telegram-bot)
- [Step 5: Get API Keys](#step-5-get-api-keys)
- [Step 6: Clone and Configure OpenClaw](#step-6-clone-and-configure-openclaw)
- [Step 7: Build and Run OpenClaw](#step-7-build-and-run-openclaw)
- [Step 8: Enable Telegram and Pair Device](#step-8-enable-telegram-and-pair-device)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)

---

## Prerequisites

- VPS running Ubuntu 20.04 or newer (minimum 2GB RAM recommended)
- SSH access to your VPS
- Domain name (optional, but recommended for webhooks)
- Basic command line knowledge

---

## Step 1: Server Preparation

### Connect to your VPS

```bash
ssh root@your_vps_ip_address
```

### Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

### Install basic dependencies

```bash
sudo apt install -y git curl wget nano build-essential
```

---

## Step 2: Install Docker

### Install Docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

### Add your user to docker group

```bash
sudo usermod -aG docker $USER
```

### Install Docker Compose

```bash
sudo apt install docker-compose -y
```

### Log out and back in for group changes to take effect

```bash
exit
# Then reconnect via SSH
ssh root@your_vps_ip_address
```

---

## Step 3: Install PostgreSQL and Redis

### Install PostgreSQL

```bash
sudo apt install postgresql postgresql-contrib -y
```

### Create database and user

```bash
# Switch to PostgreSQL user
sudo -i -u postgres

# Create database
createdb your_project_name

# Create user and set permissions
psql
```

In the PostgreSQL prompt, run:

```sql
CREATE USER your_project_name WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE your_project_name TO your_project_name;
\q
```

Exit postgres user:

```bash
exit
```

### Install and start Redis

```bash
sudo apt install redis-server -y
sudo systemctl start redis-server
sudo systemctl enable redis-server
```

### Test Redis is running

```bash
redis-cli ping
# Should return: PONG
```

---

## Step 4: Create Telegram Bot

1. Open Telegram and search for **@BotFather**
2. Send `/newbot`
3. Follow the prompts:
   - Choose a name for your bot
   - Choose a username (must end with 'bot')
4. Save the bot token that BotFather gives you (looks like `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`)

---

## Step 5: Get API Keys

### Anthropic API Key

1. Go to https://console.anthropic.com
2. Sign up or log in
3. Navigate to "API Keys"
4. Click "Create Key"
5. Copy and save the key (starts with `sk-ant-`)
6. Add payment method and credits ($5 minimum)

### OpenAI API Key

1. Go to https://platform.openai.com
2. Sign up or log in
3. Navigate to https://platform.openai.com/api-keys
4. Click "+ Create new secret key"
5. Copy and save the key (starts with `sk-`)
6. Add payment method in Settings → Billing

⚠️ **IMPORTANT**: Never share these API keys publicly!

---

## Step 6: Clone and Configure OpenClaw

### Clone the repository

```bash
cd /opt
sudo git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

### Create required directories

```bash
sudo mkdir -p /opt/openclaw/config
sudo mkdir -p /opt/openclaw/workspace
sudo chmod -R 777 /opt/openclaw/config
sudo chmod -R 777 /opt/openclaw/workspace
```

### Create .env file

```bash
sudo nano .env
```

Add the following (replace with your actual values):

```env
# Telegram Bot Token (from @BotFather)
TELEGRAM_BOT_TOKEN=your_telegram_bot_token_here

# API Keys
OPENAI_API_KEY=your_openai_api_key_here
ANTHROPIC_API_KEY=your_anthropic_api_key_here

# Database Configuration
DATABASE_URL=postgresql://your_project_name:your_secure_password@postgres:5432/your_project_name
REDIS_URL=redis://redis:6379

# Gateway Configuration
OPENCLAW_GATEWAY_TOKEN=generate_random_token_here

# Claude Session Keys (optional - leave blank)
CLAUDE_AI_SESSION_KEY=
CLAUDE_WEB_SESSION_KEY=
CLAUDE_WEB_COOKIE=

# Directories
OPENCLAW_CONFIG_DIR=/opt/openclaw/config
OPENCLAW_WORKSPACE_DIR=/opt/openclaw/workspace

# Gateway Settings
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_BRIDGE_PORT=18790
OPENCLAW_GATEWAY_BIND=lan
```

**To generate a secure gateway token:**

```bash
openssl rand -hex 32
```

Copy the output and paste it as your `OPENCLAW_GATEWAY_TOKEN`.

Save the file: `Ctrl+O`, `Enter`, then exit: `Ctrl+X`

### Update docker-compose.yml to pass environment variables

```bash
sudo nano docker-compose.yml
```

Make sure both services have the API keys in their environment sections:

```yaml
services:
  openclaw-gateway:
    image: ${OPENCLAW_IMAGE:-openclaw:local}
    environment:
      HOME: /home/node
      TERM: xterm-256color
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      CLAUDE_AI_SESSION_KEY: ${CLAUDE_AI_SESSION_KEY}
      CLAUDE_WEB_SESSION_KEY: ${CLAUDE_WEB_SESSION_KEY}
      CLAUDE_WEB_COOKIE: ${CLAUDE_WEB_COOKIE}
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
    ports:
      - "${OPENCLAW_GATEWAY_PORT:-18789}:18789"
      - "${OPENCLAW_BRIDGE_PORT:-18790}:18790"
    init: true
    restart: unless-stopped
    command:
      [
        "node",
        "dist/index.js",
        "gateway",
        "--bind",
        "${OPENCLAW_GATEWAY_BIND:-lan}",
        "--port",
        "18789",
        "--allow-unconfigured"
      ]
  openclaw-cli:
    image: ${OPENCLAW_IMAGE:-openclaw:local}
    environment:
      HOME: /home/node
      TERM: xterm-256color
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      BROWSER: echo
      CLAUDE_AI_SESSION_KEY: ${CLAUDE_AI_SESSION_KEY}
      CLAUDE_WEB_SESSION_KEY: ${CLAUDE_WEB_SESSION_KEY}
      CLAUDE_WEB_COOKIE: ${CLAUDE_WEB_COOKIE}
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
    stdin_open: true
    tty: true
    init: true
    entrypoint: ["node", "dist/index.js"]
```

Save and exit.

---

## Step 7: Build and Run OpenClaw

### Build the Docker image

```bash
cd /opt/openclaw
sudo docker build -t openclaw:local .
```

This will take 5-10 minutes. Wait for "Successfully tagged openclaw:local" message.

### Start the services

```bash
sudo docker-compose up -d
```

### Check if containers are running

```bash
sudo docker-compose ps
```

### View logs

```bash
sudo docker-compose logs -f openclaw-gateway
```

You should see:
- `[gateway] listening on ws://0.0.0.0:18789`
- `[heartbeat] started`
- `[telegram] starting provider`

---

## Step 8: Enable Telegram and Pair Device

### Enable Telegram in config

```bash
sudo nano /opt/openclaw/config/openclaw.json
```

Find the section:

```json
"telegram": {
  "enabled": false
}
```

Change to:

```json
"telegram": {
  "enabled": true
}
```

Save and exit.

### Restart the gateway

```bash
sudo docker-compose restart openclaw-gateway
```

### Watch the logs

```bash
sudo docker-compose logs -f openclaw-gateway
```

You should see:
```
[telegram] [default] starting provider (@yourbotname)
```

### Message your bot

1. Open Telegram
2. Search for your bot username
3. Send `/start`

You'll receive a pairing code like:

```
OpenClaw: access not configured.
Your Telegram user id: 1234567890
Pairing code: ABCD1234
Ask the bot owner to approve with:
openclaw pairing approve telegram ABCD1234
```

### Approve the pairing

```bash
sudo docker exec openclaw_openclaw-gateway_1 node dist/index.js pairing approve telegram ABCD1234
```

Replace `ABCD1234` with your actual pairing code.

### Test your bot

Send any message to your bot:
- "Hello!"
- "What's the weather today?"
- "Help me write an email"

Your bot should now respond! 🎉

---

## Troubleshooting

### Container keeps restarting

Check logs for errors:

```bash
sudo docker-compose logs openclaw-gateway --tail=100
```

### Telegram not connecting

1. Verify your bot token is correct in `.env`
2. Check that Telegram is enabled in `openclaw.json`
3. Restart the gateway: `sudo docker-compose restart openclaw-gateway`

### API key errors

1. Verify keys are correct in `.env`
2. Check API key validity on provider websites
3. Ensure you have credits/payment method set up

### Database connection errors

Test database connection:

```bash
psql -U your_project_name -d your_project_name -h localhost
```

If it fails, recreate the user/database following Step 3.

### View all logs

```bash
# Gateway logs
sudo docker-compose logs -f openclaw-gateway

# All logs
sudo docker-compose logs -f

# Last 50 lines
sudo docker-compose logs --tail=50
```

### Restart everything

```bash
sudo docker-compose down
sudo docker-compose up -d
```

### Rebuild from scratch

```bash
sudo docker-compose down
sudo docker build -t openclaw:local . --no-cache
sudo docker-compose up -d
```

---

## Security Best Practices

### 1. Never commit secrets to Git

Add to `.gitignore`:

```
.env
openclaw.json
*.log
```

### 2. Use strong passwords

- Database passwords: minimum 16 characters, mix of letters/numbers/symbols
- Gateway token: use `openssl rand -hex 32`

### 3. Firewall configuration

```bash
sudo ufw allow 22    # SSH
sudo ufw allow 80    # HTTP (optional)
sudo ufw allow 443   # HTTPS (optional)
sudo ufw enable
```

### 4. Keep API keys secure

- Never share in screenshots, logs, or messages
- Rotate keys periodically
- Set usage limits in provider dashboards
- Monitor API usage regularly

### 5. Regular updates

```bash
cd /opt/openclaw
sudo git pull
sudo docker build -t openclaw:local .
sudo docker-compose restart
```

### 6. Backup your data

```bash
# Backup database
sudo -u postgres pg_dump your_project_name > backup.sql

# Backup config
sudo cp /opt/openclaw/config/openclaw.json ~/openclaw_config_backup.json
```

---

## Useful Commands

### Check service status

```bash
# Docker containers
sudo docker-compose ps

# PostgreSQL
sudo systemctl status postgresql

# Redis
sudo systemctl status redis-server
```

### Stop/start services

```bash
# Stop all
sudo docker-compose down

# Start all
sudo docker-compose up -d

# Restart specific service
sudo docker-compose restart openclaw-gateway
```

### Monitor logs in real-time

```bash
sudo docker-compose logs -f openclaw-gateway
```

### Access container shell

```bash
sudo docker exec -it openclaw_openclaw-gateway_1 /bin/sh
```

---

## Additional Resources

- OpenClaw Documentation: https://docs.openclaw.ai
- OpenClaw GitHub: https://github.com/openclaw/openclaw
- Anthropic API Docs: https://docs.anthropic.com
- OpenAI API Docs: https://platform.openai.com/docs
- Telegram Bot API: https://core.telegram.org/bots/api

---

## Contributing

Found an issue or want to improve this guide? Please submit a pull request or open an issue!

---

## License

This guide is provided as-is for educational purposes.

---

## Acknowledgments

Special thanks to the OpenClaw team for creating this amazing AI agent framework!

---

**Setup complete! Enjoy your OpenClaw Telegram bot! 🤖✨** # CLAWDBOT
