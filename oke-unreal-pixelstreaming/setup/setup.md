# Getting started with Unreal Engine and Oracle Cloud GPU

## Introduction

A Kubernetes cluster is a group of nodes. The nodes are the machines running applications. Each node can be a physical machine or a virtual machine. The node's capacity (its number of CPUs and amount of memory) is defined when the node is created. A cluster comprises:

- one or more master nodes (for high availability, typically there will be a number of master nodes)
- one or more worker nodes (sometimes known as minions)

A Kubernetes cluster can be organized into namespaces to divide the cluster's resources between multiple users. Initially, a cluster has the following namespaces:

- default, for resources with no other namespace
- kube-system, for resources created by the Kubernetes system
- kube-node-lease, for one lease object per node to help determine node availability
- kube-public, usually used for resources that have to be accessible across the cluster

Estimated time: 10 minutes

### Objectives
- Create Kubernetes Cluster


### Prerequisites
- OCI Command Line Interface (CLI) installation on your local machine


## Task 1: Unreal Engine docker images

For this piece, an example `Dockerfile` is provided in the [unreal](../unreal/Dockerfile) directory.

In this example, it is expected that the `./project` relative path contains the
full project source, which would be `./project/PixelStreamingDemo.uproject` in
this case - update as necessary.

> **NOTE** Access to the [official](https://unrealcontainers.com/docs/obtaining-images/official-images) Unreal Engine docker images
(hosted on `ghcr.io/epicgames/unreal-engine`) is restricted, so it is necessary
to sign up and register for access. 

Instructions are [here](https://github.com/EpicGames/Signup)


https://www.unrealengine.com/en-US/ue-on-github


Once repo access is obtained, the basic build process is as follows:

1. Authenticate to [`ghcr`](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry#authenticating-to-the-container-registry)

1. Build the unreal project

    ```sh
    # change to the project directory containing dockerfile described above
    cd path/to/ue4/project
    # docker build (in current directory '.')
    docker build -t my-pixelstream:latest .
    ```

1. Tag and push to OCIR per [documentation](https://docs.oracle.com/en-us/iaas/Content/Registry/Tasks/registrypushingimagesusingthedockercli.htm).

*Please proceed to the next lab*

## Acknowledgements

- **Author** - Kay Malcolm, Director, Product Management
- **Adapted by** -  Yaisah Granillo, Cloud Solution Engineer, NA Cloud
- **Contributors** - LiveLabs QA Team (Arabella Yao, Product Manager Intern | Isa Kessinger, QA Intern)
- **Last Updated By/Date** - Kay Malcolm, October 2020
