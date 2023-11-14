# 

# <h1 align="center">CircleCI implementation </a>

> [!IMPORTANT]
> Check the [link](https://github.com/itsyndicate-support/webserver-ec2-module-terraform/pull/6) to the PR

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
