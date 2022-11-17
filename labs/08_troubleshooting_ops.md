# Lab 8: Troubleshooting

This lab shows how to proceed in case of an error and which tools are available.

## Log into container

For this we use the project from[Lab 4](04_deploy_dockerimage.md) `[USERNAME]-dockerimage`.

<details><summary><b>Hint</b></summary>oc project [USERNAME]-dockerimage</details><br/>

Running containers are treated as immutable infrastructure and should generally not be modified.
Nevertheless, there are use cases where you need to log into the containers.
For example, for debugging and analyses.

## Task 1: Remote Shells

Mit OpenShift können Remote Shells in die Pods geöffnet werden, ohne dass man darin vorgängig SSH installieren müsste.
Dafür steht einem der Befehl `oc rsh` zur Verfügung.

Wählen Sie einen Pod aus und öffnen Sie die Remote Shell.

<details><summary><b>Hint</b></summary>oc get pods<br/>oc rsh [POD]</details><br/>

__Hint__:
If `oc rsh` does not work, open a terminal in the Web Console (Applications -> Pods -> [pod name] -> Terminal).

You can now run analytics in the container using this shell:

```bash
bash-4.2$ ls -la
total 16
drwxr-xr-x. 7 default root   99 May 16 13:35 .
drwxr-xr-x. 4 default root   54 May 16 13:36 ..
drwxr-xr-x. 6 default root   57 May 16 13:35 .gradle
drwxr-xr-x. 3 default root   18 May 16 12:26 .pki
drwxr-xr-x. 9 default root 4096 May 16 13:35 build
-rw-r--r--. 1 root    root 1145 May 16 13:33 build.gradle
drwxr-xr-x. 3 root    root   20 May 16 13:34 gradle
-rwxr-xr-x. 1 root    root 4971 May 16 13:33 gradlew
drwxr-xr-x. 4 root    root   28 May 16 13:34 src
```

With `exit` or `ctrl`+`d` you can log out of the pod or shell again.


## Task 2: Execute commands in container

Individual commands within the container can be executed via `oc exec`:

```bash
oc exec [POD] -- env
```

For example:

```bash
$ oc exec example-spring-boot-4-8mbwe -- env
PATH=/opt/app-root/src/bin:/opt/app-root/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=example-spring-boot-4-8mbwe
KUBERNETES_SERVICE_PORT_DNS_TCP=53
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=172.30.0.1
KUBERNETES_PORT_53_UDP_PROTO=udp
KUBERNETES_PORT_53_TCP=tcp://172.30.0.1:53
...
```


## View log files

The log files for a pod can be viewed both in the Web Console and in the CLI:

```bash
oc logs [POD]
```

The parameter `-f` causes the same behavior as `tail -f`, i.e. "follow".

If a pod is in __CrashLoopBackOff__ status, it means that it could not be successfully started even after repeated restarts.
The log files can be viewed even if the pod is not running with the following command:

```bash
oc logs -p [POD]
```

The `-p` parameter stands for "previous", i.e. it refers to a pod of the same DeploymentConfig that was running before but is no longer running.
Accordingly, this command works only if there actually was a pod before.

With OpenShift, an EFK stack (Elasticsearch, Fluentd, Kibana) is provided that collects, rotates and aggregates all log files.
Kibana allows you to search, filter and graph logs.


## Metrics

The OpenShift Platform also provides a basic set of metrics that are integrated into the Web Console and can be used to scale pods automatically.

You can now influence the resource consumption of a pod by logging in directly to that pod and monitor the impact of this in the Web Console.


## Task 3: Port Forwarding

OpenShift allows arbitrary ports to be forwarded from the development workstation to a pod.
This is useful, for example, to access administration consoles, databases, etc. that are not exposed to the Internet and are not otherwise accessible.
Port forwarding is tunneled over the same HTTPS connection that the OpenShift client (oc) otherwise uses.
This allows connecting to pods even if there are restrictive firewalls and/or proxies between the workstation and OpenShift.

Lab: Access the Spring Boot Metrics from [Lab 4](04_deploy_dockerimage.md).

```bash
oc get pod --namespace="[USERNAME]-dockerimage"
oc port-forward [POD] 9000:9000 --namespace="[USERNAME]-dockerimage"
```

Do not forget to adapt the pod name to your own installation.
If installed, you can use Autocompletion for this.

The metrics can now be accessed at the following URL: [http://localhost:9000/metrics/](http://localhost:9000/metrics/).

The metrics are displayed as JSON.
Using the same concept, you can now connect to a database with your local SQL client, for example.

See the [Documentation](https://docs.openshift.com/container-platform/latest/nodes/containers/nodes-containers-port-forwarding.html) for more information on port forwarding.

__Note__:
The `oc port-forward` process will continue to run until it is aborted by the user.
So as soon as port-forwarding is no longer needed, it can be stopped with ctrl+c.

---

__Ende Lab 8__

<p width="100px" align="right"><a href="09_database.md">Datenbank deployen und anbinden →</a></p>

[← back to the overview](../README.md)
