# s390x Porting Changes for acmeair-mainservice-java

## Summary

This document describes the changes made to port the acmeair-mainservice-java project to the IBM s390x (Linux on Z) architecture.

## New File: Dockerfile.s390x

A new multi-stage Dockerfile was created for building and running on s390x.

### Key Changes from Original Dockerfile

| Aspect | Original Dockerfile | Dockerfile.s390x |
|--------|-------------------|-------------------|
| Build stage | None (pre-built WAR expected) | Multi-stage with `eclipse-temurin:11-jdk` |
| Runtime base image | `open-liberty:full-java17-openj9` | `icr.io/appcafe/open-liberty:full-java17-openj9-ubi` |
| Platform pinning | None | `--platform=linux/s390x` on all FROM lines |
| WAR copy | Hardcoded `acmeair-mainservice-java-7.1.war` | Wildcard `*.war` |

### Architecture Decisions

#### Two-stage JDK strategy (Java 11 build / Java 17 runtime)

- **Build stage uses Java 11** (`eclipse-temurin:11-jdk`): The maven-war-plugin version used by this project has known compatibility issues with Java 16+. Java 11 is used for the Maven build to avoid these issues. The `eclipse-temurin:11-jdk` image is available for s390x from Docker Hub.
- **Runtime stage uses Java 17** (`icr.io/appcafe/open-liberty:full-java17-openj9-ubi`): The application's `server.xml` enables `microProfile-7.1`, which requires Java 17 or later at runtime. The ICR (IBM Container Registry) Open Liberty image provides verified s390x support.

#### Base image selection

- **Build image**: `eclipse-temurin:11-jdk` — Eclipse Adoptium provides official s390x builds. Available from Docker Hub (reachable from the target environment).
- **Runtime image**: `icr.io/appcafe/open-liberty:full-java17-openj9-ubi` — IBM's official Open Liberty image from ICR with UBI base. Provides s390x multi-arch manifest support. ICR is reachable from the target environment.
- The original `open-liberty:full-java17-openj9` image from Docker Hub may not have s390x manifests, hence the switch to the ICR-hosted UBI variant.

#### Wildcard WAR copy

The COPY instruction uses `*.war` instead of the hardcoded artifact name `acmeair-mainservice-java-7.1.war`. This makes the Dockerfile resilient to version changes in pom.xml.

### What did NOT need to change

- **Java source code**: The application is pure Java with no JNI, native code, or architecture-specific dependencies. The WAR is fully portable across architectures.
- **server.xml / jvm.options**: Liberty server configuration is architecture-independent.
- **Maven dependencies**: All dependencies are pure Java (JUnit, CXF, javax.json) and require no s390x-specific variants.
- **Application configuration**: No architecture-specific environment variables or configuration changes are needed.

## Other Dockerfiles (Not Ported)

The following variant Dockerfiles were **not** ported to s390x:

| Dockerfile | Reason |
|-----------|--------|
| `Dockerfile-qn` (Quarkus native) | GraalVM CE native-image does not support s390x. Would require Mandrel for s390x native builds, which is a significant change. Use the JVM-mode Quarkus variant instead. |
| `Dockerfile-wf` (WildFly) | `quay.io/wildfly/wildfly:latest-jdk17` s390x availability is unverified. |
| `Dockerfile-pm` (Payara Micro) | `payara/micro` does not publish s390x images. |
| `Dockerfile-daily` | Uses `openliberty/daily` tags which may not consistently have s390x manifests. |
| `Dockerfile-slim` | Uses `open-liberty:kernel-slim-java11-openj9` — can be ported similarly to the main Dockerfile if needed. |

## docker-compose

The `mongo:latest` image used in docker-compose files supports s390x via multi-arch manifests. No changes needed for MongoDB.

## Build and Test Instructions

```bash
# Build the s390x image
docker build --platform linux/s390x -f Dockerfile.s390x -t acmeair-mainservice:s390x .

# Run with docker-compose (on s390x host or with QEMU emulation)
# Update docker-compose.yml to reference the new image tag and Dockerfile if needed
docker run --platform linux/s390x -p 9080:9080 acmeair-mainservice:s390x
```

## Environment Requirements

- Docker Hub: reachable (for `eclipse-temurin` build image)
- ICR (`icr.io`): reachable (for Open Liberty runtime image)
- Maven Central: reachable (for dependency resolution during build)
- `public.dhe.ibm.com`: **not reachable** (DNS does not resolve in target environment — no IBM artifacts from this host are required)
