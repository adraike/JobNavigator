# JobNavigator Docker installation

Installed on October 3, 2026 from https://github.com/vesaias/JobNavigator.
Source revision: `b06fb31d6f0ab4a9e9a4415be7b8e754508d0130`.

## Installation verification

All four containers started successfully. PostgreSQL passed its readiness check; the backend initialized and seeded its database and started its scheduler. The dashboard, `/health`, and `/openapi.json` returned HTTP 200. `/health` reported `status: ok`, service `JobNavigator`, version `2.1.0`. These checks verify startup; AI and third-party integrations require your configuration.

## Access and configuration

Open http://localhost:8080. Port 8080 was selected after scanning Windows TCP listeners and successfully binding a temporary listener to it. The existing application on port 3000 remains running.

`compose.yaml` defines the isolated `jobnavigator` Compose project, PostgreSQL, backend, frontend, and Caddy. Only Caddy publishes a host port, bound to `127.0.0.1`. Database and backend ports remain internal. Containers restart automatically when Docker starts.

The local `.env` contains a generated database password and `JOBNAVIGATOR_PORT=8080`. Keep it private and preserve it for reinstalls using the same database. The dashboard initially accepts a blank sign-in key; set a key in Settings > Advanced, then configure your provider in Settings > AI. Provider keys and optional integrations are not configured by this installation.

## Start or reinstall on this computer

Prerequisites: Git, Docker Desktop running with Linux containers, and Docker Compose.

In PowerShell:

```powershell
Set-Location 'C:\Users\araike\OneDrive\Documents\GitHub\Vibe-Trading\JobNavigator'
docker compose -f compose.yaml config --quiet
docker compose -f compose.yaml pull
docker compose -f compose.yaml up -d --no-build
docker compose -f compose.yaml ps
Invoke-RestMethod http://localhost:8080/health
```

Always specify `-f compose.yaml`; the repository also contains its original `docker-compose.yml` with different port mappings.

## Install on another computer

Clone the repository, then copy this `compose.yaml` into its root:

```powershell
git clone https://github.com/vesaias/JobNavigator.git
Set-Location JobNavigator
Copy-Item .env.example .env
$databasePassword = [Guid]::NewGuid().ToString('N') + [Guid]::NewGuid().ToString('N')
Add-Content .env "`nPOSTGRES_PASSWORD=$databasePassword"
$occupiedPorts = @(Get-NetTCPConnection -State Listen | Select-Object -ExpandProperty LocalPort)
$chosenPort = $null
foreach ($candidate in 8080..8099) {
    if ($occupiedPorts -contains $candidate) { continue }
    $probe = [System.Net.Sockets.TcpListener]::new([System.Net.IPAddress]::Loopback, $candidate)
    try { $probe.Start(); $chosenPort = $candidate; break }
    catch { }
    finally { $probe.Stop() }
}
if ($null -eq $chosenPort) { throw 'No available port in 8080-8099' }
Add-Content .env "JOBNAVIGATOR_PORT=$chosenPort"
docker compose -f compose.yaml config --quiet
docker compose -f compose.yaml pull
docker compose -f compose.yaml up -d --no-build
Start-Process "http://localhost:$chosenPort"
```

Compose overrides the template's DATABASE_URL using POSTGRES_PASSWORD. Do not change the database password for an existing PostgreSQL volume without also changing the database user's password.

## Operations

```powershell
# Inspect status and recent output
docker compose -f compose.yaml ps
docker compose -f compose.yaml logs --tail 100

# Stop while retaining data
docker compose -f compose.yaml down

# Start again
docker compose -f compose.yaml up -d --no-build

# Build the checked-out source instead of using published images
docker compose -f compose.yaml up -d --build --pull never
```

The default image tag is `latest`. For a consistent release, set `JN_IMAGE_TAG` in `.env` to an available upstream release tag before pulling. Backend source is mounted from `./backend`, following upstream's configuration; keep the checkout and image version compatible. To update, back up first, review upstream changes, update the checkout, pull images, and run `up -d --no-build`.

PostgreSQL persists in `jobnavigator_pgdata`. CLI logins persist in the `codex_auth` and `antigravity_auth` named volumes. Scheduled application backups are stored in `./backups`; they contain credentials. Preserve the database volume, backups, `.env`, and any custom backend templates when moving the installation. Ordinary `down` preserves named volumes; `down -v` deletes them and must only be used when you intend to erase data.

If the selected port becomes occupied, choose another free port, change `JOBNAVIGATOR_PORT` in `.env`, then run `up -d --no-build`. For startup failures, check Docker Desktop, `ps`, and backend/database logs. The health endpoint should return HTTP 200 once initialization finishes.

Upstream setup and first-use walkthrough: [README](https://github.com/vesaias/JobNavigator#quick-start) and [Getting Started](https://github.com/vesaias/JobNavigator/blob/main/docs/GETTING-STARTED.md).
