# Repository Guidelines

## Project Structure & Module Organization
This repository is a Maven-based Spring Boot service (`Java 21`, `Spring Boot 4`).

- `src/main/java/com/yongzhiai/knowledge`: application source code
- `src/main/resources`: runtime config (`application.properties`)
- `src/test/java/com/yongzhiai/knowledge`: test code (JUnit + Spring Boot Test)
- `document/prd`: product requirement docs
- `document/prototype-ui`: HTML UI prototypes and review artifacts
- `target`: generated build output (do not edit directly)

## Build, Test, and Development Commands
Use the Maven wrapper so local Maven installation is optional.

- `./mvnw clean compile` (Windows: `.\mvnw.cmd clean compile`): compile main and test sources
- `./mvnw test`: run unit/integration tests
- `./mvnw clean verify`: full verification before opening a PR
- `./mvnw spring-boot:run`: run the app locally
- `./mvnw clean package`: produce the runnable jar in `target/`

Note: `compose.yaml` currently has no services. If you enable Docker Compose integration, define required services first.

## Coding Style & Naming Conventions
- Follow standard Java conventions: package names lowercase, classes/interfaces `PascalCase`, methods/fields `camelCase`.
- Keep package structure aligned with directory paths (for example, `com.yongzhiai.knowledge` -> `src/main/java/com/yongzhiai/knowledge`).
- Prefer one top-level class per file.
- Use your IDE formatter consistently; keep indentation and imports uniform across touched files.

## Testing Guidelines
- Frameworks: `JUnit 5` + `@SpringBootTest` from Spring Boot test starter.
- Place tests under mirrored package paths in `src/test/java`.
- Naming: use `*Test` or `*Tests` suffixes (current baseline: `KnowledgeApplicationTests`).
- Minimum contributor check before PR: run `./mvnw clean verify` and include results in PR notes.

## Commit & Pull Request Guidelines
Git history is not available in this workspace snapshot, so use Conventional Commit style by default:

- `feat(api): add knowledge search endpoint`
- `fix(test): stabilize context load test`

PRs should include:
- clear summary of behavioral changes
- linked issue/task ID (if available)
- test evidence (command + outcome)
- screenshots only when updating `document/prototype-ui` artifacts

## Agent-Specific Instructions
- 每当代理在任务中出现错误、遗漏或误判时，必须同步更新 `AGENTS.md`，补充可执行的预防规则，避免同类问题再次发生。
- 默认使用中文回复用户；仅在用户明确要求其他语言时，才切换回复语言。
