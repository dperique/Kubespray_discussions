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

Upgrading docker involves some drama because pods need to be restarted.  If you have
a service with certain availability characteristics, be mindful of this when upgrading
docker.  For example, ensure you have enough pod replicas in place so that worker
nodes getting a docker upgrade do not degrade your service in a noticable way.

In our case, we do this to upgrade docker:

* Modify the your kubespray related yamls and add `docker_version: '18.06'` for the
  case of docker 18.06.
* Possibly modify the ansible to point to a different docker repo if that version
  of docker is not available in the repo specified in kubespray.
* Create a version of cluster.yml that runs one node at a time or N nodes at a time
  so that your service has enough pod replicas to maintain service through a docker
  upgrade
* Run your version of cluster.yml that uses the upgrade docker version.

## Looking at kube apiserver load balancing on k8s

When you create a highly available k8s cluster using kubespray, you will need more
than one master.  But since the workers still need to talk to a master, you have to
put something in between the worker and the masters so that worker nodes can talk to
the masters in a load balanced way.  Kubespray does this via "localhost loadbalancing".
See the kubespray/docs/ha_mode.md document for more information.

Under the hood, we see that kubespray creates one "static pod" per worker k8s node
called `nginx-proxy-(nodeName)` in the kube-system namespace.  For example, if one of your
worker nodes is called kube-test-node-4, the pod will be named nginx-proxy-kube-test-node-4.
That nginx static pod uses a file called `/etc/nginx/nginx.conf` on the host; this is
a file mounted on the static pod using the hostPath method.  The static pod is created
as part of kubespray and the config file is create at
`kubespray/roles/kubernetes/node/tasks/nginx-proxy.yml`

The worker nodes talk to the apiserver via 127.0.0.1:6443.  The nginx pod forwards
the traffic to one of the masters in a load balanced way.

If you tweak that file, kill the right docker container to get the pod to restart and
re-read the file.
You can kill/start that pod by renaming the pod manifest in `/etc/kubernetes/manifests` to
something that begins with a dot, waiting for the static pod to terminate, and then
renaming it back.  This is the standard "static pod" behavior as described in the
Kubernetes docs.  You can also kill the docker container for that pod residing on the
Kubernetes host where that pod is running.  Do something like `docker ps |grep nginx-proxy`
to find it and then do a `docker rm ...` on the id of the one that doesn't have the
string "POD" in it.

The `/etc/nginx/nginx.conf` looks something like this (see my comments after the `<--`):

```
root@kube-test-k8s-node-4:/etc/nginx# cat nginx.conf
error_log stderr notice;

worker_processes auto;
events {
  multi_accept on;
  use epoll;
  worker_connections 1024;
}

stream {
        upstream kube_apiserver {
            least_conn;
            server 192.168.236.159:6443; <-- master1
            server 192.168.236.197:6443; <-- master2
            server 192.168.237.45:6443;  <-- master3
                    }

        server {
            listen        127.0.0.1:6443; <-- worker node sends apiserver traffic here
            proxy_pass    kube_apiserver;
            proxy_timeout 10m;
            proxy_connect_timeout 1s;

        }
}
```
