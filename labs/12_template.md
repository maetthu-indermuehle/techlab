# Lab 12: Application templates

In this lab, we show how templates describe entire infrastructures and can be instantiated accordingly with one command.


## Templates

As you have seen in the previous labs, applications, databases, services and their configuration can be created and deployed simply by entering different commands.

This is error-prone and poorly suited for automation.

OpenShift offers the concept of templates, in which you can describe a list of resources that can be parameterized.
They are therefore a recipe for an entire infrastructure (e.g. 3 application containers, a database with persistent storage).

__Note__:
The cluster administrator can create global templates that are available to all users.

Display all existing templates:

```bash
oc get template -n openshift
```

This can be done via the Web Console using the "Developer Catalog". To do this, make sure you are in the Developer view, click "\+Add" and filter by Type Template.

These templates can be in JSON or YAML format, stored in the Git repository next to your source code, accessed via a URL or even stored locally in the file system.


## Task 1: Template instanzieren

The individual steps that we performed manually in the previous labs can now be performed in one "go" using the template.

Create a new project with name`[USERNAME]-template`.

<details><summary><b>Hint</b></summary>oc new-project [USERNAME]-template</details><br/>

Create template:

```bash
oc create -f https://raw.githubusercontent.com/appuio/example-spring-boot-helloworld/master/example-spring-boot-template.json
```

Instantiate template:

```bash
oc new-app example-spring-boot

--> Deploying template example-spring-boot for "example-spring-boot"
     With parameters:
      APPLICATION_DOMAIN=
      MYSQL_DATABASE_NAME=appuio
      MYSQL_USER=appuio
      MYSQL_PASSWORD=appuio
      MYSQL_DATASOURCE=jdbc:mysql://mysql/appuio?autoReconnect=true
      MYSQL_DRIVER=com.mysql.jdbc.Driver
--> Creating resources ...
    imagestream "example-spring-boot" created
    deploymentconfig "example-spring-boot" created
    deploymentconfig "mysql" created
    route "example-spring-boot" created
    service "example-spring-boot" created
    service "mysql" created
--> Success
    Run 'oc status' to view your app.

```

The following command imports the image and starts the deployment:

```bash
oc import-image example-spring-boot
```

This command is used to roll out the database:

```bash
oc rollout latest mysql
```

__Hint__:
You could also process templates directly by running a template with `oc new-app -f template.json -p param=value`.

To conclude this lab, you can take a closer look at the [template](https://github.com/appuio/example-spring-boot-helloworld/blob/master/example-spring-boot-template.json).

__Note__:
Existing resources can be exported as templates
Use the command `oc get -o json` or `oc get -o yaml` for this.

E.g.:

```bash
oc get is,bc,dc,route,service -o json > example-spring-boot-template.json
```

It is important that the ImageStreams are defined at the top of the template file.
Otherwise the first build will not work.

---

__End Lab 12__

<p width="100px" align="right"><a href="13_template_creation.md">Create your own templates →</a></p>

[← back to the overview](../README.md)
