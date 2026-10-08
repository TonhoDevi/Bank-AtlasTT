# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

Phase 0 done (app boots, connects to Postgres, `/actuator/health` is UP). No domain code or migrations yet. The rest below is the intended design from [planejamento.md](planejamento.md) (study plan, in Portuguese); verify against the code as it grows.

Educational project with fictional data: it must never move real money or connect to Pix or any real bank. Docs and plan are written in Portuguese; keep that language for docs.

## Configuration

`application.yml` holds shared config with `${DB_HOST}`, `${DB_PORT}`, `${DB_NAME}`, `${DB_USER}`, `${DB_PASSWORD}` placeholders (no defaults). Profile comes from `SPRING_PROFILES_ACTIVE` (default `dev`).
- `dev`: `application-dev.yml` imports the root `.env` via `spring.config.import`; more Actuator endpoints exposed.
- `prod`: no `.env`; variables must come from the real environment (app fails to start if missing); minimal Actuator exposure.
- `.env` is gitignored; copy from `.env.example`. `docker-compose.yml` reads the same `.env`.

## Goal

"Mini Banco Digital" to practice interview topics: NoSQL (MongoDB, Redis), messaging (Kafka preferred, or RabbitMQ) and observability (Actuator/Micrometer/Prometheus/Grafana, later OpenTelemetry + Jaeger). Stack: Java 21, Spring Boot **4.1.1** (the plan says 3.x, but 3.x is discontinued; Boot 4 renamed starters, e.g. `spring-boot-starter-webmvc` and per-starter `-test` artifacts), Maven, Flyway, Docker Compose.

## Architecture

Hexagonal (ports and adapters). Dependencies point inward only; `domain` must not depend on any framework (domain tests run without Spring).

Root package: `com.bankatlastt.minibanco` (the plan's `com.seuprojeto` is a placeholder).

- `domain` — `Conta`, `Transferencia`, domain exceptions
- `application` — `port.in` (use cases, e.g. `RealizarTransferenciaUseCase`), `port.out` (`ContaRepositoryPort`, `EventPublisherPort`, `ExtratoRepositoryPort`), `service` (`TransferenciaService`)
- `infrastructure` — `adapter.in.web`, `adapter.in.messaging`, `adapter.out.persistence` (JPA/Postgres), `adapter.out.mongo`, `adapter.out.cache` (Redis), `adapter.out.messaging`, `config`

Data flow: `POST /transferencias` → domain debits/credits in Postgres (`@Transactional`) → publishes `TransferenciaRealizada` event → consumers build the statement in MongoDB (eventual consistency, read via `GET /contas/{id}/extrato`), run antifraud rules, and send simulated notifications. Redis provides `Idempotency-Key` handling and balance cache.

Planned phases (each on a `feature/...` branch, Git Flow with `main`/`develop`): 0 setup, 1 domain + Postgres, 2 messaging, 3 Mongo, 4 Redis, 5 observability, 6 resilience (retry/DLQ/outbox), 7 polish (Testcontainers, GitHub Actions).

## Commands

```bash
docker compose up -d          # Postgres 16, values from .env
./mvnw spring-boot:run           # run app; check http://localhost:8080/actuator/health
./mvnw test                      # all tests
./mvnw test -Dtest=ClassName#method   # single test
```

Maven wrapper is present (`./mvnw` works in place of `mvn`). The JDK is pinned by [.sdkmanrc](.sdkmanrc) (Java 21 via SDKMAN; `sdk env` activates it, and `sdkman_auto_env=true` does it on `cd`). The default `java` outside the project is 17.

`flyway-database-postgresql` is in the pom. JPA uses `ddl-auto: validate`, so schema changes go through Flyway migrations (`src/main/resources/db/migration`). `MinibancoApplicationTests` (context load) needs a reachable Postgres until Testcontainers is added.
