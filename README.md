# Movie Picture Pipeline

You've been brought on as the DevOps resource for a development team that manages a web application that is a catalog of Movie Picture movies. They're in dire need of automating their development workflows in hopes of accelerating their release cycle. They'd like to use GitHub Actions to automate testing, building, and deploying their applications to an existing Kubernetes cluster.

The team's project is comprised of 2 applications:
1. A frontend UI written in TypeScript, using the React framework.
2. A backend API written in Python using the Flask framework.

---

## Initial Setup & Prerequisites

### 1. Local Development Dependencies
If you work outside the managed online workspace, ensure the following tools are installed:
* [docker](https://docs.docker.com/desktop/install/debian/) - Build frontend and backend containers
* [kubectl](https://kubernetes.io/docs/tasks/tools/) - Apply Kubernetes manifests to the cluster
* [pipenv](https://pipenv.pypa.io/en/latest/install/#pragmatic-installation-of-pipenv) - Manage Python dependencies and virtual environments
* [nvm](https://github.com/nvm-sh/nvm#installing-and-updating) - Manage Node.js versions
* [tfswitch](https://tfswitch.warrensbox.com/Install/) or [tfenv](https://github.com/tfutils/tfenv.git) - Manage Terraform versions
* [kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/) - Build Kubernetes manifests dynamically
* [jq](https://stedolan.github.io/jq/download/) - Parse JSON on the command line

### 2. Workspace and Repository Initialization
Execute these one-time steps in your terminal to initialize and connect your repository:

#### Authenticate with GitHub CLI
```bash
gh auth login
```
* Select `GitHub.com` > `HTTPS` > Authenticate with credentials (`Y`).
* Select **Login with a web browser**, copy the one-time code, and authorize access.

#### Configure Git Email
```bash
git config --global user.email "YOUR_EMAIL"
```

#### Initialize and Push Project
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
  * `lint`: Checkout code, setup Node.js, restore cache before dependency install, install dependencies (`npm ci`), run linter (`npm run lint`).
  * `test`: Checkout code, setup Node.js, restore cache before dependency install, install dependencies (`npm ci`), run tests (`npm run test`).
  * *Note:* `lint` and `test` must execute in parallel.
  * `build`: Runs only after `lint` and `test` succeed (`needs: [lint, test]`). Builds the application container using Docker.

### 2. Continuous Deployment Workflow
* **File:** `.github/workflows/frontend-cd.yaml`
* **Workflow Name:** `Frontend Continuous Deployment`
* **Triggers:**
  * Automated on `push` (merges) to the `main` branch (only when files in `frontend/` change).
  * Manual execution via `workflow_dispatch`.
* **Jobs:**
  * Runs the same `lint` and `test` jobs as the CI workflow.
  * `build`: Runs only after `lint` and `test` complete. Builds the Docker image and tags it with the Git SHA.
    * **Important Variable Abstraction:** Do not hard-code `http://localhost:5000`. Define an environment variable `REACT_APP_MOVIE_API_URL` within the workflow and pass it via `--build-arg=REACT_APP_MOVIE_API_URL=$REACT_APP_MOVIE_API_URL`.
  * `deploy`:
    * Logs into Amazon ECR using `aws-actions/amazon-ecr-login` and GitHub Secrets.
    * Pushes the tagged Docker image to the ECR repository.
    * Updates Kubernetes manifests with `kustomize edit set image` using the Git SHA.
    * Deploys the application using `kubectl`.

### 3. Submission & Failure Criteria (Grading Rubric)
* **Zero Credential Tolerance:** Storing unmasked AWS credentials directly in any workflow file = **AUTOMATIC FAIL**. Always use GitHub Secrets.
* **Pipeline Integrity:** Any pipeline failing, encountering failed steps, or passing when tests fail = **AUTOMATIC FAIL**.
* **ECR Verification:** If the Docker image is not pushed to ECR = **FAIL**.
* **Cluster Verification:** If the application is not running successfully on the cluster = **FAIL**.
* **Verification Proof Required:** Submit a working URL or screenshots demonstrating that the frontend is live, displays the movie catalog, and verified that the environment variable was passed correctly.

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
  * *Note:* `lint` and `test` must execute in parallel.
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
    * Updates manifests using `kustomize edit set image` with the Git SHA.
    * Applies Kubernetes manifests using `kubectl`.

### 3. Submission & Failure Criteria (Grading Rubric)
* **Zero Credential Tolerance:** AWS credentials present in any pipeline = **AUTOMATIC FAIL**.
* **Pipeline Integrity:** Any pipeline failure or pass on simulated test failure = **AUTOMATIC FAIL**.
* **ECR Verification:** Docker image not uploaded to ECR = **FAIL**.
* **Cluster Verification:** Backend API not reachable or not running on the cluster = **FAIL**.
* **Verification Proof Required:** Submit a working URL or screenshot confirming that the Backend API returns the movie list JSON payload (`/movies`).

---

## Step 3: Setup CD Environment

Prepare the underlying AWS and Kubernetes infrastructure before deploying applications.

### 1. Install Terraform 1.3.9
```bash
git clone [https://github.com/tfutils/tfenv.git](https://github.com/tfutils/tfenv.git) ~/.tfenv
export PATH="$HOME/.tfenv/bin:$PATH"
source ~/.bashrc
tfenv install 1.3.9
tfenv use 1.3.9

cd /workspace/setup/terraform
terraform init
```

### 2. Administrator IAM User & Federated Account ("voclabs") Troubleshooting
* **Issue:** Running `terraform apply` under the default Udacity federated user account (containing `voclabs`) fails due to insufficient IAM permissions.
* **Resolution:**
  1. Open the AWS IAM Console and create a new dedicated IAM user.
  2. Attach the `AdministratorAccess` policy to this user.
  3. Go to **Security credentials** > **Create access key**.
  4. Export these credentials in your workspace terminal before executing Terraform:

```bash
export AWS_ACCESS_KEY_ID={copied-access-key}
export AWS_SECRET_ACCESS_KEY={copied-secret-key}
```

### 3. Provision AWS Infrastructure with Terraform
```bash
cd /workspace/setup/terraform
terraform apply
```
*Review the plan, type `yes`, and save the output values for later use:*
```bash
terraform output
```

### 4. Generate AWS Credentials for GitHub Actions
1. In the AWS IAM Console under **Users**, select `github-action-user`.
2. Open **Security credentials** > **Access keys** > **Create access key**.
3. Select **Application running outside AWS** and click **Next** > **Create access key**.
4. Store these access keys as secrets in your GitHub repository (`Settings` > `Secrets and variables` > `Actions`).

### 5. Add GitHub Actions User to Kubernetes (`init.sh` Mechanics)
Authorize the `github-action-user` to manage the EKS cluster:
```bash
aws eks update-kubeconfig --name cluster --region us-east-1
kubectl get configmap aws-auth -n kube-system
cd /workspace/setup
./init.sh
```

**What `./init.sh` does under the hood:**
1. Fetches the ARN for the IAM user `github-action-user` using the AWS CLI and saves it in `$userarn`.
2. Downloads `aws-iam-authenticator` (v0.6.2) and makes it executable.
3. Maps the IAM user to the Kubernetes role/username `github-action-role` and binds it to the `system:masters` group in the `aws-auth` ConfigMap.
4. Cleans up by deleting the temporary `aws-iam-authenticator` binary.

### 6. Infrastructure Cleanup
*To avoid exhausting AWS lab credits, destroy all resources when pausing work:*
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

# Run development server with backend URL
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
# Define environment variable (Do not hard-code into docker build)
export REACT_APP_MOVIE_API_URL=http://localhost:5000

# Build Docker image using the environment variable
docker build --build-arg=REACT_APP_MOVIE_API_URL=$REACT_APP_MOVIE_API_URL --tag=mp-frontend:latest .

# Run container locally
docker run --name mp-frontend -p 3000:3000 -d mp-frontend

# Open browser to localhost:3000 to verify
# Stop container
docker stop mp-frontend
```

### 5. Deploy Kubernetes Manifests
```bash
cd starter/frontend/k8s

# Set image tag to the newly built version
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

# Set image tag to the newly built version