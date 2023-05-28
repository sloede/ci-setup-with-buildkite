# CI with self-hosted Buildkite runners

Following https://buildkite.com/docs/tutorials/getting-started

## Convert to YAML steps
Following https://buildkite.com/docs/tutorials/pipeline-upgrade#using-yaml-steps-for-new-pipelines

1. Go to Buildkite.com website
2. Go to Settings -> YAML Migration
3. Click "Start using YAML pipelines"

## Installation on Ubuntu

Following https://buildkite.com/docs/agent/v3/installation

From https://buildkite.com/docs/agent/v3/ubuntu

All these instructions are assumed to be done as `root`.

### Preparations
```shell
# Go to folder only writable by root
cd $HOME

# Get PGP key
curl -fsSL https://keys.openpgp.org/vks/v1/by-fingerprint/32A37959C2FA5C3C99EFBC32A79206696452D198 \
         | sudo gpg --dearmor -o /usr/share/keyrings/buildkite-agent-archive-keyring.gpg

# Add to apt sources
echo "deb [signed-by=/usr/share/keyrings/buildkite-agent-archive-keyring.gpg] https://apt.buildkite.com/buildkite-agent stable main" | sudo tee /etc/apt/sources.list.d/buildkite-agent.list
```

### Install agent
```shell
# Install Buildkite agent
sudo apt-get update && sudo apt-get install -y buildkite-agent
```

### Configuration & startup
Get token by navigating to agents page https://buildkite.com/organizations/trixi-framework/agents
and clicking on "Reveal Agent Token".
```shell
# Configure agent token
sudo sed -i "s/xxx/396e83f8433ca62a8b3b3e9a03886ce120f44d6cf51650692e/g" /etc/buildkite-agent/buildkite-agent.cfg

# Start agent
sudo systemctl enable buildkite-agent && sudo systemctl start buildkite-agent
```

To view the logs, execute
```shell
journalctl -f -u buildkite-agent
```

### Adding multiple agents
Following https://buildkite.com/docs/agent/v3/ubuntu#running-multiple-agents:
```shell
# Disable the default unit
sudo systemctl stop buildkite-agent && sudo systemctl disable buildkite-agent

# Create a systemd template
sudo cp /lib/systemd/system/buildkite-agent.service /etc/systemd/system/buildkite-agent@.service

# Now, as many times as you like
sudo systemctl enable --now buildkite-agent@1
sudo systemctl enable --now buildkite-agent@2

# Follow them all
sudo journalctl -f -u "buildkite-agent@*"

# Or one-by-one
sudo journalctl -f -u buildkite-agent@2
```

## Important directories on Ubuntu

From https://buildkite.com/docs/agent/v3/ubuntu#file-locations

* Configuration: `/etc/buildkite-agent/buildkite-agent.cfg`
* Agent Hooks: `/etc/buildkite-agent/hooks/`
* Builds: `/var/lib/buildkite-agent/builds/`
* Logs: `journalctl -f -u buildkite-agent`
* Agent user home: `/var/lib/buildkite-agent/`
* SSH keys: `/var/lib/buildkite-agent/.ssh/`

## Connect to GitHub

Follow steps on https://buildkite.com/docs/integrations/github

## Create CI pipeline for repo and configure it

Following https://buildkite.com/docs/integrations/github#set-up-a-new-pipeline-for-a-github-repository

1. Make sure that the repo you want to add is enabled
   (https://buildkite.com/organizations/trixi-framework/repository-providers)
2. Create a new pipline https://buildkite.com/organizations/trixi-framework/pipelines/new
3. Name pipeline after repo name, e.g., `TrixiTestBuildkite.jl`
4. Enter repo details (make sure that auto-webhooks is enabled and select HTTPS)
5. Add steps from script below

The purpose of the pipeline created in the web interface will be to use the
`pipeline.yml` file from the repo to run the actual pipeline. Also, it will
block runs on forks unless the user is a member of the Trixi Framework GitHub
organization. For this, we will run an "Authorize fork" step if the PR comes
from a fork, then checks if the branch contains a `:` (which indicates a
third-party fork) and then proceeds to check if the user is a member of the
Trixi Framework org.

In the following pipeline, replace `GITHUB_TOKEN` with a personal access token with `read:org`
permissions (https://github.com/settings/tokens):
```yaml
# This default command changes the pipeline of a running build by uploading a configuration file.
# https://buildkite.com/docs/agent/v3/cli-pipeline
# For information on different step types, check out the sidebar to the right.

steps:
  - label: "Authorize fork"
    if: build.pull_request.repository.fork == true
    command: |
        if [[ "$$BUILDKITE_BRANCH" = *:* ]]; then
            echo "PR might be from a third-party fork"
            GITHUB_USER=$${BUILDKITE_BRANCH%%:*}
            echo "Identified user as '$$GITHUB_USER'"
            echo "Checking if $$GITHUB_USER is part of the Trixi Framework organization"
            set +e
            curl -L --fail \
                -H "Accept: application/vnd.github+json" \
                -H "Authorization: Bearer GITHUB_TOKEN"\
                -H "X-GitHub-Api-Version: 2022-11-28" \
                https://api.github.com/orgs/trixi-framework/members/$$GITHUB_USER
            exit_code=$$?
            set -e
            if [[ $$exit_code = 0 ]]; then
                echo "$$GITHUB_USER is a member of Trixi Framework"
                echo "Proceeding directly with build..."
            else
                echo "$$GITHUB_USER is *not* a member of Trixi Framework"
                echo "Wait for manual authorization before proceeding with build..."
                cat <<- YAML | buildkite-agent pipeline upload
                steps:
                  - block: ":rocket: Run on third-party fork?"
                    blocked_state: running
        YAML
            fi
        else
            echo "PR is from trusted source"
        fi
  - wait
  - label: ":pipeline:"
    command: "buildkite-agent pipeline upload"

```

For the configuration, go to "Pipelines", click on the pipeline, then "Pipeline
settings". Here is a number of useful settings to be changed from the default:

### General
* Verify default branch is `main`
* "Make Pipeline Public"

### Builds
* ✅ Skip Intermediate Builds
* ✅ Cancel Intermediate Builds
* Rebuilds: 🔘 Allow rebuilds

### GitHub
* 🔘 Use HTTPS
* Branch Limiting:
  ```
  main
  ```
* GitHub Settings
  * 🔘 Trigger builds after pushing code
    * ❌ Skip builds with existing commits
    * ✅ Build Pull Requests
      * ✅ Skip pull request builds for existing commits
      * ✅ Build pull requests from third-party forked repositories
        * ✅ Prefix third-party fork branch names
    * ✅ Build branches
    * ✅ Cancel deleted branch builds
    * ✅ Update commit statuses
      * Show blocked builds in GitHub as: `Pending`
      * ✅ Create a status for each job

### Schedules
Nothing to do.

### Build Badges
Nothing to do (except maybe copy badge code to README.md).

## Create a pipeline file

1. Clone the repository we want to add Buildkite testing to
2. Create a folder `.buildkite`
3. Add a file `.buildkite/pipeline.yml`
4. Fill pipeline file with code

## Codecov plugin: add `CODECOV_TOKEN` to agent hook
Following https://buildkite.com/docs/pipelines/secrets#exporting-secrets-with-environment-hooks

1. Go to `/etc/buildkite-agent/hooks`
2. Create file `environment` (if not existing) under the `buildkite-agent` user
3. Set `CODECOV_TOKEN` environment variable to value given in Codecov.io
   settings (e.g., https://app.codecov.io/gh/sloede/TrixiTestBuildkite.jl/settings)

Full code:
```shell
cd /etc/buildkite-agent/hooks
su - buildkite-agent -c "touch $PWD/environment; chmod +x $PWD/environment"
su - buildkite-agent -c "vim $PWD/environment"
```

File contents (modify pipeline slug accordingly):
```shell
#!/bin/bash
set -euo pipefail

if [[ "$BUILDKITE_PIPELINE_SLUG" == "trixitestbuildkite-dot-jl" ]]; then
  export CODECOV_TOKEN=xxx-yyy-zzz
fi
```

## Configure for pipelines with Docker plugin

### Install Docker on Ubuntu
Install Docker as described above.

### Install Buildkite agent
See instructions above.

### Update queue configuration
Edit `/etc/buildkite-agent/buildkite-agent.cfg` and set tags to include the
`docker` queue:
```shell
tags="queue=default,queue=docker"
```

## Allow Buildkite agent to run Docker images
Following https://docs.docker.com/engine/install/linux-postinstall/

```shell
sudo groupadd docker
sudo usermod -aG docker buildkite-agent
```

Just in case, restart Buildkite agent:
```shell
sudo systemctl restart buildkite-agent
```

