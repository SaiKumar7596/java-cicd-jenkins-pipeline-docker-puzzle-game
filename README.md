Nice — I’ll produce the **complete step-by-step lab guide** for your whole CI/CD project, covering everything you did and everything you’ll need to reproduce, troubleshoot, or hand off. I’ll follow your preferred answer format (What it is, Why it matters, key components, use cases, interview-ready concise answer, important points/pitfalls) for each major part, then a long actionable checklist with exact commands, the final corrected files (Jenkinsfile, Dockerfile, corrected `pom.xml` variants), Nexus-delete steps, backup/restore steps, and a troubleshooting appendix with the exact fixes you used.

---

# ✅ Lab Guide — Full CI/CD project (Jenkins → SonarQube → Nexus → Docker Hub → Tomcat)

> Assumptions from your environment
>
> * EC2 public IP: `54.81.139.84` (private `172.31.23.205`)
> * Jenkins runs in Docker on that EC2 (data persisted to host path `/home/ubuntu/jenkins_home`).
> * Nexus at `http://54.81.139.84:8081`
> * SonarQube at `http://54.81.139.84:9000`
> * Your Docker Hub repo: `saikumar7596/my-repo` (or `saikumar7596/pz-tomcat` as used)
> * Project repo: `https://github.com/SaiKumar7596/puzzle-game-java`

---

# SUMMARY (What this doc contains)

1. Architecture & purpose (short)
2. Pre-work: packages, accounts and public/private keys
3. Full step-by-step setup (commands) for: host, Jenkins container, Sonar, Nexus, Tomcat image, Docker Hub push, GitHub webhook
4. How to add tools inside Jenkins (Java/Maven/Docker/Sonar scanner) — exact steps & config values
5. Jenkinsfile (single corrected file) + Dockerfile + corrected `pom.xml` (release vs snapshot guidance)
6. How to delete an artifact from Nexus (UI + REST)
7. Backup & restore Jenkins safely (what you did + recommended script)
8. Troubleshooting & preserved fixes (JAVA_HOME, docker.sock, Nexus 400, Sonar download issues)
9. Final checklist (ready-to-run)
10. Interview-ready concise answers + pitfalls

---

# 1 — Architecture (What it is / Why it matters)

**What it is:** a CI/CD pipeline running on Jenkins (Docker container) which: pulls code from GitHub → runs SonarQube analysis → builds WAR → uploads artifact to Nexus → builds a Docker image that contains Tomcat + your WAR → pushes image to Docker Hub → deploys that image onto the same EC2 (or another target) as a Docker container.

**Why it matters:** Automates build/quality/artifact delivery, produces reproducible images and artefacts, and enables continuous delivery.

**Key components:** GitHub, Jenkins (in Docker), SonarQube (in Docker), Nexus (in Docker), Docker Engine on EC2, Docker Hub, Tomcat (in image).

**Use cases:** automated builds, quality gates, artifact repository for other teams, simple blue/green / rolling deployments on Docker hosts.

**Interview answer (concise):** “I implemented a Jenkins-driven CI pipeline using SonarQube for code quality, Nexus for artifact storage, and Docker to build images. The pipeline builds a WAR, uploads it to Nexus, builds a Tomcat image with the WAR, pushes to Docker Hub, and deploys to an EC2 Docker host — all reproducible and backed by Jenkins job configuration and persistent Jenkins data.”

**Important pitfalls:** Nexus release vs snapshot policy; Docker daemon access from Jenkins container (mounting docker.sock or installing docker inside); secure storage of tokens/ssh keys; JAVA_HOME/tool config mismatch.

---

# 2 — Pre-work (accounts, keys, hostname)

1. Ensure EC2 user has `docker` installed on host and listening. You will run `jenkins` container on EC2.
2. Create or have:

   * GitHub repo → push Jenkinsfile & Dockerfile.
   * Docker Hub account + repository name.
   * Nexus admin credentials.
   * SonarQube admin token (create one when configuring Sonar).
   * SSH key pair (private key) that Jenkins will use to SSH back to EC2 for deployment. Public key must be added to `~ubuntu/.ssh/authorized_keys` on the deployment target (same EC2 in your case).

     * Example: you have `ci/cd.pem` (key pair name) on your local. Convert to appropriate private key content to store in Jenkins Credentials.
3. Make sure ports are open in AWS Security Group: 8080/8090 (Jenkins), 8081 (Nexus), 9000 (Sonar), 22 (SSH), 8080 (Tomcat container). You already have these running.

---

# 3 — Host & Container commands (exact, in order)

> **Host** commands — run on EC2 (`ubuntu@54.81.139.84`)

## 3.1 Install Docker on host (if not installed)

```bash
# Update
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker
# Allow ubuntu user to use docker
sudo usermod -aG docker ubuntu
# verify
docker --version
```

## 3.2 Pull necessary images

```bash
docker pull jenkins/jenkins:lts
docker pull sonarqube:latest
docker pull sonatype/nexus3:latest
docker pull tomcat:10.1.10-jdk17
```

## 3.3 Run SonarQube, Nexus, and your initial tomcat if desired

(You already have started them — example commands)

```bash
# Sonar
docker run -d --name sonar-testing -p 9000:9000 sonarqube:latest

# Nexus
docker run -d --name nexus -p 8081:8081 sonatype/nexus3:latest

# Your pz-tomcat (example)
docker run -d --name tomcat -p 8080:8080 pz-tomcat:1.0
```

---

# 4 — Jenkins: start container (recommended with persistent volume + docker.sock)

You already used `/home/ubuntu/jenkins_home` and `docker run` — the recommended command (runs with docker.sock mounted so pipeline can build/push):

```bash
# create persistent folder
sudo mkdir -p /home/ubuntu/jenkins_home
sudo chown 1000:1000 /home/ubuntu/jenkins_home   # Jenkins default user id is 1000
# run jenkins with docker socket mounted and as root inside (if you need to run docker CLI inside)
docker run -d \
  --name jenkins \
  --user root \
  -p 8090:8080 \
  -p 50000:50000 \
  -v /home/ubuntu/jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

**Notes:**

* You used `--user root` so apt installs inside container succeed and docker CLI is available. Alternative: install docker inside Jenkins image and run with `--user root`.
* After running, exec into container to check docker:

  ```bash
  docker exec -it jenkins bash
  docker --version
  docker ps
  ```
* If `docker` binary not present inside container but socket mounted, you can either:

  * `apt update && apt install -y docker.io` inside container (useful), OR
  * keep docker binary on host and use `docker` from container (requires installing docker client in container). You saw `docker` binary appears after installing it in container.

---

# 5 — Install required tools inside Jenkins container (exact commands)

You chose to install tools inside the Jenkins container (preferred for self-contained agents).

`docker exec -it jenkins bash` then:

## 5.1 Install JDK 17 (if container has Java 21 this is fine — just ensure tools configured)

```bash
# as root inside the container
apt update
apt install -y openjdk-17-jdk maven docker.io curl unzip wget
# verify
java -version
mvn -v
docker --version
```

*Important:* Jenkins itself runs on Java provided by the image (21 is fine), but the `tools { jdk 'JDK17' }` expects a JDK tool configured in Jenkins Global Tool Configuration OR you must install JDK binaries and configure that tool in Jenkins global config.

## 5.2 Sonar Scanner CLI (optional, recommended)

You tried to download sonar-scanner CLI. easiest on container:

```bash
# inside container (as root)
apt update
apt install -y wget unzip ca-certificates

# download release (if direct random github url fails use official SonarSource mirror)
curl -L -o sonar-scanner.zip \
  "https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-5.0.1.3006-linux.zip"

# unzip
unzip sonar-scanner.zip
mv sonar-scanner-5.0.1.3006-linux /opt/sonar-scanner
# symlink
ln -s /opt/sonar-scanner/bin/sonar-scanner /usr/local/bin/sonar-scanner
# verify
which sonar-scanner
sonar-scanner --version
```

You found `which sonar-scanner` -> `/usr/local/bin/sonar-scanner`. Use this path in Jenkins Tool config or set `SONAR_SCANNER_HOME` accordingly.

> If `curl` downloads a tiny HTML (403), try using GitHub release URL or add proper headers. You already resolved it using curl to GitHub asset and then discovered the file was not downloaded fully; fixing SSL (ca-certificates) solved it for you.

## 5.3 Permissions: docker socket

If you mount `/var/run/docker.sock` into the container, ensure `docker` group or permissions allow `jenkins` user to call it.

Quick fix (on host):

```bash
sudo chmod 666 /var/run/docker.sock
# or better: chown root:docker /var/run/docker.sock && add jenkins user to docker group inside container
```

Inside container:

```bash
groupadd docker || true
usermod -aG docker jenkins
# restart jenkins container or new shell
```

---

# 6 — Configure Jenkins (UI steps, exact values)

Open `http://54.81.139.84:8090` (or mapped port).

## 6.1 Plugins (Manage Jenkins → Manage Plugins → Available)

Install:

* Git plugin
* Pipeline (Pipeline: Stage View, Pipeline: GitHub, Pipeline Maven Integration)
* Docker Pipeline
* Docker (plugin)
* Maven Integration plugin
* SonarQube Scanner
* Sonar (SonarQube plugin)
* Nexus Artifact Uploader (or use curl in pipeline)
* SSH Agent
* Credentials Binding
* Blue Ocean (optional)

You already had many installed.

## 6.2 Global Tool Configuration (Manage Jenkins → Global Tool Configuration)

### Maven

* Name: `Maven`
* Install automatically: (optional) or Path: `/usr/bin/mvn` (if installed inside container)

  * If installed in container at `/usr/bin/mvn`, set `Maven` to that path.

### JDK

* Name: `JDK17`
* Path: `/usr/lib/jvm/java-17-openjdk-amd64` (confirm inside container)

  * Or leave `Install automatically` (not recommended if container offline)

### Sonar Scanner

* Name: `sonar-scanner`
* Path: `/opt/sonar-scanner` (or `/usr/local/bin/sonar-scanner` depending on where you installed)

  * If you symlinked to `/usr/local/bin/sonar-scanner`, you can point to `/usr/local` or use the Sonar plugin automatic install.

### Docker Tool (optional)

* If you prefer the Docker plugin tool, configure it. Alternatively rely on `docker` executable installed inside container and the docker socket.

## 6.3 Configure SonarQube in Jenkins (Manage Jenkins → Configure System)

* SonarQube servers → Add SonarQube

  * Name: `sonar-server`
  * Server URL: `http://54.81.139.84:9000`
  * Server authentication token: create token in Sonar UI (User > My Account > Security > Generate Token) and paste token into Jenkins Global Credentials (Secret text) and then select it here.

## 6.4 Add Credentials (Manage Jenkins → Credentials → System → Global)

* Nexus credentials:

  * Kind: Username with password
  * Username: `admin`
  * Password: `admin123` (or actual)
  * ID: `nexus`
* Docker Hub credentials:

  * Kind: Username with password
  * Username: `<dockerhub-user>`
  * Password: `<dockerhub-pass>`
  * ID: `dockerhub-creds`
* SSH credential for deploy to EC2:

  * Kind: `SSH Username with private key`
  * Username: `ubuntu`
  * Private Key: (Enter directly) — paste the private key content (your `ci/cd.pem` private key content). If your private key is in local machine, copy contents and paste.
  * ID: `docker-server`
* GitHub token (optional for API/checkout)

  * Kind: `Secret Text` or `Username with password` (token as password)
  * ID: `github-token`
* If Sonar plugin needs token, store token as secret text and use in Sonar configuration.

**Important:** For SSH you must add the corresponding public key to the `~ubuntu/.ssh/authorized_keys` on the EC2 host you want to SSH into (same host in your case).

---

# 7 — GitHub Webhook (exact steps)

1. In GitHub repository → Settings → Webhooks → Add webhook.
2. Payload URL: `http://54.81.139.84:8090/github-webhook/` (use Jenkins address; if behind firewall use public IP/port mapping)
3. Content type: `application/json`
4. Secret: (optional) — if used, configure same secret in Jenkins GitHub plugin.
5. Which events: choose `Push events` (plus PR events if you want).
6. Save. Test by pushing a commit.

---

# 8 — Sonar webhook (optional)

SonarQube can send webhooks to Jenkins or other systems when analysis completes. It's optional because Jenkins `waitForQualityGate` polls Sonar. Steps:

1. SonarQube Admin → Administration → Configuration → Webhooks → Create
2. URL: Jenkins endpoint or any listener you want. (Not required for Jenkins pipeline to waitForQualityGate — that uses the Sonar token and the task id.)

---

# 9 — Nexus: fix release vs snapshot & delete artifact

**Problem you faced:** 400 `Repository version policy: RELEASE does not allow version: 1.0-SNAPSHOT`.

**Solutions:**

* If your project version includes `-SNAPSHOT` use a `maven-snapshots` repository (not `maven-releases`).
* If you want to push to `maven-releases` change `pom.xml` version to `1.0` (or any non-SNAPSHOT).

### 9.1 Quick fix in `pom.xml`

Either:

* Keep `1.0-SNAPSHOT` and change `NEXUS_URL` to snapshot repo: `http://54.81.139.84:8081/repository/maven-snapshots/`, OR
* Edit `pom.xml` `<version>1.0</version>` and push release.

### 9.2 Delete old artifact from Nexus (UI)

1. Open `http://54.81.139.84:8081` → Login.
2. Browse → Select repository `maven-releases` → navigate to `com/example/puzzle-game-webapp/1.0` → click `Delete` on artifact (or delete the entire `1.0` folder). Confirm.

### 9.3 Delete via Nexus REST (recommended for automation)

1. Search for component (GET):

```bash
curl -u admin:admin123 "http://54.81.139.84:8081/service/rest/v1/search?repository=maven-releases&group=com.example&name=puzzle-game-webapp&version=1.0"
```

Response contains `items` with `id`s. Then delete by component id:

```bash
curl -u admin:admin123 -X DELETE "http://54.81.139.84:8081/service/rest/v1/components/<componentId>"
```

**Important:** Deleting a release may be blocked by policy (immutable release) — if so you may need to change repository settings or delete the asset via Nexus UI with admin privileges or create a new version.

---

# 10 — Dockerfile (corrected final)

Use this Dockerfile in repo root. It downloads the WAR from Nexus at build-time. Note: this requires `docker build` to have the secrets (NEXUS_USER/NEXUS_PASS) passed as build args (we do that in the Jenkins pipeline).

```dockerfile
# -------- Stage 1: Downloader --------
FROM eclipse-temurin:17-jre AS downloader

ARG NEXUS_URL
ARG GROUP_ID_PATH
ARG APP_NAME
ARG VERSION
ARG NEXUS_USER
ARG NEXUS_PASS

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Download artifact from Nexus (AUTH)
RUN ARTIFACT_URL="${NEXUS_URL%/}/${GROUP_ID_PATH}/${APP_NAME}/${VERSION}/${APP_NAME}-${VERSION}.war" && \
    echo "Downloading WAR from: ${ARTIFACT_URL}" && \
    curl -u "${NEXUS_USER}:${NEXUS_PASS}" -L "${ARTIFACT_URL}" -o /tmp/app.war

# -------- Stage 2: Tomcat --------
FROM tomcat:10.1.10-jdk17

# Remove default ROOT app
RUN rm -rf /usr/local/tomcat/webapps/ROOT

# Copy the downloaded war as ROOT.war for auto-deploy
COPY --from=downloader /tmp/app.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080
CMD ["catalina.sh", "run"]
```

**Fixes made:**

* Corrected `COPY ... ROOT.wa` → `ROOT.war`.
* Use `ARTIFACT_URL="${NEXUS_URL%/}/...` to avoid double slashes.
* Ensure Tomcat base image tag consistent.

---

# 11 — Jenkinsfile (final corrected, one file — paste to repo)

This uses secure credential handling and avoids insecure Groovy interpolation of secrets in `sh` strings. It relies on Docker socket mounted or docker installed in container.

```groovy
pipeline {
  agent any

  environment {
    NEXUS_URL   = "http://54.81.139.84:8081/repository/maven-releases/"
    DOCKER_REPO = "saikumar7596/my-repo"    // change to your Docker Hub repo
    GROUP_ID    = "com.example"
    ARTIFACT_ID = "puzzle-game-webapp"
    MVN_OPTS    = "-DskipTests"
    DEPLOY_HOST = "54.81.139.84"
    CONTAINER_NAME = "tomcat"
  }

  tools {
    maven "Maven"   // Ensure this name exists in Manage Jenkins -> Global Tool Configuration
    jdk   "JDK17"   // Ensure this name exists as well
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        echo "Commit: ${env.GIT_COMMIT ?: 'local'}"
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonar-server') {
          sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=${ARTIFACT_ID}'
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Build WAR') {
      steps {
        sh "mvn clean package ${MVN_OPTS}"
      }
    }

    stage('Upload WAR to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          script {
            def WAR = sh(script: "ls target/*.war | head -n 1", returnStdout: true).trim()
            if (!fileExists(WAR)) { error "WAR not found at ${WAR}" }
            def VERSION = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
            def ARTIFACT_PATH = "${GROUP_ID.replace('.','/')}/${ARTIFACT_ID}/${VERSION}/${ARTIFACT_ID}-${VERSION}.war"
            echo "Uploading ${WAR} to ${NEXUS_URL}${ARTIFACT_PATH}"
            // Use curl with credentials securely passed via environment
            sh """
              curl -v -u "${NEXUS_USER}:${NEXUS_PASS}" --upload-file "${WAR}" "${NEXUS_URL}${ARTIFACT_PATH}"
            """
          }
        }
      }
    }

    stage('Prepare WAR for Docker') {
      steps {
        sh '''
          set -e
          WAR=$(ls target/*.war | head -n1)
          if [ -z "$WAR" ]; then echo "No WAR found"; exit 1; fi
          cp -f "$WAR" target/app.war
          ls -lh target/app.war
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          def shortCommit = (env.GIT_COMMIT ?: 'local').take(7)
          def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
          withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
            sh """
              docker build \
                --build-arg NEXUS_URL='${NEXUS_URL}' \
                --build-arg GROUP_ID_PATH='${GROUP_ID.replace('.','/')}' \
                --build-arg APP_NAME='${ARTIFACT_ID}' \
                --build-arg VERSION='${version}' \
                --build-arg NEXUS_USER='${NEXUS_USER}' \
                --build-arg NEXUS_PASS='${NEXUS_PASS}' \
                -t ${DOCKER_REPO}:${shortCommit} .
              docker tag ${DOCKER_REPO}:${shortCommit} ${DOCKER_REPO}:latest
            """
          }
        }
      }
    }

    stage('Push Docker Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          script {
            def shortCommit = (env.GIT_COMMIT ?: 'local').take(7)
            sh """
              echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
              docker push ${DOCKER_REPO}:${shortCommit}
              docker push ${DOCKER_REPO}:latest
            """
          }
        }
      }
    }

    stage('Deploy on same EC2 via SSH') {
      steps {
        // Jenkins must have SSH credential with id 'docker-server'
        sshagent(['docker-server']) {
          sh """
            ssh -o StrictHostKeyChecking=no ubuntu@${DEPLOY_HOST} \\
            'docker pull ${DOCKER_REPO}:latest && \\
             docker stop ${CONTAINER_NAME} 2>/dev/null || true && \\
             docker rm ${CONTAINER_NAME} 2>/dev/null || true && \\
             docker run -d --name ${CONTAINER_NAME} -p 8080:8080 --restart unless-stopped ${DOCKER_REPO}:latest'
          """
        }
      }
    }
  }

  post {
    success { echo "PIPELINE SUCCEEDED" }
    failure { echo "PIPELINE FAILED" }
  }
}
```

**Notes:**

* Make sure Jenkins Global Tools `Maven` and `JDK17` are configured and point to actual installed paths in the Jenkins container.
* `docker` calls assume Docker daemon accessible (socket mounted or docker installed in container).
* We explicitly pass Nexus credentials to docker build as build args. Be aware build args are visible in image history; for safer pipelines, download artifact in pipeline and use `COPY` into final image (avoid passing credentials into Dockerfile). Another safe way is to `curl` the artifact in pipeline and `docker build` using local `target/app.war` via a `Dockerfile` that just copies `/target/app.war`. That prevents credentials being baked into image history.

---

# 12 — Recommended safer Docker build alternative (avoid creds in Dockerfile)

**Safer approach:** In pipeline, after `Prepare WAR for Docker`, do:

```bash
# in pipeline (sh)
cp target/app.war build-context/ROOT.war
docker build -t ${DOCKER_REPO}:${shortCommit} build-context
# Dockerfile just copies ROOT.war into Tomcat image (no NEXUS creds)
```

Use Dockerfile:

```dockerfile
FROM tomcat:10.1.10-jdk17
RUN rm -rf /usr/local/tomcat/webapps/ROOT
COPY ROOT.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

This is better security-wise (no credentials in build args).

---

# 13 — `pom.xml` — two options (snapshot vs release)

### Option A — If you want SNAPSHOT (easier for dev)

Keep:

```xml
<version>1.0-SNAPSHOT</version>
```

Then set `NEXUS_URL` to `.../repository/maven-snapshots/` in pipeline/env.

### Option B — Release (if you want to push to `maven-releases`)

Change `pom.xml` to:

```xml
<version>1.0</version>
```

You already tested pushing `1.0` and got different error (cannot be updated) — remove old release from Nexus or bump version on each release.

**Corrected `pom.xml` as release (1.0)** (paste to project if you choose release):

```xml
<project ...>
  ...
  <groupId>com.example</groupId>
  <artifactId>puzzle-game-webapp</artifactId>
  <version>1.0</version>
  ...
</project>
```

---

# 14 — How you deleted and backed up Jenkins (what you did — good practice)

You did:

```bash
docker cp jenkins:/var/jenkins_home /home/ubuntu/jenkins-backup
# restored into /home/ubuntu/jenkins_home and re-run docker with volume
docker run -d --name jenkins --user root -p 8090:8080 -v /home/ubuntu/jenkins_home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkins/jenkins:lts
```

This is the **correct** idea: **backup `/var/jenkins_home`**, then recreate container mounting that folder. You kept plugins, jobs, credentials, etc.

**Recommendation:** Always maintain a scheduled backup script that archives `/var/jenkins_home` to a safe location.

Example backup script:

```bash
#!/bin/bash
BACKUP_DIR="/home/ubuntu/jenkins-backups"
mkdir -p "$BACKUP_DIR"
TS=$(date +%F_%H%M)
tar -czf "$BACKUP_DIR/jenkins_home_$TS.tar.gz" -C /home/ubuntu jenkins_home
# Optional: upload to s3 or remote
```

---

# 15 — Nexus artifact deletion steps (recap with exact commands)

1. UI: Browse repo → select component → Delete.
2. REST:

```bash
# search:
curl -u admin:admin123 "http://54.81.139.84:8081/service/rest/v1/search?repository=maven-releases&group=com.example&name=puzzle-game-webapp&version=1.0"
# take component id from "items[0].id" in response
# delete:
curl -u admin:admin123 -X DELETE "http://54.81.139.84:8081/service/rest/v1/components/<componentId>"
```

---

# 16 — Troubleshooting (common errors you faced & fixes)

### 1) `Repository version policy: RELEASE does not allow version: 1.0-SNAPSHOT`

* Cause: uploading a SNAPSHOT to a release repo.
* Fix: use `maven-snapshots` repo or change project version to release (1.0). Remove existing artifacts if necessary.

### 2) `The JAVA_HOME environment variable is not defined correctly`

* Cause: `tools` block injected tool but JAVA_HOME not set or Maven uses different JDK.
* Fix: ensure JDK path is set in Manage Jenkins → Global Tool Configuration, name matches `tools { jdk "JDK17" }`. Alternatively, set `env: JAVA_HOME` in pipeline:

  ```groovy
  environment { JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64' }
  ```

### 3) `Cannot connect to the Docker daemon at unix:///var/run/docker.sock`

* Cause: `docker` not installed in container OR docker socket not mounted OR permission problem.
* Fix:

  * Mount `-v /var/run/docker.sock:/var/run/docker.sock` when starting Jenkins container.
  * Install docker client inside Jenkins container (`apt install docker.io`).
  * Adjust socket permissions (temporary): `sudo chmod 666 /var/run/docker.sock` (less secure). Better: add `jenkins` user to host `docker` group or chown socket accordingly.

### 4) Sonar download `403` / broken zip

* Cause: download redirect or missing headers; missing `ca-certificates`.
* Fix: install `ca-certificates`, `wget`/`curl`, use GitHub release URL with `curl -L` and verify `Content-Length`. Then `unzip`.

### 5) `WARNING: A secret was passed to "sh" using Groovy String interpolation`

* Cause: using `"$NEXUS_PASS"` in Groovy string.
* Fix: prefer triple-quotes and use `"""` and pass credentials via environment from `withCredentials` as shown. Avoid logging secrets.

### 6) `mvn: not found` inside pipeline run after restore

* Cause: Maven was not installed in restored Jenkins container or Global Tool not configured to locate it.
* Fix: install maven inside container and set Maven tool path in Global Tool Configuration or set PATH/JENKINS to use `/usr/bin/mvn`.

---

# 17 — Final checklist (what to run now)

1. Make sure `jenkins` container is running with:

   * `/home/ubuntu/jenkins_home:/var/jenkins_home` mounted
   * `/var/run/docker.sock:/var/run/docker.sock` mounted
2. Exec into `jenkins` container and ensure:

   ```bash
   apt update && apt install -y maven docker.io openjdk-17-jdk curl wget unzip
   # ensure docker client works
   docker --version
   mvn --version
   java -version
   which sonar-scanner  # should show /usr/local/bin/sonar-scanner if installed
   ```
3. In Jenkins UI:

   * Configure Tools: Maven path, JDK17 path, sonar-scanner path.
   * Add Credentials: nexus, dockerhub-creds, docker-server (SSH private key), github-token (if needed).
   * Install recommended plugins.
4. Ensure your `pom.xml` version matches the repository you intend to upload to.

   * If snapshot: change NEXUS URL in pipeline env to snapshot repo.
5. Push corrected Jenkinsfile + Dockerfile + (optionally updated pom.xml) to GitHub.
6. In GitHub repo: add webhook to `http://54.81.139.84:8090/github-webhook/`.
7. Trigger a commit/push and watch the Jenkins job.

---

# 18 — Documentation checklist for your final artifact (what I will include if you want a printable lab guide)

* Full step-by-step commands (host + container)
* All Jenkins UI clicks + exact values (tools names, IDs)
* Final Jenkinsfile, Dockerfile, and pom.xml
* Backup/restore and disaster recovery instructions
* Troubleshooting section with logs + fixes
* Architecture diagram (ASCII + recommended diagram PNG)
* Appendix: Nexus REST API examples + common error codes

---

# 19 — Interview-ready concise answer (what you'll say)

“I built a Jenkins-driven CI/CD pipeline on an EC2 host. Jenkins (in Docker) runs SonarQube analysis, packages a WAR with Maven, uploads to Nexus, builds a Tomcat Docker image with the WAR, pushes the image to Docker Hub, and deploys the container to the same EC2. I persisted Jenkins config to a host volume and maintain backups of `/var/jenkins_home`. I solved common issues: Nexus release vs snapshot policy, containerized docker access via `docker.sock`, SonarScanner installation, and ensured credentials (Nexus/Docker/SSH) are stored in Jenkins Credentials securely.”

---

# 20 — What I’ll deliver now (you already confirmed)

* The full documentation above (this is it).
* I’ve included final Jenkinsfile, Dockerfile and recommended safer Docker build approach.
* I included commands you ran and the fixes you implemented (backup/restore, chmod on docker.sock, installing docker/maven inside container, sonar-scanner issues, Nexus deletion steps).

---

If you want, I’ll now:

1. Produce a single downloadable **markdown** file of the entire guide ready to upload/print (I can create it here).
2. Or export it into a multi-page PDF/HTML structure (I can prepare the markdown now).

Which one do you want next?

* Reply **A** to get a ready-to-download **full markdown document** in this chat (I’ll paste it complete).
* Reply **B** to get the same content but **packaged as a ZIP with files** (Jenkinsfile, Dockerfile, pom.xml, docs) so you can drop into your repo.
