# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the matrix-docker-ansible-deploy project - an Ansible playbook for deploying a Matrix homeserver and related services using Docker containers. The playbook automates setup and maintenance of Matrix infrastructure including homeservers (Synapse/Dendrite/Conduit variants), bridges, bots, clients, and supporting services.

## Common Development Commands

### Installation and Updates

```bash
# Update playbook and install/update Ansible roles
just update

# Install roles only (without updating playbook)
just roles

# Run a full installation (install + create users + start)
just install-all

# Setup services without starting them
just setup-all
```

### Running the Playbook

```bash
# Run playbook with specific tags
just run-tags <comma-separated-tags>

# Run playbook with custom arguments
just run <extra-args>

# Install a specific service
just install-service <service-name>
```

### Service Management

```bash
just start-all
just stop-all
just start-group <group-name>
just stop-group <group-name>
```

### User Management

```bash
# Register a new user (admin_yes_or_no: 'yes' or 'no')
just register-user <username> <password> <admin_yes_or_no>
```

### Development

```bash
# Run ansible-lint against custom roles
just lint
# or
make lint
```

## Key Playbook Tags

- `setup-all` — Runs all setup tasks (install + uninstall) but doesn't start services
- `install-all` — Like `setup-all` but skips uninstallation tasks
- `setup-SERVICE` / `install-SERVICE` — Per-service setup (e.g., `setup-synapse`, `install-mautrix-telegram`)
- `start` / `stop` — Start/stop all services
- `ensure-matrix-users-created` — Creates special users needed by bots/bridges

**Note:** `just install-all` differs from raw `--tags=install-all` because just recipes also run `ensure-matrix-users-created` and `start` automatically.

## High-Level Architecture

### Project Structure

- `setup.yml` — Main playbook orchestrating all role execution
- `roles/custom/` — Matrix-specific roles maintained by this project (~80 roles for services, bridges, bots)
- `roles/galaxy/` — External roles fetched via ansible-galaxy (gitignored)
- `group_vars/matrix_servers` — Central wiring file connecting all roles with variable overrides
- `inventory/hosts` — Target server definitions
- `inventory/host_vars/<domain>/vars.yml` — Per-server configuration

### Role Structure

Each role in `roles/custom/` follows this pattern:
- `defaults/main.yml` — Default variables (service disabled by default)
- `tasks/main.yml` — Tagged task blocks for setup/install/start/stop operations
- `templates/` — Jinja2 templates for configs and systemd units
- `vars/` — Internal variables

### Variable Naming Convention

- `matrix_<service>_enabled: true/false` — Enable/disable a service
- `matrix_<service>_*` — Service-specific configuration
- `matrix_domain` — Base domain for Matrix identity (e.g., `example.com`)
- `matrix_server_fqn_matrix` — FQDN where services run (defaults to `matrix.{{ matrix_domain }}`)
- `matrix_homeserver_implementation` — Homeserver choice: `synapse`, `dendrite`, `conduit`, `conduwuit`, `continuwuity`

### Configuration Flow

1. User defines `inventory/host_vars/<domain>/vars.yml` with their settings
2. `group_vars/matrix_servers` wires together all roles and sets interdependencies
3. Each role's defaults are overridden by group_vars and then host_vars
4. Services are enabled explicitly via `matrix_<service>_enabled: true`

### Deployment Architecture

- All services run in Docker containers with systemd unit files
- Traefik handles reverse proxying and SSL termination (Let's Encrypt)
- Services communicate via Docker networks
- Data persists in Docker volumes under `/matrix/`
- Bridge/bot registrations are automatically mounted into homeserver containers

### Key Dependencies

External roles from `requirements.yml` provide:
- Docker installation (geerlingguy/ansible-role-docker)
- PostgreSQL (mother-of-all-self-hosting/ansible-role-postgres)
- Traefik reverse proxy
- systemd service management
- Various supporting services (Grafana, Prometheus, Valkey, etc.)