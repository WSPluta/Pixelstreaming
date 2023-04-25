# Terraform Scripts for deploying the Unreal Pixel Streaming infrastructure on OCI OKE

## Introduction
Get hands-on learning with training labs about Oracle cloud solutions. The workshops featured cover various solutions, skill levels, and categories based on Oracle Cloud Infrastructure (OCI).

Estimated time: 10 minutes


## Task 1: Destroying the Stack

```bash
<copy>terraform destroy -refresh=false</copy>
```

> Note: The `-refresh=false` flag is required to prevent Terraform from attempting to refresh the state of the kubernetes API url, which will return `localhost` without the refresh-false.