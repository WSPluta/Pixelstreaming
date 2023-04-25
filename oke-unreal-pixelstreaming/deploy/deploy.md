# Deploying the Unreal Pixel Streaming infrastructure on OCI OKE

## Introduction

A Kubernetes cluster is a group of nodes. The nodes are the machines running applications. Each node can be a physical machine or a virtual machine. The node's capacity (its number of CPUs and amount of memory) is defined when the node is created. A cluster comprises:

- one or more master nodes (for high availability, typically there will be a number of master nodes)
- one or more worker nodes (sometimes known as minions)

A Kubernetes cluster can be organized into namespaces to divide the cluster's resources between multiple users. Initially, a cluster has the following namespaces:

- default, for resources with no other namespace
- kube-system, for resources created by the Kubernetes system
- kube-node-lease, for one lease object per node to help determine node availability
- kube-public, usually used for resources that have to be accessible across the cluster

Estimated time: 45 minutes

### Objectives
- Create Kubernetes Cluster with GPU node pool
- Deploy Unreal 'Hello World' demo

### Prerequisites
- OCI Command Line Interface (CLI) installation on your local machine


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

## Task 2: Pixelstreaming Hello World

Prebuilt images are included with this repo, along with a demo
Pixel Streaming image. With a cluster configured per the instructions
above, you can deploy the entire runtime with the following:

```bash
<copy>kubectl create ns demo</copy>
```

```bash
<copy>kubectl apply -f ../demo.yaml</copy>
```

> Note: Demo App uses Prebuilt images are included with this repo, along with a demo Pixel Streaming image. You can build your own images using the instructions [here](../README.md#pixel-streaming-build).

## Cluster Setup

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
as a DaemonSet on this specific node pool. These following setup used to acheive
a single node deployment:

1. Create a public subnet (regional) in the OKE cluster VCN. (This example used `10.0.30.0/24` CIDR block)
1. Assign default DHCP options for cluster vcn
1. Assign the public route table to the public subnet (default from OKE is fine)
1. Assign/update the _existing_ **node** security list for the TURN subnet CIDR block
  
    | Dir     | Source/Dest    | Protocol      | Src Port | Dest Port   | Type/Code | Description                                                             |
    | ------- | -------------- | ------------- | -------- | ----------- | --------- | ----------------------------------------------------------------------- |
    | Ingress | `10.0.30.0/24` | All Protocols | *        | *           |           | Allow pods on turn nodes to communicate with pods on other worker nodes |
    | Egress  | `10.0.30.0/24` | All Protocols | *        | *           |           | Allow pods on turn nodes to communicate with pods on other worker nodes |
    | Ingress | `0.0.0.0/0`    | TCP           | *        | 3478        |           | STUN TCP                                                                |
    | Ingress | `0.0.0.0/0`    | UDP           | *        | 3478        |           | TURN UDP                                                                |
    | Ingress | `0.0.0.0/0`    | TCP           | *        | 49152-65535 |           | STUN Connection ports                                                   |
    | Ingress | `0.0.0.0/0`    | UDP           | *        | 49152-65535 |           | TURN Connection ports                                                   |

1. Update K8s API endpoint security list to include ingress/egress to the turn CIDR block

    | Dir     | Source/Dest    | Protocol | Src Port | Dest Port | Type/Code | Description                      |
    | ------- | -------------- | -------- | -------- | --------- | --------- | -------------------------------- |
    | Ingress | `10.0.30.0/24` | TCP      | *        | 6443      |           | turn worker to k8s API endpoint  |
    | Ingress | `10.0.30.0/24` | TCP      | *        | 12250     |           | turn worker to OKE control plane |
    | Ingress | `10.0.30.0/24` | ICMP     |          |           | 3, 4      | path discovery turn              |
    | Egress  | `10.0.30.0/24` | ICMP     |          |           | 3, 4      | path discovery turn              |
    | Egress  | `10.0.30.0/24` | TCP      | *        | *         |           | TURN traffic from worker nodes   |
  
1. Create the node pool using **Advanced Options** to specify additional k8s key-value labels:

    ```text
    app.pixel/turn=true
    ```

1. Taint each node after they start to ensure selective node assignment/affinity

    ```sh
    # assuming node pool was labeled with 'app.pixel/turn=true' in provisioning
    kubectl taint nodes $(kubectl get nodes -l app.pixel/turn=true --no-headers | awk '{print $1}') app.pixel/turn=true:NoSchedule
    ```

    > NOTE: this is done automatically as part of the `turn` DaemonSet

### GPU Node Pool

This is the node pool for the Pixel Streaming GPU workloads. Part of the design,
each Pixel Streaming runtime is directly asosciated with a corresponding node.js
signal server known as "cirrus" (`./signalserver` here). As such, each pixel streaming
container runs with cirrus as a sidecar on the same pod.

It is necessary to differentiate the GPU pool from others with a kubernetes label.
Create the node pool using **Advanced Options** to specify additional k8s labels:

```text
app.pixel/gpu=true
```

> Read more on this under [GPU Allocation](#gpu-allocation)

The architecture used here for pixel streaming does not require any specific
network/subnet other than the general OKE node subnet

### Dependencies

As with many kubernetes systems, there are several choices for anciliary services
such as ingress controllers, certificate management, metrics, etc.. This solution
aims to offer viability using the most basic/standard dependencies:

- Ingress Controller: [documentation](https://kubernetes.github.io/ingress-nginx/deploy/)

    ```sh
    helm upgrade --install ingress-nginx ingress-nginx \
      --repo https://kubernetes.github.io/ingress-nginx \
      --namespace ingress-nginx --create-namespace 
    ```

- Cert Manager: [documentation](https://cert-manager.io/docs/installation/helm/)

    1. Install CRDs (if not already installed)

        ```sh
        kubectl apply -f \
          https://github.com/jetstack/cert-manager/releases/download/v1.6.0/cert-manager.crds.yaml
        ```

    1. Install chart

        ```sh
        helm upgrade --install cert-manager cert-manager \
          --repo https://charts.jetstack.io \
          --namespace cert-manager --create-namespace \
          --version v1.6.0
        ```

    1. Create associative `ClusterIssuer` or `Issuer` resources depending on needs. As an example:

        ```sh
        # adjust as needed
        kubectl apply -f ./support/misc/issuer.yaml
        ```

- Metrics Server: [documentation](https://artifacthub.io/packages/helm/metrics-server/metrics-server)

    1. Install chart

        ```sh
        helm upgrade --install metrics-server metrics-server \
          --repo https://kubernetes-sigs.github.io/metrics-server/ \
          --namespace metrics --create-namespace
        ```
