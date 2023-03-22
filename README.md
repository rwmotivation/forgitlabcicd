# forgitlabcicd
resources to creare gitlab ci/cd pipeline on gitlab for ecs, postgres and django 
[git-cheat-sheet.pdf](https://github.com/rwmotivation/forgitlabcicd/files/11036748/git-cheat-sheet.pdf)
Gitlab CI with ECS
The flow is pretty much like this:
Push code changes to repository.
Gitlab create new pipeline and notify the Master Runner. (AWS EC2).
Master Runner launch new Gitlab Runner server or use the existing one. (AWS EC2).
Gitlab Runner run test.
Test done and start Docker build process.
Build process done, store Docker image to AWS Elastic Container Registry (ECR) and start deployment process.

    Deployment process will replace old container that exist on AWS Elastic Container Service (ECS) and replace with the new one.

The process above takes 15–20 minutes for most of our services.
So, how we do that?
Infrastructure
We are using Gitlab services for our code repository and it come with Gitlab Runner feature on it. But the feature will not work unless you have the runner itself. Usually if you using Gitlab you can use either their own shared runner or our own runner.
For us, we are using our own runner because we want better performance for our CI process since we can configure the runner resources (CPU and memory) as we like. Also we want to make sure secret credentials stays on our AWS environment.
We are using 
 on our Gitlab runner which you can see more about it here. In simple terms, Autoscale require one master server that have a task to launch new runner if there is a new process need to be handled. In our Autoscale configuration, we choose Docker as our runner options.
Autoscale also allow us to be more efficient in terms of cost, since it only launch new runner server only when it needed. We also set the timeout of the runner really short around 5 minutes which means if there is no more jobs after 5 minutes idle the server will shutdown automatically.
This configuration has a drawback though. It will increase cold-start time when there is a job available and there is no active runner server. However, We have very small engineering team and we currently don’t mind with this because our deployment rate is still small. Since we implemented this architecture 2 months ago, we only have made around 700 deployment process (~11 deployments/day).
Next is container registry. After Docker build process done, it will produce an image. We push the image to  (ECR) then deploy it to  (ECS). The setup of ECR and ECS is pretty simple since they both are AWS services. On ECS we want simplify things, that’s why we choose 
 instead EC2 as our container cluster. It’s higher cost but lower maintenance work required.
gitlab-ci.yml
In Gitlab, you need configuration called gitlab-ci.yml This file is use by Gitlab to configure each step of the Continuous Integration (CI) process. In our setup we have 3 step: test, build, and deploy. So we have configuration like this:
stages:
 - test
 - build
 - deploy
#=========
# Test
#=========
test:
  stage: test
  image: gitlab.example.com/test_image
  script:
    - ./test.sh
#=========
# Build
#=========
build:
  stage: build
  image: backstreetacademy/docker-aws
  services: 
    - docker:dind
  script: 
    - ./build.sh
#=========
# Deploy
#=========
deploy:
  stage: deploy
  image: backstreetacademy/docker-aws
  script: 
    - ecs deploy ClusterName ServiceName --timeout=1800
Code above is the simple version of our gitlab-ci.yml and the file usually have more configuration which I will not explain deeply since you can read the 
 by yourself. In our code-base usually we have 200–300 LoC of gitlab-ci.yml.
Let’s break it down.
Test Process
Test process is covering our unit test and integration test. Currently we don’t have E2E test but I really want to have it in the future.
#=========
# Test
#=========
test:
  stage: test
  image: gitlab.example.com/test_image
  script:
    - ./test.sh
As you can see in the code above, we have specific Docker image that has built as test runner. We store it on Gitlab Registry since it contains some of our code that should not available publicly. We also have test script that will run the test once the container is launched.
Build Process
Build process cover the process building our code into proper Docker image.
#=========
# Build
#=========
build:
  stage: build
  image: backstreetacademy/docker-aws
  services: 
    - docker:dind
  script: 
    - ./build.sh
In our repository, we have Dockerfile that will help us generate the image. But in order to have docker inside the runner, we need activate Docker-in-Docker service. This service allow us to have docker binary available to build the image. Without this the command will not exists.
We are using image called . This image is specifically build for our needs to have build and deployment process in our CI infrastructure. It contains toolkit such as 
 and AWS command line.
In our build script, we have something like this:
#!/bin/sh
echo "Login to ECR Repository"
$(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
echo "Preparation task"
echo "Build Docker Image"
docker build -t sample-image .
echo "Push to ECR Repository"
docker tag sample-image:latest $AWS_ECR_REPOSITORY:sample-tag
docker push $AWS_ECR_REPOSITORY:sample-tag
In order to have this script running we need to have environment variable implemented on our CI/CD configuration on Gitlab repository which we set to be available only on protected branches.
It means we only trigger build process for our feature, staging, and production branch. So, usually we have more than two build process configured in our gitlab-ci.yml. Once the build process completed and image is stored in ECR, we continue to deployment process.
Deploy Process
Deploy process cover the deployment of the image that has been generated in build process. The task will launch new image, and shutdown the old image.
#=========
# Deploy
#=========
deploy:
  stage: deploy
  image: backstreetacademy/docker-aws
  script: 
    - ecs deploy ClusterName ServiceName --timeout=1800
In this process, we heavily depends on , a python command line tool that make us easier managing ECS service (Kudos to  for creating awesome tool like this. 🍻)
