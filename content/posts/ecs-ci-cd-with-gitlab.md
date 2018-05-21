---
title: "CI / CD for ECS with gitlab pipelines"
date: 2017-10-01T21:40:29+03:00
draft: false
tags: ["ECS", "gitlab", "pipelines", "CI/CD"]
categories: ["Docker", "DevOps"]
---

Here is a short guide on how to easily manage your CI/CD on ECS using Gitlab Pipeline & ecs-deploy

a few weeks ago we needed to adopt a new product into the company, this product required building from scratch since it was relatively old and based on old technologies.
So first i created an ECS-cluster

I’ll focus on the CI/CD process, this means you already have:
1. ECS cluster running
2. ALB & Target groups
3. Task Definition/s
4. Service/s
5. code Repository/s (github or any other)
6. ECR with pull/push permissions set for a gitlab user.

Normally what you want is to ship your code into the services. you also probably want to make sure it runs ok after deployment, and you want to be able to rollback
You also want to be able to know right away, which commit triggered the deployment so you could quickly review it.

now… there are many ways to do the same thing, and there is also a more complicated configuration which involves multiple environment (Development,Staging,Production) which i don’t show here, i’ll explain the most basic configuration with only a “staging” environment. here is what i came up with:

* At first i installed Gitlab EE & Runner on a Docker node. i used persistent storage removed and recreated the container to make sure all data is intact.
You can read about running gitlab on docker Here:

~~~yaml
image: docker:git
services:
  - docker:dind

stages:
  - build
  - deploy
  - test
  - rollback

variables:
  IMAGE_TAG: $CI_PROJECT_NAME:$CI_COMMIT_SHA
  IMAGE_NAME: "$ECR_REPO/$IMAGE_TAG"

build:
  stage: build
  script:
    # Setup
    - export AWS_REGION="us-east-1"
    - export AWS_ACCESS_KEY_ID=$aws_access_key_id
    - export AWS_SECRET_ACCESS_KEY=$aws_secret_access_key
    - export REPO=$ECR_REPO
    - apk update
    - apk --no-cache add --update curl python python-dev py-pip
    - pip install awscli --upgrade --user
    - export PATH=~/.local/bin:/usr/bin/:$PATH
    # AUTH
    - CERT=`aws ecr get-login --no-include-email --region ${AWS_REGION}`
    - ${CERT}
    # Build
    - docker build -t ${CI_PROJECT_NAME} .
    - docker tag $CI_PROJECT_NAME:latest $REPO/$IMAGE_TAG
    - docker tag $CI_PROJECT_NAME:latest $REPO/${CI_PROJECT_NAME}:latest
    - docker push $REPO/$IMAGE_TAG
    - docker push $REPO/${CI_PROJECT_NAME}:latest
  environment:
    name: staging

deploy:
  stage: deploy
  script:
    - export AWS_REGION="us-east-1"
    - export AWS_ACCESS_KEY_ID=$aws_access_key_id
    - export AWS_SECRET_ACCESS_KEY=$aws_secret_access_key
    - apk --no-cache add --update python python-dev py-pip
    - pip install ecs-deploy
    # Deploy
    - ecs deploy --region ${AWS_REGION} ${CLUSTER_NAME} ${CI_PROJECT_NAME} --tag ${CI_COMMIT_SHA}
  environment:
    name: staging

test:
  stage: test
  script:
    - export AWS_REGION="us-east-1"
    - export AWS_ACCESS_KEY_ID=$aws_access_key_id
    - export AWS_SECRET_ACCESS_KEY=$aws_secret_access_key
    - apk --no-cache add --update curl python python-dev py-pip jq
    - pip install awscli --upgrade --user
    - export PATH=~/.local/bin:/usr/bin/:$PATH
    # Discover the ALB name
    - ALB=`aws elbv2 describe-load-balancers --region ${AWS_REGION} --names ${CI_PROJECT_NAME} | jq .LoadBalancers[0].DNSName`
    # Test Keepalive
    - /usr/bin/curl --fail http://${ALB//'"'}/keepalive
    # IF Keepalive return 200...
    # Retrieve & Store this revision as 'last known successful revision' in S3 Bucket
    - REV=`aws ecs describe-services --region ${AWS_REGION} --cluster ${CLUSTER_NAME} --service ${CI_PROJECT_NAME} |jq -r '.services[0].deployments[0].taskDefinition'`
    - echo successful revision is ${REV} Storing it in S3 Bucket
    - echo ${REV} > /${CI_PROJECT_NAME}
    # sync rev to S3 here
    - aws s3 cp /${CI_PROJECT_NAME} s3://${REV_BUCKET}
  environment:
    name: staging

rollback:
  stage: rollback
  script:
    - export AWS_REGION="us-east-1"
    - export AWS_ACCESS_KEY_ID=$aws_access_key_id
    - export AWS_SECRET_ACCESS_KEY=$aws_secret_access_key
    - apk --no-cache add --update curl python python-dev py-pip
    - pip install awscli --upgrade --user
    - export PATH=~/.local/bin:/usr/bin/:$PATH
    - pip install ecs-deploy
    - aws s3 cp s3://${REV_BUCKET}/${CI_PROJECT_NAME} ./
    - REV=`cat ./${CI_PROJECT_NAME}`
    - echo rev is $REV
    - ecs deploy --region ${AWS_REGION} ${CLUSTER_NAME} ${CI_PROJECT_NAME} --task ${REV}
  environment:
    name: staging

  when: on_failure
  
~~~

* The reason the same environment variables are set multiple times is because this is required.
Each Stage runs in it’s own clean container. This is explained here:


##### So what does this pipeline do ?


{{< figure src="/img/Screenshot-from-2017-10-04-11-07-28.png" title="gitlab-pipeline" >}}

Each time you commit & push code to your repository, gitlab will detect the changes & trigger the pipeline automatically
This flow mostly fits a basic “Staging environment”

1. Start a docker container and build our image.
2. Tag & push (twice: with ‘commit SHA’ & ‘latest’ as tags) this build
3. Set this image in a new ECS Task Definition based on the ACTIVE one & update your service to use it.
if deployment completed ok:
4. Test the application first, if OK: save this revision ARN name in S3 bucket and you are done.
If deployment not OK: Rollback.

Notice this part:
~~~yaml
  # AUTH
    - CERT=`aws ecr get-login --no-include-email --region ${AWS_REGION}`
    - ${CERT}
    # Build
    - docker build -t ${CI_PROJECT_NAME} .
    - docker tag $CI_PROJECT_NAME:latest $REPO/$IMAGE_TAG
    - docker tag $CI_PROJECT_NAME:latest $REPO/${CI_PROJECT_NAME}:latest
    - docker push $REPO/$IMAGE_TAG
    - docker push $REPO/${CI_PROJECT_NAME}:latest
  environment:
    name: staging
~~~

It is very clear why you should try to work with environment variables.
The above part could fit any of your projects in the future as is.


The deployment to the ECS cluster is done by the command ecs-deploy
This is a great project on Github which will save you a lot of time, instead of manipulating json files to create new task definition, and then update the service,
simply invoke the ecs-deploy command which does a great job. it is based on boto3, you can read about it here:


##### Also, notice this part:
~~~yaml
    - aws s3 cp s3://${REV_BUCKET}/${CI_PROJECT_NAME} ./
    - REV=`cat ./${CI_PROJECT_NAME}`
    - echo rev is $REV
    - ecs deploy --region ${AWS_REGION} ${CLUSTER_NAME} ${CI_PROJECT_NAME} --task ${REV}
    
~~~

Since you saved the last ARN of the Task Definition that passed the Test Stage, you can always roll-back to it by invoking:

~~~bash
ecs deploy --region ${AWS_REGION} ${CLUSTER_NAME} ${CI_PROJECT_NAME} --task ${REV}
~~~

That’s it
You should end up with a working CI/CD pipeline that will save you a lot of time

