# Discussions about Kuberspray

I have been using [kubespray](https://github.com/kubernetes-incubator/kubespray) for
close to two years now and have created, upgraded, repaired, etc. many k8s clusters
with it.  We have many test/staging k8s clusters as well as about 20 production
k8s clusters.  This document talks about various things I've observed in hopes of
helping others understand kubespray from a user perspective including:

* understand what kubespray does
* experiences operating k8s clusters after they are created by kubespray

The first one is about analyzing pieces of what kubespray does and what
it created.  The point is if you know something about what it's doing, you will
have a better idea of how to debug kubespray and/or Kubernetes in the case something
goes wrong.

The second one is something I think is lacking.  A lot of people run kubespray
and create a Kubernetes cluster.  After that, they are done.  They load a few
Pods or Deployments, etc. and then destroy the k8s cluster.  They never really
get to operate k8s long term.  Because we have long lived staging and production
k8s clusters, these k8s cluster cannot just be destroyed or re-created.  They need
to be maintained like any other equipment including patching them for never-ending
security updates, monitored for resource usage, repaired in case of hardware failure,
etc.

In short, this document is about the "day 2" things or "day N" things.  I hope
people find it helpful and if things are wrong, could be better, or augmented, PRs
are always welcome.

## Our kubespray use-case

In my case, we build k8s clusters something like this:

* Get some BM machines, put Ubuntu 16.04 minimal on them, apply updates, etc.
* Network the machines so they are all on the same subnet
* Build an ansible inventory and encrypted ssh key and put it in a git repo
* We took kubespray v2.4.0 and made a local copy of it -- we froze kubespray to
  this version
  * We use Calico
  * Etcd is run as docker containers on the BM machines
* Run some home grown ansible that decrypts the ssh key and runs kubespray, extract
  the kubespray generated certificates for kubectl, place those in a git repo as
  encrypted.  Any kubespray settings are put into the git repo so that we can
  rebuild the k8s cluster using the same options.
* When we want kubectl, we run a script that decrypts the certificates and sets
  up the kubectl context.  Then we can run `kubectl ...`

We run kubespray periodically (once a day) on the working k8s cluster.  This is
idempotent for the most part.  But kubespray removes the Kubernetes apiserver,
controller, and scheduler pods and they come back on their own (this is expected).
Doing this is only disruptive to the control plane so we are ok with this.  Plus,
I like to know
that if any of these pods are removed, they can come back on their own -- I think
of this as a type health check to ensure the k8s cluster control plane remains
robust.  We monitor the kubespray run to ensure it succeeds without error.  If it
does not, we address it.

We turn off all automatic Ubuntu apt-get updates; this ensures that our software
is relatively static.  If we want to upgrade software (e.g., for security patches
or for more functionality, etc.), we will do those upgrades via some automation
that tracks the software versions.  The point here is nothing changes except our
software that runs on top of Kubernetes.

## Building your k8s cluster

For now, I'm going to just say that for the most part, Kubespray works well for
our use-case.  The biggest problem we may have on the initial installation is
going to be missed preparation steps (e.g., forgot to disable swap, forgot to
add correct ssh public keys, etc.).

We run the `cluster.yml` playbook.

One typical problem is something happens and the Kubernetes etcd did not come up.
We have instructions on how to fix that (for the most part, that's just an etcd
problem) and once fixed, Kubespray succeeds.  Another typical problem is that
Calico has a problem because it stores its data inside etcd -- again, fix etcd and
then this problem goes away as well.


## Upgrading Kubernetes

We upgrade Kubernetes by modifying the `kube_version` variable and possibly the
repo variables like this:

```
kube_version: v1.11.9
hyperkube_image_repo: "gcr.io/google-containers/hyperkube"
hyperkube_image_tag: "{{ kube_version }}"
```

There are two ways to do this:

* Rerun the `cluster.yml` playbook.  When you do this, kubespray upgrades all k8s
  nodes simultaneously.  We, of course, test this on an identical staging k8s
  cluster to ensure it's ok.
* Rerun a modified version of `cluster.yml` where you run with one k8s node at a
  time or use ansible-playbook with the `--limit` option.

## Upgrading Docker
