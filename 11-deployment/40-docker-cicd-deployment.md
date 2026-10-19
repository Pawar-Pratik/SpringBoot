# Lesson 40 — Docker, CI/CD & deployment

> **Why this lesson exists:** An application that isn't deployed is a hobby project. This lesson gets Vigil onto a real URL that real people can use — which is the single biggest difference between a portfolio project that impresses and one that gets skimmed. It also covers the deployment mechanics interviewers ask about: image layering, zero-downtime rollouts, migration ordering, and the 12-factor principles behind them.

**Time:** ~70 minutes · **Prereq:** Lessons 08, 12, 24, 35

---

## 1. The idea in one sentence

> **One artifact, promoted through environments, differing only in what the environment hands it — if you rebuild per environment you are not testing what you ship, and everything else in this lesson exists to make that one image start fast, stop cleanly, and be replaceable at any moment.**

That is also why a deploy is a correctness problem rather than a packaging one: during a rolling release two versions run at once against one database, so the schema, the API and the shutdown behaviour all have to tolerate having a predecessor.

---

## 2. The 12-Factor principles that actually matter

Not all twelve are equally important. These five determine whether deployment is calm or terrifying:

| Factor | Meaning | Consequence if violated |
|---|---|---|
| **III. Config in the environment** | One artifact, environment-supplied config | You rebuild per environment, so you don't test what you ship |
| **VI. Processes are stateless** | No local files, no in-memory sessions | You can't scale horizontally or replace an instance |
| **IX. Disposability** | Fast startup, graceful SIGTERM handling | Deploys drop requests; scaling is slow |
| **X. Dev/prod parity** | Same database, same versions | "Works on my machine"; H2-vs-Postgres bugs (Lesson 30) |
| **XI. Logs are event streams** | Write to stdout; the platform collects | Logs trapped in a dead container |

**The one that matters most: one artifact, promoted through environments.** The exact image that passed CI runs in staging and then production, differing only by environment variables. **If you rebuild per environment, you are not testing what you ship** — and that's the root of an entire genre of production surprise.

**Statelessness is the second.** Any in-memory session, local temp file, or in-process scheduler state means instances aren't interchangeable — which breaks scaling, rolling deploys, and instance replacement. Everything that must survive goes to Postgres, Redis, or object storage.

---

## 3. Dockerizing a Spring Boot app properly

### The naive version, and why it's wrong

```dockerfile
# ❌ Works, but poorly
FROM eclipse-temurin:21-jdk
COPY target/vigil.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Problems: a ~400MB JDK image when you only need a JRE; the whole 60MB fat JAR is **one layer**, so a one-line code change re-uploads all 60MB; it runs as **root**; and it ignores container memory limits.

### The production Dockerfile

```dockerfile
# ─── Stage 1: build ─────────────────────────────────────────────────────
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build

# ★ Copy dependency descriptors FIRST and download deps as their own layer.
#   Source changes then don't invalidate the dependency cache.
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline -B

COPY src/ src/
RUN ./mvnw clean package -DskipTests -B

# ★ Unpack the fat JAR into its natural layers
RUN java -Djarmode=tools -jar target/vigil-*.jar extract --layers --destination extracted

# ─── Stage 2: runtime ───────────────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

RUN addgroup -S vigil && adduser -S vigil -G vigil     # ★ never run as root
RUN apk add --no-cache curl

# ★ Layers ordered least- to most-frequently-changed → maximum cache reuse
COPY --from=builder --chown=vigil:vigil /build/extracted/dependencies/ ./
COPY --from=builder --chown=vigil:vigil /build/extracted/spring-boot-loader/ ./
COPY --from=builder --chown=vigil:vigil /build/extracted/snapshot-dependencies/ ./
COPY --from=builder --chown=vigil:vigil /build/extracted/application/ ./

USER vigil
EXPOSE 8080 9090

ENV JAVA_TOOL_OPTIONS="\
  -XX:MaxRAMPercentage=75 \
  -XX:+ExitOnOutOfMemoryError \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heapdump.hprof \
  -Djava.security.egd=file:/dev/./urandom"

HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD curl -fsS http://localhost:9090/actuator/health/liveness || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Why layer extraction matters

```
WITHOUT layers:                       WITH layers:
[ 60MB fat JAR ]  ← rebuilt every     [ 55MB dependencies  ]  ← changes on pom.xml edits
                    code change       [  1MB loader        ]  ← almost never
                                      [  0MB snapshots     ]
                                      [ 400KB application  ]  ← ★ the only layer that changes

Push/pull per deploy: 60MB            Push/pull per deploy: 400KB
```

**150× less data per deploy**, which means faster CI, faster rollouts, and lower registry cost. **Spring Boot's `jarmode=tools extract --layers` exists specifically for this** — and using it is a concrete signal you understand container mechanics rather than just copying a Dockerfile.

> **Note:** older tutorials use `-Djarmode=layertools`. That was replaced by `-Djarmode=tools extract --layers` in Boot 3.3+. Recognizing which you're looking at dates the material.

### The JVM flags that matter

| Flag | Why |
|---|---|
| `-XX:MaxRAMPercentage=75` | **Container-aware.** Hardcoded `-Xmx` in a container is a latent OOMKill (Lesson 36 §3) |
| `-XX:+ExitOnOutOfMemoryError` | Die cleanly so the orchestrator restarts you, rather than limping in an undefined state |
| `-XX:+HeapDumpOnOutOfMemoryError` | Diagnose a crash you can't reproduce (and treat the dump as a secret) |
| `-Djava.security.egd=file:/dev/./urandom` | Avoids blocking on entropy at startup in containers — a classic mysterious slow-start |

**Use `JAVA_TOOL_OPTIONS`, not a hardcoded `ENTRYPOINT` array**, so an operator can add flags at deploy time without rebuilding the image.

### The alternative: Buildpacks

```bash
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=vigil:1.0
```

**No Dockerfile at all.** Cloud Native Buildpacks produce an optimized, layered image with a non-root user, container-aware JVM settings, and automatic memory calculation. They also support **reproducible builds** and easy base-image patching (rebase without rebuilding).

**When to use which:** Buildpacks for speed and safe defaults — genuinely excellent, and the right default for most teams. A Dockerfile when you need OS packages, precise control, or your organization mandates one. **Knowing Buildpacks exist and choosing a Dockerfile deliberately is better than only knowing Dockerfiles.**

### Image hygiene

```dockerfile
# ★ Pin digests in production, not floating tags
FROM eclipse-temurin:21-jre-alpine@sha256:abc123...
```

**Also:** `.dockerignore` (exclude `target/`, `.git`, `.env` — **you do not want secrets or 500MB of git history in your build context**), scan images with Trivy or Grype in CI, and prefer distroless or Alpine for a smaller attack surface. **Alpine caveat worth knowing: it uses musl libc rather than glibc, which occasionally breaks native libraries** — if something behaves oddly, try a Debian-slim base.

---

## 4. Docker Compose — the whole platform locally

```yaml
# compose.yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: vigil
      POSTGRES_USER: vigil
      POSTGRES_PASSWORD: ${DB_PASSWORD:-vigil_dev}
    ports: [ "5432:5432" ]
    volumes: [ pgdata:/var/lib/postgresql/data ]
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U vigil" ]
      interval: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    ports: [ "6379:6379" ]
    healthcheck: { test: [ "CMD", "redis-cli", "ping" ], interval: 5s, retries: 10 }

  kafka:
    image: apache/kafka:3.8.0
    ports: [ "9092:9092" ]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  vigil:
    build: .
    depends_on:
      postgres: { condition: service_healthy }      # ★ wait for READY, not just started
      redis:    { condition: service_healthy }
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/vigil
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD:-vigil_dev}
      SPRING_DATA_REDIS_HOST: redis
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      JWT_SECRET: ${JWT_SECRET:?JWT_SECRET is required}     # ★ fail fast if unset
    ports: [ "8080:8080", "9090:9090" ]

  prometheus:
    image: prom/prometheus
    volumes: [ ./ops/prometheus.yml:/etc/prometheus/prometheus.yml:ro ]
    ports: [ "9091:9090" ]

  grafana:
    image: grafana/grafana
    ports: [ "3000:3000" ]
    volumes: [ ./ops/grafana/provisioning:/etc/grafana/provisioning:ro ]
    environment: { GF_AUTH_ANONYMOUS_ENABLED: "true", GF_AUTH_ANONYMOUS_ORG_ROLE: Admin }

volumes:
  pgdata:
```

**Two details worth copying:**

**`condition: service_healthy`, not just `depends_on`.** Plain `depends_on` waits for the container to *start*, not for Postgres to accept connections — so your app starts, fails to connect, and crashes. Health-gated dependencies fix it properly. (Your app should *also* tolerate a database that isn't ready yet, but this removes the noise.)

**`${JWT_SECRET:?...}` fails immediately with a clear message if unset.** Compose files silently substituting empty strings for missing variables is a genuinely annoying debugging experience; this makes it loud.

**`docker compose up` must bring up the entire platform.** If a reviewer can't run your project in one command, they won't run it — and an unrun project is an unassessed project. **This single file materially changes how your portfolio is received.**

Add a `compose.override.yaml` for local development conveniences (source mounts, debug ports) so the base file stays production-shaped.

---

## 5. Kubernetes essentials

You don't need to be an expert, but these are the objects and settings that come up.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: vigil-api }
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }     # ★ never drop below capacity
  selector: { matchLabels: { app: vigil-api } }
  template:
    metadata:
      labels: { app: vigil-api }
    spec:
      terminationGracePeriodSeconds: 45        # ★ MUST exceed your graceful shutdown timeout
      containers:
        - name: vigil
          image: ghcr.io/you/vigil@sha256:abc123...
          ports: [ { containerPort: 8080 }, { containerPort: 9090 } ]

          resources:
            requests: { memory: "512Mi", cpu: "250m" }     # ★ scheduling guarantee
            limits:   { memory: "768Mi", cpu: "1000m" }    # ★ hard cap; memory limit = OOMKill

          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef: { name: vigil-secrets, key: db-password }

          startupProbe:                        # ★ protects liveness during slow JVM startup
            httpGet: { path: /actuator/health/liveness, port: 9090 }
            failureThreshold: 30
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 9090 }   # ★ ping only!
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 9090 }
            periodSeconds: 5
            failureThreshold: 2

          lifecycle:
            preStop:
              exec: { command: [ "sh", "-c", "sleep 10" ] }   # ★★ see below

          securityContext:
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities: { drop: [ "ALL" ] }
```

### ⭐ The `preStop` sleep — the trick that gives you truly zero-downtime deploys

**The problem:** when Kubernetes terminates a pod it does two things *concurrently* — sends SIGTERM to your container, and removes the pod from Service endpoints. **Endpoint removal propagates asynchronously** through kube-proxy and ingress controllers, taking a few hundred milliseconds to seconds. So for a brief window, **traffic is still being routed to a pod that has already begun shutting down.**

**The fix:** a `preStop` hook that sleeps. During the sleep the pod keeps serving normally while endpoint removal propagates. Only afterwards does SIGTERM arrive and graceful shutdown begin.

**Combined with Spring Boot's graceful shutdown** (Lesson 08 §2):
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

```
t=0     Pod marked Terminating; endpoint removal begins; preStop sleep starts
t=0–10  Pod still serving. Endpoints converge. New traffic stops arriving.
t=10    SIGTERM → Boot stops accepting new connections, finishes in-flight requests
t=~12   In-flight complete → beans destroyed (@PreDestroy) → JVM exits
        (terminationGracePeriodSeconds=45 > 10 + 30, so we're never SIGKILLed)
```

**`terminationGracePeriodSeconds` must exceed `preStop` + shutdown timeout**, or Kubernetes SIGKILLs you mid-request — and `@PreDestroy` never runs (Lesson 08 §2).

**Being able to explain the preStop sleep, and *why* it's needed, is one of the strongest deployment answers you can give.** Most people configure graceful shutdown, see occasional 502s during deploys, and never work out why.

### Resource requests vs limits

- **`requests`** = what the scheduler reserves. Too low → your pod lands on a saturated node.
- **`limits`** = the hard cap. **Exceed the memory limit and you're OOMKilled instantly** (Lesson 36 §3). Exceed the CPU limit and you're *throttled*, not killed.
- **CPU throttling is a subtle latency killer:** a container at its CPU limit is paused for the rest of each 100ms scheduling period, producing periodic latency spikes that look like GC pauses but aren't. **Check `container_cpu_cfs_throttled_seconds_total` before blaming the JVM** — that's a genuinely useful diagnostic most people don't know.

**Also relevant:** a `PodDisruptionBudget` (`minAvailable: 2`) so node drains and cluster upgrades don't take all your replicas at once, and an `HorizontalPodAutoscaler` — for Vigil's checkers, scale on **check backlog** (a custom metric) rather than CPU, since they're I/O-bound and CPU tells you nothing.

---

## 6. CI/CD

```yaml
# .github/workflows/deploy.yml
name: Build & Deploy
on:
  push: { branches: [main] }

env:
  REGISTRY: ghcr.io
  IMAGE: ${{ github.repository }}

jobs:
  test:
    uses: ./.github/workflows/ci.yml          # reuse the CI pipeline from Lesson 31

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write, id-token: write }
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: push
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true                     # ★ SLSA build provenance

      - name: Scan the image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE }}:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: '1'                       # ★ fail the build on serious CVEs

  deploy-staging:
    needs: build-and-push
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: kubectl set image deployment/vigil-api vigil=${{ env.REGISTRY }}/${{ env.IMAGE }}@${{ needs.build-and-push.outputs.digest }}
      - name: Smoke test
        run: ./scripts/smoke-test.sh https://staging.vigil.io

  deploy-production:
    needs: deploy-staging
    environment: production                    # ★ requires manual approval in GitHub
    runs-on: ubuntu-latest
    steps:
      - run: kubectl set image deployment/vigil-api vigil=...@${{ needs.build-and-push.outputs.digest }}
      - run: ./scripts/smoke-test.sh https://vigil.io
```

**Four practices worth noting:**

1. **Tag with the commit SHA and deploy by digest.** `latest` is not a version — it's ambiguous, unrollbackable, and racy. **Digests are immutable, so you know exactly what's running.**
2. **One build, promoted.** Staging and production deploy the *same digest*. Rebuilding per environment means the thing you tested isn't the thing you shipped.
3. **A smoke test after each deploy.** Hit `/actuator/health`, create and delete a monitor, verify the status page renders. **A deploy that isn't verified isn't finished.**
4. **A manual approval gate for production** (GitHub Environments) with automatic staging — the pragmatic middle ground between "deploy on every merge" and "release once a month."

### Migrations in the deploy pipeline

**The ordering question interviewers ask:**

```
Rolling deploy: old and new pods run SIMULTANEOUSLY against ONE database.
```

**Therefore: every migration must be compatible with the currently-running version** (Lesson 24 §3). Which gives the ordering rule:

```
Additive migration (add a nullable column, add a table, add an index):
   → run the MIGRATION FIRST, then deploy. Old code ignores the new column.

Destructive change (drop or rename a column):
   → EXPAND/CONTRACT over multiple deploys. Never in one step.
```

**Where migrations run — three options, with a clear recommendation:**

| Approach | Notes |
|---|---|
| **Flyway on application startup** | Simple, and what most projects do. **Risk: N pods race**, though Flyway takes a lock. A long migration delays or blocks the rollout. |
| **A Kubernetes Job / init container before rollout** | ✅ **Cleaner.** One execution, and the deploy fails before any new pod starts if migration fails. |
| **Separate from deployment entirely** | Full control for large or risky migrations; more process. |

**And the rule for long-running data changes:** never put a multi-million-row `UPDATE` in a startup migration. It blocks the deploy, may hold locks, and there's no progress visibility. **Schema change as a migration; data backfill as a batched background job** (Lesson 24 §3).

### Deployment strategies

| Strategy | How | Trade-off |
|---|---|---|
| **Rolling** (Kubernetes default) | Replace pods gradually | ✅ Simple, no extra cost. Both versions run together, so compatibility is mandatory. |
| **Blue/green** | Two full environments, switch traffic | Instant rollback; **2× infrastructure**, and database compatibility is *still* required |
| **Canary** | Route a small % to the new version | ✅ Best risk reduction; needs traffic-splitting (gateway `Weight`, or a mesh) and good metrics |
| **Feature flags** | Deploy dark, enable per cohort | Decouples deploy from release entirely; flag debt accumulates |

**The point people miss about blue/green:** it does **not** free you from database compatibility. Both environments share one database, so a destructive migration breaks blue the moment you apply it for green. **Expand/contract is required regardless of deployment strategy** — a genuinely good clarification to offer.

---

## 7. Hosting Vigil for real — pick one and ship it

**This is the step that makes the project count.** A live URL transforms how it's received.

| Option | Cost | Effort | Notes |
|---|---|---|---|
| **Fly.io** | Free tier → ~$5/mo | Low | ✅ **Best fit.** Postgres, global regions, `fly deploy` from a Dockerfile |
| **Railway** | ~$5/mo | Very low | ✅ Easiest. Postgres + Redis one click, deploy from GitHub |
| **Render** | Free tier (spins down) | Very low | Free web service + Postgres; cold starts on the free tier |
| **Hetzner VPS + Docker Compose** | ~€4/mo | Medium | ✅ **Best value and most educational.** You run everything. |
| **AWS ECS/Fargate + RDS** | ~$30+/mo | High | Most "enterprise-relevant" on a CV; genuinely more work |
| **Oracle Cloud Free Tier** | Free (4 ARM cores, 24GB RAM) | Medium | Absurdly generous; ARM builds needed |

**My recommendation: Fly.io or a Hetzner VPS.** Fly if you want it working this afternoon; Hetzner if you want to learn ops properly (and you'll be able to talk about nginx, TLS, systemd, and backups — which is genuinely valuable).

```toml
# fly.toml
app = "vigil"
primary_region = "lhr"

[build]
  dockerfile = "Dockerfile"

[env]
  SPRING_PROFILES_ACTIVE = "prod"
  SERVER_PORT = "8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 1

[[http_service.checks]]
  interval = "15s"
  timeout = "2s"
  method = "GET"
  path = "/actuator/health/readiness"
  port = 9090

[[vm]]
  memory = "1gb"
  cpu_kind = "shared"
  cpus = 1
```
```bash
fly launch --no-deploy
fly postgres create --name vigil-db
fly postgres attach vigil-db                  # sets DATABASE_URL automatically
fly redis create
fly secrets set JWT_SECRET=$(openssl rand -base64 48)
fly deploy
```

**The deployment checklist — do all of it:**
- [ ] **A custom domain with HTTPS** (a free Let's Encrypt certificate via the platform). `vigil.yourdomain.dev` reads far better than `vigil-production-a1b2.fly.dev`.
- [ ] **Seeded demo data** and a **public demo login** (`demo@vigil.io` / `demo`), read-only or reset hourly. **Nobody will register to evaluate your project.** This one item probably doubles the number of people who actually look at it.
- [ ] **A public status page** demonstrating the product — monitoring some real public endpoints
- [ ] Actuator on a separate port, not publicly exposed (Lesson 28 §5)
- [ ] Automated database backups (managed Postgres does this; **verify a restore works** — an untested backup isn't a backup)
- [ ] `/actuator/info` exposing the git commit, so you know what's running
- [ ] **External monitoring of Vigil itself** — a dead-man's-switch heartbeat (Lesson 35 §6). *Vigil cannot monitor Vigil.*
- [ ] A sensible cost cap or budget alert so a runaway loop doesn't produce a surprise bill

---

## 8. Build & break

**1. The multi-stage Dockerfile** from §2 with layer extraction and a non-root user. Then measure:
```bash
docker build -t vigil:1 .
# change one line in a controller
docker build -t vigil:2 .        # ★ note how few layers rebuild
docker images vigil              # note the final size
docker history vigil:2           # ★ inspect the layers and their sizes
```
**Then build the naive single-layer version and compare both the size and the rebuild time.** Concrete numbers, again.

**2. Verify the security posture.**
```bash
docker run --rm vigil:2 whoami                       # must NOT be root
docker run --rm vigil:2 java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
docker run --rm -m 512m vigil:2 java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
```
**Confirm the heap size changes with the container limit** — that's `MaxRAMPercentage` working, and it's the fix for the OOMKill from Lesson 36.

**3. Try Buildpacks and compare.**
```bash
./mvnw spring-boot:build-image
docker images | grep vigil
```
Compare size, layer count, and build time against your Dockerfile. **Then decide which you'd ship and be able to justify it.**

**4. Scan the image.**
```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image vigil:2
```
**Read the findings.** Then update the base image and re-scan. **This is a real workflow, and seeing your own image's CVE list is instructive.**

**5. The full Compose stack** from §3. Then, from a clean clone:
```bash
git clone <your repo> && cd vigil && cp .env.example .env && docker compose up
```
**It must work.** Time it. **If a reviewer can't get it running in under five minutes, fix that before adding any feature.**

**6. ⭐ Prove zero-downtime deployment.** Locally with Compose (or on Fly):
```bash
# Terminal 1 — continuous load
hey -z 120s -c 10 http://localhost:8080/api/v1/monitors
# Terminal 2 — deploy mid-flight
docker compose up -d --build vigil
```
- **First, with `server.shutdown=immediate`** → count the failed requests in the hey output
- **Then with `graceful` + a `preStop` delay** → confirm **zero** failures

**That before/after number is one of the best things you can put in a README**, and the exercise teaches the preStop mechanic properly.

**7. The CI/CD pipeline** from §5: test → build → scan → push by digest → deploy staging → smoke test → gated production. **Then break something deliberately** (a failing test, a critical CVE) and confirm the pipeline stops.

**8. ⭐ Deploy it for real.** Fly.io or Hetzner. Custom domain. HTTPS. Demo login. Seeded data. **A public status page monitoring some real endpoints** (github.com, your own site) so a visitor sees the product actually working.

**9. Practise a rollback.** Deploy a deliberately broken version, notice it via your own monitoring, and roll back by digest. **Time yourself.** Knowing your MTTR — and that rollback works — is the difference between having a deployment and having a deployment *process*.

**10. Migration ordering, demonstrated.** Add a nullable column via Flyway and deploy — old pods keep working. Then attempt a destructive rename in one step while old pods run, and **watch the old pods fail.** Then do it properly with expand/contract. **You've now experienced the reason for that pattern rather than just read it.**

**11. Verify your backup.** Take a backup, restore it into a fresh database, and confirm the data is intact. **An untested backup is a hope, not a backup** — and this is a genuinely common real-world failure.

### Explain out loud (60s)

1. Why one artifact promoted through environments matters more than any other 12-factor rule.
2. What layer extraction changes about a deploy, in numbers.
3. The `preStop` sleep: the race it closes, and why graceful shutdown alone isn't enough.
4. Where migrations should run, and the ordering rule that makes rolling deploys safe.
5. Why blue/green doesn't excuse you from expand/contract.

---

## 9. Production rules

| Do / Never | Why |
|---|---|
| **Build one image and promote it** | The exact artifact that passed CI runs in staging and production; rebuilding per environment is the root of a whole genre of surprise |
| **Keep processes stateless** | An in-memory session, local temp file or in-process scheduler state makes instances non-interchangeable, which breaks scaling and rolling deploys |
| **Copy dependency descriptors first and extract Boot's layers** | Source changes then don't invalidate the dependency layer, which is the difference between pushing a few megabytes and a few hundred |
| **Use `-XX:MaxRAMPercentage`, never a hardcoded `-Xmx`** | The JVM reads the cgroup limit, so changing the pod's memory no longer leaves a latent OOMKill behind |
| **Pass flags via `JAVA_TOOL_OPTIONS`, not a hardcoded `ENTRYPOINT`** | An operator can then add a flag at deploy time without rebuilding and re-promoting the image |
| **Write a `.dockerignore`** | Otherwise `target/`, `.git` and `.env` go into the build context — half a gigabyte of history and, sometimes, your secrets |
| **Pin image digests in production, not floating tags** | A tag can be moved under you, which makes "the same deploy" produce a different image |
| **Make `docker compose up` bring up the whole platform** | A reviewer who can't run your project in one command won't run it, and an unrun project is an unassessed project |
| **Gate Compose dependencies on `service_healthy`** | Plain `depends_on` waits for the container to start, not for Postgres to accept connections, so your app starts and crashes |
| **Add a `preStop` sleep before SIGTERM** | Endpoint removal propagates asynchronously, so for a few hundred milliseconds traffic still arrives at a pod that has begun shutting down |
| **Keep `terminationGracePeriodSeconds` above `preStop` plus the shutdown timeout** | Otherwise Kubernetes SIGKILLs you mid-request and `@PreDestroy` never runs |
| **Run migrations as a Job or init container, not on pod startup** | One execution instead of N pods racing, and the deploy fails before any new pod starts if the migration does |
| **Never backfill millions of rows in a startup migration** | It blocks the deploy, may hold locks, and gives you no progress visibility — schema as a migration, data as a batched job |
| **Expand/contract regardless of deployment strategy** | Blue/green doesn't save you: both environments share one database, so a destructive migration breaks the old one immediately |
| **Scale I/O-bound workers on a backlog metric, not CPU** | A checker waiting on the network shows almost no CPU, so a CPU-based autoscaler never reacts to the work piling up |
| **Deploy it somewhere real** | A live URL is the single biggest difference between a portfolio project that gets opened and one that gets skimmed |

---

## 10. Interview traps

**Q1. How do you containerize a Spring Boot app properly?**
Multi-stage build (JDK to build, JRE to run), dependencies downloaded as their own cached layer, **extract the fat JAR into layers** (`-Djarmode=tools extract --layers`) ordered least- to most-frequently-changed so a code change pushes ~400KB instead of 60MB, a non-root user, `-XX:MaxRAMPercentage` for container awareness, and a health check. Or use Buildpacks (`spring-boot:build-image`), which does all of it without a Dockerfile.

**Q2. Why extract the JAR into layers?**
Docker layers are cached by content. One 60MB JAR layer is rebuilt and re-uploaded on every code change; splitting dependencies from application code means only the small application layer changes — dramatically faster builds, pushes and pulls.

**Q3. Why is hardcoded `-Xmx` wrong in a container?**
Heap isn't total memory — metaspace, code cache, thread stacks and direct buffers are outside it — so `-Xmx512m` in a 512MB container gets OOMKilled with no Java exception and no heap dump, just exit code 137. `-XX:MaxRAMPercentage=75` reads the cgroup limit and survives someone changing the pod's memory limit.

**Q4. Liveness vs readiness, and the classic mistake?**
Liveness = "is this irrecoverably broken?" → the pod is **killed**. Readiness = "can it serve now?" → removed from the load balancer. **Checking the database in liveness means a brief DB blip kills every pod simultaneously**, they restart, overwhelm the recovering database, and loop. Liveness = ping; readiness = essential dependencies only. Add a `startupProbe` so slow JVM startup doesn't trip liveness.

**Q5. ⭐ How do you achieve genuinely zero-downtime deploys?**
Four things together: `maxUnavailable: 0` so capacity never drops; **a `preStop` sleep** because endpoint removal propagates asynchronously and traffic still arrives for a moment after SIGTERM; Spring Boot `server.shutdown=graceful` so in-flight requests complete; and `terminationGracePeriodSeconds` greater than preStop + shutdown timeout so you're never SIGKILLed mid-request. Missing the preStop sleep is why people see occasional 502s despite configuring graceful shutdown.

**Q6. When do you run database migrations relative to deployment?**
Additive migrations first, then deploy — old code ignores new columns. Destructive changes never in one step: expand/contract across multiple deploys, because rolling deploys run old and new code against one database. Prefer a Kubernetes Job or init container over startup migrations (single execution; the deploy fails before new pods start), and never put a multi-million-row backfill in a migration — that's a batched background job.

**Q7. Does blue/green free you from migration compatibility?**
No. Both environments share one database, so a destructive migration breaks the old environment the moment it's applied. **Expand/contract is required regardless of deployment strategy** — the strategy affects rollback speed and cost, not schema compatibility.

**Q8. Rolling vs blue/green vs canary?**
Rolling: simple, no extra cost, both versions coexist. Blue/green: instant rollback at 2× infrastructure cost. Canary: best risk reduction — route a small percentage and watch error rates — but needs traffic splitting (gateway `Weight`, or a mesh) and good metrics. Feature flags decouple deploy from release entirely, at the cost of flag debt.

**Q9. Why tag images with a commit SHA and deploy by digest?**
`latest` is ambiguous, racy, and unrollbackable — you can't tell what's running or return to a known state. A SHA tag identifies the source; a digest is immutable and identifies the exact bytes. And promote **one** artifact through environments: rebuilding per environment means you didn't test what you shipped.

**Q10. What are resource requests and limits, and what's the subtle one?**
Requests are the scheduler's reservation; limits are the hard cap. Exceeding the memory limit is an instant OOMKill; exceeding the CPU limit causes **throttling**, not death — the container is paused for the remainder of each scheduling period, producing periodic latency spikes that look like GC pauses. Check `container_cpu_cfs_throttled_seconds_total` before blaming the JVM.

**Q11. Which 12-factor principles matter most?**
Config in the environment (one artifact, promoted — otherwise you don't test what you ship), stateless processes (or instances aren't interchangeable, so scaling and rolling deploys break), disposability (fast startup and graceful SIGTERM), dev/prod parity (the same database, not H2), and logs to stdout.

**Q12. How would you host this cheaply and credibly?**
Fly.io or Railway for the fastest path with managed Postgres and Redis; a Hetzner VPS with Docker Compose for the best value and the most operational learning. Then: custom domain with HTTPS, seeded demo data with a public demo login, Actuator on a non-public port, verified backups, and **external** monitoring — because Vigil can't monitor Vigil.

---

## 11. Retention check

1. Why does layer extraction reduce a deploy from 60MB to 400KB?
2. Explain the `preStop` sleep — what race does it close?
3. Give the migration ordering rule for additive vs destructive changes, and why rolling deploys force it.
4. Why doesn't blue/green save you from expand/contract?
5. What's the difference between exceeding a CPU limit and exceeding a memory limit?
6. Name three things your deployed demo must have for a reviewer to actually engage with it.

---

## Module 11 complete ✅

Vigil is now a deployed, monitored, containerized service on a real URL with a CI/CD pipeline and zero-downtime deploys. **That sentence is worth more on a resume than a list of frameworks.**

**Next:** [Lesson 41 — Vigil blueprint](../12-project-vigil/41-vigil-blueprint.md) — the complete system specification, so you can see how every lesson fits together.
