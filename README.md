# roboshop-ansible-collection

> Reusable Ansible roles for the RoboShop platform. Consumed by [`roboshop-infra-provisioning`](https://github.com/rahul-paladugu/Ansible-Roboshop) for full-stack environment deployment.

---

## Overview

This repository provides the **role library** for the RoboShop e-commerce platform. Each service in the stack has a dedicated, self-contained Ansible role that handles its complete installation and configuration lifecycle.

The design uses a **single dynamic entrypoint** (`main.yaml`) that accepts a `component` variable at runtime, eliminating the need for per-service playbook files. Roles are idempotent and can be applied individually for targeted deployments or re-runs.

This repository is the **roles layer** of a two-repo automation model:

| Repository | Purpose |
|---|---|
| `roboshop-ansible-collection` *(this repo)* | Roles library — reusable, service-level configuration |
| [`roboshop-infra-provisioning`](https://github.com/rahul-paladugu/Ansible-Roboshop) | Orchestration layer — EC2 provisioning, DNS, full-stack execution |

---

## Repository Structure

```
.
├── roles/
│   ├── mongodb/        # MongoDB installation & configuration
│   ├── catalogue/      # Catalogue service (Node.js)
│   ├── redis/          # Redis installation & configuration
│   ├── user/           # User service (Node.js)
│   ├── cart/           # Cart service (Node.js)
│   ├── mysql/          # MySQL installation, schema & seed data
│   ├── shipping/       # Shipping service (Java / Maven)
│   ├── rabbitmq/       # RabbitMQ installation & configuration
│   ├── payment/        # Payment service (Python)
│   ├── dispatch/       # Dispatch service (Go)
│   └── frontend/       # Nginx reverse proxy & frontend
├── 0-instances.yaml    # EC2 instance provisioning + Route 53 DNS
├── inventory.ini       # Static host inventory
└── main.yaml           # Single dynamic entrypoint playbook
```

Each role follows the standard Ansible Galaxy directory layout (`tasks/`, `templates/`, `handlers/`, `defaults/`, `vars/`, `files/`).

---

## How It Works

`main.yaml` is the single entrypoint for all service deployments. It uses the `component` variable to dynamically resolve both the target host group and the role to apply:

```yaml
- name: configuring {{ component }} server
  hosts: "{{ component }}"
  become: true
  roles:
    - "{{ component }}"
```

This means you never need to edit the playbook itself — you pass the target service at runtime using `-e component=<service-name>`. This pattern keeps the entrypoint DRY and makes per-service deployments and re-runs trivial.

---

## Prerequisites

- Ansible `>= 2.14`
- Python `>= 3.9` on the control node
- SSH access to target EC2 instances
- `inventory.ini` populated with correct hostnames or IP addresses (see [Inventory Setup](#inventory-setup))
- AWS credentials configured if running instance provisioning (`0-instances.yaml`)

---

## Inventory Setup

Update `inventory.ini` to reflect your environment's host addresses. Each service must be in its own host group matching the role name exactly:

```ini
[mongodb]
mongodb.rscloudservices.icu

[catalogue]
catalogue.rscloudservices.icu

[redis]
redis.rscloudservices.icu

[user]
user.rscloudservices.icu

[cart]
cart.rscloudservices.icu

[mysql]
mysql.rscloudservices.icu

[shipping]
shipping.rscloudservices.icu

[rabbitmq]
rabbitmq.rscloudservices.icu

[payment]
payment.rscloudservices.icu

[dispatch]
dispatch.rscloudservices.icu

[frontend]
frontend.rscloudservices.icu
```

> The host group name **must match** the role directory name exactly. `main.yaml` uses `component` for both `hosts:` and `roles:` resolution.

---

## Usage

### Deploy a Single Service

Pass the `component` variable with `-e` to target any service:

```bash
ansible-playbook -i inventory.ini main.yaml -e component=mongodb
ansible-playbook -i inventory.ini main.yaml -e component=catalogue
ansible-playbook -i inventory.ini main.yaml -e component=redis
```

### Deploy All Services (Full Stack)

Run each component in dependency order. Services must be deployed sequentially — upstream dependencies must be healthy before downstream services are configured.

```bash
# Recommended execution order
for component in mongodb catalogue redis user cart mysql shipping rabbitmq payment dispatch frontend; do
  echo "==> Deploying: $component"
  ansible-playbook -i inventory.ini main.yaml -e component=$component
done
```

### Provision Infrastructure First

If starting from scratch, run the EC2 provisioning playbook before deploying any services:

```bash
ansible-playbook 0-instances.yaml
```

Then update `inventory.ini` with the assigned IPs/DNS records before proceeding with service deployments.

---

## Service Dependency Order

The following execution order must be respected. Services listed lower depend on those listed above them being fully operational:

```
mongodb
  └── catalogue (reads product data from MongoDB)
        └── redis
              └── user (session cache via Redis)
                    └── cart (depends on catalogue + user)
                          └── mysql
                                └── shipping (reads city/state data from MySQL)
                                      └── rabbitmq
                                            └── payment (queues via RabbitMQ)
                                                  └── dispatch (consumes from RabbitMQ)
                                                        └── frontend (reverse proxy — last)
```

---

## Re-running Roles

All roles are written to be idempotent. Re-applying a role to an already-configured host is safe and will only make changes if configuration drift is detected:

```bash
# Safe to re-run — will only apply changes if needed
ansible-playbook -i inventory.ini main.yaml -e component=payment
```

---

## Extending the Collection

To add a new service role:

1. Scaffold the role structure:
   ```bash
   ansible-galaxy role init roles/<new-service>
   ```
2. Implement tasks in `roles/<new-service>/tasks/main.yaml`
3. Add the new service host group to `inventory.ini`
4. Deploy with:
   ```bash
   ansible-playbook -i inventory.ini main.yaml -e component=<new-service>
   ```

No changes to `main.yaml` are required.

---

## Related Repository

This roles library is consumed by **[`roboshop-infra-provisioning`](https://github.com/rahul-paladugu/Ansible-Roboshop)**, which handles EC2 instance lifecycle, Route 53 DNS registration, and full-stack orchestration. If you're looking to stand up a complete environment from scratch, start there.

---

## Maintainer

**Rahul Paladugu** — Infrastructure & Platform Engineering
