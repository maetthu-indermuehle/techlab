# Lab 10: Connect and use persistent storage for database

Per se, data in a pod is not persistent, which is also the case in our example.
If our MariaDB pod disappears, e.g. due to a change in the image, the data that existed before is no longer available in the new pod.
To prevent this, we now attach Persistent Storage to our MariaDB pod.

## Task 1: Persistent Storage anbinden

### Request Storage

Attaching persistent storage is actually done in two steps.
The first step involves creating a PersistentVolumeClaim for our project.
In the claim we define, among other things, its name and size, i.e. how much persistent storage we want to have in the first place.

The PersistentVolumeClaim represents the request, but not the resource itself.
It is therefore automatically associated by OpenShift with an available persistent volume, namely one with at least the requested size.
If only larger Persistent Volumes are available, one of these volumes is used and the size of the Claim is adjusted.
If only smaller persistent volumes are available, the claim cannot be fulfilled and remains open until a volume of the appropriate size (or even larger) appears.


### Include volume in Pod

The second step is to include the previously created PVC in the correct pod.
In [Lab 6](06_scale.md), we edited the DeploymentConfig to include the Readiness Probe.
We now do the same for the Persistent Volume.
However, unlike [Lab 6](06_scale.md), we can use `oc set volume` to automatically extend the DeploymentConfig.

We use the project from [Lab 9](09_database.md) [USERNAME]-dockerimage for this.

<details><summary><b>Hint</b></summary>oc project [USERNAME]-dockerimage</details><br/>

The following command performs both of the described steps at the same time, so it first creates the claim and then also includes it as a volume in the pod:

```bash
oc set volume dc/mariadb --add --name=mariadb-data --type pvc \
  --claim-name=mariadbpvc --claim-size=256Mi --overwrite
```

__Note__:
Due to the changed DeploymentConfig, OpenShift automatically deploys a new Pod.
Unfortunately, this also means that the previously created DB schema and already inserted data have been lost.

Our application creates the DB schema on its own at startup.

__Hint__:
Redeploy the application pod with:

```bash
oc rollout latest example-spring-boot
```

With the command `oc get persistentvolumeclaim`, or somewhat simpler `oc get pvc`, we can now display the PersistentVolumeClaim freshly created in the project:

```
oc get pvc
NAME         STATUS    VOLUME    CAPACITY   ACCESSMODES   AGE
mariadbpvc   Bound     pv34      1Gi        RWO,RWX       14s
```

The two attributes Status and Volume tell us that our claim has been associated with the Persistent Volume pv34.

With the following command we can also check whether the mounting of the volume in the DeploymentConfig worked:

```bash
oc set volume dc/mariadb --all
deploymentconfigs/mariadb
  pvc/mariadbpvc (allocated 1GiB) as mariadb-data
    mounted at /var/lib/mysql/data
```


## Task 2: Persistence test

### Restore data

Repeat[Lab-Task 9.4](09_database.md#l%C3%B6sung-lab84).


### Test

Now scale the MariaDB pod to 0 and then scale it back to 1. Observe that the new pod no longer loses the data.

---

__Ende Lab 10__

<p width="100px" align="right"><a href="11_dockerbuild_webhook.md">Integrate code changes directly via webhook →</a></p>

[← back to the overview](../README.md)

