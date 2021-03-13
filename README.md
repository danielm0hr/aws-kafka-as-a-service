# Kafka as a Service (KaaS) on AWS

An example on how to run Apache Kafka as a SaaS product on AWS, focusing on a simple, scalable, secure, multi-tenant design which abstracts the complexities of running Kafka away from the user. An application developer shall be able to just point her Kafka producers and consumers to the KaaS service, providing only some basic preferences like desired throughput and data retention.

From operations perspective, the goal is to be able to create appropriately scaled tenants in a fully automated way. This example implements [hard multi-tenancy](#hard-multi-tenancy) on runtime level by default. If you don't require hard multi-tenancy (and want to save some infrastructure costs), the provided scripts and resources can be adjusted relatively easily to fall back to Kubernetes namespace based tenant isolation instead. The needed changes are summarized in [Changes for Soft Multi-Tenancy](#changes-for-soft-multi-tenancy).

This example can be easily ported to other cloud providers and bare-metal because only a few AWS-only components are used, which all have similar replacements on other providers. Please refer to the documentation of the individual components on how to change the configuration.

One additional design goal was to reduce the amount of custom code needed to provide a fully functional SaaS product to a minimum. It uses off-the-shelf components for solving many of the non-functional (security, scalability, operational efficiency, simplicity, ...) and functional (multi-tenant, full-fledged Apache Kafka service) requirements, which are non-differentiating for a theoretical business around that SaaS product. The business should be able to spend its efforts focusing on the differentiating parts of the product.

## Table of Contents
- [Kafka as a Service (KaaS) on AWS](#kafka-as-a-service-kaas-on-aws)
  - [Table of Contents](#table-of-contents)
  - [Design choices](#design-choices)
    - [Runtime Platform](#runtime-platform)
    - [Apache Kafka Distribution](#apache-kafka-distribution)
    - [Multi-Tenancy](#multi-tenancy)
    - [Additional components](#additional-components)
  - [Getting started](#getting-started)
    - [Prerequisites](#prerequisites)
    - [Create the management cluster](#create-the-management-cluster)
    - [Create your first Kafka tenant](#create-your-first-kafka-tenant)
## Design choices

Which components are used and why.

### Runtime Platform

An obvious choice for the runtime platform is [**Kubernetes**](https://kubernetes.io/) because of its straight-forward sclability, reliability and security properties and because its the de-facto standard for container orchestration (yes, we use containers). Concretely, the AWS [**Elastic Kubernetes Service (EKS)**](https://aws.amazon.com/eks) is the Kubernetes distribution (managed through [Cluster API](https://cluster-api.sigs.k8s.io/)) of our choice, providing currently the simplest way of running production grade Kubernetes clusters on AWS.

### Apache Kafka Distribution

Kafka, as a complex stateful system, can be run in many different ways and configurations. We are chosing the [**Strimzi Kafka on Kubernetes**](https://strimzi.io/) distribution because it seems to be the most advanced and feature rich open source and thus free to use option for running Kafka on Kubernetes. There are comparable and even more advanced commercial options available.

### Multi-Tenancy

This example is implementing hard multi-tenancy on runtime level by running the tenant Kafka clusters on separate Kubernetes clusters. The assumption is that a KaaS customer would require a higher level of isolation of her Kafka installation and the data in it than a single Kubernetes cluster can provide out of the box e.g. by separating tenants on Kubernetes namespace level.

The largest drawback of this solution is obviously the added infrastracture costs for running a full Kubernetes control plane and a set of dedicated worker nodes per tenant (compute and storage can't be shared between multiple tenants and thus the amount of unused resources can be higher). The significance of this drawback is inversely proportional to the size of the tenant clusters. For tenants with hundreds of worker nodes the additional resources for running the Kubernetes control plane are usually neglectable. Thus it can be postulated that this example is mainly focusing on having rather large tenants.

Please see [Changes for soft Multi-Tenancy](#changes-for-soft-multi-tenancy) for a theoretical consideration of how to change this example for running your tenants on a single Kubernetes cluster with namespace-level tenant isolation.
### Additional components

- TLS certificate management: [**cert-manager**](https://cert-manager.io/) with [**Let's Encrypt**](https://letsencrypt.org/)
- DNS management: [**external-dns**](https://github.com/kubernetes-sigs/external-dns) with [**AWS Route53**](https://aws.amazon.com/route53)
- K8s cluster management (see [Multi-Tenancy](#multi-tenancy)): [**Cluster API**](https://cluster-api.sigs.k8s.io/).

## Getting started

How to get this example running yourself.

### Prerequisites

- An AWS account with appropriate permissions to create the neccessary resources
- A DNS zone managed by Route 53 which can be used to create the neccessary endpoints
- Install the following CLI tools
    - [`aws`](https://docs.aws.amazon.com/cli/latest/userguide/)
    - [`eksctl`](https://eksctl.io/)
    - [`clusterctl`](https://cluster-api.sigs.k8s.io/user/quick-start.html#install-clusterctl)
    - [`clusterawsadm`](https://github.com/kubernetes-sigs/cluster-api-provider-aws/releases)
    - [`flux`](https://toolkit.fluxcd.io/get-started/#install-the-flux-cli)
- A Git repository you want to use for your tenant cluster configuration. This will be used by the FluxCD GitOps operator on the management cluster. In the example we use this repository (subpath [tenants](tenants)).

### Create the management cluster

First, the management Kubernetes cluster needs to be created. This is the cluster which you mainly interact with directly and where the lifecycle of the tenant (workload) clusters will be managed.

1. Setup your AWS credentials using e.g. `aws configure`.
2. [Import or create an SSH key pair for EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#prepare-key-pair)
3. Create the management cluster with
    ```
    eksctl create cluster \
      --name kaas-mgmt \
      --region us-east-1 \
      --with-oidc \
      --ssh-access \
      --ssh-public-key <key> \
      --managed
    ```
   and wait for it to be ready.
4. In order to set the appropriate IAM configuration for Cluster API, run
    ```
    clusterawsadm bootstrap iam create-cloudformation-stack
    ```
5. Set an environment variable for putting the IAM credentials in a Kubernetes secret
    ```
    export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)
    ```
6. Initialize the AWS Cluster API provider on your management cluster
    ```
    clusterctl init --infrastructure=aws
    ```

7. Install FluxCD to your management cluster and connect it to this Github repo (for other Git providers see the [FluxCD bootstrap docs](https://toolkit.fluxcd.io/guides/installation/#bootstrap))
    ```
    export GITHUB_USER=<username>
    export GITHUB_TOKEN=<acces-token>

    flux bootstrap github \
    --owner=$GITHUB_USER \
    --repository=aws-kafka-as-a-service \
    --branch=main \
    --path=./tenants \
    --personal
    ```


### Create your first Kafka tenant

1. Create your [Cluster API configuration](https://cluster-api.sigs.k8s.io/user/quick-start.html#required-configuration-for-common-providers) in `~/.cluster-api/clusterctl.yml` (adjust fields appropriately):
    ```
    export AWS_REGION=us-east-1
    export AWS_SSH_KEY_NAME=default
    # Select instance types
    export AWS_CONTROL_PLANE_MACHINE_TYPE=t3.large
    export AWS_NODE_MACHINE_TYPE=t3.large
    ```







