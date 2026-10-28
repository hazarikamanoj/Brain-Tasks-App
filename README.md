This project containerizes the Brain Tasks web app (a prebuilt Vite `dist/` bundle), stores the image in **Amazon ECR**, deploys it to **Amazon EKS** behind a **Classic Load Balancer on port 3000**, and automates build and deploy with **Jenkins** and **AWS CodePipeline**. Monitoring uses **CloudWatch Container Insights**.

> AWS CodeBuild could not be used because the account had a concurrent-build quota of 0 (see [Troubleshooting](#14-troubleshooting-and-lessons-learned)). As agreed with my mentor, Jenkins performs the build stage.

## Architecture

```mermaid
flowchart LR
  A[GitHub push] --> B[CodePipeline Source]
  B --> C[Jenkins on EC2]
  C --> D[docker build]
  D --> E[Amazon ECR]
  C --> F[kubectl apply]
  F --> G[Amazon EKS]
  G --> H[Classic Load Balancer :3000]
  G --> I[CloudWatch Container Insights]
```

## Table of Contents

1. [Project Details](#2-project-details)
2. [Repository Setup](#3-step-1-repository-setup)
3. [Dockerize](#4-step-2-dockerize-the-application-port-3000)
4. [Amazon ECR](#5-step-3-amazon-ecr)
5. [Amazon EKS](#6-step-4-amazon-eks-cluster)
6. [Kubernetes Deployment](#7-step-5-kubernetes-manifests-and-manual-deployment)
7. [CodeBuild Attempt](#8-step-6-aws-codebuild-attempt)
8. [Jenkins Server](#9-step-7-jenkins-server-on-ec2)
9. [Jenkins Pipeline](#10-step-8-jenkins-pipeline-job-jenkinsfile)
10. [Auto-Trigger Test](#11-step-9-auto-trigger-test-jenkins-scm-polling)
11. [CloudWatch](#12-step-10-cloudwatch-container-insights)
12. [CodePipeline](#13-step-11-aws-codepipeline-with-jenkins)
13. [Troubleshooting](#14-troubleshooting-and-lessons-learned)
14. [Cleanup](#15-cleanup-after-evaluation)

## 2. Project Details

| Item | Value |
|---|---|
| Application | Brain Tasks (prebuilt Vite `dist/` bundle) |
| GitHub repository | https://github.com/hazarikamanoj/Brain-Tasks-App |
| AWS region | `us-east-1` |
| ECR repository | `brain-tasks-app` |
| Image URI | `629845796956.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app` |
| EKS cluster | `brain-tasks-cluster` (2 x `c7i-flex.large` nodes) |
| Kubernetes objects | Deployment `brain-tasks-app` (2 replicas, port 3000); Service `brain-tasks-service` (LoadBalancer, port 3000) |
| Load balancer type | Classic Load Balancer |
| Load balancer name | `ad895c2cd6b4e471b84a13364633f382` |
| Application URL | http://ad895c2cd6b4e471b84a13364633f382-716084115.us-east-1.elb.amazonaws.com:3000 |
| Load balancer ARN | `arn:aws:elasticloadbalancing:us-east-1:629845796956:loadbalancer/ad895c2cd6b4e471b84a13364633f382` |
| Jenkins server | EC2 `t3.small`, Amazon Linux 2023, http://54.242.146.206:8080 (public IP changes if the instance is stopped) |
| Jenkins jobs | `brain-tasks-pipeline` (Jenkinsfile) and `brain-tasks-codepipeline-job` (CodePipeline Build stage) |
| CodePipeline | `brain-tasks-pipeline`: Source (GitHub) then Build (Jenkins, provider `JenkinsProvider`) |
| Monitoring | CloudWatch Container Insights; log groups under `/aws/containerinsights/brain-tasks-cluster/` |
| IAM roles | `jenkins-ec2-role` (Jenkins server); `codebuild-brain-tasks-build-service-role` (CodeBuild attempt) |

> **Note:** a Classic Load Balancer does not expose an ARN field in the CLI, so the ARN above is built from the standard format `arn:aws:elasticloadbalancing:<region>:<account-id>:loadbalancer/<name>` using the name returned by `aws elb describe-load-balancers`.

## Repository Structure

```
Brain-Tasks-App/
├── dist/                  # prebuilt Vite app
│   ├── assets/
│   ├── index.html
│   └── vite.svg
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── screenshots/           # all proof images
├── .dockerignore
├── buildspec.yml
├── Dockerfile
├── Jenkinsfile
├── nginx.conf
└── README.md
```

## 3. Step 1: Repository Setup

Created a new public GitHub repository, cloned the source project, pointed the remote at my repository and pushed to `main`.

```bash
git clone https://github.com/Vennilavanguvi/Brain-Tasks-App.git
cd Brain-Tasks-App
git remote set-url origin https://github.com/hazarikamanoj/Brain-Tasks-App.git
git push -u origin main
```

![A0-1-CreateRepo](screenshots/A0-1-CreateRepo.PNG)
*Figure 1: Creating the new public GitHub repository Brain-Tasks-App.*

![A0-2-GitRemote-vGitpush](screenshots/A0-2-GitRemote-vGitpush.PNG)
*Figure 2: `git remote -v` showing my repository, and the successful push to main.*

![A0-3-distReadme](screenshots/A0-3-distReadme.PNG)
*Figure 3: Repository on GitHub showing the `dist/` folder and `README.md` after the first push.*

![05-GitHubRepo](screenshots/05-GitHubRepo.PNG)
*Figure 4: GitHub repository with all project files (Dockerfile, nginx.conf, k8s/, buildspec.yml, Jenkinsfile).*

![01-FoldersVSCode](screenshots/01-FoldersVSCode.PNG)
*Figure 5: VS Code Explorer showing the final folder structure.*

## 4. Step 2: Dockerize the Application (port 3000)

The app is served as static files by nginx on port 3000.

```bash
docker build -t brain-tasks-app .
docker run -d -p 3000:3000 --name brain-tasks brain-tasks-app
docker images
docker ps
```

![02-Dockerfile](screenshots/02-Dockerfile.PNG)
*Figure 6: Dockerfile (nginx:alpine, copies `dist/` to the web root, exposes port 3000).*

![02-nginx](screenshots/02-nginx.PNG)
*Figure 7: `nginx.conf` listening on port 3000 with a single-page-app fallback to `index.html`.*

![06-DockerBuild](screenshots/06-DockerBuild.PNG)
*Figure 8: `docker build` completing successfully.*

![07-dockerImages](screenshots/07-dockerImages.PNG)
*Figure 9: `docker images` listing `brain-tasks-app:latest`.*

![08-dockerPs](screenshots/08-dockerPs.PNG)
*Figure 10: `docker ps` showing the container running with port 3000 mapped.*

![09-localhost3000](screenshots/09-localhost3000.PNG)
*Figure 11: Application running locally at http://localhost:3000.*

## 5. Step 3: Amazon ECR

Created a private repository, logged in, tagged and pushed the image.

```bash
aws ecr create-repository --repository-name brain-tasks-app --region us-east-1
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 629845796956.dkr.ecr.us-east-1.amazonaws.com
docker tag brain-tasks-app:latest 629845796956.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app:latest
docker push 629845796956.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app:latest
```

![10-ECR-repositoryCreated](screenshots/10-ECR-repositoryCreated.PNG)
*Figure 12: ECR repository `brain-tasks-app` created.*

![11-docker-push](screenshots/11-docker-push.PNG)
*Figure 13: `docker push` output with the image layers pushed to ECR.*

![12-ECR-image-itsTag](screenshots/12-ECR-image-itsTag.PNG)
*Figure 14: ECR console showing the pushed image with its `latest` tag.*

## 6. Step 4: Amazon EKS Cluster

The account only offered free-tier-eligible instance types, so the node group uses `c7i-flex.large` instead of `t3.medium`.

```bash
eksctl create cluster --name brain-tasks-cluster --region us-east-1 \
  --nodegroup-name ng-1 --node-type c7i-flex.large --nodes 2
kubectl get nodes
```

![13-eksctlCreateCluster](screenshots/13-eksctlCreateCluster.PNG)
*Figure 15: eksctl creating the `brain-tasks-cluster` cluster and managed node group.*

![14-EKSConsole](screenshots/14-EKSConsole.PNG)
*Figure 16: EKS console showing the cluster with status Active.*

![15-kubectlGetNodes](screenshots/15-kubectlGetNodes.PNG)
*Figure 17: `kubectl get nodes` showing both worker nodes Ready.*

![16-Node-group](screenshots/16-Node-group.PNG)
*Figure 18: Managed node group `ng-1` in the EKS console.*

## 7. Step 5: Kubernetes Manifests and Manual Deployment

The Deployment runs 2 replicas on port 3000 and the Service of type LoadBalancer exposes it through a Classic Load Balancer. The `image:` line in `deployment.yaml` is replaced with the commit-tagged ECR image at build time.

```bash
kubectl apply -f k8s/
kubectl get pods,svc,deployments
aws elb describe-load-balancers --region us-east-1 \
  --query "LoadBalancerDescriptions[].[LoadBalancerName,DNSName]" --output table
```

![03-k8s-deployment-yaml](screenshots/03-k8s-deployment-yaml.PNG)
*Figure 19: `k8s/deployment.yaml` (2 replicas, containerPort 3000, readiness probe, resource requests and limits).*

![03-k8s-service-yaml](screenshots/03-k8s-service-yaml.PNG)
*Figure 20: `k8s/service.yaml` (type LoadBalancer, port 3000 to targetPort 3000).*

![17-kubectlApply-f-k8s](screenshots/17-kubectlApply-f-k8s.PNG)
*Figure 21: `kubectl apply -f k8s/` creating the Deployment and Service.*

![18-kubectlGetNodesSvcDeployments](screenshots/18-kubectlGetNodesSvcDeployments.PNG)
*Figure 22: Pods Running and the Service showing the load balancer EXTERNAL-IP.*

![19-ClassicLoadBalancer](screenshots/19-ClassicLoadBalancer.PNG)
*Figure 23: EC2 console showing the Classic Load Balancer with 2 of 2 instances in service.*

![20-Browser-at-LB-URL-showing-app](screenshots/20-Browser-at-LB-URL-showing-app.PNG)
*Figure 24: Brain Tasks application open through the load balancer URL on port 3000.*

![21-aws-elbDescribe-load-balancers](screenshots/21-aws-elbDescribe-load-balancers.PNG)
*Figure 25: `aws elb describe-load-balancers` output showing the load balancer name and DNS name.*

![21b-LB-ARN-echo](screenshots/21b-LB-ARN-echo.PNG)
*Figure 26: Load balancer ARN built from the standard format and printed in the terminal.*

## 8. Step 6: AWS CodeBuild Attempt

A CodeBuild project (`brain-tasks-build`) was configured with a `buildspec.yml` that builds the image, pushes it to ECR and deploys to EKS. Builds could not start because of the account-level quota (details in [Troubleshooting](#14-troubleshooting-and-lessons-learned)). The `buildspec.yml` is kept in the repository.

![04-buildspec-yml](screenshots/04-buildspec-yml.PNG)
*Figure 27: `buildspec.yml` (ECR login, docker build and push, kubectl apply, rollout status).*

![22-Project-configuration-source](screenshots/22-Project-configuration-source.PNG)
*Figure 28: CodeBuild project configuration with the GitHub source and main branch.*

![22-Project-configuration-environment-a](screenshots/22-Project-configuration-environment-a.PNG)
*Figure 29: CodeBuild environment settings (managed image, Amazon Linux, service role).*

![22-Project-configuration-environment-b](screenshots/22-Project-configuration-environment-b.PNG)
*Figure 30: CodeBuild additional configuration with Privileged mode enabled for Docker builds.*

![22-Project-configuration-buildspecLogs](screenshots/22-Project-configuration-buildspecLogs.PNG)
*Figure 31: CodeBuild buildspec, artifact and CloudWatch Logs settings.*

![23-IAM-role-permissions](screenshots/23-IAM-role-permissions.PNG)
*Figure 32: CodeBuild service role permissions (ECR PowerUser and `eks-describe` inline policy).*

![24-iamidentitymapping-cmd-output](screenshots/24-iamidentitymapping-cmd-output.PNG)
*Figure 33: eksctl identity mapping giving the CodeBuild role access to the cluster.*

![25-CodeBuild-errorZeroBuilds](screenshots/25-CodeBuild-errorZeroBuilds.PNG)
*Figure 34: CodeBuild error: "Cannot have more than 0 builds in queue for the account".*

## 9. Step 7: Jenkins Server on EC2

Jenkins runs on an Amazon Linux 2023 EC2 instance with Docker, kubectl and AWS CLI. An IAM instance role (`jenkins-ec2-role`) gives it ECR and EKS access without stored keys, and the role is mapped into the cluster.

```bash
eksctl create iamidentitymapping --cluster brain-tasks-cluster --region us-east-1 \
  --arn arn:aws:iam::629845796956:role/jenkins-ec2-role \
  --group system:masters --username jenkins

sudo -u jenkins docker ps
sudo -u jenkins aws sts get-caller-identity
sudo -u jenkins kubectl get nodes
```

![30-EC2-Jenkins-instance](screenshots/30-EC2-Jenkins-instance.PNG)
*Figure 35: EC2 instance `jenkins-server` running.*

![29-SecurityGroup-jenkins](screenshots/29-SecurityGroup-jenkins.PNG)
*Figure 36: Jenkins instance details and security group rules (SSH and port 8080).*

![31-Jenkins-user-checks](screenshots/31-Jenkins-user-checks.PNG)
*Figure 37: Jenkins user can run docker, assumes `jenkins-ec2-role`, and kubectl lists both nodes.*

![31-Jenkins-user-checks-b](screenshots/31-Jenkins-user-checks-b.PNG)
*Figure 38: Additional Jenkins server validation (service status and tool checks).*

![32-Jenkins-dashboard](screenshots/32-Jenkins-dashboard.PNG)
*Figure 39: Jenkins dashboard after setup.*

## 10. Step 8: Jenkins Pipeline Job (Jenkinsfile)

The job `brain-tasks-pipeline` reads the `Jenkinsfile` from GitHub and runs four stages: **Checkout, Build Image, Push to ECR, Deploy to EKS**. The image is tagged with the short commit hash.

![33-Jenkins-job-config](screenshots/33-Jenkins-job-config.PNG)
*Figure 40: Jenkins job configuration: Pipeline script from SCM with the GitHub repository.*

![33-Jenkins-job-config-b](screenshots/33-Jenkins-job-config-b.PNG)
*Figure 41: Jenkins job configuration: branch, script path and build trigger.*

![34-Jenkinsfile-VSCode](screenshots/34-Jenkinsfile-VSCode.PNG)
*Figure 42: Jenkinsfile in VS Code (environment and Checkout / Build stages).*

![34-Jenkinsfile-VSCode-b](screenshots/34-Jenkinsfile-VSCode-b.PNG)
*Figure 43: Jenkinsfile in VS Code (Push to ECR and Deploy to EKS stages).*

![35-Jenkins-StageView](screenshots/35-Jenkins-StageView.PNG)
*Figure 44: Jenkins Stage View with all stages successful.*

![36-Jenkins-ConsoleOutput](screenshots/36-Jenkins-ConsoleOutput.PNG)
*Figure 45: Console output ending with the ECR push and successful rollout.*

![37-ECR-new-commit-tag](screenshots/37-ECR-new-commit-tag.PNG)
*Figure 46: ECR showing the new commit-tagged image beside `latest`.*

![38-kubectl-pods](screenshots/38-kubectl-pods.PNG)
*Figure 47: `kubectl get pods` showing freshly rolled-out pods Running.*

![38b-App-in-browser](screenshots/38b-App-in-browser.PNG)
*Figure 48: Application served by the Jenkins-deployed version at the load balancer URL.*

## 11. Step 9: Auto-Trigger Test (Jenkins SCM Polling)

A small commit was pushed to GitHub and Jenkins started a new build by itself, built a new image tag and rolled out new pods.

```bash
git add README.md
git commit -m "Test: auto-trigger Jenkins pipeline"
git push origin main
```

![44-Auto-trigger-commit-push](screenshots/44-Auto-trigger-commit-push.PNG)
*Figure 49: Terminal showing the commit and `git push` used for the test.*

![45-Auto-triggered-build](screenshots/45-Auto-triggered-build.PNG)
*Figure 50: Jenkins build started automatically by an SCM change.*

![46-ECR-second-tag-and-pods](screenshots/46-ECR-second-tag-and-pods.PNG)
*Figure 51: ECR with a second image tag and the new pods after the rollout.*

![46b-EC2-Terminal-Validation](screenshots/46b-EC2-Terminal-Validation.PNG)
*Figure 52: Terminal validation on the Jenkins server: running image tag and pod ages.*

## 12. Step 10: CloudWatch Container Insights

The `amazon-cloudwatch-observability` add-on was installed after attaching `CloudWatchAgentServerPolicy` to the node role.

```bash
aws iam attach-role-policy --role-name <node-instance-role> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
aws eks create-addon --cluster-name brain-tasks-cluster \
  --addon-name amazon-cloudwatch-observability --region us-east-1
kubectl get pods -n amazon-cloudwatch
```

![39-40-kubectl_get_pods_nAmazon-cloudwatch-RunningPods](screenshots/39-40-kubectl_get_pods_nAmazon-cloudwatch-RunningPods.PNG)
*Figure 53: Add-on ACTIVE and the CloudWatch agent and Fluent Bit pods Running.*

![41-Log-groups-list](screenshots/41-Log-groups-list.PNG)
*Figure 54: CloudWatch log groups created for the cluster (application, dataplane, host, performance).*

![42-Container-Insights-dashboard-brain-tasks-cluster](screenshots/42-Container-Insights-dashboard-brain-tasks-cluster.PNG)
*Figure 55: Container Insights performance dashboard for `brain-tasks-cluster`.*

![43-LogsInsights-application-log-lines](screenshots/43-LogsInsights-application-log-lines.PNG)
*Figure 56: Logs Insights query showing application log lines from the `brain-tasks-app` pods.*

## 13. Step 11: AWS CodePipeline with Jenkins

CodePipeline pulls the source from GitHub and hands the Build stage to Jenkins through the AWS CodePipeline plugin. The Jenkins job `brain-tasks-codepipeline-job` downloads the source artifact, builds and pushes the image, deploys to EKS and reports the result back to CodePipeline.

![28-SourceStageConnectionRepoDetails](screenshots/28-SourceStageConnectionRepoDetails.PNG)
*Figure 57: Source stage configuration: GitHub connection, repository and branch.*

![47a-Jenkins-CodePipeline-plugin](screenshots/47a-Jenkins-CodePipeline-plugin.PNG)
*Figure 58: AWS CodePipeline plugin installed in Jenkins.*

![47-Jenkins-console-output](screenshots/47-Jenkins-console-output.PNG)
*Figure 59: Jenkins console output for the CodePipeline-triggered build, ending with a successful rollout.*

![48-Pipeline-stage-config](screenshots/48-Pipeline-stage-config.PNG)
*Figure 60: Build stage action configuration: Jenkins provider name, server URL and project name.*

![48-Pipeline-stage-config-b](screenshots/48-Pipeline-stage-config-b.PNG)
*Figure 61: Build stage action: input artifact (`SourceArtifact`) and remaining settings.*

![49-Pipeline-successful-run](screenshots/49-Pipeline-successful-run.PNG)
*Figure 62: CodePipeline with the Source and Build stages succeeded.*

![50-Pipeline-auto-trigger](screenshots/50-Pipeline-auto-trigger.PNG)
*Figure 63: Commit and push that triggers the pipeline automatically.*

![50b-Pipeline-execution-started](screenshots/50b-Pipeline-execution-started.PNG)
*Figure 64: New pipeline execution started by the push.*

![50c-Jenkins-build](screenshots/50c-Jenkins-build.PNG)
*Figure 65: Jenkins build started by CodePipeline and finished successfully.*

![50d-ECR-and-pods](screenshots/50d-ECR-and-pods.PNG)
*Figure 66: ECR with the new `cp-N` image tag and the updated pods.*

## 14. Troubleshooting and Lessons Learned

| Issue | Cause | Fix |
|---|---|---|
| `t3.medium` not available for EKS nodes | Account offered only free-tier-eligible instance types | Used `c7i-flex.large` nodes |
| CodeBuild: "Cannot have more than 0 builds in queue" | Account-level concurrent build quota was 0; quota request was denied | As agreed with my mentor, Jenkins performs the build stage |
| Classic Load Balancer has no ARN in the CLI | CLBs are not listed by `aws elbv2` | Built the ARN from the standard format and the load balancer name |
| Jenkins built-in node offline ("lack of disk space") | `/tmp` is a small RAM-backed tmpfs on Amazon Linux 2023 | Set `java.io.tmpdir` to `/var/lib/jenkins/tmp` and restarted Jenkins |
| `kubectl` Unauthorized from the build server | IAM role not mapped into the cluster | Ran `eksctl create iamidentitymapping` for the role |
| Inline IAM policy granted nothing | Policy was created without actions | Replaced it with a JSON policy allowing `eks:DescribeCluster` |
| Dockerfile not found in the CodePipeline job workspace | Plugin could not unpack the source zip (missing `commons-compress` library) and the Build action needs `SourceArtifact` as input | Added the `commons-compress` jar to the plugin lib folder, restarted Jenkins and set the input artifact |
| CodePipeline role name already exists | Role left over from a deleted pipeline | Used a new unique service role name |


## 15. Conclusion

The application is containerized, stored in ECR, deployed to EKS behind a Classic Load Balancer on port 3000, built and deployed automatically on every push through Jenkins and CodePipeline, and monitored with CloudWatch Container Insights.
