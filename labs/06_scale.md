# Lab 6: Pod Scaling, Readiness Probe 3nd and Self Healing

In this lab we will show how to scale applications in OpenShift.
We will also show how OpenShift ensures that the number of expected pods is started and how an application can report back to the platform that it is ready for requests.

## Task 1: Scale up the Example-Application 

For this we create a new project with the name `[USERNAME]-scale`.

<details><summary><b>Hint</b></summary>oc new-project [USERNAME]-scale</details><br/>

Add an application to the project:

```bash
oc new-app appuio/example-php-docker-helloworld --name=appuio-php-docker --as-deployment-config
```

And provide the service `appuio-php-docker` (expose).

<details><summary><b>Hint</b></summary>oc expose service appuio-php-docker</details><br/>

If we want to scale our example application, we need to tell our ReplicationController (rc) that we want to have, for example, 3 replicas of the image running at all times.

Let's take a closer look at the ReplicationController (rc):

```bash
oc get rc
NAME                  DESIRED   CURRENT   AGE
appuio-php-docker-1   1         1         33s
```

For more details output json or yaml output.

<details><summary><b>Hint</b></summary>oc get rc appuio-php-docker-1 -o json<br/>oc get rc appuio-php-docker-1 -o yaml</details><br/>

The rc tells us how many pods we expect (spec) and how many are currently deployed (status).


## Task 2: Scaling our sample application

Now we scale our example application to 3 replicas.
The ReplicationController we just looked at is controlled by the DeploymentConfig (dc), which is why we need to scale it so that the desired number of replicas is taken over by the rc:

```bash
oc scale --replicas=3 dc/appuio-php-docker
```

Let's check the number of replicas on the ReplicationController:

```bash
oc get rc
NAME                  DESIRED   CURRENT   AGE
appuio-php-docker-1   3         3         1m
```

and display the pods:

```bash
oc get pods
NAME                        READY     STATUS    RESTARTS   AGE
appuio-php-docker-1-2uc89   1/1       Running   0          21s
appuio-php-docker-1-evcre   1/1       Running   0          21s
appuio-php-docker-1-tolpx   1/1       Running   0          2m
```

Finally, let's look at the service. It should now reference all three endpoints:

```bash
$ oc describe svc appuio-php-docker
Name:              appuio-php-docker
Namespace:         techlab-scale
Labels:            app=appuio-php-docker
Annotations:       openshift.io/generated-by=OpenShiftNewApp
Selector:          app=appuio-php-docker,deploymentconfig=appuio-php-docker
Type:              ClusterIP
IP:                172.30.152.213
Port:              8080-tcp  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.128.2.204:8080,10.129.1.56:8080,10.131.0.141:8080
Port:              8443-tcp  8443/TCP
TargetPort:        8443/TCP
Endpoints:         10.128.2.204:8443,10.129.1.56:8443,10.131.0.141:8443
Session Affinity:  None
Events:            <none>
```

Scaling pods within a service is very fast because OpenShift simply launches a new instance of the container image as a container.

__Hint__:
OpenShift also supports[Autoscaling](https://docs.openshift.com/container-platform/latest/nodes/pods/nodes-pods-autoscaling.html).


## Task 3: Scaled app in the Web Console

Take a look at the scaled application in the Web Console as well.
How can you control the number of replicas via Web Console?


## Check interruption-free scaling

You can now use the following command to verify that your service is available while you scale up and down.
Run the following command in a terminal window and let it run while you scale up later.

Replace `[HOSTNAME]` with the hostname of your defined route:

__Hint__:
To display the corresponding hostname, the following command can be used:

`oc get route -o custom-columns=NAME:.metadata.name,HOSTNAME:.spec.host`

```bash
while true; do sleep 1; ( { curl -fs http://[HOSTNAME]/health/; date "+ TIME: %H:%M:%S,%3N" ;} & ) 2>/dev/null; done
```

or in PowerShell (__caution__: only from PowerShell version 3.0 upwards!):

```bash
while(1) {
        Start-Sleep -s 1
        Invoke-RestMethod http://[HOSTNAME]/pod/
        Get-Date -Uformat "+ TIME: %H:%M:%S,%3N"
}
```

The output shows the pod that responded to the request:

```bash
POD: appuio-php-docker-6-9w9t4 TIME: 16:40:04,991
POD: appuio-php-docker-6-9w9t4 TIME: 16:40:06,053
POD: appuio-php-docker-6-6xg2b TIME: 16:40:07,091
POD: appuio-php-docker-6-6xg2b TIME: 16:40:08,128
POD: appuio-php-docker-6-ctbrs TIME: 16:40:09,175
POD: appuio-php-docker-6-ctbrs TIME: 16:40:10,212
POD: appuio-php-docker-6-9w9t4 TIME: 16:40:11,279
POD: appuio-php-docker-6-9w9t4 TIME: 16:40:12,332
POD: appuio-php-docker-6-6xg2b TIME: 16:40:13,369
POD: appuio-php-docker-6-6xg2b TIME: 16:40:14,407
POD: appuio-php-docker-6-6xg2b TIME: 16:40:15,441
POD: appuio-php-docker-6-6xg2b TIME: 16:40:16,493
POD: appuio-php-docker-6-6xg2b TIME: 16:40:17,543
POD: appuio-php-docker-6-6xg2b TIME: 16:40:18,591
```

Now scale from __3__ replicas to __1__ while the While loop is running.

The requests are distributed to the different pods.
Once the pods have been scaled down, only one responds.

Now what happens if we start a new deployment while the While loop is still running?
Let's test it:

```bash
oc rollout latest appuio-php-docker
```

As the timestamp at the end of the output shows, during a short time the public route does not give an answer:

```bash
POD: appuio-php-docker-6-6xg2b TIME: 16:42:17,743
POD: appuio-php-docker-6-6xg2b TIME: 16:42:18,776
POD: appuio-php-docker-6-6xg2b TIME: 16:42:19,813
POD: appuio-php-docker-6-6xg2b TIME: 16:42:20,853
POD: appuio-php-docker-6-6xg2b TIME: 16:42:21,891
POD: appuio-php-docker-6-6xg2b TIME: 16:42:22,943
POD: appuio-php-docker-6-6xg2b TIME: 16:42:23,980
#
# no answer
#
POD: appuio-php-docker-7-pxnr3 TIME: 16:42:42,134
POD: appuio-php-docker-7-pxnr3 TIME: 16:42:43,181
POD: appuio-php-docker-7-pxnr3 TIME: 16:42:44,226
POD: appuio-php-docker-7-pxnr3 TIME: 16:42:45,259
POD: appuio-php-docker-7-pxnr3 TIME: 16:42:46,297
POD: appuio-php-docker-7-pxnr3 TIME: 16:42:47,571
POD: appuio-php-docker-7-pxnr3 TIME: 16:42:48,606
POD: appuio-php-docker-7-pxnr3 TIME: 16:42:49,645
POD: appuio-php-docker-7-pxnr3 TIME: 16:42:50,684
```

In our example, we use a very lightweight pod.
The failure would be more pronounced if the container took longer to process requests, e.g. the Java application of LAB 4 (__Startup: 30 seconds__).

It can even happen that the service is no longer online and the routing layer returns a __503 error__ as a response.

The following chapter describes how you can configure your services to enable interruption-free deployments.


## Uninterrupted deployment thanks to health checks and rolling update

The "[Rolling Strategy](https://docs.openshift.com/container-platform/latest/applications/deployments/deployment-strategies.html#deployments-rolling-strategy_deployment-strategies)" enables uninterrupted deployments.
With this, the new version of the application is started, and as soon as the application is ready, requests are directed to the new pod and the old version is removed.

In addition, [Container Health Checks](https://docs.openshift.com/container-platform/latest/applications/application-health.html#application-health-configuring_application-health) allows the deployed application to provide detailed feedback to the platform about its current health.

Basically, there are two types of health checks that can be implemented:

- Liveness Probe: Tells if a running container is still running cleanly
- Readiness Probe: Provides feedback on whether an application is ready to receive requests.

These two checks can be implemented as HTTP check, container execution check (command or e.g. shell script in container) or as TCP socket check.

In our example, the application should tell the platform if it is ready for requests.
For this we use the Readiness Probe. Our sample application returns a status code 200 under the path `/health` as soon as the application is ready.

```bash
http://[route]/health/
```


## Task 4: Readiness Probe

Add the Readiness Probe with the following command in the DeploymentConfig (dc):

```bash
oc set probe dc/appuio-php-docker --readiness --get-url=http://:8080/health --initial-delay-seconds=10
```

A look into the DeploymentConfig shows that the following entry has now been added under `.spec.template.spec.containers`:

```yaml
readinessProbe:
  failureThreshold: 3
  httpGet:
    path: /health
    port: 8080
    scheme: HTTP
  initialDelaySeconds: 10
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 1
```

During a deployment of the application, verify that an update of the application now also runs without interruption by observing the while loop already used during the following update command:

```bash
oc rollout latest appuio-php-docker
```

Now the answers should come without interruption from the new pod.


## Self Healing

Through the Replication Controller, we have now told the platform to run __n__ replicas at a time. What happens now when we delete a pod?

Using `oc get pods`, pick a pod in the "running" state that you can _kill_.

In a separate terminal, run the following command (displaying changes to pods)

```bash
oc get pods -w
```

In the other terminal, delete Pods with the following command:

```bash
oc delete pods -l deploymentconfig=appuio-php-docker
```

OpenShift makes sure that __n__ replicas of said pod are running again.

In the Web Console, it is easy to see how the pod is initially light blue until the Readiness Probe reports that the application is now ready.

---

__Ende Lab 6__

<p width="100px" align="right"><a href="07_operators.md">Operators →</a></p>

[← back to the overview](../README.md)
