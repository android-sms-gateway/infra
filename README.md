# ☁️ SMSGate Infrastructure

[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stars][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![License][license-shield]][license-url]

Docker Swarm deployment stack for the SMSGate platform: reverse proxy, database, cache, application services, monitoring, and alerting. This README covers local cluster bring-up only; deep deployment and operations guides live at [docs.sms-gate.app](https://docs.sms-gate.app/).

## 📖 About

The platform delivers SMS messages through mobile devices. This repository packages the server-side infrastructure for single- or multi-node deployments:

- **Problem**: the gateway needs a production-grade runtime - TLS termination, persistent storage, scheduled backups, monitoring, and alerting.
- **Solution**: four self-contained Docker Swarm stacks, declarative service configuration, and secrets management through Swarm secrets.
- **Intended users**: operators and developers who deploy or maintain the gateway infrastructure.

Everything here is configuration-as-code: no application source code lives in this repository, only the stacks that run it.

## 📚 Table of Contents

- [☁️ SMSGate Infrastructure](#️-smsgate-infrastructure)
  - [📖 About](#-about)
  - [📚 Table of Contents](#-table-of-contents)
  - [⭐ Features](#-features)
  - [🏗️ Architecture](#️-architecture)
  - [📦 Prerequisites](#-prerequisites)
  - [🚀 Quickstart](#-quickstart)
  - [⚙️ Operations](#️-operations)
    - [Deploy the stacks](#deploy-the-stacks)
    - [Exposed hostnames](#exposed-hostnames)
    - [Update configuration](#update-configuration)
    - [Backups and metrics](#backups-and-metrics)
  - [📚 Documentation](#-documentation)
  - [🤝 Contributing](#-contributing)
  - [⚖️ License](#️-license)

## ⭐ Features

- **TLS by default**: Traefik v3 terminates HTTPS with Let's Encrypt certificates; the `cloudflare` cert resolver uses DNS-01 challenges for SMPPS, SMTPS, and public HTTPS routes.
- **HTTP, SMTP, and SMPP gateways**: traffic routing on ports 80/443, 587/465, and 2775/2776.
- **Mobile push**: the backend sends Firebase Cloud Messaging (FCM) notifications from a Google service-account credential.
- **Rate limiting and access control**: per-route rate limits for API and webhook endpoints; basic auth and IP allowlists for dashboards and metrics.
- **Scheduled database backups**: MariaDB dumps uploaded to S3 on a cron schedule via swarm-cronjob.
- **Certificate authority**: `ca-backend` issues certificates signed by a root CA stored in Swarm secrets.
- **Monitoring and alerting**: Prometheus scrapes labeled services; Alertmanager routes `critical` alerts to Telegram and `warning`/`info` to email; Grafana for visualization; cAdvisor runs globally on every node.
- **Multi-node placement**: services pin to labeled nodes (`redis`, `db`, `traefik`, `cronjob`, `prometheus`, `grafana`, `portainer`).

## 🏗️ Architecture

Four stacks compose the deployment. The `infra` stack must be deployed first because `app` and `monitoring` consume its `internal` and `public` overlay networks, and each references its own external secrets.

| File                     | Stack        | Contents                                                                                   |
| ------------------------ | ------------ | ------------------------------------------------------------------------------------------ |
| `compose.infra.yml`      | `infra`      | Traefik v3, MariaDB, Redis, phpMyAdmin, Redis Commander, S3 backup job, swarm-cronjob      |
| `compose.app.yml`        | `app`        | Backend API + worker, ca-backend, smpp-server, email-to-sms, webhook-tester, web-dashboard |
| `compose.monitoring.yml` | `monitoring` | Prometheus, Alertmanager, cAdvisor (global), Grafana                                       |
| `compose.portainer.yml`  | `portainer`  | Portainer CE (optional management UI) and its agent                                        |

A full architecture diagram lives in the central docs: https://docs.sms-gate.app/

## 📦 Prerequisites

- Docker Engine with Swarm mode enabled (one or more Linux nodes; the Portainer agent is global on Linux nodes only).
- A public domain with DNS records pointing at the cluster (see [Operations](#️-operations) for the hostnames to expose).
- A Cloudflare API token with DNS edit permission (used for ACME DNS-01 challenges).
- A terminal on a Swarm manager node.

## 🚀 Quickstart

1. Initialize or join the Swarm (run on the first manager):

   ```sh
   docker swarm init
   ```

2. Label nodes for service placement. Each service pins to a node label; labels that do not exist simply stay empty until their services are deployed:

   ```sh
   docker node update --label-add redis=true <manager-node>
   docker node update --label-add db=true <db-node>
   docker node update --label-add traefik=true <manager-node>
   docker node update --label-add cronjob=true <manager-node>
   docker node update --label-add prometheus=true <monitoring-node>
   docker node update --label-add grafana=true <monitoring-node>
   docker node update --label-add portainer=true <manager-node>
   ```

3. Create the external secrets referenced by the stacks (values are operator-provided):

   ```sh
   docker secret create mariadb_root_password -
   docker secret create users.htpasswd -
   docker secret create root-ca.crt -
   docker secret create root-ca.key -
   docker secret create telegram_bot_token -
   docker secret create email_password -
   docker secret create grafana_admin_password -
   ```

   All secrets are declared `external: true`, so they must exist before stack deployment.

4. Create the environment file from the template and fill in your values:

   ```sh
   cp .env.example .env
   ```

   `docker stack deploy` interpolates variables from the shell environment, not from `.env`. Load it into the shell before each deployment (for example, `set -a; source .env; set +a`). Set `ROOT_DOMAIN` before deploying: if it is unset, Docker substitutes empty values and public host rules become empty.

## ⚙️ Operations

### Deploy the stacks

Deploy in order: `infra` first (it creates the `internal` and `public` overlay networks), then `app` and `monitoring`, optionally followed by `portainer`.

```sh
docker stack deploy -c compose.infra.yml infra
docker stack deploy -c compose.app.yml app
docker stack deploy -c compose.monitoring.yml monitoring
docker stack deploy -c compose.portainer.yml portainer
```

- `infra` creates the `internal`/`public` overlay networks, the `redis-data`, `mariadb-data`, and `letsencrypt` volumes, and starts Traefik, MariaDB, Redis, Redis Commander, phpMyAdmin, and the cron scheduler.
- `app` deploys the backend API, worker, CA backend, SMPP server, email-to-SMS gateway, webhook tester, and web dashboard. The `worker` service starts with zero replicas; scale it if background tasks are needed.
- `monitoring` deploys Prometheus, Alertmanager, cAdvisor (global), and Grafana.
- `portainer` deploys Portainer CE (published on port 9443, constrained to manager nodes) and a global agent on all Linux nodes.

### Exposed hostnames

After deployment, every public service is reachable at `<subdomain>.<ROOT_DOMAIN>`:

| Stack        | Subdomain                | Service               |
| ------------ | ------------------------ | --------------------- |
| `infra`      | `admin.`                 | Traefik dashboard     |
| `infra`      | `pma.`                   | phpMyAdmin            |
| `infra`      | `redis.`                 | Redis Commander       |
| `app`        | `api.` (`API_SUBDOMAIN`) | Backend API           |
| `app`        | `ca.`                    | Certificate authority |
| `app`        | `smpp.`                  | SMPP gateway          |
| `app`        | `smtp.`                  | Email-to-SMS gateway  |
| `app`        | `webhook.`               | Webhook tester        |
| `app`        | `dashboard.`             | Web dashboard         |
| `monitoring` | `prometheus.`            | Prometheus            |
| `monitoring` | `alerts.`                | Alertmanager          |
| `monitoring` | `mon.`                   | Grafana               |

Portainer is not routed through Traefik; it publishes port 9443 directly on manager nodes.

### Update configuration

Compose `configs` are versioned with `STACK_VERSION`. After editing a config file, bump `STACK_VERSION` in `.env` and re-deploy the stack that consumes it:

- Traefik (`traefik/traefik.yml`, `traefik/dynamic.yml`) and MariaDB configs -> `infra`
- Backend (`backend/config.yml`) -> `app`
- Prometheus and Alertmanager configs -> `monitoring`

### Backups and metrics

- `db-backup` runs on the `DB_BACKUP__SCHEDULE` cron (default `@daily`) and uploads a MariaDB dump to `DB_BACKUP__STORAGE_URL`; it runs as a cron job, so `docker service ps` shows no running tasks between runs.
- Services labeled `prometheus.io/scrape=true` publish Prometheus metrics; scrape targets are discovered via those labels on the `internal` network.
- The full environment reference (every `__`-separated variable in [.env.example](.env.example)).

## 📚 Documentation

- [Central docs](https://docs.sms-gate.app/)
- [GitHub repository](https://github.com/android-sms-gateway/infra)

## 🤝 Contributing

1. Fork the repository and create a feature branch.
2. Keep changes scoped to a single concern (a compose file, a config file, or docs).
3. Open a pull request with a clear description of the change and any verification performed.
4. Add the `ready` label once the PR passes review, and `deployed` once the change is live.

Note the automation in `.github/workflows/`:

- `close-issues-prs.yml` marks issues/PRs stale after 7 days of inactivity and closes them after 7 more days; assigned items are exempt.
- `pr-labels.yml` removes the `ready` and `deployed` labels whenever new commits are pushed to a PR.

## ⚖️ License

Apache-2.0. See [LICENSE](LICENSE).

<!-- Reference-style badge URLs: style=for-the-badge is mandatory -->
[contributors-shield]: https://img.shields.io/github/contributors/android-sms-gateway/infra?style=for-the-badge
[contributors-url]: https://github.com/android-sms-gateway/infra/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/android-sms-gateway/infra?style=for-the-badge
[forks-url]: https://github.com/android-sms-gateway/infra/network/members
[stars-shield]: https://img.shields.io/github/stars/android-sms-gateway/infra?style=for-the-badge
[stars-url]: https://github.com/android-sms-gateway/infra/stargazers
[issues-shield]: https://img.shields.io/github/issues/android-sms-gateway/infra?style=for-the-badge
[issues-url]: https://github.com/android-sms-gateway/infra/issues
[license-shield]: https://img.shields.io/github/license/android-sms-gateway/infra?style=for-the-badge
[license-url]: https://github.com/android-sms-gateway/infra/blob/master/LICENSE
