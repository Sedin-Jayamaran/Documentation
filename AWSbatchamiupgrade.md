# AWS Batch Compute Environment AMI Migration Plan

This plan details how to migrate the existing AWS Batch compute environment `getting-started-wizard-compute-env` from Amazon Linux 2 (`ECS_AL2`) to Amazon Linux 2023 (`ECS_AL2023`) in the `ap-southeast-1` region.

Since there are **no active jobs running** and no ECS instances currently scaled up, we can perform a direct swap immediately with zero risk.

---

## User Review Required

> [!IMPORTANT]
> **Subnets and Security Groups:** The networking configuration (Subnets and Security Groups) is not visible in the provided screenshot (located under the **Networking** tab of the old environment). You will need to copy these same network settings when creating the new environment.

---

## Proposed Changes

### Replicating and Upgrading the Environment

We will create a new compute environment `getting-started-wizard-compute-env-v2` replicating the configuration of `getting-started-wizard-compute-env` but upgrading the AMI settings.

#### Old Configuration vs. New Configuration

| Configuration Parameter | Old Environment | New Environment (`v2`) |
| :--- | :--- | :--- |
| **Compute Environment Name** | `getting-started-wizard-compute-env` | `getting-started-wizard-compute-env-v2` |
| **Image Type** | `ECS_AL2` | **`ECS_AL2023`** (Upgraded) |
| **Allocation Strategy** | `BEST_FIT` (or None) | **`BEST_FIT_PROGRESSIVE`** (Enables future in-place updates) |
| **Min / Max vCPUs** | `0` / `2` | `0` / `2` |
| **Instance Type** | `m5.large` | `m5.large` |
| **Instance Profile** | `ecsInstanceRole` | `ecsInstanceRole` |
| **Service Role** | `AWSServiceRoleForBatch` | `AWSServiceRoleForBatch` |

---

## Action Plan

### Step 1: Create the New Compute Environment

#### Option A: Via AWS Management Console
1. Navigate to **AWS Batch** -> **Compute environments** -> **Create**.
2. Set the name to `getting-started-wizard-compute-env-v2`.
3. In **Instance configuration**:
   - Set **Minimum vCPUs** to `0`, **Maximum vCPUs** to `2`.
   - Set **Allowed instance types** to `m5.large`.
   - Set **Allocation strategy** to **Best fit progressive**.
4. In **AMI settings**:
   - Select **Amazon Linux 2023** (`ECS_AL2023`) as the Image Type.
5. In **Network configuration**:
   - Copy the exact VPC, Subnets, and Security Groups from the **Networking** tab of the old environment.
6. In **Permissions**:
   - Select the same EC2 instance role (`ecsInstanceRole`) and Service role (`AWSServiceRoleForBatch`).
7. Click **Create compute environment**.

#### Option B: Via AWS CLI
```bash
# 1. Fetch exact networking settings of the old environment
aws batch describe-compute-environments \
    --compute-environments getting-started-wizard-compute-env \
    --region ap-southeast-1 \
    --query "computeEnvironments[0].computeResources.{subnets:subnets,securityGroupIds:securityGroupIds}"

# 2. Create the new compute environment (replace SUBNETS and SECURITY_GROUPS with output from step 1)
aws batch create-compute-environment \
    --compute-environment-name getting-started-wizard-compute-env-v2 \
    --type MANAGED \
    --state ENABLED \
    --compute-resources '{
        "type": "EC2",
        "minvCpus": 0,
        "maxvCpus": 2,
        "desiredvCpus": 0,
        "instanceTypes": ["m5.large"],
        "subnets": ["subnet-xxxxxx", "subnet-yyyyyy"],
        "securityGroupIds": ["sg-zzzzzz"],
        "instanceRole": "ecsInstanceRole",
        "allocationStrategy": "BEST_FIT_PROGRESSIVE",
        "ec2Configuration": [
            {
                "imageType": "ECS_AL2023"
            }
        ]
    }' \
    --service-role arn:aws:iam::463949419568:role/aws-service-role/batch.amazonaws.com/AWSServiceRoleForBatch \
    --region ap-southeast-1
```

---

### Step 2: Swap Compute Environments in the Job Queue

Once the new environment is `VALID` and `ENABLED`:

1. Go to **AWS Batch** -> **Job queues** -> Select your active job queue -> click **Edit**.
2. In the **Compute environments** section:
   - Add `getting-started-wizard-compute-env-v2` to the queue.
   - Detach/remove the old `getting-started-wizard-compute-env` from the queue.
3. Save changes.

---

### Step 3: Delete the Old Environment

Since no jobs are running:
1. Go to **Compute environments** -> Select `getting-started-wizard-compute-env` -> Click **Disable**.
2. Once disabled, select it again and click **Delete**.


---

## Verification Plan

1. Verify that the new compute environment status is `VALID`.
2. Submit a test job to the queue and confirm that it launches successfully on an instance in `getting-started-wizard-compute-env-v2`.
3. Verify that the instance running the job is using Amazon Linux 2023 (AL2023).
