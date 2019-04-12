# cgroup updates in Kata

## Background

With 1.6 release of Kata Containers there are some issues with resource management resulting in inconsistent
behavior.  This document descibes the state of 1.6, and a suggested implementation for 1.7 version of Kata.

## Existing Behavior (Kata 1.6):

### In Docker
### In CRI-O
### In Containerd


## Proposed Changes

### Alternatives Considered


## Opens 


Cgroup Updates in Kata



# Backup / to be handled:


## Containerd Handling Today

The hierarchy and cgroup handling seems pragmatic in the case of containerd. The
container cgroups are currently placed under the podcgroup.

Output from containerd guaranteed pod, with two containers:

```
kata@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct/kubepods$ tree pode3bb46f0-5c99-11e9-903b-000d3a6d0876/
pode3bb46f0-5c99-11e9-903b-000d3a6d0876/
├── 47c6122142c76db9f2929c23e8693c641559cb6588fbf5e2f5c21623f5af2fd1
│   ├── cgroup.clone_children
│   ├── cgroup.procs
│   ├── cpu.cfs_period_us
│   ├── cpu.cfs_quota_us
│   ├── cpu.shares
│   ├── cpu.stat
│   ├── cpuacct.stat
│   ├── cpuacct.usage
│   ├── cpuacct.usage_all
│   ├── cpuacct.usage_percpu
│   ├── cpuacct.usage_percpu_sys
│   ├── cpuacct.usage_percpu_user
│   ├── cpuacct.usage_sys
│   ├── cpuacct.usage_user
│   ├── notify_on_release
│   └── tasks
├── a18373fc91cf91d9b362a536afaecd0712a858bb992777d956948911a6d3b248
│   ├── cgroup.clone_children
│   ├── cgroup.procs
│   ├── cpu.cfs_period_us
│   ├── cpu.cfs_quota_us
│   ├── cpu.shares
│   ├── cpu.stat
│   ├── cpuacct.stat
│   ├── cpuacct.usage
│   ├── cpuacct.usage_all
│   ├── cpuacct.usage_percpu
│   ├── cpuacct.usage_percpu_sys
│   ├── cpuacct.usage_percpu_user
│   ├── cpuacct.usage_sys
│   ├── cpuacct.usage_user
│   ├── notify_on_release
│   └── tasks
├── cgroup.clone_children
├── cgroup.procs
├── cpu.cfs_period_us
├── cpu.cfs_quota_us
├── cpu.shares
├── cpu.stat
├── cpuacct.stat
├── cpuacct.usage
├── cpuacct.usage_all
├── cpuacct.usage_percpu
├── cpuacct.usage_percpu_sys
├── cpuacct.usage_percpu_user
├── cpuacct.usage_sys
├── cpuacct.usage_user
├── f90eb972d44634f85726552a818578c4f28085f9a2f755fb8b18c402fe9cef6d
│   ├── cgroup.clone_children
│   ├── cgroup.procs
│   ├── cpu.cfs_period_us
│   ├── cpu.cfs_quota_us
│   ├── cpu.shares
│   ├── cpu.stat
│   ├── cpuacct.stat
│   ├── cpuacct.usage
│   ├── cpuacct.usage_all
│   ├── cpuacct.usage_percpu
│   ├── cpuacct.usage_percpu_sys
│   ├── cpuacct.usage_percpu_user
│   ├── cpuacct.usage_sys
│   ├── cpuacct.usage_user
│   ├── notify_on_release
│   └── tasks
├── notify_on_release
└── tasks

```

I suggest that we *do not* create cgroups with names identical to the containers, and instead create a single cgroup, sized appropriately, to constrain our hypervisor.


## CRI-O

`insert example here?
`
## Docker

`$ docker run --cpus=3 -it busybox  sh `

containerID: `9621fa5988bdd7ba5128f9530a618aa270f9425137e6ffb4d207e5329be9b3f4`

```
eernst@eernstworkstation:/sys/fs/cgroup/cpu,cpuacct/docker$ tree
.
├── 9621fa5988bdd7ba5128f9530a618aa270f9425137e6ffb4d207e5329be9b3f4
│   ├── cgroup.clone_children
│   ├── cgroup.procs
│   ├── cpuacct.stat
│   ├── cpuacct.usage
│   ├── cpuacct.usage_all
│   ├── cpuacct.usage_percpu
│   ├── cpuacct.usage_percpu_sys
│   ├── cpuacct.usage_percpu_user
│   ├── cpuacct.usage_sys
│   ├── cpuacct.usage_user
│   ├── cpu.cfs_period_us
│   ├── cpu.cfs_quota_us
│   ├── cpu.shares
│   ├── cpu.stat
│   ├── notify_on_release
│   └── tasks
├── cgroup.clone_children
├── cgroup.procs
├── cpuacct.stat
├── cpuacct.usage
├── cpuacct.usage_all
├── cpuacct.usage_percpu
├── cpuacct.usage_percpu_sys
├── cpuacct.usage_percpu_user
├── cpuacct.usage_sys
├── cpuacct.usage_user
├── cpu.cfs_period_us
├── cpu.cfs_quota_us
├── cpu.shares
├── cpu.stat
├── notify_on_release
└── tasks

```



## Kata implementation
1. Move qemu and proxy from conmon cgroup to a new cgroup (kata-sandbox)
	1.1 - The kata-sandbox cgroup must be a child of the parent cgroup, that is specified in the OCI spec
	1.2 - The constraint for the kata-sandbox cgroup should be equal to -1 (no constraints), that way it inherits the constraint of its parent
2. Only next cgroups will be honored:
	* cpu
	* cpuset: cpuset  initially  is a join of all cpusets?  
3. Qemu vcpu threads shouldn't be moved into the sandbox cgroup, since the whole Qemu pid is moved into the kata-sandbox.
4. Kata-shim ???

## Appendix

### Guaranteed YAML:
```
apiVersion: v1
kind: Pod
metadata:
  name: guar-runc
spec:
  containers:
  - name: cont-2cpu-400m
    image: busybox
    resources:
      limits:
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

```
```

## Desired End State


```
s# for i in `ls pod*/**/tasks`; do echo $i && for j in `cat $i`; do pstree -pt $j;done; done;
podf277e232-5ca6-11e9-b514-000d3a6d0876/kata-sandbox
{qemu-system-x86}(24011)
{qemu-system-x86}(24093)
{qemu-system-x86}(24097)
{qemu-system-x86}(24156)
{qemu-system-x86}(24157)
{qemu-system-x86}(24158)
```

```
root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct/kubepods/podf277e232-5ca6-11e9-b514-000d3a6d0876# cat */cpu.shares
3072
2048
3072
```

```
# cat */cpu.cfs_quota_us
300000
200000
500000

```


## RESULTS

### containerd+kata

**Where is the shimv2 process. It is somewhere else?**

It is under system.slice/containerd.service/


**Where is the QEMU parent process and the iothreads?**

Again under system.slice/containerd.service/

This is not desired behavior as the system.slice is bounded to be 1024 CPU shares typically which is typically set to match kube+system reserved. So we will get bad CPU performance for iothreads under load.

The memory for this slice is unbounded. Which makes it worst of both worlds. So unbounded memory and highly bounded CPU.

**Why is this bad?**

This means we are using up kube and system reserved resources for kubepods.

Hence we have to stay within kubepods.

** What should we do?**

It is ok for the shim which is launched directly by containerd to stay where it is.

1. The pause cgroup cpu shares should be setup correctly. This needs fixing.

- Julio can you fix this.

2. Change the naming so that cri-o stats work at k8s level

- https://github.com/kata-containers/runtime/pull/1518


3. *We should move the qemu threads into the sandbox.*

- https://github.com/kata-containers/runtime/pull/1431

Why?
a. Node stability
b. The io's should be charged to the pod performing the io, specially in multi tenant enviornments.
c. OOM is for memory. Memory limits are enforced not requests. So users need to have higher limits till POD overhead support is added.
   In the case of CRIO the memory is being charged to conmon which is bounded by pod limits. So can still OOMed. But is still ok from a node stability POV. But not ok from POD stability POV. So resonably correct from a memory point of view. And bad pod performance from a CPU point of view if the workload is iobound.
   In case of containerd the memory is being charged to the caller (containerd) which is basically unbounded. So bad for node stability, also the POD is essentially unbounded. Also CPU here is bounded to ~kube+system (~1024), which is also bad.

4. *We should not create any containers cgroups on the host. We should create a pod sandbox cgroup that is entirely managed by us*

- The pod sandbox cgroup should always be the summation of all container group resources.

- In the case of cri-o where it creates other cgroups for conmon, they will be siblings.

- The conmon cgroups today are rather large, so if conmon goes wild there is a possibility we will get fewer resources. But it will not introduce any other side effects.


```
s# for i in `ls pod*/**/tasks`; do echo $i && for j in `cat $i`; do pstree -pt $j;done; done;
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

```
root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct/kubepods/podf277e232-5ca6-11e9-b514-000d3a6d0876# cat */cpu.shares
3072
2048
3072
```

```
# cat */cpu.cfs_quota_us
300000
200000
500000

```
#### shim
```
root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct# ps -aef | grep containerd-shim-kata | grep -v grep

root     23992     1  0 22:13 ?        00:00:00 /opt/kata/bin/containerd-shim-kata-v2 -namespace k8s.io -address /run/containerd/containerd.sock -publish-binary /usr/local/bin/containerd -id 9d17d1c5d075ca42dfefb58ef7b82c8b1b234cc128ebf9332b902b866c0ebed7 -debug
root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct# grep -ir 23992
system.slice/containerd.service/cgroup.procs:23992
system.slice/containerd.service/tasks:23992
```

#### qemu:
```
root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct# ps -ae | grep qemu
24007 ?        00:37:26 qemu-system-x86

root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct# grep -ir 24007
kubepods/podf277e232-5ca6-11e9-b514-000d3a6d0876/9d17d1c5d075ca42dfefb58ef7b82c8b1b234cc128ebf9332b902b866c0ebed7/cgroup.procs:24007
system.slice/containerd.service/cgroup.procs:24007
system.slice/containerd.service/tasks:24007
```

#### Vhost:

```
root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct# ps -aef | grep vhost | grep -v qemu | grep -v grep
root     24010     2  0 22:13 ?        00:00:00 [vhost-24007]

root@kata-k8s-containerd:/sys/fs/cgroup/cpu,cpuacct# grep -ir 24010
system.slice/containerd.service/cgroup.procs:24010
system.slice/containerd.service/tasks:24010
```

### cri-o


```
kata-runtime  : 1.6.1
   commit   : 8efc5718813224722f87ad119edcf9753fd6147d
   OCI specs: 1.0.1-dev
```


#### cgroup hierarchy
```
root@kata /sys/fs/cgroup/cpu/kubepods # tree /sys/fs/cgroup/cpu/kubepods/po*
/sys/fs/cgroup/cpu/kubepods/pod1cc61d33-5ca1-11e9-90bc-525400cfa589
├── crio-1b05886a39901ef3a7555796d38dcfaafd8fda929aef223ea576324a4949f9ef
│   ├── cpu.shares
├── crio-a8f744f610e72a4a20a3eb75582a6355ebae812e9d0ddcb3eb4d3c4ceb214485
│   ├── cpu.shares
├── crio-conmon-1b05886a39901ef3a7555796d38dcfaafd8fda929aef223ea576324a4949f9ef
│   ├── cpu.shares
├── crio-conmon-a8f744f610e72a4a20a3eb75582a6355ebae812e9d0ddcb3eb4d3c4ceb214485
│   ├── cpu.shares
├── crio-conmon-d3b54328d888c763c20a36958f3a25cece4f71f8a2c5900f0e54a80aef68fb1b
│   ├── cpu.shares
├── crio-d3b54328d888c763c20a36958f3a25cece4f71f8a2c5900f0e54a80aef68fb1b
│   ├── cpu.shares
└── tasks

```

##### cpusets

The first container we see is the pause container

```
root@kata /sys/fs/cgroup/cpuset/kubepods # for i in `ls pod*/**/cpuset.cpus`; do echo $i && cat $i;done;
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-1b05886a39901ef3a7555796d38dcfaafd8fda929aef223ea576324a4949f9ef/cpuset.cpus
1-5
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-a8f744f610e72a4a20a3eb75582a6355ebae812e9d0ddcb3eb4d3c4ceb214485/cpuset.cpus
1-2
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-conmon-1b05886a39901ef3a7555796d38dcfaafd8fda929aef223ea576324a4949f9ef/cpuset.cpus
0-7
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-conmon-a8f744f610e72a4a20a3eb75582a6355ebae812e9d0ddcb3eb4d3c4ceb214485/cpuset.cpus
0-7
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-conmon-d3b54328d888c763c20a36958f3a25cece4f71f8a2c5900f0e54a80aef68fb1b/cpuset.cpus
0-7
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-d3b54328d888c763c20a36958f3a25cece4f71f8a2c5900f0e54a80aef68fb1b/cpuset.cpus
3-5
```

##### shares

```
root@kata /sys/fs/cgroup/cpu/kubepods # for i in `ls pod*/**/cpu.shares`; do echo $i && cat $i;done;
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-1b05886a39901ef3a7555796d38dcfaafd8fda929aef223ea576324a4949f9ef/cpu.shares
3072
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-a8f744f610e72a4a20a3eb75582a6355ebae812e9d0ddcb3eb4d3c4ceb214485/cpu.shares
2048
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-conmon-1b05886a39901ef3a7555796d38dcfaafd8fda929aef223ea576324a4949f9ef/cpu.shares
1024
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-conmon-a8f744f610e72a4a20a3eb75582a6355ebae812e9d0ddcb3eb4d3c4ceb214485/cpu.shares
1024
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-conmon-d3b54328d888c763c20a36958f3a25cece4f71f8a2c5900f0e54a80aef68fb1b/cpu.shares
1024
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-d3b54328d888c763c20a36958f3a25cece4f71f8a2c5900f0e54a80aef68fb1b/cpu.shares
3072
```

##### tasks (runc)

```
root@runc /sys/fs/cgroup/cpu/kubepods # for i in `ls pod*/**/tasks`; do echo $i && for j in `cat $i`; do pstree -pt $j;done; done;
pod53ff10c2-5ca0-11e9-8a48-525400eac274/crio-340f2c953412c3c0c5d5a7ee68a850563a93f2ec3e4b292776e5ecee0279506d/tasks
md5sum(19646)
pod53ff10c2-5ca0-11e9-8a48-525400eac274/crio-96f7e692f20ae48664bef2a6cd5bee7782f4945593c6dda884c5a1454ea9121b/tasks
pause(19312)
pod53ff10c2-5ca0-11e9-8a48-525400eac274/crio-conmon-340f2c953412c3c0c5d5a7ee68a850563a93f2ec3e4b292776e5ecee0279506d/tasks
conmon(19634)─┬─md5sum(19646)
              └─{gmain}(19636)
{gmain}(19636)
pod53ff10c2-5ca0-11e9-8a48-525400eac274/crio-conmon-96f7e692f20ae48664bef2a6cd5bee7782f4945593c6dda884c5a1454ea9121b/tasks
conmon(19300)─┬─pause(19312)
              └─{gmain}(19302)
{gmain}(19302)
pod53ff10c2-5ca0-11e9-8a48-525400eac274/crio-conmon-fb40fb917f431ee82ffb73d8794c1634968557a6989d7481057973bbdfaa8fab/tasks
conmon(19422)─┬─md5sum(19434)
              └─{gmain}(19424)
{gmain}(19424)
pod53ff10c2-5ca0-11e9-8a48-525400eac274/crio-fb40fb917f431ee82ffb73d8794c1634968557a6989d7481057973bbdfaa8fab/tasks
md5sum(19434)
```
##### tasks (kata)

There are a whole bunch of threads under the pause cgroup. 

What are they?

Are they the vCPU threads or iothreads?

**They must be vCPU threads as we have 5 vCPUs and we see 5 qemu threads.**

The QEMU and vhost threads are under the conmon cgroup.

pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-conmon-1b05886a39901ef3a7555796d38dcfaafd8fda929aef223ea576324a4949f9ef/tasks

This incorrect?

So where should they go? 

**Ideally they should also go into the pause cgroup as there is no other cgroup that we can sit under.**




```
root@kata /sys/fs/cgroup/cpu/kubepods # for i in `ls pod*/**/tasks`; do echo $i && for j in `cat $i`; do pstree -pt $j;done; done;
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
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-a8f744f610e72a4a20a3eb75582a6355ebae812e9d0ddcb3eb4d3c4ceb214485/tasks
kata-shim(18288)─┬─{kata-shim}(18289)
                 ├─{kata-shim}(18290)
                 ├─{kata-shim}(18291)
                 ├─{kata-shim}(18292)
                 ├─{kata-shim}(18293)
                 ├─{kata-shim}(18296)
                 ├─{kata-shim}(18297)
                 ├─{kata-shim}(18298)
                 └─{kata-shim}(18299)
{kata-shim}(18289)
{kata-shim}(18290)
{kata-shim}(18291)
{kata-shim}(18292)
{kata-shim}(18293)
{kata-shim}(18296)
{kata-shim}(18297)
{kata-shim}(18298)
{kata-shim}(18299)
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
kata-proxy(18063)─┬─{kata-proxy}(18070)
                  ├─{kata-proxy}(18071)
                  ├─{kata-proxy}(18072)
                  ├─{kata-proxy}(18073)
                  ├─{kata-proxy}(18074)
                  ├─{kata-proxy}(18075)
                  ├─{kata-proxy}(18076)
                  ├─{kata-proxy}(18077)
                  ├─{kata-proxy}(18286)
                  ├─{kata-proxy}(20199)
                  ├─{kata-proxy}(20200)
                  ├─{kata-proxy}(20201)
                  ├─{kata-proxy}(20202)
                  ├─{kata-proxy}(20203)
                  └─{kata-proxy}(28490)
{kata-proxy}(18070)
{kata-proxy}(18071)
{kata-proxy}(18072)
{kata-proxy}(18073)
{kata-proxy}(18074)
{kata-proxy}(18075)
{kata-proxy}(18076)
{kata-proxy}(18077)
{kata-proxy}(18286)
{kata-proxy}(20199)
{kata-proxy}(20200)
{kata-proxy}(20201)
{kata-proxy}(20202)
{kata-proxy}(20203)
{kata-proxy}(28490)
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-conmon-a8f744f610e72a4a20a3eb75582a6355ebae812e9d0ddcb3eb4d3c4ceb214485/tasks
conmon(18265)─┬─kata-shim(18288)─┬─{kata-shim}(18289)
              │                  ├─{kata-shim}(18290)
              │                  ├─{kata-shim}(18291)
              │                  ├─{kata-shim}(18292)
              │                  ├─{kata-shim}(18293)
              │                  ├─{kata-shim}(18296)
              │                  ├─{kata-shim}(18297)
              │                  ├─{kata-shim}(18298)
              │                  └─{kata-shim}(18299)
              └─{gmain}(18267)
{gmain}(18267)
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-conmon-d3b54328d888c763c20a36958f3a25cece4f71f8a2c5900f0e54a80aef68fb1b/tasks
conmon(18353)─┬─kata-shim(18377)─┬─{kata-shim}(18378)
              │                  ├─{kata-shim}(18379)
              │                  ├─{kata-shim}(18380)
              │                  ├─{kata-shim}(18381)
              │                  ├─{kata-shim}(18382)
              │                  ├─{kata-shim}(18383)
              │                  ├─{kata-shim}(18384)
              │                  ├─{kata-shim}(18385)
              │                  ├─{kata-shim}(18386)
              │                  └─{kata-shim}(18387)
              └─{gmain}(18355)
{gmain}(18355)
pod1cc61d33-5ca1-11e9-90bc-525400cfa589/crio-d3b54328d888c763c20a36958f3a25cece4f71f8a2c5900f0e54a80aef68fb1b/tasks
kata-shim(18377)─┬─{kata-shim}(18378)
                 ├─{kata-shim}(18379)
                 ├─{kata-shim}(18380)
                 ├─{kata-shim}(18381)
                 ├─{kata-shim}(18382)
                 ├─{kata-shim}(18383)
                 ├─{kata-shim}(18384)
                 ├─{kata-shim}(18385)
                 ├─{kata-shim}(18386)
                 └─{kata-shim}(18387)
{kata-shim}(18378)
{kata-shim}(18379)
{kata-shim}(18380)
{kata-shim}(18381)
{kata-shim}(18382)
{kata-shim}(18383)
{kata-shim}(18384)
{kata-shim}(18385)
{kata-shim}(18386)
{kata-shim}(18387)
```


### Kata in Docker:

```
$ for j in `cat docker/2026c3747499019a8b33589e6fdc89194117879c0d47b4796c32c587b47bdf92/tasks`; do pstree -pt $j; done
{qemu-system-x86}(9690)
{qemu-system-x86}(9744)
{qemu-system-x86}(9745)
{qemu-system-x86}(9746)
kata-shim(9750)─┬─{kata-shim}(9751)
                ├─{kata-shim}(9752)
                ├─{kata-shim}(9754)
                ├─{kata-shim}(9755)
                ├─{kata-shim}(9762)
                ├─{kata-shim}(9763)
                ├─{kata-shim}(9764)
                └─{kata-shim}(9800)
{kata-shim}(9751)
{kata-shim}(9752)
{kata-shim}(9754)
{kata-shim}(9755)
{kata-shim}(9762)
{kata-shim}(9763)
{kata-shim}(9764)
{kata-shim}(9800)

```
