---
title: "USING GITLAB AS YOUR PRODUCTION CD"
date: 2018-02-25T21:14:29+03:00
draft: false
tags: ["ECS", "gitlab", "pipelines", "CI/CD"]
categories: ["Docker", "DevOps"]
---

### USING GITLAB AS YOUR PRODUCTION CI / CD SYSTEM ? YES, YOU SHOULD


![gitlab-banner](/img/gitlab-banner.png)

> Gitlab is a complete solution to unify your deployments and create a robust infrastructure tool, to manage your organization environments.

This is another take on a previous post i wrote last year, in which i needed to adopt a new product into the company, this product required building from scratch since it was relatively old and based on old technologies. 

The original post information is also included here, yet i wanted to come back a few months later and share some more information about where Containers, Kubernetes & Gitlab are taking us this year. 
The Original post is Here 

So first i created an ECS-cluster


I’ll focus on the CI/CD process, this means you already have:

1. ECS cluster running 
2. ALB & Target groups 
3. Task Definition/s 
4. Service/s 
5. code Repository/s (github or any other)
6. ECR with pull/push permissions set for a gitlab user.

Normally what you want is to ship your code into the services. you also probably want to make sure it runs ok after deployment, and you want to be able to rollback You also want to be able to know right away, which commit triggered the deployment so you could quickly review it.

now… there are many ways to do the same thing, and there is also a more complicated configuration which involves multiple environment (Development,Staging,Production) which i don’t show here, i’ll explain the most basic configuration with only a “staging” environment.
here is what i came up with:

1. At first i installed Gitlab EE & Runner on a Docker node. i used persistent storage removed and recreated the container to make sure all data is intact. You can read about running gitlab on docker Here

2. I created a user for gitlab on Github so it can Read the code repository,the Dockerfiles & the .gitlab-ci.yml file.
3. Created a projects group & Mirror the repositories into the projects (use a naming convention to easily identify your services later on)
4. Created a .gitlab-ci.yml file that will run your CI/CD pipelines
5. Recommended – Use environment variables: 

i set environment secrets at the gitlab group, this way all my services at this environment got the same env vars & i can run overwrite them for a specific project if needed. this keeps the .gitlab-ci.yml clean from hardcoded text and allows you to have a generic flow that will generally fit other projects as well.

Here is my .gitlab-ci.yml file that does CI and rolls back in case of test failure (modify to your needs) as mentioned, it is generic and uses Environment Variables, some of them are given to each pipeline as default by GitLab & some are defined at the project group Environment Secrets:

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

> It is very clear why you should try to work with environment variables. The above part could fit any of your projects in the future as is.


The deployment to the ECS cluster is done by the command ecs-deploy This is a great project on Github which will save you a lot of time, instead of manipulating json files to create new task definition, and then update the service, simply invoke the ecs-deploy command which does a great job. it is based on boto3, you can read about it here:
[ecs-deploy](https://github.com/fabfuel/ecs-deploy)

Also, notice this part:
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

So after a few months and many deployments later, this flow seems to work well. We had no issues related to GitLab/, and as new features coming out we just recently decided that Gitlab will be our entire infrastructure deployment tool. 

##### OK... SO GOODBYE JENKINS ?
here is what this means:
If you have Jenkins / Ansible Tower you can create jobs at the UI, you can specify parameters there easily, 
You can schedule jobs and you can for sure have a way to view all your jobs at one screen, 
So unlike gitlab they, generally allow any user to: choose a job, edit a variable and run their job or schedule it 

In Gitlab the jobs are real builds and a build is linked directly to a project which is created only if it has a git repository
Sadly this is a show stopper in the manner of gitlab dismissing Job managers such as Jenkins for example
which allows you to set up as many jobs as you want regardless to repositories and projects.

Also a user can see all his jobs grouped together with their last run statuses easily.
This simply does not apply to Gitlab at this stage and i can only hope that that will change soon.
However it is currently not mentioned on their 2018 Roadmap, but this does seem to be a requested feature at the community.

Beside the lack of Jobs management, to my opinion Gitlab is a great solution for CI/CD
Gitlab can be set up in your Cloud / Datacenter in High Available mode without any costs. Try to set up Jenkins (Cloudbees) in HW will drain your pockets
Gitlab fully supports containers, Kubernetes and it has DevOps point of view in all that related to Environments, Deployments, Testing...

For example:
In our infrastructure Gitlab itself scales automatically, the Runners too. It runs securly at our Private Cloud
when writing this post, less then 10 projects are considered "Production" and their CI/CD flow runs with gitlab, so scaling is not an issue right now, but it's always good to be ready before demand will spike.

If you are interested in setting up Gitlab in HA mode keep reading below


![Gitlab](/img/GitLab.png)

Gitlab is deployed on AWS with the following components: 

* RDS - postgresql 
* Redis - For gitlab cache 
* S3 buckets for config & the docker registry 
* EFS - Share the local files of gitlab between all gitab nodes 

The Gitlab application runs on 2 ElasticGroups on spot instances (easily integrated with "SpotInst" service) 

The 1st-app ElasticGroup only holds a single instance. This instance is the only one responsible for handling data migrations The second "Gitlab" ElasticGroup (This is the one you scale up/down) UserData contains:

~~~bash
touch /etc/gitlab/skip-auto-migrations 
 ~~~

Which prevents database migration The configuration is managed by the gitlab.rb file 
The instances are configured using the user-data file And the cloud-init.txt file 

The Gitlab application servers are sharing the same EFS mount in order to use their persistent data It is monitored with Cloudwatch and presented with our Grafana: Keep in mind that Gitlab do not recommend using EFS, but we decided it was good enough.

##### GITLAB RUNNERS
There is a single Runner instance which is using docker-machine to create instances in AWS, run the gitlab builds on them by demand and terminates them when Idle. 
Information about this Here 
##### RUNNERS CACHE
The Runners are using distributed cache which is written to S3 in order to reduce docker images build time. 
When configured This will make your builds run much faster, even on new runner-nodes running the build containers.

##### CONNECTIVITY & SECURITY
All access to the services is made with SSL in HTTPS 
The Gitlab applications (both ES groups) are behind the same internal ELB 
All instances + ELB are in a private VPC, and can be accessed only via: VPC / VPN client / DataCenter (IPSEC Tunnels)

TO BE CONTINUED...


