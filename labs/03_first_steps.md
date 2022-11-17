# Lab 3: First Steps on the platform

In this lab, we will interact with the Lab platform together for the first time, both via the `oc` client and via the Web Console.


## Login

__Note__:
Make sure that you have successfully completed [Lab 2](02_cli.md), i.e. successfully logged into the Web Console as well as installed the `oc` client.

### Copy command via Web Console

The command for the login with `oc` can be fetched comfortably via Web Console.
To do this, click on your own username in the upper right corner and then on _Copy Login Command_:

![oc-login](../images/lab_03_login.png)

Now the command as shown under "Log in with this token" can be copied and pasted into a terminal window.

### Login direct with `oc`

As an alternative to copying the command, it is possible to log in directly with `oc`:

```bash
oc login FIXME: URL
```


## OpenShift Projects

A project in OpenShift is the top-level concept to organize your resources like deployments, builds, container images etc..
Users authorized for the project can manage these resources.
Within an OpenShift cluster, the name of a project must be unique.
See also [Lab 1](01_quicktour.md).


## Task 1: Create new project

Create a new project on the Lab platform named `[USERNAME]-example1`.

To find out how to create a new project with `oc`, you can use the following command:

```bash
oc help
```

<details><summary><b>Hint</b></summary>oc new-project [USERNAME]-example1</details><br/>


## Web Console

The OpenShift Web Console allows users to perform certain tasks directly via the browser.


## Task 2: Create application

1. Go to the overview of the project you just created. Currently the project is still empty.

1. Add your first application to your project. As an example project we use an APPUiO Example:

   1. first switch from the Administrator to the Developer view in the upper left corner.

   1. Make sure that your newly created project is selected under "Project".

   1. Click the "+Add" button on the top of the left menu stack.

   1. Now select "All services" under "Developer Catalog" in the bottom left-hand corner.

   1. limit the selection by clicking on _Languages_ and then _PHP_.

   1. now select the field "PHP" and click on "Create Application

   1. fill the field "Git Repo URL" with the following URL

   ```
   https://github.com/appuio/example-php-sti-helloworld.git
   ```

1. Leave the remaining fields blank or at their default value and click _Create_.

You just deployed your first application on OpenShift with __[Source to Image](https://docs.openshift.com/container-platform/latest/cicd/builds/build-strategies.html#builds-strategy-s2i-build_build-strategies)__ .

__Hint__:
The following commands can be used to create the above example on the command line:

```bash
oc new-app https://github.com/appuio/example-php-sti-helloworld.git
oc expose svc example-php-sti-helloworld
```

__Note__:
The `oc new-app` command requires `git`.
If `git` is not installed, esp. on Windows, the tool [downloaded here](https://git-scm.com/download/win) can be installed.

__Hint__:
An application created in this way, together with the additionally created resources, can be deleted in one fell swoop using labels, e.g. with the following command:

```bash
oc delete all --selector app=example-php-sti-helloworld-git
```

Um die Labels der verschiedenen Ressourcen anzuzeigen kann folgender Befehl verwendet werden:

```bash
oc get all --show-labels
```

---

__Ende Lab 3__

<p width="100px" align="right"><a href="04_deploy_dockerimage.md">Deploy a Container Image →</a></p>

[← back to the overview](../README.md)
