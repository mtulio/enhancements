---
title: aws-compute-edge-machineset
authors:
  - "@mtulio"
reviewers:
  - "@rvanderp3"
  - "@patrickdillon"
  - "@elmiko"
  - "@JoelSpeed"
  - "@sdodson"
  - "@wking"
approvers:
  - "@patrickdillon"
  - "@JoelSpeed"
  - "@sdodson"
creation-date: 2022-08-18
last-updated: 2022-08-18
status: ?
superseded-by: []
tracking-link:
  - https://issues.redhat.com/browse/SPLAT-636
  - https://issues.redhat.com/browse/RFE-2782
---

# Allow Customer-Provisioned AWS MachineSets in the edge Locations

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

The edge adoption is increasing and providers are launching many offerings that fill
that area. AWS launched the Local Zones and the Wavelength zones to extend the VPC network
to the metropolitan area from big centers, and for Radio Access Network, respectively.
Both Local Zones and Wavelength has limited services and resources, for example, the NLB
is not available to the remote location, and the instance price is more expensive.

As both services provide capabilities to extend the workloads running within the VPC to
locations closer to the users, it provides much lower latency in the application that is sensitive to,
being decisive to some businesses, like recommendation systems.

Cluster administrators teams currently can extend the OpenShift nodes to the Local Zones in Day-2 operations
by creating the subnet in the specific availability zone, then create the MachineSet correctly
to Machine, controllers can create EC2 Instances and run custom workloads. However, as mentioned,
the resources are more expensive, and the latency within the Local Zone and the normal zones in the
parent region could be higher, thus setting the correct taints to allow users to choose what workloads run
on the edge could be preferable.

Personas should be able to specify the availability zone on the edge location to place compute nodes
pools when installing OpenShift, which will run minimal OpenShift services.

The installer must allow the user to create Worker nodes in the Local Zones, network edge of AWS Cloud Provider.

The implementation was divided in three phases:
- Phase 0: Research - understand the Local Zones using OCP, create use case and support installing in existing VPC
- Phase 1: Install a cluster in user provided network. Introduce on installer a new compute pool named `edge`, creating MachineSet manifests for each subnet provided on `platform.aws.subnets` for Local Zones. The Machine will be protected with taints to avoid running common workloads.
- Phase 2: The installer should automate the network resource creation when the user provides zone names with type `local-zone` on the `platform.aws.zones` or `compute[?name=="edge"].platform.aws.zones`

The results of Phase 0 guided to create the later steps. The Phase 1 and 2 are covered by this enhancement.

## Motivation

### Goals

> Open Question: we can reuse the existing flow to create a cluster in the existing VPC, looking at the type of subnet specified on the field `platform.aws.subnets`, then create the MachineConfig for that new location. But it will be much logic inside the installer. Introducing a new "compute type" with that standard (taint:NoSchedule, node-role=edge, etc) could be more clear for users, IMO.

- Introduce a new compute type named `edge` on the install-config.yaml

```yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: hybrid-cluster
compute:
- name: edge
  platform:
    aws:
      rootVolume:
        type: gp2
        size: 120
      zones:
      - us-east-1-nyc-1a
platform:
  aws:
    region: us-east-1
    subnets:
    - subnetID-us-east-1a
    - subnetID-us-east-1b
    - subnetID-us-east-1c
    - subnetID-us-east-1-nyc-1a
pullSecret: '{"auths": ...}'
sshKey: ssh-ed25519 AAAA...
```

- For each `zones` specified on the `edge` compute type, select the subnetID matching it on `platform.aws.subnets`, then create one MachineSet for each one according to the platform attributes specified on the `platform.aws.*`. The node should be added the label `node-role.kubernetes.io/edge`, and the taints to `NoSchedule` workloads on that resources.

- Fail when:
    - if any availability zone specified in `compute[.name=='edge'].platform.aws.zones` is not part of the "edge" types (AWS API `ZoneType`: `local-zone` or `wavelength-zone`)
    - if any availability zone specified in `compute[.name=='worker'].platform.aws.zones` is not part of the parent region (AWS API `ZoneType`: `availability-zone`)
    - if any subnetIDs specified in `platform.aws.subnets` is not a part of availability zones specified in the `compute[.name=='edge'].platform.aws.zones`

### Non-Goals

- Create the network resources on the edge locations. That implementation will be done in the second stage, when the installer will fully automate the creation of resources on the availability zones specified on the `compute[.name=='edge'].platform.aws.zones`.

- Create any resource for Wavelength. Although Wavelength is also running on the edge, it introduces a new resource called Carrier Gateway, which provides internet connectivity to the compute nodes on the edge locations/RANs. The Wavelength zones support only public subnets.

## Proposal



To achieve this, each phase must satisfy the requirements below.

Phase 1:

- the installer must allow one item of machine pool type `edge` (`compute[?name=="edge"]`)
- the installer must extract Local Zones subnets from resource IDs on `platform.aws.subnets`, creating one MachineSet manifest for each zone
- the `compute[?name=="edge"]` must create MachineSets manifests on Local Zone subnets with labels `node-role.kubernetes.io/edge=''` and `localtion=local-zone`, and taint `NoSchedule` matching the label `node-role.kubernetes.io/edge`
- the MachineSet should be created when any Local Zone subnet is added on `platform.aws.subnets`, even if compute pool was not defined on `compute[?name=="edge"]`
- when provided, all the zones on `compute[?name=="edge"].platform.aws.zones` must have one resource ID on `platform.aws.subnets`

> Question 1 - Local Zones scope
- only `public` subnets are allowed for Local Zones subnets added on `platform.aws.subnets`
- the Local Zone type should not be allowed on `worker` compute pool `compute[?name=="worker"].platform.aws.zones`

- the default or user-provided (`compute[?name=="edge"].platform.aws.type` or `platform.aws.defaultMachinePlatform.type`) instance type for the machine pool `compute[?name=="edge"]` should be validated on the EC2 offering for the target availability zone

> Note: Currently the resource types are limited. The intersection between zones are c5d.2xlarge and gp2
- the default `edge` machine pool instance type must be common in the current availability zone offerings
- the default `edge` machine pool EBS type must be common in the current availability zone offerings

Phase 2:
> Question 1 - Local Zones scope
- the installer must create public subnets on Local Zones for each item provided on `compute[?name=="edge"].platform.aws.zones`, and one respective MachineSet for each zone
- by default, the installer will not create any resource in Local Zone


The installer allows users to provide the compute type `edge` that should
automate the creation of MachineSet manifests setting the correct attributes
to avoid running normal workloads in remote locations.

The installer validates the assumptions about the networking setup.

Infrastructure resources that are specific to the cluster and resources owned by the cluster will be created by
the installer. So resources like security groups, IAM roles/users, ELBs, S3 buckets, and ACL on s3 buckets remain cluster-managed.

The infrastructure resources owned by the cluster continue to be clearly identifiable and distinct from other resources.

Destroying a cluster must make sure that no resources are deleted that didn't belong to the cluster. Leaking
resources are preferred over any possibility of deleting non-cluster resources.

### User Stories

- As an administrator, I would like to specify availability zones on the "edge" location, Local Zone, when installing a cluster 
in existing VPC, so I can use the MachinePool created on Day-0 to deploy user-provided workloads delivering single-digit latency to my end users.

- As a company with hybrid cloud architecture, I would like to process real-time machine learning models in AWS specialized instances closer to my application.

### Implementation Details/Notes/Constraints

#### Resource provided to the installer

The users provide a compute node structure(1) with the name of `edge`, with the availability zones(2) to be created in the MachineSets, alongside the subnetIDs(3) of the resources previously created, including the Local Zone subnet ID(4).

```yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: hybrid-cluster
compute:
- name: edge  (1)
  platform:
    aws:
      rootVolume:
        type: gp2
        size: 120
      zones: 
      - us-east-1-nyc-1a  (2)
platform:
  aws:
    region: us-east-1
    subnets: (3)
    - subnetID-az1
    - subnetID-az2
    - subnetID-az3
    - subnetID-localzone-az1 (4)
pullSecret: '{"auths": ...}'
sshKey: ssh-ed25519 AAAA...
```


#### Resources created by the installer

The installer will continue to create (vs. the fully-IPI flow): AMI, IAM Roles, Load Balancers, etc

The installer will create the new MachineSet manifest for the edge pool. The MachineSet
will be as follow:

`99_openshift-cluster-api_edge-machineset-01.yaml`
```yaml
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  labels:
    machine.openshift.io/cluster-api-cluster: ${CLUSTER_ID}
  name: ${CLUSTER_ID}-edge-${ZONE_NAME}  (1)
  namespace: openshift-machine-api
spec:
  replicas: 1
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: ${CLUSTER_ID}
      machine.openshift.io/cluster-api-machineset: ${CLUSTER_ID}-edge-${ZONE_NAME} (1)
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: ${CLUSTER_ID}
        machine.openshift.io/cluster-api-machine-role: edge  (2)
        machine.openshift.io/cluster-api-machine-type: edge  (2)
        machine.openshift.io/cluster-api-machineset: ${CLUSTER_ID}-edge-${ZONE_NAME}
    spec:
      metadata:
        labels: (3)
          location: local-zone (4)
          node-role.kubernetes.io/edge: "" (5)
      taints: (5)
        - key: node-role.kubernetes.io/edge
          effect: NoSchedule
      providerSpec:
        value:
          ami:
            id: ${AMI_ID}
          apiVersion: awsproviderconfig.openshift.io/v1beta1
          blockDevices:
          - ebs: (6)
              volumeSize: 120
              volumeType: gp2
          credentialsSecret:
            name: aws-cloud-credentials
          deviceIndex: 0
          iamInstanceProfile:
            id: ${CLUSTER_ID}-worker-profile
          instanceType: ${INSTANCE_TYPE}
          kind: AWSMachineProviderConfig
          placement:
            availabilityZone: ${ZONE_NAME} (7)
            region: ${CLUSTER_REGION}
          securityGroups:
          - filters:
            - name: tag:Name
              values:
              - ${CLUSTER_ID}-worker-sg
          subnet:
            id: ${SUBNET_ID} (8)
          publicIp: true  (9)
          tags:
          - name: kubernetes.io/cluster/${CLUSTER_ID}
            value: owned
          userDataSecret:
            name: worker-user-data
```

- 1) the `worker` part of the MachineSet name will be replaced with `edge`.
- 2) (do we need that?) MachineSet labels for the `edge` compute nodes
- 3) New Labels are added to the node.
- 4) (do we need that? optional but could be useful when having other edge types) `location` label could assume two values `local-zone` or `wavelength` (future)
- 5) The `NoSchedule` taint will be added for each node with the tag `node-role.kubernetes.io/edge=''`
- 6) Local Zones have limited resource types. Currently, only `gp2` EBS type is supported. (IPI default is gp3)
- 7) The `ZONE_NAME` will be the availability zone name for Local Zones (ex: `us-east-1-nyc-1a`)
- 8) The `subnet.id` should be selected from the install-config.yaml parameter `platform.aws.subnets` 
- 9)The `PublicIP` should be true for Wavelength and Local Zones as it does not support Nat Gateways

#### Destroying cluster

Destroying a cluster will remain mostly unchanged:
- hinging on the `kubernetes.io/cluster/<cluster-infra-id>: owned` tag;
- removes the `kubernetes.io/cluster/<cluster-infra-id>: shared` tag from resources that have it. These resources are not deleted.

#### Limitations

Phase 1:

- The `edge` Machine Pool (`compute[?.name=="edge"]`) is valid only for AWS Platform


- The `edge` MachineSet will be created only for AWS platform on the Local Zones, the Parent and Wavelength zones specified on the Compute pool .

- KCM has a bug[1] in the subnets discovery on AWS when setting up a Load Balancer. When
  installing a cluster in an existing VPC. The bug will not affect the installations in existing VPCs
  but could be a limitation when the installer automates the subnet creation (not covered in this document).

- OpenShift Clusters can't be deployed entirely on the Local Zones, due to the limited services available there,
  and also required for a fully integrated AWS support on the default IPI, like NLB, Nat Gateways, etc.

- All subnetsLocal Zones does not support Nat Gateways. The subnets on Local Zones can use the
  Nat Gateways from the parent region's availability zones as a gateway to the internet.

[1] https://bugzilla.redhat.com/show_bug.cgi?id=2105337

### Risks and Mitigations

#### Isolation between clusters

Deploying OpenShift clusters to a pre-existing network has the side effect of reducing the isolation of cluster services.

## Design Details

### Test Plan

> ToDo: We can use the steps defined here (CloudFormation templates) to create the test plan https://github.com/mtulio/mtulio.labs/blob/article-ocp-aws-lz/docs/articles/ocp-aws-local-zones-day-0.md

### Graduation Criteria

Not applicable

### Upgrade / Downgrade Strategy

Not applicable

### Version Skew Strategy

Not applicable.

## Implementation History

## Drawbacks

> Is it still valid?

> Customer-owned networking components mean the cluster cannot automatically change things about the networking during an upgrade.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed

Not applicable.
