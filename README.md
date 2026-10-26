# Quarkus Plugin for Claude Code (v0.1.1)

A Claude Code plugin with production-ready skills and agents for building Quarkus / Java 21+ applications. Install the plugin and start generating your app — the skills and agents guide Claude to follow best practices for REST APIs, Panache, security, testing, and cloud-native deployment (Kubernetes, OpenShift, Knative).

> **Not yet listed on the official Anthropic plugin marketplace.** Until it is, install directly from this repository (see below) — once listed, `claude plugin install quarkus-template@piomin` will work without the `marketplace add` step too.

## Installation

Install from Anthropic marketplace **(soon...)**

```bash
claude plugin install quarkus-template@piomin
```

Or install directly from repository (works today)

```bash
claude plugin marketplace add https://github.com/piomin/claude-ai-quarkus.git
claude plugin install quarkus-template@piomin
```

To test locally during development:

```bash
claude --plugin-dir /path/to/claude-ai-quarkus
```

## What's Included

### Agents (`agents/`)

Specialized agents that Claude Code can invoke for specific tasks:

| Agent | Purpose |
|-------|---------|
| `quarkus-engineer` | Build Quarkus apps with layered architecture, Panache, and native-image-ready code |
| `security-engineer` | Quarkus Security, OIDC/JWT, Keycloak Dev Services, OWASP |
| `kubernetes-specialist` | Generated Kubernetes/OpenShift/Knative manifests, ServiceMonitor, migration Jobs, remote-dev |
| `devops-engineer` | CI/CD pipelines, Podman/Docker/Jib container images, native builds |

### Skills (`skills/`)

Reusable knowledge that Claude loads contextually:

| Skill | What it teaches |
|-------|-----------------|
| `quarkus` | Core Quarkus development with 6 reference guides (web, data, security, cloud, testing, messaging) |
| `code-quality` | Clean code principles, API contracts, null safety, exception handling |
| `design-patterns` | Classic patterns (Factory, Builder, Strategy) + enterprise (Saga, Outbox, CQRS) with CDI |
| `testing-patterns` | JUnit 5, Mockito, RestAssured, Dev Services (built-in Testcontainers), `@QuarkusIntegrationTest` |
| `java-modern` | Java 21+ features: records, sealed classes, pattern matching, virtual threads |
| `jpa-patterns` | JPA/Hibernate pitfalls (N+1, lazy loading) with Panache, optimistic locking |
| `logging-patterns` | Structured JSON logging, SLF4J/JBoss Logging, MDC, AI-friendly formats |

### Key Conventions

- **Explicit code** — explicit constructors, public fields on Panache active-record entities, `LoggerFactory.getLogger()`
- **Constructor injection** — a single constructor, no `@Inject` needed on it, never on fields
- **Panache** repository/active-record pattern over hand-written JPA boilerplate
- **RestAssured** for all HTTP tests — one client for blocking and reactive (Mutiny) resources alike
- **Dev Services** for local databases/brokers/Keycloak — no manual Testcontainers setup
- **Jakarta REST / CDI** throughout
- **CI pipeline** — auto-detected from existing project config
- **Podman preferred, Docker fully supported** for container and native-image builds

### Cloud-Native Coverage

The `cloud.md` reference and the `kubernetes-specialist`/`devops-engineer` agents cover:

- Generated Kubernetes/OpenShift/Knative manifests from code — no hand-written YAML
- Auto-generated health probes (SmallRye Health), Prometheus `ServiceMonitor`, and Liquibase/Flyway migration Jobs
- SmallRye Stork for service discovery and client-side load balancing
- SmallRye Fault Tolerance (`@CircuitBreaker`, `@Retry`, `@RateLimit`)
- OpenTelemetry with built-in OTLP export
- Virtual threads (`@RunOnVirtualThread`), REST Client, GraalVM native image (container-based builds, no local GraalVM needed)
- `quarkus:remote-dev` as the built-in live-coding loop for Kubernetes development
- TLS registry with hot certificate reload, paired with cert-manager on Kubernetes

## Usage

After installing, describe what you want to build — Claude will use the skills and agents automatically:

```bash
cd my-project
claude
> "Build a REST API for managing orders with PostgreSQL"
```

You can also invoke skills and agents directly:

```bash
# Invoke a skill
> /quarkus-template:jpa-patterns

# Use an agent
> @agent-quarkus-template:security-engineer
```

## Directory Structure

```
.
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── agents/
│   ├── devops-engineer.md
│   ├── kubernetes-specialist.md
│   ├── quarkus-engineer.md
│   └── security-engineer.md
├── skills/
│   ├── code-quality/SKILL.md
│   ├── design-patterns/SKILL.md
│   ├── java-modern/SKILL.md
│   ├── jpa-patterns/SKILL.md
│   ├── logging-patterns/SKILL.md
│   ├── quarkus/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── cloud.md
│   │       ├── data.md
│   │       ├── messaging.md
│   │       ├── security.md
│   │       ├── testing.md
│   │       └── web.md
│   └── testing-patterns/SKILL.md
├── README.md
└── LICENSE
```

## License

Apache License 2.0
