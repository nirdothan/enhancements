# VEP #NNNN: Seamless TCP migration and vhost-user support with passt 

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue created, which links to VEP dir in [kubevirt/enhancements] (not the initial VEP PR)

## Overview

This proposal outlines the enhacements that have been implemented in passt and how Kubevirt will be enhanced to intergare with passt in order be able to benefit from them.

## Motivation

At the moment kubevirt does not support all networking elementary requirements expected by our users:
- Seamless network connectivity on migration.
- Observability/Service Mesh support.
- High Throughput (on par with Tap).
- Support all IP network protocols.

Bridge binding does not support observability and services meshes, Masquerade hides the “real” IP from the user and passt does not support seamless migration and its performance is not on par with tap devices.
The gaps of passt can be overcome by development of multiple features, such that it would support all required features and become the optimal network binding for kubevirt.    
In order to integrate with the enhanced release of passt, some adaptations will be required in kubevirt.


## Goals

A working deployment and intgration of kubevirt with passt supporting seamless live migration of TCP connections and vhost-user based network binding.

## Non Goals

- Support network protocols other than TCP/UDP/ICMP (ARP/DNS).
- Adress the design of passt itself. 

## Definition of Users
- Cluster Admins
- VM/Namespace owners
- Guest VM Users - are most likely not aware of their network binding


## User Stories

- As a Guest VM user, I want to have optimal network performance, on par with bridge and masquerade, so that my virtualized applications would perform well.   
- As a Cluster Admin I’d like to use observability and service meshes in order to improve operability, and enjoy service mesh features (monitoring, encryption, flexible load balancing and etc)
- As a Cluster Admin I need to serve my users with seamlessly migratable VMs, so that I can upgrade my infrastructure without affecting business continuity. 
- As a Cluster Admin in a customized environment, I want to use a custom namespace to deploy the passt network-attachment-defintion CR, in order to align with the other common kubevirt infrastructure.   


## Repos

- https://github.com/kubevirt/kubevirt
- https://github.com/kubevirt/hyperconverged-cluster-operator
- https://github.com/kubevirt/user-guide
- https://github.com/kubevirt/cluster-network-addons-operator


## Design

### Set TCP_REPAIR socket option
During migration, active TCP sessions need to be transferred from one instance of passt to another running on a different machine. However, the new instance cannot simply "hijack" the connection, even if it knows all the necessary details (IPs, ports, sequences). Instead, a privileged operation is required using the `TCP_REPAIR` socket option. This operation must be performed on both the source and target sockets to enable session handover.

Since passt runs in the unprivileged Linux context of virt-launcher, it lacks the necessary `CAP_NET_ADMIN` capability to perform this operation. To work around this, passt relays the operation to a privileged helper running in virt-handler.

#### The Role of passt-repair
The helper program responsible for handling `TCP_REPAIR` is called passt-repair.[C program](https://passt.top/passt/tree/passt-repair.c) that is distributed as part of the passt package.
How passt-repair Works:
- Executed as a **one-off process**, running separately for both the **migration source** and **migration target**.
- Establishes communication with the passt server over a named Linux Domain Socket.
- Signals its availability and waits for a request from passt.
- Once a request is received:
  - Sets or unsets the `TCP_REPAIR` socket option as required.
  - Sends a completion notification back to passt.
This communication method is similar to the existing mechanism used between virt-handler and virt-launcher.

#### Integration with virt-handler

To support this process, virt-handler will be modified to invoke `passt-repair` in a dedicated goroutine. This goroutine will run to completion or timeout, ensuring the migration process does not hang indefinitely.

The call to `passt-repair` will be added in two locations within virt-handler. 
It will be conditioned by the VMI having an interface with `passt` network binding, and an enabled feature gate.

1. Before migration starts (on the source node), in [vmUpdateHelperMigrationSource](https://github.com/kubevirt/kubevirt/blob/release-1.5/pkg/virt-handler/vm.go#L2714) , to set the TCP_REPAIR socket option.
2. Before migration finalizes (on the target node), in [finalizeMigration](https://github.com/kubevirt/kubevirt/blob/release-1.5/pkg/virt-handler/vm.go#L2714), to remove the TCP_REPAIR socket option.

In addition, the `passt-repair` binary needs to be added to the virt-handler container image. The passt rpm consists if this binary. The simplest solution is to install the rpm in the image.
A more secure and storage efficient solution would be to extract only the passt-repair binary from the rpm and install it standalone.

### Implement vhost-user support in passt sidecar
vhost-user is a protocol that enables high-performance communication between virtual machines and user-space processes in a host operating system. It is commonly used to offload VirtIO device emulation from the hypervisor (e.g., QEMU) to a faster, dedicated process, often implemented in user space, such as DPDK or passt.
This approach improves I/O performance by allowing zero-copy mechanisms and reducing context-switching overhead. 
In addition, it is required for passt to support TCP seamaless migration. 

vhost-user needs to be enabled in the DomainXML. As such small code changes are required in passt sidecar code. 
The format is described [here](https://libvirt.org/formatdomain.html#vhost-user-connection-with-passt-backend).

### Deployment of passt dependencies 
The deployment of passt CNI and the cluster-wide network-attachement-defintion CR associated with the passt binding will be carried out by CNAO.
The passt plugin sidecar container will remain unchanged, i.e. It will reside and be built in the kubevirt/kubvirt repo.  
The CNAO engine will use the following entities to automatically reconcile the CNI deployment daemonset and NAD:

#### NetworkAddonsConfigs API
The CNAO CRD spec will be enhanced to add another field passt of type object.
This will indicate to CNAO to build and deploy the passt CNI and its associated network-attachment-defintion CR. 

Example:
```yaml
apiVersion: networkaddonsoperator.network.kubevirt.io/v1
kind: NetworkAddonsConfig
metadata:
  name: cluster
spec:
  passt{}
```

### passt cni
The passt-cni process is called by `Multus` when the prinary network interface is setup or removed. Its role is simply to run 2 sysctl command:
1. Allow binding to all ports starting form 0.
2. Set ping group range to virt-launcher user id.

The codebase, under the [passt-binding directory](https://github.com/kubevirt/kubevirt/tree/release-1.5/cmd/cniplugins/passt-binding), 
will be enhanced to consist of a `Dockerfile` that will build and publish the container image. 
It will also consist of yaml manifests for a daemon set that will deploy the CNI into nodes.
Those artifacts alredy exist under the IPAM Controller repository, and can be moved as-is from their [current location](https://github.com/kubevirt/ipam-extensions/tree/v0.1.10-alpha/passt).
In addition, this repo will also include a manifest to apply a `network-attachement-definition` CR, named `netbindingpasst`, pointing to the CNI. 

The NAD will be formatted as follows:
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: netbindingpasst
  namespace: {{ .namespace }}
spec:
  config: '{ "cniVersion": "0.3.1", "name": "netbindingpasst", "plugins": [{"type":
    "kubevirt-passt-binding"}]}'
```
**Note**: No changes are expected in the passt sidecar container.


### passt cni repo registration in CNAO
The passt-cni will be registered in [CNAO components.yaml](https://github.com/kubevirt/cluster-network-addons-operator/blob/main/components.yaml) under the kubevirt/kubevirt repository and a bump script will be added to clone/bump the repo into CNAO data directory.
CNAO will deploy the passt-cni daemonset and the `netbindingpasst` network-attachement-definition CR mentioned above.
The default deployment namespace for the NAD will be `default`, however, in specific environment setups CNAO has the logic to deploy to a different dedicted namespace.


### Network binding registration by HCO
In order to support passt based VMs, HCO needs to register it.
The registration function is already [present](passtUserDefinedNetworkBinding). 
The sidecar image tag will be updated and memory reduced to 250Mi. 
```yaml
configuration:
  network:
    binding:
      l2bridge:
        domainAttachmentType: managedTap
        migration: {}
      passt:
        computeResourceOverhead:
          requests:
            memory: 250Mi
        migration: {}
        networkAttachmentDefinition: default/netbindingpass
        sidecarImage: 
          Registry: quay.io/kubevirt/network-passt-binding:<tag/sha>
```

In addition HCO, will extend the population of the NetworkAddonsConfig (CNAO) cluster CR to include the passt object.
See example below.


## API Examples

A passt object will be added to NetworkAddonsConfig CRD.

```yaml
apiVersion: networkaddonsoperator.network.kubevirt.io/v1
kind: NetworkAddonsConfig
metadata:
  name: cluster
spec:
  passt: {}
  ```

## Alternatives

TCP live migration requires a privilged action on the part of passt (TCP_REPAIR socket option). The most strighforward approach is to grant passt with the required capabilities (CAP_NET_ADMIN).
In order to do so, Ambient capbilities can be applied to the passt binary file, similar to the [way that they're applied to virt-launcher](https://github.com/kubevirt/kubevirt/blob/release-1.5/cmd/virt-launcher-monitor/virt-launcher-monitor.go#L176).
The issue with that is that the contianer runtime drops all capabilities from the [Bounding set](https://man7.org/linux/man-pages/man7/capabilities.7.html#:~:text=for%20the%20thread.-,Bounding,-(per%2Dthread%20since)), and the only way to get them back is to add them in the container SecurityContext as we do for CAP_NET_BIND_SERVICE in [virt-laucher](https://github.com/kubevirt/kubevirt/blob/release-1.5/pkg/virt-controller/services/rendercontainer.go#L284).
Since virt-launcher compute container [drops the root user](https://github.com/kubevirt/kubevirt/blob/release-1.5/pkg/virt-controller/services/template.go#L360) which the image was built with, there's no way for another process to obtain the capability granted to passt, maing this solution secure.
However,kubevirt follows the `Restricted` policy in the kubernetes [Pod Security Standard](https://kubernetes.io/docs/concepts/security/pod-security-standards/#os-specific-policy-controls) and CAP_NET_ADMIN is not listed as allowed.



## Scalability

- passt currently has a memory footprint overhead of 250Mi per VM. Users running VMs at scale should take it into consideration, and revert to other network bindings if that's a concern.
- VMs with a large number of TCP connections, will take longer to live migrate. That's becuase sockets are handled in a serial fasion, each one separatly sent back and forth to passt-repair for handling. 
- The number of concurrent live migration per node is already limited and bound by the number of virt-handler worker goroutines. 

## Update/Rollback Compatibility

Live migration of passt VMs from nodes running older versions of Kubevirt to an upgraded kuebvirt environment will fail.
Rollback of passt migration functionality (in case system stability is impacted) will be possbile by disablement of the feature gate. It is expected that passt would still work, but live migrations would disconnect TCP connections.

## Functional Testing Approach

The existing e2e networking tests cover basic connectivity using passt, those should remain sufficient with vhost-user support.
For seamless connectivity migration, dedicated e2e tests will be developed to assure TCP session disconnection occurs during live migration.

## Implementation Phases

Much of the work can be done in parallel:
### virt-handler track
- Implement calls to passt-repair.

### passt sidecar track
- Implement vhost-user format.

### HCO Track
- Implement passt object in NetworkAddonsConfig.
- Bump passt sidecar image.

### CNAO track
- Implement passt-cni build in kubevirt/kubevirt repo.
- Implement deployment of network-attachement-defnition CR and cni daemonset.

Following competion of virt-handler and passt-sidecar e2e tests will be enhanced.

## Feature lifecycle Phases
passt is currently a network binding plugin and its use is optional.

### Alpha

`passt live migration` will be feature gated. Users will have to opt in.
virt-handler will not run `passt-repair` if the FG is not enabled.

passt-sidecar will not condition the FG and will populate DomainXML in vhost-user mode regadless of FG.
There's no justification for kubevirt to maintain the older format, and the risk is much lower compared to live-migration.

### Beta

Perhaps after one or two releases, when we are confident that the feature is working as expected, move to beta.

### GA

GA once the feature has been running in production without issues.
passt is currently a network binding plugin, and for GA we may decide to move it into kubevirt core.
