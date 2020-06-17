# How to monitor Kata Containers in Kubernetes clusters

This document describes how to run `kata-magent` in a Kubernetes cluster using Prometheus's service discovery to scrape metrics from `kata-agent`.

- [Introduction](#introduction)
- [Pre-requisites](#pre-requisites)
- [Configure Prometheus](#configure-prometheus)
- [Configure `kata-magent`](#configure-kata-magent)

> **Warning**: This how-to is only for evaluation purpose, you **SHOULD NOT** running it in production using this configurations.

## Introduction

If you are running Kata containers in a Kubernetes cluster, the best way to run `kata-magent` is using Kubernetes native `DaemonSet`, `kata-magent` will run on desired Kubernetes nodes without other operations when new nodes joined the cluster.

Prometheus also support a Kubernetes service discovery that can find scrape targets dynamically without explicitly setting `kata-magent`'s metric endpoints.

## Pre-requisites

You must have a running Kubernetes cluster first. If not, [install a Kubernetes cluster](https://kubernetes.io/docs/setup/) first.

Also you should ensure that `kubectl` working correctly.

> **Note**: More information about Kubernetes integrations:
>   - [Run Kata Containers with Kubernetes](run-kata-with-k8s.md)
>   - [How to use Kata Containers and Containerd](containerd-kata.md)
>   - [How to use Kata Containers and CRI (containerd plugin) with Kubernetes](how-to-use-k8s-with-cri-containerd-and-kata.md)

## Configure Prometheus

Start Prometheus by utilizing our sample manifest:

```
$ kubectl apply -f https://raw.githubusercontent.com/kata-containers/documentation/master/how-to/data/prometheus.yml
```

This will create a new namespace, `prometheus`, and create the following resources:

* `ClusterRole`, `ServiceAccount`, `ClusterRoleBinding` to let Prometheus to access Kubernetes API server.
* `ConfigMap` that contains minimum configurations to let Prometheus run Kubernetes service discovery.
* `Deployment` that run Prometheus in `Pod`.
* `Service` with `type` of `NodePort`(`30909` in this how to), that we can access Prometheus through `<hostIP>:30909`. In production environment, this `type` may be `LoadBalancer` or `Ingress` resource.

After the Prometheus server is running, run `curl -s http://hostIP:NodePort:30909/metrics`, if Prometheus is working correctly, you will get response like these:

```
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 3.9403e-05
go_gc_duration_seconds{quantile="0.25"} 0.000169907
go_gc_duration_seconds{quantile="0.5"} 0.000207421
go_gc_duration_seconds{quantile="0.75"} 0.000229911
```

## Configure `kata-magent`

`kata-magent` can be started on the cluster as follows:

```
$ kubectl apply -f https://raw.githubusercontent.com/kata-containers/documentation/master/how-to/data/kata-magent-daemontset.yml
```

This will create a new namespace `kata-system` and a `daemonset` in it.

Once the `daemonset` is running, Prometheus should discover `kata-magent` as a target. You can open `http://<hostIP>:30909/service-discovery` and find `kubernetes-pods` under the `Service Discovery` list

## References

- [Prometheus `kubernetes_sd_config`](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)

