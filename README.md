# eEarly Admin Module Keycloak

This repository contains Jenkinsfiles and configuration to build and upload the Admin Keycloak artifact to Nexus.

It packages a [Keycloak](https://www.keycloak.org/) image pre-loaded with:

- the `eearly-admin` realm (`realm-config/eearly-admin-realm.json`) with clients `eearly-admin-service`, `eearly-admin-ui`, …
- the `e-early-admin` custom login theme (`themes/e-early-admin`)

---

## Prerequisites

| Requirement | Version / note |
|-------------|----------------|
| Docker | to build and run the image; Docker Compose is optional |

No Java/Maven toolchain is needed — the image is built directly from the upstream Keycloak base image.

## Run locally

### 1. Build the image

```bash
docker build -t eearly-admin-keycloak .
```

This runs `kc.sh build` with the realm and theme baked in, matching what Jenkins ships to Nexus/production.

### 2. Start a Postgres database for Keycloak

```bash
docker network create eearly-admin-keycloak-net

docker run -d --name eearly-admin-keycloak-db \
  --network eearly-admin-keycloak-net \
  -e POSTGRES_DB=keycloak \
  -e POSTGRES_USER=keycloak \
  -e POSTGRES_PASSWORD=keycloak \
  -p 5434:5432 \
  postgres:17
```

### 3. Run Keycloak

```bash
docker run -d --name eearly-admin-keycloak \
  --network eearly-admin-keycloak-net \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  -e KC_DB=postgres \
  -e KC_DB_URL=jdbc:postgresql://eearly-admin-keycloak-db:5432/keycloak \
  -e KC_DB_USERNAME=keycloak \
  -e KC_DB_PASSWORD=keycloak \
  -e KC_HOSTNAME=localhost \
  -e KC_HTTP_ENABLED=true \
  -p 8080:8080 \
  -p 9001:9000 \
  eearly-admin-keycloak \
  start --optimized --import-realm
```

`9001` (host) → `9000` (container management port) is used instead of `9000` on the host to avoid clashing with `eearly-admin-module-service`, which listens on `9000` locally.

### 4. Verify

- Admin console: http://localhost:8080 (log in with `admin` / `admin`, as set above)
- Health: http://localhost:9001/health/ready
- **Realm settings → eearly-admin** should already exist, imported from `realm-config/eearly-admin-realm.json`
- **Clients** should list `eearly-admin-service`, `eearly-admin-ui`, etc. — open a client's **Credentials** tab to fetch the secret needed by consuming services (see [eearly-admin-module-service](https://github.com/e-early/eearly-admin-module-service.git))

### 5. Stop / clean up

```bash
docker rm -f eearly-admin-keycloak eearly-admin-keycloak-db
docker network rm eearly-admin-keycloak-net
```

---

## Releasing

This repository uses a lightweight git-flow release helper. It only updates `version.json` and git state; it does not
run a local Keycloak build. Jenkins builds and deploys the Docker image from the release branch.

Start a release from an up-to-date `development` branch:

```bash
bash release.sh start
```

The start command asks for the release version, defaulting to `version.json` with `-SNAPSHOT` removed. It creates
`release/<version>`, sets `version.json` to the release version on that branch, bumps `development` to the next patch
snapshot version automatically, and pushes both branches. It fails if any local `release/*` branch already exists.

After Jenkins succeeds, finish the release from `development`:

```bash
bash release.sh finish
```

The finish command requires exactly one local `release/*` branch. It merges that branch into `master`, creates an
annotated tag named after the release version, deletes the local and remote release branch, and returns to
`development`.
