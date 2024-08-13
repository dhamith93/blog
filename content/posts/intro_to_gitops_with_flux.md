+++
title = "Introduction to GitOps with GitLab and Flux"
date = "2024-08-13T09:53:31+05:30"
author = "Dhamith Hewamullage"
authorTwitter = "" #do not include @
cover = ""
tags = ["gitops", "gitlab", "flux", "helm", "kubernetes"]
keywords = ["gitops", "gitlab", "flux", "helm", "kubernetes"]
description = "Introduction to GitOps with GitLab and Flux"
showFullContent = false
readingTime = true
hideComments = false
+++

## About 

This is a simple introduction to GitOps using **GitLab**, **Flux**, and **Helm charts**. Objective of this is to deploy a small microservice application written in Go on to a Kubernetes cluster with minimal manual interactions. There are few notes to consider; the pipelines, triggers, conditions, etc. are kept in a simplest form to keep things easier to understand and replicated. There are many strategies to test, build, and deploy applications as well as create deployment pipelines. I believe readers can customize, improve, and make it more secure to suite their needs. 

> Note: Source code used in this is used for learning purposes only. If you are using them for production work, please consider your project infrastructure and security needs.

I used GitLab Community Edition (v17.0.1) with a local Kubernetes cluster (v1.28.10) and Flux (v2.3.0) deployed on Proxmox VMs. 

> **Full repo can be found here:** https://github.com/dhamith93/gitops-sample

### Repo structure

```
├── app
│   ├── api
│   │   └── maths
│   ├── client
│   └── internal
├── charts
│   ├── client
│   └── maths-api
└── clusters
    └── playground
        ├── calculator
        └── flux-system
```
This is the structure of the repo. `app` directory contains the application source code. `charts` directory contains the helm charts for two services. `clusters` directory contains Kubernetes and flux resources to deploy the application on to a cluster. 

### Application

Application is a tiny calculator which takes an expression from the user and returns the result. `client` service handles the frontend and `maths-api` takes the expression, resolves it and returns an answer. `maths-api` also have few unit tests configured.  Both services have Docker files to create an Alpine linux based docker image. 

Client service and the maths-api service both uses env variables to get the port number to listen to. The client service also get the endpoint to the maths-api through a env variable. 

Version number for both `client` service and `maths-api` are stored in `.version` file in application source root directory. Changes to this file will be used to trigger test, build, and deploy pipeline jobs. 

## Building and releasing the application using GitLab-CI

```yml
stages:
  - test
  - build
  - release

include:
  - local: '$CI_PROJECT_DIR/app/client/.gitlab-ci.yml'
  - local: '$CI_PROJECT_DIR/app/api/maths/.gitlab-ci.yml'
  - local: '$CI_PROJECT_DIR/charts/client/.gitlab-ci.yml'
  - local: '$CI_PROJECT_DIR/charts/maths-api/.gitlab-ci.yml'
```

This is the `.gitlab-ci.yml` file in the root directory of the repo. Included four CI configuration files need to be created for the application and the helm charts. Let's start with the application building. 

#### CI variables

| Variable              |                                                  |
| --------------------- | ------------------------------------------------ |
| `GL_PAT`              | Password or access token for the docker registry |
| `IMAGE_REGISTRY`      | Docker registry host                             |
| `IMAGE_REGISTRY_USER` | Docker registry username                         |

### Client service

> Complete CI file: https://github.com/dhamith93/gitops-sample/blob/master/app/client/.gitlab-ci.yml

#### Build stage

Go language version 1.22 docker image is used to build the client service. Build stage will compile the source code and retain the executable with the `.version` file as an artifact. There is a filter setup to trigger the build pipeline. The pipeline only triggers when the `.version` file is modified and committed.

> **Note:** Here I used the `.version` file for the simplicity. This can be set up using many ways as per the language/framework the application written in. Using git tags is another alternative. I picked the `.version` file because, this is a monorepo and both services will have separate version numbers.

```yaml
build-client:
  image: golang:1.22.4-alpine3.20
  stage: build
  artifacts:
    paths:
    - "$CI_PROJECT_DIR/app/client/client"
    - "$CI_PROJECT_DIR/app/client/.version"
  script:
  - cd "$CI_PROJECT_DIR/app/client/"
  - go build -o client
  only:
    changes:
    - app/client/.version
```

#### Release stage

The release stage is depended on the build stage. This stage uses a docker image with docker-in-docker service to build the docker image for the client service using the included dockerfile. 

```yaml
FROM alpine:3.20
ADD client /client
ADD frontend /frontend
CMD ./client
```

The dockerfile is very simple one. It uses Alpine linux image and copies the executable and the frontend directory, which contains html and javascript source files. 

Just like the build stage, the release stage also triggers when there is a change to the `.version` file. All CI variables stated above are needed for this stage to log in to the container repo and push the image. `.version` file is used to get the version number to add it as a tag to the image.

```yaml
release-client-docker-img:
  image: docker:27.1.1
  stage: release
  dependencies: 
  - build-client
  services:
  - name: docker:27.1.1-dind
    alias: docker
  variables:
    DOCKER_HOST: tcp://docker:2375/
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
  script:
  - echo $GL_PAT | docker login $IMAGE_REGISTRY -u $IMAGE_REGISTRY_USER --password-stdin
  - VERSION=`cat $CI_PROJECT_DIR/app/client/.version`
  - cd "$CI_PROJECT_DIR/app/client/"
  - docker build -t client:latest -t $IMAGE_REGISTRY_PATH/client_v:$VERSION -t $IMAGE_REGISTRY_PATH/client_v:latest .
  - docker image push $IMAGE_REGISTRY_PATH/client_v --all-tags
  only:
    changes:
    - app/client/.version
```

### maths-api service

CI pipelines and the dockerfile for building and releasing the maths-api service is similar to the client service. Only addition is the `test` stage. Which uses golang docker image to run the unit tests under the internal maths module. Build stage is depended on the test stage and the release stage is depended on the build stage. Which means if any unit tests get failed, the whole pipeline will stop. 

```yaml
FROM alpine:3.20
ADD maths_api /maths_api
CMD ./maths_api
```

```yaml
test-maths-api:
  image: golang:1.22.4-alpine3.20
  stage: test
  script: 
    - cd "$CI_PROJECT_DIR/app/internal/maths"
    - go test .
  only:
    changes:
      - app/api/maths/.version

build-maths-api:
  image: golang:1.22.4-alpine3.20
  stage: build
  artifacts:
    paths:
      - "$CI_PROJECT_DIR/app/api/maths/maths_api"
      - "$CI_PROJECT_DIR/app/api/maths/.version"
  dependencies:
    - test-maths-api
  script: 
    - cd "$CI_PROJECT_DIR/app/api/maths"
    - go build -o maths_api
  only:
    changes:
      - app/api/maths/.version

release-maths-api-docker-img:
  image: docker:27.1.1
  stage: release
  dependencies: 
  - build-maths-api
  services:
  - name: docker:27.1.1-dind
    alias: docker
  variables:
    DOCKER_HOST: tcp://docker:2375/
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
  script:
  - echo $GL_PAT | docker login $IMAGE_REGISTRY -u $IMAGE_REGISTRY_USER --password-stdin
  - VERSION=`cat $CI_PROJECT_DIR/app/api/maths/.version`
  - cd "$CI_PROJECT_DIR/app/api/maths"
  - docker build -t maths_api:latest -t $IMAGE_REGISTRY_PATH/maths_api_v:$VERSION -t $IMAGE_REGISTRY_PATH/maths_api_v:latest .
  - docker push $IMAGE_REGISTRY_PATH/maths_api_v --all-tags
  only:
    changes:
    - app/api/maths/.version
```

Once all the pipelines for both services are completed, docker images will be pushed to the container image registry.

## Setting up Flux

Configuring Flux was simple enough. I followed the official docs for bootstrapping Flux for GitLab (https://fluxcd.io/flux/installation/bootstrap/gitlab/). As mentioned in the docs, to set up Flux, Kubernetes cluster admin rights and full access to the GitLab project are needed. In my repo, `clusters/playground` is the path for the `flux-system` manifests.

## Helm charts

> Link to Helm chart source: https://github.com/dhamith93/gitops-sample/tree/master/charts

Helm charts for client and maths-api service are also similar. Only difference being the environmental variables specifying the port numbers to listen to, and the maths-api endpoint specified for the client service. `values.yaml` file specifies the app name, port, replica count, env vars, and image name with the tag. 

### Building the Kubernetes manifest file

### CI vars

| Variable         |                                                                  |
| ---------------- | ---------------------------------------------------------------- |
| `CI_KNOWN_HOSTS` | Known host entry for GitLab host taken from `~/.ssh/known_hosts` |
| `SSH_PUSH_KEY`   | Private key for the deploy key RSA key pair                      |


There are build and release stages implemented for the helm charts in `/charts/<service>/.gitlab-ci.yml` files. The build stage creates the Kubernetes manifest file for the service using the `helm template` command. The command will use the values specified in the `values.yaml` file in the helm chart and output the filled up template. The output then redirected to the file `/clusters/playground/calculator/<service>/release.yml`. The path for the `release.yml` file is already created on the repo before hand to avoid any errors during the build. This manifest file will be used by Flux to ultimately deploy the application to the Kubernetes cluster. For this stage `alpine/helm` image is used.

Even though the build stage creates the `release.yml` manifest to be deployed, it is not yet readable to Flux. The release stage which is depended on the build stage will take the `release.yml` file then it will commit it to the repo. For this stage, a GitLab project **deploy key** with write access is used to gain write access to the repo. `alpine/git` image is used with the release stage. 

Both build and release stages are triggered by any changes to `/charts/<service>/` directory. Again since the both services are very much similar, the build and release stages for both services are also similar. Only difference being the paths. 

```yaml
build-maths-api-helm-chart:
  image: 
    name: alpine/helm:3.15.1
    entrypoint: [""] 
  stage: build
  artifacts:
    paths:
      - "$CI_PROJECT_DIR/clusters/playground/calculator/maths-api/release.yml"
  script:
    - helm template maths-api $CI_PROJECT_DIR/charts/maths-api > $CI_PROJECT_DIR/clusters/playground/calculator/maths-api/release.yml
  only:
    changes:
      - charts/maths-api/**/*

release-maths-api-helm-chart:
  image: 
    name: alpine/git:v2.45.1
    entrypoint: [""]
  stage: release
  dependencies:
    - build-maths-api-helm-chart
  before_script:
    - mkdir ~/.ssh/
    - echo "${CI_KNOWN_HOSTS}" > ~/.ssh/known_hosts
    - echo "${SSH_PUSH_KEY}" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - git config user.email "ci@gitlab.local"
    - git config user.name "CI"
    - git remote remove ssh_origin || true
    - git remote add ssh_origin "git@$CI_GIT_HOST:$CI_PROJECT_PATH.git"
  script:
    - git add $CI_PROJECT_DIR/clusters/playground/calculator/maths-api/release.yml
    - git commit -m "Adding maths-api heml template output"
    - git push ssh_origin HEAD:$CI_COMMIT_REF_NAME
  only:
    changes:
      - charts/maths-api/**/*
```

## Next steps

Since this is done in an existing local environment, there is no IaC to provision any infrastructure. But tools like Terraform or Ansible can be configured in the same repo with similar CI pipelines to automate/autoscale infrastructure provisioning as needed.  

## Conclusion

Now with all the CI stages are configured, once the source code changes for either service and `.version` file is updated with the new version, build stage and docker image release stage will be triggered. During these stages, unit test will be running, then the source code will be compiled, and finally the docker image will be built with the version tag and pushed in to the container repo. 

Then when for example in the Helm chart's `values.yaml` file is modified with the new image tag, another CI pipeline with build and release stage will be triggered. Which will create a single manifest file for Flux and commit it to the same repo. 

With Flux running, it will see the new manifest and it will either install or upgrade the manifest into the Kubernetes cluster. 

This is a very simplistic take on GitOps for understanding the process and steps behind it. There are many ways to improve this. For example, you can set this up to be deployed to `dev`, `qa`, and `prod` environments. You can have this configured to be run only when MR is approved to the `main/master` branch. You can also add or remove extra CI stages as needed. Specially for provisioning infrastructure. 

I believe GitLab with Flux is very flexible and straightforward way to get into GitOps.
