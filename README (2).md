# Three-Tier Application Deployment on AWS EKS — Step-by-Step Runbook

This guide explains **where to go, where to type each command, what to click, and what to check next** for the deployment. Follow the steps in order. Do not repeat completed steps unless you changed something.

## Project details

| Item | Value |
|---|---|
| GitHub repository | `https://github.com/YashDolare15/aws-devops-automation` |
| Branch | `main` |
| AWS Region | `us-east-1` |
| EKS Cluster | `AWSCluster` |
| Docker Hub image | `yashdolare2230/three-tier-app` |
| Jenkins job | `Three_tier_app` |
| Kubernetes Deployment | `three-tier-app-deployment` |
| Kubernetes Service | `three-tier-app-service` |
| Application container port | `3000` |
| RDS database name | `task_tracker` |
| Kubernetes database Secret | `rds-credentials` |

> **Important:** Terraform resources and paths depend on the `.tf` files in the repository. This guide does not assume a Terraform folder name. First locate the directory that contains the Terraform configuration files.

---

## Step 1 — Open a terminal and clone the GitHub repository

**Where to go**

1. On your Windows computer, open **File Explorer** and choose the folder where you want to keep the project, for example `D:\Projects`.
2. Open **Windows PowerShell** or the VS Code terminal (**Terminal → New Terminal**).
3. In PowerShell, move to the folder you chose. Example:

```powershell
cd D:\Projects
```

4. Clone the repository:

```powershell
git clone https://github.com/YashDolare15/aws-devops-automation.git
cd aws-devops-automation
```

5. Confirm that you are inside the repository:

```powershell
git status
Get-ChildItem
```

**What to check**

- `git status` should show the current branch (normally `main`) and whether there are uncommitted changes.
- `Get-ChildItem` lists the project files and folders.

**If you already cloned the repository:** do not clone it again. Use `cd` to enter the existing `aws-devops-automation` folder.

---

## Step 2 — Find the Terraform files

**Where to go**

Stay in the PowerShell terminal opened in the repository folder.

Run:

```powershell
Get-ChildItem -Recurse -Include *.tf
```

This lists Terraform configuration files. Identify the folder that contains the main `.tf` files (for example, files defining the VPC, EKS, RDS, or related resources).

Move into that folder. Replace `<terraform-folder>` with the actual folder path shown on your computer:

```powershell
cd <terraform-folder>
Get-ChildItem
```

Do not type the angle brackets; use the real folder name/path.

**What to check:** you should see one or more `.tf` files such as `main.tf`, `variables.tf`, or module configuration files.

---

## Step 3 — Initialize, validate, plan, and apply Terraform

**Where to type:** the same PowerShell terminal, currently in the Terraform working directory.

Run each command one at a time:

```powershell
terraform init
```

This initializes Terraform and downloads the providers/modules required by the configuration.

```powershell
terraform validate
```

This checks whether the Terraform configuration is syntactically valid.

```powershell
terraform plan
```

Review the resources Terraform proposes to create, change, or destroy.

```powershell
terraform apply
```

Read the plan carefully. If it is correct, type `yes` when Terraform asks for confirmation.

**What to do after applying**

- Wait until Terraform finishes.
- Read the final output and check for errors.
- Do not run `terraform destroy`; it would remove infrastructure.
- If Terraform reports an error, stop and resolve it before continuing.
- Keep any generated outputs, such as resource names or endpoints, for verification. Do not publish passwords or secrets.

---

## Step 4 — Verify the AWS resources in the AWS Console

**Where to go**

1. Open the [AWS Management Console](https://console.aws.amazon.com/).
2. Sign in to the AWS account used for this project.
3. In the AWS Region selector at the top-right, choose **US East (N. Virginia) — `us-east-1`**.

**Check EKS**

1. Use the Console search bar and search for **EKS**.
2. Open **Amazon Elastic Kubernetes Service**.
3. Choose **Clusters**.
4. Open `AWSCluster`.
5. Confirm the cluster status is **Active**.
6. Open the cluster's **Compute** or node-group area and check that the worker nodes/node group are healthy.

**Check RDS**

1. Search for **RDS** in the AWS Console.
2. Open **Amazon RDS → Databases**.
3. Locate the database instance used by the application.
4. Confirm it is available and that the database name/configuration matches `task_tracker`.

Do not copy database passwords into screenshots, README files, or public repositories.

---

## Step 5 — Check the Dockerfile and application port

**Where to go**

1. Return to your local project folder in VS Code or File Explorer.
2. Open the repository folder.
3. Locate the application's `Dockerfile` (Linux filenames are case-sensitive when used on Linux hosts/build agents).
4. Open it in VS Code and review the instructions.

The Dockerfile describes how Docker packages the application: the base image/runtime, dependency installation, application files, working directory, and startup command.

**Check these points**

- Required application files are included in the Docker build context.
- Dependencies are installed as part of the image build where required.
- The startup command launches the application.
- The application listens on container port `3000`, as expected by this deployment.
- Do not add real database passwords or other secrets directly to the Dockerfile.

If the Dockerfile already exists and works, use it; do not create a duplicate.

---

## Step 6 — Build and optionally test the Docker image manually

The normal pipeline builds and pushes the image automatically. This manual step is optional and is mainly for troubleshooting.

**Where to type:** PowerShell or a terminal with Docker installed and running, in the directory containing the Dockerfile.

Build a test image:

```powershell
docker build -t yashdolare2230/three-tier-app:test .
```

Check that the image exists:

```powershell
docker images
```

If you want to test it locally and the application is configured to run without requiring unavailable AWS resources, start it:

```powershell
docker run --rm -p 3000:3000 yashdolare2230/three-tier-app:test
```

Open `http://localhost:3000` in a browser to test it. Press `Ctrl+C` in the terminal to stop it.

If the app requires RDS environment variables or other services, a local run may not work until those dependencies are configured. The Jenkins pipeline has its own **Build Docker Image** and **Test Docker Image** stages.

---

## Step 7 — Push any required changes to GitHub

**Where to type:** PowerShell in the repository folder (not necessarily the Terraform subfolder).

Return to the repository root if needed:

```powershell
cd <path-to-aws-devops-automation>
```

Check which files changed:

```powershell
git status
```

If you edited files, stage and commit only the intended files. Example:

```powershell
git add Dockerfile kubernetes Jenkinsfile
git commit -m "Update deployment configuration"
git push origin main
```

Adjust the paths to match the actual files you changed. If you did not change anything, do not make an empty commit.

**Security check:** make sure `.env` files containing passwords, database credentials, tokens, or keys are not committed. If real secrets were pushed to GitHub, remove them from the repository and rotate the exposed credentials; deleting a file in a later commit alone does not make an exposed secret safe.

---

## Step 8 — Open Jenkins and check credentials

**Where to go**

1. Open a browser on your computer.
2. Open the Jenkins URL configured for your EC2 instance. The previously used URL was `http://54.90.83.23:8081`; use the current public IP/DNS and port if that address has changed.
3. Sign in to Jenkins.
4. Open **Manage Jenkins → Credentials** (the exact menu wording may vary by Jenkins version).
5. Open the appropriate credentials store/domain and verify the credential IDs below. Do not reveal or paste the secret values into the README.

| Credential ID | Jenkins credential type | Value to store |
|---|---|---|
| `dockerhub-credentials` | Username with password | Docker Hub username and password/token |
| `rds-db-host` | Secret text | RDS endpoint/host |
| `rds-db-name` | Secret text | `task_tracker` |
| `rds-mysql-credentials` | Username with password | Database username and password |

If a credential is missing, choose **Add Credentials**, select the type shown in the table, enter the value, and use the exact ID listed above. Save it.

---

## Step 9 — Verify the Jenkins EC2 IAM role and EKS access

The Jenkins EC2 instance uses the IAM role **`JenkinsEKSAccessRole`**. This lets the AWS CLI on the instance obtain temporary credentials without storing long-lived AWS access keys in Jenkins.

### 9.1 Check the IAM role in AWS Console

1. Open the AWS Console and select `us-east-1`.
2. Search for **EC2 → Instances**.
3. Select the EC2 instance running Jenkins.
4. In the instance details, find **IAM role** and confirm it is `JenkinsEKSAccessRole`.
5. If it is not attached, do not attach an arbitrary role; verify the intended instance and role before making changes.

### 9.2 Check the role policy

1. Search for **IAM → Roles**.
2. Open `JenkinsEKSAccessRole`.
3. Under **Permissions**, verify the custom policy `JenkinsEKSDescribeCluster` is attached.
4. The confirmed custom policy JSON is:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "eks:DescribeCluster",
      "Resource": "arn:aws:eks:us-east-1:543525762244:cluster/AWSCluster"
    }
  ]
}
```

This policy allows `eks:DescribeCluster` for the named cluster. By itself, it does **not** grant Kubernetes deployment permissions.

### 9.3 Check EKS access

An EKS access entry was created for:

`arn:aws:iam::543525762244:role/JenkinsEKSAccessRole`

For the initial test, `AmazonEKSClusterAdminPolicy` was associated with **cluster** scope. This is broad administrator access; for production, replace it with appropriately scoped permissions after validating the deployment workflow.

**Where to check it:** AWS Console → **EKS → Clusters → `AWSCluster` → Access**. Confirm the access entry and associated policy.

### 9.4 Test access from the Jenkins machine

**Where to type:** connect to the Jenkins EC2 instance using your approved SSH method, then run the following commands on that EC2 terminal:

```bash
sudo -u jenkins aws sts get-caller-identity
```

This should show the AWS identity available to the Jenkins operating-system user.

```bash
sudo -u jenkins aws eks describe-cluster --region us-east-1 --name AWSCluster --query 'cluster.status' --output text
```

Expected result:

```text
ACTIVE
```

Then verify Kubernetes access:

```bash
sudo -u jenkins aws eks update-kubeconfig --region us-east-1 --name AWSCluster
sudo -u jenkins kubectl get nodes
```

The worker nodes should appear with status `Ready`. If a command fails, fix the AWS/IAM/EKS access issue before starting a deployment.

---

## Step 10 — Open the Jenkins pipeline job and run it

**Where to go**

1. In the Jenkins browser page, click **Dashboard**.
2. Open the job **`Three_tier_app`**.
3. If the job is configured to build from the repository, confirm it points to the correct GitHub repository and `main` branch.
4. Click **Build Now** (or **Build with Parameters** if that is what your job shows).
5. Open the new build number in **Build History**.
6. Click **Console Output** to watch the pipeline.

**Expected pipeline stages**

1. **Checkout** — downloads the configured source from GitHub.
2. **Validate Application** — checks required files/configuration.
3. **Build Docker Image** — builds the application image from the Dockerfile.
4. **Test Docker Image** — runs the configured image checks.
5. **Push Image to Docker Hub** — publishes the versioned image to `yashdolare2230/three-tier-app`.
6. **Connect to EKS** — configures `kubectl` access to `AWSCluster`.
7. **Configure RDS Secret** — creates/updates Kubernetes Secret `rds-credentials`.
8. **Deploy to Kubernetes** — applies the Kubernetes Deployment and Service and updates the image tag.
9. **Verify Deployment** — waits for the rollout and checks the deployment status.

Wait for the build to finish. **Success means the pipeline completes successfully**, but you should still verify the Kubernetes resources and open the application URL.

---

## Step 11 — Confirm the database Secret name matches the Deployment

The pipeline creates or updates Kubernetes Secret **`rds-credentials`**, containing these keys:

- `DB_HOST`
- `DB_NAME`
- `DB_USER`
- `DB_PASSWORD`

The Deployment manifest must refer to the same Secret name. A previous issue occurred when the Deployment expected `three-tier-app-db-secrets` while the pipeline created `rds-credentials`; that mismatch caused `CreateContainerConfigError`. The manifest was corrected to use `rds-credentials`.

**Where to check**

1. In the local repository, open the Kubernetes Deployment manifest (expected path in this project: `kubernetes/deployment.yaml`).
2. Find the `env`/`envFrom` section that references the Secret.
3. Confirm it refers to `rds-credentials` and the key names match those created by the pipeline.
4. If you change the manifest, save it, commit/push the change to `main`, and run the Jenkins job again.

Do not put the Secret's actual password values in the YAML or README.

---

## Step 12 — Verify the Kubernetes deployment

**Where to type:** SSH terminal on the Jenkins EC2 instance, using the same user/access setup that successfully ran `kubectl`.

Check the Deployment:

```bash
sudo -u jenkins kubectl get deployment three-tier-app-deployment
```

Check the Pods:

```bash
sudo -u jenkins kubectl get pods
```

Check the Service:

```bash
sudo -u jenkins kubectl get service three-tier-app-service
```

Check the rollout:

```bash
sudo -u jenkins kubectl rollout status deployment/three-tier-app-deployment --timeout=300s
```

**What success looks like**

- Deployment shows all desired replicas available.
- Pods show `Running` and `READY` counts such as `1/1`.
- Service exists and has the expected port mapping.
- Rollout reports that it successfully rolled out.

The latest observed successful state had **3/3 pods Running/Ready**, Deployment **3/3 available**, and image `yashdolare2230/three-tier-app:25`. Your current build may use a newer tag.

If a Pod is not ready, inspect it:

```bash
sudo -u jenkins kubectl describe pod <pod-name>
sudo -u jenkins kubectl logs <pod-name>
```

Replace `<pod-name>` with the actual Pod name from `kubectl get pods`. Do not paste passwords or other secrets into logs or public issues.

---

## Step 13 — Find the website URL and open it

**First, check the Service type and address.** On the Jenkins EC2 terminal, run:

```bash
sudo -u jenkins kubectl get service three-tier-app-service
```

Read the `TYPE`, `EXTERNAL-IP`, and `PORT(S)` columns.

- **If `TYPE` is `LoadBalancer`**: wait for `EXTERNAL-IP` to show a hostname/address rather than `<pending>`. Copy that hostname/address into your browser using the protocol configured for the application (often `http://` unless HTTPS has been configured).
- **If `TYPE` is `NodePort`**: use a reachable worker-node address plus the assigned NodePort, provided the network/security rules allow access.
- **If `TYPE` is `ClusterIP`**: it is internal to the cluster by default. It will not be directly reachable from your home browser; an ingress/load balancer or another approved access method is needed.

If the external address is still pending, wait a few minutes and run the Service command again. Do not guess a public URL from the Pod IP.

After obtaining the real address:
1. Open Chrome or another browser on your computer.
2. Paste the Service/load-balancer address into the address bar.
3. Check whether the application page loads.
4. If it does not load, verify the Service, Pod readiness, application port, load balancer/network rules, and application logs.

---

## IAM roles and permissions — confirmed project details

| Name | Purpose |
|---|---|
| `JenkinsEKSAccessRole` | IAM role attached to the Jenkins EC2 instance |
| `JenkinsEKSDescribeCluster` | Custom IAM policy allowing `eks:DescribeCluster` on `AWSCluster` |
| `AmazonEKSClusterAdminPolicy` | EKS access policy associated with the Jenkins role at cluster scope during initial testing |

The JSON shown in Step 9 is the confirmed JSON for `JenkinsEKSDescribeCluster`. The EKS access policy `AmazonEKSClusterAdminPolicy` is associated through EKS access management; it is not the custom IAM policy JSON shown above.

The exact IAM role names and policy JSON for the EKS cluster service role, worker-node role, and Terraform execution identity have not been confirmed here. Check the actual AWS Console/Terraform configuration before documenting those names or copying policy JSON. Do not attach broad administrator permissions to roles just to bypass an error.

---

## Troubleshooting quick reference

| Symptom | Where to check next |
|---|---|
| `terraform` command not found | Install Terraform or correct the terminal `PATH`, then reopen PowerShell |
| Terraform fails during `apply` | Read the first meaningful error; verify AWS account/region, permissions, variables, and resource conflicts |
| Docker build fails | Check the Dockerfile, build context, dependency installation, and Docker daemon |
| Jenkins cannot push the image | Verify `dockerhub-credentials` ID and Docker Hub repository permissions |
| `aws eks describe-cluster` fails | Check the Jenkins EC2 IAM role, policy, region, and cluster name |
| `kubectl get nodes` is forbidden | Check the EKS access entry and associated access policy for the Jenkins role |
| Pod shows `CreateContainerConfigError` | Run `kubectl describe pod <pod-name>`; check Secret name and required Secret keys |
| Pod is `CrashLoopBackOff` | Check `kubectl logs <pod-name>` and application configuration |
| Service external address is pending | Check Service type, AWS load balancer provisioning, subnet tags, and cluster events |
| Website does not open | Check Service port, container port `3000`, Pod readiness, load balancer address, and network/security rules |

## Final deployment checklist

- [ ] Repository cloned and correct branch checked out.
- [ ] Terraform initialized, validated, reviewed, and applied successfully.
- [ ] EKS cluster is Active and worker nodes are Ready.
- [ ] Dockerfile and container port checked.
- [ ] Required changes pushed to GitHub `main`.
- [ ] Jenkins credential IDs verified without exposing secret values.
- [ ] Jenkins EC2 IAM role and EKS access verified.
- [ ] `Three_tier_app` pipeline completed successfully.
- [ ] Kubernetes Secret is named `rds-credentials` and matches the Deployment manifest.
- [ ] Deployment rollout completed and Pods are Ready.
- [ ] Service address obtained from Kubernetes and website tested in a browser.
