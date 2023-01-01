+++
title = "Building and pushing docker images to AWS ECR using Gitlab CI"
date = "2023-01-01T21:26:31+05:30"
author = "Dhamith Hewamullage"
authorTwitter = "" #do not include @
cover = ""
tags = ["cicd", "automation", "docker", "AWS", "ECR", "gitlabci"]
keywords = ["cicd", "automation", "docker", "AWS", "ECR", "gitlabci", "Building and pushing docker images to AWS ECR using Gitlab CI"]
description = "How to build and push docker images from Gitlab CI pipelines to AWS ECR"
showFullContent = false
readingTime = true
hideComments = true
+++

Gitlab CI lets you quickly set up build automation pipelines to test, verify, build, and deploy your applications. In this guide, I'm showing how to use Gitlab CI pipelines to build and push a docker image of your application to AWS Elastic Container Registry.

First, a user should be created in AWS (if you haven't already) with permission to push to AWS ECR private/public repositories. You can read up on required policies for users from here: [private](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-policy-examples.html)/[public](https://docs.aws.amazon.com/AmazonECR/latest/public/public-repository-policy-examples.html). Once the user is created, create the ECR repository in AWS. Once that is done, you can navigate to the repository and click on the `View push commands` button to see the required commands to push the docker images to that repository. Once the repository is also created, a few variables that must to be set in Gitlab. Navigate to the repository in Gitlab and go to Settings -> CI/CD, and expand the **Variables** section. Add the following variables and mark them as `Protected`: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`, `IMAGE_REGISTRY`. The `IMAGE_REGISTRY` variable is the path to the ECR repository. 

![](/variables.png)

Then create the `.gitlab-ci` file as required by the codebase. I have created a simple pipeline for my use case with three stages: `Test`, `Build`, and `Release`. This can be simple or complex as needed. But since in this guide, we are only interested in creating the docker image and pushing it to the AWS ECR, it is going to be simpler. In my pipeline stages, if something is committed to the repository, it will be tested first; in the build stage, the application will be compiled and packed as specified in `Makefile`, and in the release stage, if the branch is `Main`, then the docker image will be created and pushed to AWC ECR. 

Here is the release stage, which deals, with the docker image creation and publishing to ECR.

```yaml
build_image:
  image: docker:stable
  stage: release
  dependencies: 
    - build
  services:
    - docker:dind
  before_script:
    - apk add --no-cache python3 py3-pip
    - pip3 install --upgrade pip
    - pip3 install --no-cache-dir awscli 
    - aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
    - aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
    - aws configure set region $AWS_DEFAULT_REGION
    - aws ecr-public get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $IMAGE_REGISTRY
  script:
    - tar -xvf ./release/client_linux_x86_64.tar.gz 
    - cd client_linux_x86_64
    - docker build -t symon_client:latest .
    - docker tag symon_client:latest $IMAGE_REGISTRY/symon_client:latest
    - docker push $IMAGE_REGISTRY/symon_client:latest
  only:
    refs:
      - main
```

Let's see a breakdown of that stage. I am using the stable docker image in this stage with `docker:dind` service. This stage depends on the build stage, as artifacts are required in that stage. This can be changed as per the codebase.

```yaml
image: docker:stable
stage: release
dependencies: 
    - build
services:
    - docker:dind
```

As specified in the `before_script`, I am setting up a few things, before executing the main script. First, I'm installing `python3` and `py3-pip` packages to install and use the `aws-cli`. Once AWS CLI is installed, it is configured with the user I have created in AWS. I'm using the variables set up earlier in CI/CD settings. 

```yaml
before_script:
    - apk add --no-cache python3 py3-pip
    - pip3 install --upgrade pip
    - pip3 install --no-cache-dir awscli 
    - aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
    - aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
    - aws configure set region $AWS_DEFAULT_REGION
    - aws ecr-public get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $IMAGE_REGISTRY
```

The next step is to extract the token required to push the image to the ECR repository. This command can be changed depending on the private/public status of the ECR repository. It is better to copy and modify that command from the `View push commands` section in the ECR repository. Once the token has been retrieved, it is used to log in to the ECR repository through docker. Which concludes the `before_script` section. 

In the actual `script` section, what must be done first is to navigate to where the `Dockerfile` is located. In this example, I'm using the built archive in artifacts to extract and navigate to the `Dockerfile` location. Again this should be changed as required. The next step is actually to build the image. In this case, I'm directly using the command specified in the AWC ECR `View push commands` section, but this should be changed as required. Then the `IMAGE_REGISTRY` variable is tagged to the image before pushing it to the repository.

```yaml
script:
    - tar -xvf ./release/client_linux_x86_64.tar.gz 
    - cd client_linux_x86_64
    - docker build -t symon_client:latest .
    - docker tag symon_client:latest $IMAGE_REGISTRY/symon_client:latest
    - docker push $IMAGE_REGISTRY/symon_client:latest
```

Finally, I have specified this stage to be run only on changes in the `Main` branch. 

```yaml
only:
    refs:
      - main
```

This is the full `.gitlab-ci` file:

```yaml
image: golang:1.18

stages:
  - test
  - build
  - release

test:
  stage: test
  script:
    - go fmt $(go list ./...)
    - go vet $(go list ./...)
    - go test -race $(go list ./...)

build:
  stage: build
  script:
    - make pack-all
  artifacts:
    paths:
      - ./release/collector_linux_x86_64.tar.gz
      - ./release/agent_linux_x86_64.tar.gz
      - ./release/alertprocessor_linux_x86_64.tar.gz
      - ./release/client_linux_x86_64.tar.gz

build_image:
  image: docker:stable
  stage: release
  dependencies: 
    - build
  services:
    - docker:dind
  before_script:
    - apk add --no-cache python3 py3-pip
    - pip3 install --upgrade pip
    - pip3 install --no-cache-dir awscli 
    - aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
    - aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
    - aws configure set region $AWS_DEFAULT_REGION
    - aws ecr-public get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $IMAGE_REGISTRY
  script:
    - tar -xvf ./release/client_linux_x86_64.tar.gz 
    - cd client_linux_x86_64
    - docker build -t symon_client:latest .
    - docker tag symon_client:latest $IMAGE_REGISTRY/symon_client:latest
    - docker push $IMAGE_REGISTRY/symon_client:latest
  only:
    refs:
      - main
```