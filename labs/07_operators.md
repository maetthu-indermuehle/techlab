# Lab 7: Operators

Operators are a way to package, deploy and manage Kubernetes-native applications. Kubernetes-native applications are applications that are deployed in Kubernetes/OpenShift and also managed via the Kubernetes/OpenShift API (kubectl/oc). Since OpenShift 4, OpenShift itself also uses a set of operators to manage the OpenShift cluster, i.e. itself.


## Introduction / Terms

To understand what an operator is and how it works, we first look at the so-called controller, since operators are based on its concept.


### Controller

A controller consists of a loop in which the desired state (_desired state_) and the current state (_actual state/obseved state_) of the cluster are read again and again. If the current state does not correspond to the desired state, the controller attempts to establish the desired state. The desired state is described with resources (deployments, ReplicaSets, pods, services, etc.).

The whole way OpenShift/Kubernetes works is based on this pattern. A large number of controllers run on the master (controller-manager), which establish the desired state based on resources (ReplicaSets, Pods, Services, etc.). For example, if a ReplicaSet is created, the ReplicaSet controller sees this and creates the corresponding number of pods as a result.

__Optional__: The article [The Mechanics of Kubernetes](https://medium.com/@dominik.tornow/the-mechanics-of-kubernetes-ac8112eaa302) provides a deep dive into how Kubernetes works. The graphic in the _Cascading Commands_ section nicely shows that four different controllers are involved from the creation of a deployment to the effective launching of the pods.


### Operator

An operator is a controller who is responsible for installing and managing an application. An operator therefore has application-specific knowledge. This is especially useful for more complex applications that consist of different components or require additional administration effort (e.g. newly started pods have to be added to an application cluster, etc.).

Also for the operator the desired state must be represented by a resource. For this purpose, there are so-called Custom Resource Definitions (CRD). CRDs can be used to define any new resources in OpenShift/Kubernetes. The operator then constantly (_watch_) checks whether custom resources for which the operator is responsible are changed and executes actions according to the target specification in the custom resource.

Operators thus make it easier to run more complex applications, since the management is taken over by the operator. Any complex configurations are abstracted by custom resources and operating tasks such as backups or rotating certificates etc. can also be executed by the operator.


## Installation of an operator

An operator runs like a normal application as a pod in the cluster. The installation of an operator usually includes the following resources:

* ***Custom Resource Definition***: In order to create the new custom resources that the operator handles, the corresponding CRDs must be installed.
* ***Service Account***: A service account with which the operator runs.
* ***Role und RoleBinding***: With a role you define all rights which the operator needs. This includes at least rights to the own custom resource. With a RoleBinding the new role is assigned to the service account of the operator.
* ***Deployment***: A deployment to run the actual operator. The operator usually runs only once (replicas set to 1), otherwise the different operator instances would get in each other's way.

OpenShift 4 has Operator Lifecycle Manager (OLM) installed by default. OLM simplifies the installation of operators. OLM allows us to _subscribe_ to an operator from a catalog, which is then automatically installed and, depending on the settings, automatically updated.

As an example we will install the ETCD operator in the next steps. Normally, setting up an ETCD cluster is a process with a few steps and you need to know many options to start each cluster member. The ETCD operator allows us to easily set up an ETCD cluster using the EtcdCluster custom resource. In doing so, we don't need detailed knowledge of ETCD, which would normally be required for setup, as this is all handled by the operator.
As for ETCD, there are also ready-made operators for many other applications, which massively simplify their operation.


### Task 1: Install ETCD operator

First, we create a new project:

```
oc new-project [USERNAME]-operator-test
```

The first thing we do is to see which operators are available. Among the available operators we find the operator `etcd` in the Community Operators catalog:

```
oc -n openshift-marketplace get packagemanifests.packages.operators.coreos.com | grep etcd
```

**Note**: As a cluster administrator, you can do this via the WebConsole (Operators -> OperatorHub).

We can now install the ETCD operator by creating a subscription. With existing `cat` binary this can be done with the following command, alternatively the content can be written to a file and created with `oc create -f <filename>`.

```
oc create -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: etcd
spec:
  channel: singlenamespace-alpha
  installPlanApproval: Automatic
  name: etcd
  source: community-operators
  sourceNamespace: openshift-marketplace
EOF
```

With the subscription we tell the operator lifecycle manager which operator (`name`) from which catalog (`source` and `sourceNamespace`) we would like to install. Most operators offer different update channels (`channel`), such as alpha, beta or stable. OLM will then install the latest version (ClusterServiceVersion) of the selected channel. With the option `installPlanApproval` you can also set OLM to automatically (`Automatic`) update the corresponding operator when a new version is available on the update channel.

In the rest of Task 1 we now want to examine whether the installation was successful and what OLM has created for us based on the subscription.

For the actual installation OLM searches for the newest ClusterServiceVersion of the `singlenamespace-alpha` channel and creates it:

```
$ oc get csv
NAME                  DISPLAY   VERSION   REPLACES              PHASE
etcdoperator.v0.9.4   etcd      0.9.4     etcdoperator.v0.9.2   Succeeded
```

The CSV triggers the actual installation of the operator and we should see the deployment of the operator in the project:

```
$ oc get deployment
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
etcd-operator   1/1     1            1           5m
```

Further we find a Service Account for the deployment and a Role incl. RoleBinding:

```
$ oc get serviceaccounts
NAME            SECRETS   AGE
...
etcd-operator   2         5m
```

```
$ oc get role
NAME                        AGE
etcdoperator.v0.9.4-gdmm2   5m
```

```
$ oc get rolebinding
NAME                                            AGE
etcdoperator.v0.9.4-gdmm2-etcd-operator-7lhcd   5m
...
```

In the background, the new `CustomResourceDefinition`s were also created:

* `etcdclusters.etcd.database.coreos.com` kind: `EtcdCluster`
* `etcdbackups.etcd.database.coreos.com` kind: `EtcdBackup`
* `etcdrestores.etcd.database.coreos.com` kind: `EtcdRestore`

These allow us in the next task to create the CustomResource `EtcdCluster`.

**Note**: To see CRDs, one must be a cluster administrator. Then you would find the new CRDs as follows: `oc get crd | grep etcd`.


### Task 2: Create ETCD cluster

We will now create an EtcdCluster resource to start an ETCD cluster:

```
oc create -f - <<EOF
apiVersion: etcd.database.coreos.com/v1beta2
kind: EtcdCluster
metadata:
  name: example
spec:
  size: 3
  version: 3.2.13
EOF
```

Now we can observe that three pods are/were created for the ETCD cluster:

```
$ oc get pod
NAME                             READY   STATUS    RESTARTS   AGE
etcd-operator-68c8484dc9-7sjfp   3/3     Running   0          27m
example-5fx5jxdh88               1/1     Running   1          7m22s
example-745pfjx2zt               1/1     Running   1          6m58s
example-g7856rl884               1/1     Running   1          8m2s
```

We can now easily modify the ETCD cluster via the EtcdCluster resource.
We will use `oc edit` to increase the cluser size (.spec.size) to 5, thus scaling up the ETCD cluster.

```
oc edit etcdcluster example
# update .spec.size to 5
```

We can now also observe again with `oc get pod` that the EtcdCluster is scaled up correctly.
Here, the ETCD operator not only starts the pods, but it also adds them to the ETCD cluster as new members, ensuring that the data is replicated to the new pods.
We can check this by listing the cluster members in one of the ETCD pods:

```
$ oc exec -it example-5fx5jxdh88 -- etcdctl member list
28af3778d7511ab6: name=example-99x775lzzt peerURLs=http://example-99x775lzzt.example.my-operator-test.svc:2380 clientURLs=http://example-99x775lzzt.example.my-operator-test.svc:2379 isLeader=false
5790fb58180b6680: name=example-g7856rl884 peerURLs=http://example-g7856rl884.example.my-operator-test.svc:2380 clientURLs=http://example-g7856rl884.example.my-operator-test.svc:2379 isLeader=false
92938f3e19f8df55: name=example-hxbcsxkjgw peerURLs=http://example-hxbcsxkjgw.example.my-operator-test.svc:2380 clientURLs=http://example-hxbcsxkjgw.example.my-operator-test.svc:2379 isLeader=false
9b4493c0eb24f65a: name=example-745pfjx2zt peerURLs=http://example-745pfjx2zt.example.my-operator-test.svc:2380 clientURLs=http://example-745pfjx2zt.example.my-operator-test.svc:2379 isLeader=false
e514d358ce1b7704: name=example-5fx5jxdh88 peerURLs=http://example-5fx5jxdh88.example.my-operator-test.svc:2380 clientURLs=http://example-5fx5jxdh88.example.my-operator-test.svc:2379 isLeader=true
```


### Task 3: Remove ETCD cluster

To remove the ETCD cluster, we just need to remove the EtcdCluster resource:

```
oc delete etcdcluster example
```


### Task 4: Uninstall operator

To uninstall an operator, the subscription on the one hand and the so-called ClusterServiceVersion of the operator on the other hand must be removed.

By deleting the subscription, we ensure that no new version is installed:

```
oc delete sub etcd
```

To remove the actually installed version, the corresponding ClusterServiceVersion must be uninstalled.
To do this, we first find the installed ClusterServiceVersion:

```
$ oc get csv
NAME                  DISPLAY   VERSION   REPLACES              PHASE
etcdoperator.v0.9.4   etcd      0.9.4     etcdoperator.v0.9.2   Succeeded
```

After that we remove the ClusterServiceVersion:

```
oc delete csv etcdoperator.v0.9.4
```

With `oc get pod` we can now verify that the operator pod has been removed.


## Further information

* [OpenShift Operators documentation](https://docs.openshift.com/container-platform/latest/operators/understanding/olm-what-operators-are.html)
* [Book from O'Reilly about Operators](https://www.redhat.com/cms/managed-files/cl-oreilly-kubernetes-operators-ebook-f21452-202001-en_2.pdf)

---

__Ende Lab 7__

<p width="100px" align="right"><a href="08_troubleshooting_ops.md">Troubleshooting →</a></p>

[← back to the overview](../README.md)
