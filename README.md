# devops-github-actions-runner

This repository contains documentation, tools and scripts for managing our self hosted GitHub Actions runners on Hetzner.

NOTE: Though the repository is public to provide a reference how to setup self hosted GitHub Actions runners on Hetzner, the scripts will need tweaking to fit other organisations (ie. hard coded references to our "mynewsdesk" organisation).

NOTE2: Self hosted runners will only run on private repositories by default. See [Public vs Private repositories](#public-vs-private-repositories) for more information.

## Adding a new runner

### Installing the OS

We install the OS via Hetzner's rescue mode. Rescue mode should be enabled with the `devops-talos-manager` SSH key. Once logged in to the Rescue OS we can use the `installimage` tool to install Ubuntu 22.04 on the server:

```bash
IP=<ip>
RUNNER_NAME=AX52-<n>
ssh root@$IP -i ~/.ssh/devops-talos-manager.pem "/root/.oldroot/nfs/install/installimage -a \
  -n github-actions-runner-$RUNNER_NAME \
  -r no \
  -i root/.oldroot/nfs/install/../images/Ubuntu-2204-jammy-amd64-base.tar.gz \
  -p /boot/efi:esp:256M,swap:swap:31G,/boot:ext3:1024M,/:ext4:all \
  -d nvme0n1,nvme1n1 && reboot"
```

NOTE: Because `installimage` is an alias for `/root/.oldroot/nfs/install/installimage` we need to specify the full path to the image file to run it directly via `ssh`.

### Running the bootstrap script

```bash
IP=<ip>
scp -i ~/.ssh/devops-talos-manager.pem bin/bootstrap root@$IP:
ssh root@$IP -i ~/.ssh/devops-talos-manager.pem "chmod +x bootstrap && time ./bootstrap && reboot"
```

### Installing the GitHub runner agent

The approach for bootstrapping the GitHub runner agents are based on example scripts found at:
https://github.com/actions/runner/tree/8db8bbe13a0dabc165d0ff19a1ecb85a4fe86dd8/scripts

```bash
IP=<ip>
RUNNER_NAME=AX52-<n>
GITHUB_TOKEN=<personal-access-token>
scp -i ~/.ssh/devops-talos-manager.pem bin/install-runner-agent root@$IP:
ssh root@$IP -i ~/.ssh/devops-talos-manager.pem "
  chmod +x install-runner-agent &&
  GITHUB_TOKEN=$GITHUB_TOKEN RUNNER_NAME=$RUNNER_NAME ./install-runner-agent"
```

Installation notes:
- The installer will add a service with the name `actions.runner.mynewsdesk.<$RUNNER_NAME>.service`
- The service configuration is found under `/etc/systemd/system/`
- The script used for running the service is at `/home/runner/actions-runner/runsvc.sh`

## Credentials / Secrets

In our previous CI setup using Buildkite we pre-baked all necessary credentials into the EC2 image. This had a major benefit in that a lot of integrations required 0 config and everything "just worked" out of the box. The downside is that it required quite a bit of manual work to update the image, making credential rotation more painful.

Since Github Actions provide support for secrets management, we decided to try leveraging this during our migration from Buildkite to Github Actions.

### GitHub

Authentication with the GitHub API and git repositories is done via personal access tokens. Reference our [GH_PERSONAL_ACCESS_TOKEN](https://github.com/organizations/mynewsdesk/settings/secrets/actions/GH_PERSONAL_ACCESS_TOKEN) organisation secret for complete access to all our private repositories.

When using the pre-installed `gh` CLI tool you can set the `GH_TOKEN` environment variable and it should just work.

To pull/push GitHub repositories you can use git URL's in the format of `https://$GH_TOKEN@github.com/mynewsdesk/<repository>.git`.

If you want to use repository dispatch in GitHub actions you can use something like:

```yaml
      - uses: peter-evans/repository-dispatch@v2
        with:
          token: ${{ secrets.GH_PERSONAL_ACCESS_TOKEN }}
          event-type: <event-type>
          client-payload: '{ "some": "values" }'
```

### Docker Registry

The `docker` CLI is pre-installed on the runner. You can use the [docker/login-action](https://github.com/docker/login-action) to authenticate `docker` with a private Docker registry. To authenticate with the Google Artifact Registry we use for storing images for our Kubernetes cluster use:

```yaml
      - uses: docker/login-action@v2
        with:
          registry: ${{ vars.K_DOCKER_REGISTRY }}
          username: _json_key
          password: ${{ secrets.K_DOCKER_REGISTRY_JSON_KEY }}
```

### k

The `k` CLI is pre-installed on the runner. Configuration can be done via the official [k-action](https://github.com/reclaim-the-stack/k-action) action. You can use eg. our `GITOPS_REPOSITORY_URL`, `KUBE_CONFIG`, `K_DOCKER_REGISTRY` and `K_DOCKER_NAMESPACE` organisation secrets to configure `k`:

```yaml
      - uses: reclaim-the-stack/k-action@master
        with:
          gitops-repository-url: ${{ secrets.GITOPS_REPOSITORY_URL }}
          kube-config: ${{ secrets.KUBE_CONFIG }}
          registry: ${{ vars.K_DOCKER_REGISTRY }}
          registry-namespace: ${{ vars.K_DOCKER_NAMESPACE }}
```

NOTE: For docker registry integration to work you need to use the `docker/login-action` action as described above as well.

### Kubectl

Note: if you're using the `k-action` action, that will implicitly configure `kubectl` as well.

The `kubectl` CLI is pre-installed on the runner. Configuration and authentication is normally handled via the `~/.kube/config` configuration file. Most GitHub actions dealing with Kubernetes allow passing in a complete kubeconfig as a base64 encoded string. You can reference our [KUBE_CONFIG](https://github.com/organizations/mynewsdesk/settings/secrets/actions/KUBE_CONFIG) organisation secret for this purpose.

### Heroku

The `heroku` CLI is pre-installed on the runner. Set the `HEROKU_API_KEY` and `HEROKU_ORGANIZATION` environment variables and `heroku` should just work. `HEROKU_ORGANIZATION` should be set to `mynewsdesk` and `HEROKU_API_KEY` can be set to our [HEROKU_API_KEY](https://github.com/organizations/mynewsdesk/settings/secrets/actions/HEROKU_API_KEY) organisation secret.

If you just want to git pull/push to a Heroku git remote you can use an url in the format of `https://heroku:$HEROKU_API_KEY@git.heroku.com/<app-name>.git`.

### AWS

The `aws` CLI is pre-installed on the runner. For authentication put the contents of `AWS_CLI_CREDENTIALS` into `~/.aws/credentials`. This gives you access to `dev`, `staging` and `prod` AWS profiles with full access to their respective AWS accounts, eg:

```yaml
      - name: Prepare AWS CLI credentials
        run: |
          mkdir -p ~/.aws
          echo "${{ secrets.AWS_CLI_CREDENTIALS }}" > ~/.aws/credentials
```

## Public vs Private repositories

The runners are configured to only run workflows from private repositories by default to adhere to GitHub's recommended security best practices.

For public repositories (and forks of public repositories), GitHub's own runners can be used instead. An example of a forked repository using GitHub's own runners is [mynewsdesk/omniauth-redirect-proxy](https://github.com/mynewsdesk/omniauth-redirect-proxy/blob/master/.github/workflows/).
