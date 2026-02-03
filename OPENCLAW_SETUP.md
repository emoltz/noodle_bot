# OpenClaw Setup Guide - Raspberry Pi Deployment

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Docker Installation](#docker-installation)
4. [OpenClaw Installation](#openclaw-installation)
5. [Gateway Configuration](#gateway-configuration)
6. [Messaging Platform Setup](#messaging-platform-setup)
7. [Custom Skills](#custom-skills)
8. [Remote Access](#remote-access)
9. [Troubleshooting](#troubleshooting)
10. [LangChain Integration](#langchain-integration)

---

## Prerequisites

### Hardware Requirements
- **Raspberry Pi 4 (4GB RAM minimum, 8GB recommended)**
- MicroSD card (32GB+ recommended)
- Stable power supply (5V 3A USB-C)
- Ethernet cable or reliable WiFi connection
- Optional: Case with cooling/fan

### Software Requirements
- Raspberry Pi OS (64-bit recommended)
- Docker & Docker Compose
- Internet connection
- SSH access enabled (for headless setup)

---

## Initial Setup

### 1. Prepare Raspberry Pi

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y git curl wget vim
```

### 2. Enable SSH (if not already enabled)

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

### 3. Set Static IP (Recommended)

Edit `/etc/dhcpcd.conf`:
```bash
sudo nano /etc/dhcpcd.conf
```

Add:
```
interface eth0
static ip_address=192.168.1.100/24
static routers=192.168.1.1
static domain_name_servers=8.8.8.8 8.8.4.4
```

---

## Docker Installation

### Install Docker on Raspberry Pi

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER

# Install Docker Compose
sudo apt install -y docker-compose

# Verify installation
docker --version
docker-compose --version

# Reboot to apply group changes
sudo reboot
```

After reboot, test Docker:
```bash
docker run hello-world
```

---

## OpenClaw Installation

### 1. Clone OpenClaw Repository

```bash
cd ~
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

### 2. Configure Environment

Create `.env` file:
```bash
cp .env.example .env
nano .env
```

Essential environment variables:
```env
# API Keys
ANTHROPIC_API_KEY=your_anthropic_api_key_here
OPENAI_API_KEY=your_openai_api_key_here  # Optional

# Gateway Configuration
GATEWAY_HOST=127.0.0.1
GATEWAY_PORT=18789
GATEWAY_PROTOCOL=ws

# Database (if using persistence)
DATABASE_URL=sqlite:///data/openclaw.db

# Logging
LOG_LEVEL=INFO

# Security
SECRET_KEY=generate_a_secure_random_key_here
```

### 3. Build Docker Containers

```bash
# Build the images
docker-compose build

# Start services
docker-compose up -d

# Check logs
docker-compose logs -f
```

---

## Gateway Configuration

### Understanding the Gateway

The OpenClaw Gateway runs on **ws://127.0.0.1:18789** and acts as the central hub for:
- Message routing between platforms
- Skill execution
- State management
- Request/response handling

### Gateway Configuration File

Create `config/gateway.yml`:
```yaml
gateway:
  host: 127.0.0.1
  port: 18789
  protocol: ws

  # Connection settings
  max_connections: 100
  timeout: 30

  # Security
  enable_auth: true
  auth_token: ${GATEWAY_AUTH_TOKEN}

# Model Configuration
models:
  default: claude-sonnet-4-5
  providers:
    - name: anthropic
      api_key: ${ANTHROPIC_API_KEY}
      models:
        - claude-sonnet-4-5
        - claude-opus-4-5

# Skill Settings
skills:
  enabled: true
  auto_load: true
  directory: ./skills
```

### Test Gateway Connection

```bash
# Using wscat (install if needed)
npm install -g wscat

# Test connection
wscat -c ws://127.0.0.1:18789

# Should see connection established
```

---

## Messaging Platform Setup

### WhatsApp Integration

#### Option 1: Using Twilio

1. Create Twilio account: https://www.twilio.com/
2. Get WhatsApp sandbox number
3. Configure webhook:

```env
# Add to .env
WHATSAPP_PROVIDER=twilio
TWILIO_ACCOUNT_SID=your_account_sid
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_WHATSAPP_NUMBER=whatsapp:+14155238886
```

#### Option 2: Using WhatsApp Business API

1. Apply for WhatsApp Business API access
2. Set up webhook URL pointing to your gateway
3. Configure in `config/platforms/whatsapp.yml`

### Telegram Integration

1. Create bot via [@BotFather](https://t.me/botfather)
2. Get bot token
3. Configure:

```env
# Add to .env
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_ENABLED=true
```

Update `config/platforms/telegram.yml`:
```yaml
telegram:
  enabled: true
  bot_token: ${TELEGRAM_BOT_TOKEN}
  polling_interval: 1
  allowed_users: []  # Empty = all users, or specify user IDs
```

### Discord Integration

1. Create Discord application: https://discord.com/developers
2. Create bot and get token
3. Configure:

```env
DISCORD_BOT_TOKEN=your_discord_bot_token
DISCORD_ENABLED=true
```

### Slack Integration

1. Create Slack app: https://api.slack.com/apps
2. Enable Socket Mode
3. Get app token and bot token
4. Configure:

```env
SLACK_APP_TOKEN=your_slack_app_token
SLACK_BOT_TOKEN=your_slack_bot_token
SLACK_ENABLED=true
```

---

## Custom Skills

### Creating a Custom Skill

Skills extend OpenClaw's capabilities. Create in `skills/` directory:

```python
# skills/weather_skill.py
from openclaw import Skill, skill_handler

class WeatherSkill(Skill):
    name = "weather"
    description = "Get weather information"

    @skill_handler(pattern=r"weather in (\w+)")
    async def get_weather(self, location):
        # Your weather API integration
        return f"Weather in {location}: Sunny, 72°F"
```

### Enable Custom Skills

```bash
# Restart services to load new skills
docker-compose restart
```

---

## Remote Access

### Option 1: Cloudflare Tunnel (Recommended)

```bash
# Install cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
sudo dpkg -i cloudflared-linux-arm64.deb

# Authenticate
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create openclaw-pi

# Configure tunnel
nano ~/.cloudflared/config.yml
```

Add:
```yaml
tunnel: your-tunnel-id
credentials-file: /home/pi/.cloudflared/your-tunnel-id.json

ingress:
  - hostname: openclaw.yourdomain.com
    service: ws://localhost:18789
  - service: http_status:404
```

Start tunnel:
```bash
cloudflared tunnel run openclaw-pi
```

### Option 2: Tailscale (VPN)

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate
sudo tailscale up

# Access via Tailscale IP
```

### Option 3: Port Forwarding (Less Secure)

Configure router to forward port 18789 to Raspberry Pi IP.

**Note**: Use HTTPS/WSS in production with proper certificates.

---

## Troubleshooting

### Gateway Won't Start

```bash
# Check logs
docker-compose logs gateway

# Check port availability
sudo netstat -tulpn | grep 18789

# Restart services
docker-compose down
docker-compose up -d
```

### High Memory Usage

```bash
# Check resource usage
docker stats

# Limit memory in docker-compose.yml
services:
  gateway:
    mem_limit: 2g
    mem_reservation: 1g
```

### Connection Timeouts

- Verify firewall rules
- Check network connectivity
- Ensure proper .env configuration
- Verify API keys are valid

### Platform Integration Issues

```bash
# Test webhooks
curl -X POST http://localhost:18789/webhook/test \
  -H "Content-Type: application/json" \
  -d '{"message": "test"}'

# Check platform-specific logs
docker-compose logs platform-whatsapp
docker-compose logs platform-telegram
```

---

## LangChain Integration

### Should You Use LangChain?

**When LangChain Makes Sense:**
- ✅ Building complex multi-step reasoning chains
- ✅ Need document retrieval (RAG) capabilities
- ✅ Integrating multiple external tools/APIs
- ✅ Want advanced prompt management
- ✅ Building custom agents with memory

**When to Skip LangChain:**
- ❌ OpenClaw's built-in skills are sufficient
- ❌ Want to minimize dependencies
- ❌ Simple request/response patterns
- ❌ Limited RAM (Raspberry Pi constraint)
- ❌ Need faster response times

### Integrating LangChain (Optional)

If you decide to use LangChain:

```bash
# Install in your OpenClaw environment
pip install langchain langchain-anthropic
```

Create a LangChain skill:

```python
# skills/langchain_skill.py
from openclaw import Skill, skill_handler
from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate
from langchain_anthropic import ChatAnthropic

class LangChainSkill(Skill):
    name = "langchain"

    def __init__(self):
        self.llm = ChatAnthropic(model="claude-sonnet-4-5")

    @skill_handler(pattern=r"analyze: (.+)")
    async def analyze_text(self, text):
        prompt = PromptTemplate(
            template="Analyze the following: {text}",
            input_variables=["text"]
        )
        chain = LLMChain(llm=self.llm, prompt=prompt)
        result = await chain.arun(text=text)
        return result
```

### Recommended Approach

**For Raspberry Pi deployments:**
1. Start with OpenClaw's native skills
2. Use LangChain only if you need:
   - RAG (Retrieval Augmented Generation)
   - Complex agent workflows
   - Integration with specific LangChain tools
3. Monitor memory usage carefully
4. Consider offloading heavy LangChain operations to a more powerful server

**Architecture:**
```
[Raspberry Pi] → OpenClaw Gateway (lightweight)
                 ↓
[Cloud Server] → LangChain Heavy Processing
                 ↓
         Return Results
```

---

## Monitoring & Maintenance

### Health Checks

```bash
# Check service status
docker-compose ps

# View logs
docker-compose logs -f --tail=100

# Restart specific service
docker-compose restart gateway
```

### Automatic Updates

Create update script `update.sh`:
```bash
#!/bin/bash
cd ~/openclaw
git pull
docker-compose down
docker-compose build
docker-compose up -d
```

### Backup Configuration

```bash
# Backup important files
tar -czf openclaw-backup-$(date +%Y%m%d).tar.gz \
  .env config/ skills/ data/
```

---

## Production Checklist

- [ ] Set strong SECRET_KEY in .env
- [ ] Enable authentication on gateway
- [ ] Configure rate limiting
- [ ] Set up SSL/TLS (for remote access)
- [ ] Configure backup automation
- [ ] Set up monitoring/alerts
- [ ] Document custom skills
- [ ] Test all messaging platforms
- [ ] Configure log rotation
- [ ] Enable firewall rules

---

## Resources

- **OpenClaw GitHub**: https://github.com/openclaw/openclaw
- **Documentation**: https://docs.openclaw.ai/
- **Official Site**: https://openclaw.ai/
- **Community**: Discord/Forum links
- **Raspberry Pi Docker**: https://docs.docker.com/engine/install/debian/

---

## Need Help?

- Check logs: `docker-compose logs -f`
- OpenClaw Issues: https://github.com/openclaw/openclaw/issues
- Community support channels
- This deployment: Contact your administrator

---

**Last Updated**: February 2026
**Version**: 1.0
**Platform**: Raspberry Pi + Docker
