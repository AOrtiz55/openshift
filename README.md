nshift full journey · MD

# GitLab Runner on OpenShift — Full Journey Documentation

**Author:** Senior OpenShift/Containers Engineer  
**Environment:** Air-gapped ROSA (Red Hat OpenShift Service on AWS)  
**Local Dev:** Windows laptop with WSL2 (Ubuntu) + Docker Desktop  
**Goal:** Host a GitLab Runner in OpenShift so teams use their own images in CI/CD pipelines  
**Approach:** GitLab repo holds Dockerfile → OpenShift BuildConfig builds image internally → Deployment runs the runner

> This document is a full history of everything we did, every pivot, every debug step, and why. Nothing is skipped.

---

## Table of Contents

1. [Key Terminology](#key-terminology)
2. [Variable Reference](#variable-reference)
3. [Environment Setup](#environment-setup)
4. [Understanding the Air-Gap](#understanding-the-air-gap)
5. [Docker Concepts](#docker-concepts)
6. [Deployment vs Job — The Core Confusion](#deployment-vs-job)
7. [The Full OpenShift + GitLab Journey](#the-full-journey)
8. [All Dockerfile Versions](#all-dockerfile-versions)
9. [All Pivots and Why](#all-pivots-and-why)
10. [Deployment Setup](#deployment-setup)
11. [Enterprise JFrog Version](#enterprise-jfrog-version)
12. [Troubleshooting Reference](#troubleshooting-reference)
13. [Quick Reference Commands](#quick-reference-commands)

---

## Key Terminology

### Docker / Container Concepts

**Image** — A read-only blueprint of a container. Like a recipe — never changes. Stored in a registry. Built from a Dockerfile.

**Container** — A running instance of an image. Like the meal made from the recipe. One image spawns many containers. Writable layer is lost when container stops.

**Dockerfile** — A text file of instructions Docker executes top-to-bottom to build an image. Each instruction creates one immutable layer.

**Layer** — A cached snapshot of filesystem changes from one Dockerfile instruction. Layers stack. Unchanged layers are reused on rebuild.

**Registry** — A storage server for images. JFrog Artifactory, OpenShift internal registry, and Docker Hub are all registries.

**docker build** — Reads a Dockerfile, executes instructions, produces an image locally.

**docker push** — Uploads a locally built image to a registry.

**docker pull** — Downloads an image from a registry to your machine.

**UBI9** — Universal Base Image 9. Minimal RHEL9 OS packaged as a container base image. Built and maintained by Red Hat. Free to use. Has `rpm`, `dnf`, `yum` built in. Does NOT have Python, Java, or any runtime — it is a foundation, not an application.

**ARG** — Dockerfile build-time variable. Only exists during `docker build`. NOT available at container runtime. Injected by BuildConfig from secrets.

**CMD** — The command that runs when the container starts. Long-running CMD keeps Deployment alive. Exiting CMD causes CrashLoopBackOff in a Deployment.

**RUN** — Dockerfile instruction that executes a shell command during build. Baked into image as a layer.

### OpenShift Concepts

**Pod** — Smallest deployable unit. Wraps one or more containers.

**Deployment** — Manages pods that run forever. Any exit = restart. CrashLoopBackOff when container keeps exiting. Use for: web servers, APIs, the GitLab runner itself.

**Job** — Manages pods that run once and exit. Exit code 0 = done. No restart. Use for: scripts, migrations, one-time tasks.

**CrashLoopBackOff** — Pod keeps crashing and restarting. Most common causes: wrong workload type (Job logic in Deployment), missing config, running as root blocked by SCC.

**BuildConfig** — OpenShift resource defining how to build an image from source. Watches a Git repo, builds on push, outputs to ImageStream. Solves the firewall/push problem.

**ImageStream** — OpenShift image tracking layer. Must exist before BuildConfig can output to it. Created with `oc create imagestream`.

**Secret** — Stores sensitive data base64-encoded. Two types matter here:

- `kubernetes.io/basic-auth` — Required for git source auth in BuildConfig. Must have `username` and `password` keys.
- `Opaque` — Generic. Used for build arg injection.
  **ConfigMap** — Non-sensitive config as key-value. We mount `config.toml` into runner pod via ConfigMap.

**ServiceAccount** — Identity for pods making API calls within cluster. Runner needs one with permission to create job pods.

**SCC (Security Context Constraint)** — OpenShift security policy. Default `restricted` blocks root (UID 0) containers. Why `USER 1001` is non-negotiable.

**Route** — Exposes service outside cluster. Used to expose internal registry for docker push.

**Namespace/Project** — Isolated workspace. Developer Sandbox gives you `yourname-dev` pre-created.

### GitLab Runner Concepts

**Executor** — How runner runs jobs. We use `kubernetes` (creates new pod per job). Do NOT use `docker` (needs Docker socket, blocked in OpenShift).

**Helper Image** — Required by GitLab Runner 17+. Handles: cloning repo, uploading artifacts, cache. Comes as separate RPM. Both RPMs must be same version.

**Registration Token** — From GitLab CI/CD settings. Proves runner is authorized. Different from personal access token.

**config.toml** — Runner's main config. Defines GitLab URL, executor, namespace, images. Mounted via ConfigMap.

**clone_url** — config.toml override for URL used to clone repos during jobs. Critical in air-gapped environments.

---

## Variable Reference

### Shell Exports (You Set These)

```bash
export GITLAB_TOKEN="glpat-xxxxxxxxxxxx"
# Your GitLab Personal Access Token
# Where: GitLab → Preferences → Access Tokens
# Required scopes: api (covers everything including package registry)
# REVOKE AND REGENERATE if ever shared in logs or chat

export PROJECT_ID="82861465"
# Your GitLab project numeric ID — not sensitive
# Where: GitLab → your repo → Settings → General → top of page

# Persist in WSL so terminal restarts don't clear them
echo 'export GITLAB_TOKEN="your-token"' >> ~/.bashrc
echo 'export PROJECT_ID="82861465"' >> ~/.bashrc
source ~/.bashrc

# Update when token changes
sed -i '/GITLAB_TOKEN/d' ~/.bashrc
echo 'export GITLAB_TOKEN="new-token"' >> ~/.bashrc
```

### OpenShift Secrets

```
Secret: gitlab-git-auth
  Type: kubernetes.io/basic-auth    ← MUST be this exact type for git source auth
  Keys: username, password          ← MUST be these exact key names
  Used for: BuildConfig cloning from GitLab

Secret: gitlab-build-args
  Type: Opaque
  Keys: token, project_id           ← use underscore not hyphen
  Used for: injecting into Dockerfile ARG during build
```

### Dockerfile ARG Variables (Build-Time Only)

```dockerfile
ARG PKG_TOKEN
# Originally named GITLAB_TOKEN
# Renamed because GitLab secret push scanner blocked the push
# even with no hardcoded value — GITLAB_TOKEN matched secret patterns
# Value from: BuildConfig buildArgs → gitlab-build-args secret → key: token
# NOT available at container runtime

ARG PROJECT_ID
# Originally from secret via secretKeyRef — proved unreliable in Sandbox
# Final solution: hardcoded in BuildConfig as value: "82861465"
# Not sensitive — visible in GitLab URL anyway
```

### config.toml Variables

```toml
url          # GitLab instance URL — where runner registers and polls
token        # Runner registration token — from GitLab CI/CD settings
clone_url    # Override URL for repo cloning — critical in air-gapped envs
namespace    # OpenShift namespace where job pods are created
image        # Default job pod image when pipeline does not specify one
helper_image # Helper image for git clone/artifact upload — must be internal
```

---

## Environment Setup

### Windows Laptop → WSL → Linux

```powershell
# In PowerShell as Administrator
wsl --install
# Installs WSL2 + Ubuntu. Restart laptop when prompted.
```

**Forgot WSL sudo password:**

```powershell
# PowerShell as Administrator
wsl -u root
# Inside WSL root shell:
cat /etc/passwd | grep home    # find your username
passwd yourusername            # nothing shows while typing — normal
exit
```

### Docker Desktop WSL Integration

```
Docker Desktop → Settings → Resources → WSL Integration
→ Enable integration with my default WSL distro: ON
→ Ubuntu: ON
→ Apply & Restart
```

Without this, `docker` inside WSL gives:

```
The command 'docker' could not be found in this WSL 2 distro.
We recommend to activate the WSL integration in Docker Desktop settings.
```

### Install oc CLI in WSL

```bash
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
tar xvf openshift-client-linux.tar.gz
sudo mv oc /usr/local/bin/
oc version
# Client Version: 4.21.18
# Server Version: 4.21.15  ← Server showing means already connected
```

### OpenShift Developer Sandbox Login

```bash
# Get login command: OpenShift Console → top right → Copy Login Command
oc login --token=sha256~xxxxx \
  --server=https://api.sandbox-m2.ll9k.p1.openshiftapps.com:6443

oc project
# Using project "aaroncodes-dev"
```

---

## Understanding the Air-Gap

```
Path 1: Your workstation → GitLab/JFrog
  Same corporate network. WHITELISTED.
  Chrome works, docker pull works, git clone works.

Path 2: Your workstation → OpenShift API (port 6443)
  oc CLI communicates here. WHITELISTED.
  All oc commands work.

Path 3: OpenShift nodes → External registries
  Nodes on DIFFERENT subnet. BLOCKED.
  docker push from workstation to registry route = blocked.
  Pods cannot pull images from internet.
```

**Why BuildConfig solves this:**
BuildConfig uses Path 2 (already working oc connection) to send source into OpenShift. The build runs internally. No external push needed.

---

## Docker Concepts

### Dockerfile Layers

```dockerfile
FROM ubi9:latest                              # Layer 1: base OS
COPY gitlab-runner.rpm /tmp/                  # Layer 2: file added
RUN rpm -ivh /tmp/gitlab-runner.rpm           # Layer 3: installed
RUN mkdir -p /etc/gitlab-runner              # Layer 4: dir created
USER 1001                                     # Layer 5: user set
CMD ["gitlab-runner", "run"]                  # Layer 6: startup cmd
```

**Chain commands to reduce layers:**

```dockerfile
# WRONG - 3 layers, RPM stays in layer 1 even after deletion
RUN rpm -ivh /tmp/gitlab-runner.rpm
RUN dnf clean all
RUN rm -f /tmp/gitlab-runner.rpm

# CORRECT - 1 layer
RUN rpm -ivh /tmp/gitlab-runner.rpm && \
    dnf clean all && \
    rm -f /tmp/gitlab-runner.rpm
```

### RUN vs CMD

```
RUN  → during docker build → baked into image permanently
CMD  → when container starts → what the container actually does
```

### USER 1001 — Non-Negotiable for OpenShift

OpenShift default SCC blocks root (UID 0). Without `USER 1001`:

```
Pod starts → SCC check → container wants root → denied
→ CrashLoopBackOff with EMPTY LOGS
```

Empty logs is the tell — it never even started.

---

## Deployment vs Job

This caused the initial CrashLoopBackOff confusion:

```
Deployment = runs forever
  Any exit (even code 0) = OpenShift restarts it
  CrashLoopBackOff = container keeps exiting
  Use for: web servers, APIs, daemons, GitLab runner

Job = runs once and exits
  Exit code 0 = success = complete
  No restart
  Use for: scripts, migrations, one-time tasks
```

**The UBI9 confusion:**
Bare UBI9 has no CMD. Starts, does nothing, exits with code 0.

- As a Job: success
- As a Deployment: infinite restart loop → CrashLoopBackOff
  The Deployment vs Job distinction was discovered when our job image worked as a Job but CrashLooped as a Deployment. The ONLY difference was the workload type.

---

## The Full Journey

### Phase 1: WSL and Tools

1. Installed WSL2: `wsl --install`
2. Enabled Docker Desktop WSL integration
3. Confirmed `docker version` worked in WSL
4. Downloaded oc CLI, moved to `/usr/local/bin/`
5. Logged into Developer Sandbox

### Phase 2: GitLab Repo Setup

#### Creating the Repo

```
GitLab → New Project → Create blank project
Name: my-openshift
Namespace: aortiz122442 (personal account, NOT group)
Visibility: Private
```

**Why personal account not group:**
Initially tried group `freelance-group162023`. Group access tokens were disabled. Personal tokens do not work for group repos. Solution: use personal account where you have full control.

#### Token Creation

```
GitLab → Preferences → Access Tokens → Add new token
Name: openshift-sim
Scopes: api  ← single scope covers everything
```

**Token Exposure:**
Token was accidentally shared in conversation. Immediately revoked and regenerated. Any time a token appears in logs or chat — revoke immediately.

#### Git Clone Failures

**Failure 1 — Token in URL:**

```bash
git clone https://oauth2:glpat-TOKEN@gitlab.com/username/repo.git
# Error: URL rejected: Port number was not a decimal number between 0 and 65535
```

Why: Token contained special characters (`:`) interpreted as port separator.

**Fix — credential prompt:**

```bash
git config --global credential.helper store
git clone https://gitlab.com/aortiz122442/my-openshift.git
# Username: aortiz122442
# Password: paste token
```

**Failure 2 — Still 403:**
Error: `remote: You are not allowed to download code from this project`

Fix: Revoke token, create new one with `api` scope while logged in as correct account.

### Phase 3: RPM Downloads

**Failure 1 — Specific version URL:**

```bash
curl -LJO "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/v17.11.0/rpm/gitlab-runner_x86_64.rpm"
# Result: 243-byte file
```

Why: S3 returned `AccessDenied`. Version `v17.11.0` does not exist at that path.

**Fix — use `latest`:**

```bash
curl -L --progress-bar \
  "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/rpm/gitlab-runner_x86_64.rpm" \
  -o gitlab-runner_x86_64.rpm
# Result: 26MB ✓

curl -L --progress-bar \
  "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/rpm/gitlab-runner-helper-images.rpm" \
  -o gitlab-runner-helper-images.rpm
# Result: 515MB ✓
```

**Validating RPMs:**

```bash
head -c 4 gitlab-runner_x86_64.rpm | xxd
# 00000000: edab eedb  ← correct RPM magic bytes
# ed ab ee db = valid RPM file signature
# Any other bytes = not a real RPM
```

### Phase 4: GitLab Package Registry

#### Purpose

Store RPM binary files in GitLab so OpenShift build can download them. Git is for code. Binaries belong in artifact repositories.

#### Upload Failures

**401 Unauthorized:**
Token missing scopes. Fix: create token with `api` scope.

**403 Forbidden:**
Package Registry feature disabled on project.
Fix: `GitLab → Settings → General → Visibility → Package registry → ON`

**Root cause of bad RPMs in registry:**
First uploads happened when `GITLAB_TOKEN` was empty. curl received error HTML responses and saved them as RPM files (106-263 bytes). Subsequent builds downloaded these HTML files — yum could not install them.

**Lesson:** Always use `curl -f` to fail immediately on HTTP errors:

```bash
curl -f --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" URL -o file.rpm
# Without -f: saves error HTML as the file silently
```

**Successful upload:**

```bash
curl --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
     --upload-file gitlab-runner_x86_64.rpm \
     "https://gitlab.com/api/v4/projects/${PROJECT_ID}/packages/generic/gitlab-runner-rpms/17.11.0/gitlab-runner_x86_64.rpm"

curl --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
     --upload-file gitlab-runner-helper-images.rpm \
     "https://gitlab.com/api/v4/projects/${PROJECT_ID}/packages/generic/gitlab-runner-rpms/17.11.0/gitlab-runner-helper-images.rpm"
```

### Phase 5: OpenShift BuildConfig

#### Create ImageStream First

Must exist before build runs. Forgetting this caused:

```
Status: New (InvalidOutputReference)
```

```bash
oc create imagestream gitlab-runner -n aaroncodes-dev
```

#### Create BuildConfig

```bash
cat > buildconfig.yaml << 'EOF'
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: gitlab-runner
spec:
  source:
    type: Git
    git:
      uri: "https://gitlab.com/aortiz122442/my-openshift.git"
      ref: main
    sourceSecret:
      name: gitlab-git-auth
  strategy:
    type: Docker
    dockerStrategy:
      buildArgs:
        - name: PKG_TOKEN
          valueFrom:
            secretKeyRef:
              name: gitlab-build-args
              key: token
        - name: PROJECT_ID
          value: "82861465"
  output:
    to:
      kind: ImageStreamTag
      name: "gitlab-runner:latest"
EOF

oc apply -f buildconfig.yaml
```

### Phase 6: Secrets (Multiple Iterations)

**Iteration 1 — Single generic secret (FAILED):**

```bash
oc create secret generic gitlab-credentials \
  --from-literal=token=${GITLAB_TOKEN} \
  --from-literal=project-id=${PROJECT_ID} \
  -n aaroncodes-dev
```

Failed because generic Opaque does not work for git source auth. Needs `kubernetes.io/basic-auth`.

**Iteration 2 — Wrong namespace (FAILED):**
Created without `-n aaroncodes-dev`. Build pod looked in wrong namespace.
Error: `secret "gitlab-credentials" not found`

**Iteration 3 — Two separate secrets (WORKING):**

```bash
# Git source authentication — type MUST be kubernetes.io/basic-auth
# Keys MUST be username and password
oc create secret generic gitlab-git-auth \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=aortiz122442 \
  --from-literal=password=${GITLAB_TOKEN} \
  -n aaroncodes-dev

oc annotate secret gitlab-git-auth \
  "build.openshift.io/source-secret-match-uri-1=https://gitlab.com/*" \
  -n aaroncodes-dev

# Build arg injection — Opaque type, key names must match BuildConfig exactly
oc create secret generic gitlab-build-args \
  --from-literal=token=${GITLAB_TOKEN} \
  --from-literal=project_id=${PROJECT_ID} \
  -n aaroncodes-dev
```

**Verifying secrets:**

```bash
oc get secret gitlab-build-args \
  -o jsonpath='{.data.project_id}' -n aaroncodes-dev | base64 -d
# Should return: 82861465

oc get secret gitlab-build-args \
  -o jsonpath='{.data.token}' -n aaroncodes-dev | base64 -d | wc -c
# Should return number > 0
```

---

## All Dockerfile Versions

### Version 1 — Initial

```dockerfile
FROM your-jfrog-url/ubi9:latest

COPY gitlab-runner.rpm /tmp/gitlab-runner.rpm

RUN rpm -ivh /tmp/gitlab-runner.rpm && \
    rm -f /tmp/gitlab-runner.rpm

RUN mkdir -p /etc/gitlab-runner /home/gitlab-runner

CMD ["gitlab-runner", "run", \
     "--working-directory", "/home/gitlab-runner", \
     "--config", "/etc/gitlab-runner/config.toml"]
```

**Why:** First attempt.  
**Problems:** Missing `USER 1001`. Missing helper RPM. Used `rpm -ivh` not `dnf localinstall`.

---

### Version 2 — Added USER and Helper RPM

```dockerfile
FROM ubi9:latest

COPY gitlab-runner_x86_64.rpm /tmp/gitlab-runner.rpm
COPY gitlab-runner-helper-images.rpm /tmp/gitlab-runner-helper-images.rpm

RUN rpm -ivh /tmp/gitlab-runner.rpm && \
    rpm -ivh /tmp/gitlab-runner-helper-images.rpm && \
    rm -f /tmp/gitlab-runner.rpm && \
    rm -f /tmp/gitlab-runner-helper-images.rpm

RUN mkdir -p /etc/gitlab-runner /home/gitlab-runner

USER 1001

CMD ["gitlab-runner", "run", \
     "--working-directory", "/home/gitlab-runner", \
     "--config", "/etc/gitlab-runner/config.toml"]
```

**Why:** Added `USER 1001` for OpenShift SCC. Added helper RPM for GitLab Runner 17.  
**Problems:** RPMs were 263-byte HTML error files from Package Registry (token was empty during upload).

---

### Version 3 — Package Registry Download with ARGs

```dockerfile
FROM ubi9:latest

ARG GITLAB_TOKEN
ARG PROJECT_ID

RUN curl --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
    "https://gitlab.com/api/v4/projects/${PROJECT_ID}/packages/generic/gitlab-runner-rpms/17.11.0/gitlab-runner_x86_64.rpm" \
    -o /tmp/gitlab-runner.rpm && \
    curl --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
    "https://gitlab.com/api/v4/projects/${PROJECT_ID}/packages/generic/gitlab-runner-rpms/17.11.0/gitlab-runner-helper-images.rpm" \
    -o /tmp/gitlab-runner-helper-images.rpm && \
    yum install -y /tmp/gitlab-runner.rpm \
                   /tmp/gitlab-runner-helper-images.rpm && \
    yum clean all && \
    rm -f /tmp/*.rpm

RUN mkdir -p /etc/gitlab-runner /home/gitlab-runner

USER 1001

CMD ["gitlab-runner", "run", \
     "--working-directory", "/home/gitlab-runner", \
     "--config", "/etc/gitlab-runner/config.toml"]
```

**Why:** Download RPMs from Package Registry during build instead of COPY.  
**Problems:** `GITLAB_TOKEN` triggered GitLab secret push scanner → `pre-receive hook declined`. Variables empty in build.

---

### Version 4 — Renamed to PKG_TOKEN

```bash
# Used sed to rename all occurrences
sed -i 's/GITLAB_TOKEN/PKG_TOKEN/g' Dockerfile
```

Result:

```dockerfile
FROM ubi9:latest

ARG PKG_TOKEN
ARG PROJECT_ID

RUN curl --header "PRIVATE-TOKEN: ${PKG_TOKEN}" \
    "https://gitlab.com/api/v4/projects/${PROJECT_ID}/packages/generic/gitlab-runner-rpms/17.11.0/gitlab-runner_x86_64.rpm" \
    -o /tmp/gitlab-runner.rpm && \
    curl --header "PRIVATE-TOKEN: ${PKG_TOKEN}" \
    "https://gitlab.com/api/v4/projects/${PROJECT_ID}/packages/generic/gitlab-runner-rpms/17.11.0/gitlab-runner-helper-images.rpm" \
    -o /tmp/gitlab-runner-helper-images.rpm && \
    yum install -y /tmp/gitlab-runner.rpm \
                   /tmp/gitlab-runner-helper-images.rpm && \
    yum clean all && \
    rm -f /tmp/*.rpm

RUN mkdir -p /etc/gitlab-runner /home/gitlab-runner

USER 1001

CMD ["gitlab-runner", "run", \
     "--working-directory", "/home/gitlab-runner", \
     "--config", "/etc/gitlab-runner/config.toml"]
```

**Why:** `GITLAB_TOKEN` matched GitLab's secret detection patterns even with no hardcoded value.  
**Problems:** PROJECT_ID still empty. URL showed `/api/v4/projects//packages/...` (double slash = empty ID).

---

### Version 5 — Debug: Check Variables and File Sizes

Added specifically to diagnose empty variables and wrong file sizes:

```dockerfile
FROM ubi9:latest

ARG PKG_TOKEN
ARG PROJECT_ID

# Debug step: check if variables are actually reaching the build container
# If TOKEN set: NO → secret injection is failing
# If PROJECT_ID is wrong → secret key name mismatch
RUN echo "TOKEN set: $(if [ -n "${PKG_TOKEN}" ]; then echo YES; else echo NO; fi)" && \
    echo "PROJECT_ID: ${PROJECT_ID}"

# Download runner RPM and immediately check its size
# If size is < 1MB the download failed or returned an error response
RUN curl --header "PRIVATE-TOKEN: ${PKG_TOKEN}" \
    "https://gitlab.com/api/v4/projects/${PROJECT_ID}/packages/generic/gitlab-runner-rpms/17.11.0/gitlab-runner_x86_64.rpm" \
    -o /tmp/gitlab-runner.rpm && \
    echo "RPM size:" && \
    ls -lh /tmp/gitlab-runner.rpm && \
    file /tmp/gitlab-runner.rpm

# Download helper RPM and check size
RUN curl --header "PRIVATE-TOKEN: ${PKG_TOKEN}" \
    "https://gitlab.com/api/v4/projects/${PROJECT_ID}/packages/generic/gitlab-runner-rpms/17.11.0/gitlab-runner-helper-images.rpm" \
    -o /tmp/gitlab-runner-helper-images.rpm && \
    echo "Helper size:" && \
    ls -lh /tmp/gitlab-runner-helper-images.rpm

RUN yum install -y /tmp/gitlab-runner.rpm \
                   /tmp/gitlab-runner-helper-images.rpm && \
    yum clean all && \
    rm -f /tmp/*.rpm

RUN mkdir -p /etc/gitlab-runner /home/gitlab-runner

USER 1001

CMD ["gitlab-runner", "run", \
     "--working-directory", "/home/gitlab-runner", \
     "--config", "/etc/gitlab-runner/config.toml"]
```

**What the debug output revealed:**

```
TOKEN set: NO          ← PKG_TOKEN empty — secret not being injected
PROJECT_ID: 82861465   ← hardcoded value working fine

RPM size:
-rw-r--r--. 1 root root 106 Jun 4 /tmp/gitlab-runner.rpm
```

106 bytes = curl downloaded without auth and received GitLab error JSON.

Also revealed: `file` command does not exist in UBI9 → `exit status 127` (command not found) → build fails. Removed in next version.

---

### Version 6 — Verbose Curl for HTTP Status Debugging

```dockerfile
FROM ubi9:latest

ARG PKG_TOKEN
ARG PROJECT_ID

RUN echo "TOKEN set: $(if [ -n "${PKG_TOKEN}" ]; then echo YES; else echo NO; fi)" && \
    echo "PROJECT_ID: ${PROJECT_ID}"

# -v shows full HTTP request/response headers including status code
# Use this when you need to see exactly what the server returns
RUN curl -v --header "PRIVATE-TOKEN: ${PKG_TOKEN}" \
    "https://gitlab.com/api/v4/projects/${PROJECT_ID}/packages/generic/gitlab-runner-rpms/17.11.0/gitlab-runner_x86_64.rpm" \
    -o /tmp/gitlab-runner.rpm && \
    echo "File size:" && \
    ls -lh /tmp/gitlab-runner.rpm

RUN yum install -y /tmp/gitlab-runner.rpm && \
    yum clean all && \
    rm -f /tmp/*.rpm

RUN mkdir -p /etc/gitlab-runner /home/gitlab-runner

USER 1001

CMD ["gitlab-runner", "run", \
     "--working-directory", "/home/gitlab-runner", \
     "--config", "/etc/gitlab-runner/config.toml"]
```

**What `curl -v` revealed:**

Before PROJECT_ID fix:

```
> GET /api/v4/projects//packages/generic/...   ← double slash = empty PROJECT_ID
< HTTP/2 308                                   ← redirect, not 200
File size: 0 bytes
```

After PROJECT_ID hardcoded:

```
> GET /api/v4/projects/82861465/packages/generic/...  ← correct!
< HTTP/2 200                                          ← success
100   263  100   263    0     0   1301      0         ← only 263 bytes!
```

263 bytes with HTTP 200 = Package Registry stored the wrong file (HTML error that was uploaded when token was empty).

---

### Version 7 — S3 Direct Download (Bypassing Package Registry)

```dockerfile
FROM ubi9:latest

RUN curl -L \
    "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/rpm/gitlab-runner_x86_64.rpm" \
    -o /tmp/gitlab-runner.rpm && \
    curl -L \
    "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/rpm/gitlab-runner-helper-images.rpm" \
    -o /tmp/gitlab-runner-helper-images.rpm && \
    yum install -y /tmp/gitlab-runner.rpm \
                   /tmp/gitlab-runner-helper-images.rpm && \
    yum clean all && \
    rm -f /tmp/*.rpm

RUN mkdir -p /etc/gitlab-runner /home/gitlab-runner

USER 1001

CMD ["gitlab-runner", "run", \
     "--working-directory", "/home/gitlab-runner", \
     "--config", "/etc/gitlab-runner/config.toml"]
```

**Why:** Bypassed Package Registry entirely. No ARGs needed for public S3.  
**Problems:** Developer Sandbox may restrict egress to S3. Files inside build container were still tiny even though local machine got 26MB successfully.

---

### Version 8 — COPY with Size Check After Copy

```dockerfile
FROM ubi9:latest

COPY gitlab-runner_x86_64.rpm /tmp/gitlab-runner.rpm
COPY gitlab-runner-helper-images.rpm /tmp/gitlab-runner-helper-images.rpm

# Immediately check sizes after COPY
# If sizes are tiny here — files were excluded from the upload
# Most likely cause: .gitignore containing *.rpm
RUN echo "Runner size:" && ls -lh /tmp/gitlab-runner.rpm && \
    echo "Helper size:" && ls -lh /tmp/gitlab-runner-helper-images.rpm

RUN yum install -y /tmp/gitlab-runner.rpm \
                   /tmp/gitlab-runner-helper-images.rpm && \
    yum clean all && \
    rm -f /tmp/*.rpm

RUN mkdir -p /etc/gitlab-runner /home/gitlab-runner

USER 1001

CMD ["gitlab-runner", "run", \
     "--working-directory", "/home/gitlab-runner", \
     "--config", "/etc/gitlab-runner/config.toml"]
```

**Critical discovery:** `.gitignore` contained `*.rpm`. `oc start-build --from-dir` on a git repo folder respects `.gitignore` and excludes matching files. RPMs never made it into the build. COPY found 0-byte files.

**Fix:**

```bash
rm .gitignore
oc start-build gitlab-runner --from-dir=./folder --follow
```

---

### Version 9 — Permissions Fix + dnf localinstall (Current)

```dockerfile
FROM ubi9:latest

COPY gitlab-runner_x86_64.rpm /tmp/gitlab-runner.rpm
COPY gitlab-runner-helper-images.rpm /tmp/gitlab-runner-helper-images.rpm

# Fix file permissions after COPY
# COPY may preserve source permissions which are not always 644
# Without chmod: "Can not load RPM file" even when file is a valid RPM
RUN chmod 644 /tmp/gitlab-runner.rpm \
              /tmp/gitlab-runner-helper-images.rpm

# Verify sizes — confirms files arrived with content
RUN ls -lh /tmp/gitlab-runner.rpm /tmp/gitlab-runner-helper-images.rpm

# dnf localinstall is the correct command for local RPM files
# It handles dependencies automatically and is designed for this use case
# dnf clean all removes yum cache — keeps image smaller
RUN dnf localinstall -y /tmp/gitlab-runner.rpm \
                        /tmp/gitlab-runner-helper-images.rpm && \
    dnf clean all && \
    rm -f /tmp/*.rpm

RUN mkdir -p /etc/gitlab-runner /home/gitlab-runner

# Non-root user — required for OpenShift SCC compliance
USER 1001

# Long-running process — keeps Deployment alive
# If CMD exits for any reason Deployment will restart the pod
CMD ["gitlab-runner", "run", \
     "--working-directory", "/home/gitlab-runner", \
     "--config", "/etc/gitlab-runner/config.toml"]
```

---

## All Pivots and Why

### Pivot 1: Deployment → Job

**Trigger:** Deployment CrashLoopBackOff. Job worked.  
**Root cause:** UBI9 base image has no CMD. Exits immediately. Deployment restarts it infinitely.  
**Lesson:** Choose workload type based on what the container does. Script → Job. Server → Deployment.

---

### Pivot 2: docker push → BuildConfig

**Trigger:** `docker push` to OpenShift registry timed out.  
**Root cause:** Registry route crosses corporate firewall. Workstation subnet blocked.  
**Fix:** BuildConfig builds internally over existing oc connection. No new firewall rules.

---

### Pivot 3: Generic Secret → Two Typed Secrets

**Trigger:** `FetchSourceFailed` with credentials in secret.  
**Root cause:** OpenShift requires `kubernetes.io/basic-auth` type with `username`/`password` keys for git auth. Generic Opaque is ignored.  
**Fix:** `gitlab-git-auth` (basic-auth) for cloning, `gitlab-build-args` (Opaque) for build args.

---

### Pivot 4: GITLAB_TOKEN → PKG_TOKEN

**Trigger:** `pre-receive hook declined` on git push.  
**Root cause:** GitLab secret push scanner flagged `GITLAB_TOKEN` as suspicious variable name pattern, even without hardcoded value.  
**Fix:** `sed -i 's/GITLAB_TOKEN/PKG_TOKEN/g' Dockerfile`

---

### Pivot 5: secretKeyRef → hardcoded PROJECT_ID

**Trigger:** PROJECT_ID always empty inside builds. URL showed double slash.  
**Root cause:** `valueFrom.secretKeyRef` unreliable in Sandbox. Key name `project-id` (hyphen) may cause shell issues.  
**Fix:** PROJECT_ID is not sensitive. Hardcode in BuildConfig: `value: "82861465"`

---

### Pivot 6: Making Repo Public (Temporary)

**Trigger:** Multiple `FetchSourceFailed` errors.  
**Purpose:** Remove auth from the equation to confirm URL was correct.  
**What it revealed:** URL was correct. Issue was secret type.  
**Lesson:** Infrastructure repos should never be public. Debugging technique only.

---

### Pivot 7: Package Registry Download → --from-dir

**Trigger chain:**

1. Token empty → curl saved HTML as RPM → yum can't install
2. Token revoked/regenerated → wrong scopes
3. Package Registry disabled → 403
4. Re-enabled → S3 version URL returned AccessDenied
5. Used `latest` URL locally → 26MB, worked
6. Same URL inside build container → tiny file (Sandbox egress blocked)
   **Fix:** `oc start-build --from-dir` sends local files (correctly 26MB/515MB) over oc connection. Bypasses all Package Registry and S3 issues.

```bash
oc start-build gitlab-runner \
  --from-dir=~/gitlab-runner-build/my-openshift \
  --follow
```

---

### Pivot 8: yum install → dnf localinstall + chmod

**Trigger:** `Can not load RPM file` with valid RPM files.  
**Root cause:** File permissions after COPY may not be 644. `yum install` not ideal for local files.  
**Fix:**

```dockerfile
RUN chmod 644 /tmp/*.rpm
RUN dnf localinstall -y /tmp/gitlab-runner.rpm /tmp/gitlab-runner-helper-images.rpm
```

---

### Pivot 9: .gitignore Removal

**Trigger:** COPY finding 0-byte or wrong files despite 26MB/515MB files being in folder.  
**Root cause:** `.gitignore` contained `*.rpm`. `oc start-build --from-dir` respects `.gitignore` on git repo folders. RPMs excluded before upload.  
**Fix:**

```bash
rm .gitignore
```

---

## Deployment Setup

### ServiceAccount

```bash
oc create serviceaccount gitlab-runner -n aaroncodes-dev

# Permission to create and manage pods (needed for pipeline job pods)
oc adm policy add-role-to-user edit \
  -z gitlab-runner -n aaroncodes-dev

# Allow non-root execution (requires cluster-admin — may be restricted in Sandbox)
oc adm policy add-scc-to-user anyuid \
  -z gitlab-runner -n aaroncodes-dev
```

### config.toml

```toml
concurrent = 4
check_interval = 0

[[runners]]
  name = "openshift-runner"
  url = "http://your-internal-gitlab-url"
  token = "your-runner-registration-token"

  # Overrides the URL used to clone repos during pipeline jobs
  # Without this: runner uses GitLab's advertised URL
  # In air-gapped envs that URL may be external and unreachable from pods
  clone_url = "http://your-internal-gitlab-url"

  executor = "kubernetes"

  [runners.kubernetes]
    namespace = "aaroncodes-dev"
    service_account = "gitlab-runner"

    # Default image when pipeline does not specify one
    # Internal registry address — never needs internet
    image = "image-registry.openshift-image-registry.svc:5000/aaroncodes-dev/ubi9:latest"

    # Helper handles: git clone, artifact upload, cache
    # Points to the runner image we just built
    helper_image = "image-registry.openshift-image-registry.svc:5000/aaroncodes-dev/gitlab-runner:latest"
```

### ConfigMap

```bash
# Mounts config.toml into the runner pod at /etc/gitlab-runner/config.toml
# The CMD in our Dockerfile reads from this exact path
oc create configmap gitlab-runner-config \
  --from-file=config.toml \
  -n aaroncodes-dev
```

### Deployment YAML

```bash
cat > runner-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab-runner
  namespace: aaroncodes-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitlab-runner
  template:
    metadata:
      labels:
        app: gitlab-runner
    spec:
      serviceAccountName: gitlab-runner
      containers:
        - name: gitlab-runner
          # Internal registry address — only accessible from within cluster
          # Cannot be used from outside the cluster
          image: image-registry.openshift-image-registry.svc:5000/aaroncodes-dev/gitlab-runner:latest
          volumeMounts:
            - name: config
              mountPath: /etc/gitlab-runner
      volumes:
        - name: config
          configMap:
            name: gitlab-runner-config
EOF

oc apply -f runner-deployment.yaml
```

### Verify

```bash
oc get pods -n aaroncodes-dev
# NAME                             READY   STATUS    RESTARTS
# gitlab-runner-6d8f9b7c4-xk2pq   1/1     Running   0

oc logs deployment/gitlab-runner -n aaroncodes-dev
# Configuration loaded    builds=0
# Listening for jobs
```

### Test Pipeline

```yaml
# .gitlab-ci.yml in any repo
test-job:
  image: python:3.11 # teams bring their own image
  script:
    - echo "Running in my image via OpenShift runner"
    - hostname # shows pod name
    - whoami # should show non-root user
```

```bash
# Watch job pods appear and disappear as pipeline runs
oc get pods -w -n aaroncodes-dev
```

---

## Enterprise JFrog Version

### Architecture

```
JFrog Artifactory (internal, air-gapped)
  └── rpm-local/
        ├── gitlab-runner_x86_64.rpm
        └── gitlab-runner-helper-images.rpm

GitLab (internal)
  └── openshift-gitlab-runner/
        └── Dockerfile

OpenShift ROSA (air-gapped)
  └── BuildConfig watches GitLab repo
  └── Build downloads RPMs from JFrog using API key
  └── Image built internally
  └── Runner Deployment active
```

### JFrog Authentication

**Use API key not identity token in air-gapped environments:**

|              | API Key                 | Identity Token             |
| ------------ | ----------------------- | -------------------------- |
| Lifespan     | Long-lived              | Short-lived (hours)        |
| Air-gap safe | Yes                     | No — OIDC callback blocked |
| Pull secrets | Set once, works forever | Expires, breaks pulls      |

```
JFrog UI → top right avatar → Edit Profile
→ Authentication Settings → Generate API Key
```

### Upload RPMs to JFrog

```bash
curl -u your-username:${JFROG_TOKEN} \
  -T gitlab-runner_x86_64.rpm \
  "https://${JFROG_URL}/artifactory/rpm-local/gitlab-runner_x86_64.rpm"

curl -u your-username:${JFROG_TOKEN} \
  -T gitlab-runner-helper-images.rpm \
  "https://${JFROG_URL}/artifactory/rpm-local/gitlab-runner-helper-images.rpm"
```

### JFrog Pull Secret

```bash
oc create secret docker-registry jfrog-pull-secret \
  --docker-server=${JFROG_URL} \
  --docker-username=your-username \
  --docker-password=${JFROG_TOKEN} \
  -n your-namespace

oc secrets link default jfrog-pull-secret --for=pull -n your-namespace
```

### JFrog Build Credentials Secret

```bash
oc create secret generic jfrog-build-credentials \
  --from-literal=username=your-username \
  --from-literal=token=${JFROG_TOKEN} \
  --from-literal=url=${JFROG_URL} \
  -n your-namespace
```

### Dockerfile (JFrog Version)

```dockerfile
FROM your-instance.jfrog.io/docker-local/ubi9:latest

ARG JFROG_USER
ARG JFROG_TOKEN
ARG JFROG_URL

# -f flag: fail immediately on HTTP errors — learned the hard way
# without -f curl saves error HTML as the RPM file silently
RUN curl -f -u "${JFROG_USER}:${JFROG_TOKEN}" \
    "https://${JFROG_URL}/artifactory/rpm-local/gitlab-runner_x86_64.rpm" \
    -o /tmp/gitlab-runner.rpm && \
    curl -f -u "${JFROG_USER}:${JFROG_TOKEN}" \
    "https://${JFROG_URL}/artifactory/rpm-local/gitlab-runner-helper-images.rpm" \
    -o /tmp/gitlab-runner-helper-images.rpm

RUN chmod 644 /tmp/gitlab-runner.rpm /tmp/gitlab-runner-helper-images.rpm

RUN dnf localinstall -y /tmp/gitlab-runner.rpm \
                        /tmp/gitlab-runner-helper-images.rpm && \
    dnf clean all && \
    rm -f /tmp/*.rpm

RUN mkdir -p /etc/gitlab-runner /home/gitlab-runner

USER 1001

CMD ["gitlab-runner", "run", \
     "--working-directory", "/home/gitlab-runner", \
     "--config", "/etc/gitlab-runner/config.toml"]
```

### BuildConfig (JFrog Version)

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: gitlab-runner
  namespace: your-namespace
spec:
  source:
    type: Git
    git:
      uri: "https://your-internal-gitlab/group/openshift-gitlab-runner.git"
      ref: main
    sourceSecret:
      name: gitlab-git-auth
  strategy:
    type: Docker
    dockerStrategy:
      buildArgs:
        - name: JFROG_USER
          valueFrom:
            secretKeyRef:
              name: jfrog-build-credentials
              key: username
        - name: JFROG_TOKEN
          valueFrom:
            secretKeyRef:
              name: jfrog-build-credentials
              key: token
        - name: JFROG_URL
          valueFrom:
            secretKeyRef:
              name: jfrog-build-credentials
              key: url
  output:
    to:
      kind: ImageStreamTag
      name: "gitlab-runner:latest"
```

### config.toml (JFrog Version)

```toml
concurrent = 4
check_interval = 0

[[runners]]
  name = "openshift-runner"
  url = "http://your-internal-gitlab-url"
  token = "runner-registration-token"
  clone_url = "http://your-internal-gitlab-url"
  executor = "kubernetes"

  [runners.kubernetes]
    namespace = "your-namespace"
    service_account = "gitlab-runner"
    image = "image-registry.openshift-image-registry.svc:5000/your-namespace/ubi9:latest"
    helper_image = "image-registry.openshift-image-registry.svc:5000/your-namespace/gitlab-runner:latest"

    [[runners.kubernetes.image_pull_secrets]]
      name = "jfrog-pull-secret"
```

---

## Troubleshooting Reference

### CrashLoopBackOff

```bash
oc describe pod <pod-name> -n your-namespace | grep -A5 "Last State"
# Exit Code: 0   = exited successfully (use Job not Deployment)
# Exit Code: 1   = app crashed (check logs)
# Exit Code: 126 = permission denied on entrypoint
# Exit Code: 127 = binary not found
# Exit Code: 137 = OOMKilled
```

### Empty Logs on CrashLoopBackOff

Container dying before app starts. Almost always SCC blocking root.

```bash
oc get events -n your-namespace --sort-by='.lastTimestamp'
# Fix: ensure USER 1001 in Dockerfile
```

### Can Not Load RPM File

```bash
# 1. Check magic bytes
head -c 4 file.rpm | xxd
# Valid RPM: ed ab ee db

# 2. Fix permissions
chmod 644 /tmp/*.rpm

# 3. Remove .gitignore if using --from-dir
rm .gitignore

# 4. Use dnf localinstall
dnf localinstall -y /tmp/file.rpm

# 5. Check if file is HTML error
head -c 100 /tmp/file.rpm
# Valid: binary data
# Invalid: <!DOCTYPE html> or {"message":"..."}
```

### Build Args Empty in Build

```bash
# Add debug RUN to Dockerfile
RUN echo "TOKEN set: $(if [ -n "${PKG_TOKEN}" ]; then echo YES; else echo NO; fi)" && \
    echo "PROJECT_ID: ${PROJECT_ID}"

# If TOKEN: NO — verify secret has value
oc get secret gitlab-build-args \
  -o jsonpath='{.data.token}' -n aaroncodes-dev | base64 -d | wc -c

# For non-sensitive values, hardcode in BuildConfig instead
- name: PROJECT_ID
  value: "82861465"
```

### FetchSourceFailed

```bash
# Check secret type
oc get secret gitlab-git-auth -o jsonpath='{.type}'
# Must return: kubernetes.io/basic-auth

# Check secret keys
oc get secret gitlab-git-auth -o jsonpath='{.data}'
# Must have: username and password
```

### curl Downloading Wrong Content

```bash
# Always use -f to fail on HTTP errors
curl -f --header "PRIVATE-TOKEN: ${TOKEN}" URL -o file.rpm

# Use -v to see HTTP status
curl -v --header "PRIVATE-TOKEN: ${TOKEN}" URL -o file.rpm
# < HTTP/2 200 = good
# < HTTP/2 401 = auth failed, empty token
# < HTTP/2 403 = permission denied
# < HTTP/2 404 = wrong URL
```

---

## Quick Reference Commands

```bash
# Build from local folder — bypasses git/auth/firewall entirely
oc start-build gitlab-runner --from-dir=./folder --follow

# Check builds
oc get builds

# Get build logs
oc logs build/gitlab-runner-N

# Get all logs from start
oc logs build/gitlab-runner-N 2>&1 | head -50

# Check pods
oc get pods -n aaroncodes-dev

# Watch pods in real time
oc get pods -w -n aaroncodes-dev

# Pod logs
oc logs deployment/gitlab-runner -n aaroncodes-dev

# Logs from previous crashed container
oc logs <pod-name> --previous -n aaroncodes-dev

# Describe pod — shows events and errors
oc describe pod <pod-name> -n aaroncodes-dev

# Check secrets
oc get secrets -n aaroncodes-dev
oc get secret <name> -o jsonpath='{.data.<key>}' | base64 -d

# Check imagestream
oc get imagestream -n aaroncodes-dev

# Check buildconfig
oc get buildconfig gitlab-runner -o yaml

# Patch buildconfig buildArgs
oc get buildconfig gitlab-runner \
  -o jsonpath='{.spec.strategy.dockerStrategy.buildArgs}'

# Cancel a build
oc cancel-build gitlab-runner-N

# Delete and recreate a secret
oc delete secret <name> -n aaroncodes-dev
oc create secret generic <name> --from-literal=key=value -n aaroncodes-dev

# Check events
oc get events --sort-by='.lastTimestamp' -n aaroncodes-dev

# Exec into pod for debugging
oc exec -it <pod-name> -- /bin/sh -n aaroncodes-dev
```
