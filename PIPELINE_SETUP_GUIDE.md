# Movie Picture CI/CD Pipeline Automation Guide

This repository contains the complete automated Continuous Integration (CI) and Continuous Deployment (CD) pipelines for the **Movie Picture** application (React Frontend and Flask Backend) deploying to Amazon Elastic Kubernetes Service (EKS) and Amazon Elastic Container Registry (ECR) via GitHub Actions.

---

## 1. Architecture & Pipeline Overview

```
[ Developer / PR ] ──► [ GitHub Actions CI Workflow ]
                              │
                              ├──► Lint Job (ESLint / Flake8)
                              ├──► Test Job (Jest / Pytest)
                              └──► Docker Build (Only after Lint & Test pass)

[ Merge to Main ]  ──► [ GitHub Actions CD Workflow ]
                              │
                              ├──► Lint & Test Jobs
                              ├──► Docker Build & Tag with Git SHA
                              ├──► ECR Login & Image Push
                              └──► Kustomize Edit & Kubectl Deploy to EKS
```

### Workflows Included:
1. **Frontend CI** (`.github/workflows/frontend-ci.yaml`): Runs lint, unit tests, and Docker build on pull requests modifying `starter/frontend/**`.
2. **Backend CI** (`.github/workflows/backend-ci.yaml`): Runs flake8 linting, pytest suite, and Docker build on pull requests modifying `starter/backend/**`.
3. **Frontend CD** (`.github/workflows/frontend-cd.yaml`): Runs lint, test, builds Docker image with `REACT_APP_MOVIE_API_URL`, tags with Git SHA, pushes to Amazon ECR, and applies Kubernetes manifests to EKS on merge/push to `main`.
4. **Backend CD** (`.github/workflows/backend-cd.yaml`): Runs lint, test, builds Docker image, tags with Git SHA, pushes to Amazon ECR, and applies Kubernetes manifests to EKS on merge/push to `main`.

---

## 2. Infrastructure Setup (Terraform & AWS)

### Step 2.1: Provision AWS Resources
Navigate to `setup/terraform` and provision the VPC, EKS cluster, ECR repositories, and IAM roles:

```bash
cd setup/terraform
terraform init
terraform apply
```

Review the planned resources and type `yes` to confirm.

### Step 2.2: Note Terraform Outputs
Once applied, view the output values:

```bash
terraform output
```
Key outputs include:
- `frontend_ecr`: Frontend ECR repository URL
- `backend_ecr`: Backend ECR repository URL
- `cluster_name`: EKS Cluster Name (`cluster`)
- `github_action_user_arn`: IAM ARN for GitHub Actions deployer

### Step 2.3: Generate AWS Access Keys for GitHub Actions
1. Go to AWS IAM Console -> **Users** -> Select `github-action-user`.
2. Navigate to the **Security Credentials** tab -> Under **Access keys**, click **Create access key**.
3. Choose **Application running outside AWS**, click **Next**, and create the key.
4. Copy the `Access Key ID` and `Secret Access Key`.

### Step 2.4: Configure GitHub Secrets
In your GitHub repository:
1. Navigate to **Settings** -> **Secrets and variables** -> **Actions** -> Click **New repository secret**.
2. Add the following secrets:
   - `AWS_ACCESS_KEY_ID`: Your copied Access Key ID
   - `AWS_SECRET_ACCESS_KEY`: Your copied Secret Access Key
   - `AWS_DEFAULT_REGION`: `us-east-1`

### Step 2.5: Configure Kubernetes Access for GitHub Actions User
Run the `init.sh` script to grant `github-action-user` administrative access to the EKS cluster in the `aws-auth` ConfigMap:

```bash
aws eks update-kubeconfig --name cluster --region us-east-1
cd setup
chmod +x init.sh
./init.sh
```

---

## 3. Running & Verifying CI/CD Pipelines

### 3.1 Triggering Continuous Integration (CI)
- Open a Pull Request targeting the `main` branch with changes in `starter/frontend/` or `starter/backend/`.
- Verify in the GitHub **Actions** tab that:
  - Lint and Test jobs execute in parallel.
  - The Build job runs only after both Lint and Test pass.

### 3.2 Triggering Continuous Deployment (CD)
- Merge your Pull Request into `main` (or trigger the workflow manually using **Run workflow** via `workflow_dispatch`).
- The CD pipeline will:
  1. Execute linting and testing.
  2. Authenticate to Amazon ECR using GitHub Secrets.
  3. Build and tag the Docker container with the exact `${{ github.sha }}` commit hash.
  4. Push the image to Amazon ECR.
  5. Update Kubernetes manifests dynamically using `kustomize`.
  6. Deploy the updated container to Amazon EKS.

---

## 4. Testing Pipeline Failure Handling

To verify that the pipelines reliably block faulty builds:

### Test Failure Simulation
- **Frontend**: Set `FAIL_TEST=true` in `starter/frontend/src/components/__tests__/App.test.js` or run:
  ```bash
  FAIL_TEST=true npm test
  ```
- **Backend**: Set `FAIL_TEST=true` in `starter/backend/test_app.py` or run:
  ```bash
  FAIL_TEST=true pipenv run test
  ```
- Pushing these changes to a PR will cause the `test` job to fail and prevent the `build` or `deploy` job from running.

### Lint Failure Simulation
- **Frontend**: Run `FAIL_LINT=true npm run lint` in `starter/frontend`.
- **Backend**: Run `pipenv run lint-fail` in `starter/backend`.

---

## 5. Verifying Deployment on Kubernetes

Check the running pods and services:

```bash
kubectl get pods
kubectl get svc
```

### Accessing the Web Application:
- Fetch the Frontend External LoadBalancer URL:
  ```bash
  kubectl get service frontend -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
  ```
- Open `http://<FRONTEND_LOAD_BALANCER_URL>` in your browser. The Movie Picture UI should load and display the movie catalog.

---

## 6. Teardown & Resource Cleanup

To avoid unnecessary AWS charges, destroy all provisioned infrastructure once evaluation is complete:

```bash
cd setup/terraform
terraform destroy
```
Type `yes` when prompted to tear down the EKS cluster, node groups, VPC, and ECR repositories.

