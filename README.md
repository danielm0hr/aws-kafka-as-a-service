# Kafka as a Service (KaaS) on AWS

An example on how to run Apache Kafka as a SaaS product on AWS, focusing on a simple, scalable, secure, multi-tenant design which abstracts the complexities of running Kafka away from the user. An application developer shall be able to just point her Kafka producers and consumers to the KaaS service, providing only some basic preferences like desired throughput and data retention.

From operations perspective, the goal is to be able to create appropriately scaled tenants in a fully automated way. This example implements [hard multi-tenancy](#hard-multi-tenancy) on runtime level by default. If you don't require hard multi-tenancy (and want to save some infrastructure costs), the provided scripts and resources can be adjusted easily to fall back to Kubernetes namespace based tenant isolation instead. The needed changes are summarized in [Changes for Soft Multi-Tenancy](#changes-for-soft-multi-tenancy).

This example can be easily ported to other cloud providers and bare-metal because no AWS-only components are used. Please refer to the documentation of the individual components on how to change the configuration.

## Table of Contents
- [Kafka as a Service (KaaS) on AWS](#kafka-as-a-service-kaas-on-aws)
  - [Table of Contents](#table-of-contents)
  - [Components](#components)
    - [Runtime Platform](#runtime-platform)
    - [Apache Kafka Distribution](#apache-kafka-distribution)
    - [Additional components](#additional-components)
  - [Getting started](#getting-started)
    - [Prerequisites](#prerequisites)
    - [K8s cluster setup](#k8s-cluster-setup)
    - [Install FluxCD to your management cluster](#install-fluxcd-to-your-management-cluster)
    - [Create your first Kafka tenant](#create-your-first-kafka-tenant)
## Components

The following components have been used in this exercise.

### Runtime Platform

An obvious choice for the runtime platform is [**Kubernetes**](https://kubernetes.io/) because of its straight-forward sclability, reliability and security properties and because its the de-facto standard for container orchestration (yes, we use containers). Concretely, the AWS [**Elastic Kubernetes Service (EKS)**](https://aws.amazon.com/eks) is the Kubernetes distribution of our choice, providing currently the simplest way of running production grade Kubernetes clusters on AWS.

### Apache Kafka Distribution

Kafka, as a complex stateful system, can be run in many different ways and configurations. We are chosing the [**Strimzi Kafka on Kubernetes**](https://strimzi.io/) distribution because it seems to be the most advanced and feature rich open source and thus free to use option for running Kafka on Kubernetes. There are comparable and even more advanced commercial options available.

### Additional components

- TLS certificate management: [**cert-manager**](https://cert-manager.io/) with [**Let's Encrypt**](https://letsencrypt.org/)
- DNS management: [**external-dns**](https://github.com/kubernetes-sigs/external-dns) with [**AWS Route53**](https://aws.amazon.com/route53)
- K8s cluster management (see [Hard Multi-Tenancy](#hard-multi-tenancy) below): [**Cluster API**](https://cluster-api.sigs.k8s.io/).

## Getting started

How to get this example running yourself.

### Prerequisites

- AWS account with appropriate permissions to create the neccessary resources
- A DNS zone managed by Route 53 which can be used to create user-facing endpoints

### K8s cluster setup

1. Bootstrap a Cluster API management cluster like described in the [Cluster API Book](https://cluster-api.sigs.k8s.io/user/quick-start.html#quick-start). This also includes configuring your AWS access credentials and installing the `clusterctl` and `clusterawsadm` CLI tools.
2. Create your [Cluster API configuration](https://cluster-api.sigs.k8s.io/user/quick-start.html#required-configuration-for-common-providers) in `~/.cluster-api/clusterctl.yml` (adjust fields appropriately):
    ```
    export AWS_REGION=us-east-1
    export AWS_SSH_KEY_NAME=default
    # Select instance types
    export AWS_CONTROL_PLANE_MACHINE_TYPE=t3.large
    export AWS_NODE_MACHINE_TYPE=t3.large
    ```

### Install FluxCD to your management cluster

### Create your first Kafka tenant



