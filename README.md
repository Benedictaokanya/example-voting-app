# Voting App Capstone

A containerized voting application deployed to Kubernetes with Helm. This capstone documents the local Kind deployment, infrastructure-as-code, CI image build and scan, and Kubernetes deployment validation.

## Project status

- **Local Kubernetes deployment:** verified on a Kind cluster named `voting-app`.
- **Application flow:** tested by submitting a vote on the Vote page and confirming it appeared on the Result page.
- **Helm:** chart linting and template rendering passed; release `voting-app` reached `deployed`.
- **GitHub Actions:** the Capstone CI CD workflow and Kind Kubernetes Deployment Check both completed successfully for the latest recorded commit.
- **AWS EKS:** Terraform configuration is included, but an EKS cluster was not created. AWS Organizations policies denied the required `eks:CreateCluster` and `eks:CreateNodegroup` actions. The workaround was to validate and run the application locally on Kind rather than attempt to bypass the organization's policy.

## Architecture

![Voting App architecture](architecture.excalidraw.png)

The application consists of five workloads:

1. **Vote** — Python web interface where a user selects an option.
2. **Redis** — temporarily holds incoming votes.
3. **Worker** — consumes votes from Redis and stores them in PostgreSQL.
4. **PostgreSQL** — stores the vote data.
5. **Result** — Node.js web interface that displays the vote totals.

The Vote and Result services are exposed to the local machine through Kubernetes port-forwarding during local testing. Redis and PostgreSQL are internal ClusterIP services.

## Repository layout

- `helm/voting-app/` — Helm chart and Kubernetes templates for the five workloads and services.
- `terraform/` — Terraform configuration for an AWS EKS cluster and managed node group.
- `k8s-specifications/` — standalone Kubernetes YAML manifests.
- `.github/workflows/capstone-ci-cd.yml` — builds application images, scans them with Trivy, and publishes images to GitHub Container Registry on pushes to `main`.
- `.github/workflows/kind-deploy-test.yml` — creates a temporary Kind cluster in GitHub Actions, builds and loads the images, installs the Helm chart, checks workload rollouts, and smoke-tests both web apps.
- `architecture.excalidraw.png` — architecture diagram.

## Prerequisites

For local deployment, install Docker, kubectl, Kind, and Helm. The versions used during this validation were:

- Docker Engine 29.9.2
- kubectl 1.35.9
- Kind 0.30.0
- Helm 4.3.1

The local Kind node ran Kubernetes v1.34.0. Other compatible versions may also work.

## Deploy locally with Kind

From the repository root:

```bash
kind create cluster --name voting-app
kubectl cluster-info
kubectl get nodes
```

Install the chart:

```bash
helm lint ./helm/voting-app
helm template voting-app ./helm/voting-app
helm upgrade --install voting-app ./helm/voting-app --wait --timeout 5m
```

Check the release and workloads:

```bash
helm status voting-app
helm history voting-app
kubectl get deployments
kubectl get pods
kubectl get services
```

A healthy deployment should show the `db`, `redis`, `result`, `vote`, and `worker` deployments available and their pods in `Running` state with ready containers.

### Open the Vote and Result pages

Open two terminal windows and keep both port-forward commands running.

Terminal 1:

```bash
kubectl port-forward service/vote 18080:5000
```

Terminal 2:

```bash
kubectl port-forward service/result 18081:5001
```

Then open:

- Vote page: http://127.0.0.1:18080
- Result page: http://127.0.0.1:18081

Submit a vote on the Vote page and check the Result page. The port-forward processes must remain running while you use these pages. In this environment, using `127.0.0.1` worked when `localhost` did not.

### Useful diagnostics

```bash
kubectl get pods -o wide
kubectl describe pods
kubectl logs deployment/worker --tail=50
kubectl logs deployment/result --tail=50
curl -I http://127.0.0.1:18080
```

The Worker log should show successful database and Redis connections and vote processing. A body-parser deprecation warning appeared in the Result logs during testing, but the Result app still started and connected to the database.

## GitHub Actions

Two workflows support automated validation:

### Capstone CI CD

`.github/workflows/capstone-ci-cd.yml` runs on pushes to `main`, pull requests targeting `main`, and manual dispatch. It builds the Vote, Result, and Worker images and scans each image with Trivy for high and critical vulnerabilities. On a push to `main`, it publishes both `latest` and commit-SHA-tagged images to GitHub Container Registry using the workflow's `GITHUB_TOKEN`.

The current Trivy configuration reports high and critical findings but uses `exit-code: '0'`; findings do not by themselves fail the job. Review scan output rather than interpreting a successful job as proof that no vulnerabilities exist.

### Kind Kubernetes Deployment Check

`.github/workflows/kind-deploy-test.yml` creates an ephemeral Kind cluster on the GitHub runner, builds and loads the three application images, installs the Helm chart, waits for all five deployments to roll out, and checks that the Vote and Result web pages return successful HTTP responses.

Latest recorded successful runs:

- [Capstone CI CD](https://github.com/Benedictaokanya/example-voting-app/actions/runs/40722590349)
- [Kind Kubernetes Deployment Check](https://github.com/Benedictaokanya/example-voting-app/actions/runs/40722590327)

Earlier Kind checks failed while the service target ports were being corrected. The target ports for Vote and Result were aligned with the applications' non-root port 8080, then the latest recorded deployment check passed.

## Rollback

Helm keeps release revisions so a previous revision can be restored. First inspect the available revisions and identify the revision to restore:

```bash
helm history voting-app
helm rollback voting-app <REVISION> --wait --timeout 5m
helm status voting-app
kubectl get deployments,pods
```

Replace `<REVISION>` with an actual revision shown by `helm history`. Confirm the Vote and Result pages still work after rollback. A rollback demonstration should be performed and captured as evidence before claiming that rollback was demonstrated live.

## Terraform and AWS EKS limitation

The `terraform/` directory defines AWS EKS cluster and managed node group resources. The configuration was validated, but applying it was blocked because the AWS account's Organizations service control policy denied `eks:CreateCluster` and `eks:CreateNodegroup`. No EKS cluster was created by this project. The local Kind deployment was used to continue testing Kubernetes and Helm without bypassing the account restriction.

To validate the Terraform configuration locally, from the `terraform/` directory:

```bash
terraform init -backend=false
terraform validate
terraform plan
```

A successful validation or plan does not mean AWS resources were created. An authorized AWS environment is required to deploy the EKS infrastructure.

## Security notes

- GitHub Actions uses the automatically provided `GITHUB_TOKEN` for publishing images; do not paste tokens into source files.
- The Helm values currently include a simple PostgreSQL password intended only for this learning/demo setup. It is not suitable for production. Before treating the project as fully compliant with the “no secrets in Git” requirement or deploying to a shared environment, move database credentials to a Kubernetes Secret and avoid committing real credentials.
- The chart uses `emptyDir` for the database and Redis data, so those data directories are ephemeral and are not a production persistence strategy.
- Trivy findings should be reviewed; a successful scan job does not imply zero vulnerabilities.

## Evidence to capture

Add screenshots of the actual results to a `screenshots/` folder before final submission:

1. Vote page after submitting a vote.
2. Result page showing the updated vote count.
3. `kubectl get pods` showing the five workloads ready.
4. `helm status voting-app` showing the release as deployed.
5. Worker logs showing Redis/database connections and vote processing.
6. Successful GitHub Actions runs for both workflows.
7. Helm history and the result of a rollback followed by a health check, after you have performed that demonstration.

Do not include passwords, access tokens, or other sensitive values in screenshots.

## Cleanup

When you no longer need the local test cluster:

```bash
kind delete cluster --name voting-app
```

This removes the local Kind cluster and its workloads.
