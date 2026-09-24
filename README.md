# Movie Picture Pipeline

You've been brought on as the DevOps resource for a development team that manages a web application that is a catalog of Movie Picture movies. They're in dire need of automating their development workflows in hopes of accelerating their release cycle. They'd like to use GitHub Actions to automate testing, building, and deploying their applications to an existing Kubernetes cluster.

The team's project is comprised of 2 applications:
1. A frontend UI written in TypeScript, using the React framework.
2. A backend API written in Python using the Flask framework.

---

## Initial Setup: Workspace and Repository Initialization

Execute these one-time steps in your terminal to initialize and connect your repository:

### 1. Authenticate with GitHub CLI
```bash
gh auth login
```
* Select `GitHub.com` > `HTTPS` > Authenticate with credentials (`Y`).
* Choose **Login with a web browser**, copy the one-time code, and authorize access.

### 2. Configure Git Email
```bash
git config --global user.email "YOUR_EMAIL"
```

### 3. Initialize and Push Project
```bash
git init
git add .
git commit -m "initial"
gh repo create udacity-build-cicd-project --source=. --public --push
```

---

## Step 1: Frontend Deliverable

Configure the CI and CD pipelines for the frontend application using GitHub Actions.

### 1. Continuous Integration Workflow
* **File:** `.github/workflows/frontend-ci.yaml`
* **Workflow Name:** `Frontend Continuous Integration`
* **Triggers:**
  * Automated on `pull_request` against the `main` branch (only when files in `frontend/` change).
  * Manual execution via `workflow_dispatch`.
* **Jobs:**
  * `lint`: Checkout code, Setup Node.js, restore cache, install dependencies (`npm ci`), run linter (`npm run lint`).
  * `test`: Checkout code, Setup Node.js, restore cache, install dependencies (`npm ci`), run tests (`npm run test`).
  * *Note:* `lint` and `test` must run in parallel.
  * `build`: Runs only after `lint` and `test` succeed (`needs: [lint, test]`). Builds the application container using Docker.

### 2. Continuous Deployment Workflow
* **File:** `.github/workflows/frontend-cd.yaml`
* **Workflow Name:** `Frontend Continuous Deployment`
* **Triggers:**
  * Automated on `push` (merges) to the `main` branch (only when files in `frontend/` change).
  * Manual execution via `workflow_dispatch`.
* **Jobs:**
  * Runs the same `lint` and `test` jobs as the CI workflow.
  * `build`: Builds the Docker image using `--build-arg=REACT_APP_MOVIE_API_URL=http://localhost:5000` and tags it with the Git SHA.
  * `deploy`:
    * Logs into Amazon ECR using `aws-actions/amazon-ecr-login` and GitHub Secrets.
    * Pushes the tagged image to the ECR repository.
    * Updates manifests using `kustomize edit set image` with the Git SHA tag.
    * Applies the Kubernetes manifests to the cluster using `kubectl`.

---

## Step 2: Backend Deliverable

Configure the CI and CD pipelines for the backend application using GitHub Actions.

### 1. Continuous Integration Workflow
* **File:** `.github/workflows/backend-ci.yaml`
* **Workflow Name:** `Backend Continuous Integration`
* **Triggers:**
  * Automated on `pull_request` against the `main` branch (only when files in `backend/` change).
  * Manual execution via `workflow_dispatch`.
* **Jobs:**
  * `lint`: Checkout code, setup environment, install dependencies (`pipenv install`), run linter (`pipenv run lint`).
  * `test`: Checkout code, setup environment, install dependencies (`pipenv install`), run tests (`pipenv run test`).
  * *Note:* `lint` and `test` must run in parallel.
  * `build`: Runs only after `lint` and `test` succeed (`needs: [lint, test]`). Builds the Docker container.

### 2. Continuous Deployment Workflow
* **File:** `.github/workflows/backend-cd.yaml`
* **Workflow Name:** `Backend Continuous Deployment`
* **Triggers:**
  * Automated on `push` (merges) to the `main` branch (only when files in `backend/` change).
  * Manual execution via `workflow_dispatch`.
* **Jobs:**
  * Runs the same `lint` and `test` jobs as the CI workflow.
  * `build`: Builds the Docker image and tags it with the Git SHA.
  * `deploy`:
    * Logs into Amazon ECR using `aws-actions/amazon-ecr-login` and GitHub Secrets.
    * Pushes the tagged image to the ECR repository.
    * Updates manifests using `kustomize edit set image` with the Git SHA tag.
    * Applies the Kubernetes manifests to the cluster using `kubectl`.

---

## Step 3: Setup CD Environment

Set up the underlying AWS and Kubernetes infrastructure before testing deployments.

### 1. Terraform Setup & Infrastructure Provisioning
```bash
git clone [https://github.com/tfutils/tfenv.git](https://github.com/tfutils/tfenv.git) ~/.tfenv
export PATH="$HOME/.tfenv/bin:$PATH"
source ~/.bashrc
tfenv install 1.3.9
tfenv use 1.3.9

cd /workspace/setup/terraform
terraform init
```

Set the AWS credentials of your created administrator user:
```bash
export AWS_ACCESS_KEY_ID={copied-access-key}
export AWS_SECRET_ACCESS_KEY={copied-secret-key}
```

Apply the Terraform template:
```bash
cd /workspace/setup/terraform
terraform apply
```
*Note outputs using `terraform output` for later configuration.*

### 2. Credentials for GitHub Actions
1. Navigate to the IAM service in the AWS Console.
2. Under Users, select `github-action-user`.
3. Open **Security Credentials** > **Access keys** > **Create access key**.
4. Choose **Application running outside AWS** and copy the access key pair into your GitHub repository secrets.

### 3. Add GitHub Actions User to Kubernetes
Authorize the `github-action-user` within the cluster's `aws-auth` ConfigMap:
```bash
aws eks update-kubeconfig --name cluster --region us-east-1
kubectl get configmap aws-auth -n kube-system
cd /workspace/setup
./init.sh
```

### 4. Infrastructure Cleanup
*To prevent depletion of AWS credits, destroy resources when work is paused:*
```bash
cd /workspace/setup/terraform
terraform destroy
```

---

## Step 4: Frontend Development

Development guidelines, local commands, and test-failure simulations for `frontend/`.

### 1. Dependency Installation & Local Execution
```bash
cd starter/frontend

# Install dependencies
npm ci

# Run development server
REACT_APP_MOVIE_API_URL=http://localhost:5000 npm start
```

### 2. Running Tests
```bash
# Interactive test runner
npm test

# CI simulation mode
CI=true npm test

# Simulate test failure
FAIL_TEST=true CI=true npm test
```

### 3. Running Linter
```bash
# Run lint check
npm run lint

# Simulate lint failure
FAIL_LINT=true npm run lint
```

### 4. Docker Build & Production Run
```bash
# Build Docker image with build argument
docker build --build-arg=REACT_APP_MOVIE_API_URL=http://localhost:5000 --tag=mp-frontend:latest .

# Run container locally
docker run --name mp-frontend -p 3000:3000 -d mp-frontend
```

### 5. Deploy Kubernetes Manifests
```bash
cd starter/frontend/k8s

# Update image tag
kustomize edit set image frontend=<ECR_REPO_URL>:<NEW_TAG_HERE>

# Apply manifests to cluster
kustomize build | kubectl apply -f -
```

---

## Step 5: Backend Development

Development guidelines, local commands, and test-failure simulations for `backend/`.

### 1. Dependency Installation & Local Execution
```bash
cd starter/backend

# Install dependencies
pipenv install

# Run application locally
pipenv run serve
```

### 2. Running Tests
```bash
# Run tests
pipenv run test

# Simulate test failure
FAIL_TEST=true pipenv run test
```

### 3. Running Linter
```bash
# Run lint check
pipenv run lint

# Simulate lint failure
pipenv run lint-fail
```

### 4. Docker Build & Production Run
```bash
# Build Docker image
docker build --tag mp-backend:latest .

# Run container locally
docker run -p 5000:5000 --name mp-backend -d mp-backend

# Verify API output
curl http://localhost:5000/movies
docker logs -f mp-backend

# Stop container
docker stop mp-backend
```

### 5. Deploy Kubernetes Manifests
```bash
cd starter/backend/k8s

# Update image tag
kustomize edit set image backend=<ECR_REPO_URL>:<NEW_TAG_HERE>

# Apply manifests to cluster
kustomize build | kubectl apply -f -
```