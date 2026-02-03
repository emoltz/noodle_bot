# OpenClaw Setup Guide - Raspberry Pi Deployment

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Docker Installation](#docker-installation)
4. [OpenClaw Installation](#openclaw-installation)
5. [Gateway Configuration](#gateway-configuration)
6. [Messaging Platform Setup](#messaging-platform-setup)
7. [Custom Skills](#custom-skills)
8. [Multi-Agent Routing](#multi-agent-routing)
9. [Remote Access](#remote-access)
10. [Troubleshooting](#troubleshooting)
11. [LangChain Integration](#langchain-integration)

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

## Multi-Agent Routing

### What is Multi-Agent Routing?

OpenClaw's multi-agent routing enables **multiple isolated AI agents** to run on a single Gateway instance (ws://127.0.0.1:18789). Each agent is a fully independent "brain" with its own workspace, configuration, and state.

### Why Use Multi-Agent Routing?

**On a Raspberry Pi, this feature is invaluable for:**
- **Resource optimization** - Route simple tasks to lightweight Sonnet, complex ones to Opus
- **Context separation** - Keep work conversations separate from personal ones
- **Multi-user support** - Share one Gateway server across family/team while maintaining privacy
- **Platform-specific agents** - Different personalities for WhatsApp vs. Telegram vs. Discord
- **Cost control** - Use cheaper models for routine interactions, powerful models for important tasks

### Agent Isolation Architecture

Each agent maintains complete isolation:

```
~/.openclaw/
├── agents/
│   ├── fast-agent/
│   │   ├── workspace/
│   │   │   ├── AGENTS.md          # Agent identity
│   │   │   ├── SOUL.md            # Personality/behavior
│   │   │   ├── USER.md            # User preferences
│   │   │   └── notes/             # Local notes
│   │   ├── state/
│   │   │   ├── auth-profiles.json # Auth credentials
│   │   │   └── config.json        # Agent config
│   │   └── sessions/              # Chat history
│   ├── smart-agent/
│   │   └── [same structure]
│   └── personal-agent/
│       └── [same structure]
```

### Configuration

#### Basic Multi-Agent Setup

Create `config/agents.yml`:

```yaml
# Define multiple agents
agents:
  # Fast everyday agent (Sonnet)
  - id: fast-agent
    name: "QuickBot"
    model: claude-sonnet-4-5
    workspace: /home/openclaw/agents/fast-agent/workspace
    state_dir: /home/openclaw/agents/fast-agent/state
    description: "Fast agent for quick tasks and casual chat"

  # Powerful reasoning agent (Opus)
  - id: smart-agent
    name: "ThinkBot"
    model: claude-opus-4-5
    workspace: /home/openclaw/agents/smart-agent/workspace
    state_dir: /home/openclaw/agents/smart-agent/state
    description: "Advanced agent for complex reasoning and analysis"

  # Personal assistant
  - id: personal-agent
    name: "MyAssistant"
    model: claude-sonnet-4-5
    workspace: /home/openclaw/agents/personal-agent/workspace
    state_dir: /home/openclaw/agents/personal-agent/state
    description: "Personal tasks and project management"
```

#### Routing Rules

Create `config/routing.yml`:

```yaml
# Route inbound messages to agents
routing:
  # Route by channel type
  channel_bindings:
    whatsapp:
      default_agent: fast-agent

    telegram:
      default_agent: smart-agent

    discord:
      default_agent: personal-agent

    slack:
      default_agent: fast-agent

  # Route by account ID (for multiple accounts on same platform)
  account_bindings:
    # Personal WhatsApp
    "whatsapp:+1234567890": fast-agent

    # Work WhatsApp
    "whatsapp:+0987654321": smart-agent

    # Family Telegram
    "telegram:@family_bot": personal-agent

  # Fine-grained peer routing (specific conversations)
  peer_bindings:
    # Boss always gets the smart agent
    "whatsapp:+1111111111": smart-agent

    # Spouse gets personal agent
    "telegram:@spouse": personal-agent

  # Fallback agent if no rules match
  default_agent: fast-agent
```

### Practical Use Cases

#### Use Case 1: Resource-Optimized Routing

**Goal**: Save money and RAM by routing appropriately

```yaml
routing:
  # Simple chat → Sonnet (fast, cheap)
  channel_bindings:
    whatsapp: fast-agent
    sms: fast-agent

  # Complex work → Opus (powerful, expensive)
  channel_bindings:
    slack: smart-agent
    email: smart-agent

  # Pattern-based routing
  rules:
    - pattern: "analyze|research|detailed|complex"
      agent: smart-agent
    - pattern: "quick|simple|remind|what's"
      agent: fast-agent
```

#### Use Case 2: Multi-User Family Setup

**Goal**: Multiple family members share one Raspberry Pi

```yaml
agents:
  - id: dad-agent
    model: claude-sonnet-4-5
    workspace: /data/agents/dad

  - id: mom-agent
    model: claude-sonnet-4-5
    workspace: /data/agents/mom

  - id: kids-agent
    model: claude-sonnet-4-5
    workspace: /data/agents/kids

routing:
  account_bindings:
    "whatsapp:+1234567890": dad-agent
    "telegram:@mom": mom-agent
    "discord:kids#1234": kids-agent
```

Each family member gets:
- Private workspace and chat history
- Custom persona (configured in SOUL.md)
- Separate auth profiles
- Complete isolation from others

#### Use Case 3: Context-Specific Agents

**Goal**: Different agent personalities for different contexts

```yaml
agents:
  - id: work-agent
    model: claude-opus-4-5
    workspace: /data/agents/work
    # SOUL.md: "Professional, formal, focused on productivity"

  - id: creative-agent
    model: claude-sonnet-4-5
    workspace: /data/agents/creative
    # SOUL.md: "Imaginative, playful, helps with creative projects"

  - id: learning-agent
    model: claude-opus-4-5
    workspace: /data/agents/learning
    # SOUL.md: "Patient teacher, explains concepts thoroughly"

routing:
  channel_bindings:
    slack: work-agent
    discord: creative-agent
    telegram: learning-agent
```

### Agent Personality Configuration

Each agent can have a unique personality via `SOUL.md`:

```bash
# Edit agent personality
nano ~/.openclaw/agents/fast-agent/workspace/SOUL.md
```

Example `SOUL.md`:
```markdown
# Agent Soul: QuickBot

## Personality
- Concise and to the point
- Friendly but brief
- Optimized for quick responses
- Use emojis occasionally

## Behavior Rules
- Keep responses under 100 words unless asked for detail
- Always respond within 5 seconds
- If a complex question comes in, suggest switching to ThinkBot
- Remember user's preferences in USER.md

## Limitations
- Don't attempt deep analysis
- Don't write long-form content
- Defer complex reasoning to smart-agent
```

### Agent User Preferences

Store user-specific info in `USER.md`:

```bash
nano ~/.openclaw/agents/fast-agent/workspace/USER.md
```

Example `USER.md`:
```markdown
# User Profile

## Name
John Doe

## Preferences
- Timezone: PST (UTC-8)
- Communication style: Direct and casual
- Preferred model: Sonnet (for speed)
- Language: English

## Context
- Software engineer
- Works on React projects
- Interested in AI and robotics
- Raspberry Pi enthusiast

## Notes
- Don't explain basic programming concepts
- Always include code examples
- Prefers TypeScript over JavaScript
```

### Dynamic Agent Switching

Users can manually switch agents mid-conversation:

```
User: @smart-agent analyze this complex algorithm
[Message routes to smart-agent instead of default]

User: @fast-agent what time is it?
[Routes back to fast-agent]
```

Configure in `config/routing.yml`:
```yaml
routing:
  enable_mentions: true
  mention_prefix: "@"

  # Agent aliases
  aliases:
    smart: smart-agent
    fast: fast-agent
    think: smart-agent
    quick: fast-agent
```

### Monitoring Multiple Agents

#### Check Agent Status

```bash
# View all agent sessions
ls -la ~/.openclaw/agents/*/sessions/

# Check active agents
docker-compose exec gateway openclaw agents list

# Monitor agent-specific logs
docker-compose logs -f | grep "agent=fast-agent"
```

#### Agent Resource Usage

```bash
# Create monitoring script
cat > monitor-agents.sh << 'EOF'
#!/bin/bash
echo "Agent Resource Monitor"
echo "====================="

for agent in ~/.openclaw/agents/*/; do
    agent_name=$(basename "$agent")
    session_count=$(ls -1 "$agent/sessions/" | wc -l)
    workspace_size=$(du -sh "$agent/workspace" | cut -f1)

    echo "$agent_name:"
    echo "  Active Sessions: $session_count"
    echo "  Workspace Size: $workspace_size"
    echo ""
done
EOF

chmod +x monitor-agents.sh
./monitor-agents.sh
```

### Memory Management for Multi-Agent on Raspberry Pi

With limited RAM (4GB), manage resources carefully:

```yaml
# config/agents.yml
agents:
  - id: fast-agent
    model: claude-sonnet-4-5
    max_concurrent_sessions: 5
    session_timeout: 300  # 5 minutes

  - id: smart-agent
    model: claude-opus-4-5
    max_concurrent_sessions: 2  # Limit expensive agent
    session_timeout: 600  # 10 minutes
```

### Troubleshooting Multi-Agent Setup

#### Agent Not Routing Correctly

```bash
# Check routing configuration
cat config/routing.yml

# Test routing
curl -X POST http://localhost:18789/test-route \
  -H "Content-Type: application/json" \
  -d '{
    "channel": "whatsapp",
    "from": "+1234567890",
    "message": "test"
  }'

# View routing logs
docker-compose logs gateway | grep "routing"
```

#### Agent State Issues

```bash
# Reset agent state (will lose chat history!)
rm -rf ~/.openclaw/agents/fast-agent/sessions/*

# Backup before reset
tar -czf agent-backup.tar.gz ~/.openclaw/agents/
```

#### Session Conflicts

```bash
# Clear stale sessions
find ~/.openclaw/agents/*/sessions/ -mtime +7 -delete

# Restart gateway
docker-compose restart gateway
```

### Best Practices

1. **Start Simple** - Begin with 2 agents (one fast, one smart)
2. **Monitor Resources** - Watch RAM usage with `docker stats`
3. **Clear Old Sessions** - Implement periodic cleanup
4. **Unique Workspaces** - Never share workspace directories
5. **Document Routing** - Keep `routing.yml` well-commented
6. **Test Thoroughly** - Verify routing rules with test messages
7. **Backup Agent Data** - Regular backups of workspace/state directories

### Example: Complete 3-Agent Setup

```yaml
# docker-compose.yml additions
services:
  gateway:
    volumes:
      - ./config/agents.yml:/app/config/agents.yml
      - ./config/routing.yml:/app/config/routing.yml
      - agent-data:/home/openclaw/agents

volumes:
  agent-data:
```

```yaml
# config/agents.yml
agents:
  - id: everyday
    name: "EverydayBot"
    model: claude-sonnet-4-5
    workspace: /home/openclaw/agents/everyday/workspace

  - id: expert
    name: "ExpertBot"
    model: claude-opus-4-5
    workspace: /home/openclaw/agents/expert/workspace

  - id: creative
    name: "CreativeBot"
    model: claude-sonnet-4-5
    workspace: /home/openclaw/agents/creative/workspace
```

```yaml
# config/routing.yml
routing:
  channel_bindings:
    whatsapp: everyday
    telegram: expert
    discord: creative

  rules:
    - keywords: ["analyze", "research", "explain in detail"]
      agent: expert
    - keywords: ["write", "story", "creative", "imagine"]
      agent: creative

  default_agent: everyday
```

### Advanced: Conditional Routing

Route based on message content:

```yaml
routing:
  # AI analyzes incoming message and routes appropriately
  intelligent_routing:
    enabled: true

    # Classify message complexity
    complexity_threshold: 0.7  # 0-1 scale

    # Route high complexity → expert agent
    high_complexity_agent: expert

    # Route low complexity → everyday agent
    low_complexity_agent: everyday
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
