# Jenkins DevOps Toolkit

The project's goal is to provide a ready-made, easily-modifiable DevOps toolkit in a Docker container. The container toolkit includes the latest copies of Jenkins, Jenkins plugins, and the most common DevOps tools frequently used with Jenkins. These DevOps tools include Git, AWS CLI, Terraform, Packer, Python, Docker, Docker Compose, cURL, and jq. The container is designed to be a short-lived, stood up, used for CI/CD, and torn down, and is ideal for the Cloud.

![Jenkins UI Preview](pics/jenkins_startup.png)

![Jenkins UI Preview](pics/jenkins_preview2.png)

The `Jenkins DevOps Toolkit` image is based on the latest [`jenkins/jenkins:latest`](https://hub.docker.com/r/jenkins/jenkins/) Docker image. The Jenkins Docker image is based on [Debian GNU/Linux 9 (stretch)](https://wiki.debian.org/DebianStretch).

## Installed Tools

Based on latest packages as of 2018.04.19:

- [AWS CLI](https://aws.amazon.com/cli/) v1.15.4
- [Docker CE](https://docker.com/) v18.03.0-ce
- [Docker Compose](https://docs.docker.com/compose/) v1.21.0
- [Git](https://git-scm.com/) v2.11.0
- [HashiCorp Packer](https://www.packer.io/) v1.2.2
- [HashiCorp Terraform](https://www.terraform.io/) v0.11.7
- [Jenkins](https://jenkins.io/) v2.116
- [jq](https://stedolan.github.io/jq/) v1.5.1
- [OpenNTPD](http://www.openntpd.org/) (time sync)
- [pip3](https://pip.pypa.io/en/stable/#) v9.0.1
- [Python3](https://www.python.org/) v3.5.3
- [tzdata](https://www.iana.org/time-zones) (time sync)

Built Output

```text
*** INSTALLED SOFTWARE VERSIONS ***

PRETTY_NAME="Debian GNU/Linux 9 (stretch)"
NAME="Debian GNU/Linux"
VERSION_ID="9"
VERSION="9 (stretch)"
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"

Python 3.5.3
Docker version 18.03.0-ce, build 0520e24
docker-compose version 1.21.0, build 5920eb0
docker-py version: 3.2.1
CPython version: 3.6.5
OpenSSL version: OpenSSL 1.0.1t  3 May 2016
git version 2.11.0
jq-1.5-1-a5b5cbe
pip 9.0.1 from /usr/lib/python3/dist-packages (python 3.5)
aws-cli/1.15.4 Python/3.5.3 Linux/4.9.87-linuxkit-aufs botocore/1.10.4
Packer v1.2.2
Terraform v0.11.7
```

## Architecture

The Jenkins DevOps Toolkit Docker container uses two bind-mounted directories on the host. The first, the Jenkins' home directory, contains all required configuration. The second directory is used for backups, created using the Jenkins Backup plugin. Additionally, Jenkins can back up its configuration, using the SCM Sync plugin, to GitHub. Both these backup methods require additional configuration.

![Jenkins DevOps Docker Image Architecture](https://github.com/garystafford/jenkins-devops/blob/master/pics/architecture.png)

## Quick Start

Don't want to read the instructions?

```bash
sh ./stack_deploy_local.sh
```

Jenkins will be running on [`http://localhost:8083`](http://localhost:8083).

## Optional: Adding Jenkins Plugins

The `Dockerfile` loads plugins from the `plugin.txt`. Currently, it installs two backup plugins. You can add more plugins to this file, before building Docker image. See the Jenkins [Plugins Index](https://plugins.jenkins.io/) for more.

Built Output

```text
Downloading thinBackup:1.9
Downloading backup:1.6.1
---------------------------------------------------
INFO: Successfully installed 2 plugins.
---------------------------------------------------
```

## Optional: Create Docker Image

The latest `garystafford/jenkins-devops` image is available on [Docker Hub](https://hub.docker.com/r/garystafford/jenkins-devops/).

Optionally, to create a new image from the Dockerfile

```bash
docker build -t garystafford/jenkins-devops:2018.04.19 .
```

## Run the Container

Create a new container from `garystafford/jenkins-devops:2018.04.19` image

```bash
sh ./stack_deploy_local.sh
```

Check logs

```bash
docker logs $(docker ps | grep jenkins-devops | awk '{print $1}')
```

This script also creates local directories `~/jenkins_home/` and `~/jenkins_backup/`.<br>
All relevant Jenkins files are stored in bind-mounted `~/jenkins_home/` directory.<br>
Backups are saved to the bind-mounted `~/jenkins_backup/` host directory, using the Jenkins' [backup](https://wiki.jenkins-ci.org/display/JENKINS/Backup+Plugin) plugin.

Jenkins will be running on [`http://localhost:8083`](http://localhost:8083), by default.

## SCM

Install the SCM Sync Configuration Plugin (`scm-sync-configuration:0.0.10`)

Set git/GitHub repo path to your config repo, for example: `https://<personal_access_token>@github.com/<your_username>/jenkins-config.git`

```bash
docker exec -it $(docker ps | grep jenkins-devops | awk '{print $1}') \
  bash -c 'git config --global user.email "garystafford@rochester.rr.com"'

docker exec -it $(docker ps | grep jenkins-devops | awk '{print $1}') \
  bash -c 'git config --global user.name "Gary A. Stafford"'
```

## Optional: AWS SSL Keys

Copy any required AWS SSL key pairs to bind-mounted `jenkins_home` directory.

```bash
mkdir -p ~/jenkins_home/.ssh

# used for git SCM Sync plugin
cp ~/.ssh/id_rsa ~/jenkins_home/.ssh/id_rsa
cp ~/.ssh/id_rsa.pub ~/jenkins_home/.ssh/id_rsa.pub

# in container for cloning config if on github
docker exec -it $(docker ps | grep jenkins-devops | awk '{print $1}') \
  bash -c 'ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts'

# used for Consul cluster project
cp ~/.ssh/consul_aws_rsa* ~/jenkins_home/.ssh
```

## Optional: AWS Credentials

Copy any required AWS credentials to bind-mounted `jenkins_home` directory

```bash
# used to connect to AWS with Packer/Terraform
cp ~/credentials/jenkins_credentials.env ~/jenkins_home/
```

## Troubleshooting

Fix time skew with container time:

```bash
docker run -it --rm --privileged \
  --pid=host debian nsenter -t 1 -m -u -n -i \
  date -u $(date -u +%m%d%H%M%Y)
```

## Further Development

To modify, build, and test locally, replacing my Docker Hub repo name switch your own:

```bash
# build
docker build --no-cache -t garystafford/jenkins-devops:2018.04.19 .

# run temp copy only
docker run -d --name jenkins-temp -p 8083:8080/tcp -p 50000:50000/tcp garystafford/jenkins-devops:2018.04.19

# push
docker push garystafford/jenkins-devops:2018.04.19

# clean up container and local bind-mounted directory
rm -rf ~/jenkins_home
docker rm -f devopstack_jenkins-devops_1
```

## References

- [Jenkins by Docker](https://hub.docker.com/r/jenkins/jenkins/)
- [SCM Sync configuration plugin](https://wiki.jenkins-ci.org/display/JENKINS/SCM+Sync+configuration+plugin)
- [thinBackup](https://wiki.jenkins-ci.org/display/JENKINS/thinBackup)
- [Backup Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Backup+Plugin)
