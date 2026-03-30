# Krypton.CICD

Shared composite GitHub Actions for Krypton projects.

## Actions

| Action | Path | Purpose |
|---|---|---|
| [CodeQL Vulnerability Scan](#codeql-vulnerability-scan) | `actions/codeql-scan` | Static analysis via GitHub CodeQL |
| [Build and Scan Docker Image](#build-and-scan-docker-image) | `actions/run-trivy-image-scan` | Docker image vulnerability scan via Trivy |
| [Push Docker Image to Registry](#push-docker-image-to-registry) | `actions/docker-push` | Push a built image to a container registry |

---

## CodeQL Vulnerability Scan

Initialises CodeQL, builds the project, runs queries, and uploads SARIF results to the GitHub Security tab. Supports flexible report delivery via artifact upload, email, and/or console output.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `language` | ✅ | — | CodeQL language to analyse. Supported: `c-cpp` \| `csharp` \| `go` \| `java-kotlin` \| `javascript-typescript` \| `python` \| `ruby` \| `swift` |
| `query-suite` | | `security-extended` | Query suite: `security-extended` \| `security-and-quality` \| _(empty for default)_ |
| `exclude-paths` | | `""` | Comma-separated glob patterns to exclude. e.g. `src/**/test/*, **/migrations/*` |
| `continue-on-error` | | `false` | Continue workflow even if findings are detected |
| `build-command` | | `""` | Custom build command instead of CodeQL autobuild. e.g. `dotnet build --configuration Release /p:GenerateOpenApi=false`. Leave empty to use autobuild. Not needed for interpreted languages (JS, Python, Ruby) |
| `report-mode` | | `artifact` | Delivery modes — see table below |
| `retention-days` | | `30` | Artifact retention days (used when mode includes `artifact` or `all`) |
| `notify-email` | | `""` | Comma-separated recipient addresses (required for `email` / `all`) |
| `smtp-server` | | `smtp.gmail.com` | SMTP hostname |
| `smtp-port` | | `587` | SMTP port (`465` SSL, `587` TLS) |
| `smtp-username` | | `""` | SMTP username — pass via a secret |
| `smtp-password` | | `""` | SMTP password / app password — pass via a secret |

### Report Mode Options

| `report-mode` Value | Artifact | Console Log | Email |
|---|:---:|:---:|:---:|
| `artifact` _(default)_ | ✅ | | |
| `console` | | ✅ | |
| `email` | | | ✅ |
| `artifact,console` | ✅ | ✅ | |
| `artifact,email` | ✅ | | ✅ |
| `email,console` | | ✅ | ✅ |
| `all` | ✅ | ✅ | ✅ |
| `none` | | | |

> The GitHub Security tab always receives results regardless of `report-mode`.  
> The workflow run summary page is always written regardless of `report-mode`.

### Outputs

| Output | Description |
|---|---|
| `sarif-path` | Path to the SARIF results directory on the runner |

### Caller Example — Console

```yaml
name: CodeQL Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'   # every Monday at 02:00 UTC

permissions:
  actions: read
  contents: read
  security-events: write   # required for SARIF upload to Security tab

jobs:
  codeql:
    name: CodeQL (${{ matrix.language }})
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        language: [csharp, javascript-typescript]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: CodeQL Scan
        uses: your-org/Krypton.CICD/actions/codeql-scan@main
        with:
          language: ${{ matrix.language }}
          report-mode: "console"
          exclude-paths: "src/**/test/*, **/migrations/*"
```

### Caller Example — Custom Build Command

> Use `build-command` when autobuild cannot detect your build system or when you need specific build flags.
> For C#, always pass `--no-restore` if restore has already run in a prior step.

```yaml
      - name: CodeQL Scan
        uses: your-org/Krypton.CICD/actions/codeql-scan@main
        with:
          language: csharp
          report-mode: "console"
          build-command: "dotnet build --configuration Release --no-restore /p:GenerateOpenApi=false"
          exclude-paths: "**/Tests/**, **/Migrations/**"
```

### Caller Example — Artifact + Email

```yaml
      - name: CodeQL Scan
        uses: your-org/Krypton.CICD/actions/codeql-scan@main
        with:
          language: csharp
          report-mode: "artifact,email"
          build-command: "dotnet build --configuration Release --no-restore /p:GenerateOpenApi=false"
          notify-email: "security@your-org.com"
          smtp-username: ${{ secrets.SMTP_USERNAME }}
          smtp-password: ${{ secrets.SMTP_PASSWORD }}
```

---

## Build and Scan Docker Image

Builds a Docker image and scans it for vulnerabilities using [Trivy](https://github.com/aquasecurity/trivy). Optionally writes a Markdown vulnerability summary to the GitHub Actions job summary page.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `image-name` | ✅ | — | Docker image name, e.g. `docker.io/my-org/my-app` |
| `image-tag` | ✅ | — | Docker image tag |
| `dockerfile-path` | | `.` | Path to the directory containing the Dockerfile |
| `trivy-severity` | | `CRITICAL,HIGH` | Comma-separated severities to scan for |
| `trivy-exit-code` | | `1` | `1` = fail on findings, `0` = never fail |
| `continue-on-error` | | `false` | Continue workflow even if vulnerabilities are found |
| `trivy-ignore-unfixed` | | `true` | Ignore vulnerabilities without an available fix |
| `enable-summary` | | `false` | Write a Markdown vulnerability summary to the job summary page |

### Outputs

| Output | Description |
|---|---|
| `image-ref` | Full image reference (`name:tag`) |

### Caller Example — Console Summary

```yaml
name: Docker Build and Scan

on:
  push:
    branches: [main]

jobs:
  build-and-scan:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build and Scan
        uses: your-org/Krypton.CICD/actions/run-trivy-image-scan@main
        with:
          image-name: docker.io/my-org/my-app
          image-tag: ${{ github.sha }}
          trivy-severity: "CRITICAL,HIGH"
          trivy-exit-code: "1"
          enable-summary: "true"
```

### Caller Example — Warn Only (no failure)

```yaml
      - name: Build and Scan
        uses: your-org/Krypton.CICD/actions/run-trivy-image-scan@main
        with:
          image-name: docker.io/my-org/my-app
          image-tag: ${{ github.sha }}
          trivy-severity: "CRITICAL,HIGH,MEDIUM"
          trivy-exit-code: "0"
          continue-on-error: "true"
          enable-summary: "true"
```

---

## Push Docker Image to Registry

Logs in, tags, and pushes a Docker image to a container registry (defaults to GitLab).

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `image-name` | ✅ | — | Local image name, e.g. `my-org/my-app` |
| `image-tag` | ✅ | — | Image tag to push |
| `registry-url` | | `registry.gitlab.com` | Container registry URL |
| `registry-token` | ✅ | — | PAT for registry authentication |
| `registry-username` | ✅ | — | Registry login username |
| `registry-source` | ✅ | — | Full registry image path, e.g. `registry.gitlab.com/my-org/my-app` |
| `push-latest` | | `true` | Also tag and push as `latest` |

### Outputs

| Output | Description |
|---|---|
| `image-ref` | Full registry image reference that was pushed |

### Caller Example

```yaml
name: Build, Scan and Push

on:
  push:
    branches: [main]

jobs:
  build-scan-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build and Scan
        uses: your-org/Krypton.CICD/actions/run-trivy-image-scan@main
        with:
          image-name: my-org/my-app
          image-tag: ${{ github.sha }}
          enable-summary: "true"

      - name: Push to Registry
        uses: your-org/Krypton.CICD/actions/docker-push@main
        with:
          image-name: my-org/my-app
          image-tag: ${{ github.sha }}
          registry-url: registry.gitlab.com
          registry-username: ${{ secrets.REGISTRY_USERNAME }}
          registry-token: ${{ secrets.REGISTRY_TOKEN }}
          registry-source: registry.gitlab.com/my-org/my-app
```
