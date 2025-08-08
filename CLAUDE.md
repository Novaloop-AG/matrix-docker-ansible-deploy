# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the matrix-docker-ansible-deploy project - an Ansible playbook for deploying a Matrix homeserver and various Matrix-related services using Docker containers. The playbook automates the setup and maintenance of Matrix infrastructure.

## Common Development Commands

### Installation and Updates

```bash
# Update playbook and install/update Ansible roles
just update

# Install all roles (without updating playbook)
just roles

# Run a full installation
just install-all

# Setup services without starting them
just setup-all
```

### Service Management

```bash
# Start all services
just start-all

# Stop all services
just stop-all

# Install a specific service
just install-service <service-name>

# Start/stop a specific service group
just start-group <group-name>
just stop-group <group-name>
```

### User Management

```bash
# Register a new user (admin_yes_or_no should be 'yes' or 'no')
just register-user <username> <password> <admin_yes_or_no>
```

### Development and Testing

```bash
# Run ansible-lint against custom roles
just lint
# or
make lint

# Run playbook with specific tags
just run-tags <comma-separated-tags>

# Run playbook with custom arguments
just run <extra-args>
```

## High-Level Architecture

### Project Structure

- **Ansible Playbook Core**: The main `setup.yml` orchestrates the deployment
- **Roles Organization**:
  - `roles/custom/` - Matrix-specific roles maintained by this project
  - `roles/galaxy/` - External roles fetched via ansible-galaxy (gitignored)
- **Configuration**:
  - `inventory/hosts` - Defines target servers
  - `inventory/host_vars/<domain>/vars.yml` - Server-specific configuration
- **Documentation**: Comprehensive guides in `docs/` directory

### Service Categories

1. **Core Services**:
   - Matrix homeserver (Synapse/Conduit/Dendrite)
   - PostgreSQL database
   - Reverse proxy (Traefik/nginx)
   - SSL certificates (Let's Encrypt)

2. **Bridges**: Connect Matrix to external services (Discord, Telegram, WhatsApp, etc.)

3. **Bots**: Automation and moderation tools (Mjolnir, Draupnir, maubot)

4. **Clients**: Web-based Matrix clients (Element, Hydrogen, Cinny)

5. **Supporting Services**: Authentication, backups, monitoring, administration tools

### Deployment Flow

1. Ansible connects to target server(s) via SSH
2. Playbook ensures Docker and dependencies are installed
3. Each service runs in its own Docker container
4. Traefik handles reverse proxying and SSL termination
5. Services communicate via Docker networks
6. Data persists in Docker volumes

### Key Design Principles

- **Container-based**: All services run in Docker containers for isolation and portability
- **Modular**: Services can be enabled/disabled via configuration variables
- **Idempotent**: Playbook can be run multiple times safely
- **Configurable**: Extensive customization through Ansible variables
- **Maintainable**: Regular updates via container image updates

## Configuration Approach

- Each service has a corresponding role in `roles/custom/matrix-*`
- Services are configured via `matrix_*` variables in `vars.yml`
- Most services default to disabled and must be explicitly enabled
- Service interdependencies are handled automatically by the playbook