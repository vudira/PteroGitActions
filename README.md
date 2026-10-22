# GitHub Actions Runner for Pterodactyl

Run a self-hosted GitHub Actions runner directly from your Pterodactyl panel using GitHub's official [`ghcr.io/actions/actions-runner:latest`](https://github.com/actions/runner/pkgs/container/actions-runner) image.

Import the egg, configure your repository or organization, and start the server. Registration happens during installation, and subsequent starts launch the configured runner automatically. No custom Docker image or manual startup commands are required.

## Features

* **Official runner image** for both installation and runtime.
* **Automatic registration** with a GitHub repository or organization.
* **Non-root runtime**, with runner files assigned to UID/GID `988:988`.
* **Persistent configuration** across normal stops and restarts.
* **Optional runner name, labels, and group** through panel variables.
* **Panel-managed startup and shutdown** using the runner's `run.sh` script.
* **Pterodactyl PTDL v2 egg** ready to import.

## Requirements

* A working Pterodactyl Panel and Wings installation with permission to import eggs and create servers.
* A Linux x64 host for the runner configuration described here.
* Administrative access to the target GitHub repository or organization, or equivalent runner-management permissions.
* A valid GitHub runner registration token.
* Outbound HTTPS access to GitHub, GitHub Container Registry, and any services required by your workflows.
* Enough CPU, memory, and disk space for your workloads.

This egg expects the runtime UID and GID to be `988:988`. Installations using different IDs must adapt the ownership configuration accordingly.

## Installation

### 1\. Generate a registration token

In your GitHub repository or organization, open:

**Settings → Actions → Runners → New self-hosted runner**

Select **Linux** and **x64**, then copy the URL and registration token shown in the configuration command.

Example URLs:

```text
# Repository runner
https://github.com/<owner>/<repository>

# Organization runner
https://github.com/<organization>
```

Registration tokens expire after one hour. Generate the token shortly before installation. This is a runner registration token, not a personal access token.

### 2\. Import the egg

1. Download [`egg-git-hub-actions-runner.json`](./egg-git-hub-actions-runner.json) from this repository.
2. Open the **Pterodactyl Admin Panel**.
3. Navigate to **Nests → Import Egg**.
4. Upload the JSON file and select the destination nest.

### 3\. Create and configure the server

Create a server using the imported egg and configure its startup variables:

|Variable|Required|Description|
|-|-|-|
|`REPO\_URL`|Yes|GitHub repository or organization URL used for registration.|
|`REG\_TOKEN`|Yes|Registration token issued for that repository or organization. Used during installation.|
|`RUNNER\_NAME`|No|Custom runner name. Leave blank to use the egg's generated name.|
|`RUNNER\_LABELS`|No|Additional comma-separated labels, such as `pterodactyl,build`.|
|`RUNNER\_GROUP`|No|Runner group for organization registration. The group must exist and allow access to the intended repositories.|

### 4\. Start the runner

Wait for installation to finish successfully, then click **Start**.

The console should display:

```text
Listening for Jobs
```

Pterodactyl marks the server as **Running** when it detects this message. Confirm that the runner is also online under **Settings → Actions → Runners** on GitHub.

## Using the runner

Add a workflow under `.github/workflows/` in your repository:

```yaml
name: Self-hosted runner check

on:
  workflow\_dispatch:

jobs:
  check:
    runs-on: \[self-hosted, linux, x64]
    steps:
      - name: Verify runner environment
        run: |
          echo "Running on $RUNNER\_NAME"
          uname -a
```

To target runners with a custom label, include it in `runs-on`:

```yaml
runs-on: \[self-hosted, linux, x64, pterodactyl]
```

The runner must match every requested label. For organization runners, repository access must also be permitted by the runner group.

## How it works

### Installation

The install container uses the official runner image and mounts the server's persistent volume at `/mnt/server`. The install script:

1. Removes the previous `actions-runner` directory, if present.
2. Copies the runner bundled at `/home/runner` into `/mnt/server/actions-runner`.
3. Registers the runner using `config.sh --replace --unattended` and the configured variables.
4. Sets directory ownership to `988:988` for the Pterodactyl runtime user.

### Runtime

The persistent volume is mounted at `/home/container`. The egg launches:

```sh
cd /home/container/actions-runner \&\& exec ./run.sh
```

Pterodactyl uses the egg's startup command rather than relying on the image's default `ENTRYPOINT` or `CMD`. Using the same image for installation and runtime keeps the runner environment consistent, while explicit ownership allows the runtime user to access the installed files.

### Stopping and restarting

The egg uses `SIGINT` (`^C`) to request shutdown. Avoid stopping the server while a job is running unless you intend to interrupt it.

Stopping the server does **not** unregister this persistent runner from GitHub. It appears offline until restarted. To permanently remove it, follow GitHub's [runner removal instructions](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/remove-runners).

## Changing the configuration

Registration happens during **installation**, not on every startup. Changing the repository URL, runner name, labels, or group in the panel requires a reinstall to apply.

1. Wait for active jobs to finish and stop the server.
2. Update the relevant startup variables.
3. Generate a fresh `REG\_TOKEN` for the target repository or organization.
4. Open **Admin → Server → Manage → Reinstall**.
5. Wait for installation to complete, then start the server.

> \*\*Reinstallation deletes and recreates the `actions-runner` directory\*\*, including its local configuration, workspace, and any files stored inside it. Back up anything you need first.

An expired registration token does not prevent an already-configured runner from restarting. A valid token is needed when registering again. If you change the runner name or registration target, check GitHub for an old runner entry and remove it if necessary.

## Workflow dependencies

The official image provides the runner environment. Your workflows may still need additional tools, SDKs, or system packages.

* Use appropriate setup actions for supported languages and toolchains.
* For persistent system-level additions, build a derived image and keep the installation and runtime images aligned.
* Do not assume the Pterodactyl runtime UID has passwordless `sudo`, even if another user inside the image does.

`./bin/installdependencies.sh` installs the runner's own Linux dependencies. It is not a general installer for workflow tools.

Docker-based actions, job containers, and service containers require a separately configured Docker environment. Importing this egg alone does not provide one.

## Troubleshooting

|Problem|What to check|
|-|-|
|Registration fails during `config.sh`|Generate a fresh token, verify that it was issued for `REPO\_URL`, and check network access and organization policies. Then reinstall.|
|Server immediately returns to **Offline**|Inspect the installation log and console. Confirm that installation succeeded and `/home/container/actions-runner/run.sh` exists.|
|`Permission denied` in `actions-runner` or `\_work`|Verify that the runtime UID/GID matches the install script's `988:988` ownership. Adapt the configuration if necessary, then reinstall.|
|Runner is online but jobs remain queued|Check all `runs-on` labels, runner availability, and runner-group repository access. Label changes through the egg require reinstalling.|
|Workflow cannot find a command|Install or configure the required toolchain. The runner image does not include every build tool.|
|Old runner remains listed after reconfiguration|Remove the obsolete entry from the previous repository or organization.|

Runner diagnostic logs are available in `actions-runner/\_diag`.

## Security

Workflows execute code inside your runner environment. Restrict runner access to trusted repositories and workflows, protect the runner's credentials and persistent files, and review the risks before allowing untrusted pull requests to use it. Containerization alone does not make untrusted workflows safe.

The `latest` image tag can change. For controlled deployments, use a tested image tag or digest consistently for both installation and runtime.

## Maintainer

This repository is maintained by [@Wolvarik](https://github.com/Wolvarik), Manager of [**Vudira Gaming**](https://vudira.com).

For questions, feedback, or support, contact our community at [hey@vudira.com](mailto:hey@vudira.com) or join our [Discord server](https://discord.vudira.com).

## License

This project is licensed under the **GNU General Public License v3.0 (GPLv3)**. See the [LICENSE](./LICENSE) file for the full terms.

