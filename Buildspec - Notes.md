
# 🚧 Pre-Build Phase — `buildspec.yml`

```yaml
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
      - COMMIT_ID=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c1-7)
      - IMAGE_TAG=$COMMIT_ID
      - echo "IMAGE_TAG=$IMAGE_TAG" > $CODEBUILD_SRC_DIR/image_tag.env
```

---

## 🔍 Step-by-Step Explanation

### 🔐 Authenticate Docker with ECR

```sh
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

* This logs Docker into your **Amazon ECR** (Elastic Container Registry) using a temporary token.
* It uses environment variables:

  * `$AWS_REGION` — the AWS region (e.g., `ap-south-1`)
  * `$AWS_ACCOUNT_ID` — your AWS account ID, used to access your ECR registry
* This login step is required for Docker to push/pull images from your private ECR.

---

### 🔎 Extract Git Commit ID (first 7 characters)

```sh
COMMIT_ID=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c1-7)
```

* `$CODEBUILD_RESOLVED_SOURCE_VERSION` contains the **full Git commit SHA** that triggered the build.
* `cut -c1-7` trims it to the first 7 characters for a **short commit hash**.
* Example:

  * Full SHA: `fba5f6a8bcd54ef2d6f62348a4a3b11fdca6e51e`
  * Short SHA: `fba5f6a`

---

### 🏷️ Set the Docker image tag

```sh
IMAGE_TAG=$COMMIT_ID
```

* This stores the short Git commit ID into a variable named `IMAGE_TAG`.
* This tag will be used when building and pushing the Docker image.
* Benefit: it uniquely identifies the image version with the commit it came from.

---

### 📝 Store image tag for later use

```sh
echo "IMAGE_TAG=$IMAGE_TAG" > $CODEBUILD_SRC_DIR/image_tag.env
```

* This writes the image tag into a temporary file (`image_tag.env`).
* The file will later be `source`d in other phases (`build` and `post_build`), ensuring consistent tagging throughout the process.

---

Here's the **Markdown** version for the `build` phase of your `buildspec.yml`, explained step-by-step in the same style:

---

# 🏗️ Build Phase — `buildspec.yml`

```yaml
build:
  commands:
    - source $CODEBUILD_SRC_DIR/image_tag.env
    - echo Building the Docker image
    - docker build -f Dockerfile.app --build-arg BASE_IMAGE=$BASE_IMAGE -t $ECR_REPO:$IMAGE_TAG .
```

---

## 🔍 Step-by-Step Explanation

### 🔄 Load previously stored image tag

```sh
source $CODEBUILD_SRC_DIR/image_tag.env
```

* Reads the file created in the `pre_build` phase: `image_tag.env`.
* This makes the `IMAGE_TAG` variable available again in this phase.
* Ensures consistency — the same image tag is used throughout the pipeline.

---

### 🧱 Print status message

```sh
echo Building the Docker image
```

* Outputs a log line to indicate the Docker image build is starting.
* Useful for debugging and understanding pipeline progress in the CodeBuild logs.

---

### 🐳 Build the Docker image

```sh
docker build -f Dockerfile.app --build-arg BASE_IMAGE=$BASE_IMAGE -t $ECR_REPO:$IMAGE_TAG .
```

* `-f Dockerfile.app`: Specifies the custom Dockerfile (`Dockerfile.app`) to use.
* `--build-arg BASE_IMAGE=$BASE_IMAGE`: Passes a build argument (`BASE_IMAGE`) from Parameter Store.

  * Allows dynamic base image selection.
* `-t $ECR_REPO:$IMAGE_TAG`: Tags the Docker image with the ECR repository and the Git commit ID (from `IMAGE_TAG`).

  * Example image tag: `156916773321.dkr.ecr.ap-south-1.amazonaws.com/my-ecr-repo:fba5f6a`
* `.`: Uses the current directory as the build context.

---

---

Here is the full **Markdown-formatted explanation** of the `post_build` phase from your `buildspec.yml`:

---

# 🚀 Post-Build Phase — `buildspec.yml`

```yaml
post_build:
  commands:
    - source $CODEBUILD_SRC_DIR/image_tag.env
    - echo Pushing the Docker image to ECR
    - docker push $ECR_REPO:$IMAGE_TAG
    - echo Build and push complete.

    - echo Creating imagedefinitions.json
    - printf '[{"name":"buggy-web","imageUri":"%s"}]' "$ECR_REPO:$IMAGE_TAG" > $CODEBUILD_SRC_DIR/imagedefinitions.json

    # ===== Deployment section starts here =====
    - echo Installing jq
    - yum install -y jq

    - echo Fetching current ECS task definition
    - TASK_DEF=$(aws ecs describe-task-definition --task-definition jai-ecs-web)

    - echo Creating new task definition with updated image
    - |
      NEW_TASK_DEF=$(echo $TASK_DEF | jq --arg IMAGE "$ECR_REPO:$IMAGE_TAG" '
        .taskDefinition | {
          family: .family,
          containerDefinitions: (.containerDefinitions | map(if .name == "buggy-web" then .image = $IMAGE | . else . end)),
          requiresCompatibilities: .requiresCompatibilities,
          networkMode: .networkMode,
          cpu: .cpu,
          memory: .memory,
          executionRoleArn: .executionRoleArn,
          taskRoleArn: .taskRoleArn
        }')

    - echo "$NEW_TASK_DEF" > new-task-def.json
    - cat new-task-def.json

    - echo Registering new task definition
    - NEW_TASK_REVISION=$(aws ecs register-task-definition --cli-input-json file://new-task-def.json | jq -r '.taskDefinition.taskDefinitionArn')

    - echo Updating ECS service
    - aws ecs update-service --cluster Jai-Manual-Cluster --service jai-ecs-web-service-777 --task-definition $NEW_TASK_REVISION

    - echo ECS Deployment complete.
```

---

## 🔍 Step-by-Step Breakdown

### 🧾 Load image tag

```sh
source $CODEBUILD_SRC_DIR/image_tag.env
```

* Loads the `IMAGE_TAG` variable saved during the pre-build phase.

---

### 📤 Push Docker Image to ECR

```sh
docker push $ECR_REPO:$IMAGE_TAG
```

* Pushes the image (tagged with the commit ID) to Amazon ECR for use in ECS deployments.

---

### 📄 Create `imagedefinitions.json`

```sh
printf '[{"name":"buggy-web","imageUri":"%s"}]' "$ECR_REPO:$IMAGE_TAG" > $CODEBUILD_SRC_DIR/imagedefinitions.json
```

* This JSON file is used if you are deploying via CodeDeploy or using CodePipeline.
* It maps the ECS container name `buggy-web` to the new image URI.

---

### 🔧 Install `jq` (JSON processor)

```sh
yum install -y jq
```

* Required to parse and manipulate the ECS task definition JSON.

---

### 📦 Fetch Existing ECS Task Definition

```sh
TASK_DEF=$(aws ecs describe-task-definition --task-definition jai-ecs-web)
```

* Pulls the current task definition for the service.
* Used as a template to modify only the image URI.

---

### 🏗️ Generate New Task Definition (Updated Image Only)

```sh
NEW_TASK_DEF=$(echo $TASK_DEF | jq --arg IMAGE "$ECR_REPO:$IMAGE_TAG" '
  .taskDefinition | {
    family: .family,
    containerDefinitions: (.containerDefinitions | map(if .name == "buggy-web" then .image = $IMAGE | . else . end)),
    requiresCompatibilities: .requiresCompatibilities,
    networkMode: .networkMode,
    cpu: .cpu,
    memory: .memory,
    executionRoleArn: .executionRoleArn,
    taskRoleArn: .taskRoleArn
  }')
```

* Extracts the `taskDefinition` block and injects the **new Docker image** into the `buggy-web` container.
* Keeps all other fields identical (e.g., CPU, memory, IAM roles).

---

### 💾 Save and Print New Task Definition

```sh
echo "$NEW_TASK_DEF" > new-task-def.json
cat new-task-def.json
```

* Saves the modified definition to a file and prints it for logging/debugging.

---

### 📝 Register New Task Definition

```sh
NEW_TASK_REVISION=$(aws ecs register-task-definition --cli-input-json file://new-task-def.json | jq -r '.taskDefinition.taskDefinitionArn')
```

* Submits the new task definition to ECS.
* The response includes the new revision ARN (e.g., `arn:aws:ecs:region:account-id:task-definition/jai-ecs-web:52`).

---

### 🔄 Update ECS Service

```sh
aws ecs update-service --cluster Jai-Manual-Cluster --service jai-ecs-web-service-777 --task-definition $NEW_TASK_REVISION
```

* Tells ECS to **use the new revision** of the task definition for the service.
* Triggers deployment of the updated container image.

---

### ✅ Final Log Message

```sh
echo ECS Deployment complete.
```

* Signals the successful end of the deployment process.

---



