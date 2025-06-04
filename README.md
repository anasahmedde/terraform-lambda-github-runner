# 🚀 Self-Hosted GitHub Action Runners on AWS Spot Instances

> **By Anas Ahmed**

This Terraform module builds the AWS infrastructure required to run self-hosted GitHub Actions runners on spot instances, auto-scaling in and out as needed. Using AWS Lambda functions, it watches for GitHub workflow events, spins up spot instances when jobs queue, and terminates idle runners to keep costs minimal.

---

## 🔍 Why This Module Exists

GitHub Actions’ built-in runners can be convenient, but self-hosted runners let you control performance, security, and cost. This module automates the lifecycle of self-hosted runners—provisioning spot instances only when workflows require them, and tearing them down once they go idle. By relying on Lambda functions instead of traditional Auto Scaling Groups, we keep permissions tight and scale with lower latency.

---

## 📦 Major Components

- **GitHub App / Webhook**  
  Receives `workflow_job` or `check_run` events from your GitHub repository or organization.  
- **API Gateway + Lambda**  
  An HTTP endpoint validates GitHub webhook signatures, enqueues relevant events to SQS, and triggers runner provisioning.  
- **SQS Queues**  
  Buffer incoming GitHub events and delay bookings so that existing idle runners can pick up jobs.  
- **Scale-Up Lambda**  
  Listens to the build queue, checks runner availability, and launches new spot instances (via EC2 Fleet) only when necessary.  
- **Scale-Down Lambda**  
  Periodically inspects active runners; if they’re idle longer than a configured threshold, removes them from GitHub and terminates their instances.  
- **Binaries Sync Lambda (Optional)**  
  Mirrors the GitHub Actions runner binaries to an S3 bucket on a schedule, speeding up instance bootstrapping.  
- **EC2 Spot Instances**  
  Each instance comes up with Docker installed and the GitHub Actions runner agent configured—automatically registers itself, executes workflows, then deregisters and powers off when idle.  

![Module Architecture](docs/component-overview.svg)

---

## 🔧 Getting Started

1. **Create a GitHub App**  
   - Go to GitHub → Settings → Developer settings → GitHub Apps.  
   - Click **“New GitHub App”** (organization-level or repo-level).  
   - Give it a name (e.g., `aws-spot-runner`).  
   - Under **Permissions → Repository**, grant:  
     - **Actions**: Read  
     - **Checks**: Read  
     - **Metadata**: Read  
     - _If using repo-level runners_, also grant **Administration**: Read & Write.  
   - Under **Webhook** (temporarily disabled), generate a private key (`.pem`) and note the App ID & Client ID.  
   - Save the private key (we’ll use it soon).  

2. **Prepare Lambda Packages**  
   - Download precompiled Lambda zip files from the module’s GitHub releases or build them locally:  
     ```bash
     cd modules/download-lambda
     terraform init && terraform apply
     ```  
   - You’ll get `webhook.zip`, `runners.zip`, and `runner-binaries-syncer.zip` in that folder.

3. **Grant EC2 Spot Role**  
   - Ensure your AWS account has the **AWSServiceRoleForEC2Spot** service-linked role.  
   - You can create it via Terraform by setting `create_service_linked_role_spot = true` in your configuration, or manually follow AWS docs to add that role.

4. **Configure Terraform Module**  
   - Create a new workspace and reference the module:

     ```hcl
     module "github_runners" {
       source  = "By Anas Ahmed/github-runner/aws"
       version = "REPLACE_WITH_LATEST"

       aws_region = "us-west-2"
       vpc_id     = "vpc-0abc1234"
       subnet_ids = ["subnet-aaa111", "subnet-bbb222", "subnet-ccc333"]

       prefix = "ci-runner"

       github_app = {
         key_base64     = base64encode(file("app.private-key.pem"))
         id             = "123456"
         webhook_secret = "YOUR_WEBHOOK_SECRET"
       }

       webhook_lambda_zip                = "modules/download-lambda/webhook.zip"
       runner_binaries_syncer_lambda_zip = "modules/download-lambda/runner-binaries-syncer.zip"
       runners_lambda_zip                = "modules/download-lambda/runners.zip"

       enable_organization_runners = true
     }
     ```

   - Run Terraform:
     ```bash
     terraform init
     terraform apply
     ```

   - Note the outputs:  
     - **`webhook_url`** – the API Gateway endpoint for your GitHub App.  
     - **`webhook_secret`** – the shared secret for validating GitHub signatures.

5. **Finish GitHub App Setup**  
   - Return to your GitHub App’s **Settings → Webhook**.  
   - Enable the webhook, paste the `webhook_url`, and set the secret to `webhook_secret`.  
   - Under **Permissions & Events**, subscribe to **“Workflow Job”** (preferred) or **“Check Run”**.  
   - Install the App on your organization or individual repositories as needed.

6. **Verify Runners**  
   - Push a simple workflow that targets a self-hosted runner label.  
   - Check CloudWatch Logs for your Lambda functions and the SQS queue to confirm events are flowing.  
   - In the GitHub Actions → Runners page, you should see spot instances coming online as jobs queue, then disappearing when the jobs finish.

---

## ⚙️ Key Configuration Blocks

### 1. Runner Pool vs. On-Demand Pods

- **Idle Runners**  
  By default, the scale-down Lambda tears down all idle runners. To keep a baseline pool, define `idle_config` with cron expressions and `idleCount`.  
- **Spot vs. On-Demand**  
  Choose `instance_target_capacity_type = "spot"` (default) or `"on-demand"`. You can also mix in multiple instance families (e.g., `[ "c6g.large", "m6i.large" ]`) to improve spot availability.

### 2. Event Delay and FIFO

- **Delay Webhook Event**  
  A short delay (30s by default) gives existing idle runners a chance to accept tasks before new instances spin up.  
- **FIFO Queue**  
  For repo-level runners, preserve event order by enabling `fifo_build_queue = true`. Otherwise, set it to `false` for organization-level workflows.

### 3. Ephemeral vs. Reusable Runners

- **Ephemeral Runners**  
  Set `enable_ephemeral_runners = true` to create a runner per workflow job, then immediately retire it once the job completes. Requires `workflow_job` events—no reuse.  
- **Reusable Runners**  
  The default mode—runners handle multiple jobs until idle. Use `minimum_running_time_in_minutes` to ensure a runner stays online long enough to pick up jobs.

---

## 🔐 Encryption & Secrets

- **SSM Parameter Store**  
  All sensitive values (GitHub App key, webhook secret) are stored in SSM and encrypted with a KMS key. By default, the module creates a KMS key; you can supply your own via `kms_key_arn`.  
- **Permissions Boundary**  
  Optionally attach a permissions boundary to limit what IAM roles can do. Set `role_permissions_boundary` to an ARN of your chosen policy.

---

## 📚 Examples Directory

Explore these examples to see common setups:

- **Default**: Basic spot-based runners at repo level.  
- **ARM64**: Use AWS Graviton2 instances by setting `runner_architecture = "arm64"` and custom `instance_types`.  
- **Ubuntu**: Swap the AMI filter to Ubuntu Linux for runner hosts.  
- **Windows**: Windows Server runners with PowerShell support.  
- **Ephemeral**: Single-use runners that spin up and tear down per workflow.  
- **Prebuilt Images**: Use a custom AMI (with Docker+runners preinstalled) for faster cold starts.  
- **Permissions Boundary**: Enforce tighter IAM boundaries on runner-related roles.

---

## 🐞 Debugging Tips

- **Webhook Troubleshooting**  
  - In GitHub, go to your App → Webhooks → Recent Deliveries to confirm events are firing.  
  - In AWS CloudWatch Logs, inspect the **`webhook-lambda`** log group for signature validation errors.  
- **Scale-Up Lambda**  
  - Check **`scale-up-lambda`** logs to see why a new spot instance was (or wasn’t) requested.  
  - Ensure your SQS queue isn’t throttling—check queue depth and message age.  
- **Runner Boot Logs**  
  - If you enabled `enable_ssm_on_runners = true`, connect via Session Manager and inspect `/var/log/user-data.log`.  
  - CloudWatch logs (under `<prefix>/runners` log group) show runner registration and job execution details.  

---

## 🔒 Security Considerations

- **Minimal Permissions**  
  The module’s Lambda functions only need rights to manage SQS, EC2 Spot Fleets, and SSM. Runners themselves request tokens to register with GitHub, but cannot create new tokens (no elevated GitHub permissions).  
- **Hardened AMIs**  
  By default, runners use public AMIs furnished by AWS. For maximum security, build a custom hardened AMI with only the packages you trust, then supply it via `ami_id_ssm_parameter_name`.  
- **Network Security Groups**  
  Runners live in private subnets. Only the scale-down Lambda and SSM (if enabled) can access them, preventing direct public exposure.

---

## 📈 Monitoring & Metrics

- **CloudWatch Alarms**  
  Set alarms on high queue depth (SQS) to ensure your scale-up Lambda is keeping pace.  
- **Runner Utilization**  
  Use CloudWatch metrics for EC2 instance count vs. idle time to fine-tune `minimum_running_time_in_minutes`.  
- **Spot Interruption Rate**  
  Monitor spot instance termination events to adjust your `instance_types` to more reliable families.

---

## 🎉 Moving Forward

1. **Refine Instance Families**  
   Continuously analyze spot prices and instance availability. Update your `instance_types` list to improve reliability and cut costs.  
2. **Automate Provisioner**  
   Store your entire configuration in version control (Terraform). Use CI/CD pipelines to apply changes when you test new runners or update permissions.  
3. **Advanced Pooling**  
   If you see intermittent “cold spot” gaps, enable a small `pool_config` to keep a handful of runners warmed up during peak hours.  
4. **Graceful Spot Handling**  
   Implement PodDisruptionBudgets or graceful shutdown hooks so your workflow jobs can tolerate spot termination signals.

---

> **Empower your CI/CD pipeline with flexible, cost-effective self-hosted runners—spinning up spot instances on demand and shutting them down when idle.**  
> By Anas Ahmed