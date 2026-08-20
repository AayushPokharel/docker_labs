
# MODULE 2 — ENTERPRISE DOCKER AND AZURE CONTAINER REGISTRY

**Objective.** Take the working-but-unshippable image from previous Module and turn it into an artifact you would defend in a security review: minimal, non-root, read-only, scanned, signed, immutably tagged, and published by a pipeline that holds no long-lived credentials.

**Deliverable.** A hardened image in Azure Container Registry, published by a pipeline that fails on critical CVEs, referenced by digest.

**The bridge from earlier Module.** Nothing about the application changes. Every change in this module is to the *build*, the *runtime posture*, and the *supply chain*. Keep `docker compose up` working throughout — if a hardening step breaks the stack, that is the lesson, and you fix it rather than reverting it.

---

## Lab 2.1 — Multi-Stage Builds

### Why and what

Your build needs the SDK. Your runtime does not. A multi-stage build uses one image to compile and a different, smaller image to run, copying only the published output across the boundary. Everything left behind in the build stage — compilers, NuGet caches, source code, Git history — never reaches production.

This is not primarily an optimization. It is a security control. Source code and build tooling in a production image give an attacker material to work with and CVEs you have no reason to be exposed to.

### Do this — write the multi-stage Dockerfile

```bash
cat > Dockerfile <<'EOF'
# syntax=docker/dockerfile:1.7
# v4: multi-stage, non-root, minimal runtime surface.

ARG DOTNET_VERSION=8.0

# ---------- Stage 1: build ----------
FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION}-noble AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

# Dependency layer: changes only when the manifest changes.
COPY src/LegacyLedger/LegacyLedger.csproj ./LegacyLedger/
RUN --mount=type=cache,target=/root/.nuget/packages,sharing=locked \
    dotnet restore LegacyLedger/LegacyLedger.csproj

# Source layer: changes on every commit.
COPY src/LegacyLedger/ ./LegacyLedger/
RUN --mount=type=cache,target=/root/.nuget/packages,sharing=locked \
    dotnet publish LegacyLedger/LegacyLedger.csproj \
      -c ${BUILD_CONFIGURATION} \
      -o /app/publish \
      --no-restore \
      /p:UseAppHost=false

# ---------- Stage 2: runtime ----------
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-noble AS runtime

# OCI labels: provenance that survives the build. Populated by the pipeline.
ARG GIT_SHA=local
ARG BUILD_VERSION=0.0.0-local
ARG BUILD_DATE=unknown
LABEL org.opencontainers.image.title="LegacyLedger" \
      org.opencontainers.image.description="Invoice tracking monolith" \
      org.opencontainers.image.revision="${GIT_SHA}" \
      org.opencontainers.image.version="${BUILD_VERSION}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.source="https://dev.azure.com/contoso/ledger" \
      org.opencontainers.image.vendor="Contoso Engineering"

ENV ASPNETCORE_HTTP_PORTS=8080 \
    DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_EnableDiagnostics=0 \
    LEDGER_DATA_PATH=/app/App_Data

WORKDIR /app

# UID 1654 is the 'app' user provided by the Microsoft .NET base images.
# Use the numeric ID explicitly so Kubernetes runAsUser can match it.
COPY --from=build --chown=1654:1654 /app/publish .
RUN mkdir -p /app/App_Data && chown 1654:1654 /app/App_Data

USER 1654

EXPOSE 8080
ENTRYPOINT ["dotnet", "LegacyLedger.dll"]
EOF
```

Three things to notice:

- `DOTNET_EnableDiagnostics=0` disables the diagnostic IPC socket. It is a debugging channel you do not want listening in production.
- `/p:UseAppHost=false` skips generating a native launcher you will never use, since the entrypoint invokes `dotnet` directly.
- There is deliberately **no `HEALTHCHECK`** instruction. Minimal base images have no shell and no `curl`. Health probes belong in the orchestrator — Compose, Kubernetes, or App Service — where they are configurable without a rebuild.

### Do this — build and compare

```bash
docker build -t ledger:v4-multistage \
  --build-arg GIT_SHA="$(git rev-parse --short HEAD)" \
  --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" .

docker images ledger --format 'table {{.Tag}}\t{{.Size}}' | sort
```

**Verify.** `v4-multistage` should be roughly a quarter the size of `v3-ignore`. Record it in your table from Lab 1.4.

### Do this — prove the build tooling is gone

```bash
docker run --rm --entrypoint sh ledger:v4-multistage -c 'which dotnet; ls /usr/share/dotnet/sdk 2>&1 | head -1; ls /app/*.cs 2>&1 | head -1'
```

**Verify.** The `dotnet` runtime exists, but there is no SDK directory and no `.cs` source files. Your source code is no longer shipped to production.

### Do this — confirm it still runs and still works

```bash
docker compose build app && docker compose up -d
sleep 45
curl -s http://localhost:8080/healthz ; echo
```

**Verify.** Still `{"status":"healthy"}`. Note that Compose builds `app` from this same Dockerfile, so your local stack is now running the hardened image.

### Troubleshoot if health fails

```bash
docker compose logs app --tail 30
```

If you see `Access to the path '/app/App_Data/audit.log' is denied`, the volume was created earlier with root ownership. Fix it:

```bash
docker compose down
docker volume rm ledger_app-data
docker compose up -d
```

This is the non-root/volume-ownership interaction warned about in Lab 1.8. Named volumes take ownership from the image directory at first creation, which is exactly why the Dockerfile creates and chowns `/app/App_Data` before switching user.

---

## Lab 2.2 — Base Image Selection

### Why and what

Your base image is the majority of your attack surface and almost none of your value. The engineering question is: what is the smallest base that still runs your application correctly?

| Base                                | Approximate size | Shell  | Package manager | glibc/musl | Trade-off                                                                                           |
| ----------------------------------- | ---------------- | ------ | --------------- | ---------- | --------------------------------------------------------------------------------------------------- |
| `aspnet:8.0-noble`                  | ~220 MB          | yes    | apt             | glibc      | Familiar, debuggable, largest CVE surface                                                           |
| `aspnet:8.0-alpine`                 | ~110 MB          | yes    | apk             | **musl**   | Small, but musl breaks some native libraries, and **no ICU**                                        |
| `aspnet:8.0-alpine-extra`           | ~135 MB          | yes    | apk             | **musl**   | Alpine plus ICU and tzdata                                                                          |
| `aspnet:8.0-noble-chiseled`         | ~115 MB          | **no** | **no**          | glibc      | Ubuntu with everything non-essential removed. **No ICU.**                                           |
| `aspnet:8.0-noble-chiseled-extra`   | ~135 MB          | no     | no              | glibc      | Chiselled plus ICU and tzdata. **The right base for this application.**                             |
| `cgr.dev/chainguard/aspnet-runtime` | ~110 MB          | no     | no              | glibc      | Frequently rebuilt, often zero known CVEs, signed and SBOM-attested. Paid tier for version pinning. |

The chiselled variants are the default recommendation for .NET on Linux. You get glibc compatibility with a distroless-scale footprint, and you do not inherit musl's edge cases.

**The `-extra` suffix is the decision this application forces.** The plain `alpine` and `noble-chiseled` images ship without ICU and set `DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=true`. As you saw in Lab 1.3, `Microsoft.Data.SqlClient` resolves named cultures internally and throws on startup without ICU. So the 20 MB you would save with a non-extra variant costs you a working application. Measure it yourself in the next steps rather than taking it on trust.

### Do this — build four variants and measure

```bash
for BASE in noble alpine-extra noble-chiseled noble-chiseled-extra; do
  docker build -t "ledger:base-${BASE}" -f - . <<DOCKERFILE
# syntax=docker/dockerfile:1.7
FROM mcr.microsoft.com/dotnet/sdk:8.0-noble AS build
WORKDIR /src
COPY src/LegacyLedger/LegacyLedger.csproj ./LegacyLedger/
RUN dotnet restore LegacyLedger/LegacyLedger.csproj
COPY src/LegacyLedger/ ./LegacyLedger/
RUN dotnet publish LegacyLedger/LegacyLedger.csproj -c Release -o /app/publish --no-restore /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0-${BASE}
ENV ASPNETCORE_HTTP_PORTS=8080 DOTNET_EnableDiagnostics=0 LEDGER_DATA_PATH=/app/App_Data
WORKDIR /app
COPY --from=build --chown=1654:1654 /app/publish .
USER 1654
EXPOSE 8080
ENTRYPOINT ["dotnet", "LegacyLedger.dll"]
DOCKERFILE
done

docker images ledger --format 'table {{.Tag}}\t{{.Size}}' | sort
```

**Verify.** Four new images. Record the sizes. The `-extra` variants and Alpine land in a similar range, all dramatically under the Debian-based `noble` variant.

### Do this — prove the ICU dependency at runtime, not in theory

```bash
for BASE in noble-chiseled noble-chiseled-extra; do
  echo "=== $BASE ==="
  docker run --rm \
    -e ConnectionStrings__LedgerDb='Server=127.0.0.1,1433;Database=LedgerDb;User Id=sa;Password=x;Encrypt=False;Connect Timeout=2' \
    -e DB_WAIT_ATTEMPTS=1 \
    "ledger:base-${BASE}" 2>&1 | grep -m1 -E 'invariant culture|Database not ready' || echo "(no output captured)"
done
```

**Verify.** `noble-chiseled` reports the invariant-culture error from Lab 1.3. `noble-chiseled-extra` reports an ordinary connection failure instead, because there is no database at that address — which is the *correct* error and proves ICU is present. Two images of near-identical size, one of which cannot run your application at all.

> The chiselled Dockerfile above does not `mkdir /app/App_Data` because there is no shell to run `RUN` commands that need one. In chiselled images, create writable paths by mounting a volume at that path instead. Adjust the Compose volume mount accordingly, which you already have.

### Do this — feel what "no shell" means

```bash
docker run --rm --entrypoint sh ledger:base-noble-chiseled-extra -c 'echo hi' 2>&1 | head -2
docker run --rm --entrypoint /bin/ls ledger:base-noble-chiseled-extra / 2>&1 | head -2
```

**Verify.** Both fail. There is no `sh` and no `ls` in the image. An attacker who achieves remote code execution has no shell to pivot with, no `curl` to pull a second-stage payload, and no package manager to install one.

### Do this — learn how to debug an image with no shell

```bash
docker run -d --name chisel-test --network ledger_backend \
  -e ConnectionStrings__LedgerDb='Server=db,1433;Database=LedgerDb;User Id=sa;Password=L3dger#Lab2024!;Encrypt=True;TrustServerCertificate=True' \
  ledger:base-noble-chiseled-extra
sleep 15

# Inspect from outside using a debug sidecar that shares the target's namespaces.
docker run --rm -it --pid=container:chisel-test --network=container:chisel-test \
  --cap-add SYS_PTRACE nicolaka/netshoot ps -ef

docker run --rm --network=container:chisel-test nicolaka/netshoot \
  curl -s http://localhost:8080/healthz
```

**Verify.** You can see the target's process table and reach its localhost port from a throwaway container that has all the tooling. This is the production debugging pattern: the debug tools live in a separate, temporary container, not permanently in your production image. In Kubernetes the equivalent is `kubectl debug --image=nicolaka/netshoot --target=<container>`.

```bash
docker rm -f chisel-test >/dev/null
```

### Do this — adopt chiselled as the runtime base

```bash
sed -i.bak 's|FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-noble AS runtime|FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-noble-chiseled-extra AS runtime|' Dockerfile
sed -i.bak '/RUN mkdir -p \/app\/App_Data/d' Dockerfile
rm -f Dockerfile.bak
grep -n 'FROM' Dockerfile
```

**Verify.** The runtime stage now uses `noble-chiseled-extra`, and the `RUN mkdir` line is gone.

```bash
docker compose down
docker volume rm ledger_app-data 2>/dev/null
docker compose up -d --build
sleep 45
curl -s http://localhost:8080/healthz ; echo
docker images ledger:compose --format '{{.Size}}'
```

**Verify.** Healthy, and materially smaller than where you started. Commit this.

```bash
git add -A && git commit -q -m "Multi-stage build on chiselled runtime base"
```

### Digest pinning the base image

Tags are mutable. `aspnet:8.0-noble-chiseled-extra` points at a different manifest every patch Tuesday, which means your "reproducible" build is not.

**Do this.**

```bash
docker buildx imagetools inspect mcr.microsoft.com/dotnet/aspnet:8.0-noble-chiseled-extra \
  --format '{{.Manifest.Digest}}'
```

**Verify.** You get a `sha256:...` value. In a regulated environment, pin it:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0-noble-chiseled-extra@sha256:<digest>
```

Then bump the digest deliberately through a pull request, with the base-image scan result attached. Renovate or Dependabot automates the bump. Pinning without an automated bump process is worse than not pinning — you end up frozen on a base image with a year of unpatched CVEs.

---

## Lab 2.3 — Runtime Hardening: Non-Root, Capabilities, Read-Only Filesystem

### Why and what

A minimal image reduces what an attacker finds. Runtime hardening reduces what they can do with it. Four controls, in order of value:

1. **Run as non-root.** Root in a container is root on the host if any isolation boundary fails.
2. **Drop all capabilities.** A default container holds around 14 Linux capabilities it almost certainly does not need, including `NET_RAW` (packet crafting) and `CHOWN`.
3. **Read-only root filesystem.** An attacker who cannot write cannot persist a webshell or drop a binary.
4. **`no-new-privileges`.** Blocks privilege escalation through setuid binaries.

### Do this — confirm you are no longer root

```bash
docker compose exec app id 2>/dev/null || \
docker run --rm --entrypoint /usr/bin/id ledger:compose 2>/dev/null || \
docker inspect ledger:compose --format 'Image USER = {{.Config.User}}'
```

**Verify.** The configured user is `1654`. Chiselled images have no `id` binary, so `docker inspect` is the reliable check.

### Do this — see the default capability set, then drop it

```bash
docker run --rm alpine:3.20 sh -c 'grep CapEff /proc/self/status'
docker run --rm --cap-drop=ALL alpine:3.20 sh -c 'grep CapEff /proc/self/status'
```

**Verify.** The first prints a non-zero bitmask. The second prints `CapEff: 0000000000000000`. Zero capabilities.

### Do this — prove the capability drop is enforced

```bash
docker run --rm alpine:3.20 ping -c1 -W2 127.0.0.1 >/dev/null && echo "ping works with default caps"
docker run --rm --cap-drop=ALL alpine:3.20 ping -c1 -W2 127.0.0.1 2>&1 | head -1
```

**Verify.** The second fails on a permission error — `ping` needs `NET_RAW`, which you removed. Your web application does not need it either.

### Do this — apply full hardening to the app service

```bash
cat > docker-compose.override.yml <<'EOF'
# Hardening overlay. Compose merges this on top of docker-compose.yml automatically.
services:
  app:
    read_only: true
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    user: "1654:1654"
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m
    volumes:
      - app-data:/app/App_Data
      - dp-keys:/home/app/.aspnet/DataProtection-Keys
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512m

  proxy:
    read_only: true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETUID
      - SETGID
    security_opt:
      - no-new-privileges:true
    tmpfs:
      - /var/cache/nginx:rw,noexec,nosuid,size=32m
      - /var/run:rw,noexec,nosuid,size=8m

volumes:
  dp-keys:
EOF
```

The nginx service keeps three capabilities because its master process starts as root, binds port 80, and then drops to the `nginx` user — that sequence needs `SETUID`, `SETGID`, and `CHOWN`. This is a real example of least privilege: not "drop everything", but "drop everything, then add back exactly what is provably required".

### Do this — apply and verify

```bash
docker compose down
docker compose up -d
sleep 45
docker compose ps --format 'table {{.Service}}\t{{.Status}}'
curl -s http://localhost:8080/healthz ; echo
```

**Verify.** All services healthy and the endpoint responds under full hardening.

### Do this — prove the root filesystem is actually read-only

```bash
docker compose exec proxy sh -c 'touch /etc/nginx/pwned' 2>&1 | tail -1
docker compose exec proxy sh -c 'touch /tmp/allowed 2>/dev/null || touch /var/run/allowed; echo "tmpfs write OK"'
```

**Verify.** The write to `/etc/nginx` fails with a read-only filesystem error. The write to the tmpfs succeeds. An attacker who lands in this container cannot modify the nginx configuration or drop a binary anywhere except a 32 MB in-memory scratch area that is mounted `noexec`.

### Do this — verify the application can still write where it must

```bash
curl -s -c /tmp/ck http://localhost:8080/ >/dev/null
TOKEN=$(curl -s -b /tmp/ck -c /tmp/ck http://localhost:8080/ | grep -o 'name="__RequestVerificationToken"[^>]*value="[^"]*"' | sed 's/.*value="//;s/"//')
curl -s -b /tmp/ck -o /dev/null -w 'create status: %{http_code}\n' -X POST http://localhost:8080/Home/Create \
  --data-urlencode "customer=Nepal Clearing House" --data-urlencode "amount=125000.00" \
  --data-urlencode "__RequestVerificationToken=$TOKEN"
curl -s http://localhost:8080/Home/Audit
```

**Verify.** The create returns `302` (redirect after post) and the audit log contains the entry. The named volume at `/app/App_Data` is writable even though the root filesystem is not — volume and tmpfs mounts are independent of the `read_only` flag.

### The Data Protection lesson

**Do this.**

```bash
docker compose logs app | grep -i -E 'dataprotection|keys' | head -5
```

**Verify and internalize.** Without the `dp-keys` volume you added, ASP.NET Core cannot persist its Data Protection key ring on a read-only filesystem. It falls back to ephemeral in-memory keys and logs a warning. The consequence in production: antiforgery tokens, auth cookies, and any `IDataProtector` payload become invalid on every restart, and break entirely across multiple replicas. Every user gets logged out on each deployment.

In production, do not solve this with a local volume. Persist the key ring to a shared store and encrypt it with a managed key:

```csharp
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(blobUri, credential)
    .ProtectKeysWithAzureKeyVault(keyIdentifier, credential);
```

This is the class of failure that only appears under hardening plus horizontal scaling, which is exactly why you harden in the lab rather than in production.

### Do this — commit

```bash
git add -A && git commit -q -m "Runtime hardening: non-root, cap-drop, read-only rootfs, resource limits"
```

---

## Lab 2.4 — Vulnerability Scanning with Trivy

### Why and what

A scanner reads your image's package inventory — OS packages plus language dependencies — and matches it against public vulnerability databases. It answers one question: *what known, published flaws am I shipping?*

Two design decisions separate a useful scan gate from theatre:

- **Gate on severity plus fixability.** A CRITICAL with no vendor fix available cannot be actioned by your team today. Blocking on it stops the pipeline without improving security, and teams respond by disabling the gate. Gate on `--ignore-unfixed`, and track unfixed criticals through a separate reporting path.
- **Scan the artifact you will ship, before you ship it.** Scanning after push means the vulnerable image already exists in the registry where something can pull it.

### Do this — install Trivy and prime the database

```bash
trivy --version || {
  curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
}
trivy image --download-db-only
```

**Verify.** `trivy --version` prints a version and the DB download completes.

### Do this — scan the naive image and the hardened image

```bash
echo "=== v3-ignore (SDK base, root) ==="
trivy image --severity HIGH,CRITICAL --quiet ledger:v3-ignore | tail -20

echo "=== v4 hardened (chiselled base, non-root) ==="
trivy image --severity HIGH,CRITICAL --quiet ledger:compose | tail -20
```

**Verify.** Compare the finding counts. The chiselled image typically reports a fraction of the vulnerabilities, and often zero HIGH or CRITICAL. You did not patch anything — you removed the packages that carried the flaws. That is the argument for minimal base images in one screen.

### Do this — record the delta

```bash
for IMG in ledger:v3-ignore ledger:compose; do
  COUNT=$(trivy image --severity HIGH,CRITICAL --format json --quiet "$IMG" \
    | jq '[.Results[]?.Vulnerabilities // [] | length] | add // 0')
  echo "$IMG : $COUNT HIGH/CRITICAL"
done
```

### Do this — practise the gate behaviour

```bash
# Reporting run: never fails, produces the artifact.
trivy image --severity HIGH,CRITICAL --format json --output trivy-report.json --quiet ledger:compose
echo "report exit=$?"

# Gate run: fails the build on fixable HIGH or CRITICAL.
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 --quiet ledger:compose
echo "gate exit=$?"
```

**Verify.** The reporting run exits `0`. The gate run exits `0` if clean or `1` if fixable findings exist. Your pipeline runs both: the report always, the gate as the pass/fail decision.

### Do this — prove the gate actually blocks something

```bash
docker pull nginx:1.14 >/dev/null 2>&1
trivy image --severity CRITICAL --ignore-unfixed --exit-code 1 --quiet nginx:1.14 >/dev/null 2>&1
echo "deliberately vulnerable image gate exit=$?"
docker rmi nginx:1.14 >/dev/null 2>&1
```

**Verify.** Exit code `1`. A gate you have never seen fail is a gate you cannot trust.

### Do this — scan the Dockerfile itself for misconfiguration

```bash
trivy config Dockerfile
trivy config docker-compose.yml
```

**Verify.** Trivy checks against built-in policies — running as root, missing `USER`, use of `:latest`, unpinned `apt` installs. Your hardened Dockerfile should report clean or near-clean. Fix anything it flags.

### Do this — scan the repository for leaked secrets

```bash
trivy fs --scanners secret --quiet .
git log --all -p | trivy fs --scanners secret - 2>/dev/null || echo "(pipe scan optional)"
```

**Verify.** No findings. If Trivy reports the `MSSQL_SA_PASSWORD` in your `.env`, that is correct behaviour — the file is gitignored locally, but the finding reminds you that this credential must move to Key Vault before any real deployment.

### Do this — generate an SBOM

```bash
trivy image --format cyclonedx --output ledger-sbom.cdx.json --quiet ledger:compose
jq '{format: .bomFormat, spec: .specVersion, components: (.components | length)}' ledger-sbom.cdx.json
```

**Verify.** You get a CycloneDX document listing every component. Publish this as a build artifact and attach it to the image in ACR. When the next Log4Shell-class event happens, the question "which of our images contain package X at version Y" is answered by querying stored SBOMs in minutes instead of rebuilding everything to find out.

### Do this — manage accepted risk with expiry

```bash
cat > .trivyignore.yaml <<'EOF'
vulnerabilities:
  - id: CVE-2024-00000
    paths:
      - "usr/lib/example.so"
    statement: >-
      Not exploitable. The vulnerable code path requires the SMTP client,
      which this application does not use. Reviewed by AppSec 2026-08-13.
    expiredAt: 2026-11-13
EOF
```

**Verify.** Every suppression carries a justification and an expiry date. An indefinite suppression is a permanent hole nobody revisits. An expiring one forces a re-review in three months.

### Air-gapped and rate-limited environments

Trivy's database is served from a public registry that applies rate limits. On shared corporate egress, builds start failing intermittently at exactly the wrong time.

**Do this — mirror the database into your own registry (run after Lab 2.5 creates the ACR).**

```bash
# Copy the Trivy DB into ACR, then point builds at the mirror.
oras copy ghcr.io/aquasecurity/trivy-db:2 "${ACR_NAME}.azurecr.io/mirror/trivy-db:2"
export TRIVY_DB_REPOSITORY="${ACR_NAME}.azurecr.io/mirror/trivy-db:2"
export TRIVY_CACHE_DIR="$PWD/.trivycache"
```

Schedule the mirror refresh daily. In CI, cache `TRIVY_CACHE_DIR` between runs.

### Do this — commit

```bash
printf 'trivy-report.json\nledger-sbom.cdx.json\n.trivycache/\n' >> .gitignore
git add -A && git commit -q -m "Add Trivy scanning, SBOM generation, and expiring suppression policy"
```

---

## Lab 2.5 — Provision Azure Container Registry with Enterprise Controls

### Why and what

ACR is a managed OCI registry with Azure identity, networking, and policy attached. The SKU you pick determines which enterprise controls you can even turn on, so pick it before you have 200 repositories in the wrong one.

| Capability                       | Basic | Standard | Premium |
| -------------------------------- | ----- | -------- | ------- |
| Included storage                 | 10 GB | 100 GB   | 500 GB  |
| Webhooks                         | 2     | 10       | 500     |
| Geo-replication                  | no    | no       | **yes** |
| Private Link / private endpoints | no    | no       | **yes** |
| Customer-managed keys            | no    | no       | **yes** |
| Repository-scoped tokens         | no    | no       | **yes** |
| Availability zones               | no    | no       | **yes** |
| Image quarantine (preview)       | no    | no       | **yes** |

The practical rule for any regulated or multi-region workload: **Premium**. Private endpoints and repository-scoped tokens are the two features you will need and cannot retrofit without an SKU change and a network redesign.

### Do this — sign in and set variables

```bash
az login
az account show --output table

export LOCATION="centralindia"
export RG="rg-ledger-lab"
export ACR_NAME="acrledger$(openssl rand -hex 3)"
export SUB_ID=$(az account show --query id -o tsv)

echo "ACR_NAME=${ACR_NAME}" | tee .lab-env
echo "RG=${RG}" >> .lab-env
echo "SUB_ID=${SUB_ID}" >> .lab-env
```

Registry names must be globally unique, alphanumeric, and lowercase.

### Do this — create the registry with secure defaults

```bash
az group create --name "$RG" --location "$LOCATION" --output none

az acr create \
  --resource-group "$RG" \
  --name "$ACR_NAME" \
  --sku Premium \
  --location "$LOCATION" \
  --admin-enabled false \
  --allow-anonymous-pull false \
  --public-network-enabled true \
  --output table
```

`--admin-enabled false` and `--allow-anonymous-pull false` are the two settings most often left wrong. Set them at creation so nobody has to remember to turn them off.

**Verify.**

```bash
az acr show --name "$ACR_NAME" --query \
  '{name:name, sku:sku.name, adminEnabled:adminUserEnabled, anonymousPull:anonymousPullEnabled, publicNetwork:publicNetworkAccess}' \
  --output table
```

**Verify.** `adminEnabled` is `false`, `anonymousPull` is `false`.

### Do this — configure registry policies

```bash
# Purge untagged manifests after 7 days. Untagged layers accrue storage cost silently.
az acr config retention update --registry "$ACR_NAME" \
  --status enabled --days 7 --type UntaggedManifests --output none

# Soft delete gives you a recovery window for accidental deletions.
az acr config soft-delete update --registry "$ACR_NAME" \
  --status enabled --days 7 --output none

az acr config retention show --registry "$ACR_NAME" --output table
az acr config soft-delete show --registry "$ACR_NAME" --output table
```

**Verify.** Both policies report enabled.

### Do this — restrict network access (Premium)

```bash
MY_IP=$(curl -s https://ifconfig.me)

az acr update --name "$ACR_NAME" --default-action Deny --output none
az acr network-rule add --name "$ACR_NAME" --ip-address "$MY_IP" --output none
az acr network-rule list --name "$ACR_NAME" --output table
```

**Verify.** Your IP is allowed and the default action is `Deny`. In production you replace the IP allowlist entirely with a private endpoint and `--public-network-enabled false`, so registry traffic never leaves the virtual network. The command for that, for reference:

```bash
# Production pattern. Do not run in the lab unless you have a VNet and a private DNS zone.
# az acr update --name "$ACR_NAME" --public-network-enabled false
# az network private-endpoint create --name pe-acr --resource-group "$RG" \
#   --vnet-name vnet-core --subnet snet-privatelink \
#   --private-connection-resource-id $(az acr show -n "$ACR_NAME" --query id -o tsv) \
#   --group-id registry --connection-name acr-conn
```

---

## Lab 2.6 — ACR Authentication: From Worst to Best

### Why and what

There are four ways to authenticate to ACR. They are not equivalent, and the convenient one is the dangerous one.

| Method                                          | Credential lifetime       | Scoping                        | Audit attribution             | Verdict                    |
| ----------------------------------------------- | ------------------------- | ------------------------------ | ----------------------------- | -------------------------- |
| Admin user                                      | Permanent until rotated   | Full registry, push and pull   | **None — one shared account** | Never. Disable it.         |
| Service principal + secret                      | 1 to 2 years typically    | RBAC role                      | Per-principal                 | Legacy fallback only       |
| Repository-scoped token                         | Configurable              | **Per repository, per action** | Per token                     | Partners, external systems |
| Managed identity / workload identity federation | **Minutes, auto-rotated** | RBAC role                      | Per identity                  | **Default choice**         |

### Do this — demonstrate why the admin user is unacceptable

```bash
az acr update --name "$ACR_NAME" --admin-enabled true --output none
az acr credential show --name "$ACR_NAME" --query '{user:username, pw1:passwords[0].value}' -o table
```

**Verify and internalize.** One username, one password, full push and delete rights across every repository. Every pipeline, every developer, and every partner that needs access shares it. When someone leaves, you rotate one credential and break everything simultaneously. Registry logs will attribute every action to the same identity, so a forensic investigation cannot tell you who pushed the malicious image.

### Do this — turn it off and keep it off

```bash
az acr update --name "$ACR_NAME" --admin-enabled false --output none
az acr show --name "$ACR_NAME" --query adminUserEnabled -o tsv
```

**Verify.** `false`. Enforce this organization-wide with Azure Policy so it cannot be re-enabled:

```bash
az policy assignment create \
  --name "deny-acr-admin-user" \
  --scope "/subscriptions/${SUB_ID}/resourceGroups/${RG}" \
  --policy "dc921057-6b28-4fbe-9b83-f7bec05db6c2" \
  --output none 2>/dev/null && echo "Policy assigned: Container registries should have local admin account disabled" \
  || echo "Assign the built-in policy 'Container registries should have local admin account disabled' via the portal if the CLI call is blocked."
```

### Do this — authenticate as yourself with Entra ID

```bash
az acr login --name "$ACR_NAME"
cat ~/.docker/config.json | jq '.auths | keys'
```

**Verify.** Your registry appears in the Docker credential store. `az acr login` exchanges your Entra ID token for a short-lived registry refresh token. No password was stored anywhere.

### Do this — assign least-privilege RBAC

```bash
ACR_ID=$(az acr show --name "$ACR_NAME" --query id -o tsv)

# A CI identity that publishes images.
az ad sp create-for-rbac --name "sp-ledger-ci" --skip-assignment --output json > /tmp/sp-ci.json
CI_APP_ID=$(jq -r .appId /tmp/sp-ci.json)
az role assignment create --assignee "$CI_APP_ID" --role AcrPush --scope "$ACR_ID" --output none

# A runtime identity that only consumes images.
az identity create --name "id-ledger-runtime" --resource-group "$RG" --output none
RUNTIME_PRINCIPAL=$(az identity show --name "id-ledger-runtime" --resource-group "$RG" --query principalId -o tsv)
az role assignment create --assignee-object-id "$RUNTIME_PRINCIPAL" --assignee-principal-type ServicePrincipal \
  --role AcrPull --scope "$ACR_ID" --output none

az role assignment list --scope "$ACR_ID" --query '[].{principal:principalName, role:roleDefinitionName}' -o table
```

**Verify.** You see `AcrPush` for CI and `AcrPull` for runtime. Note what is absent: neither has `AcrDelete`, and neither is `Contributor`. `Contributor` on a registry grants credential retrieval and configuration changes and is the most common over-grant in real subscriptions.

The role you need, by task:

| Task                                             | Role                                           |
| ------------------------------------------------ | ---------------------------------------------- |
| Pull images (AKS nodes, App Service, developers) | `AcrPull`                                      |
| Push images (CI pipelines)                       | `AcrPush`                                      |
| Delete tags or manifests (cleanup automation)    | `AcrDelete`                                    |
| Sign images (Notation identity)                  | `AcrImageSigner`                               |
| Manage the registry resource itself              | `Owner` / `Contributor` — humans only, via PIM |

### Do this — bind a runtime identity to the registry with no secret at all

```bash
# The AKS pattern, for reference. Attaching the cluster grants AcrPull to its kubelet identity.
# az aks update --name aks-prod --resource-group "$RG" --attach-acr "$ACR_NAME"

# The Container Apps / App Service pattern:
# az webapp config container set --name app-ledger --resource-group "$RG" \
#   --docker-registry-server-url "https://${ACR_NAME}.azurecr.io" \
#   --docker-custom-image-name "${ACR_NAME}.azurecr.io/ledger:1.0.0"
# az webapp identity assign --name app-ledger --resource-group "$RG"
```

The property that matters: **no credential is stored in the workload's configuration.** The platform obtains a token from Entra ID at pull time. There is nothing to leak and nothing to rotate.

### Do this — create a repository-scoped token for a partner

```bash
az acr scope-map create --registry "$ACR_NAME" --name partner-readonly \
  --repository ledger content/read metadata/read \
  --description "Read-only pull access to the ledger repository for external partner" \
  --output none

az acr token create --registry "$ACR_NAME" --name partner-token \
  --scope-map partner-readonly --output json > /tmp/partner-token.json

jq '{name: .name, status: .status, scopeMap: .scopeMapId}' /tmp/partner-token.json
```

**Verify.** The token can read exactly one repository. It cannot push, cannot delete, cannot enumerate other repositories, and can be revoked in isolation without affecting anything else.

```bash
az acr scope-map show --registry "$ACR_NAME" --name partner-readonly --query actions -o tsv
```

### Do this — clean up the token

```bash
az acr token delete --registry "$ACR_NAME" --name partner-token --yes --output none
az acr scope-map delete --registry "$ACR_NAME" --name partner-readonly --yes --output none
```

---

## Lab 2.7 — Tagging Strategy and Immutability

### Why and what

`latest` is not a version. It is a mutable pointer that means "whatever was pushed most recently, by anyone, from any branch". Deploying `latest` means you cannot answer the two questions that matter during an incident: *what is running right now*, and *what changed*.

The rules:

1. **Never deploy a mutable tag.** Not `latest`, not `main`, not `staging`.
2. **Tag with something traceable to a commit.** `1.4.2-a3f9c1d` maps to semantic version and Git SHA.
3. **Make release tags immutable at the registry.** Prevent overwrite in the registry, not just by convention.
4. **Deploy by digest.** The tag is for humans. The digest is for machines.

### Do this — build with a traceable tag

```bash
source .lab-env
GIT_SHA=$(git rev-parse --short HEAD)
VERSION="1.0.0"
IMAGE_TAG="${VERSION}-${GIT_SHA}"
FQIN="${ACR_NAME}.azurecr.io/ledger:${IMAGE_TAG}"

docker build -t "$FQIN" \
  --build-arg GIT_SHA="$GIT_SHA" \
  --build-arg BUILD_VERSION="$VERSION" \
  --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  .

echo "FQIN=$FQIN" | tee -a .lab-env
```

**Verify.**

```bash
docker inspect "$FQIN" --format '{{json .Config.Labels}}' | jq
```

The OCI labels carry the revision, version, and build date into the artifact. Six months from now, `docker inspect` on a running production image tells you which commit produced it.

### Do this — scan before push, then push

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 --quiet "$FQIN"
echo "gate exit=$?"

az acr login --name "$ACR_NAME"
docker push "$FQIN"
```

**Verify.**

```bash
az acr repository show-tags --name "$ACR_NAME" --repository ledger --output table
```

Your tag is listed. Note the order of operations: build, scan, gate, **then** push. The vulnerable image never reaches the registry.

### Do this — capture and use the digest

```bash
DIGEST=$(az acr repository show --name "$ACR_NAME" --image "ledger:${IMAGE_TAG}" --query digest -o tsv)
PINNED="${ACR_NAME}.azurecr.io/ledger@${DIGEST}"
echo "DIGEST=$DIGEST" | tee -a .lab-env
echo "Deploy this, not the tag: $PINNED"

docker pull "$PINNED"
docker buildx imagetools inspect "$PINNED" --format '{{.Manifest.Digest}}'
```

**Verify.** The digest you pulled matches the digest you requested. A digest reference is cryptographically bound to exact content — it cannot be repointed, overwritten, or spoofed.

### Do this — prove tags are mutable and digests are not

```bash
# Build a deliberately different image and push it over the same tag.
docker build -t "$FQIN" --build-arg BUILD_VERSION="1.0.0-TAMPERED" .
docker push "$FQIN" >/dev/null 2>&1

NEW_DIGEST=$(az acr repository show --name "$ACR_NAME" --image "ledger:${IMAGE_TAG}" --query digest -o tsv)
echo "original: $DIGEST"
echo "after re-push: $NEW_DIGEST"
[ "$DIGEST" = "$NEW_DIGEST" ] && echo "SAME" || echo "TAG NOW POINTS SOMEWHERE ELSE"
```

**Verify.** The tag now resolves to a different digest. If a deployment manifest referenced that tag, the next pod restart would silently run different code. This is the supply chain attack in its simplest form, and digest pinning is the complete defence.

### Do this — lock the tag so this cannot happen

```bash
az acr repository update --name "$ACR_NAME" \
  --image "ledger:${IMAGE_TAG}" \
  --write-enabled false --output none

docker push "$FQIN" 2>&1 | tail -2
```

**Verify.** The push is rejected. The tag is now immutable. Apply this automatically in the pipeline immediately after pushing a release tag.

For a blanket policy across a repository:

```bash
az acr repository update --name "$ACR_NAME" --repository ledger --write-enabled false --output none
# Re-enable when you next need to publish:
az acr repository update --name "$ACR_NAME" --repository ledger --write-enabled true --output none
```

### Recommended tag scheme

| Tag pattern      | Mutable    | Example                | Purpose                                                                        |
| ---------------- | ---------- | ---------------------- | ------------------------------------------------------------------------------ |
| `<semver>-<sha>` | no, locked | `1.4.2-a3f9c1d`        | The release artifact. Deploy by its digest.                                    |
| `<semver>`       | no, locked | `1.4.2`                | Human-readable release alias                                                   |
| `<branch>-<sha>` | yes        | `feature-auth-9d2b1f0` | Ephemeral, subject to retention purge                                          |
| `latest`         | yes        | —                      | Local development convenience only. Never referenced by a deployment manifest. |

### Do this — the quarantine-and-promote pattern

Where policy forbids an unscanned image existing in the production registry at all, split the registry into stages and promote server-side:

```bash
# Conceptual. Requires a second registry in the lab to run end to end.
# 1. Pipeline pushes to the staging registry only.
# 2. Scan the pushed artifact by digest.
# 3. On pass, promote without rebuilding — az acr import copies the exact manifest.
#
# az acr import --name "$PROD_ACR" \
#   --source "${STAGING_ACR}.azurecr.io/ledger@${DIGEST}" \
#   --image "ledger:${IMAGE_TAG}"
```

`az acr import` copies the manifest and layers server-side. The bytes that passed the scan are bit-for-bit the bytes that land in production, because a rebuild between scanning and promoting would produce a different artifact than the one you approved.

---

## Lab 2.8 — Image Signing: Content Trust and What Replaced It

### Why and what

Scanning tells you an image is not known to be vulnerable. Signing tells you it is the image *you* built and nobody swapped it. Those are different guarantees and you need both.

ACR's original mechanism was Docker Content Trust, built on Notary v1. <cite index="1-1,4-1">Microsoft has retired it: deprecation began on 31 March 2025, it can no longer be newly enabled on registries after 31 May 2026, and it is removed entirely on 31 March 2028.</cite> <cite index="2-1">The replacement is the Notary Project's Notation tool with Azure Key Vault for key management, which stores OCI-standard signatures alongside the image in the registry and integrates with Azure DevOps pipelines, GitHub workflows, and AKS deployment verification.</cite>

The practical consequence for this course: **do not teach or configure Docker Content Trust.** If a client environment still has `DOCKER_CONTENT_TRUST=1` set, that is a migration item.

**Do this — check whether DCT is set anywhere in your environment.**

```bash
env | grep -i DOCKER_CONTENT_TRUST || echo "DCT not enabled — correct"
az acr config content-trust show --registry "$ACR_NAME" 2>/dev/null || echo "content trust not enabled on this registry — correct"
```

### Do this — sign with Notation and Azure Key Vault

```bash
# Install Notation and the Azure Key Vault plugin.
curl -sSL -o /tmp/notation.tar.gz \
  https://github.com/notaryproject/notation/releases/download/v1.2.0/notation_1.2.0_linux_amd64.tar.gz
tar -xzf /tmp/notation.tar.gz -C /tmp && sudo mv /tmp/notation /usr/local/bin/
notation version

# Create a Key Vault and a self-signed signing certificate.
export AKV_NAME="kv-ledger-$(openssl rand -hex 3)"
az keyvault create --name "$AKV_NAME" --resource-group "$RG" --location "$LOCATION" \
  --enable-rbac-authorization true --output none

MY_OID=$(az ad signed-in-user show --query id -o tsv)
az role assignment create --assignee "$MY_OID" --role "Key Vault Certificates Officer" \
  --scope "$(az keyvault show -n "$AKV_NAME" --query id -o tsv)" --output none
az role assignment create --assignee "$MY_OID" --role "Key Vault Crypto User" \
  --scope "$(az keyvault show -n "$AKV_NAME" --query id -o tsv)" --output none

cat > /tmp/cert-policy.json <<'POLICY'
{
  "issuerParameters": { "certificateTransparency": null, "name": "Self" },
  "x509CertificateProperties": {
    "ekus": ["1.3.6.1.5.5.7.3.3"],
    "keyUsage": ["digitalSignature"],
    "subject": "CN=Contoso Engineering,O=Contoso,C=NP",
    "validityInMonths": 12
  }
}
POLICY

sleep 20
az keyvault certificate create --vault-name "$AKV_NAME" --name ledger-signing \
  --policy @/tmp/cert-policy.json --output none
```

The EKU `1.3.6.1.5.5.7.3.3` is Code Signing. Notation requires it.

### Do this — register the key and sign the image by digest

```bash
KEY_ID=$(az keyvault certificate show --vault-name "$AKV_NAME" --name ledger-signing --query sid -o tsv)

notation key add --plugin azure-kv --id "$KEY_ID" --plugin-config self_signed=true ledger-key --default
notation key ls

source .lab-env
notation sign "${ACR_NAME}.azurecr.io/ledger@${DIGEST}" --signature-format cose
```

**Verify.**

```bash
notation ls "${ACR_NAME}.azurecr.io/ledger@${DIGEST}"
az acr manifest list-referrers --name "$ACR_NAME" --image "ledger@${DIGEST}" --output table
```

The signature is stored in the registry as an OCI referrer artifact attached to the image manifest. It travels with the image.

### Do this — define a trust policy and verify

```bash
az keyvault certificate download --vault-name "$AKV_NAME" --name ledger-signing \
  --file /tmp/ledger-signing.crt --encoding PEM
notation cert add --type ca --store contoso-trust /tmp/ledger-signing.crt

cat > /tmp/trustpolicy.json <<POLICY
{
  "version": "1.0",
  "trustPolicies": [
    {
      "name": "ledger-production",
      "registryScopes": [ "${ACR_NAME}.azurecr.io/ledger" ],
      "signatureVerification": { "level": "strict" },
      "trustStores": [ "ca:contoso-trust" ],
      "trustedIdentities": [ "x509.subject: CN=Contoso Engineering,O=Contoso,C=NP" ]
    }
  ]
}
POLICY

notation policy import /tmp/trustpolicy.json
notation policy show
notation verify "${ACR_NAME}.azurecr.io/ledger@${DIGEST}"
```

**Verify.** Verification succeeds and reports the signature as trusted.

### Do this — prove verification rejects an unsigned image

```bash
docker tag ledger:v3-ignore "${ACR_NAME}.azurecr.io/ledger:unsigned-test"
docker push "${ACR_NAME}.azurecr.io/ledger:unsigned-test" >/dev/null
UNSIGNED_DIGEST=$(az acr repository show --name "$ACR_NAME" --image "ledger:unsigned-test" --query digest -o tsv)
notation verify "${ACR_NAME}.azurecr.io/ledger@${UNSIGNED_DIGEST}" 2>&1 | tail -2
```

**Verify.** Verification fails with no signature found. In Kubernetes, the Ratify admission controller runs exactly this check and refuses to schedule the pod. That is how signing becomes enforcement instead of documentation.

```bash
az acr repository delete --name "$ACR_NAME" --image "ledger:unsigned-test" --yes --output none
```

---

## Lab 2.9 — Defender for Containers on the Registry

### Why and what

Trivy in your pipeline scans at build time against the database available that day. It cannot tell you about a CVE published next Tuesday affecting an image you shipped last month. Registry-side scanning closes that gap by continuously re-assessing what is already stored.

<cite index="10-1">Defender for Cloud performs agentless vulnerability assessment on images in Azure Container Registry when Registry access is enabled under either the Defender CSPM or the Defender for Containers plan. New or imported images are scanned within a few hours, and continuous rescanning keeps findings current as new vulnerabilities are published.</cite> <cite index="10-1">Coverage focuses on images pushed or pulled within the last 30 days and images currently running on monitored Kubernetes clusters.</cite>

The division of labour is the point:

| Control                 | Runs       | Answers                                                  | Enforces                                                |
| ----------------------- | ---------- | -------------------------------------------------------- | ------------------------------------------------------- |
| Trivy in the pipeline   | Per build  | Is this new artifact shippable today?                    | **Yes — blocks the push**                               |
| Defender for Containers | Continuous | Which stored and running images became vulnerable since? | Reporting, plus gated deployment for supported clusters |

Use both. Neither substitutes for the other.

### Do this — enable the plan

```bash
az provider register --namespace Microsoft.Security --output none
az security pricing create --name Containers --tier Standard --output none 2>/dev/null \
  || echo "Enable Defender for Containers via Portal: Defender for Cloud -> Environment settings -> subscription -> Defender plans"

az security pricing show --name Containers --query '{plan:name, tier:pricingTier}' -o table
```

**Verify.** Tier reports `Standard`. Then confirm the extension in the portal: Defender for Cloud → Environment settings → your subscription → Containers **Settings** → *Agentless container vulnerability assessment* must be **On**.

> **Cost warning for the classroom.** Defender for Containers is billed per vCore and per registry image scanned. Enable it for the demonstration, then disable it in the cleanup step. Confirm the cost model with the client before enabling it in their subscription.

### Do this — push a deliberately vulnerable image and observe detection

```bash
az acr import --name "$ACR_NAME" --source docker.io/library/nginx:1.14 --image demo-vulnerable:1.14 --output none
az acr repository show-tags --name "$ACR_NAME" --repository demo-vulnerable -o table
```

**Verify.** Findings appear in Defender for Cloud → Recommendations, filtered to resource type *Container Image*, under the recommendation about container images in Azure registries having vulnerability findings resolved. Scanning is asynchronous and takes a few hours — do not wait for it live. Instructors should pre-push this image before the session.

### Do this — query findings programmatically

```bash
az graph query -q "
securityresources
| where type =~ 'microsoft.security/assessments/subassessments'
| where properties.additionalData.assessedResourceType =~ 'AzureContainerRegistryVulnerability'
| extend severity = tostring(properties.status.severity),
         cve      = tostring(properties.id),
         image    = tostring(properties.additionalData.repositoryName)
| summarize count() by severity, image
| order by severity asc
" --output table 2>/dev/null || echo "Install the resource-graph extension: az extension add --name resource-graph"
```

**Verify.** You get a per-image severity breakdown. Wire this into a scheduled job that opens tickets automatically, because a finding nobody sees is not a control.

---

## Lab 2.10 — The Pipeline: Build, Scan, Gate, Publish

### Why and what

Everything so far you ran by hand. A hand-run security control is not a control — it is a habit, and habits lapse under deadline pressure. The pipeline makes the gate structural.

Design requirements for the pipeline you are about to build:

1. **No stored secrets.** Authenticate to Azure with workload identity federation (OIDC). No service principal password anywhere.
2. **Build once.** The artifact that is scanned is byte-identical to the artifact that is pushed.
3. **Gate before push.** A failing scan means nothing reaches the registry.
4. **Publish evidence.** SBOM and scan report as build artifacts, retained for audit.
5. **Emit the digest.** Downstream deployment stages consume the digest, never the tag.

### Do this — create the Azure DevOps pipeline

```bash
mkdir -p .azuredevops
cat > azure-pipelines.yml <<'EOF'
trigger:
  branches:
    include: [ main ]
  paths:
    include: [ src/**, Dockerfile, azure-pipelines.yml ]

pr:
  branches:
    include: [ main ]

variables:
  - name: acrName
    value: 'REPLACE_WITH_YOUR_ACR_NAME'
  - name: repository
    value: 'ledger'
  - name: version
    value: '1.0.$(Build.BuildId)'
  - name: imageTag
    value: '$(version)-$(Build.SourceVersion)'
  - name: TRIVY_CACHE_DIR
    value: '$(Pipeline.Workspace)/.trivycache'

pool:
  vmImage: 'ubuntu-latest'

stages:

# =====================================================================
- stage: Build
  displayName: Build and Security Gate
  jobs:
  - job: BuildScanPush
    displayName: Build, scan, gate, publish
    steps:

    - checkout: self
      fetchDepth: 1

    - task: Cache@2
      displayName: Restore Trivy vulnerability database cache
      inputs:
        key: 'trivy | "$(Agent.OS)" | v1'
        path: '$(TRIVY_CACHE_DIR)'
        restoreKeys: 'trivy | "$(Agent.OS)"'

    - script: |
        set -euo pipefail
        SHORT_SHA=$(git rev-parse --short HEAD)
        echo "##vso[task.setvariable variable=shortSha]$SHORT_SHA"
        echo "##vso[task.setvariable variable=buildDate]$(date -u +%Y-%m-%dT%H:%M:%SZ)"
      displayName: Compute build metadata

    # ---- 1. BUILD ONCE. Load locally; do not push yet. ----
    - script: |
        set -euo pipefail
        docker buildx create --use --name ci-builder --driver docker-container 2>/dev/null || true
        docker buildx build \
          --builder ci-builder \
          --file Dockerfile \
          --tag "$(acrName).azurecr.io/$(repository):$(imageTag)" \
          --build-arg GIT_SHA="$(shortSha)" \
          --build-arg BUILD_VERSION="$(version)" \
          --build-arg BUILD_DATE="$(buildDate)" \
          --cache-from "type=registry,ref=$(acrName).azurecr.io/$(repository):buildcache" \
          --provenance=true \
          --sbom=true \
          --load \
          .
      displayName: Build image (single build, loaded locally)

    # ---- 2. INSTALL SCANNER ----
    - script: |
        set -euo pipefail
        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
          | sh -s -- -b /usr/local/bin v0.53.0
        trivy --version
        mkdir -p "$(TRIVY_CACHE_DIR)"
      displayName: Install Trivy

    # ---- 3. REPORT (never fails) ----
    - script: |
        set -euo pipefail
        IMG="$(acrName).azurecr.io/$(repository):$(imageTag)"

        trivy image --exit-code 0 --format json \
          --output "$(Build.ArtifactStagingDirectory)/trivy-report.json" "$IMG"

        trivy image --exit-code 0 --format template \
          --template "@/usr/local/share/trivy/templates/junit.tpl" \
          --output "$(Build.ArtifactStagingDirectory)/trivy-junit.xml" "$IMG" || true

        trivy image --exit-code 0 --format cyclonedx \
          --output "$(Build.ArtifactStagingDirectory)/sbom.cdx.json" "$IMG"

        trivy image --exit-code 0 --severity HIGH,CRITICAL --format table "$IMG"
      displayName: Scan and publish evidence

    - task: PublishBuildArtifacts@1
      displayName: Publish security evidence
      condition: always()
      inputs:
        pathToPublish: '$(Build.ArtifactStagingDirectory)'
        artifactName: 'security-evidence'

    - task: PublishTestResults@2
      displayName: Surface findings as test results
      condition: always()
      inputs:
        testResultsFormat: 'JUnit'
        testResultsFiles: '$(Build.ArtifactStagingDirectory)/trivy-junit.xml'
        testRunTitle: 'Trivy vulnerability scan'
        failTaskOnFailedTests: false

    # ---- 4. THE GATE. Fails the build; nothing is pushed. ----
    - script: |
        set -euo pipefail
        IMG="$(acrName).azurecr.io/$(repository):$(imageTag)"

        echo "Gating on fixable HIGH and CRITICAL findings..."
        trivy image \
          --severity HIGH,CRITICAL \
          --ignore-unfixed \
          --exit-code 1 \
          --no-progress \
          "$IMG"

        echo "Checking Dockerfile misconfiguration policy..."
        trivy config --severity HIGH,CRITICAL --exit-code 1 Dockerfile

        echo "Checking for leaked secrets..."
        trivy fs --scanners secret --severity HIGH,CRITICAL --exit-code 1 .
      displayName: 'SECURITY GATE: fail on fixable HIGH/CRITICAL'

    # ---- 5. PUSH. Only reached if the gate passed. ----
    - task: AzureCLI@2
      displayName: Push to ACR (workload identity federation, no stored secret)
      condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
      inputs:
        azureSubscription: 'sc-ledger-acr'   # Service connection using workload identity federation
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          set -euo pipefail
          az acr login --name "$(acrName)"

          IMG="$(acrName).azurecr.io/$(repository):$(imageTag)"
          docker push "$IMG"

          # Refresh the layer cache for subsequent builds.
          docker buildx build --builder ci-builder --file Dockerfile \
            --cache-to "type=registry,ref=$(acrName).azurecr.io/$(repository):buildcache,mode=max" \
            --target runtime --output type=cacheonly . || true

          DIGEST=$(az acr repository show --name "$(acrName)" \
            --image "$(repository):$(imageTag)" --query digest -o tsv)
          echo "Published digest: $DIGEST"
          echo "##vso[task.setvariable variable=imageDigest;isOutput=true]$DIGEST"

          # Make the released tag immutable.
          az acr repository update --name "$(acrName)" \
            --image "$(repository):$(imageTag)" --write-enabled false

          echo "$(acrName).azurecr.io/$(repository)@${DIGEST}" \
            > "$(Build.ArtifactStagingDirectory)/image-digest.txt"
      name: push

    - task: PublishBuildArtifacts@1
      displayName: Publish image digest for downstream deployment
      condition: succeeded()
      inputs:
        pathToPublish: '$(Build.ArtifactStagingDirectory)/image-digest.txt'
        artifactName: 'image-reference'
EOF

sed -i.bak "s/REPLACE_WITH_YOUR_ACR_NAME/${ACR_NAME}/" azure-pipelines.yml && rm -f azure-pipelines.yml.bak
grep -n 'acrName' azure-pipelines.yml | head -2
```

### Do this — create the service connection with no secret

In Azure DevOps: **Project settings → Service connections → New → Azure Resource Manager → Workload Identity federation (automatic)**. Scope it to the resource group, name it `sc-ledger-acr`, then grant it only `AcrPush`:

```bash
SC_APP_ID="<application-id-of-the-service-connection>"
az role assignment create --assignee "$SC_APP_ID" --role AcrPush \
  --scope "$(az acr show -n "$ACR_NAME" --query id -o tsv)" --output none
az role assignment list --scope "$(az acr show -n "$ACR_NAME" --query id -o tsv)" \
  --query "[?principalId=='$SC_APP_ID'].roleDefinitionName" -o tsv
```

**Verify.** Only `AcrPush`. Federated credentials expire in minutes and are issued per pipeline run. There is no password in a variable group, nothing to rotate, and nothing to leak in a log.

### Do this — simulate the gate locally before pushing the pipeline

```bash
cat > scripts/ci-gate.sh <<'EOF'
#!/usr/bin/env bash
# Local reproduction of the CI security gate. Run before every push.
set -euo pipefail

IMG="${1:?usage: ci-gate.sh <image-ref>}"

echo "==> Vulnerability gate (fixable HIGH/CRITICAL)"
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 --no-progress "$IMG"

echo "==> Dockerfile misconfiguration gate"
trivy config --severity HIGH,CRITICAL --exit-code 1 Dockerfile

echo "==> Secret gate"
trivy fs --scanners secret --severity HIGH,CRITICAL --exit-code 1 .

echo "==> ALL GATES PASSED"
EOF

mkdir -p scripts && mv scripts/ci-gate.sh scripts/ 2>/dev/null || true
chmod +x scripts/ci-gate.sh
./scripts/ci-gate.sh "$FQIN"
```

**Verify.** `ALL GATES PASSED`. Giving developers the exact gate as a local script is what makes a security gate acceptable rather than resented — they can reproduce and fix a failure in 30 seconds instead of waiting for a pipeline run.

### GitHub Actions equivalent

```bash
mkdir -p .github/workflows
cat > .github/workflows/build-publish.yml <<'EOF'
name: Build, Scan, Publish

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

permissions:
  contents: read
  id-token: write          # Required for OIDC federation to Azure
  security-events: write   # Required to upload SARIF to code scanning

env:
  ACR_NAME: REPLACE_WITH_YOUR_ACR_NAME
  REPOSITORY: ledger

jobs:
  build-scan-publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Compute tag
        id: meta
        run: |
          echo "tag=1.0.${{ github.run_number }}-$(git rev-parse --short HEAD)" >> "$GITHUB_OUTPUT"
          echo "date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> "$GITHUB_OUTPUT"

      - uses: docker/setup-buildx-action@v3

      - name: Build once (load, do not push)
        uses: docker/build-push-action@v6
        with:
          context: .
          load: true
          push: false
          provenance: true
          sbom: true
          tags: ${{ env.ACR_NAME }}.azurecr.io/${{ env.REPOSITORY }}:${{ steps.meta.outputs.tag }}
          build-args: |
            GIT_SHA=${{ github.sha }}
            BUILD_VERSION=1.0.${{ github.run_number }}
            BUILD_DATE=${{ steps.meta.outputs.date }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Generate SBOM
        uses: aquasecurity/trivy-action@0.24.0
        with:
          scan-type: image
          image-ref: ${{ env.ACR_NAME }}.azurecr.io/${{ env.REPOSITORY }}:${{ steps.meta.outputs.tag }}
          format: cyclonedx
          output: sbom.cdx.json

      - name: Scan and report (SARIF, never fails)
        uses: aquasecurity/trivy-action@0.24.0
        with:
          scan-type: image
          image-ref: ${{ env.ACR_NAME }}.azurecr.io/${{ env.REPOSITORY }}:${{ steps.meta.outputs.tag }}
          format: sarif
          output: trivy.sarif
          exit-code: '0'

      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with: { sarif_file: trivy.sarif }

      - name: SECURITY GATE
        uses: aquasecurity/trivy-action@0.24.0
        with:
          scan-type: image
          image-ref: ${{ env.ACR_NAME }}.azurecr.io/${{ env.REPOSITORY }}:${{ steps.meta.outputs.tag }}
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          exit-code: '1'

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: security-evidence
          path: |
            sbom.cdx.json
            trivy.sarif

      - name: Azure login (OIDC, no secret)
        if: github.ref == 'refs/heads/main'
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Push and lock the tag
        if: github.ref == 'refs/heads/main'
        run: |
          set -euo pipefail
          az acr login --name "$ACR_NAME"
          IMG="${ACR_NAME}.azurecr.io/${REPOSITORY}:${{ steps.meta.outputs.tag }}"
          docker push "$IMG"
          DIGEST=$(az acr repository show --name "$ACR_NAME" \
            --image "${REPOSITORY}:${{ steps.meta.outputs.tag }}" --query digest -o tsv)
          echo "PUBLISHED: ${ACR_NAME}.azurecr.io/${REPOSITORY}@${DIGEST}" >> "$GITHUB_STEP_SUMMARY"
          az acr repository update --name "$ACR_NAME" \
            --image "${REPOSITORY}:${{ steps.meta.outputs.tag }}" --write-enabled false
EOF

sed -i.bak "s/REPLACE_WITH_YOUR_ACR_NAME/${ACR_NAME}/" .github/workflows/build-publish.yml
rm -f .github/workflows/build-publish.yml.bak
```

The `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID` values are identifiers, not secrets. Authentication happens through a federated credential bound to this repository and branch, so a fork or another repository cannot obtain a token even if it knows all three values.

### Do this — commit the pipeline

```bash
git add -A && git commit -q -m "Add build-scan-gate-publish pipeline for Azure DevOps and GitHub Actions"
git log --oneline
```

---

## Module 2 Sign-Off Checklist

| #   | Requirement                              | Proof command                                                                  | Expected                   |
| --- | ---------------------------------------- | ------------------------------------------------------------------------------ | -------------------------- |
| 1   | Multi-stage build with no SDK in runtime | `docker run --rm --entrypoint sh ledger:compose -c 'ls /usr/share/dotnet/sdk'` | Fails: no shell, no SDK    |
| 2   | Image is substantially smaller           | `docker images ledger --format '{{.Tag}} {{.Size}}'`                           | v4 well under v1           |
| 3   | Runs as non-root                         | `docker inspect ledger:compose --format '{{.Config.User}}'`                    | `1654`                     |
| 4   | All capabilities dropped                 | `grep cap_drop docker-compose.override.yml`                                    | `ALL`                      |
| 5   | Read-only root filesystem                | `docker compose exec proxy sh -c 'touch /etc/pwned'`                           | Read-only error            |
| 6   | Trivy gate is functional                 | `./scripts/ci-gate.sh $FQIN`                                                   | `ALL GATES PASSED`         |
| 7   | ACR admin user disabled                  | `az acr show -n $ACR_NAME --query adminUserEnabled`                            | `false`                    |
| 8   | Anonymous pull disabled                  | `az acr show -n $ACR_NAME --query anonymousPullEnabled`                        | `false`                    |
| 9   | Least-privilege RBAC only                | `az role assignment list --scope $ACR_ID -o table`                             | `AcrPush` / `AcrPull` only |
| 10  | Image published with traceable tag       | `az acr repository show-tags -n $ACR_NAME --repository ledger -o table`        | `1.0.0-<sha>`              |
| 11  | Release tag is immutable                 | `docker push $FQIN`                                                            | Rejected                   |
| 12  | Deployment reference is a digest         | `cat .lab-env \| grep DIGEST`                                                  | `sha256:...`               |
| 13  | Image is signed and verifiable           | `notation verify "$ACR_NAME.azurecr.io/ledger@$DIGEST"`                        | Verification succeeded     |
| 14  | SBOM produced                            | `jq '.components \| length' ledger-sbom.cdx.json`                              | Non-zero                   |
| 15  | Pipeline gates before push               | Read `azure-pipelines.yml` step order                                          | Gate precedes push task    |

---
