# CI with self-hosted runners

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

This project aims to curate and present information on how to set up a
continuous integration (CI) infrastructure with self-hosted runners for an open
source project on GitHub. That is, instead of using only the GitHub-hosted
runners for testing, documentation building etc., we want to run additional CI
jobs on hardware what we control, either on cloud servers or on on-premise
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

### Create GitHub App for runner management

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

Until now, all steps on the server have been done user-agnostic, i.e., it could
have been down by any user with `sudo` permissions. From now on, we will assume
everything is done as the `root` user. This is not the best approach, but it
makes this setup easier and can be refined later (make it work, make it nice).

1. Switch to the `root` user:
   ```shell
   sudo su -
   ```
2. Create a directory for managing the GitHub authentication:
   ```shell
   cd $HOME
   mkdir github-authentication
   chmod 700 github-authentication
   ```
   The directory can only be accessed by `root`, protecting all files inside
   from prying eyes.
3. Upload the private app key generated above to the folder, e.g., by running
   the following command on your _local_ machine where the key is located:
   ```shell
   scp ~/Downloads/self-hosted-runner-administration.*.private-key.pem root@195.201.31.129:/root/github-authentication/
   ```
   then log in back to the server and make the key read-only,
   ```shell
   chmod 600 /root/github-authentication/*.pem
   ```
   and create a symbolic link to the current key:
   ```shell
   ln -rs $HOME/github-authentication/self-hosted-runner-administration.*.private-key.pem \
       $HOME/github-authentication/self-hosted-runner-administration.current.private-key.pem
   ```
4. Install the Python and the `pip` package:
   ```shell
   sudo apt-get install -y python3 python3-pip
   ```
5. Install the Python packages `jwt` and `requests`
   ```shell
   pip install jwt requests
   ```
6. Create a file `$HOME/github-authentication/access_token.py` with the
   following content (loosely based on
   [this](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-json-web-token-jwt-for-a-github-app#example-using-python-to-generate-a-jwt)
   snippet):
   ```python
   #!/usr/bin/env python3
   import json
   import jwt
   import requests
   import time
   import sys
   
   # Set PEM file path
   pem = '/root/github-authentication/self-hosted-runner-administration.current.private-key.pem'
   
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
7. Verify that the token generation works by executing
   `$HOME/github-authentication/access_token.py` as root. The output should be
   something like this:
   ```shell
   ghs_jq4Iy0M08WJ0qHFJcvukHjCKrIGhfG0c6u4f
   ```

#### [obsolete] Create a personal access token
**Note: this is not necessary if using the GitHub App based authentication as
described above.**

According to the Docker runner
[wiki](https://github.com/myoung34/docker-github-actions-runner/wiki/Usage#token-scope),
the following permissions are required:
* `repo (all)`
* `workflow`
* `admin:org (all)` (mandatory for organization-wide runner)
* `admin:public_key` - `read:public_key`
* `admin:repo_hook` - `read:repo_hook`
* `admin:org_hook`
* `notifications`
Go to https://github.com/settings/tokens and create the token.

### Create script for service startup
We will run the GitHub runners inside a Docker container that is automatically
started as a systemd service. For convenience, we will create a script that
holds all the startup commands, including the command to get a fresh access
token.

Create a file `$HOME/start-docker-github-runner.sh` with the following content
```shell
#!/bin/bash

# Store arguments
DOCKER_NAME="$1"
RUNNER_NAME="$2"

# Get access token
ACCESS_TOKEN=$(/root/github-authentication/access_token.py)

# Start docker
/usr/bin/docker run --rm \
                    --env-file /etc/ephemeral-github-actions-runner.env \
                    -e RUNNER_NAME="$RUNNER_NAME" \
                    -e ACCESS_TOKEN="$ACCESS_TOKEN" \
                    --name "$DOCKER_NAME" \
                    myoung34/github-runner:latest
```
and make it executable with
```shell
chmod +x $HOME/start-docker-github-runner.sh
```

### Create service environment file
Following https://github.com/myoung34/docker-github-actions-runner/wiki/Usage#systemd

Create the file `/etc/ephemeral-github-actions-runner.env` with the following
content
```shell
RUNNER_SCOPE=org
ORG_NAME=trixi-framework
LABELS=ephemeral,2core-8gib
DISABLE_AUTO_UPDATE=1
EPHEMERAL=1
DISABLE_AUTOMATIC_DEREGISTRATION=1
```
and make it only editable by `root`:
```shell
chmod 644  /etc/ephemeral-github-actions-runner.env
```

### Create service definition file for systemd
Following https://github.com/myoung34/docker-github-actions-runner/wiki/Usage#systemd

Create the file `/etc/systemd/system/ephemeral-github-actions-runner@.service`
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
ExecStart=/root/start-docker-github-runner.sh %p-%i %H-%i

[Install]
WantedBy=multi-user.target
```

Set permissions and enable the service:
```shell
chmod 644 /etc/systemd/system/ephemeral-github-actions-runner@.service 
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

### Verification of runner registration
Go to https://github.com/organizations/trixi-framework/settings/actions/runners.
If everything was successful, you should see your newly created runner in the
list of runners.

### Using a self-hosted runner
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
