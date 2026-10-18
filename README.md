# Jenkins with Docker Compose on macOS

A reference guide for running Jenkins with Docker Compose and Docker-in-Docker (DinD), verifying Docker connectivity, and creating Jenkins Pipelines.

## Table of Contents

1. [Architecture](#1-architecture)
2. [Prerequisites](#2-prerequisites)
3. [Project Structure](#3-project-structure)
4. [Important Concepts](#4-important-concepts)
5. [Jenkins Dockerfile](#5-jenkins-dockerfile)
6. [Docker Compose Configuration](#6-docker-compose-configuration)
7. [Start Jenkins](#7-start-jenkins)
8. [Unlock Jenkins](#8-unlock-jenkins)
9. [Verify Docker Connectivity](#9-verify-docker-connectivity)
10. [Create a Jenkins Pipeline](#10-create-a-jenkins-pipeline)
11. [Understanding the Jenkinsfile](#11-understanding-the-jenkinsfile)
12. [Useful Commands](#12-useful-commands)
13. [Troubleshooting](#13-troubleshooting)
14. [Security Notes](#14-security-notes)

---

## 1. Architecture

This setup runs two containers:

- **Jenkins:** Hosts the web interface, manages jobs, and executes Pipeline steps.
- **Docker-in-Docker (DinD):** Runs the Docker daemon that Jenkins uses to build and run containers.

```text
macOS
  |
  +-- Docker Desktop
        |
        +-- jenkins-network
              |
              +-- jenkins
              |     +-- Jenkins UI: localhost:8080
              |     +-- Docker CLI
              |     +-- Jenkins Pipelines
              |
              +-- jenkins-dind
                    +-- Docker daemon: port 2376
                    +-- TLS certificates
```

Jenkins connects to the DinD daemon using the Docker network alias `docker:2376` over the shared Docker network.

On macOS, Docker Desktop runs Linux containers inside its Linux environment. You do not need to install Linux or Java directly on your Mac for this setup.

## 2. Prerequisites

Install Docker Desktop for Mac:

https://docs.docker.com/desktop/setup/install/mac-install/

Start Docker Desktop and verify the installation in Terminal:

```bash
docker --version
docker compose version
docker info
```

All three commands should work before continuing.

## 3. Project Structure

Create a project directory:

```bash
mkdir jenkins-101
cd jenkins-101
```

The project will contain:

```text
jenkins-101/
├── Dockerfile
├── compose.yaml
├── Jenkinsfile
└── README.md
```

| File           | Purpose                                             |
| -------------- | --------------------------------------------------- |
| `Dockerfile`   | Defines how to build the custom Jenkins image.      |
| `compose.yaml` | Defines containers, networking, ports, and volumes. |
| `Jenkinsfile`  | Defines a Jenkins Pipeline using Groovy syntax.     |
| `README.md`    | Documents setup and usage.                          |

## 4. Important Concepts

### 4.1 Docker Image vs. Container

An image is a template used to create a container. A container is a running instance of an image.

In this configuration:

```yaml
image: jenkins:local
container_name: jenkins-ui
```

- `image` specifies the image name and tag.
- `container_name` specifies the running container's name.

The Jenkins image is built locally from the Dockerfile.

### 4.2 Why Use a Custom Jenkins Image?

The standard Jenkins image does not include Docker CLI. The custom Dockerfile installs Docker CLI and Jenkins Pipeline plugins. The Docker daemon runs separately in the DinD container.

### 4.3 Docker-in-Docker

DinD runs a Docker daemon inside a container. Jenkins sends Docker commands to this daemon over TCP with TLS enabled.

```yaml
privileged: true
```

This grants extensive privileges and should only be used in a trusted environment.

### 4.4 Docker Network

Both services join the same network:

```yaml
networks:
  - jenkins-network
```

Jenkins connects to DinD using:

```yaml
DOCKER_HOST: tcp://docker:2376
```

The DinD service exposes the Docker hostname as `docker` through a network alias, which matches the TLS certificate that the daemon generates.

### 4.5 Docker Volumes

Named volumes preserve data when containers are recreated.

- `jenkins-data`: Stores Jenkins home data, including jobs, plugins, and configuration.
- `jenkins-dind-certs`: Shares TLS client certificates between DinD and Jenkins.

These volumes are declared at the bottom of `compose.yaml`.

**Important:** This configuration also mounts `jenkins-data` into the privileged DinD container, following the tutorial-style approach. Sharing Jenkins home with a privileged container has security implications. Review this arrangement before running untrusted Pipeline jobs.

### 4.6 TLS Configuration

Jenkins uses these environment variables:

```yaml
DOCKER_HOST: tcp://docker:2376
DOCKER_CERT_PATH: /certs/client
DOCKER_TLS_VERIFY: '1'
```

The client certificates are mounted read-only:

```yaml
- jenkins-dind-certs:/certs/client:ro
```

The DinD container generates and manages the certificates in the shared volume.

### 4.7 Startup Dependency and Readiness

```yaml
depends_on:
  jenkins-dind:
    condition: service_healthy
```

This ensures that Jenkins starts only after the DinD daemon is ready to accept TLS connections. The DinD container also defines a `healthcheck` that verifies the Docker daemon is responding before Jenkins proceeds.

## 5. Jenkins Dockerfile

Create a file named `Dockerfile`:

```dockerfile
FROM jenkins/jenkins:2.580.1-jdk21

USER root

RUN apt-get update && \
    apt-get install -y \
      lsb-release \
      ca-certificates \
      curl && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL \
      https://download.docker.com/linux/debian/gpg \
      -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) \
      signed-by=/etc/apt/keyrings/docker.asc] \
      https://download.docker.com/linux/debian \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
      > /etc/apt/sources.list.d/docker.list && \
    apt-get update && \
    apt-get install -y docker-ce-cli && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

USER jenkins

RUN jenkins-plugin-cli --plugins \
    "docker-workflow json-path-api"
```

### What Does This Dockerfile Do?

1. Starts from a Jenkins image running JDK 21.
2. Switches to root to install system packages.
3. Adds Docker's official Debian package repository.
4. Installs Docker CLI, not the Docker daemon.
5. Returns to the unprivileged `jenkins` user.
6. Installs Jenkins Docker Pipeline plugins.

The resulting image is tagged `jenkins:local` by Compose.

The Jenkins base image tag is pinned in this example. Check that the tag remains available when upgrading or changing versions.

## 6. Docker Compose Configuration

Create `compose.yaml` with the following configuration:

```yaml
services:
  jenkins-dind:
    image: docker:dind
    container_name: jenkins-dind
    privileged: true
    restart: unless-stopped

    environment:
      DOCKER_TLS_CERTDIR: /certs

    volumes:
      - jenkins-dind-certs:/certs/client
      - jenkins-data:/var/jenkins_home

    networks:
      jenkins-network:
        aliases:
          - docker

    healthcheck:
      test:
        [
          'CMD-SHELL',
          "docker version --format '{{.Server.Version}}' >/dev/null 2>&1 || exit 1",
        ]
      interval: 5s
      timeout: 5s
      retries: 12
      start_period: 10s

    command:
      - --storage-driver=overlay2

  jenkins:
    build:
      context: .
      dockerfile: Dockerfile
    image: jenkins:local
    container_name: jenkins-ui
    restart: unless-stopped

    depends_on:
      jenkins-dind:
        condition: service_healthy

    environment:
      DOCKER_HOST: tcp://docker:2376
      DOCKER_CERT_PATH: /certs/client
      DOCKER_TLS_VERIFY: '1'

    ports:
      - '8080:8080'
      - '50000:50000'

    volumes:
      - jenkins-data:/var/jenkins_home
      - jenkins-dind-certs:/certs/client:ro

    networks:
      - jenkins-network

networks:
  jenkins-network:
    name: jenkins-network
    driver: bridge

volumes:
  jenkins-data:
  jenkins-dind-certs:
```

### Configuration Explained

| Setting                | Purpose                                       |
| ---------------------- | --------------------------------------------- |
| `jenkins-dind`         | Compose service running Docker-in-Docker.     |
| `jenkins`              | Compose service running Jenkins.              |
| `image: docker:dind`   | Uses the official Docker-in-Docker image.     |
| `image: jenkins:local` | Names the locally built Jenkins image.        |
| `privileged: true`     | Allows DinD to run its Docker daemon.         |
| `jenkins-network`      | Shared network for container communication.   |
| `DOCKER_HOST`          | Tells Docker CLI where the daemon is located. |
| `DOCKER_CERT_PATH`     | Points to the client TLS certificates.        |
| `DOCKER_TLS_VERIFY`    | Enables TLS certificate verification.         |
| `8080:8080`            | Exposes the Jenkins web interface on the Mac. |
| `50000:50000`          | Exposes the inbound agent port if required.   |
| `jenkins-data`         | Persists Jenkins home data.                   |
| `jenkins-dind-certs`   | Persists and shares TLS client certificates.  |

Port `2376` is not published on the Mac because Jenkins accesses the daemon over the internal network.

The `jenkins-data` volume is shared with DinD in this configuration. This follows the tutorial-style setup but requires careful consideration of security and workspace mounts.

## 7. Start Jenkins

From the directory containing `Dockerfile` and `compose.yaml`, run:

```bash
docker compose up -d --build
```

This command:

1. Creates the network if needed.
2. Creates named volumes if needed.
3. Builds the custom Jenkins image and tags it `jenkins:local`.
4. Starts the DinD container.
5. Starts the Jenkins container.

Check container status:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f
```

Press `Ctrl+C` to stop following logs without stopping the containers.

## 8. Unlock Jenkins

Open:

http://localhost:8080

Retrieve the initial administrator password:

```bash
docker compose exec jenkins \
  cat /var/jenkins_home/secrets/initialAdminPassword
```

Then:

1. Enter the password in the browser.
2. Choose **Install suggested plugins**.
3. Create an administrator account.
4. Complete the setup wizard.

Jenkins stores its configuration in the `jenkins-data` named volume.

## 9. Verify Docker Connectivity

Run:

```bash
docker compose exec jenkins docker version
```

A successful result displays both Docker Client and Server information.

Test downloading an image:

```bash
docker compose exec jenkins docker pull hello-world
```

If successful, Jenkins can communicate with the DinD daemon and pull images.

If a command fails, inspect the logs:

```bash
docker compose logs jenkins
docker compose logs jenkins-dind
```

## 10. Create a Jenkins Pipeline

There are two common ways to define a Pipeline.

### Option A: Write the Pipeline in the Jenkins UI

1. Open Jenkins at `http://localhost:8080`.
2. Click **New Item**.
3. Enter `docker-connectivity-test`.
4. Select **Pipeline** and click **OK**.
5. Scroll to the Pipeline section.
6. Set **Definition** to `Pipeline script`.
7. Paste the Groovy script below.
8. Click **Save**.
9. Select **Build Now**.
10. Open **Console Output**.

Use this script:

```groovy
pipeline {
    agent any

    stages {
        stage('Check Docker Connection') {
            steps {
                sh 'docker version'
            }
        }

        stage('Pull Test Image') {
            steps {
                sh 'docker pull hello-world'
            }
        }
    }
}
```

If successful, the output displays Docker Client and Server information and pulls the `hello-world` image.

### Option B: Store the Pipeline in a Jenkinsfile

Create a file named `Jenkinsfile` in the project directory:

```groovy
pipeline {
    agent any

    stages {
        stage('Check Docker Connection') {
            steps {
                sh 'docker version'
            }
        }

        stage('Pull Test Image') {
            steps {
                sh 'docker pull hello-world'
            }
        }
    }
}
```

To make Jenkins load this file automatically, commit the project to a Git repository and configure a Pipeline job:

1. Set **Definition** to `Pipeline script from SCM`.
2. Set **SCM** to `Git`.
3. Enter your repository URL.
4. Set **Script Path** to `Jenkinsfile`.
5. Save and run the job.

Simply placing a `Jenkinsfile` next to `compose.yaml` does not automatically make Jenkins execute it. You must configure Jenkins to load it or paste its contents into the UI.

## 11. Understanding the Jenkinsfile

### `pipeline`

Declares a Declarative Jenkins Pipeline.

### `agent any`

Runs the Pipeline on any available agent. In this setup, jobs normally run on the built-in controller unless separate agents are configured.

### `stages`

Groups the major phases of the Pipeline, such as checkout, build, test, package, and deploy.

### `stage`

Defines a named phase:

```groovy
stage('Build') {
    steps {
        sh 'docker build -t example-app:latest .'
    }
}
```

### `steps`

Contains the operations Jenkins executes within a stage.

### `sh`

Runs a shell command in the Pipeline's execution environment:

```groovy
sh 'docker version'
```

The Docker CLI uses `DOCKER_HOST` to communicate with the separate DinD daemon.

**Important:** Docker CLI connectivity does not guarantee every Docker Pipeline operation will work. For example, `docker.image(...).inside { ... }` may require workspace mounts accessible to the Docker daemon. More complex pipelines may need additional workspace and mount configuration.

## 12. Useful Commands

Run these commands from the project directory.

| Task                          | Command                                      |
| ----------------------------- | -------------------------------------------- |
| Start containers              | `docker compose up -d`                       |
| Build and start               | `docker compose up -d --build`               |
| Check status                  | `docker compose ps`                          |
| Follow logs                   | `docker compose logs -f`                     |
| Jenkins logs                  | `docker compose logs -f jenkins`             |
| DinD logs                     | `docker compose logs -f jenkins-dind`        |
| Stop containers               | `docker compose stop`                        |
| Start stopped containers      | `docker compose start`                       |
| Restart containers            | `docker compose restart`                     |
| Remove containers and network | `docker compose down`                        |
| Rebuild Jenkins image         | `docker compose build jenkins`               |
| Open a shell in Jenkins       | `docker compose exec jenkins sh`             |
| Test Docker connectivity      | `docker compose exec jenkins docker version` |

### Persistence and Cleanup

Remove containers and the network while preserving named volumes:

```bash
docker compose down
```

Remove containers, the network, and Compose-managed named volumes:

```bash
docker compose down -v
```

**Warning:** `docker compose down -v` can permanently delete Jenkins jobs, plugins, configuration, and TLS certificates. Use it only when you intend to reset the setup.

## 13. Troubleshooting

### Jenkins Cannot Connect to Docker

Run:

```bash
docker compose logs jenkins-dind
docker compose exec jenkins docker version
```

Check that:

- Both containers are running.
- Both services use `jenkins-network`.
- `DOCKER_HOST` is `tcp://docker:2376`.
- `DOCKER_CERT_PATH` is `/certs/client`.
- `DOCKER_TLS_VERIFY` is `"1"`.
- The client certificate volume is mounted correctly.
- The DinD daemon has finished starting and is healthy.

### Jenkins Starts Before DinD Is Ready

`depends_on` controls startup order but not readiness. Retry the connection test after startup. A health check or retry logic can improve reliability.

### Jenkins UI Is Unavailable

Check:

```bash
docker compose ps
docker compose logs jenkins
```

Ensure port `8080` is available on your Mac.

### Docker CLI Is Missing

Check:

```bash
docker compose exec jenkins which docker
docker compose exec jenkins docker version
```

If Docker CLI is missing, rebuild the custom image:

```bash
docker compose up -d --build
```

### Dockerfile Changes Are Not Reflected

Rebuild the image and recreate the service:

```bash
docker compose build jenkins
docker compose up -d
```

## 14. Security Notes

- DinD requires `privileged: true`, which grants extensive container privileges.
- Do not expose the Docker daemon to untrusted networks.
- Keep DinD on the internal Compose network.
- Protect the Jenkins administrator account and credentials.
- Run only trusted Pipeline code.
- Back up Jenkins data before upgrades or major changes.
- Pin and test image versions when reproducibility matters.

This setup is suitable for local learning and development. Production deployments require additional hardening, backup planning, credential management, and access controls.

---

## Quick Reference

Start the environment:

```bash
docker compose up -d --build
```

Open Jenkins:

http://localhost:8080

Get the initial password:

```bash
docker compose exec jenkins \
  cat /var/jenkins_home/secrets/initialAdminPassword
```

Test Docker connectivity:

```bash
docker compose exec jenkins docker version
```

Stop the environment while preserving named volumes:

```bash
docker compose down
```
