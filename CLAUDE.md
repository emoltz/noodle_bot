# Noodle Bot - OpenClaw Project

## Overview
This is an OpenClaw (formerly Clawdbot/Moltbot) AI assistant deployment running on Raspberry Pi using Docker. OpenClaw is a self-hosted personal AI assistant that can run on any platform.

## Platform
- **Hardware**: Raspberry Pi
- **Containerization**: Docker
- **Architecture**: OpenClaw Gateway

## What is OpenClaw?
OpenClaw is an open-source AI assistant framework that allows you to run your own personal AI assistant locally. It supports multiple messaging platforms (WhatsApp, Telegram, Slack, Discord) and can be extended with custom skills.

## System Requirements
- Raspberry Pi (4GB RAM recommended)
- Docker installed
- OpenClaw Gateway runs on ws://127.0.0.1:18789

## Key Features
- Self-hosted AI assistant
- Multi-platform messaging support
- Docker-based sandboxing for security
- Extensible skills system
- Low power consumption (ideal for always-on deployment)

## Deployment Notes
- Raspberry Pi is well-suited for OpenClaw gateway deployment
- Docker provides isolation and easy management
- Can be exposed via Cloudflare tunnels for remote access
- Lightweight - runs well on 4GB RAM

## Resources
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw Documentation](https://docs.openclaw.ai/)
- [OpenClaw Official Site](https://openclaw.ai/)

## Getting Started
1. Ensure Docker is installed on your Raspberry Pi
2. Follow OpenClaw installation guides for Docker deployment
3. Configure the gateway and desired messaging platforms
4. Add custom skills as needed

## Notes
- This bot is configured for local/home deployment
- Docker sandboxing provides security boundaries
- Cost-effective solution for always-on AI assistant
