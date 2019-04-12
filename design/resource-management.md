# cgroup updates in Kata

* [Background](#background)
* [Existing Behavior (Kata 1.6):](#existing-behavior--kata-16--)
  + [In Docker](#in-docker)
  + [In CRI-O](#in-cri-o)
  + [In Containerd](#in-containerd)
* [Proposed Changes](#proposed-changes)
  + [Alternatives Considered](#alternatives-considered)
* [Opens](#opens)


## Background

With 1.6 release of Kata Containers there are some issues with resource management resulting in inconsistent
behavior.  This document descibes the state of 1.6, and a suggested implementation for 1.7 version of Kata.

Before diving into the gaps and behavior exhibited in Kata Containes, it is important to have a thorough understanding
of how cgroups are leveraged by Kubernetes.  It is both straight forward and confusing.  An in-depth guide is available
for background [in mcastelino's gist](https://gist.github.com/mcastelino/b8ce9a70b00ee56036dadd70ded53e9f). This may be
good content to include in our repository eventually, or as part of the Kata blog series, but let's leave it out of the
scope of this initial documentation.

This document is part of the `cgroup-sprint` GitHub milestone, and can be observed in the [Milestone's GitHub project](https://github.com/orgs/kata-containers/projects/17).


## Existing Behavior (Kata 1.6):

### In Docker
### In CRI-O
### In Containerd


## Proposed Changes

### Alternatives Considered


## Opens
t
