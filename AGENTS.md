# Agent Guidelines — Elara Centralized Configuration

## Read first

- [README.md](README.md) — profile matrix, property ownership, Vault paths, add-service and add-profile procedures. The README owns the matrix; do not duplicate it here.
- [infra/README.md#verify-the-stack](../infra/README.md#verify-the-stack) — canonical full-stack build/start/verify commands.
- [infra/README.md](../infra/README.md) — Vault bootstrap (`scripts/setup-vault.sh`), Compose stacks and ports.
- [config-service](https://github.com/elara-app/config-service) — Config Server endpoints, Git settings and `main` label.

## Scope and boundaries

- This repository holds configuration only: per-service YAML, dev-only Vault bootstrap JSON, one CI workflow. There is no Maven build, no wrapper and no application code.
- Edit YAML only through this repository. Client repos and `infra` must not fork or override these property files with copies.
- Allowed changes: profile files, `service-secrets/*-dev-secrets.json`, `validate-config.yml`, `README.md`, `AGENTS.md`.
- Not owned here: Config Server behavior (`config-service`), client bootstrap imports/profile/retry/Vault connection (each client JAR), container topology and credentials (`infra`), production secrets (Vault).

## Quick facts

| Item | Value |
|---|---|
| Build | None — YAML and JSON only |
| Served by | `config-service` on 8888, default label `main` |
| Consumers | `api-gateway`, `inventory-service`, `unit-of-measure-service` |
| Naming | `<spring.application.name>-<profile>.yml`; the prefix must equal the app name |
| Secrets | Vault `secret/<app>/dev`; `service-secrets/*-dev-secrets.json` bootstrap dev only |
| Validation | `validate-config.yml`: YAML parse + `${EUREKA_URI:` check in the three dev files |

## Commands

There is no build and no test suite. Run these before handing off:

```bash
# Parse every YAML file (same parser as CI)
ruby -ryaml -e 'ARGV.each { |f| YAML.load_file(f) }' ./*.yml ./.github/dependabot.yml ./.github/workflows/*.yml

# Verify the required placeholder in every dev file that registers with Eureka
for f in api-gateway-dev.yml inventory-service-dev.yml unit-of-measure-service-dev.yml; do
  grep -qF '${EUREKA_URI:' "$f" || { echo "missing placeholder: $f"; exit 1; }
done

# Secret JSON files parse as JSON
jq empty service-secrets/*.json

# Compose still interpolates with this repo mounted (no daemon needed)
docker compose -f ../infra/compose.yml config --quiet
```

Full-stack verification is owned by `infra`: [infra/README.md#verify-the-stack](../infra/README.md#verify-the-stack).

## Repository map

| Path | Role |
|---|---|
| `api-gateway-dev.yml` | Gateway dev: port, routes, discovery locator, Swagger URLs, actuator, Eureka |
| `inventory-service-{dev,test,prod}.yml` | Inventory properties per profile; `prod` is a placeholder |
| `unit-of-measure-service-{dev,test,prod}.yml` | UOM properties per profile; `prod` is a placeholder |
| `service-secrets/*-dev-secrets.json` | Dev-only Vault bootstrap values (four keys each) |
| `.github/workflows/validate-config.yml` | CI: YAML parse and Eureka placeholder check |
| `.github/dependabot.yml` | Weekly GitHub Actions updates, target branch `develop` |
| `.gitignore` | Guard against committed prod secret files |

## Contracts and invariants

- Filename must be `<spring.application.name>-<profile>.yml`; Config Server resolves by exact prefix. Never rename a file or add a shared `application-*.yml` — none exists and layering is not supported here.
- Keep `eureka.client.service-url.defaultZone` as `${EUREKA_URI:<default>}` in every dev file that registers with Eureka. CI requires the literal `${EUREKA_URI:` in the three dev files it names.
- Config Server returns `${...}` placeholders raw. Clients resolve them. Do not "fix" placeholders by hardcoding hostnames.
- The served label is `main`. A merge to `develop` alone is not served.
- Vault paths are derived from `<app>-<profile>-secrets.json` as `secret/<app>/<profile>`. Paths in use are `secret/inventory-service/dev` and `secret/unit-of-measure-service/dev`; never introduce aliases such as `uom`.
- No production secret or real credential may enter this repository. `service-secrets/*-dev-secrets.json` stay dev-only default credentials.
- Gateway routes live here (`api-gateway-dev.yml`): explicit routes with `StripPrefix=3` plus the discovery-locator include-expression. Route changes affect only the gateway.
- The `-test.yml` files here are served on request, not test-harness config: each client's `application-test.yml` disables Config Server.

## Configuration and runtime

- Precedence: client JAR bootstrap < Config Server property sources < Vault (profile-specific import) < OS environment (Compose) < command line. See the README's Config precedence.
- The source stack bind-mounts this repository into Config Server and serves committed Git only. Commit before testing; uncommitted working-tree edits are not served.
- `force-pull: true` and `refresh-rate: 60` are server-side fetches. There is no push: business services refresh via `busrefresh`/`refresh`; the gateway exposes neither, so route changes need a restart.
- Both Compose stacks run `dev`; neither sets `SPRING_PROFILES_ACTIVE`.
- Client bootstrap files (`application.yml`, `application-dev.yml`, `application-test.yml`) are owned by the client repos. Change them there, not here.

## Testing and quality gates

- `validate-config.yml` runs only on `pull_request` to `develop` or `main`. It parses every `**/*.yml` with Ruby `Psych.parse_stream` (including `.github/dependabot.yml`) and requires the literal `${EUREKA_URI:` in exactly `api-gateway-dev.yml`, `inventory-service-dev.yml` and `unit-of-measure-service-dev.yml`.
- It does not check JSON, schema, unknown or duplicate keys, Vault paths, the profile matrix, or secrets.
- Local gate: the four commands above. This repository has no JaCoCo or Maven gate.

## Definition of done

- YAML and JSON parse locally; every dev file keeps the `${EUREKA_URI:` placeholder; the CI check list is updated when the set of dev files changes.
- Filename prefix equals `spring.application.name`, and the profile is present (or explicitly marked absent) in the README matrix.
- No production secrets or real credentials in any changed file; dev secret values remain non-sensitive defaults.
- Cross-repo coordination is done where applicable: client bootstrap/profile, Vault path/JSON, both Compose files and CI lists, gateway route if externally exposed.
- README matrix and limitations updated; service READMEs keep linking here instead of duplicating the matrix.
- `git diff --name-only` shows only the intended files.

## Stop and ask

- Adding a service or a new profile — client, Vault, Compose, CI and docs all need coordinated changes.
- Changing the datasource, RabbitMQ or Eureka override contract, or the `${EUREKA_URI:` pattern.
- Adding or editing any value in `service-secrets/*.json`, and anything involving prod secrets or a real Vault.
- Changing the gateway route table, `StripPrefix`, or Swagger aggregation.
- Renaming, deleting or moving files.
- Changing the served label or anything that makes `main` and `develop` diverge in what clients receive.

## Known traps

- `*-prod.yml` files are two-line placeholders, not production configuration; they carry no datasource, Eureka, RabbitMQ or management settings.
- The `-test.yml` files do not drive Maven tests; clients disable Config Server in their own `application-test.yml`, and the repo test files enable the H2 console while the client disables it.
- RabbitMQ drift: YAML defaults are `guest/guest`, while Compose and Vault use `elara/elara`; bare local runs fail against the Compose broker.
- `uom` is not a valid Vault alias; the path is `secret/unit-of-measure-service/dev`.
- Config Server returns placeholders raw; HTTP 200 with empty `propertySources` means the application name or profile did not match a file.
- Default label `main`: source stacks serve committed Git only, so even a committed `develop` branch is not served.
- The gateway has no `refresh`/`busrefresh` endpoint; route changes need a restart.
- The Vault loader is get-or-put and skips existing paths; after editing a dev JSON file, delete the path metadata before re-running.
- The Eureka placeholder check and the `infra` publish preflight both hardcode the three dev files; a new dev file stays unchecked until CI is extended.

## Documentation maintenance

- This README owns the profile matrix, property ownership, Vault paths and add-service/add-profile procedures.
- Service READMEs (`api-gateway`, `inventory-service`, `unit-of-measure-service`) link here for the matrix; do not duplicate it there.
- Update `README.md` and this file in the same change as any matrix, Vault-path, CI or Compose-coordination change.
