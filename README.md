HEAD
# dtc245160042-HoangCongSu

# RecipeHub - Recipe Website & DevOps

## 1. Project information

- Course: Software System Deployment and Administration
- Topic: 29 - Recipe Website
- Application: WordPress
- Database: MySQL
- Database management: phpMyAdmin
- Operating system: Ubuntu 18.04.6 LTS on VMware
- Container runtime: Docker Engine
- Orchestration: Docker Compose

## 2. Objectives

Deploy a recipe-sharing website with reverse proxy,
monitoring, centralized logging and basic hardening.

## 3. Technology stack

- WordPress
- MySQL
- phpMyAdmin
- Nginx Reverse Proxy and HTTPS
- Prometheus and Grafana
- Loki and Promtail
- Docker Engine and Docker Compose

## 4. Repository structure

- app/: application
- mysql/: database initialization
- nginx/: reverse proxy configuration
- prometheus/: metrics collection
- grafana/: dashboards and data sources
- loki/: centralized logging
- promtail/: log collection
- docs/: report and screenshots

## 5. Deployment progress

- [x] Initial project structure
- [ ] WordPress, MySQL and phpMyAdmin
- [ ] Nginx and HTTPS
- [ ] Prometheus and Grafana
- [ ] Loki, Promtail and LogQL
- [ ] Hardening and testing
- [ ] Final report and demonstration

## 6. Security

- Never commit real environment secrets.
- Never commit TLS private keys.
- Expose only necessary ports.
- Apply least-privilege database access.
 chore: initialize RecipeHub DevOps project
