# Deploying the Unreal Pixel Streaming infrastructure on OCI OKE

## Introduction

This project represents a scalable pixel streaming deployment on Oracle
Container Engine for Kubernetes (OKE). It is built intentionally using
the simplest constructs and/or dependencies with minimal customizations
to original samples from Epic Games

Estimated time: 5 minutes

### Objectives
- Use Terraform to create Kubernetes Cluster with GPU node pool and network

### Prerequisites
- Environment with GPU
- CLI and Terraform knowledge

## Task 1: Deploy Using the Terraform CLI

### Clone the Module

Clone the source code from suing the following command:

```bash
<copy>git clone https://github.com/oracle-quickstart/oke-unreal-pixel-streaming.git</copy>
```

```bash
<copy>cd oke-unreal-pixel-streaming/deploy/terraform</copy>
```

### Updating Terraform variables

```bash
<copy>cp terraform.tfvars.example terraform.tfvars</copy>
```

Update the `terraform.tfvars` file with the required variables, including the OCI credentials information.

Make sure that the information of the Instance Shape on each Node Pool are correct and you have enough quota to deploy the infrastructure, including the GPU nodes. This scripts defaults to `BM.GPU.A10.4`.

### Running Terraform

After specifying the required variables you can run the stack using the following commands:

```bash
<copy>terraform init</copy>
```

```bash
<copy>terraform plan</copy>
```

```bash
<copy>terraform apply</copy>
```

### Cluster Setup

There are three distinct node pools to use in this setup. Specifics regarding
node shape are suggested as baseline starting points, and can be customized by
requirements.

| Name                          | Description                                          | Node Shape            | Node Count |
| ----------------------------- | ---------------------------------------------------- | --------------------- | ---------- |
| [Default](#default-node-pool) | General cluster workloads                            | `VM.Standard.E4.Flex` | 3+         |
| [Turn](#turn-node-pool)       | Deploy `coturn` as DaemonSet with host networking    | `VM.Standard.E4.Flex` | 1+         |
| [GPU](#gpu-node-pool)         | PixelStreaming runtime with signal server as sidecar | *                     | 1+         |

> `*` Specific GPU shape can also vary depending on the application and scaling demands.
It is recommended to evaluate performance and settings accordingly.

### Default Node Pool

The default (or general) node pool is considered for multipurpose installations
or cluster-wide resources such as ingress controller, telemetry services,
applications, etc.

For purposes of this example, the standard _Quick Create_ workflow with public API
and private workers is considered adequate. Select alternatives, or customize as
necessary.

> Once created, note that the worker node subnet will have a `10.0.10.0/24` CIDR range.

> `*` Specific GPU shape can also vary depending on the application and scaling demands.
It is recommended to evaluate performance and settings accordingly.

### Turn Node Pool

This node pool is used exclusively for the STUN/TURN services running [coturn][coturn].
While coTURN is the most prevalent suggestion for hosting our own TURN services, alternates
like [Pion TURN][pion-turn] may be viable.

> `STUN` and `TURN` are network bound services, so specific attention to network bandwidth
> and associative compute sizing should be considered.

For public access, the nature of STUN/TURN dictates that the node pool is created
in a public subnet, with associative security list rules and a public route table
to work within OKE. In order to leverage host networking, the services are run
as a DaemonSet on this specific node pool. 

### GPU Node Pool

This is the node pool for the Pixel Streaming GPU workloads. Part of the design,
each Pixel Streaming runtime is directly asosciated with a corresponding node.js
signal server known as "cirrus" (`./signalserver` here). As such, each pixel streaming
container runs with cirrus as a sidecar on the same pod.

### Dependencies

As with many kubernetes systems, there are several choices for ancillary services
such as ingress controllers, certificate management, metrics, etc.. This solution
aims to offer viability using the most basic/standard dependencies:

- Ingress Controller: [documentation](https://kubernetes.github.io/ingress-nginx/deploy/)
- Cert Manager: [documentation](https://cert-manager.io/docs/installation/helm/)
- Metrics Server: [documentation](https://artifacthub.io/packages/helm/metrics-server/metrics-server)