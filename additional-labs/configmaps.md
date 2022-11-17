# ConfigMaps

ConfigMaps are used to separate the configuration for an application from the image and make it available to the pod at runtime, similar to setting environment variables.
This allows applications to be kept as portable as possible within containers.

In this lab, you will learn how to create ConfigMaps and use them accordingly.

## Create ConfigMap in OpenShift project

To create a ConfigMap in a project the following command can be used:

```bash
oc create configmap [Name der ConfigMap] [Options]
```

## Java Properties Files as a ConfigMap

A classic example for ConfigMaps are property files in Java applications, which primarily cannot be configured via environment variables.

For this example we use the Spring Boot example from [Lab 4](../labs/04_deploy_dockerimage.md), `[USERNAME]-dockerimage`.

<details><summary><b>Hint</b></summary>oc project [USERNAME]-dockerimage</details>

With the following command we now create the first ConfigMap based on a local file:

```bash
oc create configmap javaconfiguration --from-file=additional-labs/resources/properties.properties
```

The following command can now be used to verify that the ConfigMap was successfully created:

```bash
oc get configmaps
NAME                DATA   AGE
javaconfiguration   1      7s
```

We can also look at the contents of the ConfigMap:

```bash
oc get configmaps javaconfiguration -o json
```

## Provide ConfigMap in Pod

Next, we want to make the ConfigMap available in the pod.

Basically there are the following possibilities:

- ConfigMap Properties as environment variables in the deployment.
- Commandline Arguments via environment variables
- mounted as volumes in the container

In the example here, we want the ConfigMap to be a file on a volume.

For this we need to edit either the pod or in our case the DeploymentConfig with `oc edit dc example-spring-boot`.

The volumeMounts part (`spec.template.spec.containers.volumeMounts`: how to mount the volume into the container) and the volumes part (`spec.template.spec.volumes`: which volume in our case the ConfigMap will be mounted into the container) have to be considered.

```
apiVersion: v1
kind: DeploymentConfig
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftNewApp
  creationTimestamp: 2018-12-05T08:24:06Z
  generation: 2
  labels:
    app: example-spring-boot
  name: example-spring-boot
  namespace: [namespace]
  resourceVersion: "149323448"
  selfLink: /oapi/v1/namespaces/[namespace]/deploymentconfigs/example-spring-boot
  uid: 21f6578b-f867-11e8-b72f-001a4a026f33
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    app: example-spring-boot
    deploymentconfig: example-spring-boot
  strategy:
    activeDeadlineSeconds: 21600
    resources: {}
    rollingParams:
      intervalSeconds: 1
      maxSurge: 25%
      maxUnavailable: 25%
      timeoutSeconds: 600
      updatePeriodSeconds: 1
    type: Rolling
  template:
    metadata:
      annotations:
        openshift.io/generated-by: OpenShiftNewApp
      creationTimestamp: null
      labels:
        app: example-spring-boot
        deploymentconfig: example-spring-boot
    spec:
      containers:
      - env:
        - name: SPRING_DATASOURCE_USERNAME
          value: appuio
        - name: SPRING_DATASOURCE_PASSWORD
          value: appuio
        - name: SPRING_DATASOURCE_DRIVER_CLASS_NAME
          value: com.mysql.cj.jdbc.Driver
        - name: SPRING_DATASOURCE_URL
          value: jdbc:mysql://mysql/appuio?autoReconnect=true
        image: appuio/example-spring-boot
        imagePullPolicy: Always
        name: example-spring-boot
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/config
          name: config-volume
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 420
          name: javaconfiguration
        name: config-volume
```

Afterwards the values can be accessed in the container in the file `/etc/config/properties.properties`.

```bash
oc exec [POD] -- cat /etc/config/properties.properties
key=appuio
key2=openshift
```

These properties files can now be read and used by the Java application in the container.
The image remains environment-neutral.

## Task: LAB10.4.1 ConfigMap Data Sources

Create a ConfigMap for each and use the different types of [Data Sources](https://docs.openshift.com/container-platform/3.11/dev_guide/configmaps.html#consuming-configmap-in-pods).

Make the values within Pods available in the different ways.

---

__End Lab ConfigMaps__

[← back to the overview](../README.md)
