# Jenkins Scripting
## Environment :
```
environment {
    AWS_REGION      = 'ap-south-1'
    ECR_REGISTRY    = '156916773321.dkr.ecr.ap-south-1.amazonaws.com'
    ECR_REPO        = 'jayamaran/sample-for-ecs'
    IMAGE_TAG       = 'latest'
    GIT_CREDENTIALS = '10962414-951f-44c5-921e-9e1afffe0993'
  }
```  
### Explanation :

I am creating a Jenkins Credential in Jenkin UI & then , taking the credential ID ie , ```10962414-951f-44c5-921e-9e1afffe0993``` and directing it into a variable ```GIT_CREDENTIALS``` .
## Code Checkout Stage :
### Code Sample :
```stage('Checkout Code') {
      steps {
        checkout([
          $class: 'GitSCM',
          branches: [[name: '*/main']],
          userRemoteConfigs: [[
            url: 'https://github.com/Sedin-Jayamaran/Buggy-app',
            credentialsId: "${env.GIT_CREDENTIALS}"
          ]]
        ])
      }
    }
```
### Explanation : 
#### 1. Checkout :
#### This invokes the checkout step, which is a built-in Jenkins method used to check out source code from a version control system.It supports many SCMs (Git, SVN, Mercurial, etc.)You're passing a configuration block as a parameter.
#### 2 . What is credentailsID ? 
#### It is a keyword ( parameter ) of Jenkins . It help in authentication of Github with Jenkins . This is being directed with a env variable ```GIT_CREDENTIALS``` as ```credentialsId: "${env.GIT_CREDENTIALS}```
### Docker Login to ECR:
```stage('Docker Login to ECR') {
      steps {
        withCredentials([
          string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY'),
          string(credentialsId: 'aws-session-toke', variable: 'AWS_SESSION_TOKEN')
        ]) {
          sh '''
            export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
            export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
            export AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN

            aws ecr get-login-password --region ${AWS_REGION} \
              | docker login --username AWS --password-stdin ${ECR_REGISTRY}
          '''
        }
      }
    }
```
### Explanation :
#### What does this ```withCredentials([ ... ]) ``` do ?
#### This is a secure Jenkins wrapper that , Temporarily exposes sensitive credentials to your pipeline. Automatically hides them from logs and removes them after the block ends.
```
| Property                  | Description                                              
| 🔐 **Stored Securely**    | Encrypted at rest in Jenkins                             
| 👁️ **Hidden by Default**  | Not exposed to the pipeline or logs unless you choose to 
| 🔐 **Access Controlled**  | Only exposed through steps like `withCredentials`        
```
### Docker Build :
```
stage('Build Docker Image') {
      steps {
        sh '''
          docker build --no-cache -f Dockerfile.app -t $ECR_REPO:$IMAGE_TAG .
          docker tag $ECR_REPO:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
        '''
      }
    }
```
#### What does it do `docker build --no-cache -f Dockerfile.app -t $ECR_REPO:$IMAGE_TAG .` ?

##### `--no-cache` Force Docker to ignore cache and rebuild everything fresh. Useful to avoid stale builds.
##### `-f Dockerfile.app` Use a specific Dockerfile named Dockerfile.app.
##### `-t $ECR_REPO:$IMAGE_TAG` Tag the image. $ECR_REPO is your local image name, $IMAGE_TAG is the tag (like latest or build-42).
##### `.` Use current directory (.) as the build context (i.e., where the code + Dockerfile lives).

#### What does `docker tag $ECR_REPO:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG` do ?
#### This command re-tags the image so that it's ready to be pushed to Amazon ECR.
## Note : 
### We tag the image twice because:
#### 1.The first tag is for local naming and reuse.
#### 2.The second tag is required to push to Amazon ECR, which needs the full ECR registry URL .

### Pushing Image to ECR :
```
stage('Push Docker Image to ECR') {
      steps {
        sh '''
          docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
        '''
      }
    }
```
#### Explanation :
#### This stage pushes the previously built and ECR-tagged Docker image to Amazon Elastic Container Registry (ECR) so it can be pulled by ECS (or other AWS services) for deployment.

#### What does `docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG` do ?
#### This is the actual Docker command that pushes your tagged image to Amazon ECR.
 
### Delpoying in ECS :
```
stage('Deploy to ECS') {
  steps {
    withCredentials([
      string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
      string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY'),
      string(credentialsId: 'aws-session-toke', variable: 'AWS_SESSION_TOKEN')
    ]) {
      sh '''
        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
        export AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN

        aws ecs update-service \
          --cluster Jai-Manual-Cluster \
          --service jai-ecs-web-service-26129xjx \
          --force-new-deployment \
          --region $AWS_REGION
      '''
    }
  }
}
```
#### What This Stage Does ?
#### This stage triggers a new deployment in your ECS cluster using the most recently pushed Docker image in Amazon ECR.
#### Jenkins does not edit the task definition directly — instead, it forces ECS to pull the new image version by restarting the service.
 