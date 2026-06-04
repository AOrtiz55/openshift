me · MD

# GitLab Runner on OpenShift — Complete Setup Guide

**Author:** Senior OpenShift Engineer  
**Environment:** Air-gapped (no internet access) | ROSA (Red Hat OpenShift Service on AWS)  
**Goal:** Host a GitLab Runner inside OpenShift so teams can use their own images in CI/CD pipelines

---

## Table of Contents

1. [Overview and Architecture](#overview-and-architecture)
2. [Key Terminology](#key-terminology)
3. [Variable Reference](#variable-reference)
4. [Environment Setup](#environment-setup)
5. [Understanding the Air-Gap Problem](#understanding-the-air-gap-problem)
6. [Docker Concepts](#docker-concepts)
7. [Building the GitLab Runner Image](#building-the-gitlab-runner-image)
8. [OpenShift BuildConfig Setup](#openshift-buildconfig-setup)
9. [Deploying the Runner](#deploying-the-runner)
10. [Pivots and Why We Made Them](#pivots-and-why-we-made-them)
11. [Enterprise JFrog Example](#enterprise-jfrog-example)
12. [Troubleshooting Reference](#troubleshooting-reference)

---

## Overview and Architecture

### What We Are Building

A GitLab Runner hosted inside OpenShift that allows development teams to run CI/CD pipelines using their own custom Docker images — without any container needing to reach the internet.

```
GitLab (internal)
        │
        │  "I have a pipeline job"
        ▼
GitLab Runner Pod (Deployment — runs forever)
        │
        │  spins up a new job pod per pipeline
        ▼
Job Pod (runs team's image, exits when done)
        │
        ▼
Results reported back to GitLab ✓
```

### Why OpenShift Instead of EC2

| EC2 Runner                     | OpenShift Runner               |
| ------------------------------ | ------------------------------ |
| Installed as a Linux service   | Deployed as a pod (Deployment) |
| You manage the OS              | OpenShift manages the pod      |
| Crashes require manual restart | Self-healing — auto restarts   |
| One machine                    | Scales automatically           |
| Shell or Docker executor       | Kubernetes executor            |

### What Teams Get

Once the runner is deployed, each team defines their own image in `.gitlab-ci.yml`:

```yaml
# Each team's pipeline — completely independent of your runner image
job:
  image: your-registry/their-custom-image:latest
  script:
    - echo "Running in my own image"
    - python3 myscript.py
```

Your runner receives the job, spins up a pod using their image, runs it, and reports back. The runner image and the job image are completely separate.

---

## Key Terminology

### Docker / Container Concepts

**Image** — A read-only blueprint of a container. Stored in a registry. Built from a Dockerfile. Think of it as a recipe — it never changes.

**Container** — A running instance of an image. Think of it as the meal made from the recipe. One image can spawn many containers.

**Dockerfile** — A text file containing instructions that Docker executes top to bottom to build an image. Each instruction creates one layer.

**Layer** — A cached snapshot of changes made by one Dockerfile instruction. Layers stack on top of each other to form the final image.

**Registry** — A storage server for images. Examples: JFrog Artifactory, OpenShift internal registry, Docker Hub.

**docker build** — The command that reads a Dockerfile and produces an image on your local machine.

**docker push** — The command that uploads a locally built image to a registry.

**docker pull** — The command that downloads an image from a registry to your local machine.

**UBI9** — Universal Base Image 9. A minimal Red Hat Enterprise Linux 9 OS packaged as a container base image. Built and maintained by Red Hat. Free to use. Preferred in enterprise OpenShift environments because it is RHEL-compatible and security-certified.

### OpenShift Concepts

**Pod** — The smallest deployable unit in OpenShift/Kubernetes. Wraps one or more containers.

**Deployment** — Manages pods that need to run continuously (web servers, APIs, daemons). If a pod exits for any reason, the Deployment restarts it. This is why a script-based container will CrashLoopBackOff — it exits on purpose but the Deployment thinks something went wrong.

**Job** — Manages pods designed to run once and exit. When the container exits with code 0, the Job marks it complete. No restart loop.

**CronJob** — Like a Job but triggered on a schedule (e.g. every night at midnight).

**CrashLoopBackOff** — OpenShift error state meaning a pod keeps crashing and restarting. Usually means the container exits unexpectedly. Most common causes: wrong workload type (should be a Job), missing config, running as root when not allowed.

**BuildConfig** — An OpenShift resource that defines how to build a container image from source code. Can watch a Git repo and trigger builds automatically.

**ImageStream** — OpenShift's internal image tracking system. A pointer to image versions stored in the internal registry. BuildConfigs output to ImageStreams.

**ServiceAccount** — An identity that pods use to make API calls within the cluster. The GitLab runner needs a ServiceAccount with permission to create job pods.

**ConfigMap** — Stores non-sensitive configuration data (like `config.toml`) and mounts it into pods as files.

**Secret** — Stores sensitive data (tokens, passwords) and makes them available to pods. Base64 encoded at rest.

**SCC (Security Context Constraint)** — OpenShift's security policy for pods. The default `restricted` SCC blocks containers that run as root. This is why `USER 1001` in the Dockerfile is critical.

**Namespace / Project** — An isolated workspace inside the cluster. All your resources live in a namespace. Developer Sandbox gives you one pre-created namespace.

**Route** — An OpenShift resource that exposes a service to external traffic. Used to expose the internal registry for pushing images.

### GitLab Runner Concepts

**Executor** — How the runner actually runs pipeline jobs. Options: Shell (on the runner machine), Docker (in a container on the runner), Kubernetes (in a new pod on the cluster). We use the **Kubernetes executor** because we are on OpenShift.

**Helper Image** — A second image used by GitLab Runner 17+ alongside the job image. Handles: cloning the repo, uploading artifacts, managing cache. Required as a separate RPM package starting in GitLab Runner 17.

**Registration Token** — A secret token from GitLab that proves your runner is authorized to receive jobs for a specific project or group.

**config.toml** — The runner's main configuration file. Defines GitLab URL, token, executor type, namespace, and job image defaults.

**clone_url** — A config.toml setting that overrides the URL the runner uses to clone repositories. Critical in air-gapped environments where the default GitLab URL might not be reachable from inside pods.

---

## Variable Reference

This section explains every variable used in this guide, distinguishing between shell exports (set by you) and references (used by tools/configs).

### Shell Exports (You Set These)

These are set in your terminal session with `export`. They are your credentials and identifiers.

```bash
export GITLAB_TOKEN="glpat-xxxxxxxxxxxx"
# Your GitLab Personal Access Token
# Created at: GitLab → Preferences → Access Tokens
# Required scopes: api, read_repository, write_repository
# Used for: authenticating git clone, uploading to Package Registry
# NEVER commit this to git

export PROJECT_ID="82861465"
# Your GitLab project's numeric ID
# Found at: GitLab → your repo → Settings → General → Project ID
# Used for: GitLab API calls to Package Registry

export JFROG_URL="your-instance.jfrog.io"
# (Enterprise only) Your JFrog Artifactory instance URL
# Used for: pulling base images, downloading RPMs

export JFROG_TOKEN="your-jfrog-api-key"
# (Enterprise only) JFrog API key or identity token
# Created at: JFrog → Edit Profile → Authentication Settings
# Use API key not identity token in air-gapped environments
# Identity tokens are short-lived and cause pull failures
```

### OpenShift Secrets (Stored in Cluster)

These are stored as Kubernetes Secrets and injected into builds/pods at runtime.

```bash
# Secret: gitlab-git-auth
# Type: kubernetes.io/basic-auth
# Keys: username, password
# Used for: BuildConfig cloning from private GitLab repos
# Why basic-auth type: OpenShift requires this specific type for git source auth

# Secret: gitlab-build-args
# Type: Opaque (generic)
# Keys: token, project-id
# Used for: passing credentials into Dockerfile ARG during build
# Note: key names must exactly match what BuildConfig references

# Secret: jfrog-pull-secret (enterprise)
# Type: kubernetes.io/dockerconfigjson
# Used for: OpenShift nodes authenticating to JFrog to pull images
```

### Dockerfile ARGs (Build-Time Variables)

ARG variables exist only during `docker build` / OpenShift build. They are NOT available at container runtime.

```dockerfile
ARG PKG_TOKEN
# Renamed from GITLAB_TOKEN to avoid GitLab's secret push detection
# Value injected by BuildConfig from gitlab-build-args secret
# Used in curl commands to authenticate Package Registry downloads

ARG PROJECT_ID
# GitLab project numeric ID
# Value injected by BuildConfig (hardcoded or from secret)
# Used in Package Registry API URL construction
```

### config.toml Variables

```toml
url = "http://your-internal-gitlab-url"
# The GitLab instance the runner registers with and polls for jobs
# Must be reachable from inside OpenShift pods

token = "your-registration-token"
# NOT your personal access token
# This is the runner registration token from:
# GitLab → Settings → CI/CD → Runners → Registration token
# Obtained by running: gitlab-runner register

clone_url = "http://your-internal-gitlab-url"
# Overrides the URL used to clone repos during pipeline jobs
# Critical in air-gapped environments
# Without this, runner uses whatever URL GitLab advertises
# which may be external/unreachable from inside pods

namespace = "your-namespace"
# The OpenShift namespace where job pods will be created
# Must match where the runner Deployment lives

image = "registry/namespace/ubi9:latest"
# Default image for job pods when pipeline doesn't specify one
# Should point to internal registry in air-gapped environments

helper_image = "registry/namespace/gitlab-runner:latest"
# The helper image used alongside job pods
# Handles: git clone, artifact upload, cache management
# Must be in internal registry in air-gapped environments
```

---

## Environment Setup

### Prerequisites

On your workstation (WSL on Windows or native Linux):

```bash
# 1. Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# If using Docker Desktop on Windows — enable WSL integration in Docker Desktop settings

# 2. oc CLI
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
tar xvf openshift-client-linux.tar.gz
sudo mv oc /usr/local/bin/
oc version

# 3. Login to OpenShift
# Get login command from: OpenShift Console → top right → Copy Login Command
oc login --token=sha256~xxxxx \
  --server=https://api.your-cluster.openshiftapps.com:6443

# 4. Check your namespace
oc project
```

### Persist Variables Across Sessions

WSL clears environment variables when the terminal closes. Persist them:

```bash
echo 'export GITLAB_TOKEN="your-token"' >> ~/.bashrc
echo 'export PROJECT_ID="your-project-id"' >> ~/.bashrc
source ~/.bashrc
```

---

## Understanding the Air-Gap Problem

This is the most important concept to understand. There are three separate network paths and they are NOT the same.

```
Path 1: Your workstation → GitLab/JFrog
─────────────────────────────────────────
Your workstation is on the corporate network.
GitLab and JFrog are also on the corporate network.
This path is WHITELISTED. You can browse GitLab in Chrome,
run docker pull from JFrog, and clone git repos.

Path 2: Your workstation → OpenShift API
─────────────────────────────────────────
The oc CLI communicates with the OpenShift API server.
This path is WHITELISTED. That is why oc commands work.

Path 3: OpenShift nodes → External registries
──────────────────────────────────────────────
OpenShift worker nodes live on a DIFFERENT subnet.
Their outbound traffic goes through different firewall rules.
This path is BLOCKED for most external destinations.
This is why pods cannot pull images from the internet
and why docker push to an external registry fails.
```

### Why You Can Open JFrog in Chrome But Pods Can't Pull From It

Your browser and your Docker client share your workstation's network identity. OpenShift nodes have their own network identity on a separate subnet. The whitelist controls traffic by source IP. Your workstation's IP is allowed. The node's IP is not (without explicit whitelisting).

### The Solution: Internal Registry

```
Step 1: Pull image to your workstation     (Path 1 — allowed)
Step 2: Push to OpenShift internal registry (Path 2 — allowed via oc)
Step 3: Pods pull from internal registry   (internal cluster traffic — always allowed)
```

Pods can always reach the OpenShift internal registry because it lives inside the same cluster network.

---

## Docker Concepts

### How a Dockerfile Builds an Image

Each instruction in a Dockerfile creates one read-only layer. Layers are cached — if nothing changed in a layer, Docker reuses the cached version.

```dockerfile
FROM ubi9:latest                    # Layer 1: base OS
COPY gitlab-runner.rpm /tmp/        # Layer 2: RPM file added
RUN rpm -ivh /tmp/gitlab-runner.rpm # Layer 3: RPM installed
RUN mkdir -p /etc/gitlab-runner     # Layer 4: directory created
USER 1001                           # Layer 5: user set
CMD ["gitlab-runner", "run"]        # Layer 6: startup command set
```

### RUN vs CMD

```
RUN  → executes during docker build — baked into the image
CMD  → executes when a container starts from the image
```

`RUN` is construction. `CMD` is what the building does when occupied.

### Why USER 1001 is Critical for OpenShift

By default Docker images run as root (UID 0). OpenShift's default Security Context Constraint (`restricted`) blocks root containers. Without `USER 1001`, your pod starts, hits the SCC restriction, and enters CrashLoopBackOff with empty logs. Setting a non-root UID in the Dockerfile fixes this before it becomes a problem.

### Deployment vs Job — The Most Common Confusion

This caused significant debugging time. Understanding this distinction is essential.

```
Deployment
  └── Expects container to run FOREVER
  └── Any exit = failure = restart
  └── Use for: web servers, APIs, daemons, the GitLab runner itself

Job
  └── Expects container to run ONCE and exit
  └── Exit code 0 = success = done
  └── Use for: scripts, migrations, one-time tasks

A UBI9 base image with no CMD will:
  ✓ Complete successfully as a Job (exits immediately with 0)
  ✗ CrashLoop forever as a Deployment (keeps restarting the exited container)
```

---

## Building the GitLab Runner Image

### Why We Build a Custom Image

GitLab provides official runner Docker images, but in an air-gapped environment these cannot be pulled from the internet. Instead, we have the GitLab runner RPM files stored in JFrog (or GitLab Package Registry for learning). We install them onto a UBI9 base image to create our own runner image.

### GitLab Runner 17 — Two Required RPMs

Starting with GitLab Runner 17, the helper images were split into a separate package. Both must be installed:

| RPM                               | Purpose                           | Typical Size |
| --------------------------------- | --------------------------------- | ------------ |
| `gitlab-runner_x86_64.rpm`        | The runner binary itself          | ~26MB        |
| `gitlab-runner-helper-images.rpm` | Pre-built helper container images | ~500MB       |

Both must be the SAME version. Mismatched versions cause dependency errors.

### The Dockerfile

```dockerfile
FROM ubi9:latest

# Copy RPMs from local build context into the image
COPY gitlab-runner_x86_64.rpm /tmp/gitlab-runner.rpm
COPY gitlab-runner-helper-images.rpm /tmp/gitlab-runner-helper-images.rpm

# Fix permissions after copy (prevents "Can not load RPM file" errors)
RUN chmod 644 /tmp/gitlab-runner.rpm \
              /tmp/gitlab-runner-helper-images.rpm

# Install both RPMs using dnf localinstall
# dnf localinstall is preferred over yum install for local RPM files
# It handles dependencies automatically and is the correct command for this use case
RUN dnf localinstall -y /tmp/gitlab-runner.rpm \
                        /tmp/gitlab-runner-helper-images.rpm && \
    dnf clean all && \
    rm -f /tmp/*.rpm
# dnf clean all removes yum cache to keep image size smaller
# rm -f removes the RPM files — no point keeping them after install

# Create required directories
# Runner crashes on startup if these don't exist
RUN mkdir -p /etc/gitlab-runner /home/gitlab-runner

# CRITICAL for OpenShift: run as non-root
# OpenShift's restricted SCC blocks root (UID 0) containers
# Without this line the pod will CrashLoopBackOff with empty logs
USER 1001

# CMD is what keeps the Deployment alive
# gitlab-runner run is a long-running process that polls GitLab for jobs
# It never exits on its own — perfect for a Deployment
CMD ["gitlab-runner", "run", \
     "--working-directory", "/home/gitlab-runner", \
     "--config", "/etc/gitlab-runner/config.toml"]
```

### Build Folder Structure

```
gitlab-runner-build/
    ├── Dockerfile                       ← pushed to GitLab repo
    ├── .gitignore (contains *.rpm)      ← prevents RPMs from going into git
    ├── gitlab-runner_x86_64.rpm         ← NOT in git, in JFrog/Package Registry
    └── gitlab-runner-helper-images.rpm  ← NOT in git, in JFrog/Package Registry
```

RPMs are binary files and should never be committed to git. They belong in a binary artifact repository (JFrog Artifactory or GitLab Package Registry).

---

## OpenShift BuildConfig Setup

### Why BuildConfig Instead of docker push

In an air-gapped environment, `docker push` from your workstation to OpenShift's internal registry is blocked by the firewall. BuildConfig solves this by having OpenShift pull the source and build the image internally — no external push required.

```
docker push approach (BLOCKED)
Your machine → firewall → OpenShift registry ✗

BuildConfig approach (WORKS)
Your machine → oc start-build → OpenShift builds internally → image in registry ✓
```

### Two Ways to Trigger a BuildConfig

**Option A: --from-dir (local folder)**
Sends your local folder directly to OpenShift over the existing oc connection. No firewall issue because it uses the same channel as all oc commands. Best for: testing, air-gapped environments where RPMs can't be downloaded during build.

```bash
oc start-build gitlab-runner \
  --from-dir=~/gitlab-runner-build \
  --follow
```

**Option B: Git source (production approach)**
BuildConfig watches a GitLab repo and builds when changes are pushed. Dockerfile is in git. RPMs are downloaded from JFrog/Package Registry during the build using credentials from secrets.

### Setting Up Secrets

Two separate secrets are needed for different purposes:

```bash
# Secret 1 — for git source authentication
# Type must be kubernetes.io/basic-auth for OpenShift git cloning
# username and password are the required key names for this type
oc create secret generic gitlab-git-auth \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=your-gitlab-username \
  --from-literal=password=${GITLAB_TOKEN} \
  -n your-namespace

# Annotate so OpenShift knows which URLs this secret applies to
oc annotate secret gitlab-git-auth \
  "build.openshift.io/source-secret-match-uri-1=https://gitlab.com/*" \
  -n your-namespace

# Secret 2 — for build args (passed into Dockerfile ARGs)
# Type is Opaque (generic) — just key-value pairs
# Key names must exactly match what BuildConfig references
oc create secret generic gitlab-build-args \
  --from-literal=token=${GITLAB_TOKEN} \
  --from-literal=project_id=${PROJECT_ID} \
  -n your-namespace
```

### Creating the ImageStream

The ImageStream must exist before the build runs. It is where the finished image is stored in the internal registry.

```bash
oc create imagestream gitlab-runner -n your-namespace
```

### The BuildConfig YAML

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
      uri: "https://gitlab.com/your-username/your-repo.git"
      ref: main
    sourceSecret:
      name: gitlab-git-auth # uses the basic-auth secret for cloning
  strategy:
    type: Docker
    dockerStrategy:
      buildArgs:
        - name: PKG_TOKEN # maps to ARG PKG_TOKEN in Dockerfile
          valueFrom:
            secretKeyRef:
              name: gitlab-build-args
              key: token # must match key name in secret exactly
        - name: PROJECT_ID # maps to ARG PROJECT_ID in Dockerfile
          value: "82861465" # hardcoded because it is not sensitive
  output:
    to:
      kind: ImageStreamTag
      name: "gitlab-runner:latest" # destination ImageStream
```

Apply and trigger:

```bash
oc apply -f buildconfig.yaml
oc start-build gitlab-runner --follow
```

---

## Deploying the Runner

### ServiceAccount and Permissions

The runner pod needs permission to create other pods (for pipeline jobs):

```bash
# Create dedicated service account
oc create serviceaccount gitlab-runner -n your-namespace

# Allow it to create and manage pods
oc adm policy add-role-to-user edit \
  -z gitlab-runner -n your-namespace

# Allow non-root execution
# Note: requires cluster-admin — may be restricted in Developer Sandbox
oc adm policy add-scc-to-user anyuid \
  -z gitlab-runner -n your-namespace
```

### config.toml

```toml
concurrent = 4          # max simultaneous pipeline jobs
check_interval = 0      # how often runner polls GitLab (0 = default)

[[runners]]
  name = "openshift-runner"
  url = "http://your-internal-gitlab-url"
  token = "your-runner-registration-token"

  # clone_url overrides the URL used to clone repos during jobs
  # Critical in air-gapped environments where the advertised
  # GitLab URL may not be reachable from inside cluster pods
  clone_url = "http://your-internal-gitlab-url"

  executor = "kubernetes"   # creates a new pod per pipeline job

  [runners.kubernetes]
    namespace = "your-namespace"
    service_account = "gitlab-runner"

    # Default image for jobs that don't specify their own
    # Points to internal registry — never needs internet
    image = "image-registry.openshift-image-registry.svc:5000/your-namespace/ubi9:latest"

    # Helper image handles: git clone, artifact upload, cache
    # Must be internal in air-gapped environment
    helper_image = "image-registry.openshift-image-registry.svc:5000/your-namespace/gitlab-runner:latest"
```

### ConfigMap

```bash
# Create ConfigMap from the config.toml file
# ConfigMap mounts the file into the pod at /etc/gitlab-runner/config.toml
oc create configmap gitlab-runner-config \
  --from-file=config.toml \
  -n your-namespace
```

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab-runner
  namespace: your-namespace
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
          # Points to internal registry — no internet needed
          image: image-registry.openshift-image-registry.svc:5000/your-namespace/gitlab-runner:latest
          volumeMounts:
            - name: config
              mountPath: /etc/gitlab-runner # where runner reads config.toml
      volumes:
        - name: config
          configMap:
            name: gitlab-runner-config # the ConfigMap we created
```

```bash
oc apply -f runner-deployment.yaml

# Verify
oc get pods -n your-namespace
oc logs deployment/gitlab-runner -n your-namespace
```

Healthy logs look like:

```
Configuration loaded    builds=0
Listening for jobs
```

---

## Pivots and Why We Made Them

### Pivot 1: Job Instead of Deployment (Initial Discovery)

**What happened:** The first Deployment kept CrashLoopBackOff-ing. A Job worked fine.

**Why:** The image being tested was a UBI9 base image with no long-running CMD. It started, did nothing, and exited with code 0. A Job celebrates that as success. A Deployment panics and restarts it in a loop.

**Lesson:** Always ask "does this container run forever or does it finish?" before choosing Deployment vs Job.

### Pivot 2: BuildConfig Instead of docker push

**What happened:** Tried to `docker push` a locally built image into OpenShift's registry. Got connection refused / timeout.

**Why:** OpenShift's registry route is exposed on a URL that goes through the corporate firewall. The firewall blocks this traffic from the workstation subnet.

**Fix:** BuildConfig builds the image inside OpenShift — no external push needed. The build output goes directly into the internal registry.

### Pivot 3: --from-dir Instead of Git Source

**What happened:** BuildConfig with Git source kept failing with `FetchSourceFailed` and authentication errors (403).

**Why — multiple causes:**

1. Private repo with group access tokens disabled
2. Personal token format causing URL parse errors (special characters interpreted as port numbers)
3. Token scope mismatches between git cloning and Package Registry uploads
   **Fix:** `oc start-build --from-dir` bypasses git entirely. It sends the local folder over the existing oc connection which is already authenticated. No git credentials needed.

**Important note:** `--from-dir` respects `.gitignore`. If `*.rpm` is in `.gitignore`, the RPMs will be excluded and the build will fail with "Can not load RPM file". Remove `.gitignore` before running `--from-dir` builds.

### Pivot 4: Making the Repo Public (Temporary)

**What happened:** After multiple 403 errors on git clone, we temporarily made the repo public to isolate whether the issue was auth or something else.

**Why it was needed:** The 403 errors were masking the real issue (URL format with token containing special characters). By removing auth from the equation, we confirmed the repo URL itself was correct and the issue was purely authentication.

**What to do in production:** Never make a repo containing infrastructure code public. Use SSH keys or a properly scoped deploy token instead of personal access tokens for BuildConfig git sources.

### Pivot 5: PKG_TOKEN Instead of GITLAB_TOKEN

**What happened:** Pushed Dockerfile with `ARG GITLAB_TOKEN` to GitLab. Got `pre-receive hook declined`.

**Why:** GitLab has secret push protection that scans for patterns matching known secret formats. `GITLAB_TOKEN` as a variable name triggered the secret scanner even though no actual token value was hardcoded.

**Fix:** Renamed the ARG to `PKG_TOKEN` which doesn't match GitLab's secret detection patterns.

### Pivot 6: Hardcoding PROJECT_ID in BuildConfig

**What happened:** PROJECT_ID was always empty inside the build despite the secret having the correct value.

**Why:** The `valueFrom.secretKeyRef` mechanism for build args has reliability issues in some OpenShift versions and sandbox environments. The secret key name `project-id` (with hyphen) may also cause shell variable issues since hyphens are not valid in shell variable names.

**Fix:** PROJECT_ID is not sensitive information (it is just a number visible in the GitLab URL). Hardcoding it directly as a `value` in the BuildConfig is simpler and more reliable than referencing a secret.

### Pivot 7: Separate Secrets for Git Auth vs Build Args

**What happened:** Used one generic secret for both git cloning and build args. Git cloning kept failing.

**Why:** OpenShift requires `type: kubernetes.io/basic-auth` with specific keys (`username` and `password`) for git source secrets. A generic Opaque secret with different key names is ignored for git auth even if the credentials are correct.

**Fix:** Two separate secrets:

- `gitlab-git-auth` — type `kubernetes.io/basic-auth` for git cloning
- `gitlab-build-args` — type `Opaque` for Dockerfile ARG injection

---

## Enterprise JFrog Example

This section describes the production setup for an air-gapped environment where JFrog Artifactory is the artifact repository instead of GitLab Package Registry.

### Architecture

```
JFrog Artifactory (internal)
  └── generic-repo/
        ├── gitlab-runner_x86_64.rpm
        └── gitlab-runner-helper-images.rpm

GitLab (internal)
  └── openshift-gitlab-runner/
        └── Dockerfile

OpenShift (ROSA on AWS — air-gapped)
  └── BuildConfig watches GitLab
  └── During build, pulls RPMs from JFrog
  └── Builds image internally
  └── Stores in internal registry
```

### JFrog Authentication

In air-gapped enterprise environments, use an **API key** not an identity token.

|                  | API Key     | Identity Token                    |
| ---------------- | ----------- | --------------------------------- |
| Lifespan         | Long-lived  | Short-lived (hours)               |
| Air-gap safe     | Yes         | No (OIDC callback may be blocked) |
| For pull secrets | Best choice | Will expire and break pulls       |

Generate API key:

```
JFrog UI → top right avatar → Edit Profile
→ Authentication Settings → Generate API Key
```

### JFrog Pull Secret for Image Pulls

If your base image (UBI9) is stored in JFrog:

```bash
oc create secret docker-registry jfrog-pull-secret \
  --docker-server=your-instance.jfrog.io \
  --docker-username=your-username \
  --docker-password=${JFROG_TOKEN} \
  -n your-namespace

# Link to service account so all pods can pull
oc secrets link default jfrog-pull-secret --for=pull -n your-namespace
```

### JFrog Credentials Secret for Build

```bash
oc create secret generic jfrog-build-credentials \
  --from-literal=username=your-jfrog-username \
  --from-literal=token=${JFROG_TOKEN} \
  --from-literal=url=your-instance.jfrog.io \
  -n your-namespace
```

### The Dockerfile (JFrog Version)

```dockerfile
FROM your-instance.jfrog.io/docker-local/ubi9:latest

# Build args injected from OpenShift secrets at build time
# ARG variables exist ONLY during build — not at runtime
ARG JFROG_USER
ARG JFROG_TOKEN
ARG JFROG_URL

# Download RPMs from JFrog using API key authentication
# -f flag: fail immediately on HTTP error instead of saving error HTML as RPM
# Without -f, a 403 response gets saved as the RPM file and yum install fails silently
RUN curl -f -u "${JFROG_USER}:${JFROG_TOKEN}" \
    "https://${JFROG_URL}/artifactory/rpm-repo/gitlab-runner_x86_64.rpm" \
    -o /tmp/gitlab-runner.rpm && \
    curl -f -u "${JFROG_USER}:${JFROG_TOKEN}" \
    "https://${JFROG_URL}/artifactory/rpm-repo/gitlab-runner-helper-images.rpm" \
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

### The BuildConfig (JFrog Version)

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
      uri: "https://your-internal-gitlab/your-group/openshift-gitlab-runner.git"
      ref: main
    sourceSecret:
      name: gitlab-git-auth # basic-auth secret for internal GitLab
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

### config.toml (Air-Gapped JFrog Version)

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

    # Both images point to internal registry
    # Nodes never need to reach JFrog or internet at runtime
    image = "image-registry.openshift-image-registry.svc:5000/your-namespace/ubi9:latest"
    helper_image = "image-registry.openshift-image-registry.svc:5000/your-namespace/gitlab-runner:latest"

    # JFrog pull secret allows job pods to pull custom images from JFrog
    [[runners.kubernetes.image_pull_secrets]]
      name = "jfrog-pull-secret"
```

### Full Enterprise Flow

```
1. Developer pushes Dockerfile change to internal GitLab
         │
         │  webhook triggers BuildConfig
         ▼
2. OpenShift BuildConfig clones Dockerfile from GitLab
         │
         │  using gitlab-git-auth secret (basic-auth)
         ▼
3. Build pod starts
         │
         │  downloads RPMs from JFrog using JFROG_TOKEN build arg
         │  installs them onto UBI9
         │  builds the image
         ▼
4. Image pushed to OpenShift internal registry
         │
         │  ImageStream updated to point to new image
         ▼
5. Deployment detects new image, rolling update
         │
         ▼
6. New runner pod starts, connects to internal GitLab ✓
         │
         ▼
7. Teams run pipelines using their own images from JFrog ✓
```

---

## Troubleshooting Reference

### CrashLoopBackOff

```bash
# Get exit code
oc describe pod <pod-name> | grep -A5 "Last State"

# Exit code meanings:
# 0   = success (wrong workload type — use Job not Deployment)
# 1   = app crashed (check logs)
# 126 = permission denied on entrypoint
# 127 = binary not found
# 137 = OOMKilled (increase memory limit)
```

### Empty Logs on CrashLoopBackOff

Container is dying before the app starts. Almost always a root user blocked by SCC.

```bash
# Fix: ensure USER 1001 is in Dockerfile
# Or temporarily grant anyuid (requires cluster-admin)
oc adm policy add-scc-to-user anyuid -z default -n your-namespace
```

### ImagePullBackOff / ErrImagePull

```bash
oc describe pod <pod-name> | grep -A10 Events

# Common causes:
# 1. No pull secret configured
# 2. Wrong registry URL
# 3. Image doesn't exist at that tag
# 4. Registry not reachable from nodes
```

### BuildConfig FetchSourceFailed

```bash
oc logs build/<build-name>

# Common causes:
# 1. Wrong git URL
# 2. Missing or wrong sourceSecret
# 3. Secret type must be kubernetes.io/basic-auth
# 4. Token lacks read_repository scope
# 5. Repo visibility is private without proper credentials
```

### Can Not Load RPM File

```bash
# Verify the RPM is a valid file
head -c 4 your-file.rpm | xxd
# Valid RPM: ed ab ee db

# Fix permissions
chmod 644 /tmp/*.rpm

# Use dnf localinstall not yum install
dnf localinstall -y /tmp/gitlab-runner.rpm

# Check .gitignore is not excluding RPMs from --from-dir builds
cat .gitignore
# If *.rpm is listed, remove .gitignore before running --from-dir
```

### Build Args Empty Inside Build

```bash
# Verify secret has values
oc get secret your-secret -o jsonpath='{.data.your-key}' | base64 -d

# Verify BuildConfig references correct secret and key names
oc get buildconfig your-bc -o jsonpath='{.spec.strategy.dockerStrategy.buildArgs}'

# Key names must match exactly — project-id in secret must match project-id in BuildConfig
# Hyphens in key names can cause issues — prefer underscores

# For non-sensitive values like PROJECT_ID, hardcode directly instead of secret reference
```

### 403 on Package Registry Upload

```bash
# Token must have write_package_registry scope
# OR api scope (which covers everything including package registry)
# Create new token at: GitLab → Preferences → Access Tokens
# Check: api scope
```

### Token in URL Causing Port Error

```
URL rejected: Port number was not a decimal number between 0 and 65535
```

Token contains special characters (`:`, `@`) being parsed as URL components.

```bash
# Fix: use git credential helper instead of embedding token in URL
git config --global credential.helper store
git clone https://gitlab.com/username/repo.git
# Enter username and token when prompted
```

---

## Quick Reference Commands

```bash
# Build from local folder (bypasses all git/auth issues)
oc start-build gitlab-runner --from-dir=./your-folder --follow

# Check build status
oc get builds

# Get build logs
oc logs build/gitlab-runner-N

# Check running pods
oc get pods -n your-namespace

# Watch pods in real time
oc get pods -w -n your-namespace

# Get pod logs
oc logs deployment/gitlab-runner -n your-namespace

# Describe pod (shows events and errors)
oc describe pod <pod-name> -n your-namespace

# Check secrets
oc get secrets -n your-namespace
oc get secret <secret-name> -o jsonpath='{.data.<key>}' | base64 -d

# Check imagestream
oc get imagestream -n your-namespace

# Cancel a build
oc cancel-build gitlab-runner-N
```
