# cluster-autoscaler

[The cluster autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) scales worker nodes within an AWS autoscaling group (ASG) or Spotinst Elastigroup.

## TL;DR:

```console
$ helm install stable/cluster-autoscaler --name my-release --set "autoscalingGroups[0].name=your-asg-name,autoscalingGroups[0].maxSize=10,autoscalingGroups[0].minSize=1"
```

## Introduction

This chart bootstraps a cluster-autoscaler deployment on a [Kubernetes](http://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.

## Prerequisites

  - Kubernetes 1.8+
> [older versions](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler#releases) may work by overriding the `image`. Cluster-autoscaler internally simulates the scheduler and bugs between mismatched versions may be subtle.
  - Azure AKS specific Prerequisites:
    - Kubernetes 1.10+ with RBAC-enabled

## Installing the Chart

**By default, no deployment is created and nothing will autoscale**.

You must provide some minimal configuration, either to specify instance groups or enable auto-discovery. It is not recommended to do both.

Either:
  - set `autoDiscovery.clusterName` and tag your autoscaling groups appropriately (`--cloud-provider=aws` only) **or**
  - set at least one ASG as an element in the `autoscalingGroups` array with its three values: `name`, `minSize` and `maxSize`.

To install the chart with the release name `my-release`:

### Using auto-discovery of tagged instance groups

#### AWS

Auto-discovery finds ASGs tags as below and automatically manages them based on the min and max size specified in the ASG. `cloudProvider=aws` only.

1) tag the ASGs with _key_ `k8s.io/cluster-autoscaler/enabled` and _key_ `kubernetes.io/cluster/<YOUR CLUSTER NAME>`
2) verify the [IAM Permissions](#iam)
3) set `autoDiscovery.clusterName=<YOUR CLUSTER NAME>`
4) set `awsRegion=<YOUR AWS REGION>`
5) set `awsAccessKeyID=<YOUR AWS KEY ID>` and `awsSecretAccessKey=<YOUR AWS SECRET KEY>` if you want to [use AWS credentials directly instead of an instance role](https://github.com/kubernetes/autoscaler/blob/5ac706fdfa5601348f33d5b634e62de6655bb9bf/cluster-autoscaler/cloudprovider/aws/README.md#using-aws-credentials)

```console
$ helm install stable/cluster-autoscaler --name my-release --set autoDiscovery.clusterName=<CLUSTER NAME>
```

The [auto-discovery](#auto-discovery) section provides more details and examples

#### GCE
##### Required parameters
- `autoDiscovery.clusterName=any-name`
- `--cloud-provider=gce`
- `autoscalingGroupsnamePrefix[0].name=your-ig-prefix,autoscalingGroupsnamePrefix[0].maxSize=10,autoscalingGroupsnamePrefix[0].minSize=1`

To use Managed Instance Group (MIG) auto-discovery, provide a YAML file setting `autoscalingGroupsnamePrefix` (see values.yaml) or use `--set` when installing the Chart - e.g.

```console
$ helm install stable/cluster-autoscaler \
--name my-release \
--set autoDiscovery.clusterName=<CLUSTER NAME> \
--set cloudProvider=gce \
--set "autoscalingGroupsnamePrefix[0].name=your-ig-prefix,autoscalingGroupsnamePrefix[0].maxSize=10,autoscalingGroupsnamePrefix[0].minSize=1"
```

Note that `your-ig-prefix` should be a _prefix_ matching one or more MIGs, and _not_ the full name of the MIG. For example, to match multiple instance groups - `k8s-node-group-a-standard`, `k8s-node-group-b-gpu`, you would use a prefix of `k8s-node-group-`.

In the event you want to explicitly specify MIGs instead of using auto-discovery, set members of the `autoscalingGroups` array directly - e.g.

```
# where 'n' is the index, starting at 0
-- set autoscalingGroups[n].name=https://content.googleapis.com/compute/v1/projects/$PROJECTID/zones/$ZONENAME/instanceGroupManagers/$FULL-MIG-NAME,autoscalingGroups[n].maxSize=$MAXSIZE,autoscalingGroups[n].minSize=$MINSIZE
```

#### Azure AKS
##### Required Parameters
- `cloudProvider=azure`
- `autoscalingGroups[0].name=your-agent-pool,autoscalingGroups[0].maxSize=10,autoscalingGroups[0].minSize=1`
- `azureClientID: "your-service-principal-app-id"`
- `azureClientSecret: "your-service-principal-client-secret"`
- `azureSubscriptionID: "your-azure-subscription-id"`
- `azureTenantID: "your-azure-tenant-id"`
- `azureClusterName: "your-aks-cluster-name"`
- `azureResourceGroup: "your-aks-cluster-resource-group-name"`
- `azureVMType: "AKS"`
- `azureNodeResourceGroup: "your-aks-cluster-node-resource-group"`


### Specifying groups manually (only aws)

Without autodiscovery, specify an array of elements each containing ASG name, min size, max size. The sizes specified here will be applied to the ASG, assuming IAM permissions are correctly configured.

1) verify the [IAM Permissions](#iam)
2) Either provide a yaml file setting `autoscalingGroups` (see values.yaml) or use `--set` e.g.:

```console
$ helm install stable/cluster-autoscaler --name my-release --set "autoscalingGroups[0].name=your-asg-name,autoscalingGroups[0].maxSize=10,autoscalingGroups[0].minSize=1"
```

## Uninstalling the Chart

To uninstall `my-release`:

```console
$ helm delete my-release
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

> **Tip**: List all releases using `helm list` or start clean with `helm delete --purge my-release`

## Configuration

The following table lists the configurable parameters of the cluster-autoscaler chart and their default values.

Parameter | Description | Default
--- | --- | ---
`affinity` | node/pod affinities | None
`autoDiscovery.clusterName` | enable autodiscovery for name in ASG tag (only `cloudProvider=aws`). Must be set for `cloudProvider=gce`, but no MIG tagging required.| `""` **required unless autoscalingGroups[] provided**
`autoscalingGroups[].name` | autoscaling group name | None. Required unless `autoDiscovery.enabled=true`
`autoscalingGroups[].maxSize` | maximum autoscaling group size | None. Required unless `autoDiscovery.enabled=true`
`autoscalingGroups[].minSize` | minimum autoscaling group size | None. Required unless `autoDiscovery.enabled=true`
`awsRegion` | AWS region (required if `cloudProvider=aws`) | `us-east-1`
`awsAccessKeyID` | AWS access key ID ([if AWS user keys used](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md#using-aws-credentials)) | `""`
`awsSecretAccessKey` | AWS access secret key ([if AWS user keys used](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md#using-aws-credentials)) | `""`
`autoscalingGroupsnamePrefix[].name` | GCE MIG name prefix (the full name is invalid) | None. Required for `cloudProvider=gce`
`autoscalingGroupsnamePrefix[].maxSize` | maximum MIG size | None. Required for `cloudProvider=gce`
`autoscalingGroupsnamePrefix[].minSize` | minimum MIG size |  None. Required for `cloudProvider=gce`
`sslCertPath` | Path on the pod where ssl ca cert exists | `/etc/ssl/certs/ca-certificates.crt`
`sslCertHostPath` | Path on the host where ssl ca cert exists | `/etc/ssl/certs/ca-certificates.crt`
`cloudProvider` | `aws` or `spotinst` are currently supported for AWS. `gce` for GCE. `azure` for Azure AKS | `aws`
`image.repository` | Image | `k8s.gcr.io/cluster-autoscaler`
`image.tag` | Image tag  | `v1.13.1`
`image.pullPolicy` | Image pull policy  | `IfNotPresent`
`extraArgs` | additional container arguments | `{}`
`podDisruptionBudget` | Pod disruption budget | `maxUnavailable: 1`
`extraEnv` | additional container environment variables | `{}`
`envFromSecret` | additional container environment variables from secret | `nil`
`nodeSelector` | node labels for pod assignment | `{}`
`podAnnotations` | annotations to add to each pod | `{}`
`deployment.apiVersion` | apiVersion for the deployment | `extensions/v1beta1`
`rbac.create` | If true, create & use RBAC resources | `false`
`rbac.serviceAccountName` | existing ServiceAccount to use (ignored if rbac.create=true) | `default`
`rbac.pspEnabled` | Must be used with `rbac.create` true. If true, creates & uses RBAC resources required in the cluster with [Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) enabled. | `false`
`replicaCount` | desired number of pods | `1`
`priorityClassName` | priorityClassName | `nil`
`dnsPolicy` | dnsPolicy | `nil`
`resources` | pod resource requests & limits | `{}`
`service.annotations` | annotations to add to service | none
`service.clusterIP` | IP address to assign to service | `""`
`service.externalIPs` | service external IP addresses | `[]`
`service.loadBalancerIP` | IP address to assign to load balancer (if supported) | `""`
`service.loadBalancerSourceRanges` | list of IP CIDRs allowed access to load balancer (if supported) | `[]`
`service.servicePort` | service port to expose | `8085`
`service.portName` | name for service port | `http`
`service.type` | type of service to create | `ClusterIP`
`spotinst.account` | Spotinst Account ID (required if `cloudprovider=spotinst`) | `""`
`spotinst.token` | Spotinst API token (required if `cloudprovider=spotinst`) | `""`
`spotinst.image.repository` | Image (used if `cloudProvider=spotinst`) | `spotinst/kubernetes-cluster-autoscaler`
`spotinst.image.tag` | Image tag (used if `cloudProvider=spotinst`) | `v0.6.0`
`spotinst.image.pullPolicy` | Image pull policy (used if `cloudProvider=spotinst`) | `IfNotPresent`
`tolerations` | List of node taints to tolerate (requires Kubernetes >= 1.6) | `[]`
`serviceMonitor.enabled` | if `true`, creates a Prometheus Operator ServiceMonitor | `false`
`serviceMonitor.interval` | Interval that Prometheus scrapes Cluster Autoscaler metrics | `10s`
`serviceMonitor.namespace` | Namespace which Prometheus is running in | `monitoring`
`serviceMonitor.selector` | Default to kube-prometheus install (CoreOS recommended), but should be set according to Prometheus install | `{ prometheus: kube-prometheus }`
`azureClientID` | Service Principal ClientID with contributor permission to Cluster and Node ResourceGroup | none
`azureClientSecret` | Service Principal ClientSecret with contributor permission to Cluster and Node ResourceGroup | none
`azureSubscriptionID` | Azure subscription where the resources are located | none
`azureTenantID` | Azure tenant where the resources are located | none
`azureClusterName` | Azure AKS cluster name | none
`azureResourceGroup` | Azure resource group that the cluster is located | none
`azureVMType: "AKS"` | Azure VM type | `AKS`
`azureNodeResourceGroup` | azure resource group where the clusters Nodes are located, typically set as `MC_<cluster-resource-group-name>_<cluster-name>_<location>` | none
`azureUseManagedIdentityExtension` | Whether to use Azure's managed identity extension for credentials | false

Specify each parameter you'd like to override using a YAML file as described above in the [installation](#installing-the-chart) section or by using the `--set key=value[,key=value]` argument to `helm install`. For example, to change the region and [expander](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#what-are-expanders):

```console
$ helm install stable/cluster-autoscaler --name my-release \
    --set extraArgs.expander=most-pods \
    --set awsRegion=us-west-1
```

## IAM

The worker running the cluster autoscaler will need access to certain resources and actions:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup"
            ],
            "Resource": "*"
        }
    ]
}
```

  - `DescribeTags` is required for autodiscovery.
  - `DescribeLaunchconfigurations` is required to scale up an ASG from 0

Unfortunately AWS does not support ARNs for autoscaling groups yet so you must use "*" as the resource. More information [here](http://docs.aws.amazon.com/autoscaling/latest/userguide/IAM.html#UsingWithAutoScaling_Actions).

## Auto-discovery

For auto-discovery of instances to work, they must be tagged with
`k8s.io/cluster-autoscaler/enabled` and `kubernetes.io/cluster/<ClusterName>`

The value of the tag does not matter, only the key.

An example kops spec excerpt:

```
apiVersion: kops/v1alpha2
kind: Cluster
metadata:
  name: my.cluster.internal
spec:
  additionalPolicies:
    node: |
      [
        {"Effect":"Allow","Action":["autoscaling:DescribeAutoScalingGroups","autoscaling:DescribeAutoScalingInstances","autoscaling:DescribeLaunchConfigurations","autoscaling:DescribeTags","autoscaling:SetDesiredCapacity","autoscaling:TerminateInstanceInAutoScalingGroup"],"Resource":"*"}
      ]
      ...
---
apiVersion: kops/v1alpha2
kind: InstanceGroup
metadata:
  labels:
    kops.k8s.io/cluster: my.cluster.internal
  name: my-instances
spec:
  cloudLabels:
    k8s.io/cluster-autoscaler/enabled: ""
    kubernetes.io/cluster/my.cluster.internal: owned
  image: kope.io/k8s-1.8-debian-jessie-amd64-hvm-ebs-2018-01-14
  machineType: r4.large
  maxSize: 4
  minSize: 0
```

In this example you would need to `--set autoDiscovery.clusterName=my.cluster.internal` when installing.

It is not recommended to try to mix this with setting `autoscalingGroups`

See [autoscaler AWS documentation](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md#auto-discovery-setup) for a more discussion of the setup

### Troubleshooting

The chart will succeed even if the container arguments are incorrect. A few minutes after starting
`kubectl logs -l "app=aws-cluster-autoscaler" --tail=50` should loop through something like

```
polling_autoscaler.go:111] Poll finished
static_autoscaler.go:97] Starting main loop
utils.go:435] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
static_autoscaler.go:230] Filtering out schedulables
```

If not, find a pod that the deployment created and `describe` it, paying close attention to the arguments under `Command`. e.g.:

```
Containers:
  cluster-autoscaler:
    Command:
      ./cluster-autoscaler
      --cloud-provider=aws
# if specifying ASGs manually
      --nodes=1:10:your-scaling-group-name
# if using autodiscovery
      --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,kubernetes.io/cluster/<ClusterName>
      --v=4
```
