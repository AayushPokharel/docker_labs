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
      test: ["CMD-SHELL", "wget -qO- http://localhost/healthz || exit 1"]
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
curl -s -o /dev/null -w 'proxy status: %{http_code}\n' http://localhost:8080/
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
```

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

Module 2 fixes items 1 through 5 and makes item 6 a documented exception with a remediation path.

---

# MODULE 2 — ENTERPRISE DOCKER AND AZURE CONTAINER REGISTRY

**Objective.** Take the working-but-unshippable image from Module 1 and turn it into an artifact you would defend in a security review: minimal, non-root, read-only, scanned, signed, immutably tagged, and published by a pipeline that holds no long-lived credentials.

**Deliverable.** A hardened image in Azure Container Registry, published by a pipeline that fails on critical CVEs, referenced by digest.

**The bridge from Module 1.** Nothing about the application changes. Every change in this module is to the *build*, the *runtime posture*, and the *supply chain*. Keep `docker compose up` working throughout — if a hardening step breaks the stack, that is the lesson, and you fix it rather than reverting it.

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

# APPENDICES

## Appendix A — Failure Modes and Fixes

| Symptom                                                                                                              | Cause                                                                                   | Fix                                                                                                                               |
| -------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `Only the invariant culture is supported in globalization-invariant mode ... en-us is an invalid culture identifier` | `InvariantGlobalization=true`, or a base image with no ICU (`alpine`, `noble-chiseled`) | Set the property to `false` **and** use an `-extra` base image. `Microsoft.Data.SqlClient` requires ICU. See Lab 1.3 and Lab 2.2. |
| Container exits with code `137`                                                                                      | OOM kill by cgroup                                                                      | Raise the memory limit or fix the leak. `docker inspect <c> --format '{{.State.OOMKilled}}'` confirms it.                         |
| Container exits with code `143`                                                                                      | SIGTERM, graceful stop                                                                  | Normal shutdown. If it is unexpected, check the orchestrator's liveness probe.                                                    |
| Container exits with code `125`                                                                                      | Docker daemon could not run the container                                               | Bad flag or missing image. Read the daemon's error text.                                                                          |
| Container exits with code `126`                                                                                      | Command found but not executable                                                        | Missing execute bit on an entrypoint script, or CRLF line endings on a shell script.                                              |
| Container exits with code `127`                                                                                      | Command not found                                                                       | You are on a minimal base with no shell. Use an exec-form ENTRYPOINT with an absolute path.                                       |
| `dial tcp: lookup db: no such host`                                                                                  | Container is on the default bridge                                                      | Attach it to a user-defined network. See Lab 1.7.                                                                                 |
| `Access to the path ... is denied` after hardening                                                                   | Non-root UID against a root-owned volume or bind mount                                  | `chown` the host path to `1654`, or use a named volume created after the `USER` switch.                                           |
| Build cache never hits                                                                                               | `COPY . .` before dependency restore                                                    | Reorder as in Lab 1.5. Confirm `.dockerignore` excludes `bin/` and `obj/`.                                                        |
| `unauthorized: authentication required` on push                                                                      | Registry token expired (they last about 3 hours)                                        | Re-run `az acr login --name $ACR_NAME`.                                                                                           |
| `denied: client is not authorized to perform this operation`                                                         | Missing `AcrPush`, or a network rule is blocking you                                    | Check `az role assignment list` and `az acr network-rule list`.                                                                   |
| `TOOMANYREQUESTS` pulling from Docker Hub                                                                            | Anonymous pull rate limit                                                               | Mirror public base images into ACR and pull from there.                                                                           |
| Trivy DB download fails intermittently                                                                               | Public registry rate limiting                                                           | Set `TRIVY_DB_REPOSITORY` to your ACR mirror. See Lab 2.4.                                                                        |
| SQL Server container restarts in a loop                                                                              | Under 2 GB memory available                                                             | Raise the Docker VM memory. `docker logs` shows the memory error explicitly.                                                      |
| Antiforgery or auth cookies break on restart                                                                         | Data Protection keys not persisted                                                      | Persist the key ring. See the Data Protection note in Lab 2.3.                                                                    |
| `no space left on device` mid-build                                                                                  | Builder cache and dangling images                                                       | `docker system df` then `docker builder prune -af`.                                                                               |
| Memory-limit demo exits `1` instead of `137`                                                                         | Default `/dev/shm` is 64 MB, so the tmpfs filled before the cgroup limit was hit        | Pass `--shm-size=256m` so the memory limit is the binding constraint. See Lab 1.1.                                                |
| `docker inspect ... .NetworkSettings.IPAddress` returns empty                                                        | Container is on a user-defined network                                                  | Use `'{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'`.                                                                 |
| `ls`, `sh`, or `cat` appended to `docker run` produces a .NET host error                                             | The image entrypoint is `dotnet`; your arguments were passed to it                      | Override it: `docker run --rm --entrypoint /bin/sh <image> -c '...'`.                                                             |
| WSL: builds are extremely slow                                                                                       | Repository lives under `/mnt/c/...`                                                     | Move it to the Linux filesystem at `~/labs/...`.                                                                                  |
| WSL: script fails with exit code `126`                                                                               | CRLF line endings from Git for Windows                                                  | `git config --global core.autocrlf input`, then `dos2unix` the affected files.                                                    |
| WSL: Azure CLI reports token or certificate validity errors                                                          | VM clock drift after laptop sleep                                                       | `sudo hwclock -s`, or `wsl --shutdown` from PowerShell.                                                                           |

## Appendix B — Command Reference

```bash
# --- Diagnosis ---
docker system df                                   # where disk is going
docker stats --no-stream                           # live CPU and memory per container
docker inspect <c> --format '{{json .State}}' | jq # exit code, OOM flag, health
docker history <image> --no-trunc                  # per-layer size attribution
docker diff <c>                                    # what changed in the writable layer
docker events --since 30m --filter 'event=die'     # why containers stopped

# --- Cleanup, least to most destructive ---
docker container prune -f          # stopped containers
docker image prune -f              # dangling images
docker builder prune -af           # entire build cache
docker system prune -a             # everything unused. Does NOT touch named volumes.
docker system prune -a --volumes   # DESTROYS NAMED VOLUMES. Never on a shared host.

# --- ACR operations ---
az acr repository list --name $ACR_NAME -o table
az acr repository show-tags --name $ACR_NAME --repository ledger --orderby time_desc -o table
az acr manifest list-metadata --registry $ACR_NAME --name ledger -o table
az acr repository delete --name $ACR_NAME --image ledger:oldtag --yes
az acr build --registry $ACR_NAME --image ledger:{{.Run.ID}} .   # build in the cloud
az acr import --name $ACR_NAME --source docker.io/library/nginx:1.27-alpine --image mirror/nginx:1.27-alpine
az acr check-health --name $ACR_NAME --yes
```

## Appendix C — Lab Teardown

```bash
# Local
docker compose down -v
docker rmi -f $(docker images 'ledger*' -q) 2>/dev/null
docker builder prune -af

# Azure. Verify the resource group name before running.
source .lab-env
az security pricing create --name Containers --tier Free --output none 2>/dev/null
az group delete --name "$RG" --yes --no-wait
az group list --query "[?name=='$RG'].{name:name, state:properties.provisioningState}" -o table
```

Deleting the resource group removes the registry, Key Vault, and managed identities. Soft-deleted Key Vaults persist for the retention period — purge with `az keyvault purge --name $AKV_NAME` if the name must be reused.

## Appendix D — Instructor Timing

| Segment                                 | Minutes    | Common overrun                                                 |
| --------------------------------------- | ---------- | -------------------------------------------------------------- |
| Lab 0 environment prep                  | 30         | Corporate proxy configuration. Pre-stage images.               |
| Lab 0.2b WSL setup (Windows rooms only) | 15         | `.wslconfig` memory cap, and moving repos off `/mnt/c`         |
| 1.1 Containers vs VMs                   | 25         | —                                                              |
| 1.2 Docker architecture                 | 20         | Socket path differs on Docker Desktop for macOS                |
| 1.3 Materialize the app                 | 25         | Typos in heredocs. Distribute as a zip if the class is large.  |
| 1.4 Naive Dockerfile                    | 30         | First SDK pull is slow on shared bandwidth                     |
| 1.5 Layer cache                         | 35         | High value. Do not compress this one.                          |
| 1.6 .dockerignore                       | 20         | —                                                              |
| 1.7 Networking                          | 45         | Host mode differs on Docker Desktop. Say so up front.          |
| 1.8 Volumes                             | 40         | SQL Server startup time. Start the pull early.                 |
| 1.9 Compose                             | 60         | Healthcheck escaping and YAML indentation                      |
| **Module 1 total**                      | **~5.5 h** |                                                                |
| 2.1 Multi-stage                         | 35         | Stale `app-data` volume ownership                              |
| 2.2 Base images                         | 45         | The three-variant loop is slow. Consider two.                  |
| 2.3 Hardening                           | 45         | The Data Protection lesson generates the best discussion       |
| 2.4 Trivy                               | 45         | DB download on restricted networks                             |
| 2.5 ACR provisioning                    | 30         | Subscription permissions. Verify access the day before.        |
| 2.6 Authentication                      | 45         | RBAC propagation lag of a few minutes                          |
| 2.7 Tagging                             | 35         | —                                                              |
| 2.8 Signing                             | 40         | Key Vault RBAC propagation. Sleep before certificate creation. |
| 2.9 Defender                            | 20         | Findings are asynchronous. Pre-push the vulnerable image.      |
| 2.10 Pipeline                           | 60         | Service connection creation requires DevOps project admin      |
| **Module 2 total**                      | **~6.5 h** |                                                                |

**Pre-session requirements to confirm with the client one week ahead:**

- Contributor on a resource group, plus User Access Administrator or Owner scoped to it (RBAC assignment and Key Vault certificate creation both require it).
- Azure DevOps project administrator rights, or a pre-created service connection.
- Registry pull access to `mcr.microsoft.com`, `ghcr.io`, and `docker.io` from the training network.
- 8 GB RAM minimum per workstation, with 4 GB allocated to Docker.
- **Windows students: WSL 2 installed and verified before day one.** Confirm `wsl --list --verbose` reports `VERSION 2`, that Docker Desktop WSL integration is enabled for the distro (or Docker Engine is installed inside it), and that `jq` and `curl` are present. See Lab 0.2b.

## Appendix E — What This Course Did Not Cover

State these explicitly at the close so nobody leaves with a false sense of completeness.

- **Kubernetes.** Pod security standards, network policy, admission control with Ratify or Kyverno, and workload identity are the next module. Everything hardened here maps directly onto a `securityContext`.
- **Runtime threat detection.** Scanning covers known vulnerabilities in packages. It does not detect a compromised container behaving abnormally. That needs Falco, Tetragon, or Defender's sensor.
- **Secrets management.** The lab used `.env` and the `sa` account. Production needs Key Vault with CSI driver or workload identity, and a per-application database login with the minimum required grants. This is the highest-priority remaining item on the LegacyLedger backlog.
- **Windows containers.** A genuine .NET Framework 4.8 monolith cannot be containerized on Linux. It needs Windows Server Core base images and Windows nodes, at roughly 3 GB per image, with a materially different cost and operational profile.
- **Provenance attestation.** The pipeline emits SLSA provenance via `--provenance=true`, but verifying it at deploy time is a separate control not exercised here.
