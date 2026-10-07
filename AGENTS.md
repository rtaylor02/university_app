# AGENTS.md — University App

## 1. Project Overview

A simple yet real-life university management app. Students, courses, and enrollments across
microservices, with an API gateway fronting a React UI.

| Layer | Technology |
|---|---|
| Backend | Spring Boot 4.x, Spring Cloud (OpenFeign, Eureka, LoadBalancer, Gateway, Resilience4j, Config Server) |
| Auth | Spring Security (OAuth2 Resource Server / JWT) |
| Tracing | Micrometer Tracing + Zipkin (Sleuth is EOL — do not use it) |
| API docs | springdoc-openapi / OpenAPI |
| Frontend | React + Vite, Tailwind CSS, shadcn/ui |
| Persistence | MySQL (one schema per service), Spring Data JPA |
| Boilerplate | Lombok (`@Getter`/`@Setter`/`@Builder`/`@RequiredArgsConstructor`; avoid `@Data` on JPA entities) |
| Messaging | None (sync OpenFeign only) |
| VCS / CI/CD | Git monorepo, GitHub Actions |
| Deploy | AWS free tier (EC2 + optional RDS), Docker Compose |

## 2. Services & Exposure

| Service | Port | Exposure | Purpose |
|---|---|---|---|
| frontend | 5173 | **Public** | React UI |
| api-gateway | 8080 | **Public** | Single entry point: routing, JWT validation, CORS, load balancing |
| auth-service | 8084 | **Public** (login endpoints only) | Issues JWTs via Spring Security |
| student-service | 8081 | Private | Student CRUD |
| course-service | 8082 | Private | Course CRUD |
| enrollment-service | 8083 | Private | Enrollments; calls student/course via OpenFeign |
| config-server | 8888 | Private | Serves externalized config from `config-repo/` |
| eureka-server | 8761 | Private | Service registry |
| zipkin | 9411 | Private | Trace collection/UI |
| MySQL | 3306 | Private | `student_db`, `course_db`, `enrollment_db`, `auth_db` |

Rules:
- Only `frontend` and `api-gateway` are directly reachable by clients.
- Private services must never be exposed outside the host; call them through the gateway + Eureka.
- No database sharing: each service owns its schema.

## 3. Startup Order

1. `docker compose up -d` (MySQL, Zipkin)
2. config-server → 3. eureka-server → 4. auth-service, student/course/enrollment → 5. api-gateway → 6. frontend

Services should retry config-server/Eureka connections rather than crashing (config import retry + eureka retry).

## 4. Commands

Backend (per service dir):
- Run: `./mvnw spring-boot:run`
- Test: `./mvnw test`
- Package: `./mvnw clean package`

Frontend:
- Dev: `npm run dev`
- Lint: `npm run lint`
- Typecheck: `npm run typecheck`
- Test: `npm test`
- Build: `npm run build`

Infra: `docker compose up -d` / `docker compose down`

## 5. Conventions

- **OpenAPI-first**: every controller endpoint documented via springdoc; Swagger UI at `/swagger-ui.html`.
- **Config from config-server only** — no hardcoded URLs, credentials, or ports in `application.yml`; only identifiers (`spring.application.name`, config import).
- **JWT required** on all gateway routes except `/auth/**`. auth-service signs tokens; gateway and private services validate the same token (resource server). Secrets only in env vars / GitHub secrets, never in git.
- **Resilience**: every OpenFeign client and gateway route using Feign/load-balanced calls must define a Resilience4j circuit breaker + fallback. No silent failures.
- **Tracing**: Micrometer Tracing (Brave) on every service; Zipkin receives spans automatically. `traceId`/`spanId` must appear in logs.
- **No cross-service DB access.** Use Feign.
- **DB migrations (Flyway)**: schema changes live in `src/main/resources/db/migration/VN__name.sql`; set `spring.jpa.hibernate.ddl-auto=validate`. Never `update`/`create-drop` outside local dev.
- **Error responses**: RFC 7807 `ProblemDetail` across all controllers/services (timestamp, status, title, detail, instance, `traceId`).
- **Validation**: `@Valid` on every `@RequestBody`; explicit annotations (`@NotBlank`, `@Size`, etc.) on DTO records.
- **Pagination**: collection endpoints return the standard Spring `Page<T>` JSON shape (`content`, `pageNumber`, `pageSize`, `totalElements`, `totalPages`, `last`).
- **CORS**: configured only at the api-gateway; disabled on all private services.
- **Testing**: `@WebMvcTest` for controllers, `@DataJpaTest` for repositories, WireMock for Feign clients, Testcontainers for MySQL in integration tests. Use `./mvnw test` / `./mvnw verify`.
- Java 25 (Spring Boot 4 baseline). Constructor injection. Records for DTOs.
- **Lombok**: use `@Getter`/`@Setter`/`@Builder`/`@RequiredArgsConstructor` for services/entities; constructor injection via `@RequiredArgsConstructor` + `final` fields. Do **not** use `@Data` on JPA entities — its `equals`/`hashCode`/`toString` trigger lazy loading and recursion on relations; write equals/hashCode from the business key instead. Keep records for DTOs. Use a Lombok version compatible with Java 25, and enable annotation processing in IDE/CI.

## 6. Authentication Flow (JWT via Spring Security)

1. Client `POST /auth/login` → auth-service validates credentials (Spring Security `AuthenticationManager`) → returns signed JWT (HS256 dev / RS256 prod).
2. Client sends `Authorization: Bearer <token>` to the gateway.
3. Gateway (resource server) validates signature/expiry, then forwards the token downstream.
4. Private services also validate the token independently (defense in depth) and read the caller via `@AuthenticationPrincipal` / principal name.
5. Key/secret lives in config-server backed by an env var; rotated via redeploy.
6. Frontend: an Axios request interceptor attaches `Authorization: Bearer <token>` to every gateway call; a response interceptor redirects to `/login` on 401 (expired/invalid token).

## 7. Recording Changes

- **Branches**: `main` (deployable), `feat/...`, `fix/...`, `chore/...` per change.
- **Commits**: Conventional Commits — `feat:`, `fix:`, `docs:`, `ci:`, `chore:`, `refactor:`, `test:`. Enforced by a PR lint check.
- **PRs**: required to merge to `main`; description must include what/why/how-to-test; CI must pass.
- **CHANGELOG.md**: auto-generated by `git-cliff` from commit history on every `v*` tag (CD pipeline); do not edit manually.

## 8. CI/CD (GitHub Actions)

CI (`.github/workflows/ci.yml`, on PRs + pushes to `main`):
- Backend: `./mvnw verify` per service (matrix).
- Frontend: `npm ci`, lint, typecheck, test, build.
- Docker: build images (push only on tags).

CD (`.github/workflows/cd.yml`, on tags `v*` only):
1. Build & push images to Amazon ECR.
2. `git-cliff` regenerates `CHANGELOG.md` → commit back to `main` + GitHub Release notes.
3. SSM/SSH into EC2: `docker compose pull && docker compose up -d`.
4. Rollback: re-run CD on a previous tag.

AWS free-tier shape: single EC2 t3.micro running all services via Docker Compose, Elastic IP, security groups open only 80/443/22 (restricted); optional RDS db.t3.micro (12-month free tier) instead of containerized MySQL. Kafka intentionally excluded — sync Feign is sufficient for this app's scale.

## 9. Definition of Done

For any feature to be considered complete:
- Service registered in Eureka and config loaded from config-server.
- All endpoints documented in OpenAPI/Swagger UI.
- JWT enforced (except `/auth/**`); CI green; traces visible in Zipkin.
- Unit/integration tests pass (`./mvnw verify`, frontend test/typecheck).
- Commit follows Conventional Commits; merged via PR.
