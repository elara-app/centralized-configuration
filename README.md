[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/elara-app/centralized-configuration)

# Elara Centralized Configuration

This repository is the Git-backed configuration source for the Elara Config Server (`config-service`). It serves versioned, non-secret operational settings to three clients — `api-gateway`, `inventory-service` and `unit-of-measure-service` — and owns the gateway route table plus the dev-only Vault bootstrap inputs. It does not own Config Server behavior (`config-service/application.yml`), client bootstrap imports and profiles (each client JAR), container topology (`infrastructure`), or production secrets (Vault).

## At a glance

| Item | Value |
|---|---|
| Served by | Spring Cloud Config Server (`config-service`), port 8888 |
| Consumers | `api-gateway` (8080), `inventory-service` (8081), `unit-of-measure-service` (8082) — only these three |
| Default label | `main` |
| File naming | `<spring.application.name>-<profile>.yml` |
| Profiles served | gateway: `dev`; inventory/UOM: `dev`, `test`, `prod` |
| Secret source | Vault `secret/<app>/dev`; `service-secrets/*-dev-secrets.json` seed dev only |
| Validation | YAML parse plus `${EUREKA_URI:` check in the three dev files |
| Build tool | None — this repository holds YAML and JSON only |

## Responsibilities and boundaries

Owns:

- Per-service runtime properties (server port, datasource defaults, RabbitMQ, Eureka registration, actuator exposure, JPA dialect, app-specific properties) and the gateway route table, discovery locator, Swagger aggregation URLs and Eureka instance metadata in `api-gateway-dev.yml`.
- Dev-only Vault bootstrap values under `service-secrets/`.

Does not own:

- Config Server Git URI/label, clone/force-pull behavior and server port — `config-service`; client bootstrap (`spring.config.import`, active profile, retry, Vault connection) — each client JAR.
- Container hostnames, ports, credentials and the Vault loader — `infrastructure` Compose and scripts.
- Production secrets — Vault at runtime. See [Secrets](#secrets).

## Serving model

- Config Server resolves `<spring.application.name>-<profile>.yml` from the default label `main` and serves `GET /{application}/{profile}` and `GET /{application}/{profile}/{label}`.
- It returns `${...}` placeholders unresolved; the client resolves them against its own environment. `SERVER_PORT`, `EUREKA_URI` and credential values are never substituted server-side.
- An unknown application/profile can return HTTP 200 with an empty `propertySources` list. There is no fallback file here.
- The server only reads Git (`force-pull: true`, `refresh-rate: 60`; `clone-on-start: false` in the source stack, overridden to `true` by the published stack) and never pushes changes to clients.
- Clients import it as `optional:configserver:${CONFIG_URI:http://localhost:8888}` with `fail-fast: false` and retry, so a missing server does not abort startup.

## Profile matrix

| Application | dev | test | prod |
|---|---|---|---|
| `api-gateway` | `api-gateway-dev.yml` | not present | not present |
| `inventory-service` | `inventory-service-dev.yml` | `inventory-service-test.yml` | `inventory-service-prod.yml` (placeholder) |
| `unit-of-measure-service` | `unit-of-measure-service-dev.yml` | `unit-of-measure-service-test.yml` | `unit-of-measure-service-prod.yml` (placeholder) |

- **`prod` files are sparse placeholders, not production configuration.** Each holds a single app-specific property (`inventory.property` or `uom.property`) and no datasource, Eureka, RabbitMQ, management or Vault settings. Do not deploy `prod` expecting this repository to complete the client configuration.
- **`test` files are only served on request.** Maven tests do not consume them: the business clients' own `application-test.yml` disables Config Server, Vault, bus and discovery and supplies H2 settings locally; the gateway's test file only disables Config Server and Eureka.
- **The gateway has no `test`/`prod` file.** A non-dev gateway request gets no routes from this repository. Both Compose stacks run `dev` because `SPRING_PROFILES_ACTIVE` is never set there.

## Property ownership

| Concern | Keys set in the dev files |
|---|---|
| Server port | `server.port: ${SERVER_PORT:...}` — 8080 gateway, 8081 inventory, 8082 UOM |
| Datasource | `spring.datasource.url/username/password`; JPA `ddl-auto: update`; PostgreSQL dialect |
| RabbitMQ | `spring.rabbitmq.host/port/username/password` |
| Eureka | `eureka.client.service-url.defaultZone: ${EUREKA_URI:...}`; instance id and metadata |
| Vault | `spring.cloud.vault.application-name: ${spring.application.name}` |
| Actuator | Gateway `health,info,gateway`; business services `health,info,busrefresh,refresh` with health probes |
| App-specific | `inventory.property` for inventory; `uom.property` for UOM |
| Gateway routes | `/api/v1/inventory/**` → `lb://inventory-service`; `/api/v1/uom/**` → `lb://unit-of-measure-service`; both `StripPrefix=3` |

Credentials belong in Vault. The YAML files carry only non-secret settings plus dev-only default credentials (see [Secrets](#secrets)).

## Secrets

`service-secrets/*-dev-secrets.json` hold the four dev credentials — `spring.datasource.username`, `spring.datasource.password`, `spring.rabbitmq.username`, `spring.rabbitmq.password` — for both business services.

- **Dev-only defaults.** The values are the publicly known `postgres`/`postgres` and `elara`/`elara`. They are tracked in Git because they are not sensitive.
- **No production secrets exist here.** There are no `test` or `prod` secret files. Real secrets must live in the real Vault; never commit them to this repository.
- **`.gitignore` is a guard, not enforcement.** It ignores `service-secrets/*-prod-secrets.json` and `service-secrets/*-prod-*.json` (plus `.env`, `.env.*`, `*.log`), but `git add -f` bypasses it and nothing scans history.

Vault paths are exactly:

| Application | Path | Keys |
|---|---|---|
| `inventory-service` | `secret/inventory-service/dev` | the four keys above |
| `unit-of-measure-service` | `secret/unit-of-measure-service/dev` | the four keys above |

The path is derived from the filename as `secret/<app>/<profile>`. There are no aliases: `uom` or any short prefix is invalid. The source loader is get-or-put — an existing path is skipped, never overwritten — so after editing a JSON file, delete the Vault metadata before re-running. The source stack runs the loader as a profile-gated one-shot (`--profile bootstrap`, `vault-loader`, `PROFILES=dev`); the published demo stack duplicates these values inline and does not read this directory.

## Environment variable contract

Environment-variable placeholders are `${VAR:default}`; `${spring.application.name}` and `${random.uuid}` resolve client-side without defaults. Compose overrides the defaults through OS environment variables:

| Setting | Default here | Source Compose override |
|---|---|---|
| `SERVER_PORT` | 8080 / 8081 / 8082 | `SERVER_PORT_GATEWAY` / `SERVER_PORT_INVENTORY` / `SERVER_PORT_UOM` |
| `SPRING_DATASOURCE_URL` | `jdbc:postgresql://localhost:5432/inventory_db` / `unit_of_measure_db` | `postgres-inventory:5432` / `postgres-uom:5432` |
| `SPRING_RABBITMQ_HOST` / `PORT` | `localhost` / `5672` | `rabbitmq` / `5672` |
| `SPRING_RABBITMQ_USERNAME` / `PASSWORD` | `guest` / `guest` | `elara` / `elara` (Vault values win at runtime) |
| `EUREKA_URI` | `http://localhost:8761/eureka/` | `http://discovery-service:8761/eureka/` |
| `VAULT_URI` / `VAULT_TOKEN` | not set here — client defaults `http://localhost:8200` / all-zero dev token | `http://vault:8200` / Vault dev root token |

`CONFIG_URI` is also declared in the client JARs, not in this repository. **Known drift:** a bare local run without Vault uses the YAML `guest/guest` RabbitMQ defaults, while Compose and Vault use `elara/elara`; the broker rejects mismatched credentials. See [Current limitations](#current-limitations).

## Config precedence

The client JAR owns the bootstrap — application name, `spring.config.import`, default `dev` profile, retry and the Vault connection in `application-dev.yml`. Config Server owns the served property sources. Vault overrides same-named secret values.

Spring Boot resolution order is command line > Java system properties > OS environment > config data (local `application*.yml` plus the `configserver:` and `vault:` imports) > defaults. The profile-specific `optional:vault://secret/<app>/dev` import outranks the base Config Server import, so Vault's four credential keys beat the YAML fallbacks; Compose environment variables beat both. See the Spring Boot [externalized configuration reference](https://docs.spring.io/spring-boot/reference/features/external-config.html).

## Validation and CI

`.github/workflows/validate-config.yml` runs on pull requests to `develop` and `main`:

1. Parses every `**/*.yml` file with Ruby `Psych.parse_stream` (including `.github/dependabot.yml`) and fails on syntax errors.
2. Requires the literal `${EUREKA_URI:` in exactly `api-gateway-dev.yml`, `inventory-service-dev.yml` and `unit-of-measure-service-dev.yml`.

There is no schema, unknown-key, duplicate-key, JSON, Vault-path, secret or profile-matrix check, and no Config Server startup test. The `infrastructure` publish workflow fetches and re-checks the three dev files at the resolved `main` SHA.

Run the same checks locally:

```bash
ruby -ryaml -e 'ARGV.each { |f| YAML.load_file(f) }' ./*.yml ./.github/dependabot.yml ./.github/workflows/*.yml
for f in api-gateway-dev.yml inventory-service-dev.yml unit-of-measure-service-dev.yml; do
  grep -qF '${EUREKA_URI:' "$f" || { echo "missing placeholder: $f"; exit 1; }
done
jq empty service-secrets/*.json
```

## Add a service

1. Pick the exact `spring.application.name`; the Config Server resolves `<name>-<profile>.yml`, so the filename prefix must match.
2. Add `<name>-dev.yml` with `server.port: ${SERVER_PORT:<port>}`, management/health, and `${EUREKA_URI:...}` if it registers with Eureka. Add datasource/RabbitMQ blocks only if used, and `spring.cloud.vault.application-name` only if Vault is used.
3. Add the file to the CI Eureka check list in `validate-config.yml` if it is a Eureka client.
4. Add `<name>-test.yml` only if Config Server should serve test config; Maven tests do not load it. Add `<name>-prod.yml` only as a clearly sparse placeholder.
5. Add `service-secrets/<name>-dev-secrets.json` (dotted keys) if Vault-backed credentials are needed; the path is `secret/<name>/dev`, no aliases.
6. In the client JAR: Config Server import/profile/retry in `application.yml`, Vault import in `application-dev.yml`, Config Server/Vault disabled in `application-test.yml`.
7. If externally exposed, add the explicit gateway route (`Path=/api/v1/<prefix>/**`, `lb://<name>`, `StripPrefix=3`), the Swagger URL, and the discovery-locator `include-expression`.
8. Update both Compose files, the `infrastructure` compose-check lists, the client README/AGENTS and this README's matrix. Merge to `main`: that is the served label.

## Add a profile

1. Add `<app>-<profile>.yml`; there is no shared `application-<profile>.yml` to layer. Either make it operationally complete or label it a sparse placeholder.
2. Add or update the client's own `application-<profile>.yml` if runtime behavior differs, and activate the profile explicitly. Do not expect it to inherit datasource/Eureka/RabbitMQ from `dev`.
3. Add `<app>-<profile>-secrets.json` only if Vault-backed values are needed. `prod` loading is gated by a non-local `VAULT_ADDR` plus `CONFIRM_PROD=yes`; never commit prod values.
4. Extend CI if the new profile carries required placeholders (the Eureka check is hardcoded to the three dev files).
5. Set `SPRING_PROFILES_ACTIVE` per service in Compose if the stack must run it, update healthchecks, and update this README's matrix and limitations; merge to `main`.

## Run and verify

Static validation: run the three commands in [Validation and CI](#validation-and-ci). To serve a file locally, start `config-service` (or the full stack) and query it:

```bash
curl -s http://localhost:8888/inventory-service/dev | jq '.propertySources[].name'
```

Full-stack verification (build, start, seed, smoke endpoints) is owned by `infrastructure`; follow [infrastructure/README.md](https://github.com/elara-app/infrastructure#readme). The source stack bind-mounts this repository into Config Server and serves committed Git only — commit before testing, because uncommitted working-tree edits are not served.

## Current limitations

- `test` and `prod` files are sparse: `prod` is a two-line placeholder; `test` lacks port, Eureka, RabbitMQ, management and Vault settings.
- No automated key/schema/duplicate validation, no JSON check, no secret scan, and the CI Eureka check is hardcoded to three files.
- Config Server never pushes changes. Business services refresh through Spring Cloud Bus/actuator; the gateway exposes neither `refresh` nor `busrefresh`, so gateway route changes need a restart.
- The source stack serves committed Git only; merging to `develop` alone is not served because the default label is `main`.
- RabbitMQ credentials drift between YAML defaults (`guest/guest`) and Compose/Vault (`elara/elara`).
- There is no staging environment; validate with the local stack before merging.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Served values still contain `${...}` | Config Server returns placeholders raw | Resolve them in the client environment (env vars, Vault) |
| Tests ignore the `-test.yml` file | Client `application-test.yml` disables Config Server | Change the client test config if test coverage needs the server |
| `prod` client starts without a datasource | `*-prod.yml` is a placeholder | Supply values in the client/env; do not rely on this repository |
| Edited file is not served | Uncommitted working-tree change; Config Server serves committed Git | Commit, then refresh or restart the client |
| RabbitMQ `ACCESS_REFUSED` on a bare run | YAML default `guest/guest` | Run with the Vault loader/Compose `elara/elara` credentials |
| Vault keeps old values after a JSON edit | Loader is get-or-put and skips existing paths | `vault kv metadata delete secret/<app>/dev`, then re-run the loader |
| Gateway routes unchanged after a config edit | Gateway has no refresh endpoint | Restart the gateway |
| Config request returns 200 with empty `propertySources` | Application name or profile does not match a file | Use the exact `<spring.application.name>-<profile>` |

## Documentation

- [AGENTS.md](AGENTS.md) — agent commands, invariants, traps and definition of done.
- [infrastructure/README.md](https://github.com/elara-app/infrastructure#readme) — stack, ports, Vault bootstrap, full-stack verification.
- [config-service](https://github.com/elara-app/config-service) — Config Server behavior and endpoints; client repositories: [api-gateway](https://github.com/elara-app/api-gateway), [inventory-service](https://github.com/elara-app/inventory-service), [unit-of-measure-service](https://github.com/elara-app/unit-of-measure-service).
