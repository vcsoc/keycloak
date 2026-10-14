# Local Keycloak fork

The running image `vcsoc/keycloak:local` was built from this fork at commit `df42fa2f294a95315e7917ef07ca787d116c2701`, not downloaded as an upstream Keycloak release. The default branch identifies itself as `999.0.0-SNAPSHOT`.

## Run

```sh
# Create the shared network only if it does not already exist:
docker network inspect keycloak-local_default >/dev/null 2>&1 || docker network create keycloak-local_default
docker compose -f compose.local.yaml up -d --no-build
```

Keycloak: http://localhost:8180. Bootstrap administrator credentials are in the private, gitignored `.env`. Existing `license-admin` credentials and the `litellmlic` realm were preserved.

The bind mount `/home/vcsoc/data/keycloak` → `/opt/keycloak/data` persists the development H2 database independently of the repository and container. Directory ownership is UID 1000, GID 0, with mode 700. Do not mount this database into two running Keycloak servers at once. This localhost-only development setup is not intended for production.

Waypass runs separately from `/home/vcsoc/Projects/waypass` and reaches this stack over `keycloak-local_default`.

## Rebuild from source

Use JDK 21 and Docker. On this machine JDK 21 is available through mise; the build below does not alter the global Java version.

```sh
MAVEN_OPTS=-Xmx3g mise exec java@temurin-21.0.12+101.0.LTS -- ./mvnw -B -ntp -pl quarkus/deployment,quarkus/dist -am -DskipTests clean install
mkdir -p quarkus/container/target
cp quarkus/dist/target/keycloak-999.0.0-SNAPSHOT.tar.gz quarkus/container/target/
docker compose -f compose.local.yaml build --build-arg KEYCLOAK_SOURCE=https://github.com/vcsoc/keycloak
docker compose -f compose.local.yaml up -d --no-build
```

The build includes the fork's Java code and web UIs. Tests are skipped for the local distribution build. If the fork changes its Maven version from `999.0.0-SNAPSHOT`, update the tar filename and build arguments in `compose.local.yaml` accordingly.

## Database migration and updates

The previous upstream 26.4.0 H2 data was copied while stopped from Docker volume `waypass_keycloak-data`. That original volume is retained as an untouched rollback backup. The snapshot migration flag in `compose.local.yaml` explicitly permits migrating that copied database to the development build.

**Development snapshots do not provide a supported database upgrade path.** Before pulling/rebuilding a newer fork revision, stop Keycloak and back up `/home/vcsoc/data/keycloak`. Do not automatically update the source or overwrite backups. To roll back to upstream 26.4.0, use the original volume with the original image, not the snapshot-migrated database.
