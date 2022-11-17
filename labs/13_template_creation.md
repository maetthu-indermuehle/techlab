# Lab 13: Create your own templates

Unlike [Lab 12](12_template.md), here we write our own templates before creating applications with them.


## Helpful `oc` commands

List all commands:

```bash
oc help
```

Overview of (almost) all resources:

```bash
oc get all
```

Info about a resource:

```bash
oc get <RESOURCE_TYPE> <RESOURCE_NAME>
oc describe <RESOURCE_TYPE> <RESOURCE_NAME>
```


## Generation

Via `oc new-app` or "\+Add" in the Web Console the resources are created automatically.
The creation can be easily configured in the Web Console.

For productive use, this is usually not enough.
More control over the configuration is needed.
Custom templates are the solution for this.
However, they do not have to be written by hand but can be generated as a template.


### Generation before creation

With `oc new-app` OpenShift parses the given images, templates, source code repositories, etc. and creates the definition of the various resources.
With the `-o` option you get the definition without creating the resources.

This is how the definition of the hello-world image looks like:

```bash
oc new-app hello-world -o json
```

It is also exciting to observe what OpenShift makes of an own project.
For this, a Git repository or a local path of the computer can be specified.

Example command if one is in the root directory of the project:

```bash
oc new-app . -o json
```

If different ImageStreams are in question or none was found, it must be specified:

```bash
oc new-app . --image-stream=wildfly:latest -o json
```

`oc new-app` always creates a list of resources.
If needed, one can be converted to a template with [jq](https://stedolan.github.io/jq/):

```bash
oc new-app . --image-stream=wildfly:latest -o json | \
  jq `{ kind: "Template", apiVersion: .apiVersion, metadata: {name: "mytemplate" }, objects: .items }`
```


### Generation after creation

Existing resources are exported with `oc get -o json` or `oc get -o yaml`.

```bash
oc get route my-route -o json
```

Welche Ressourcen braucht es?

The following resources are necessary for a complete template:

- ImageStreams
- BuildConfigurations
- DeploymentConfigurations
- PersistentVolumeClaims
- Routes
- Services

Example command to generate an export of the most important resources as a template:

```bash
oc get is,bc,pvc,dc,route,service -o json > my-template.json
```

Attributes with value `null` as well as the annotation `openshift.io/generated-by` may be removed from the template.


### Export existing templates

It is also possible to pick up existing templates from the platform to create your own templates.

Available templates are stored in the project `openshift`.
These can be listed as follows:

```bash
oc get templates -n openshift
```

So we get a copy of the eap70-mysql-persistent-s2i template:

```bash
oc get template eap72-mysql-persistent-s2i -o json -n openshift > eap72-mysql-persistent-s2i.json
```


## Parameter

There are parameters so that the applications can be customized for one`s own needs.
Generated or exported templates should replace fixed values like hostnames or passwords with parameters.


### Display template parameters

With `oc process --parameters` the parameters of a template are displayed. Here we want to see which parameters are defined in the CakePHP MySQL template:

```bash
oc process --parameters cakephp-mysql-example -n openshift
NAME                           DESCRIPTION                                                                GENERATOR VALUE
NAME                           The name assigned to all of the frontend objects defined in this template.           cakephp-mysql-example
NAMESPACE                      The OpenShift Namespace where the ImageStream resides.                               openshift
MEMORY_LIMIT                   Maximum amount of memory the CakePHP container can use.                              512Mi
MEMORY_MYSQL_LIMIT             Maximum amount of memory the MySQL container can use.                                512Mi
...
```


### Replace parameters of templates with values

For the creation of the applications, desired parameters can be replaced with values.
For this purpose we use `oc process`:

```bash
oc process -f eap70-mysql-persistent-s2i.json \
  -v PARAM1=value1,PARAM2=value2 > processed-template.json
```

So parameters from the template will be replaced with the given values and written to a new file. This file will be a list of resources/items which can be created with `oc create`:

```bash
oc create -f processed-template.json
```

This can also be done in one step:

```bash
oc process -f eap72-mysql-persistent-s2i.json \
  -v PARAM1=value1,PARAM2=value2 \
  | oc create -f -
```


## Write templates

OpenShift Documentation: <https://docs.openshift.com/container-platform/latest/openshift_images/using-templates.html>

Applications should be built so that only a few configurations differ per environment.
These values are defined as parameters in the template.
Thus, the first step after generating a template definition is to define parameters.
The template is extended with variables, which are then replaced with the parameter values.
For example, the variable `${DB_PASSWORD}` is replaced with the parameter named `DB_PASSWORD`.


### Generierte Parameter

Often passwords are generated automatically because the value is only used in the OpenShift project.
This can be achieved with a "generate" definition.

```
parameters:
  - name: DB_PASSWORD
    description: "DB connection password"
    generate: expression
    from: "[a-zA-Z0-9]{13}"
```

This definition would generate a random, 13-character password with lowercase and uppercase letters and numbers.

Even if a parameter is configured with "generate" definition, it can be overwritten during generation.


### Template Merge

For example, if an app is used together with a database, the two templates can be merged.
It is important to consolidate the template parameters.
These are usually values for the connection of the database.
Simply use the same variable from the common parameter in both templates.


## Anwenden vom Templates

Templates can be instantiated with `oc new-app -f <FILE>|<URL> -p <PARAM1>=<VALUE1>,<PARAM2>=<VALUE2>...`.
If the parameters of the template have already been set with `oc process`, there is no need to specify the parameters.

---

__Ende Lab 13__

[← back to the overview](../README.md)
