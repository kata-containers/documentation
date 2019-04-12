# cgroup updates in Kata

  * [Background](#background)
  * [Existing Behavior (Kata 1.6):](#existing-behavior--kata-16--)
    + [Behavior Observed using various upper layer tools](#behavior-observed-using-various-upper-layer-tools)
      - [In Docker](#in-docker)
      - [In Kubernetes + Containerd](#in-kubernetes---containerd)
        * [Where are all the vCPUS](#where-are-all-the-vcpus)
        * [Where are the v2-shim, QEMU, and Vhost processes](#where-are-the-v2-shim--qemu--and-vhost-processes)
      - [In Kubernetes  + CRI-O (v1 shim)](#in-kubernetes----cri-o--v1-shim-)
    + [Issues with current implementation](#issues-with-current-implementation)
      - [Accurate usage accounting](#accurate-usage-accounting)
      - [Node stability](#node-stability)
      - [Consistent guaranteed pod behavior](#consistent-guaranteed-pod-behavior)
      - [OOM, unbound CPU utilization](#oom--unbound-cpu-utilization)
  * [Proposed Changes](#proposed-changes)
    + [Summary](#summary)
    + [Details](#details)
      - [Pod Sandbox Cgroup](#pod-sandbox-cgroup)
  * [Alternatives Considered](#alternatives-considered)
    + [Only constrain vCPUs, leaving remaining threads for system reserved](#only-constrain-vcpus--leaving-remaining-threads-for-system-reserved)
  * [Opens](#opens)
    + [static CPU configurations](#static-cpu-configurations)

## Background

With 1.6 release of Kata Containers there are some issues with resource management resulting
in inconsistent behavior. This document descibes the state of 1.6, and a suggested implementation
for 1.7 version of Kata.

Before diving into the gaps and behavior exhibited in Kata Containers, it is important to have
a thorough understanding of how cgroups are leveraged by Kubernetes.  It is both straight forward
and confusing.  An in-depth guide is available for background in [mcastelino's gist](https://gist.github.com/mcastelino/b8ce9a70b00ee56036dadd70ded53e9f).
This may be good content to include in our repository eventually, or as part of the Kata blog series,
but let's leave it out of the scope of this initial documentation.

This document is part of the `cgroup-sprint` GitHub milestone, and can be observed in the milestone's
[GitHub project](https://github.com/orgs/kata-containers/projects/17).


## Existing Behavior (Kata 1.6):

### Behavior Observed using various upper layer tools

To exhibit current behavior, we utilize a simple guaranteed pod description:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guar-runc
spec:
  containers:
  - name: cont-2cpu-400m
    image: busybox
    resources:
      limits:sadf
        cpu: 2
        memory: "400Mi"
    command: ["md5sum"]
    args: ["/dev/urandom"]
  - name: cont-3cpu-200m
    image: busybox
    resources:
      limits:
        cpu: 3
        memory: "200Mi"
    command: ["md5sum"]
    args: ["/dev/urandom"]
```
We'll show the behavior starting with the simplest scenario, Docker, followed by containerd
and CRI-O.

#### In Docker

#### In Kubernetes + Containerd

For each container in the pod, a cgroup is created within the pod cgroup (ie, under `/sys/fs/cgroup/*/kubepod/pod.*/`
for a guaranteed pod). This is not necessary; only a single cgroup which constrains the hypervisor
appropriately is required.

##### Where are all the vCPUS

The vCPUs are placed under the pause container:

```bash
# for i in `ls pod*/**/tasks`; do echo $i && for j in `cat $i`; do pstree -pt $j;done; done;
podf277e232-5ca6-11e9-b514-000d3a6d0876/2692eaedb55f8cfd1b9aadcbc5e3f0ac527cb39ff26d31877f1be5a495b966c1/tasks
podf277e232-5ca6-11e9-b514-000d3a6d0876/6689f72eef2161b85d5d57cb9f4670ae4e08f551d9aeb4b28efb67eb306034d8/tasks
podf277e232-5ca6-11e9-b514-000d3a6d0876/9d17d1c5d075ca42dfefb58ef7b82c8b1b234cc128ebf9332b902b866c0ebed7/tasks
{qemu-system-x86}(24011)
{qemu-system-x86}(24093)
{qemu-system-x86}(24097)
{qemu-system-x86}(24156)
{qemu-system-x86}(24157)
{qemu-system-x86}(24158)
```

In this case, `9d17d1` is the pause, as you can see based on the summation for `cpu.cfs_quota_us`:
```bash
# cat */cpu.cfs_quota_us
300000
200000
500000
```
One drawback of this is that it assumes the existence of a pause container. This is an assumption
based on the implementation of containerd/cri-o.  On the plus side, the cgroup is placed directly under
the pod cgroup, which is created and managed by Kubelet.  Overall, this isn't terrible.

##### Where are the v2-shim, QEMU, and Vhost processes

As shown below, all of these are placed under the containerd.service system.slice.  For `containerd-shim-kata-v2`
this isn't a major concern, as it is not expected to take much resource, and it is pretty closely
coupled to containerd.

QEMU itself and its vhost threads are very problematic.  Depending on the workload, these can consume a
non-negligible amount of resources. Note, in the Kata implementation, these components are purposefuly
not added to the constrained cgroup, the pause container. As a result, they fall under the caller's
cgroup, which in this case is the containerd service.

The process location is determined as follows:

v2-shim:
```bash
root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct# ps -aef | grep containerd-shim-kata | grep -v grep

root     23992     1  0 22:13 ?        00:00:00 /opt/kata/bin/containerd-shim-kata-v2 -namespace k8s.io -address /run/containerd/containerd.sock -publish-binary /usr/local/bin/containerd -id 9d17d1c5d075ca42dfefb58ef7b82c8b1b234cc128ebf9332b902b866c0ebed7 -debug
root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct# grep -ir 23992
system.slice/containerd.service/cgroup.procs:23992
system.slice/containerd.service/tasks:23992
```

qemu:
```bash
root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct# ps -ae | grep qemu
24007 ?        00:37:26 qemu-system-x86

root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct# grep -ir 24007
kubepods/podf277e232-5ca6-11e9-b514-000d3a6d0876/9d17d1c5d075ca42dfefb58ef7b82c8b1b234cc128ebf9332b902b866c0ebed7/cgroup.procs:24007
system.slice/containerd.service/cgroup.procs:24007
system.slice/containerd.service/tasks:24007
```

vhost:
```bash
root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct# ps -aef | grep vhost | grep -v qemu | grep -v grep
root     24010     2  0 22:13 ?        00:00:00 [vhost-24007]

root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct# grep -ir 24010
system.slice/containerd.service/cgroup.procs:24010
system.slice/containerd.service/tasks:24010
```

#### In Kubernetes  + CRI-O (v1 shim)

CRI-O is very similar the containerd, except for where the non-constrained processes end up. Instead
of being called by CRIO directly, kata-runtime is called from a process `conmon`, which is located
in a cgroup under the pod-cgroup. As expected based on prior exapmles, cgroups are ceated for each
container, and the QEMU vCPU threads are placed within the pause containers cgroup.

```bash
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-1b05886a39901ef3a7555796d38dcfaafd8fda929aef223ea576324a4949f9ef/tasks
{qemu-system-x86}(18061)
kata-shim(18207)─┬─{kata-shim}(18213)
                 ├─{kata-shim}(18215)
                 ├─{kata-shim}(18216)
                 ├─{kata-shim}(18217)
                 ├─{kata-shim}(18218)
                 ├─{kata-shim}(18219)
                 ├─{kata-shim}(18220)
                 ├─{kata-shim}(18221)
                 ├─{kata-shim}(18223)
                 ├─{kata-shim}(18224)
                 └─{kata-shim}(18226)
{kata-shim}(18213)
{kata-shim}(18215)
{kata-shim}(18216)
{kata-shim}(18217)
{kata-shim}(18218)
{kata-shim}(18219)
{kata-shim}(18220)
{kata-shim}(18221)
{kata-shim}(18223)
{kata-shim}(18224)
{kata-shim}(18226)
{qemu-system-x86}(18280)
{qemu-system-x86}(18281)
{qemu-system-x86}(18368)
{qemu-system-x86}(18369)
{qemu-system-x86}(18370)
```

The QEMU, vhost, proxy and shim threads, however, are placed under the caller's cgroup, which in this case
is `conmon`, which is a peer of the container cgroups we created. So, the good news is that QEMU, vhost,
etc, are constrained within the pod's cgroup. The bad news is these will be constrained based on the values
associated with conmon:

```bash
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-conmon-1b05886a39901ef3a7555796d38dcfaafd8fda929aef223ea576324a4949f9ef/tasks
conmon(18040)─┬─kata-proxy(18063)─┬─{kata-proxy}(18070)
              │                   ├─{kata-proxy}(18071)
              │                   ├─{kata-proxy}(18072)
              │                   ├─{kata-proxy}(18073)
              │                   ├─{kata-proxy}(18074)
              │                   ├─{kata-proxy}(18075)
              │                   ├─{kata-proxy}(18076)
              │                   ├─{kata-proxy}(18077)
              │                   ├─{kata-proxy}(18286)
              │                   ├─{kata-proxy}(20199)
              │                   ├─{kata-proxy}(20200)
              │                   ├─{kata-proxy}(20201)
              │                   ├─{kata-proxy}(20202)
              │                   ├─{kata-proxy}(20203)
              │                   └─{kata-proxy}(28490)
              ├─kata-shim(18207)─┬─{kata-shim}(18213)
              │                  ├─{kata-shim}(18215)
              │                  ├─{kata-shim}(18216)
              │                  ├─{kata-shim}(18217)
              │                  ├─{kata-shim}(18218)
              │                  ├─{kata-shim}(18219)
              │                  ├─{kata-shim}(18220)
              │                  ├─{kata-shim}(18221)
              │                  ├─{kata-shim}(18223)
              │                  ├─{kata-shim}(18224)
              │                  └─{kata-shim}(18226)
              ├─qemu-system-x86(18058)─┬─{qemu-system-x86}(18059)
              │                        ├─{qemu-system-x86}(18061)
              │                        ├─{qemu-system-x86}(18280)
              │                        ├─{qemu-system-x86}(18281)
              │                        ├─{qemu-system-x86}(18368)
              │                        ├─{qemu-system-x86}(18369)
              │                        └─{qemu-system-x86}(18370)
              └─{gmain}(18042)
{gmain}(18042)
qemu-system-x86(18058)─┬─{qemu-system-x86}(18059)
                       ├─{qemu-system-x86}(18061)
                       ├─{qemu-system-x86}(18280)
                       ├─{qemu-system-x86}(18281)
                       ├─{qemu-system-x86}(18368)
                       ├─{qemu-system-x86}(18369)
                       └─{qemu-system-x86}(18370)
{qemu-system-x86}(18059)
vhost-18058(18060)
```

Two things should happen here:
 * work with CRI-O to determine a more appropriate CPU shares setting for conmon, to avoid impacting the container
 cgroups (in case of runc) or the hypervisor's cgroup (in case of Kata).  See <ADD ISSUE LINK HERE>
 * do not place our IO threads, shim, proxy and QEMU process in conmon

### Issues with current implementation

There are a few major issues that result from the current implementation, and are motivation for design
changes.

#### Accurate usage accounting
The IO pocessing should be charged to the pod performing the IO, not against the system. Without
utilization of a same hierarchical cgroup, this will not be feasible.

#### Node stability

QEMU and its IO threads consume a non-negligible amount of resources. If the memory and CPU utilized is not
constrained, measured and not accounted for, the node will run into CPU and memory pressure unexpectedly.

#### Consistent guaranteed pod behavior

Predictable performance is important for end users. By pushing IO threads into a shared pool, the
achievable performance will be inconsistent.  Even if a user utilizes a `guaranteed` QoS pod, the
performance profile will differ depending on the amount of contention on the system.  Raw unconstrained
performance is important for Kata, but not as important as consistent and predictable behavior.

#### OOM, unbound CPU utilization

Memory limits are enforced, not requests. Until [Pod Overhead KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/20190226-pod-overhead.md)
is added, users or admission controllers need to provide higher limits.

In the case of CRI-O, the memory is being charged to conmon which is bounded by pod limits. As a result,
the workload can still be OOMed. This is okay from a node stability point of view, but not from a pod stability
point of view. This behavior assumes we are called by conmon and that they are constrained appropriately. Luckily,
this is reasonably correct from a memory point of view. I/O bound workloads will exhibit sub-optimal performance
due to the CPU constraints applied to conmon (where the io threads run).

For containerd, the memory is being charged to containerd, which is basically unbounded.  This is bad for
node stability, as the `pod` is essentially unbounded.

## Proposed Changes

### Summary
 * Pause cgroup cpu shaes should be setup correctly.
 * Do not create container cgroups on the host. Instead, create a pod sandbox group that is entirely managed by Kata
 * Move the QEMU threads into the sandbox cgroup

With these changes, performance and constraints for a pod is consistent. This constraining change will be
more restrictive relative to existing design.

The overheads associated with running a sandbox should be accounted for explicitly, and at the pod level.
Once the Pod Overhead KEP is available, this should become a part of RuntimeClass, applied to pods which
utilize the applicable RuntimeClass.  See [Pod Overhead KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/20190226-pod-overhead.md)
for more details.

### Details

#### Pod Sandbox Cgroup
 * The pod sandbox cgroup should always be the summation of all container group resources.
 * In the case of cri-o where it creates other cgroups for conmon, they will be siblings.
 * The conmon cgroups today are rather large, so if conmon goes wild there is a possibility
 the workload will get fewer resources. But it will not introduce any other side effects.

## Alternatives Considered

### Only constrain vCPUs, leaving remaining threads for system reserved

This will, in some cases, provide improved performance.

## Opens

### static CPU configurations

If static CPU policies are introduced, the end user will assign CPUs to a specific container within the pod.  Running
IO threads on this CPU may not necessarily be desirable, compared to the users expectations.

Long term (ie, with RuntimeClass augmented to handle pod overheads), we should create a seperate `cpuset` cgroup,
`kata-sandbox-vcpus`, alongside the standard sandbox cgroup, `kata-sandbox`. These would be siblings underneath the
pod cgroup, in the kubernetes case.  vCPU threads will be placed under `kata-sandbox-vcpus`, which will be updated
to use the CPUset suggested for the workload.  The remaining threads will be placed under `kata-sandbox`, utilizing
the remaining non-claimed CPUs (problem: is this even possible to determine?). The CPU cgroups will be managed
as normal. The non-vCPU threads will be limited to the CPU utilization provided by the pod overhead, in this case.

In the short term, non-vCPU threads will need to share the cpuset, and the end-user will need to add additional
CPUs for overhead, if desired.
