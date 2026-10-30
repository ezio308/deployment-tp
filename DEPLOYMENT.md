# Deployment and Operations Guide

This guide describes how to take a fresh checkout of this project and run it on
Docker Desktop Kubernetes, how the GitHub Actions deployment works, and how to
troubleshoot common setup problems.

The instructions use Windows PowerShell because the current workflow targets a
self-hosted Windows runner.

## 1. Architecture

The deployed application has four parts:

1. **Angular frontend** — built into static files and served by Nginx.
2. **Spring Boot backend** — exposes the user API on port `8081`.
3. **MySQL** — runs as a Kubernetes pod and listens internally on port `3306`.
4. **Ingress controller** — Traefik accepts HTTP traffic and routes it to the
   frontend Service.

The request path is:

```text
Browser -> Traefik Ingress -> frontend Service -> Angular/Nginx
                                             Nginx /api proxy -> backend Service -> MySQL Service
```

The frontend uses relative API URLs under `/api/user`. Nginx in the frontend
container proxies `/api/` to the backend Service, so the browser and API share
the same origin.

Angular's configured route is `/`, which renders the user directory. The backend
exposes these endpoints; the frontend reaches them with the `/api` prefix:

| Browser request | Backend endpoint | Purpose |
|---|---|---|
| `GET /api/user/all` | `GET /user/all` | List users |
| `POST /api/user/add` | `POST /user/add` | Add a user |
| `DELETE /api/user/delete/{id}` | `DELETE /user/delete/{id}` | Delete a user |

The user page has a responsive table, add-user form, and delete action. The
heading labels the ID column with the project owner's name.

The Angular production build is written to `angular-app/dist/angular-app`.
The frontend Dockerfile copies that directory into Nginx's document root.
The backend Dockerfile builds the Spring Boot JAR with Maven and runs it on
Java 17.

### MySQL and external installations

The Kubernetes manifests run their own MySQL container and persist its files
with a PersistentVolumeClaim. A separately installed MySQL server on the
developer's computer is **not required** for this Kubernetes deployment. An
existing host MySQL database is not automatically used or migrated.

## 2. Repository layout

| Path | Purpose |
|---|---|
| `angular-app/` | Angular source, production build, Nginx config, and frontend Dockerfile |
| `springboot/` | Spring Boot source, Maven build, and backend Dockerfile |
| `k8s/storage.yaml` | MySQL PersistentVolumeClaim |
| `k8s/database.yaml` | MySQL Deployment and internal Service |
| `k8s/backend.yaml` | Backend ConfigMap, Deployment, and internal Service |
| `k8s/frontend.yaml` | Frontend Deployment and internal Service |
| `k8s/ingress.yaml` | Standard Kubernetes Ingress routing HTTP `/` to the frontend |
| `.github/workflows/deploy.yml` | Build-and-deploy workflow triggered by pushes |
| `.gitignore` | Excludes generated outputs, local environment files, and editor files |

The database Secret is deliberately not stored in the repository. It is created
in the cluster before the database starts, or refreshed by the GitHub Actions
workflow from repository Actions secrets.

## 3. Prerequisites

For a local Kubernetes deployment:

- Windows 10/11 with PowerShell.
- Docker Desktop with Kubernetes enabled and running.
- `kubectl` available in `PATH`.
- Helm 3, for installing the Traefik Ingress controller.
- Internet access the first time container images are pulled.

For automatic deployment on pushes:

- A GitHub repository containing this project.
- A self-hosted Windows Actions runner on the Docker Desktop computer.
- Repository Actions secrets named `MYSQL_ROOT_PASSWORD` and
  `MYSQL_APP_PASSWORD`.
- The runner's Windows account must be able to use Docker Desktop and the
  intended `kubectl` context.

Check Docker Desktop and Kubernetes before proceeding:

```powershell
docker info
kubectl config current-context
kubectl get nodes
kubectl get storageclass
```

The manifests request the `hostpath` StorageClass available in this Docker
Desktop cluster. If another cluster does not have that class, either
install/configure a suitable dynamic storage provisioner or update
`storageClassName` in `k8s/storage.yaml` to a class available in that cluster.

## 4. Get the project

Clone the repository and enter its root directory:

```powershell
git clone https://github.com/ezio308/deployment-tp.git
Set-Location deployment-tp
```

For an existing local checkout, pull the latest changes before deploying:

```powershell
git pull
```

## 5. Install the Ingress controller

`k8s/ingress.yaml` defines an Ingress rule; Kubernetes does not implement that
rule by itself. A controller must be installed separately. This project uses
Traefik with the IngressClass name `traefik`.

Install Helm if needed, then install Traefik using its official Helm chart:

```powershell
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm upgrade --install traefik traefik/traefik `
  --namespace traefik `
  --create-namespace `
  --set ingressClass.enabled=true `
  --set ingressClass.name=traefik `
  --set ingressClass.isDefaultClass=false `
  --wait
```

Verify the controller and class:

```powershell
kubectl get pods,services -n traefik
kubectl get ingressclass
```

The controller is cluster infrastructure and is intentionally not defined in
the application manifests. On an existing cluster, do not install a second
Ingress controller if one already serves the `traefik` IngressClass.

## 6. Configure database credentials

The MySQL container initializes a root password and an application database
user from the `db-credentials` Kubernetes Secret. The application database is
named `demo`; the application username used by the workflow is `demo_app`.

### Local/manual deployment

Create the Secret before applying the database manifest. The following
PowerShell prompts hide the entered values and do not write a Secret manifest
to disk:

```powershell
function Read-SecretText([string] $Prompt) {
  $secureValue = Read-Host $Prompt -AsSecureString
  $pointer = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureValue)
  try {
    [Runtime.InteropServices.Marshal]::PtrToStringBSTR($pointer)
  }
  finally {
    [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($pointer)
  }
}

$rootPassword = Read-SecretText "MySQL root password"
$appPassword = Read-SecretText "MySQL demo_app password"

try {
  kubectl create secret generic db-credentials `
    --from-literal=username=demo_app `
    "--from-literal=password=$appPassword" `
    "--from-literal=root-password=$rootPassword" `
    --dry-run=client -o yaml | kubectl apply -f -
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to create the db-credentials Secret."
  }
}
finally {
  Remove-Variable rootPassword, appPassword -ErrorAction SilentlyContinue
}
```

Use unique, strong passwords for a new environment. Do not put password values
in YAML, source files, command history, screenshots, or chat messages.

### GitHub Actions deployment

In the repository, open **Settings → Secrets and variables → Actions** and add
these **repository secrets**:

| Secret | Purpose |
|---|---|
| `MYSQL_ROOT_PASSWORD` | MySQL root password used during initial database initialization |
| `MYSQL_APP_PASSWORD` | Password for the `demo_app` database user |

The workflow creates/updates the Kubernetes Secret using those values. Choose
the values privately; they are not meant to be written in this document or in
the repository. The app username is currently fixed as `demo_app` in the
workflow and database manifest.

**Persistent database warning:** MySQL only uses its initialization
environment variables when `/var/lib/mysql` is empty. If a PVC already
contains a database, changing the Kubernetes Secret or GitHub Actions secrets
does not rotate MySQL's existing passwords. The new secret values must match
the credentials already stored in that database, or the database user
passwords must be deliberately changed in MySQL as a separate operation. Keep
the MySQL image version compatible with the initialized data directory; a
database version upgrade requires a planned migration, not a routine apply.

## 7. Build and deploy manually

### Build the frontend and backend images

The Kubernetes Deployments use `imagePullPolicy: Never`, so the cluster must
be able to see the locally built images. In this Docker Desktop cluster,
Kubernetes uses the Docker Desktop image store.

Build both images from the repository root:

```powershell
docker build -f angular-app\dockerfile -t deployment-tp-frontend:local angular-app
if ($LASTEXITCODE -ne 0) { throw "Frontend image build failed." }

docker build -f springboot\dockerfile -t deployment-tp-backend:local springboot
if ($LASTEXITCODE -ne 0) { throw "Backend image build failed." }
```

The images are available to this Docker Desktop Kubernetes cluster after the
build; no node-container copy is needed. For a remote cluster or a different
Kubernetes distribution, publish images to a container registry and update
the Deployments to use images the cluster can pull.

### Apply all manifests

After creating `db-credentials` and building the images:

```powershell
kubectl apply -f k8s\
```

Applying the directory creates or updates the ConfigMap, PVC, MySQL resources,
backend, frontend, Services, and Ingress. The PVC binds to a dynamically
provisioned PersistentVolume.

Wait for the workloads:

```powershell
kubectl rollout status deployment/mysql --timeout=300s
kubectl rollout status deployment/backend --timeout=300s
kubectl rollout status deployment/frontend --timeout=300s
```

Check the result:

```powershell
kubectl get deployments,pods,services,pvc,ingress
```

Expected: MySQL, backend, and frontend each show one available replica;
`mysql-data` is `Bound`; and the Ingress uses class `traefik`.

## 8. Open the application through Ingress

On Docker Desktop, NodePort routing may not be reachable from Windows. A
port-forward through Traefik is a reliable local access method.

Run this in a terminal and keep it open:

```powershell
kubectl -n traefik port-forward service/traefik 18080:80
```

Open [http://localhost:18080](http://localhost:18080) in a browser. The Ingress
routes `/` to the frontend. The frontend's Nginx then proxies `/api/` to the
backend Service. The browser does not need to call the backend Service
directly.

## 9. GitHub Actions pipeline

The workflow in `.github/workflows/deploy.yml` runs for every push. It:

1. Checks out the pushed commit.
2. Verifies that the required repository secrets, Docker, and Kubernetes are
   available.
3. Builds the Angular and Spring Boot Docker images tagged with the commit SHA.
4. Creates/updates `db-credentials` from the repository Actions secrets.
5. Applies all YAML files under `k8s\`.
6. Updates the frontend and backend Deployments to the images built for that
   commit.
7. Waits for MySQL, backend, and frontend rollouts to finish.

The workflow does not install Docker Desktop, enable Kubernetes, or install
Traefik. Those are cluster prerequisites. It also does not use a container
registry; it uses the Docker Desktop image store shared with this local
cluster.

### Configure the self-hosted Windows runner

The runner must run on the same PC as Docker Desktop because the workflow
accesses that machine's Docker engine and Kubernetes cluster.

1. In GitHub, open **Repository → Settings → Actions → Runners → New
   self-hosted runner**.
2. Select **Windows** and **x64**.
3. Follow GitHub's current download, extraction, and configuration commands.
   Registration tokens expire; obtain a fresh one from the runner settings
   page. Never save a token in the repository or share it in chat.
4. Configure the runner with labels that match `runs-on: [self-hosted,
   Windows]`.
5. Start `run.cmd` interactively while Docker Desktop and Kubernetes are
   running. Keep the runner process alive. Running as a service is possible,
   but the service account must have the correct Docker Desktop and
   `kubectl` access.
6. Confirm the runner appears **Online** on the repository's Runners page.

The workflow uses Docker Desktop node container names to import images, so
verify that commands such as `docker info`, `kubectl get nodes`, and
`docker exec <node-name> ctr --namespace k8s.io images ls` work under the
runner's Windows account.

### Trigger and verify a deployment

Once the runner is online and the repository secrets exist, push a commit:

```powershell
git add .
git commit -m "Describe the change"
git push
```

Open the repository's **Actions** tab, select the **Build and deploy**
workflow, and inspect the run for that commit. The run must finish successfully;
the GitHub push alone does not prove that the Kubernetes rollout completed.

For a code change, make sure the changed source files are included in the
commit. Uncommitted local edits are not part of a GitHub Actions checkout and
cannot be deployed by a push.

## 10. Configuration and Kubernetes resources

### ConfigMap

`k8s/backend.yaml` defines `backend-config`. It stores the non-secret JDBC URL:

```text
jdbc:mysql://mysql-service:3306/demo
```

The backend gets this value as `SPRING_DATASOURCE_URL`. It also gets its
username and password from the Kubernetes Secret.

### Secret

`db-credentials` has three keys:

- `username` — `demo_app`
- `password` — application user's MySQL password
- `root-password` — MySQL root password used during initialization

Kubernetes Secrets are not part of the checked-in YAML. Base64 encoding in a
Kubernetes Secret is not encryption. Restrict cluster and repository access,
enable encryption at rest where appropriate, and never print Secret values in
logs.

Inspect only Secret metadata and key names without exposing the values:

```powershell
kubectl describe secret db-credentials
```

### Persistent storage

`k8s/storage.yaml` requests `1Gi` from the `hostpath` StorageClass using
`ReadWriteOnce`. Kubernetes dynamically provisions a PersistentVolume.
`k8s/database.yaml` mounts the claim at `/var/lib/mysql`.

The current Docker Desktop `standard` class uses local-path storage with a
`Delete` reclaim policy. This is suitable for local development, not as a
backup or highly available production database. The volume is tied to the
cluster/node storage. Back up important data separately. Deleting the PVC can
delete its backing volume and its data.

### Service and Deployment

- A **Deployment** maintains the desired pods and replaces failed pods.
- A **Service** provides a stable in-cluster address and selects pods by label.
- MySQL and backend use `ClusterIP` Services and are internal to the cluster.
- The frontend Service is also internal; Ingress is the HTTP entry point.

## 11. Useful operating commands

```powershell
# Overall workload status
kubectl get deployments,pods,services,pvc,ingress

# Recent events
kubectl get events --sort-by=.lastTimestamp

# Application logs
kubectl logs deployment/frontend
kubectl logs deployment/backend
kubectl logs deployment/mysql

# Rollout history and rollback options
kubectl rollout history deployment/frontend
kubectl rollout history deployment/backend

# Reapply all project resources
kubectl apply -f k8s\
```

To reach an API endpoint locally without Ingress, a temporary backend
port-forward can be used:

```powershell
kubectl port-forward service/backend 18081:8081
```

Then request `http://localhost:18081/user/all`.

## 12. Troubleshooting

### Workflow waits for a runner

- Check the repository's **Settings → Actions → Runners** page.
- The self-hosted runner must be online and have the `self-hosted` and
  `Windows` labels.
- Keep `run.cmd` running and Docker Desktop/Kubernetes available.

### Workflow fails its prerequisite step

- Verify both repository secret names exactly:
  `MYSQL_ROOT_PASSWORD` and `MYSQL_APP_PASSWORD`.
- Confirm Docker Desktop is running and `kubectl config current-context`
  selects the intended cluster under the runner's account.

### `ErrImageNeverPull`

The Kubernetes node cannot see the locally built image. Rerun the workflow's
image import procedure for every node, or use the Actions workflow, which
imports images automatically.

### MySQL or backend is not Ready

```powershell
kubectl describe pod -l app=mysql
kubectl logs deployment/mysql
kubectl describe pod -l app=backend
kubectl logs deployment/backend
```

Check that:

- `mysql-data` is Bound and the `standard` StorageClass exists.
- Secret key names match the manifests.
- Secret passwords match the users in the initialized MySQL database.
- The backend ConfigMap points to `mysql-service`.

Changing the Secret does not change passwords inside an already initialized
MySQL data directory. Avoid deleting the PVC as a troubleshooting shortcut:
that may permanently delete database data.

### Ingress returns 404 or has no address

- Confirm an Ingress controller is installed and running.
- Confirm `kubectl get ingressclass` includes `traefik`.
- Confirm `kubectl describe ingress frontend` shows the frontend Service
  endpoint.
- Use the Traefik port-forward in this guide for Docker Desktop.
- If another controller is used, update both its IngressClass and
  `ingressClassName` in `k8s/ingress.yaml`.

### Page still shows an older frontend

- Check the deployed commit in the successful Actions run.
- Check that the frontend source changes were committed and pushed.
- Check the Deployment image:

  ```powershell
  kubectl get deployment frontend -o jsonpath='{.spec.template.spec.containers[0].image}'
  ```

- Refresh without cache or open a private browser window after rollout.

## 13. Security and production considerations

- Protect the repository and its `main` branch. A self-hosted runner executes
  workflow code on the host machine with access to Docker and Kubernetes.
- Do not run untrusted pull-request code on a runner that can access deployment
  credentials or a cluster.
- Keep passwords in GitHub Actions secrets and Kubernetes Secrets; rotate
  credentials deliberately and update MySQL itself when a PVC is already
  initialized.
- This local image-transfer workflow is not suitable for a remote cluster.
  For remote deployments, publish immutable images to a private registry and
  configure secure cluster credentials for the workflow.
- Use TLS, access controls, resource limits, backups, monitoring, and an
  appropriately managed database before exposing a production deployment.
