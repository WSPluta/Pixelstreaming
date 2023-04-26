# Deploying the Unreal Pixel Streaming infrastructure on OCI OKE

## Introduction

Welcome to this hands-on lab, where you will learn how to create a Kubernetes cluster with a GPU node pool and deploy an Unreal Pixelstreaming demo.

In this lab, we will be using Kubernetes, an open-source container orchestration platform, to manage our cluster of nodes.

We will be creating a GPU node pool to enable us to run pixel streaming workloads on the cluster. Pixel streaming is a technology that enables you to stream high-quality 3D graphics over the internet.

Estimated time: 45 minutes

### Objectives
- Create Kubernetes Cluster with GPU node pool
- Deploy Unreal 'Hello World' demo

### Prerequisites
- OCI Command Line Interface (CLI) installation on your local machine


## Task 1: Deploying prebuilt Unreal Pixelstreaming demo

Prebuilt images are included with this repo, along with a demo
Pixel Streaming image. With a cluster configured per the instructions
above, you can deploy the entire runtime with the following:

```bash
<copy>kubectl create ns demo</copy>
```

```bash
<copy>kubectl apply -f ../demo.yaml</copy>
```

## Optional: Unreal Engine docker images

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