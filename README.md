
# OVN-Kubernetes Traffic Flow Test Scripts (ovn-kuber-traffic-flow-tests)

This repository contains the yaml files and test scripts to test all the traffic flows in an OVN-Kubernetes cluster.

## Table of Contents

- [Different Traffic Flows Tested](#different-traffic-flows-tested)
- [Cluster Deployment](#cluster-deployment)
	- [Upstream OVN-Kubernetes and KIND](#upstream-ovn-kubernetes-and-kind)
	- [OVN-Kubernetes Running on OCP](#ovn-kubernetes-running-on-ocp)
- [Test Pod Deployment](#test-pod-deployment)
  - [Launch Test Pods](#launch-test-pods)
  - [Cleanup Test Pods](#cleanup-test-pods)
  - [Deployment Customization](#deployment-customization)
- [Test Script Usage](#test-script-usage)
  - [curl](#curl)
  - [iperf3](#iperf3)
  - [ovnkube-trace](#ovnkube-trace)
- [Container Images](#container-images)


## Different Traffic Flows Tested

1. Pod to Pod traffic
   * Pod to Pod (Same Node)
   * Pod to Pod (Different Node)
1. Pod to Host traffic
   * Pod to Host (Same Node)
   * Pod to Host (Different Node)
1. Pod -> Cluster IP Service traffic (Pod Backend)
   * Pod to Cluster IP (Pod Backend - Same Node)
   * Pod to Cluster IP (Pod Backend - Different Node)
1. Pod -> Cluster IP Service traffic (Host Backend)
   * Pod to Cluster IP (Host Backend - Same Node)
   * Pod to Cluster IP (Host Backend - Different Node)
1. Pod -> NodePort Service traffic (Pod Backend)
   * Pod -> NodePort Service traffic (Pod Backend - Same Node)
   * Pod -> NodePort Service traffic (Pod Backend - Different Node)
1. Pod -> NodePort Service traffic (Host Backend)
   * Pod -> NodePort Service traffic (Host Backend - Same Node)
   * Pod -> NodePort Service traffic (Host Backend - Different Node)
1. Host to Pod traffic
   * Host to Pod (Same Node)
   * Host to Pod (Different Node)
1. Host to Host traffic
   * Host to Host (Same Node)
   * Host to Host (Different Node)
1. Host -> Cluster IP Service traffic (Pod Backend)
   * Host to Cluster IP (Pod Backend - Same Node)
   * Host to Cluster IP (Pod Backend - Different Node)
1. Host -> Cluster IP Service traffic (Host Backend)
   * Host to Cluster IP (Host Backend - Same Node)
   * Host to Cluster IP (Host Backend - Different Node)
1. Host -> NodePort Service traffic (Pod Backend)
   * Host -> NodePort Service traffic (Pod Backend - Same Node)
   * Host -> NodePort Service traffic (Pod Backend - Different Node)
1. Host -> NodePort Service traffic (Host Backend)
   * Host -> NodePort Service traffic (Host Backend - Same Node)
   * Host -> NodePort Service traffic (Host Backend - Different Node)
1. Cluster -> External Network
   * Pod -> External Network
   * Host -> External Network
1. External Network -> Cluster IP Service traffic
   * External Network -> Cluster IP Service traffic (Pod Backend)
   * External Network -> Cluster IP Service traffic (Host Backend)
   * **NOTE:** External doesn't now about Cluster IP, so these tests are a NOOP.
1. External Network -> NodePort Service traffic
   * External Network -> NodePort Service traffic (Pod Backend)
   * External Network -> NodePort Service traffic (Host Backend)
1. External Network -> Cluster (multiple external GW traffic)
   * **NOTE:** Special Use-Case for customer - Not Implemented


## Cluster Deployment

### Upstream OVN-Kubernetes and KIND

To test with upstream OVN-Kubernetes and KIND:
```
cd $GOPATH/src/github.com/ovn-org/ovn-kubernetes/contrib/
./kind.sh -ha -wk 4  -gm shared
```

With this KIND Cluster:
* Nodes `ovn-control-plane`, `ovn-worker` and `ovn-worker2` are master nodes.
* Nodes `ovn-worker3`, `ovn-worker4`, `ovn-worker5` and `ovn-worker6` are worker nodes.


### OVN-Kubernetes Running on OCP

Deploy OCP as normal.

In the SR-IOV Lab, the Nodes are as follows:
* Nodes `sriov-master-0`, `sriov-master-1` and `sriov-master-2` are master nodes.
* Nodes `sriov-worker-0` and `sriov-worker-1` are worker nodes.


## Test Pod Deployment

### Launch Test Pods

Test setup is as follows, create POD backed set of resources:
* Run pod-backed *'client'* (DaemonSet) on every node.
* Run one instance of a pod-backed *'http-server'*.
* Create a ClusterIP Service for the pod-backed *'http-server'* using NodePort 8080.
* Create a NodePort Service for the pod-backed *'http-server'* using NodePort 30080.
* Run one instance of a pod-backed *'iperf-server'*.
* Create a ClusterIP Service for the pod-backed *'iperf-server'* using NodePort 5201.
* Create a NodePort Service for the pod-backed *'iperf-server'* using NodePort 30201.

Create Host-POD backed set of resources:
* Run host-backed *'client'* (DaemonSet) on every node.
* Run one instance of a host-backed *'http-server'*.
* Create a ClusterIP Service for the host-backed *'http-server'* using NodePort 8081.
* Create a NodePort Service for the host-backed *'http-server'* using NodePort 30081.
* Run one instance of a host-backed *'iperf-server'*.
* Create a ClusterIP Service for the host-backed *'iperf-server'* using NodePort 5202.
* Create a NodePort Service for the host-backed *'iperf-server'* using NodePort 30202.

The script finds:
* *'client'* pod on the *'Same Node'* as the pod-backed *'server'*
* *'client'* pod on a *'Different Node'* from the pod-backed *'server'*
* *'client'* pod on the *'Same Node'* as the host-pod-backed *'server'*
* *'client'* pod on a *'Different Node'* from the host-pod-backed *'server'*

Once the *'client'* pods (LOCAL and REMOTE, POD and HOST) and IP addresses have been
collected, the script runs *'curl'* commands in different combinations to test each of
traffic flows.


To create all the pods and services (*'client'* DaemonSets, the different *'server'*
instances, and the ClusterIP and NodePort Services):

```
cd ~/src/ovn-kuber-traffic-flow-tests/

./launch.sh
```

Each *'server'* (pod backed and host-networked pod backed) needs to be on the same node.
So the setup scripts use labels to achieve this. The default is to schedule the servers
on the first worker node detected. If there is a particular node the *'server'* pods
should run on, for example on an OVS Hardware offloaded node, then use the following
environment variable to force each *'server'* pod on a desired node ('FT_' stands for
Flow Test).  *NOTE:* This needs to be set before the pods are launched.

```
FT_REQ_SERVER_NODE=ovn-worker4 \
./launch.sh

-- OR --

export FT_REQ_SERVER_NODE=ovn-worker4
./launch.sh
```

Along the same lines, the *'launch.sh'* script creates a *'client'* (pod backed and
host-networked pod backed) on each worker node. The *'test.sh'* script sends packets from
the node on the same node the *'server'* pods are running on (determined as described above)
and a remote node (node *'server'* pods are NOT running on). If there is a particular node
that should be marked as the *' remote client'* node, for example on an OVS Hardware
offloaded node, then use the following environment variable to force the *'test.sh'* script
to pick as the desired node.  *NOTE:* This needs to be set before the *'test.sh'* script is
run and can be changed between each test run.

```
FT_REQ_REMOTE_CLIENT_NODE=ovn-worker3 \
./test.sh

-- OR --

export FT_REQ_REMOTE_CLIENT_NODE=ovn-worker3
./test.sh
```

### Cleanup Test Pods

To teardown the test setup:

```
cd ~/src/ovn-kuber-traffic-flow-tests/

./cleanup.sh
```

Several files are generated during test runs. For example, `iper3` output files,
`ovn-trace` output files, and Pod and Service deployment YAML files (generated
using `j2`). All these are described more below. The files are not deleted by
default. To delete all the generated files, use `CLEAN_ALL`.
To teardown the test setup:

```
cd ~/src/ovn-kuber-traffic-flow-tests/

CLEAN_ALL=true ./cleanup.sh
```

**NOTE:** This is especially important between updates of the repository, because
this is still relatively new and there is still some churn on naming convention of
everything.

### Deployment Customization

This repository uses `j2` to customize the YAML files used to
deploy the Pods and Services. The following fields can be
overridden by setting these variables (with their default values):

```
  SRIOV_RESOURCE_NAME=openshift.io/mlnx_bf
  TEST_IMAGE=quay.io/billy99/ft-base-image:0.6

  HTTP_CLUSTERIP_POD_SVC_PORT=8080
  HTTP_CLUSTERIP_HOST_SVC_PORT=8081
  HTTP_NODEPORT_POD_SVC_PORT=30080
  HTTP_NODEPORT_HOST_SVC_PORT=30081

  IPERF_CLUSTERIP_POD_SVC_PORT=5201
  IPERF_CLUSTERIP_HOST_SVC_PORT=5202
  IPERF_NODEPORT_POD_SVC_PORT=30201
  IPERF_NODEPORT_HOST_SVC_PORT=30202
```

## Test Script Usage

To run all the tests, simply run the script.
* All the hard-coded values are printed to the screen (and can be overwritten). 
* Then all the queried values, like Pod Names and IP addresses are printed.
* Each test is run with actual command executed printed to the screen.
* <span style="color:green">**SUCCESS**</span> or <span style="color:red">**FAILED**</span> is then printed.

```
$ ./test.sh

Default/Override Values:
  Test Control:
    TEST_CASE (0 means all)            1
    VERBOSE                            false
    FT_NOTES                           true
    CURL                               true
    CURL_CMD                           curl -m 5
    IPERF                              true
    IPERF_CMD                          iperf3
    IPERF_TIME                         2
    OVN_TRACE                          false
    OVN_TRACE_CMD                      ./ovnkube-trace -loglevel=5 -tcp
    FT_REQ_REMOTE_CLIENT_NODE          all
  OVN Trace Control:
    OVN_K_NAMESPACE                    ovn-kubernetes
    SSL_ENABLE                         -noSSL
  From YAML Files:
    CLIENT_POD_NAME_PREFIX             ft-client-pod
    http Server:
      HTTP_SERVER_POD_NAME             ft-http-server-pod-v4
      HTTP_SERVER_HOST_POD_NAME        ft-http-server-host-v4
      HTTP_CLUSTERIP_POD_SVC_NAME      ft-http-service-clusterip-pod-v4
      HTTP_CLUSTERIP_HOST_SVC_NAME     ft-http-service-clusterip-host-v4
      HTTP_NODEPORT_SVC_NAME           ft-http-service-nodeport-pod-v4
      HTTP_NODEPORT_HOST_SVC_NAME      ft-http-service-nodeport-host-v4
    iperf Server:
      IPERF_SERVER_POD_NAME            ft-iperf-server-pod-v4
      IPERF_SERVER_HOST_POD_NAME       ft-iperf-server-host-v4
      IPERF_CLUSTERIP_POD_SVC_NAME     ft-iperf-service-clusterip-pod-v4
      IPERF_CLUSTERIP_HOST_SVC_NAME    ft-iperf-service-clusterip-host-v4
      IPERF_NODEPORT_POD_SVC_NAME      ft-iperf-service-nodeport-pod-v4
      IPERF_NODEPORT_HOST_SVC_NAME     ft-iperf-service-nodeport-host-v4
    POD_SERVER_STRING                  Server - Pod Backend Reached
    HOST_SERVER_STRING                 Server - Host Backend Reached
    EXTERNAL_SERVER_STRING             The document has moved
  External Access:
    EXTERNAL_IP                        8.8.8.8
    EXTERNAL_URL                       google.com
Queried Values:
  Pod Backed:
    HTTP_SERVER_POD_IP                 10.244.2.29
    IPERF_SERVER_POD_IP                10.244.2.30
    SERVER_POD_NODE                    ovn-worker3
    LOCAL_CLIENT_NODE                  ovn-worker3
    LOCAL_CLIENT_POD                   ft-client-pod-76qlj
    REMOTE_CLIENT_NODE                 ovn-worker4
    REMOTE_CLIENT_POD                  ft-client-pod-566xj
    HTTP_CLUSTERIP_POD_SVC_IPV4        10.96.39.137
    HTTP_CLUSTERIP_POD_SVC_PORT        8080
    HTTP_NODEPORT_POD_SVC_IPV4         10.96.28.182
    HTTP_NODEPORT_POD_SVC_PORT         30080
    IPERF_CLUSTERIP_POD_SVC_IPV4       10.96.153.56
    IPERF_CLUSTERIP_POD_SVC_PORT       5201
    IPERF_NODEPORT_POD_SVC_IPV4        10.96.37.54
    IPERF_NODEPORT_POD_SVC_PORT        30201
  Host backed:
    HTTP_SERVER_HOST_IP                172.18.0.5
    IPERF_SERVER_HOST_IP               172.18.0.5
    SERVER_HOST_NODE                   ovn-worker3
    LOCAL_CLIENT_HOST_NODE             ovn-worker3
    LOCAL_CLIENT_HOST_POD              ft-client-pod-host-kttz2
    REMOTE_CLIENT_HOST_NODE            ovn-worker4
    REMOTE_CLIENT_HOST_POD             ft-client-pod-host-hp5r2
    HTTP_CLUSTERIP_HOST_SVC_IPV4       10.96.21.56
    HTTP_CLUSTERIP_HOST_SVC_PORT       8081
    HTTP_NODEPORT_HOST_SVC_IPV4        10.96.252.33
    HTTP_NODEPORT_HOST_SVC_PORT        30081
    IPERF_CLUSTERIP_HOST_SVC_IPV4      10.96.11.170
    IPERF_CLUSTERIP_HOST_SVC_PORT      5202
    IPERF_NODEPORT_HOST_SVC_IPV4       10.96.154.57
    IPERF_NODEPORT_HOST_SVC_PORT       30202


FLOW 01: Typical Pod to Pod traffic (using cluster subnet)
----------------------------------------------------------

*** 1-a: Pod to Pod (Same Node) ***

kubectl exec -it ft-client-pod-76qlj -- curl -m 5 "http://10.244.2.29:8080/"
SUCCESS


*** 1-b: Pod to Pod (Different Node) ***
kubectl exec -it ft-client-pod-566xj -- curl -m 5 "http://10.244.2.29:8080/"
SUCCESS


FLOW 02: Pod -> Cluster IP Service traffic
------------------------------------------

*** 2-a: Pod -> Cluster IP Service traffic (Same Node) ***

kubectl exec -it ft-client-pod-2twqq -- curl -m 5 "http://10.96.145.26:8080/"
SUCCESS

:
```

Below are some commonly used overrides:

* If a single test needs to be run (this is at the FLOW level):
```
TEST_CASE=3 ./test.sh
```

* For readability, the output of the `curl` is masked. This can be unmasked for debugging:
```
TEST_CASE=3 VERBOSE=true ./test.sh
```

* `iperf3` is disabled by default. To enable and change the timeout (in seconds
and default is 10 seconds):
```
TEST_CASE=3 IPERF=true IPERF_TIME=2 ./test.sh
```

* `ovnkube-trace` is disabled by default. To enable:
```
TEST_CASE=3 OVN_TRACE=true ./test.sh
```

* To run on OCP:
```
SSL_ENABLE=" " OVN_K_NAMESPACE=openshift-ovn-kubernetes ./test.sh
```
<br>


*NOTE:* There are a couple of sub-FLOWs that are failing and not sure if they
are suppose to work or not, so there are some test-case notes (in blue font)
for those, for example:
> curl: (6) Could not resolve host: http-service-node-v4; Unknown error<br>
> Should this work?

### curl

`curl` is used to test connectivity between pods and ensure a given flow
is working. `curl` is enabled by default, but can be disabled using
`CURL=false`.

```
$ TEST_CASE=1 ./test.sh

:

FLOW 01: Typical Pod to Pod traffic (using cluster subnet)
----------------------------------------------------------

*** 1-a: Pod to Pod (Same Node) ***

kubectl exec -it ft-client-pod-2twqq -- curl -m 5 "http://10.244.2.26:8080/"
SUCCESS


*** 1-b: Pod to Pod (Different Node) ***

kubectl exec -it ft-client-pod-gc6dw -- curl -m 5 "http://10.244.2.26:8080/"
SUCCESS
```

### iperf3

`iperf3` is used to test packet throughput. It can be used to determine
the rough throughput of each flow. When enabled, `iperf3` is run and a
summary of the results is printed.

```
$ TEST_CASE=1 IPERF=true IPERF_TIME=2 ./test.sh

:

FLOW 01: Typical Pod to Pod traffic (using cluster subnet)
----------------------------------------------------------

*** 1-a: Pod to Pod (Same Node) ***

kubectl exec -it ft-client-pod-2twqq -- curl -m 5 "http://10.244.2.26:8080/"
SUCCESS

kubectl exec -it ft-client-pod-2twqq -- iperf3 -c 10.244.2.27 -p 5201 -t 2
Summary (see iperf-logs/1a-pod2pod-same-node.txt for full detail):
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-2.00   sec  3.43 GBytes  14.7 Gbits/sec  334             sender
[  5]   0.00-2.04   sec  3.43 GBytes  14.4 Gbits/sec                  receiver
SUCCESS


*** 1-b: Pod to Pod (Different Node) ***

kubectl exec -it ft-client-pod-gc6dw -- curl -m 5 "http://10.244.2.26:8080/"
SUCCESS

kubectl exec -it ft-client-pod-gc6dw -- iperf3 -c 10.244.2.27 -p 5201 -t 2
Summary (see iperf-logs/1b-pod2pod-diff-node.txt for full detail):
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-2.00   sec   633 MBytes  2.65 Gbits/sec    0             sender
[  5]   0.00-2.04   sec   632 MBytes  2.60 Gbits/sec                  receiver
SUCCESS
```

When `iperf3` is run on each sub-flow, the full output of the command is piped to
files in the `iperf-logs/` directory. Use 'VERBOSE=true' to when command is executed
to see full output command is run. Below is a list of sample output files:

```
$ ls -al iperf-logs/
total 1072
drwxrwxr-x. 2 user user  4096 Apr 16 11:58 .
drwxrwxr-x. 5 user user   223 Apr 16 10:09 ..
-rw-rw-r--. 1 user user 84398 Apr 16 11:57 1a-pod2pod-same-node.txt
-rw-rw-r--. 1 user user 78030 Apr 16 11:57 1b-pod2pod-diff-node.txt
-rw-rw-r--. 1 user user 94706 Apr 16 11:57 2a-pod2clusterIPsvc-same-node.txt
-rw-rw-r--. 1 user user 88338 Apr 16 11:57 2b-pod2clusterIPsvc-diff-node.txt
-rw-rw-r--. 1 user user 94673 Apr 16 11:57 3a-pod2nodePortsvc-pod-backend-same-node.txt
-rw-rw-r--. 1 user user 88305 Apr 16 11:57 3b-pod2nodePortsvc-pod-backend-diff-node.txt
-rw-rw-r--. 1 user user 76891 Apr 16 11:57 3c-pod2nodePortsvc-host-backend-same-node.txt
-rw-rw-r--. 1 user user 73304 Apr 16 11:58 3d-pod2nodePortsvc-host-backend-diff-node.txt
-rw-rw-r--. 1 user user 77620 Apr 16 11:58 5a-hostpod2clusterIPsvc-pod-backend-same-node.txt
-rw-rw-r--. 1 user user 78151 Apr 16 11:58 5b-hostpod2clusterIPsvc-pod-backend-diff-node.txt
-rw-rw-r--. 1 user user 77587 Apr 16 11:58 6a-hostpod2nodePortsvc-pod-backend-same-node.txt
-rw-rw-r--. 1 user user 78118 Apr 16 11:58 6b-hostpod2nodePortsvc-pod-backend-diff-node.txt
-rw-rw-r--. 1 user user 10841 Apr 16 11:58 7a-hostpod2clusterIPsvc-host-backend-same-node.txt
-rw-rw-r--. 1 user user  9903 Apr 16 11:58 7b-hostpod2clusterIPsvc-host-backend-diff-node.txt
-rw-rw-r--. 1 user user 10833 Apr 16 11:58 8a-hostpod2nodePortsvc-host-backend-same-node.txt
-rw-rw-r--. 1 user user  9896 Apr 16 11:58 8b-hostpod2nodePortsvc-host-backend-diff-node.txt
-rw-rw-r--. 1 user user    70 Apr 16 10:09 .gitignore
```

*NOTE:* The `cleanup.sh` script does not remove these files and each subsequent run of
`test.sh` overwrites the previous test run.

### ovnkube-trace

`ovnkube-trace` is a tool in upstream OVN-Kubernetes to trace packet simulations
between points in ovn-kubernetes. When enabled, `ovnkube-trace` is run on each sub-flow
and the output is piped to files in the `ovn-traces/` directory. Below is a list of
sample output files:

```
$ ls -al ovn-traces/
total 1072
drwxrwxr-x. 2 user user  4096 Apr 16 11:58 .
drwxrwxr-x. 5 user user   223 Apr 16 10:09 ..
-rw-rw-r--. 1 user user 84398 Apr 16 11:57 1a-pod2pod-same-node.txt
-rw-rw-r--. 1 user user 78030 Apr 16 11:57 1b-pod2pod-diff-node.txt
-rw-rw-r--. 1 user user 94706 Apr 16 11:57 2a-pod2clusterIPsvc-same-node.txt
-rw-rw-r--. 1 user user 88338 Apr 16 11:57 2b-pod2clusterIPsvc-diff-node.txt
-rw-rw-r--. 1 user user 94673 Apr 16 11:57 3a-pod2nodePortsvc-pod-backend-same-node.txt
-rw-rw-r--. 1 user user 88305 Apr 16 11:57 3b-pod2nodePortsvc-pod-backend-diff-node.txt
-rw-rw-r--. 1 user user 76891 Apr 16 11:57 3c-pod2nodePortsvc-host-backend-same-node.txt
-rw-rw-r--. 1 user user 73304 Apr 16 11:58 3d-pod2nodePortsvc-host-backend-diff-node.txt
-rw-rw-r--. 1 user user 23623 Apr 16 11:58 4a-pod2externalHost.txt
-rw-rw-r--. 1 user user 77620 Apr 16 11:58 5a-hostpod2clusterIPsvc-pod-backend-same-node.txt
-rw-rw-r--. 1 user user 78151 Apr 16 11:58 5b-hostpod2clusterIPsvc-pod-backend-diff-node.txt
-rw-rw-r--. 1 user user 77587 Apr 16 11:58 6a-hostpod2nodePortsvc-pod-backend-same-node.txt
-rw-rw-r--. 1 user user 78118 Apr 16 11:58 6b-hostpod2nodePortsvc-pod-backend-diff-node.txt
-rw-rw-r--. 1 user user 10841 Apr 16 11:58 7a-hostpod2clusterIPsvc-host-backend-same-node.txt
-rw-rw-r--. 1 user user  9903 Apr 16 11:58 7b-hostpod2clusterIPsvc-host-backend-diff-node.txt
-rw-rw-r--. 1 user user 10833 Apr 16 11:58 8a-hostpod2nodePortsvc-host-backend-same-node.txt
-rw-rw-r--. 1 user user  9896 Apr 16 11:58 8b-hostpod2nodePortsvc-host-backend-diff-node.txt
-rw-rw-r--. 1 user user    70 Apr 16 10:09 .gitignore
```

Examine these files to debug why a particular flow isn't working or to better understand
how a packet flows through OVN-Kubernetes for a particular flow.

*NOTE:* The `cleanup.sh` script does not remove these files and each subsequent run of
`test.sh` overwrites the previous test run.

## Container Images

The `http-server` pods currently use `registry.access.redhat.com/ubi8/python-38` image
to implement the http server.

The `client` pods and the `iperf-server` pods are using the same image, which uses
`docker.io/centos:8` as the base with `curl` and `iperf3` packages pulled in. The
image has been built and pushed to `quay.io` for use by this repo.

```
quay.io/billy99/ft-base-image:0.6
```

To build the image:

```
cd ~/src/ovn-kuber-traffic-flow-tests/images/base-image/

sudo podman build -t quay.io/<USER>/ft-base-image:<TAG> -f ./Dockerfile .
```
