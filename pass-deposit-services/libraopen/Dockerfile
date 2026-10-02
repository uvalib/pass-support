# syntax=docker/dockerfile:1.4
#
# LibraOpen image for pass-deposit-services.
# Build context is the pass-support repository root.
#
# Overlays pass-deposit-services/libraopen onto deposit-core, installs
# pass-core-test-config from a pass-core checkout, then packages deposit-core.
# The run stage comes from deposit-core/Dockerfile.

FROM maven:3.9-eclipse-temurin-17 AS builder

ARG PASS_CORE_REF=main
ARG PASS_CORE_REPO=https://github.com/eclipse-pass/pass-core.git

RUN apt-get update \
    && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*

RUN set -eux; \
    test -n "${PASS_CORE_REF}" || { echo "PASS_CORE_REF is required (git branch or tag, e.g. main)"; exit 1; }; \
    git clone --depth 1 --branch "${PASS_CORE_REF}" "${PASS_CORE_REPO}" /src/pass-core

WORKDIR /src/pass-support

COPY . .

COPY pass-deposit-services/libraopen/DSpaceMetadataMapper.java \
    pass-deposit-services/deposit-core/src/main/java/org/eclipse/pass/deposit/provider/dspace/DSpaceMetadataMapper.java
COPY pass-deposit-services/libraopen/DSpaceMetadataMapperTest.java \
    pass-deposit-services/deposit-core/src/test/java/org/eclipse/pass/deposit/provider/dspace/DSpaceMetadataMapperTest.java

RUN --mount=type=cache,target=/root/.m2 \
    mvn -f /src/pass-core/pom.xml -pl pass-core-test-config -am install -DskipTests \
    && mvn -pl pass-deposit-services/deposit-core -am package \
        -Dtest=DSpaceMetadataMapperTest \
        -Dsurefire.failIfNoSpecifiedTests=false \
    && mkdir -p /out \
    && cp pass-deposit-services/deposit-core/target/deposit-core-*-exec.jar \
        /out/pass-deposit-service-exec.jar \
    && cp pass-deposit-services/deposit-core/entrypoint.sh /out/entrypoint.sh

FROM eclipse-temurin:17.0.18_8-jre-noble

WORKDIR /app

COPY --from=builder /out/pass-deposit-service-exec.jar pass-deposit-service-exec.jar
COPY --from=builder /out/entrypoint.sh .

RUN apt update \
    && apt --no-install-recommends install -y curl \
    && chmod +x entrypoint.sh \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -g 1432 passdsgroup \
    && useradd -m -u 1532 -g passdsgroup passdsuser \
    && chown -R passdsuser:passdsgroup /app

USER passdsuser

ENTRYPOINT ["./entrypoint.sh"]
