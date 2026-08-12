# Docker Fundamentals and Enterprise Docker with Azure Container Registry

Delivery format: instructor-led, hands-on. Every section follows the same loop. short **Why/What**, then **Do This**, then **Verify**. There is no slide content here. Students type, run, and confirm.

---

## Lab 0 — Environment Preparation

### 0.1 Why this matters

Every failure in this course that is not a teaching moment is an environment failure. Fix the environment first, in a controlled ten minutes, instead of debugging it across six labs.

### 0.2 Required tooling

| Tool                            | Minimum version | Purpose                                                |
| ------------------------------- | --------------- | ------------------------------------------------------ |
| Docker Engine or Docker Desktop | 25.x            | Build and run containers                               |
| Docker Compose plugin           | v2.24           | Multi-service local orchestration                      |
| Azure CLI                       | 2.60            | ACR and identity operations (Module 2)                 |
| Trivy                           | 0.50            | Vulnerability and misconfiguration scanning (Module 2) |
| Git                             | 2.40            | Source control and tag generation                      |
| curl, jq                        | any             | Verification and JSON parsing                          |

**Do this — install the small utilities first.** Almost every verification step in this manual pipes through `jq` or `curl`. A fresh WSL Ubuntu image has neither.

```bash
# Debian / Ubuntu, including WSL
sudo apt-get update
sudo apt-get install -y jq curl wget git ca-certificates unzip dnsutils netcat-openbsd
```

**Do this.** Run the version check block.

```bash
docker version --format '{{.Server.Version}}'
docker compose version --short
az version --output tsv --query '"azure-cli"'
git --version
jq --version
```

**Do this.** Confirm Docker has enough memory. SQL Server needs 2 GB on its own.

```bash
docker info --format 'Total memory: {{.MemTotal}} bytes / CPUs: {{.NCPU}}'
```

**Verify.** `MemTotal` must be at least `4000000000` (4 GB). On Docker Desktop, raise it under Settings → Resources if it is lower. On Linux, this reflects host RAM.

### 0.2b Windows students: WSL 2 setup

The corporate laptops run Windows. Docker on Windows means WSL 2, and there are several differences from a native Linux host that will bite you if nobody names them up front.

**Do this — confirm you are on WSL 2, not WSL 1.**

```powershell
# Run in PowerShell, not inside the distro.
wsl --status
wsl --list --verbose
```

**Verify.** Your distro shows `VERSION 2`. If it shows `1`, convert it — cgroup v2, and therefore every resource-limit lab in Module 1, does not work on WSL 1.

```powershell
wsl --set-version Ubuntu-24.04 2
wsl --set-default-version 2
```

**Do this — pick one of two Docker installation paths, not both.**

*Path A, Docker Desktop with WSL integration (recommended for a corporate classroom).* Install Docker Desktop on Windows, then enable Settings → Resources → WSL Integration → your distro. The `docker` CLI then works inside WSL and talks to Docker Desktop's engine.

*Path B, Docker Engine installed directly inside the distro.* Use this where Docker Desktop licensing is a blocker.

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo service docker start          # or: sudo systemctl start docker, if systemd is enabled
newgrp docker
```

**Verify (either path).**

```bash
docker version --format 'client={{.Client.Version}} server={{.Server.Version}}'
docker run --rm hello-world
```

**Do this — cap WSL memory so SQL Server and Docker do not starve the host.** Create `C:\Users\<you>\.wslconfig`:

```ini
[wsl2]
memory=8GB
processors=4
swap=2GB
```

Then run `wsl --shutdown` from PowerShell and reopen the distro.

**Verify.**

```bash
free -h
docker info --format '{{.MemTotal}}'
```

### WSL differences to keep in mind for the rest of the course

| Topic                    | On WSL                                                                                      | What to do                                                                                                                   |
| ------------------------ | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Docker socket            | Docker Desktop proxies it; the daemon lives outside the distro                              | The `curl --unix-socket` step in Lab 1.2 works on Path B. On Path A it may not — use `docker version` as the fallback proof. |
| File performance         | Files under `/mnt/c/...` are 5 to 20 times slower and break inotify                         | **Keep the lab repository in the Linux filesystem**, at `~/labs/...`, never under `/mnt/c`. Build times depend on this.      |
| Line endings             | Git for Windows may convert to CRLF, which makes shell scripts fail with exit code `126`    | Run `git config --global core.autocrlf input` before cloning anything.                                                       |
| `--network host`         | The "host" is the WSL VM, not Windows. Ports do not appear on the Windows host as expected. | Treat Lab 1.7's host-mode step as read-only on WSL. The instructor demonstrates it on Linux.                                 |
| `localhost` from Windows | Published ports are forwarded from the WSL VM to Windows automatically                      | `http://localhost:8080` in a Windows browser works. If it does not, run `wsl --shutdown` and retry.                          |
| Clock skew after sleep   | The VM clock drifts, which breaks Azure CLI token validation with confusing auth errors     | `sudo hwclock -s` inside the distro, or `wsl --shutdown`.                                                                    |


### 0.3 Create the lab workspace

**Do this.**

```bash
mkdir -p ~/labs/legacy-ledger && cd ~/labs/legacy-ledger
git init -q
mkdir -p src/LegacyLedger/{Models,Data,Controllers,Views/Home,Views/Shared} deploy/nginx
```

**Verify.**

```bash
find . -type d -not -path './.git/*' | sort
```

You should see eight directories under the repository root.

### 0.4 Baseline connectivity smoke test

**Do this.**

```bash
docker run --rm hello-world
```

**Verify.** The output ends with a paragraph confirming the installation appears to be working. If you get a proxy or TLS error here, resolve it now — every later lab pulls from a registry.

---

# MODULE 1 — DOCKER FUNDAMENTALS

**Objective.** Take a legacy ASP.NET monolith that assumes a Windows server, a local SQL instance, and a writable disk, and turn it into a locally running multi-container stack with externalized configuration, persistent state, and a reverse proxy in front.

**Deliverable.** `docker compose up -d` brings up app, database, and proxy. The application is reachable through the proxy, its data survives container deletion, and its configuration comes entirely from the environment.

---

## Lab 1.1 — Containers vs VMs, Proven by Experiment

### Why and what

A VM virtualizes hardware and boots its own kernel. A container is a process on the host kernel, fenced in by namespaces (what it can see) and cgroups (what it can consume). That single difference explains everything downstream: containers start in milliseconds, share the host kernel version, cannot run a Windows binary on a Linux host, and cost nothing when idle.

You are going to prove kernel sharing rather than be told about it.

### Do this — prove the kernel is shared

```bash
uname -r
docker run --rm alpine:3.20 uname -r
docker run --rm ubuntu:24.04 uname -r
```

**Verify.** All three print the *same* kernel release. Two different Linux distributions, one kernel. That is the entire container value proposition in one line.

> On Docker Desktop for macOS or Windows, the printed kernel is the Linux VM's kernel, not your laptop's. The point still holds — all containers share it.

### Do this — prove namespace isolation

```bash
docker run --rm alpine:3.20 ps -ef
docker run --rm alpine:3.20 hostname
docker run --rm alpine:3.20 ip addr show
```

**Verify.** `ps -ef` shows PID 1 as your own command, not the host's init. The hostname is a random hex string. The network interfaces are `lo` and `eth0` only. The container sees a private process table, hostname, and network stack.

### Do this — prove cgroup resource limits

```bash
docker run --rm --memory=64m --cpus=0.5 alpine:3.20 \
  sh -c 'cat /sys/fs/cgroup/memory.max; cat /sys/fs/cgroup/cpu.max'
```

**Verify.** Memory reports `67108864` (64 MiB). CPU reports `50000 100000`, meaning 50 ms of CPU per 100 ms period.

### Do this — prove the limit is enforced

```bash
docker run --rm \
  --memory=64m \
  --shm-size=256m \
  alpine:3.20 \
  sh -c 'dd if=/dev/zero of=/dev/shm/fill bs=1M count=200' ; echo "exit=$?"
```

`--shm-size=256m` matters. Docker's default `/dev/shm` is only 64 MB, so without it `dd` fills the tmpfs and exits with a disk-full error before the memory cgroup is ever breached — you would see exit code `1`, not an OOM kill. Raising the shared-memory size above the memory limit guarantees the cgroup is the constraint that trips first. `/dev/shm` is tmpfs, so its pages count against the container's memory limit.

**Verify.** The container is killed. `exit=137` means SIGKILL from the OOM killer. This is exactly what happens in production when you under-size a memory limit, and `137` is the exit code you will learn to recognize instantly.

### Do this — measure startup cost

```bash
time docker run --rm alpine:3.20 true
```

**Verify.** Real time is well under one second. Compare mentally against a VM boot. This is why containers changed CI, autoscaling, and blue/green deployment.

---

## Lab 1.2 — Docker Architecture: Daemon, CLI, Registry

### Why and what

The `docker` command is a thin HTTP client. It talks to `dockerd`, a long-running daemon that owns images, containers, networks, and volumes. The daemon pulls from and pushes to a registry. When you understand that the CLI is just an API client, you understand why the daemon socket is root-equivalent, why remote contexts work, and why CI agents need a daemon or a daemonless builder.

### Do this — talk to the daemon directly

```bash
docker version
curl -s --unix-socket /var/run/docker.sock http://localhost/version | jq '{ApiVersion, Os, Arch}'
```

On Docker Desktop for macOS, use `~/.docker/run/docker.sock` instead.

**Verify.** The `curl` output matches the server section of `docker version`. You just made the same API call the CLI makes.

### Do this — observe the pull path

```bash
docker pull nginx:1.27-alpine
docker image inspect nginx:1.27-alpine --format '{{.Id}}'
docker image inspect nginx:1.27-alpine --format '{{range .RootFS.Layers}}{{println .}}{{end}}'
```

**Verify.** The image ID is a `sha256:` digest of the image config, and you see several layer digests. An image is a config blob plus an ordered list of content-addressed layers. Nothing more.

### Do this — inspect the registry manifest without pulling

```bash
docker buildx imagetools inspect nginx:1.27-alpine
```

**Verify.** You see a manifest list with multiple platform entries (`linux/amd64`, `linux/arm64`, and others). This is why the same tag works on your Intel laptop and an ARM build agent.

### Security checkpoint

**Do this.**

```bash
ls -l /var/run/docker.sock 2>/dev/null || echo "Docker Desktop socket path differs"
```

**Verify and internalize.** Group `docker` membership is equivalent to root on the host. Anyone who can reach the socket can run `docker run -v /:/host --privileged` and own the machine. Never mount `docker.sock` into an application container, and never grant `docker` group membership as a convenience on shared build servers.

---

## Lab 1.3 — Materialize the Legacy Application

### Why and what

"LegacyLedger" is an invoice-tracking monolith. It was written against .NET Framework on Windows and force-ported to .NET 8 without redesign. It carries three classic legacy assumptions that make containerization interesting:

1. Its database connection string is baked into `appsettings.json`.
2. It writes an audit log and file uploads to a local directory on disk.
3. It assumes the database is already up when the process starts.

You will hit all three and fix all three.

### Do this — create the project file

```bash
cat > src/LegacyLedger/LegacyLedger.csproj <<'EOF'
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>disable</Nullable>
    <InvariantGlobalization>false</InvariantGlobalization>
    <RootNamespace>LegacyLedger</RootNamespace>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.8" />
  </ItemGroup>
</Project>
EOF
```

**Why `InvariantGlobalization` is `false`.** Invariant mode drops the ICU dependency and produces the smallest possible runtime image, which is tempting. It also breaks `Microsoft.Data.SqlClient`, which resolves specific cultures such as `en-us` on several internal code paths. Setting it to `true` here produces this failure at startup, and it is worth recognising on sight:

```
warn: Program[0] Database not ready (attempt 1/30): Only the invariant culture is
supported in globalization-invariant mode. See https://aka.ms/GlobalizationInvariantMode
for more information. (Parameter 'name') en-us is an invalid culture identifier.
```

The message names globalization, so people chase the connection string for twenty minutes. The real cause is that the ADO.NET provider needs ICU and the project told the runtime there is none.

The consequence carries into the  next Module: this application needs a base image that ships ICU. That rules out plain `alpine` and plain `noble-chiseled`, and points at the `-extra` variants. Lab 2.2 makes that choice explicitly rather than by accident.

If you ever do want invariant mode, verify your data provider supports it first, and set both the MSBuild property and `DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1` so build and runtime agree.

### Do this — create the domain model and data context

```bash
cat > src/LegacyLedger/Models/Invoice.cs <<'EOF'
namespace LegacyLedger.Models;

public class Invoice
{
    public int Id { get; set; }
    public string Reference { get; set; }
    public string Customer { get; set; }
    public decimal Amount { get; set; }
    public DateTime CreatedUtc { get; set; } = DateTime.UtcNow;
}
EOF

cat > src/LegacyLedger/Data/LedgerContext.cs <<'EOF'
using LegacyLedger.Models;
using Microsoft.EntityFrameworkCore;

namespace LegacyLedger.Data;

public class LedgerContext : DbContext
{
    public LedgerContext(DbContextOptions<LedgerContext> options) : base(options) { }

    public DbSet<Invoice> Invoices => Set<Invoice>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Invoice>().Property(i => i.Amount).HasColumnType("decimal(18,2)");
        b.Entity<Invoice>().HasIndex(i => i.Reference).IsUnique();
    }
}
EOF
```

### Do this — create the controllers

```bash
cat > src/LegacyLedger/Controllers/HomeController.cs <<'EOF'
using LegacyLedger.Data;
using LegacyLedger.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace LegacyLedger.Controllers;

public class HomeController : Controller
{
    private readonly LedgerContext _db;
    private readonly IConfiguration _cfg;
    private readonly ILogger<HomeController> _log;

    public HomeController(LedgerContext db, IConfiguration cfg, ILogger<HomeController> log)
    {
        _db = db; _cfg = cfg; _log = log;
    }

    public async Task<IActionResult> Index()
    {
        ViewBag.Instance = Environment.MachineName;
        ViewBag.Tier = _cfg["Ledger:Tier"] ?? "UNSET";
        ViewBag.DataPath = DataPath();
        return View(await _db.Invoices.OrderByDescending(i => i.Id).Take(25).ToListAsync());
    }

    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(string customer, decimal amount)
    {
        var invoice = new Invoice
        {
            Reference = $"INV-{DateTime.UtcNow:yyyyMMdd}-{Guid.NewGuid().ToString("N")[..6].ToUpper()}",
            Customer = string.IsNullOrWhiteSpace(customer) ? "UNKNOWN" : customer.Trim(),
            Amount = amount
        };
        _db.Invoices.Add(invoice);
        await _db.SaveChangesAsync();

        // Legacy behaviour: append to a local audit file on disk.
        var auditFile = Path.Combine(DataPath(), "audit.log");
        Directory.CreateDirectory(DataPath());
        await System.IO.File.AppendAllTextAsync(auditFile,
            $"{DateTime.UtcNow:o}\t{invoice.Reference}\t{invoice.Customer}\t{invoice.Amount}\n");

        _log.LogInformation("Created {Reference} for {Customer}", invoice.Reference, invoice.Customer);
        return RedirectToAction(nameof(Index));
    }

    public IActionResult Audit()
    {
        var auditFile = Path.Combine(DataPath(), "audit.log");
        var body = System.IO.File.Exists(auditFile)
            ? System.IO.File.ReadAllText(auditFile)
            : "(no audit entries yet)";
        return Content(body, "text/plain");
    }

    private string DataPath() =>
        Environment.GetEnvironmentVariable("LEDGER_DATA_PATH") ?? "/app/App_Data";
}
EOF
```

### Do this — create the views

```bash
cat > src/LegacyLedger/Views/_ViewStart.cshtml <<'EOF'
@{ Layout = "_Layout"; }
EOF

cat > src/LegacyLedger/Views/Shared/_Layout.cshtml <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>LegacyLedger</title>
  <style>
    body { font-family: system-ui, sans-serif; margin: 2rem auto; max-width: 60rem; }
    table { border-collapse: collapse; width: 100%; }
    th, td { border-bottom: 1px solid #ddd; padding: .5rem; text-align: left; }
    .meta { background:#f4f4f4; padding:.75rem; font-family:monospace; font-size:.85rem; }
    input, button { padding: .4rem; margin-right: .4rem; }
  </style>
</head>
<body>
  <h1>LegacyLedger</h1>
  @RenderBody()
</body>
</html>
EOF

cat > src/LegacyLedger/Views/Home/Index.cshtml <<'EOF'
@model IEnumerable<LegacyLedger.Models.Invoice>
<div class="meta">
  instance=@ViewBag.Instance | tier=@ViewBag.Tier | datapath=@ViewBag.DataPath
</div>
<h2>New invoice</h2>
<form method="post" asp-action="Create">
  @Html.AntiForgeryToken()
  <input name="customer" placeholder="Customer" required />
  <input name="amount" type="number" step="0.01" value="100.00" required />
  <button type="submit">Create</button>
</form>
<h2>Recent invoices (@Model.Count())</h2>
<table>
  <tr><th>Reference</th><th>Customer</th><th>Amount</th><th>Created (UTC)</th></tr>
  @foreach (var i in Model)
  {
    <tr><td>@i.Reference</td><td>@i.Customer</td><td>@i.Amount</td><td>@i.CreatedUtc.ToString("u")</td></tr>
  }
</table>
<p><a href="/Home/Audit">View audit log</a> | <a href="/healthz">Health</a></p>
EOF
```

### Do this — create the application entry point

```bash
cat > src/LegacyLedger/Program.cs <<'EOF'
using LegacyLedger.Data;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();
builder.Services.AddDbContext<LedgerContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("LedgerDb")));

var app = builder.Build();

app.MapGet("/healthz", async (LedgerContext db) =>
    await db.Database.CanConnectAsync()
        ? Results.Ok(new { status = "healthy" })
        : Results.StatusCode(503));

app.UseStaticFiles();
app.MapDefaultControllerRoute();

// Legacy behaviour: schema is created at startup, and the app assumes the DB is reachable.
await WaitForDatabaseAndInitialize(app);

app.Run();

static async Task WaitForDatabaseAndInitialize(WebApplication app)
{
    var log = app.Services.GetRequiredService<ILogger<Program>>();
    var attempts = int.TryParse(Environment.GetEnvironmentVariable("DB_WAIT_ATTEMPTS"), out var a) ? a : 30;

    for (var i = 1; i <= attempts; i++)
    {
        try
        {
            using var scope = app.Services.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<LedgerContext>();
            await db.Database.EnsureCreatedAsync();
            log.LogInformation("Database ready after {Attempt} attempt(s).", i);
            return;
        }
        catch (Exception ex)
        {
            log.LogWarning("Database not ready (attempt {Attempt}/{Max}): {Message}", i, attempts, ex.Message);
            await Task.Delay(TimeSpan.FromSeconds(2));
        }
    }
    throw new InvalidOperationException("Database unreachable after all retries.");
}
EOF
```

### Do this — create the legacy configuration file

```bash
cat > src/LegacyLedger/appsettings.json <<'EOF'
{
  "Logging": { "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Warning" } },
  "AllowedHosts": "*",
  "Ledger": { "Tier": "legacy-default" },
  "ConnectionStrings": {
    "LedgerDb": "Server=.\\SQLEXPRESS;Database=LedgerDb;Trusted_Connection=True;TrustServerCertificate=True"
  }
}
EOF
```

**Verify.**

```bash
find src -type f | sort
```

You should have nine files. Note the connection string: it points at a Windows named instance with integrated authentication. That value is wrong in every environment except one developer's laptop. Lab 1.9 replaces it with configuration from the environment.

### Do this — commit the baseline

```bash
git add -A && git commit -q -m "Legacy ASP.NET monolith baseline" && git log --oneline
```

---

## Lab 1.4 — Your First Dockerfile (Deliberately Naive)

### Why and what

You are going to write the Dockerfile most teams write first: correct, and bad in four measurable ways. Then you will measure each flaw and fix it. Building the wrong version once is worth more than reading the right version three times.

A Dockerfile is a build script whose every instruction that changes the filesystem produces a layer. Layers are content-addressed, cached, and shared between images. Ordering instructions well is not style — it is the difference between a 4-second rebuild and a 90-second one.

### Do this — write the naive Dockerfile

```bash
cat > Dockerfile <<'EOF'
# Naive v1. We will fix everything about this file.
FROM mcr.microsoft.com/dotnet/sdk:8.0

WORKDIR /app
COPY . .
RUN dotnet restore src/LegacyLedger/LegacyLedger.csproj
RUN dotnet publish src/LegacyLedger/LegacyLedger.csproj -c Release -o /app/out

ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "/app/out/LegacyLedger.dll"]
EOF
```

### Do this — build and time it

```bash
time docker build -t ledger:v1-naive .
```

**Verify.**

```bash
docker images ledger --format 'table {{.Tag}}\t{{.Size}}'
```

**Record your numbers.** Fill this in as you go through Labs 1.4 to 1.6 and 2.1 to 2.2.

| Tag                        | Build time (cold) | Build time (code change) | Image size |
| -------------------------- | ----------------- | ------------------------ | ---------- |
| `v1-naive`                 |                   |                          |            |
| `v2-cached`                |                   |                          |            |
| `v3-ignore`                |                   |                          |            |
| `v4-multistage` (Module 2) |                   |                          |            |
| `v5-chiselled` (Module 2)  |                   |                          |            |

The `v1-naive` size should land near 900 MB. You shipped the entire .NET SDK, NuGet caches, compilers, and your source code to production to run a 200 KB DLL.

### Do this — inspect the layers

```bash
docker history ledger:v1-naive --no-trunc --format 'table {{.Size}}\t{{.CreatedBy}}' | head -20
```

**Verify.** You can see exactly which instruction cost which bytes. `docker history` is your first stop whenever an image is unexpectedly large.

### Do this — feel the cache miss

Change one character in a Razor view, then rebuild.

```bash
sed -i.bak 's/Recent invoices/Latest invoices/' src/LegacyLedger/Views/Home/Index.cshtml
time docker build -t ledger:v1-naive .
```

**Verify.** The `dotnet restore` step ran again, even though no package reference changed. Every rebuild pays the full restore cost. Reason about why: `COPY . .` invalidated its layer because a file changed, and every layer after an invalidated layer is rebuilt. Record the time in your table.

---

## Lab 1.5 — Cache-Efficient Layer Ordering

### Why and what

The build cache matches on instruction text plus, for `COPY` and `ADD`, the checksum of the copied content. The rule that follows: **copy the files that change rarely before the files that change constantly.** Package manifests change weekly. Source code changes hourly. So copy the manifest, restore dependencies, then copy the source.

### Do this — reorder for cache efficiency

```bash
cat > Dockerfile <<'EOF'
# v2. Same output, correct cache ordering.
FROM mcr.microsoft.com/dotnet/sdk:8.0

WORKDIR /src

# 1. Copy only the manifest. This layer changes when dependencies change.
COPY src/LegacyLedger/LegacyLedger.csproj ./LegacyLedger/
RUN dotnet restore LegacyLedger/LegacyLedger.csproj

# 2. Copy the source. This layer changes on every commit.
COPY src/LegacyLedger/ ./LegacyLedger/
RUN dotnet publish LegacyLedger/LegacyLedger.csproj -c Release -o /app/out --no-restore

WORKDIR /app
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "/app/out/LegacyLedger.dll"]
EOF
```

### Do this — measure the cold build, then the warm one

```bash
docker builder prune -af >/dev/null
time docker build -t ledger:v2-cached .

sed -i.bak 's/Latest invoices/Recent invoices/' src/LegacyLedger/Views/Home/Index.cshtml
time docker build -t ledger:v2-cached .
```

**Verify.** The second build prints `CACHED` next to the `COPY ... .csproj` and `RUN dotnet restore` steps, and completes in a fraction of the first. Record both times.

**Verify the reasoning.** Now change the dependency and watch the cache correctly invalidate.

```bash
sed -i.bak 's/8.0.8/8.0.10/' src/LegacyLedger/LegacyLedger.csproj
docker build -t ledger:v2-cached . 2>&1 | grep -E 'CACHED|restore'
git checkout src/LegacyLedger/LegacyLedger.csproj 2>/dev/null || sed -i.bak 's/8.0.10/8.0.8/' src/LegacyLedger/LegacyLedger.csproj
```

**Verify.** Restore ran again, as it should. The cache is not "faster" — it is *correct*. It re-runs exactly the work whose inputs changed.

### Production rules to carry forward

- One `RUN` per logical unit of work, but combine commands that must share a layer to avoid orphaned intermediate files. For OS packages: `RUN apt-get update && apt-get install -y --no-install-recommends X && rm -rf /var/lib/apt/lists/*` in a single instruction. Splitting `update` and `install` into two instructions causes stale-index failures when the first is cached and the second is not.
- Pin base image tags. `sdk:8.0` is a moving target; `sdk:8.0.403-noble` is reproducible. Module 2 goes further and pins by digest.
- Never `COPY` secrets into a layer. Deleting a file in a later layer does not remove it from the image — the bytes remain in the earlier layer and anyone can extract them.

---

## Lab 1.6 — The Build Context and .dockerignore

### Why and what

Before any instruction runs, the client packages your build context and sends it to the builder. Everything in the context directory goes, including `.git`, `bin/`, `obj/`, `node_modules/`, and that 400 MB database backup someone left in the repository. A large context slows every build, and a careless `COPY . .` bakes secrets like `.env` and `*.pem` straight into the image.

### Do this — measure the context you are currently sending

```bash
dotnet build src/LegacyLedger/LegacyLedger.csproj -c Release 2>/dev/null || \
  docker run --rm -v "$PWD":/w -w /w mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet build src/LegacyLedger/LegacyLedger.csproj -c Release
du -sh . && du -sh .git src/LegacyLedger/bin src/LegacyLedger/obj 2>/dev/null
docker build -t ledger:ctx-test . --progress=plain --no-cache 2>&1 | grep -i 'transferring context'
```

**Verify.** Note the transferred byte count. The `bin/` and `obj/` directories you just generated are being uploaded on every single build and are useless inside the image.

### Do this — add a .dockerignore

```bash
cat > .dockerignore <<'EOF'
# Version control and CI
.git
.gitignore
.github
.azuredevops

# Build output — never copy host build artifacts into an image
**/bin
**/obj
**/out
**/TestResults

# Local environment and secrets.
# Note the ** prefixes: a bare `.env` matches ONLY the context root, so a secret
# sitting inside src/ would still be copied. This is a common and expensive mistake.
**/.env
**/.env.*
**/*.pem
**/*.key
**/*.pfx
**/appsettings.Development.json
**/secrets.json

# Editor and OS noise
.vs
.vscode
.idea
**/*.user
.DS_Store

# Docs and compose files not needed inside the image
README.md
docs/
docker-compose*.yml
deploy/
EOF
```

### Do this — measure again

```bash
docker build -t ledger:v3-ignore . --progress=plain --no-cache 2>&1 | grep -i 'transferring context'
```

**Verify.** The transferred context should drop by an order of magnitude or more. Record it. On a real monolith with a long Git history, this routinely takes builds from minutes to seconds.

### Verify the security benefit

Plant a credential where the build would actually pick it up — inside the directory the Dockerfile copies from — then compare the image with and without `.dockerignore`.

```bash
echo "SA_PASSWORD=SuperSecret123" > src/LegacyLedger/.env

# Negative control: build with .dockerignore disabled.
mv .dockerignore .dockerignore.off
docker build -t ledger:secret-test . -q --no-cache >/dev/null
echo -n "without .dockerignore: "
docker run --rm \
  --entrypoint /bin/sh \
  ledger:secret-test \
  -c 'ls -la /src/LegacyLedger 2>/dev/null | grep -c "\.env" || echo "0 — .env excluded, correct"'

# Restore it and rebuild.
mv .dockerignore.off .dockerignore
docker build -t ledger:secret-test . -q --no-cache >/dev/null
echo -n "with .dockerignore:    "
docker run --rm \
  --entrypoint /bin/sh \
  ledger:secret-test \
  -c 'ls -la /src/LegacyLedger 2>/dev/null | grep -c "\.env" || echo "0 — .env excluded, correct"'
```

**Verify.** The first build reports `1`. The second reports `0 — .env excluded, correct`.

Note the explicit `--entrypoint /bin/sh`. The image's entrypoint is `dotnet`, so appending `ls` as arguments would ask the .NET host to run an assembly called `ls` rather than listing a directory. You will use this override constantly to inspect images.

**Verify the deeper point.** In the first build, that credential is baked into a layer. Deleting the file in a later instruction would not help — the bytes stay in the earlier layer, and anyone who can pull the image can extract them with `docker save` and `tar`.

### Do this — clean up the test artifacts

```bash
rm -f src/LegacyLedger/.env src/LegacyLedger/*.bak 2>/dev/null
docker rmi -f ledger:secret-test ledger:ctx-test >/dev/null 2>&1
git add -A && git commit -q -m "Add cache-efficient Dockerfile and .dockerignore"
```

---

## Lab 1.7 — Container Networking

### Why and what

Docker gives you four network modes that matter:

| Mode                | Behaviour                                                                                               | Use it for                                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `bridge` (default)  | Shared private subnet, NAT to host, **no segmentation, no aliases, DNS behaviour is version-dependent** | Almost nothing, deliberately                                 |
| user-defined bridge | Private subnet with an embedded DNS server that resolves container names                                | Every multi-container application                            |
| `host`              | No network namespace; the container uses the host stack directly                                        | High-throughput proxies, some monitoring agents. Linux only. |
| `none`              | Loopback only                                                                                           | Batch jobs that must not touch the network                   |

The critical fact: **every container on the default bridge shares one flat network with no segmentation and no aliases.** Historically it also had no name resolution at all, and much of the documentation still says so. Recent Docker Engine and Docker Desktop builds have begun resolving container names on the default bridge too, so what you observe depends on your version.

Do not memorise the DNS behaviour. Memorise the rule that follows from it: **the default bridge is unspecified ground, so never build on it.** Anything multi-container gets an explicit user-defined network.

### Do this — observe what your engine actually does

```bash
docker run -d --name alpha alpine:3.20 sleep 600
docker run -d --name beta  alpine:3.20 sleep 600

docker exec alpha ping -c1 -W2 beta ; echo "name lookup exit=$?"

BETA_IP=$(docker inspect beta -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}')
echo "beta ip = $BETA_IP"
docker exec alpha ping -c1 -W2 "$BETA_IP" ; echo "ip exit=$?"
```

**Verify.** Ping by IP always succeeds (`exit=0`). Ping by name may succeed or fail depending on your Docker version — record which you got, and compare across the room. 

> The older `-f '{{.NetworkSettings.IPAddress}}'` form is deprecated and returns an empty string for any container attached to a user-defined network, because the address moved under `.NetworkSettings.Networks.<name>.IPAddress`. Always use the `range` form above; it works in both cases.

### Do this — find the two differences that are not version-dependent

```bash
# 1. Aliases are rejected outright on the default bridge.
docker network connect --alias db bridge alpha 2>&1 | tail -1

# 2. There is no segmentation. Every container lands in the same broadcast domain.
docker network inspect bridge -f '{{range $k, $v := .Containers}}{{$v.Name}} {{end}}'
```

**Verify.** The alias command is refused — network-scoped aliases are supported only on user-defined networks. And every container you have running is listed on the one bridge, mutually reachable. There is no way to express "the proxy must not reach the database" here. That, not DNS, is the durable reason the default bridge is unusable for real applications.

### Do this — create a user-defined network and prove DNS works

```bash
docker network create ledger-net
docker network inspect ledger-net -f '{{range .IPAM.Config}}{{.Subnet}}{{end}}'

docker run -d --name gamma --network ledger-net alpine:3.20 sleep 600
docker run -d --name delta --network ledger-net alpine:3.20 sleep 600

docker exec gamma ping -c1 -W2 delta ; echo "exit=$?"
docker exec gamma nslookup delta 127.0.0.11
```

**Verify.** Ping by name succeeds. `nslookup` against `127.0.0.11` — Docker's embedded DNS resolver — returns delta's address. This is the mechanism your Compose file will rely on to let the app find the database by the hostname `db`.

### Do this — network aliases and multi-network attachment

```bash
docker network create ledger-backend
docker network connect --alias database ledger-backend delta
docker run -d --name epsilon --network ledger-backend alpine:3.20 sleep 600

docker exec epsilon ping -c1 -W2 database ; echo "exit=$?"
docker exec gamma  ping -c1 -W2 epsilon  ; echo "exit=$?"
```

**Verify.** `epsilon` reaches `delta` through the alias `database`. `gamma` cannot reach `epsilon` — they are on different networks. This is how you build a segmented stack: the proxy sits on a frontend network, the database sits on a backend network, and only the app bridges both. The database is then unreachable from the internet-facing tier by construction.

### Do this — none and host modes

```bash
docker run --rm --network none alpine:3.20 sh -c 'ip -o addr show | wc -l'
docker run --rm --network none alpine:3.20 sh -c 'wget -T2 -q -O- http://example.com || echo "no egress, as expected"'
```

**Verify.** Only the loopback interface exists and egress fails. Use `--network none` for untrusted batch workloads.

```bash
# Linux hosts only. On Docker Desktop, host networking behaves differently.
docker run --rm --network host alpine:3.20 hostname
```

**Verify (Linux).** The container prints the *host's* hostname because it shares the host's UTS and network namespaces. Port publishing is meaningless in host mode — the process binds host ports directly. That removes the NAT hop but also removes all network isolation.

### Do this — port publishing

```bash
docker run -d --name web --network ledger-net -p 8081:80 nginx:1.27-alpine
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8081
docker port web
```

**Verify.** `200`. `docker port` shows the mapping. Note the direction: `-p HOST:CONTAINER`. Binding to `-p 127.0.0.1:8081:80` restricts exposure to the local machine, which is what you want for admin interfaces.

### Do this — clean up

```bash
docker rm -f alpha beta gamma delta epsilon web >/dev/null
docker network rm ledger-net ledger-backend >/dev/null
```

---

## Lab 1.8 — Volumes, Bind Mounts, and Where State Belongs

### Why and what

A container's writable layer dies with the container. Anything you need to survive must live outside it.

| Mechanism    | Managed by            | Use for                                                           |
| ------------ | --------------------- | ----------------------------------------------------------------- |
| Named volume | Docker                | Database data directories, anything the container owns            |
| Bind mount   | You, from a host path | Source code during development, config files, host log collection |
| tmpfs mount  | Kernel, RAM only      | Scratch space in a read-only container                            |

The general rule: **a container should be disposable.** If deleting and recreating it loses something that mattered, that something belonged in a volume, a database, or object storage.

### Do this — prove the writable layer is ephemeral

```bash
docker run -d --name ephemeral alpine:3.20 sleep 600
docker exec ephemeral sh -c 'echo "critical business data" > /data.txt'
docker exec ephemeral cat /data.txt

docker rm -f ephemeral >/dev/null
docker run -d --name ephemeral alpine:3.20 sleep 600
docker exec ephemeral cat /data.txt 2>&1 || echo "GONE — this is the lesson"
```

**Verify.** The file is gone. Same image, new container, empty writable layer.

### Do this — named volume survives container deletion

```bash
docker volume create ledger-data
docker run -d --name keeper -v ledger-data:/data alpine:3.20 sleep 600
docker exec keeper sh -c 'echo "critical business data" > /data/ledger.txt'

docker rm -f keeper >/dev/null
docker run --rm -v ledger-data:/data alpine:3.20 cat /data/ledger.txt
```

**Verify.** The file is intact. The volume outlives every container that mounts it.

```bash
docker volume inspect ledger-data -f '{{.Mountpoint}}'
```

### Do this — run SQL Server on a persistent volume

```bash
docker network create ledger-net

docker run -d --name ledger-db \
  --network ledger-net \
  -e ACCEPT_EULA=Y \
  -e MSSQL_SA_PASSWORD='L3dger#Lab2024!' \
  -e MSSQL_PID=Developer \
  -v ledger-mssql:/var/opt/mssql \
  -p 127.0.0.1:1433:1433 \
  mcr.microsoft.com/mssql/server:2022-latest
```

**Verify.** Wait about 20 seconds, then confirm the engine is accepting connections.

```bash
sleep 20
docker exec ledger-db /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U sa -P 'L3dger#Lab2024!' -C -Q "SELECT @@VERSION"
```

You should see the SQL Server 2022 version banner. The `-C` flag trusts the self-signed server certificate, which is required from the mssql-tools18 package onward.

### Do this — prove database persistence across container replacement

```bash
docker exec ledger-db /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'L3dger#Lab2024!' -C \
  -Q "CREATE DATABASE PersistProof; SELECT name FROM sys.databases WHERE name='PersistProof'"

docker rm -f ledger-db >/dev/null

docker run -d --name ledger-db --network ledger-net \
  -e ACCEPT_EULA=Y -e MSSQL_SA_PASSWORD='L3dger#Lab2024!' -e MSSQL_PID=Developer \
  -v ledger-mssql:/var/opt/mssql -p 127.0.0.1:1433:1433 \
  mcr.microsoft.com/mssql/server:2022-latest

sleep 20
docker exec ledger-db /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'L3dger#Lab2024!' -C \
  -Q "SELECT name FROM sys.databases WHERE name='PersistProof'"
```

**Verify.** `PersistProof` is still listed. You destroyed and recreated the database *container* without losing the *data*. That separation is the whole point.

### Do this — run the application against it, wired by hostname

```bash
docker build -t ledger:v3-ignore .

docker run -d --name ledger-app \
  --network ledger-net \
  -e ConnectionStrings__LedgerDb='Server=ledger-db,1433;Database=LedgerDb;User Id=sa;Password=L3dger#Lab2024!;Encrypt=True;TrustServerCertificate=True' \
  -e Ledger__Tier='lab-1.8' \
  -e LEDGER_DATA_PATH=/app/App_Data \
  -v "$PWD/appdata":/app/App_Data \
  -p 8080:8080 \
  ledger:v3-ignore
```

**Verify.**

```bash
sleep 10
curl -s http://localhost:8080/healthz
docker logs ledger-app --tail 5
```

You should get `{"status":"healthy"}`. Open http://localhost:8080 in a browser, create an invoice, then check the bind mount from the host.

```bash
cat ./appdata/audit.log
```

**Verify.** The audit line written *inside* the container is visible on your *host* filesystem. That is a bind mount.

### Understand the configuration mechanism you just used

ASP.NET Core maps environment variables to configuration keys by replacing `:` with `__` (double underscore). `ConnectionStrings__LedgerDb` overrides the `ConnectionStrings:LedgerDb` value in `appsettings.json`, and `Ledger__Tier` overrides `Ledger:Tier`. The tier shown on the page confirms it.

**Verify.**

```bash
curl -s http://localhost:8080 | grep -o 'tier=[a-z0-9.-]*'
```

You should see `tier=lab-1.8`, not `legacy-default`. The baked-in `appsettings.json` connection string was never used. Legacy assumption number one is now solved.

### Do this — bind mount permission trap (read this, then avoid it forever)

```bash
ls -ln ./appdata/audit.log
```

**Verify and internalize.** The file is owned by UID 0, because the container currently runs as root. In Module 2 you will switch to a non-root UID, and this bind mount will start failing with permission denied unless the host directory ownership matches. Bind mounts do not translate user IDs. Named volumes sidestep this because Docker initializes their ownership from the image's directory. Prefer named volumes for anything the container owns.

### Do this — clean up

```bash
docker rm -f ledger-app ledger-db >/dev/null
docker network rm ledger-net >/dev/null
rm -rf ./appdata
```

Keep the `ledger-mssql` volume. Compose will reuse it in the next lab.

---

## Lab 1.9 — Compose: The Full Local Stack

### Why and what

`docker run` with eight flags is unrepeatable. Compose declares the same stack as data: services, networks, volumes, dependencies, and health conditions in one file that lives in Git next to the code.

You are building three services on two networks:

```
        [ host :8080 ]
              |
        +-----v-----+   frontend network
        |   proxy   |   (nginx, the only published service)
        +-----+-----+
              |
        +-----v-----+   frontend + backend
        |    app    |   (LegacyLedger)
        +-----+-----+
              |
        +-----v-----+   backend network only
        |    db     |   (SQL Server, never published)
        +-----------+
```

The database has no published port and sits on a network the proxy cannot reach. Segmentation is expressed in the topology, not in a firewall rule someone has to remember.

### Do this — externalize secrets into an env file

```bash
cat > .env <<'EOF'
# Local development values. Never commit real secrets. This file is in .dockerignore
# and must also be in .gitignore.
COMPOSE_PROJECT_NAME=ledger
MSSQL_SA_PASSWORD=L3dger#Lab2024!
LEDGER_TIER=compose-local
APP_HTTP_PORT=8080
EOF

printf '.env\nappdata/\n**/bin/\n**/obj/\n' >> .gitignore
```

**Verify.**

```bash
git check-ignore -v .env
```

It must report a match. If `.env` is ever committed, treat the credential as compromised and rotate it.

### Do this — write the reverse proxy configuration

```bash
cat > deploy/nginx/default.conf <<'EOF'
upstream ledger_app {
    server app:8080;
    keepalive 16;
}

server {
    listen 80;
    server_name _;

    # Do not advertise the version in error pages or headers.
    server_tokens off;

    location /healthz {
        access_log off;
        proxy_pass http://ledger_app;
    }

    location / {
        proxy_pass http://ledger_app;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection        "";
        proxy_connect_timeout 5s;
        proxy_read_timeout    30s;
    }
}
EOF
```

Note `server app:8080`. `app` is the Compose service name, resolved by Docker's embedded DNS on the user-defined network you proved out in Lab 1.7.

### Do this — write the Compose file

```bash
cat > docker-compose.yml <<'EOF'
services:

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: ${MSSQL_SA_PASSWORD:?MSSQL_SA_PASSWORD must be set in .env}
      MSSQL_PID: Developer
    volumes:
      - mssql-data:/var/opt/mssql
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P \"$$MSSQL_SA_PASSWORD\" -C -Q 'SELECT 1' || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 30s
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 2g

  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: ledger:compose
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      ASPNETCORE_URLS: http://+:8080
      ConnectionStrings__LedgerDb: >-
        Server=db,1433;Database=LedgerDb;User Id=sa;Password=${MSSQL_SA_PASSWORD};
        Encrypt=True;TrustServerCertificate=True;Connect Timeout=10
      Ledger__Tier: ${LEDGER_TIER:-compose-local}
      LEDGER_DATA_PATH: /app/App_Data
    volumes:
      - app-data:/app/App_Data
    networks:
      - backend
      - frontend
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8080/healthz || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 6
      start_period: 20s
    restart: unless-stopped

  proxy:
    image: nginx:1.27-alpine
    ports:
      - "${APP_HTTP_PORT:-8080}:80"
    volumes:
      - ./deploy/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - frontend
    depends_on:
      app:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1/healthz || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 5
    restart: unless-stopped

networks:
  frontend:
  backend:
    internal: true

volumes:
  mssql-data:
  app-data:
EOF
```

Four details worth pausing on:

- `${MSSQL_SA_PASSWORD:?...}` fails the run loudly if the variable is missing, instead of silently starting a broken stack.
- `$$MSSQL_SA_PASSWORD` inside the healthcheck escapes the dollar sign so Compose passes it through for the *container's* shell to expand, rather than substituting it at parse time.
- `condition: service_healthy` is real dependency ordering. Plain `depends_on` only waits for the container to *start*, which is useless for a database that takes 30 seconds to become ready. This is legacy assumption number three, solved.
- `backend: internal: true` gives the database network no gateway to the outside world.

### Do this — validate before running

```bash
docker compose config --quiet && echo "compose file is valid"
docker compose config | head -30
```

**Verify.** No errors, and the rendered output shows your `.env` values interpolated. Always run `compose config` before `compose up` in CI — it catches indentation and interpolation mistakes in one second instead of two minutes.

### Do this — bring up the stack

```bash
docker compose up -d --build
watch -n2 docker compose ps
```

**Verify.** Within about 60 seconds all three services report `Up` and `(healthy)`. Press Ctrl-C to exit `watch`.

```bash
docker compose ps --format 'table {{.Service}}\t{{.Status}}\t{{.Ports}}'
```

### Do this — verify end to end through the proxy

```bash
curl -s http://localhost:8080/healthz ; echo
curl -s -o /dev/null -w 'proxy status: %{http_code}\n' http://localhost:8080
curl -sI http://localhost:8080/ | grep -i '^server'
```

**Verify.** Health returns `{"status":"healthy"}`, the root returns `200`, and the `Server` header shows nginx — traffic is going through the proxy, not directly to Kestrel.

### Do this — verify the database is genuinely unreachable from outside

```bash
curl -s -m3 telnet://localhost:1433 ; echo "exit=$?"
docker compose exec proxy sh -c 'nc -z -w2 db 1433 && echo REACHABLE || echo "unreachable from proxy tier"'
docker compose exec app  sh -c 'timeout 2 bash -c "</dev/tcp/db/1433" && echo "reachable from app tier"'
```

**Verify.** The host cannot reach 1433 (no published port). The proxy cannot reach the database (wrong network). The app can. Your segmentation works.

### Do this — verify state survives a full teardown of containers

```bash
# Create data through the UI or via curl.
TOKEN_PAGE=$(curl -s -c /tmp/ck http://localhost:8080/)
TOKEN=$(echo "$TOKEN_PAGE" | grep -o 'name="__RequestVerificationToken"[^>]*value="[^"]*"' | sed 's/.*value="//;s/"//')
curl -s -b /tmp/ck -o /dev/null -X POST http://localhost:8080/Home/Create \
  --data-urlencode "customer=Hitachi Energy Nepal" \
  --data-urlencode "amount=45000.00" \
  --data-urlencode "__RequestVerificationToken=$TOKEN"

curl -s http://localhost:8080/Home/Audit
```

Now destroy the containers but keep the volumes.

```bash
docker compose down
docker volume ls | grep ledger
docker compose up -d
sleep 45
curl -s http://localhost:8080/Home/Audit
curl -s http://localhost:8080/ | grep -c 'Hitachi'
```

**Verify.** The audit log and the invoice row both survive. The containers were destroyed; the state was not.

### Do this — see what `down -v` does, and understand why it is dangerous

```bash
docker compose down -v
docker volume ls | grep ledger || echo "volumes deleted"
docker compose up -d && sleep 45
curl -s http://localhost:8080/Home/Audit
```

**Verify.** The audit log is empty and the invoice is gone. `-v` deletes named volumes. This flag has destroyed production data at real companies. Never alias it, never put it in a script that runs anywhere except a developer laptop.

### Do this — operational commands you will use constantly

```bash
docker compose logs -f app --tail 50        # follow one service
docker compose logs --since 5m              # all services, recent
docker compose exec app sh                  # shell into the running app
docker compose restart app                  # restart one service
docker compose up -d --build app            # rebuild and replace one service
docker compose top                          # processes per service
docker stats --no-stream                    # live resource use
docker compose events --json                # audit stream of lifecycle events
wget -qO- http://localhost/```

### Do this — commit the deliverable

```bash
git add -A && git commit -q -m "Module 1 deliverable: containerized legacy monolith with Compose stack"
git log --oneline
```

---

## Module 1 Sign-Off Checklist


| #   | Requirement                                           | Proof command                                                        | Expected                 |
| --- | ----------------------------------------------------- | -------------------------------------------------------------------- | ------------------------ |
| 1   | Legacy app is containerized                           | `docker images ledger --format '{{.Tag}} {{.Size}}'`                 | Image exists             |
| 2   | Multi-service stack runs                              | `docker compose ps --format '{{.Service}} {{.Status}}'`              | 3 services, all healthy  |
| 3   | App is reachable via reverse proxy                    | `curl -so /dev/null -w '%{http_code}' localhost:8080`                | `200`                    |
| 4   | Config comes from environment, not `appsettings.json` | `curl -s localhost:8080 \| grep -o 'tier=[a-z-]*'`                   | `tier=compose-local`     |
| 5   | Database has no host-published port                   | `docker compose port db 1433`                                        | Error / no mapping       |
| 6   | State persists across `compose down`                  | `down` then `up`, re-check `/Home/Audit`                             | Data present             |
| 7   | Build cache ordering is correct                       | Touch a `.cshtml`, rebuild                                           | `restore` shows `CACHED` |
| 8   | Build context is minimized                            | `docker build --progress=plain --no-cache 2>&1 \| grep transferring` | Small context            |
| 9   | Secrets are not in the repository                     | `git check-ignore -v .env`                                           | Match reported           |

### Honest gaps to carry into Module 2

Your stack works, and it is not shippable. Name each problem now:

1. The image is roughly 900 MB to 1 GB and contains a full SDK, compilers, and NuGet caches — all of it attack surface.
2. The container runs as **root**.
3. The filesystem is fully writable.
4. Nothing has scanned this image for known vulnerabilities.
5. The image exists only on your laptop, built by hand, tagged by hand.
6. The SA password is in a plaintext file and the app connects as `sa`, the highest-privilege account in the engine.
