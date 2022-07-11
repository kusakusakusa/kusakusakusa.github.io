---
title:  "Terraform With Docker"
date:   2019-10-26 09:00:00
permalink: terraform-with-docker
---

# Terraform With Docker

This is a documentation on how to use Terraform with Docker to provision cloud resources, mainly using AWS as the provider. It contains tips on certain practices that I personally deem best practices fo various reasons.

It will revolves around these 3 commands

```bash
docker run -v `pwd`:/workspace -w /workspace hashicorp/terraform:0.12.9 init
docker run -v `pwd`:/workspace -w /workspace hashicorp/terraform:0.12.9 apply
docker run -v `pwd`:/workspace -w /workspace hashicorp/terraform:0.12.9 destroy
```

The Terraform image comes with the entrypoint command terraform, so we will append the commands init and apply respectively.

## The Flags

The most straightforward way to run Terraform on docker is to do a docker run with a volume mount connecting the directory where the terraform files are to the working directory in the docker container. Assuming the current working directory is where the files are, we can simply run the command.

The `-v` option mounts your current working directory into the container's `/workspace` directory.

The `-w` flag creates the `/workspace` directory and sets it as the new working directory, overwriting the terraform image's original.

To verify this, we can run the command below to see that the current working directory in the container is in fact `/workspace`.

```bash
docker run -v `pwd`:/workspace -w /workspace --entrypoint /bin/sh hashicorp/terraform:0.12.9 -c pwd
```

Over here we are overwriting the default `entrypoint` command of `terraform` to run a shell command.

## terraform init

The `init` command will download the necessary provider files and modules required for the execution to the working directory in the container. And due to the volume mount, the files will be reflected in the current working directory on the local machine. The files would be downloaded to the folder `.terraform`.

It is paramount to have this files downloaded to the current directory because on subsequent runs, the files would not need to be downloaded again since they will be persisted and available.

## terraform apply

`apply` is the command to deploy the resources to the cloud.

If you do not use a Terraform backend, the `tfstate` file that holds all the information for the provisioning will be written to the working directory, and in turn to the current directory.

If you do use a `terraform` backend, there will be no `tfstate` file written locally. They will be written to the backend that you specified. However, in the event that a Terraform deployment fails, you will have a `errored.tfstate` file written to the working directory. This **`errored.tfstate`** file is extremely important to keep track of the state of provisioned environment in the event of failures.

A possible scenario is due to lost connectivity. I encountered that when I was traveling in some remote areas of Brazil. The volume mount saved my life. I am pretty surprised that there is not much documentation in the official terraform docs. I tried googling the query below but there is no result.

```bash
site:https://www.terraform.io/ "errored.tfstate"
```

Without the `errored.tfstate` file, undesirable duplicate resources may be created on subsequent deployments. In other cases, subsequent deployment itself might fail due to having resources of the same identification that is prohibited in AWS, which otherwise would not have occurred due to the out-of-sync state.

To update the state, we can run the command

```bash
docker run -v `pwd`:/workspace -w /workspace hashicorp/terraform:0.12.9 state push errored.tfstate
```

Apart from the `errored.tfstate` file, the `tf` log file that you specify will also be written, which is may be used for debugging your terraform deployments.

## terraform destroy

Lastly, destroy is the command to remove the resources from the cloud.

