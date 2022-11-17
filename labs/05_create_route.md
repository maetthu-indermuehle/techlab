# Lab 5: Create Route

In this lab, we will make the application from [Lab 4](04_deploy_dockerimage.md) accessible from the Internet using the HTTP protocol.


## Routes

The `oc new-app` command from the previous [Lab](04_deploy_dockerimage.md) does not create a route.
Thus, our service is not accessible from _outside_ at all.
If you want to make a service available, you have to create a route for it.
The OpenShift router recognizes which service to route a request to based on the host header.

Currently the following protocols are supported:

- HTTP
- HTTPS with [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication)
- TLS with [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication)
- WebSockets


## Task 1: Crate Route

Make sure that you are in the `[USERNAME]-dockerimage` project.

<details><summary><b>Hint</b></summary>oc project [USERNAME]-dockerimage</details><br/>

Create a route for the `example-spring-boot` service and make it publicly available through it.

__Hint__:
By means of `oc get routes` you can display the routes of a project.

```bash
oc get routes
No resources found.
```

Currently there is no route yet. Now we need the service name:

```bash
oc get services
NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                               AGE
example-spring-boot   ClusterIP   172.30.9.146   <none>        8080/TCP,8778/TCP,9000/TCP,9779/TCP   16m
```

And now we want to publish or expose this service:

```bash
oc expose service example-spring-boot
```

This command creates an unencrypted route, i.e. reachable via HTTP.
To create an encrypted route, we write the following:

```bash
oc create route edge example-spring-boot-secure --service=example-spring-boot
```

Using `oc get routes` we can check if the routes have been created.

```bash
oc get routes
NAME                         HOST/PORT                                         PATH      SERVICES              PORT       TERMINATION   WILDCARD
example-spring-boot          example-spring-boot-techlab.mycluster.com                   example-spring-boot   8080-tcp                 None
example-spring-boot-secure   example-spring-boot-secure-techlab.mycluster.com            example-spring-boot   8080-tcp   edge          None
```

The application is now accessible from the Internet via the specified URLs, so you can now access the application.

__Hint__:
If no explicit hostname is specified with `oc expose` or `oc create route`, _servicename-project.applicationdomain_ is used.

In the Web Console overview, this route with the hostname is now also visible (the icon at the top right of the blue ring).

Open the application in the browser and add a few "Say Hello" entries.

---

__Ende Lab 5__

<p width="100px" align="right"><a href="06_scale.md">Scaling →</a></p>

[← back to the overview](../README.md)
