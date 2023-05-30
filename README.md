# CI with self-hosted runners

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

This project aims to curate and present information on how to set up a
continuous integration (CI) infrastructure with self-hosted runners for an open
source project on GitHub. That is, instead of using only the GitHub-hosted
runners for testing, documentation building etc., we want to run additional CI
jobs on hardware that we control, either on cloud servers or on on-premise
machines. The focus of this effort is the open source research software
[Trixi.jl](https://github.com/trixi-framework/Trixi.jl).

The key goal of this project is to try to balance ease of setup/maintenance with
security. That is, we are no experts in setting up online services and we do not
want to become one. However, since our research software is developed
collaboratively as an open source project, with contributions from many
different people, we require a reasonable level of security against accidental
or deliberate abuse of the self-hosted runners.

The instructions on how to set up GitHub's own self-hosted runners can be found
directly [below](#set-up-ephemeral-github-runners-with-docker-on-ubuntu).
Additionally, _some_ instructions for setting up
self-hosted runners with [Buildkite](https://buildkite.com) can be found in
[`buildkite.md`](buildkite.md). Information on how to contribute to this project
can be found at the [bottom](#license-and-contributing) of this file.

## Set up ephemeral GitHub runners with Docker on Ubuntu

### Install Docker
Following https://docs.docker.com/engine/install/ubuntu/

1. Clean up
   ```shell
   sudo apt-get remove docker docker-engine docker.io containerd runc
   ```
2. Update `apt`
   ```shell
   sudo apt-get update
   sudo apt-get install -y ca-certificates curl gnupg
   ```
3. Add Docker GPG keys
   ```shell
   sudo install -m 0755 -d /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   sudo chmod a+r /etc/apt/keyrings/docker.gpg
   ```
4. Set up Docker repo
   ```shell
   echo \
     "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
     "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```
4. Install Docker
   ```shell
   sudo apt-get update
   sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```
5. [optional] Verify that it works
   ```shell
   sudo docker run hello-world
   ```

All commmands at once:
```shell
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker run hello-world
```

### Enable running Docker as a non-root user (optional)

*Note: Running Docker as non-root user is generally recommended as a means to
mitigate potential vulnerabilities in the daemon and the container runtime.*

The following steps are, however, optional: If you are set on running Docker as
`root`, you can just skip this entire section.

Following https://docs.docker.com/engine/security/rootless/

1. Install additional dependencies:
   ```shell
   sudo apt-get install -y uidmap
   sudo apt-get install -y dbus-user-session
   ```
   If the second command triggers an installation, i.e., if the
   `dbus-user-session` package had not been installed before, you need to
   relogin.
2. Create a non-root user `docker-rootless`:
   ```shell
   sudo adduser \
       --gecos 'Non-privileged user for rootless Docker execution' \
       --shell /bin/bash \
       --disabled-password \
       docker-rootless
   ```
3. Verify that enough subordinate UIDs/GIDs are available by running
   ```shell
   grep ^docker-rootless: /etc/subuid /etc/subgid
   ```
   which should give an output similar to
   ```
   /etc/subuid:docker-rootless:100000:65536
   /etc/subgid:docker-rootless:100000:65536
   ```
   where the final number after the last colon whould be >= 65,536.
4. Enable use of systemd services without being logged in:
   ```shell
   sudo loginctl enable-linger docker-rootless
   ```
5. Allow non-root users to limit all resources using cgroups (by default, only
   `memory` and `pids` are allowed):
   ```shell
   sudo mkdir -p /etc/systemd/system/user@.service.d
   sudo cat > /etc/systemd/system/user@.service.d/delegate.conf << EOF
   [Service]
   Delegate=cpu cpuset io memory pids
   EOF
   systemctl daemon-reload
   ```
5. Disable system-wide Docker daemon:
   ```shell
   sudo systemctl disable --now docker.service docker.socket
   ```
6. Switch to a login shell for the `docker-rootless` user,
   ```shell
   sudo su - docker-rootless
   ```
   then add your SSH key for key-based authentication:
   ```shell
   mkdir $HOME/.ssh
   echo 'YOUR_KEY_GOES_HERE' >> $HOME/.ssh/authorized_keys
   # e.g., echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIE1hyJ7bPlchRiH5x/2T5S/66CjWaqPSvoO2VIiLp//c hpcschlo@hlrs-hpcschlo' >> $HOME/.ssh/authorized_keys
   ```

Now log out from the system and re-login as `docker-rootless`. The next steps
are performed as `docker-rootless`.

*Note: It is insufficient to just switch to the `docker-rootless` user via `su`,
you really need to login via ssh.*

1. Install rootless Docker for user `docker-rootless`:
   ```shell
   dockerd-rootless-setuptool.sh install
   ```
2. The command will give you some hints about environment variables that need to
   be added to your `~/.bashrc`, you can do that by running
   ```shell
   echo 'export PATH=/usr/bin:$PATH' >> ~/.bashrc
   echo 'export DOCKER_HOST=unix:///run/user/1000/docker.sock' >> ~/.bashrc
   ```
   (but please check if the user id matches).
3. Enable the systemd service to launch the Docker daemon at startup:
   ```shell
   systemctl --user enable docker
   ```

### Create GitHub App for runner management

Note: This step needs to be done only once.

1. Go to https://github.com/organizations/trixi-framework/settings/apps/new
2. Enter app details:
   * Name: **Self-hosted runner administration**
   * Description: **App to manage self-hosted runners.**
   * Homepage URL: https://github.com/trixi-framework
   * Identifying and authorizing users
     * ❌ Expire user authorization tokens
     * Leave all other checkboxes/fields unchecked/empty as well
   * Post installation: leave everything empty
   * Webhook
     * ❌ Active
     * Leave all other checkboxes/fields unchecked/empty as well
   * Permissions
     * Organization permissions
       * Self-hosted runners: read and write
   * Where can this GitHub App be installed?
     * 🔘 Only on this account
3. Click "Create GitHub App"
4. You will be redirected to an overview of the newly created app. Write down
   the App ID near the top of the page, you will need it later. It looks
   something like `339175`.
5. Scroll down and click "Generate a private key". This will create a key and
   automatically download a file named something like
   `self-hosted-runner-administration.2023-05-26.private-key.pem`. Keep this
   file private, i.e., never commit it to a GitHub repository, do not share it
   with anyone who does not need it etc.
6. Click on "Install App" in the left navigation bar and then press "Install"
   for the current organization.
7. You will land on the configuration page for the app for the current GitHub
   organization. Note the URL - it will read something like
   https://github.com/organizations/trixi-framework/settings/installations/37934927.
   The last part is the "installation id" of the app. Write it down, you will
   need it later.

### Prepare automatic GitHub token generation
When a new runner is started, it needs to be registered with an organization
before it can be used by jobs created in the GitHub Action workflows. For this,
an authentication token is required. We will use the GitHub App created in the
previous step for this purpose.

To get an authentication token, two steps are required:
1. Authenticate with GitHub as the app itself, using the app id noted above
   (e.g., `339175`).
2. Authenticate as concrete app installation (e.g., as the app in the Trixi
   Framework organization), using the installation id noted above (e.g.,
   `37934927`).

Proceed now to install additional dependencies required for the token
generation:
```shell
sudo apt-get install -y python3 python3-pip
```
While the previous step could be done by any user with `sudo` privileges (or
`root` itself), the next steps should be performed as the user under which the
Docker containers are to be run. That is, `root` if you go with a `root`-based
Docker setup, and `docker-rootless` if you use a rootless Docker setup.

1. Create a directory for managing the GitHub authentication:
   ```shell
   mkdir $HOME/github-authentication
   chmod 700 $HOME/github-authentication
   ```
   The directory can only be accessed by our user, protecting all files inside
   from prying eyes.
2. Upload the private app key generated above to the newly folder, e.g., by running
   the following command on your _local_ machine where the key is located:
   ```shell
   scp ~/Downloads/self-hosted-runner-administration.*.private-key.pem USER@HOSTNAME:github-authentication/
   ```
   where `USER` is `root` or `docker-rootless` and `HOSTNAME` is your server's
   hostname or IP address.
3. Log in back to the server and make the key read-only,
   ```shell
   chmod 600 $HOME/github-authentication/*.pem
   ```
   and create a symbolic link to the current key:
   ```shell
   ln -rs $HOME/github-authentication/self-hosted-runner-administration.*.private-key.pem \
       $HOME/github-authentication/self-hosted-runner-administration.current.private-key.pem
   ```
4. Install the Python packages `jwt` and `requests`
   ```shell
   pip install jwt requests
   ```
5. Create a file `$HOME/github-authentication/access_token.py` with the
   following content (loosely based on
   [this](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-json-web-token-jwt-for-a-github-app#example-using-python-to-generate-a-jwt)
   snippet):
   ```python
   #!/usr/bin/env python3
   import json
   import jwt
   import os
   import requests
   import time
   import sys
   
   # Set PEM file path
   pem = os.path.expanduser('~/github-authentication/self-hosted-runner-administration.current.private-key.pem')
   
   # Set the App ID
   app_id = '339175'
   
   # Set the installation ID
   install_id = '37934927'
   
   # Open PEM
   with open(pem, 'rb') as pem_file:
       signing_key = jwt.jwk_from_pem(pem_file.read())
   
   # Create JWT
   payload = {
       # Issued at time (-60 recommended to allow for clock drift)
       'iat': int(time.time()) - 60,
       # JWT expiration time (10 minutes maximum, but we might be 60 seconds in the past)
       'exp': int(time.time()) + 500,
       # GitHub App's identifier
       'iss': app_id
   }
   
   jwt_instance = jwt.JWT()
   encoded_jwt = jwt_instance.encode(payload, signing_key, alg='RS256')
   
   # Create access token
   url = 'https://api.github.com/app/installations/' + install_id + '/access_tokens'
   headers = {
       'Accept': 'application/vnd.github+json',
       'Authorization': 'Bearer ' + encoded_jwt,
       'X-GitHub-Api-Version': '2022-11-28'
   }
   payload = {'permissions': {'organization_self_hosted_runners': 'write'}}
   r = requests.post(url, headers=headers, data=json.dumps(payload))
   print(r.json()['token'])
   ```
   and set it to executable:
   ```shell
   chmod +x $HOME/github-authentication/access_token.py
   ```
6. Verify that the token generation works by executing
   `$HOME/github-authentication/access_token.py`. The output should be
   something like this:
   ```shell
   ghs_jq4Iy0M08WJ0qHFJcvukHjCKrIGhfG0c6u4f
   ```

#### Alternative: create personal access token
*Note: this is not necessary if using the GitHub App based authentication as
described above.*

Go to https://github.com/settings/tokens and create a token with the permission
for `manage_runners:org`.

This is not as secure as the app-based authentication since it allows the
management of runners for all organizations for which the GitHub user has
sufficient permissions.

### Create script for service startup
We will run the GitHub runners inside a Docker container that is automatically
started as a systemd service. For convenience, we will create a script that
holds all the startup commands, including the command to get a fresh access
token.

Create a folder that will hold all relevant files for the Docker setup and
GitHub runner configuration:
```shell
mkdir -p $HOME/github-runner-setup
```
Next, create a file `$HOME/github-runner-setup/start-docker-github-runner.sh` with the following content
```shell
#!/bin/bash

# Store arguments
DOCKER_NAME="$1"
RUNNER_NAME="$2"

# Get access token
ACCESS_TOKEN=$($HOME/github-authentication/access_token.py)

# Start docker
/usr/bin/docker run --rm \
                    --env-file $HOME/github-runner-setup/ephemeral-github-actions-runner.env \
                    -e RUNNER_NAME="$RUNNER_NAME" \
                    -e ACCESS_TOKEN="$ACCESS_TOKEN" \
                    --name "$DOCKER_NAME" \
                    myoung34/github-runner:latest
```
and make it executable with
```shell
chmod +x $HOME/github-runner-setup/start-docker-github-runner.sh
```

### Create service environment file
Following https://github.com/myoung34/docker-github-actions-runner/wiki/Usage#systemd

Create the file `$HOME/github-runner-setup/ephemeral-github-actions-runner.env` with the following
content
```shell
RUNNER_SCOPE=org
ORG_NAME=trixi-framework
LABELS=ephemeral,2core-8gib
DISABLE_AUTO_UPDATE=1
EPHEMERAL=1
DISABLE_AUTOMATIC_DEREGISTRATION=1
```
Here, `LABELS` is a comma-separated list of tags that can be used to configure
which jobs should be run on this runner. This can later be controlled in the
GitHub Actions workflow file through the
[`runs-on`](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idruns-on)
directive.

### Create service definition file for systemd
Following https://github.com/myoung34/docker-github-actions-runner/wiki/Usage#systemd

~the next steps differ depending on whether you plan to run Docker as
`root` or as a non-root user. For simplicity, we will refer to each variants by
the respective usernames, i.e.,
either `root` for root-based Docker or `docker-rootless` for non-root Docker.

Create the file
* `root`: `/etc/systemd/system/ephemeral-github-actions-runner@.service`
* `docker-rootless`: `$HOME/.config/systemd/user/ephemeral-github-actions-runner@.service`

with the following content:
```shell
[Unit]
Description=Ephemeral GitHub Actions Runner Container
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
Restart=always
ExecStartPre=-/usr/bin/docker stop %N
ExecStartPre=-/usr/bin/docker rm %N
ExecStartPre=-/usr/bin/docker pull myoung34/github-runner:latest
ExecStart=$HOME/github-runner-setup/start-docker-github-runner.sh %p-%i %H-%i

[Install]
WantedBy=multi-user.target
```
and set the permissions accordingly:
* `root`: `chmod 644 /etc/systemd/system/ephemeral-github-actions-runner@.service`
* `docker-rootless`: `chmod 644 $HOME/.config/systemd/user/ephemeral-github-actions-runner@.service`

### Enable and control GitHub Actions runner service

#### When running Docker as `root`
Enable the service:
```shell
sudo systemctl daemon-reload
sudo systemctl enable --now ephemeral-github-actions-runner@1
# sudo systemctl enable --now ephemeral-github-actions-runner@2 # to start more than one runner 
```

Start/stop daemon with
```shell
# Run with:
sudo systemctl start ephemeral-github-actions-runner@1
# Stop with:
sudo systemctl stop ephemeral-github-actions-runner@1
```

See logs for a single runner with
```shell
journalctl -f -u ephemeral-github-actions-runner@1 --no-hostname --no-tail
```
or for all runners with
```shell
journalctl -f -u "ephemeral-github-actions-runner@*" --no-hostname --no-tail
```

#### When running Docker as non-root user
Enable the service:
```shell
systemctl --user daemon-reload
systemctl --user enable --now ephemeral-github-actions-runner@1
# systemctl --user enable --now ephemeral-github-actions-runner@2 # to start more than one runner 
```

Start/stop daemon with
```shell
# Run with:
systemctl --user start ephemeral-github-actions-runner@1
# Stop with:
systemctl --user stop ephemeral-github-actions-runner@1
```

See logs for a single runner with
```shell
journalctl --user -f -u ephemeral-github-actions-runner@1 --no-hostname --no-tail
```
or for all runners with
```shell
journalctl --user -f -u "ephemeral-github-actions-runner@*" --no-hostname --no-tail
```

### Verify registration and update runner usage permissions
Go to https://github.com/organizations/trixi-framework/settings/actions/runners.
If everything was successful, you should see your newly created runner in the
list of runners.

Before the runners can be used for running workflows of public repositories, the
permissions of the *Default* runners group need to be updated by going to the
GitHub organization's
[runner group website](https://github.com/organizations/trixi-framework/settings/actions/runner-groups)
and clicking on the "Default" group. There, you need to check "Allow public repositories".

### Use a self-hosted runner
To run a job on one of the self-hosted runners, you need to modify the
[`runs-on`](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idruns-on)
setting in your GitHub workflow configuration file to include one of the
tags that you specified under `LABELS` in the
[service environment file](#create-service-environment-file).


## Authors
This project was initiated by [Michael Schlottke-Lakemper](https://lakemper.eu)
(RWTH Aachen University/High-Performance Computing Center Stuttgart (HLRS),
Germany). It is maintained by the
[Trixi.jl authors](https://github.com/trixi-framework/Trixi.jl/blob/main/AUTHORS.md).

## License and contributing
The contents of this repository are available under the Create Commons
Attribution 4.0 International license ([CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)),
see also [`LICENSE`](LICENSE).
We are very happy to accept contributions from the community.
To get in touch with the developers,
[join us on Slack](https://join.slack.com/t/trixi-framework/shared_invite/zt-sgkc6ppw-6OXJqZAD5SPjBYqLd8MU~g)
or [create an issue](https://github.com/trixi-framework/ci-with-self-hosted-runners/issues/new).
