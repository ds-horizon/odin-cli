# Odin CLI

[![Go Version](https://img.shields.io/badge/go-1.22+-00ADD8?style=flat-square)](https://go.dev/dl/)
[![License: LGPL v3](https://img.shields.io/badge/license-LGPLv3-blue.svg)](./LICENSE)

## Table of Contents
- [Overview](#overview)
- [Features](#features)
- [Installation](#installation)
  - [Prerequisites](#prerequisites)
  - [Install with Go](#install-with-go)
  - [Build from Source](#build-from-source)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Usage](#usage)
  - [Global Flags](#global-flags)
  - [Command Catalog](#command-catalog)
- [Output Formats](#output-formats)
- [CI Status & Releases](#ci-status--releases)
- [Development](#development)
- [Contributing](#contributing)
- [License](#license)

## Overview
Odin CLI is the command-line interface for managing services, environments, and deployments on the Odin identity and deployment platform. It wraps Odin's gRPC APIs in an ergonomic CLI experience with simple configuration, rich status output, and automation-friendly JSON responses.

## Features
- Authenticate against the Odin backend and manage multiple profiles.
- Create, list, describe, and delete environments across cloud provider accounts.
- Deploy, operate, and undeploy services with safe guards for production workloads.
- Inspect environment status, including service and component level insights.
- Toggle detailed logging and export machine-readable output for automation.

## Installation

### Prerequisites
- Go **1.22** or higher (see `go.mod`).
- `protoc` when you need to regenerate protobuf stubs (the generated code is already committed).
- Access to an Odin backend endpoint and credentials issued by your organization.

### Install with Go
```bash
go install github.com/dream11/odin@latest
```
This installs an `odin` binary in your `$GOBIN` (defaults to `$HOME/go/bin`).

### Build from Source
```bash
git clone https://github.com/dream11/odin.git
cd odin/odin-cli
go build ./...
```
For reproducible artifacts across platforms you can use the provided Makefile:
```bash
make build        # build macOS (amd64/arm64) and linux (amd64) binaries under ./bin
sudo make install # optional helper that builds, installs to /usr/local/bin, and verifies the version
```

## Quick Start
1. Configure Odin with your backend endpoint, organization, and credentials:
   ```bash
   odin configure --backend-address <host:port> --org-id <id>
   ```
2. Create an environment bound to one or more cloud provider accounts:
   ```bash
   odin create env staging --accounts aws/dev,aws/shared
   ```
3. Deploy a service definition and provisioning plan:
   ```bash
   odin deploy service --env staging --file service.json --provisioning provisioning.json
   ```
4. Inspect the health of your environment or a specific service:
   ```bash
   odin status env staging
   odin status env staging --service my-service
   ```

## Configuration
- Odin stores configuration per profile in `~/.odin/config` (TOML).
- `odin configure` populates base settings (backend address, org ID, tokens) and initiates interactive authentication.
- Switch between profiles with `odin set profile <profile>`; the active profile is shared across commands unless overridden with `--profile`.
- Remember to assign a default environment per profile using `odin set env <env-name>`; commands that accept `--env` fall back to this value.
- Environment variables provide another configuration layer and follow the `ODIN_` prefix. Key variables include:
  - `ODIN_BACKEND_ADDRESS`
  - `ODIN_ORG_ID`
  - `ODIN_LOG_LEVEL`

## Usage

### Global Flags
All commands inherit the following flags from the root `odin` command:

| Flag | Default | Description |
|------|---------|-------------|
| `--profile`, `-p` | `default` | Selects the config profile stored in `~/.odin/config`. |
| `--output`, `-o`  | `text`    | Controls command output (`text` or `json`). |
| `--verbose`, `-v` | `false`   | Enables verbose logging and richer diagnostic output. |

### Command Catalog

<details open>
<summary><code>odin configure</code></summary>

Configure Odin locally and authenticate with the backend.

| Flag | Description |
|------|-------------|
| `--backend-address` | Required when no value is present in env/config. Host:port of the Odin backend. |
| `--org-id` | Required when not set elsewhere. Numeric organization identifier. |
| `--insecure`, `-I` | Skip TLS host verification (defaults to `true` for local/staging). |
| `--plaintext`, `-P` | Use plaintext gRPC without TLS. |

Environment variables with the `ODIN_` prefix take precedence over config for these values. On success, an access token is stored alongside the active profile.

</details>

<details>
<summary><code>odin create env &lt;name&gt;</code></summary>

Create a new environment tied to one or more cloud provider accounts.

| Flag | Description |
|------|-------------|
| `--accounts` | Required. Comma-separated list (`aws/dev,aws/shared`) designating provider accounts. |

</details>

<details>
<summary><code>odin delete env &lt;name&gt;</code></summary>

Delete an existing environment.

</details>

<details>
<summary><code>odin deploy service</code></summary>

Deploy a service into an environment using a service definition and provisioning config.

| Flag | Description |
|------|-------------|
| `--env` | Target environment. Falls back to the default environment configured via `odin set env`. |
| `--file` | Required. Path to the service definition JSON file. |
| `--provisioning` | Required. Path to the provisioning JSON file (component provisioning config). |

Both `--file` and `--provisioning` must be supplied together.

</details>

<details>
<summary><code>odin describe env &lt;name&gt;</code></summary>

Fetch detailed information about an environment, optionally filtered by service/component.

| Flag | Description |
|------|-------------|
| `--service` | Filter output down to a specific service deployed in the environment. |
| `--component` | Requires `--service`. Filters the response to a specific component. |

Use `--output json` to retrieve machine-readable environment descriptors.

</details>

<details>
<summary><code>odin list env</code></summary>

List environments visible to the authenticated user or organization.

| Flag | Description |
|------|-------------|
| `--account` | Filter environments associated with a specific provider account. |
| `--all`, `-A` | Include environments outside the requesting user's ownership (requires access). |

</details>

<details>
<summary><code>odin operate service</code></summary>

Run lifecycle operations (e.g., rollout, pause) against a service.

| Flag | Description |
|------|-------------|
| `--name` | Required. Service name. |
| `--env` | Target environment; defaults to the profile's environment if omitted. |
| `--operation` | Required. Operation identifier as defined by Odin (for example `redeploy`). |
| `--options` | Optional JSON string containing operation parameters. Mutually exclusive with `--file`. |
| `--file` | Optional path to a JSON file providing operation parameters. |

When the operation is `redeploy`, Odin shows a diff of pending changes and asks for interactive confirmation.

</details>

<details>
<summary><code>odin operate component</code></summary>

Execute component-level operations within a service.

| Flag | Description |
|------|-------------|
| `--name` | Required. Component name. |
| `--service` | Required. Service that hosts the component. |
| `--env` | Environment; defaults to the profile's environment if unset. |
| `--operation` | Required. Operation identifier. |
| `--options` / `--file` | Same semantics as `odin operate service`; provide either inline JSON or a file. |

</details>

<details>
<summary><code>odin set env &lt;name&gt;</code></summary>

Persist a default environment for the active profile (`~/.odin/config`). Subsequent commands without `--env` reference this value.

</details>

<details>
<summary><code>odin set profile &lt;profile&gt;</code></summary>

Switch the active profile. Profiles must already exist in the config file (created via `odin configure`).

</details>

<details>
<summary><code>odin status env &lt;env-name&gt;</code></summary>

Inspect the deployment status of an environment and optionally narrow it to a service.

| Flag | Description |
|------|-------------|
| `--service` | When supplied, returns component-level status for the given service. |

Outputs a human-readable table by default and can emit JSON with `--output json`.

</details>

<details>
<summary><code>odin undeploy service &lt;name&gt;</code></summary>

Remove a service from an environment.

| Flag | Description |
|------|-------------|
| `--env` | Required. Target environment. Defaults to the profile's environment when omitted. |

Undeploying from `prod` requires typing `PROD` to confirm, adding a safety net for critical environments.

</details>

<details>
<summary><code>odin version</code></summary>

Print the CLI version (`app.App.Version`).

</details>

## Output Formats
Most commands support `--output text` (default) and `--output json`. JSON output is designed for scripting and automation; text output favors human readability with aligned tables.

Verbose logging (`--verbose`) augments command execution with trace IDs and diagnostic messages. Advanced users can also tune `ODIN_LOG_LEVEL`.

## CI Status & Releases
- **Continuous Integration:** Track the latest lint and build pipeline results from GitHub Actions.  
  [![CI Status](https://github.com/dream11/odin/actions/workflows/go-lint.yaml/badge.svg)](https://github.com/dream11/odin/actions/workflows/go-lint.yaml)
- **Latest Release:** Download the most recent Odin CLI binaries and changelog.  
  [![GitHub Release](https://img.shields.io/github/v/release/dream11/odin?display_name=tag&sort=semver&style=flat-square)](https://github.com/dream11/odin/releases)

## Development
- Run `go test ./...` to execute unit tests.
- Use `make build` or `go build ./cmd/...` to produce binaries.
- Protobuf definitions live under `proto/`; run `make install` or invoke `protoc` manually to regenerate gRPC stubs when the proto files change.
- Odin relies on gRPC; ensure the backend endpoint is reachable (VPN, network access, etc.) before running commands that contact the service.

## Contributing
Issues and pull requests are welcome! Please:
- Fork the repository and create topic branches for your changes.
- Format code using the standard Go toolchain (`gofmt`) and run `go test ./...`.
- Document new flags or commands in this README.
- Align commit messages with the conventional style used in the project.

## License
Distributed under the [GNU Lesser General Public License v3.0](./LICENSE).
