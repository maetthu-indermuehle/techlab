# Lab 4: Deploy a Container Image

In this lab we will deploy the first "pre-built" container image together and take a closer look at the OpenShift concepts Pod, Service, DeploymentConfig and ImageStream.

## Task 1: Deploy a Container Image 

Having used the source-to-image workflow to deploy an application to OpenShift in [Lab 3](03_first_steps.md), we now turn to deploying a pre-built container image from Docker Hub (or another image registry).

To do this, again create a new project named `[USERNAME]-dockerimage`.

<details><summary><b>Hint</b></summary>oc new-project [USERNAME]-dockerimage</details><br/>

`oc new-project` automatically switches to the newly created project. The `oc project` command can be used to change the project.

Use

```bash
oc get projects
```

to display all projects to which you are entitled.

Once the new project is created, we can deploy the container image in OpenShift using the following command:

```bash
oc new-app quay.io/appuio/example-spring-boot --as-deployment-config
```

For our lab, we use an APPUiO (Java Spring Boot application) example:

- GitHub (Source): <https://github.com/appuio/example-spring-boot-helloworld>
- Quay.io: <https://quay.io/appuio/example-spring-boot/>

When running the above command, OpenShift creates the necessary resources, downloads the container image (in this case from Docker Hub) and then deploys the Pod.

__Hint__: Use `oc status` to get an overview of the project.

Or use the `oc get` command with the `-w` parameter to continuously display changes to the pod type resources (cancel with ctrl+c):

```bash
oc get pods -w
```

Depending on your Internet connection or whether the image has already been downloaded on your OpenShift Node, this may take a while.
In the meantime, you can view the current status of the deployment in the Web Console:

1. Log in to the Web Console
1. Switch from the Administrator view to the Developer view (top left)
1. Select your project `[USERNAME]-dockerimage`.
1. Click on Topology

__Hint__:
To create your own container images for OpenShift, you should follow these best practices:
<https://docs.openshift.com/container-platform/latest/openshift_images/create-images.html>

## Viewing the created resources

When we ran `oc new-app quay.io/appuio/example-spring-boot` earlier, OpenShift created some resources for us in the background.
These are needed to deploy the container image:

- Service
- [ImageStream](https://docs.openshift.com/container-platform/latest/openshift_images/index.html#about-containers-images-and-image-streams)
- [DeploymentConfig](https://docs.openshift.com/container-platform/latest/applications/deployments/what-deployments-are.html)

### Service

Services serve as abstraction layer, entry point and proxy/load balancer to the underlying pods within OpenShift.
The service enables a group of pods of the same type to be found and addressed within OpenShift.

As an example: If an application instance in our example can no longer handle the load on its own, we can scale up the application to three pods, for example.
OpenShift automatically maps these to the service as endpoints.
Once the pods are ready, requests are automatically distributed to all three pods.

__Note__: The application is currently not accessible from the outside, the service is an OpenShift internal concept.
In the following lab we will make the application publicly available.

Now let's take a closer look at our service:

```bash
oc get services
NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
example-spring-boot   ClusterIP   172.30.141.7   <none>        8080/TCP,8778/TCP,9779/TCP   3m
```

As you can see from the output, our service `example-spring-boot` is accessible via an IP address and several ports (e.g. 172.30.141.7:8080).

__Note__:
Your IP address will most likely be different from the one shown here.
This IP address is also internal to OpenShift and cannot be reached from outside.

__Note__: Service IPs always remain the same during their lifetime.

You can use the following command to read out additional information about the service:

```bash
oc get service example-spring-boot -o json
{
    "apiVersion": "v1",
    "kind": "Service",
    "metadata": {
        "annotations": {
            "openshift.io/generated-by": "OpenShiftNewApp"
        },
        "creationTimestamp": "2020-01-09T08:21:13Z",
        "labels": {
            "app": "example-spring-boot"
        },
        "name": "example-spring-boot",
        "namespace": "techlab",
        "resourceVersion": "39162349",
        "selfLink": "/api/v1/namespaces/techlab/services/example-spring-boot",
        "uid": "ff9bc391-32b8-11ea-b825-fa163e286250"
    },
    "spec": {
        "clusterIP": "172.30.9.146",
        "ports": [
            {
                "name": "8080-tcp",
                "port": 8080,
                "protocol": "TCP",
                "targetPort": 8080
            },
            {
                "name": "8778-tcp",
                "port": 8778,
                "protocol": "TCP",
                "targetPort": 8778
            },
            {
                "name": "9000-tcp",
                "port": 9000,
                "protocol": "TCP",
                "targetPort": 9000
            },
            {
                "name": "9779-tcp",
                "port": 9779,
                "protocol": "TCP",
                "targetPort": 9779
            }
        ],
        "selector": {
            "app": "example-spring-boot",
            "deploymentconfig": "example-spring-boot"
        },
        "sessionAffinity": "None",
        "type": "ClusterIP"
    },
    "status": {
        "loadBalancer": {}
    }
}
```

__Hint__:
The `-o` parameter also accepts `yaml` as a specification.

You can also use the following command to view the details of a pod:

```bash
oc get pod example-spring-boot-3-nwzku -o json
```

__Note__:
First query the pod name from your project with `oc get pods` and then replace it in the above command.

The `selector` in the service is used to define which pods (`labels`) serve as endpoints.
For this purpose, the corresponding configurations of service and pod can be viewed together.

Service (`oc get service <Service Name>`):

```json
...
"selector": {
    "app": "example-spring-boot",
    "deploymentconfig": "example-spring-boot"
},

...
```

Pod (`oc get pod <Pod Name>`):

```json
...
"labels": {
    "app": "example-spring-boot",
    "deployment": "example-spring-boot-1",
    "deploymentconfig": "example-spring-boot"
},
...
```

This link is better seen using `oc describe` command:

```bash
$ oc describe service example-spring-boot
Name:              example-spring-boot
Namespace:         techlab
Labels:            app=example-spring-boot
Annotations:       openshift.io/generated-by=OpenShiftNewApp
Selector:          app=example-spring-boot,deploymentconfig=example-spring-boot
Type:              ClusterIP
IP:                172.30.9.146
Port:              8080-tcp  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.128.2.203:8080
Port:              8778-tcp  8778/TCP
TargetPort:        8778/TCP
Endpoints:         10.128.2.203:8778
Port:              9000-tcp  9000/TCP
TargetPort:        9000/TCP
Endpoints:         10.128.2.203:9000
Port:              9779-tcp  9779/TCP
TargetPort:        9779/TCP
Endpoints:         10.128.2.203:9779
Session Affinity:  None
Events:            <none>
```

Under Endpoints you will now find the currently running pod.


### ImageStream

[ImageStreams](https://docs.openshift.com/container-platform/latest/openshift_images/image-streams-manage.html) are used to perform automatic tasks such as updating a deployment when a new version of the image is available.

Builds and deployments can monitor ImageStreams and react to changes.
In our example, the image stream is used to trigger a deployment when something has changed in the image.

You can use the following command to read additional information about the ImageStream:

```bash
oc get imagestream example-spring-boot -o json
```


### DeploymentConfig

The following parameters are defined in the [DeploymentConfig](https://docs.openshift.com/container-platform/latest/applications/deployments/what-deployments-are.html):

- Update Strategy: How are application updates executed, how is the replacement of containers performed?
- Triggers: Which events lead to an automatic deployment?
- Container:
  - Which image should be used?
  - Environment Configuration
  - ImagePullPolicy
- Replicas: Number of pods to be deployed

The following command can be used to read out additional information about the DeploymentConfig:

```bash
oc get deploymentConfig example-spring-boot -o json
```

Unlike the DeploymentConfig, which tells OpenShift how to deploy an application, the ReplicationController defines how the application should look like during runtime (e.g. that 3 replicas should always be running).

__Hint__:
For many of the resource types there is also a short form. For example, instead of `oc get deploymentconfig` you can simply write `oc get dc`, or `svc` instead of `service`.

---

## Additional task for the fast ones ;-)

Look at the created resources with `oc get [ResourceType] [Name] -o json` and `oc describe [ResourceType] [Name]` from the first project `[USERNAME]-example1`. Alternatively, the `openshift-web-console` project can be used.

---

__Ende Lab 4__

<p width="100px" align="right"><a href="05_create_route.md">Create routes →</a></p>

[← back to the overview](../README.md)
