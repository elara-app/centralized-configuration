# Centralized Configuration for Elara Platform

This repository provides version-controlled, environment-specific configuration for all Elara microservices. It is designed for use with a Spring Cloud Config Server (see [config-service](https://github.com/elara-app/config-service.git)), enabling dynamic, externalized configuration management across the platform.

## Structure and Usage

- Each YAML file follows the naming convention `<spring.application.name>-<profile>.yml` (e.g., `inventory-service-dev.yml`).
- Profiles supported: `dev`, `test`, `prod`.
- Each service loads its configuration at runtime from the config server, not from its own JAR.
- Example consumers: [inventory-service](https://github.com/elara-app/inventory-service.git), [unit-of-measure-service](https://github.com/elara-app/unit-of-measure-service.git), [api-gateway](https://github.com/elara-app/api-gateway.git).

## Service Configuration Files

| Service                | dev file                        | test file                        | prod file                        |
|------------------------|---------------------------------|----------------------------------|----------------------------------|
| api-gateway            | api-gateway-dev.yml             |                                  |                                  |
| inventory-service      | inventory-service-dev.yml       | inventory-service-test.yml        | inventory-service-prod.yml        |
| unit-of-measure-service| unit-of-measure-service-dev.yml | unit-of-measure-service-test.yml  | unit-of-measure-service-prod.yml  |

Each file contains only non-sensitive, operational configuration: database URLs, RabbitMQ, Eureka, service metadata, etc. No secrets or credentials are stored in these YAMLs.

## Service Secrets (`service-secrets/`)

The `service-secrets/` directory contains default development credentials for each service, tracked in Git for convenience. **These are not production secrets and do not expose critical data.**

- `inventory-service-dev-secrets.json`: Default dev credentials for inventory-service
- `uom-service-dev-secrets.json`: Default dev credentials for unit-of-measure-service

These files are loaded into Vault or a local dev secret store by the platform. They are safe to version control because they only contain non-critical, default values for local development and CI.

**Best practice:** Never store real production secrets or sensitive credentials in this repository. Use Vault or a secure secret manager for all critical data.

## Best Practices

- Keep all configuration changes atomic and traceable via Git history.
- Use strict naming conventions for all files to ensure correct resolution by the config server.
- Never embed secrets in YAML files; use the `service-secrets/` directory only for non-critical dev/test values.
- Validate configuration changes in a staging environment before promoting to production.
- Coordinate config changes with service deployments to avoid breaking changes.

## Example: How a Service Loads Its Config

1. Service starts and contacts the config server (see [config-service](https://github.com/elara-app/config-service.git)).
2. Config server serves the appropriate YAML based on the service name and active profile.
3. Service receives all its environment-specific properties (DB, RabbitMQ, Eureka, etc.) from this repository.

