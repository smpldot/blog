# New Features in Kubernetes 1.28

## Introduction

Kubernetes 1.28 is finally here, bringing with it 44 new or improved enhancements! This release introduces several major features, including built-in support for sidecar containers, job optimizations, and enhanced proxies. These new capabilities are poised to enhance the performance, efficiency, and security of your Kubernetes clusters.

## Sidecar Containers

Sidecar containers are a widely adopted pattern for extending the functionality of Kubernetes pods. They are often utilized for tasks like implementing service meshes, collecting metrics, and fetching secrets. However, implementing sidecar containers hasn't always been straightforward.

Until now, Kubernetes lacked a way to explicitly designate a container as a sidecar container. This meant that sidecar containers could be terminated before the primary container finished its task, or they could continue running after a job was completed.

With Kubernetes 1.28, a new feature has been introduced: the `restartPolicy` field in the specification for init containers. This field allows you to explicitly label a container as a sidecar container. Kubernetes now treats sidecar containers differently than regular containers:

- The kubelet won't wait for the sidecar container to complete; it will only wait for the startup process to finish.
- The sidecar container will be automatically restarted if it fails during startup, unless the pod's restart policy is set to `Never` in which case the entire pod will fail.
- The sidecar container will continue running as long as the main container is operational.
- The sidecar container will be terminated only when all regular containers have completed their tasks. This ensures that sidecar containers won't hinder job completion after the primary container has finished.

Here's an example of how to use the `restartPolicy` field to create a sidecar container:

```yaml
kind: Pod
...
spec:
  initContainers:
  - name: vault-agent
    image: hashicorp/vault:1.14.2
  - name: istio-proxy
    image: istio/proxyv2:1.18.2
    args: ["proxy", "sidecar"]
    restartPolicy: Always
  containers:
  ...
```

## Optimizations for Jobs

Jobs in Kubernetes have received significant attention in this release. Kubernetes Jobs allow the initiation of a large number of repetitive parallel tasks, making them ideal for machine learning workloads. This release brings further improvements to tools like Jobs.

We've already discussed the [Sidecar Containers](https://github.com/kubernetes/enhancements/issues/753) feature. This feature introduces enhancements for job users, ensuring that sidecars no longer impede job completion.

Enhancements such as [Retriable and non-retriable Pod failures for Jobs](https://github.com/kubernetes/enhancements/issues/3329) and [Backoff Limit Per Index For Indexed Jobs](https://github.com/kubernetes/enhancements/issues/3850) offer finer granularity for handling job failures. Some failures are temporary or expected, and handling them differently can prevent entire jobs from failing.

Lastly, [Allow for recreation of pods once fully terminated in the job controller](https://github.com/kubernetes/enhancements/issues/3939) provides more control options for handling completed jobs, reducing edge cases and race conditions.

Kubernetes Jobs continue to evolve with each release, reflecting the community's keen interest, especially in machine learning applications.

## Kubernetes Packages on Community Infrastructure

The Kubernetes project is shifting its package repositories to [community-owned infrastructure](https://github.com/kubernetes/enhancements/issues/1731). This move aims to decouple the project from Google's infrastructure, enhancing its resilience and sustainability.

The new repository, `pkgs.k8s.io`, joins existing repositories `apt.kubernetes.io` and `yum.kubernetes.io`. The older repositories will be deprecated in the future. The Kubernetes team will publish instructions on migrating to the new repositories around the time of the release.

## Support for User Namespaces in Pods

User namespaces are a Linux feature that enables running processes in pods with different user privileges than the host. This enhances Kubernetes cluster security by limiting the potential harm from a compromised pod.

For instance, you could run a pod with the root user inside the container but as an unprivileged user on the host. Consequently, if the pod is compromised, the attacker only gains the privileges of the unprivileged user.

To utilize user namespaces in Kubernetes, you need:
- At least Linux Kernel 6.3.
- A filesystem supporting idmap (e.g., ext4, btrfs, xfs, fat).
- A container runtime with idmap support (e.g., CRI-O 1.25, containerd 1.7).

## Reserve NodePort Ranges for Dynamic and Static Allocation

When employing a `NodePort` service, you might want to statically allocate a port by explicitly specifying it within the `service-node-port-range` (default 30000-32767).

However, you may encounter situations where your desired port has already been dynamically assigned to another service.

This new feature reserves the first ports in the `service-node-port-range` for static allocations. The number of reserved ports is determined by the formula `min(max(16, nodeport-size / 32), 128)`, resulting in a range of 16 to 128 reserved ports.

For example, with the default range (30000-32767), it reserves 86 ports. The range 30000-30085 is allocated for static allocations, leaving the remainder for dynamic allocations.

## Rolling Upgrades

Three new enhancements improve the reliability and reduce downtime during upgrades. This is a significant quality-of-life improvement for administrators who fear putting their apps into maintenance mode.

With [Unknown Version Interoperability Proxy](https://github.com/kubernetes/enhancements/issues/4020), rolling upgrades of cluster components are handled more effectively. Traffic is redirected to a peer that is of a known compatible version. The new version ensures compatibility with both the old and new versions. It's a bit like a traffic cop directing cars around an accident on the road.

[Kube-proxy improved ingress connectivity reliability](https://github.com/kubernetes/enhancements/issues/3836) introduces the ability to specify the number of pods you want to retain when you drain a node. This is useful in situations where the node has a lot of empty space and you'd prefer to migrate the pods elsewhere before you evict them.

Finally, [Proxy Terminating Endpoints](https://github.com/kubernetes/enhancements/issues/1669) allows you to retry control plane upgrades that have previously failed. Before this feature, you'd need to manually revert changes to the configuration files.

## Kube-proxy Improved Ingress Connectivity Reliability

This improvement introduces new features to kube-proxy, enhancing its ability to manage connection health.

Key features include:
1. When a node is in the process of terminating, kube-proxy will no longer immediately terminate all connections, allowing them to gracefully close.
2. A new `/livez` endpoint is added, enabling vendors and users to define a `livenessProbe` to assess kube-proxy's health. This method is more precise than simply checking whether the node is terminating.
3. The enhancement provides guidelines for vendors to implement these health checks, although standardization is not the primary goal at this stage.

## Consistent Reads From Cache

This enhancement aims to boost the performance of certain API requests, such as GET or LIST, by reading information from the watch cache of etcd rather than directly from etcd itself.

This improvement is made possible through the utilization of `WatchProgressRequest` in etcd version 3.4 and beyond. It promises substantial performance and scalability enhancements, particularly in large deployments, such as clusters with over 5,000 nodes.

For more in-depth technical details and insights into data consistency assurance, refer to [Kubernetes Enhancement Proposal (KEP)](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache/README.md).

## Conclusion

Kubernetes 1.28 brings a wealth of improvements and new features, with built-in support for sidecar containers taking center stage. These changes simplify the use and management of sidecar containers, making them more powerful and flexible tools for enhancing applications in the Kubernetes environment.

The future of Kubernetes promises even more innovation and improvement, solidifying its position as a robust platform for managing containerized applications.
