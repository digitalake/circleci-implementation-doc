# 

# <h1 align="center">CircleCI implementation </a>

> [!IMPORTANT]
> Check the [link](https://github.com/itsyndicate-support/webserver-ec2-module-terraform/pull/6) to the PR

Pipeline execution result:

![image](https://github.com/digitalake/circleci-implementation-doc/assets/109740456/0fdbb8d2-0f4e-4889-936f-606882110380)


### Preparing Dockerfiles

One ofthe most common parts for my `CircleCI pipeline` is preparing `Docker job base images`. In this case images were used:

- lighweight `docker.mirror.hashicorp.services/hashicorp/terraform:light` for all Terraform actions (init plan-apply apply plan-destroy destroy)
- custom `tfseq-cicd-git:0.0.1` with `git` installed for being able to `checkout`:
```
FROM aquasec/tfsec:v1.28

USER root

RUN apk add git
```
- custom `terratest:0.0.4` with `go` as base image, `terraform` binary and necessary `go modules`
```
FROM golang:1.21.4-bookworm

RUN apt-get update && \
    apt-get install -y wget zip && \
    rm -rf /var/lib/apt/lists/*

RUN wget https://releases.hashicorp.com/terraform/1.6.3/terraform_1.6.3_linux_amd64.zip && \
    unzip terraform_1.6.3_linux_amd64.zip && \
    mv /go/terraform /usr/local/bin/ && \
    rm terraform_1.6.3_linux_amd64.zip && \
    mkdir -p /go/src/terratest

WORKDIR /go/src/

# Download and install Go dependencies
COPY go.mod go.sum ./
RUN go mod download

WORKDIR /go/src/terratest
```

`Self-made images` allows to prepare the most effective environments for each job and exclude additional steps during executing.

### Implementing rollback

In my personal oppinion, `automated rollback` cannot be implemented on `main` branch not to damage the `git timeline`. I thing hat using the same flow as the `fix` branches can allow developers and devops engineers to follow the `familiar git flow`.

So the best option for `manual rollback` is:

- clonning the repo with `git clone`
- branching to the rollback-branch (let's say it`s the same `fix`) with `git checkout -b fix` 
- reverting the commit  `git revert HEAD`
- pushing with `git push origin fix`
- creating a PR with the CLI or with the UI

The rollback can be also imlemented with the separate `rollback pipeline`:
```
version: 2.1

jobs:
  rollback:
    docker:
      - image: circleci/golang:1.17
    steps:
      - run:
          name: Rollback
          command: |
            current_datetime=$(date +%Y-%m-%d_%H-%M)
            git config --global user.name "CircleRollback"
            git config --global user.email "circle.rollback@ci.com"
            git clone https://${TOKEN}@github.com/digitalake/webserver-ec2-module-terraform.git
            cd webserver-ec2-module-terraform
            git checkout -b rollback-fix-${current_datetime}
            git revert HEAD
            git remote set-url origin https://${TOKEN}@github.com/digitalake/rollback-fix-${current_datetime}.git
            git push origin
            curl -L \
              -X POST \
              -H "Accept: application/vnd.github+json" \
              -H "Authorization: Bearer ${TOKEN}" \
              -H "X-GitHub-Api-Version: 2022-11-28" \
              https://api.github.com/repos/digitalake/webserver-ec2-module-terraform/pulls \
              -d "{\"title\":\"Automatic Revert by CircleCI\",\"body\":\"This PR is automatically created by CircleCI for rollback. Timestamp: ${current_datetime}\",\"head\":\"rollback-fix-${current_datetime}\",\"base\":\"main\"}"
```

