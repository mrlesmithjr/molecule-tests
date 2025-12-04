# CLAUDE.md

> **Note**: For global standards (git workflow, code standards), see `~/.claude/CLAUDE.md`.

This file provides guidance to Claude Code when working with code in this repository.

## Overview

This repository contains a collection of Molecule test scenarios for testing Ansible roles across different Linux distributions and platforms. It serves as a comprehensive testing framework supporting multiple distros including Alpine, CentOS, Debian, Fedora, Rocky, and Ubuntu variants.

## Architecture

### Molecule Scenario Structure

The repository uses a **shared playbook pattern** to minimize duplication:

- **Shared playbooks** (`molecule/shared/`):
  - `converge.yml` - Main test playbook (currently empty, expects users to customize)
  - `prepare.yml` - Pre-test setup (updates apt cache on Debian-based systems)
  - `verify.yml` - Post-test verification (contains example assertions)

- **Individual scenarios** (`molecule/<distro-name>/`):
  - Each scenario has its own `molecule.yml` configuration
  - All scenarios reference the shared playbooks using relative paths: `../shared/converge.yml`
  - Two driver types supported:
    - **Docker driver**: Most scenarios (Alpine, CentOS, Debian, Fedora, Rocky, Ubuntu)
    - **Vagrant driver**: Special `vagrant` scenario for VM-based testing

### Docker Scenarios Configuration

Docker-based scenarios follow this pattern:
- Use custom images from `mrlesmithjr/<distro>:<version>`
- Run in privileged mode with systemd as init system
- Mount cgroup filesystem for systemd support
- Use `cgroupns_mode: host` for cgroup namespace handling

### Vagrant Scenario Configuration

The Vagrant scenario uses:
- Bento boxes (e.g., `bento/ubuntu-20.04-arm64`)
- Configurable memory (2048 MB) and CPUs (2)
- Custom Ansible config for remote_tmp path

## Development Commands

### Running Tests

Test a specific scenario:
```bash
molecule test --scenario-name <scenario-name>
```

Available scenarios: `alpine313`, `alpine314`, `centos7`, `centos8`, `debian8`, `debian9`, `debian10`, `debian11`, `fedora34`, `fedora35`, `rocky8`, `ubuntu1604`, `ubuntu1804`, `ubuntu2004`, `ubuntu2204`, `vagrant`

Run the full test cycle for a scenario:
```bash
molecule create --scenario-name <scenario-name>
molecule converge --scenario-name <scenario-name>
molecule verify --scenario-name <scenario-name>
molecule destroy --scenario-name <scenario-name>
```

### Linting

Run all linters (yamllint, ansible-lint, flake8):
```bash
yamllint .
ansible-lint
flake8
```

Or use pre-commit to run all checks:
```bash
pre-commit run --all-files
```

### Environment Setup

Install dependencies using Poetry:
```bash
poetry install
```

Or using pip:
```bash
pip install -r requirements.txt -r requirements-dev.txt
```

Install Ansible Galaxy collections:
```bash
ansible-galaxy collection install -r requirements.yml
```

Install pre-commit hooks:
```bash
pre-commit install
```

## Key Dependencies

- **Molecule**: 5.0.1+ with Docker extras
- **Molecule Plugins**: 23.4.1+ with Vagrant extras
- **Ansible**: 8.0.0+
- **Python**: 3.9+
- **Collections**: `community.docker` (required for Docker driver)

## CI/CD

GitHub Actions workflow (`.github/workflows/default.yml`) runs:
1. Pre-commit checks (all linters)
2. Molecule tests across all Docker scenarios in parallel (max 4 concurrent)
3. Triggered on PRs to main/master/develop and pushes to non-main branches

## Adding New Scenarios

1. Create new directory: `molecule/<distro-name>/`
2. Create `molecule.yml` configuration
3. Reference shared playbooks: `playbooks: { converge: ../shared/converge.yml, prepare: ../shared/prepare.yml, verify: ../shared/verify.yml }`
4. Add scenario to CI matrix in `.github/workflows/default.yml`

## Customizing Tests

To add actual test logic:
- Modify `molecule/shared/converge.yml` to add tasks that should run on all scenarios
- Create scenario-specific converge.yml in individual scenario directories if needed
- Update `molecule/shared/verify.yml` with assertions to validate your test requirements
