# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a fork/customization of [matrix-docker-ansible-deploy](https://github.com/spantaleev/matrix-docker-ansible-deploy) — an Ansible playbook for deploying a Matrix homeserver and related services using Docker containers. It deploys to `matrix.healthchat.ch` and includes a custom `healthchat-workspace` role.

## Common Development Commands

```bash
# Lint custom roles
just lint                    # or: make lint

# Update playbook + install/update Ansible roles
just update

# Install roles only (without git pull)
just roles

# Run full installation (install + create users + start)
just install-all

# Setup services without starting them
just setup-all

# Run playbook with specific tags
just run-tags <comma-separated-tags>

# Run with custom arguments
just run <extra-args>

# Install a single service
just install-service <service-name>

# Service management
just start-all / just stop-all
just start-group <group-name> / just stop-group <group-name>

# Register a new user ('yes' or 'no' for admin)
just register-user <username> <password> <admin_yes_or_no>
```

**Tag semantics:** `setup-all` runs install + uninstall tasks (ensures clean state); `install-all` skips uninstall tasks. `just install-all` additionally runs `ensure-matrix-users-created` and `start`.

## High-Level Architecture

### Project Structure

- `setup.yml` — Main playbook, lists all 80+ roles in execution order
- `roles/custom/` — 80 Matrix-specific roles (services, bridges, bots, clients)
- `roles/galaxy/` — External roles fetched via `ansible-galaxy` (gitignored)
- `group_vars/matrix_servers` — Central wiring file (~6000 lines) connecting all roles
- `inventory/` — **Git submodule** (`git@github.com:Novaloop-AG/matrix-docker-ansible-inventory.git`) containing host definitions and per-server config
- `inventory/host_vars/matrix.healthchat.ch/vars.yml` — Server-specific overrides

### Configuration Precedence

`role defaults/main.yml` → `group_vars/matrix_servers` → `inventory/host_vars/<domain>/vars.yml`

### Variable Naming Convention

- `matrix_<service>_enabled: true/false` — Enable/disable a service (disabled by default)
- `matrix_<service>_docker_image` / `matrix_<service>_version` — Docker image config
- `matrix_<service>_container_network` — Docker network assignment
- `matrix_<service>_container_labels_traefik_*` — Traefik routing configuration
- `matrix_domain` — Base domain for Matrix identity (e.g., `example.com`)
- `matrix_server_fqn_matrix` — FQDN where services run (default: `matrix.{{ matrix_domain }}`)
- `matrix_homeserver_implementation` — Homeserver choice: `synapse`, `dendrite`, `conduit`, `conduwuit`, `continuwuity`
- `matrix_base_data_path` — Root data directory (default: `/matrix`)

### Role Task Pattern

Every custom role follows this structure in `tasks/main.yml`:

```yaml
# Block 1: Setup/Install (tagged setup-all, setup-<service>, install-all, install-<service>)
- when: matrix_<service>_enabled | bool
  # validate_config.yml → setup_install.yml

# Block 2: Uninstall (same setup tags, but runs when DISABLED)
- when: not matrix_<service>_enabled | bool
  # setup_uninstall.yml — cleans up when service is turned off
```

Key pattern: uninstall tasks run under setup tags but with an inverted `when` condition, so disabling a service and running `setup-all` removes it cleanly.

### group_vars Wiring Pattern

`group_vars/matrix_servers` uses list/dict composition to wire services together:

- **Homeserver registration mounts**: Each enabled bridge adds its `registration.yaml` as a bind mount into the homeserver container
- **App service config files**: `matrix_homeserver_app_service_config_files_auto` is built by concatenating lists conditionally per bridge
- **Variables follow `auto + custom` pattern**: e.g., `_auto` computed by playbook, `_custom` for user overrides, combined with `| combine()` or `+`

### Deployment Architecture

- All services run in Docker containers managed by systemd unit files
- Docker networks: `matrix-homeserver`, `matrix-addons` (bridges/bots), `matrix-monitoring`, per-service networks
- Traefik handles reverse proxying and SSL termination (Let's Encrypt)
- Data persists under `/matrix/` with `matrix:matrix` ownership
- Bridge registrations are automatically bind-mounted into homeserver containers

### Custom Addition: healthchat-workspace

`roles/custom/healthchat-workspace/` deploys a custom Docker app from `ghcr.io` (Novaloop-AG/healthchat-workspace) with optional private registry auth, Traefik labels, and systemd management. Disabled by default.

### Pre-commit Hooks

Pre-push hooks are configured (`.pre-commit-config.yaml`): large file check, trailing whitespace, end-of-file fixer, codespell, REUSE compliance. A Nix flake (`.envrc` / `flake.nix`) provides the dev environment.

### Ansible-lint Configuration

Skipped rules (`.config/ansible-lint.yml`): `unnamed-task`, `no-handler`, `no-jinja-nesting`, `schema`, `command-instead-of-shell`, `role-name`, `var-naming[no-role-prefix]`, `template-instead-of-copy`. Runs in offline mode.