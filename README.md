# kubeadm-dind-cluster [![Build Status](https://travis-ci.org/Mirantis/kubeadm-dind-cluster.svg?branch=master)](https://travis-ci.org/Mirantis/kubeadm-dind-cluster)
A Kubernetes multi-node cluster for developer _of_ Kubernetes and
projects that extend Kubernetes. Based on kubeadm and DIND (Docker in
Docker).

Supports both local workflows and workflows utilizing powerful remote
machines/cloud instances for building Kubernetes, starting test
clusters and running e2e tests.

If you're an application developer, you may be better off with
[Minikube](https://github.com/kubernetes/minikube) because it's more
mature and less dependent on the local environment, but if you're
feeling adventurous you may give `kubeadm-dind-cluster` a try, too. In
particular you can run `kubeadm-dind-cluster` in CI environment such
as Travis without having issues with nested virtualization.

## Requirements
Docker 1.12+ is recommended. If you're not using one of the
preconfigured scripts (see below) and not building from source, it's
better to have `kubectl` executable in your path matching the
version of k8s binaries you're using (i.e. for example better don't
use `kubectl` 1.7.x with `hyperkube` 1.6.x).

`kubeadm-dind-cluster` supports k8s versions 1.6.x (tested with 1.6.9),
1.7.x (tested with 1.7.8) and 1.8 (tested with 1.8.1).

**As of now, running `kubeadm-dind-cluster` on Docker with `btrfs`
storage driver is not supported.**

The problems include inability to properly clean up DIND volumes due
to a [docker bug](https://github.com/docker/docker/issues/9939) which
is not really fixed and, more importantly, a
[kubelet problem](https://github.com/kubernetes/kubernetes/issues/38337).
If you want to run `kubeadm-dind-cluster` on btrfs anyway, set
`RUN_ON_BTRFS_ANYWAY` environment variable to a non-empty value.

By default `kubeadm-dind-cluster` uses dockerized builds, so no Go
installation is necessary even if you're building Kubernetes from
source. If you want you can overridde this behavior by setting
`KUBEADM_DIND_LOCAL` to a non-empty value in [config.sh](config.sh).

### Mac OS X considerations

Ensure to have `md5sha1sum` installed. If not existing can be installed via `brew install md5sha1sum`.

When building Kubernetes from source on Mac OS X, it should be
possible to build `kubectl` locally, i.e. `make WHAT=cmd/kubectl` must
work.

## Using preconfigured scripts
`kubeadm-dind-cluster` currently provides preconfigured scripts for
Kubernetes 1.6, 1.7 and 1.8. This may be convenient for use with
projects that extend or use Kubernetes. For example, you can start
Kubernetes 1.8 like this:

```shell
$ wget https://cdn.rawgit.com/Mirantis/kubeadm-dind-cluster/master/fixed/dind-cluster-v1.8.sh
$ chmod +x dind-cluster-v1.8.sh

$ # start the cluster
$ ./dind-cluster-v1.8.sh up

$ # add kubectl directory to PATH
$ export PATH="$HOME/.kubeadm-dind-cluster:$PATH"

$ kubectl get nodes
NAME          STATUS    AGE       VERSION
kube-master   Ready     6m        v1.8.1
kube-node-1   Ready     5m        v1.8.1
kube-node-2   Ready     5m        v1.8.1

$ # k8s dashboard available at http://localhost:8080/ui

$ # restart the cluster, this should happen much quicker than initial startup
$ ./dind-cluster-v1.8.sh up

$ # stop the cluster
$ ./dind-cluster-v1.8.sh down

$ # remove DIND containers and volumes
$ ./dind-cluster-v1.8.sh clean
```

Replace 1.8 with 1.6 or 1.7 to use other Kubernetes versions.
**Important note:** you need to do `./dind-cluster....sh clean` when
you switch between Kubernetes versions (but no need to do this between
rebuilds if you use `BUILD_HYPERKUBE=y` like described below).

## Using with Kubernetes source
```shell
$ git clone git@github.com:Mirantis/kubeadm-dind-cluster.git ~/dind

$ cd ~/work/kubernetes/src/k8s.io/kubernetes

$ export BUILD_KUBEADM=y
$ export BUILD_HYPERKUBE=y

$ # build binaries+images and start the cluster
$ ~/dind/dind-cluster.sh up

$ kubectl get nodes
NAME          STATUS         AGE
kube-master   Ready,master   1m
kube-node-1   Ready          34s
kube-node-2   Ready          34s

$ # k8s dashboard available at http://localhost:8080/ui

$ # run conformance tests
$ ~/dind/dind-cluster.sh e2e

$ # restart the cluster rebuilding
$ ~/dind/dind-cluster.sh up

$ # run particular e2e test based on substring
$ ~/dind/dind-cluster.sh e2e "existing RC"

$ # shut down the cluster
$ ~/dind/dind-cluster.sh down
```

The first `dind/dind-cluster.sh up` invocation can be slow because it
needs to build the base image and Kubernetes binaries. Subsequent
invocations are much faster.

## IPv6 Mode (experimental)
To run Kubernetes in IPv6 only mode, the following steps can be performed
in a Kubernetes source setup. First, build a new kubeadm-dind-cluster
image, from the DinD area:

```shell
$ cd ~/dind

$ build/build-local.sh
$ export DIND_IMAGE=mirantis/kubeadm-dind-cluster:local
```

Next, clone a Kubernetes repo from master branch:

```shell
$ git clone https://github.com/kubernetes/kubernetes.git
$ cd kubernetes
```

As of Oct. 23rd, 2017, the following IPv6 PRs are in-flight, and should
be cherry picked and added to the repo:

```
PR #45551 "Adds Support for Node Resource IPv6 Addressing"
PR #53148 "ip6tables should be set in the noop plugin"
PR #52180 "Updating kubenet for CNI with IPv6"
PR #45792 "Updating NewCIDRSet return a value"
PR #50929 "Add kubeadm config for setting kube-proxy bind address"
PR #53555 "Add IPv6 and negative UT test cases for proxier's deleteEndpointConnections"
PR #52748 "Add brackets around IPv6 addrs in e2e test IP:port endpoints"
```

These can be obtained with doing cherry picks from the PR commits:
```
git fetch origin pull/45551/head:pr45551
git log --abbrev-commit pr45551 --oneline --abbrev-commit -n 1 | cut -f 1 -d" "

git fetch origin pull/53184/head:pr53184
git log --abbrev-commit pr53184 --oneline --abbrev-commit -n 1 | cut -f 1 -d" "

git fetch origin pull/52180/head:pr52180
git log --abbrev-commit pr52180 --oneline --abbrev-commit -n 1 | cut -f 1 -d" "

git fetch origin pull/45792/head:pr45792
git log --abbrev-commit pr45792 --oneline --abbrev-commit -n 1 | cut -f 1 -d" "
```

Had conflict on above and needed to resolve.

```
git fetch origin pull/50929/head:pr50929
git log --abbrev-commit pr50929 --oneline --abbrev-commit -n 1 | cut -f 1 -d" "

git fetch origin pull/53555/head:pr53555
git log --abbrev-commit pr53555 --oneline --abbrev-commit -n 1 | cut -f 1 -d" "

git fetch origin pull/52748/head:pr52748
git log --abbrev-commit pr52748 --oneline --abbrev-commit -n 1 | cut -f 1 -d" "
```

If you have a host that allows four times more conntrack connection (max) than
hashsize, You'll need this temporary patch (until issue 50787 is fixed and DinD is
modified to customize the kubelet config file):

https://github.com/pmichali/kubernetes/commit/d76164d919f292915ebaf69d8a3c357f40457bd7

Lastly, set up environment variables for building and IPv6 mode and
then bring up the cluster:

```shell
$ export BUILD_KUBEADM=y
$ export BUILD_HYPERKUBE=y
$ export IP_MODE=ipv6

$ ../dind-cluster.sh up
```

Note: there are additional customizations that you can make for IPv6,
to set the prefix used for DNS64, subnet prefix to use for DinD, and
the service subnet CIDR:

```shell
export DNS64_PREFIX=fd00:77:64:ff9b::
export DIND_SUBNET=fd00:77::
export SERVICE_CIDR=fd00:77:30::/110
```

## Configuration
You may edit `config.sh` to override default settings. See comments in
[the file](config.sh) for more info. In particular, you can specify
CNI plugin to use via `CNI_PLUGIN` variable (`bridge`, `flannel`,
`calico`, `weave`).

## Remote Docker / GCE
It's possible to build Kubernetes on a remote machine running Docker.
kubeadm-dind-cluster can consume binaries directly from the build
data container without copying them back to developer's machine.
An example utilizing GCE instance is provided in [gce-setup.sh](gce-setup.sh).
You may try running it using `source` (`.`) so that docker-machine
shell environment is preserved, e.g.
```shell
. gce-setup.sh
```
The example is based on sample commands from
[build/README.md](https://github.com/kubernetes/kubernetes/blob/master/build/README.md#really-remote-docker-engine)
in Kubernetes source.

When using a remote machine, you need to use ssh port forwarding
to forward `KUBE_RSYNC_PORT` and `APISERVER_PORT` you choose.

## Motivation
`hack/local-up-cluster.sh` is widely used for k8s development. It has
a couple of serious issues though. First of all, it only supports
single node clusters, which means that it's hard to use it to work on
e.g. scheduler-related issues and e2e tests that require several nodes
can't be run. Another problem is that it has little resemblance to
real clusters.

There's also k8s vagrant provider, but it's quite slow. Besides,
`cluster/` directory in k8s source is now considered deprecated.

Another widely suggested solution for development clusters is
[minikube](https://github.com/kubernetes/minikube), but currently it's
not very well suited for development of Kubernetes itself. Besides,
it's currently only supports single node, too, unless used with
additional DIND layer like [nkube](https://github.com/marun/nkube).

[kubernetes-dind-cluster](https://github.com/sttts/kubernetes-dind-cluster)
is very nice & useful but uses a custom method of cluster setup
(same as 2nd problem with local-up-cluster).

There's also sometimes a need to use a powerful remote machine or a
cloud instance to build and test Kubernetes. Having Docker as the only
requirement for such machine would be nice. Builds and unit tests are
already covered by
[jbeda's work](https://github.com/kubernetes/kubernetes/pull/30787) on
dockerized builds, but being able to quickly start remote test
clusters and run e2e tests is also important.

kubeadm-dind-cluster uses kubeadm to create a cluster consisting of
docker containers instead of VMs. That's somewhat of a compromise but
allows one to (re)start clusters quickly which is quite important when
making changes to k8s source.

Moreover, some projects that extend Kubernetes such as
[Virtlet](https://github.com/Mirantis/virtlet) need a way to start
kubernetes cluster quickly in CI environment without involving nested
virtulization. Current kubeadm-dind-cluster version provides means to
do this without the need to build Kubernetes locally.

## Additional notes
At the moment, all non-serial `[Conformance]` e2e tests pass for
clusters created by kubeadm-dind-cluster. `[Serial]...[Conformance]` tests
currently have some issues. You may still try running them though:
```
$ dind/dind-cluster.sh e2e-serial
```

## Related work

* kubeadm-dind-cluster was initially derived from
  [kubernetes-dind-cluster](https://github.com/sttts/kubernetes-dind-cluster),
  although as of now the code was completely rewritten.
  kubernetes-dind-cluster is somewhat faster but uses less standard
  way of k8s deployment. It also doesn't include support for consuming
  binaries from remote dockerized builds.
* [kubeadm-ci-dind](https://github.com/errordeveloper/kubeadm-ci-dind),
  [kubeadm-ci-packager](https://github.com/errordeveloper/kubeadm-ci-packager) and
  [kubeadm-ci-tester](https://github.com/errordeveloper/kubeadm-ci-tester).
  These projects are similar to kubeadm-dind-cluster but are intended primarily for CI.
  They include packaging step which is too slow for the purpose of having
  convenient k8s "playground". kubeadm-dind-cluster uses Docker images
  from `kubeadm-ci-dind`.
* [nkube](https://github.com/marun/nkube) starts
  Kubernetes-in-Kubernetes clusters.
