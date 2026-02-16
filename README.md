# DNS Infrastructure Management with Ansible

This repository contains the configuration and automation for managing DNS records for the `anilankemlab.lan` domain using Ansible.

## Overview

The setup uses Ansible to automate the deployment of forward and reverse DNS zone files to a BIND9 server. DNS records are defined in a simple YAML file, and an Ansible playbook generates the corresponding zone files from templates. A GitHub Actions workflow automates the validation and deployment of these records.

## Prerequisites

- An Ansible control node with access to the DNS server.
- A BIND9 DNS server.
- The DNS server must be listed in the `ansible/inventory.ini` file.
- Python and `ansible-playbook` installed on the machine running the GitHub Actions runner (if self-hosted).

## Repository Structure

```
.
├── ansible/
│   ├── deploy-dns.yml      # The main Ansible playbook
│   ├── inventory.ini       # Ansible inventory defining the DNS server
│   └── templates/
│       ├── forward.j2      # Jinja2 template for the forward zone
│       └── reverse.j2      # Jinja2 template for the reverse zone
├── dns_records.yml         # YAML file to define DNS records
├── .github/workflows/
│   └── update.yml          # GitHub Actions workflow for CI/CD
└── README.md
```

## How It Works

### DNS Records

DNS records are defined in `dns_records.yml`. This file contains the domain name, network information, and a list of hosts with their corresponding IP addresses.

**Example `dns_records.yml`:**
```yaml
domain: anilankemlab.lan
network: 192.168.1
dns_server_ip: 192.168.1.10

records:
  - name: proxmox
    ip: 192.168.1.100
  - name: master
    ip: 192.168.1.20
```

### Ansible Playbook

The `ansible/deploy-dns.yml` playbook performs the following steps:
1.  **Loads Variables**: Reads the data from `dns_records.yml`.
2.  **Backs Up Zones**: Creates a backup of the existing forward and reverse zone files.
3.  **Generates Serial**: Creates a new serial number for the zone files based on the current date and time (`YYYYMMDDHH`).
4.  **Deploys Zones**: Uses the `forward.j2` and `reverse.j2` templates to generate the new zone files.
5.  **Validates Configuration**:
    -   Runs `named-checkzone` to validate the syntax of the new forward and reverse zone files.
    -   Runs `named-checkconf` to validate the overall BIND configuration.
6.  **Reloads BIND**: If the validation is successful, it reloads the `bind9` service to apply the changes.

## Usage

To add or update a DNS record:
1.  Edit the `dns_records.yml` file to add, modify, or remove an entry in the `records` list.
2.  Commit the change to the `main` branch.
3.  Push the changes to GitHub.

This will automatically trigger the GitHub Actions workflow to validate and deploy the changes.

To run the deployment manually:
```bash
cd ansible
ansible-playbook -i inventory.ini deploy-dns.yml
```

## CI/CD Pipeline

A GitHub Actions workflow is defined in `.github/workflows/update.yml` to automate the process of validating and deploying DNS changes.

The workflow has two main jobs:

1.  **`validate`**:
    -   Triggered on any `pull_request` targeting the `main` branch or on a `push` to `main` if `dns_records.yml` is changed.
    -   Performs a dry-run of the Ansible playbook (`--check` mode) to validate the changes without applying them.

2.  **`deploy`**:
    -   Triggered only on a `push` to the `main` branch, after the `validate` job succeeds.
    -   Runs the Ansible playbook to deploy the changes to the production DNS server.

This ensures that any changes are automatically validated before they are merged and deployed, providing a safe and automated way to manage DNS records.
