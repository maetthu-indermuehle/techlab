# Lab 9: Connect database

Most applications are stateful in some way (they have their own state) and store data persistently.
This can be in a database or as files on a file system or object store.
In this lab we will create a MariaDB service in our project and connect it to our application so that multiple application pods can access the same database.

For this example we will use the Spring Boot example from [Lab 4](04_deploy_dockerimage.md), `[USERNAME]-dockerimage`.

<details><summary><b>Hint</b></summary>oc project [USERNAME]-dockerimage</details><br/>


## Task 1: Create MariaDB Service 

For our example in this lab, we use an OpenShift template that creates a MariaDB database with EmptyDir data storage.
This is only to be used for test environments, as all data will be lost when the MariaDB pod is restarted.
In a later lab we will show how to attach a Persistent Volume (mariadb-persistent) to the MariaDB database.
This will keep the data even during restarts, making the setup suitable for production use.

We can create the MariaDB service via the Web Console as well as via the CLI.

To get the same result, only database name, username, password and DatabaseServiceName have to be set the same, no matter which variant is used:

- `MYSQL_USER: appuio`
- `MYSQL_PASSWORD: appuio`
- `MYSQL_DATABASE: appuio`
- `DATABASE_SERVICE_NAME: mariadb`


### CLI

Using the CLI, the MariaDB service can be created as follows.

__Note__:
The backslashes (`\`) are used to map a long command more clearly on several lines.
__Auf Windows__ the so-called multiline delimiter is not the backslash, but in cmd the caret (`^`) and in PowerShell the backtick (`` ` ``).

```bash
oc new-app mariadb-ephemeral \
   -pMYSQL_USER=appuio \
   -pMYSQL_PASSWORD=appuio \
   -pMYSQL_DATABASE=appuio
```

These resources are created:

```bash
secret/mariadb created
service/mariadb created
deploymentconfig.apps.openshift.io/mariadb created
```


### Web Console

In the Web Console, the MariaDB (Ephemeral) service can be added to the project via Catalog in the Developer view:

- First make sure that you have switched from Administrator to Developer view in the upper left corner.
- Click on "\+Add" (top left)
- Select "From Catalog
- Narrow down the selection by clicking on "Databases", "MariaDB".
- Select the field "MariaDB (Ephemeral)" and then "Instantiate Template
- Fill in the form fields "MariaDB Connection Username", "MariaDB Connection Password" and "MariaDB Database Name".
- Leave the remaining form fields empty or at their default value and create them with "Create

![MariaDBService](../images/lab_08_mariadb.png)


### Password and username as plain text?

When deploying the database via CLI as well as via Web Console, we used parameters to specify values for user, password and database.
In this chapter, we will now take a look at where this sensitive data actually ended up.

First, let's take a look at the DeploymentConfig of the database:

```bash
oc get dc mariadb -o yaml
```

Specifically, it is about configuring the containers using environment variables (`MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`) in the DeploymentConfig under `spec.templates.spec.containers`:

```yaml
spec:
  containers:
    - env:
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              key: database-user
              name: mariadb
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              key: database-password
              name: mariadb
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              key: database-root-password
              name: mariadb
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              key: database-name
              name: mariadb
```

The values for the individual environment variables thus come from a so-called secret, in our case here from the secret with name `mariadb`.
In this secret the four values are stored under the matching keys (`database-user`, `database-password`, `database-root-password`, `database-name`) and can thus be referenced.

Now let's take a look at the new Secret resource named `mariadb`:

```bash
oc get secret mariadb -o yaml
```

The key-value pairs can be seen under 'data':

```yaml
apiVersion: v1
data:
  database-name: YXBwdWlv
  database-password: YXBwdWlv
  database-root-password: dDB3ZDFLRFhsVjhKMGFHQw==
  database-user: YXBwdWlv
kind: Secret
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftNewApp
    template.openshift.io/expose-database_name: '{.data[''database-name'']}'
    template.openshift.io/expose-password: '{.data[''database-password'']}'
    template.openshift.io/expose-root_password: '{.data[''database-root-password'']}'
    template.openshift.io/expose-username: '{.data[''database-user'']}'
  creationTimestamp: 2018-12-04T10:33:43Z
  labels:
    app: mariadb-ephemeral
    template: mariadb-ephemeral-template
  name: mariadb
  ...
type: Opaque
```

The concrete values are base64 encoded.
Under Linux or in the Git Bash you can easily get the value by means of:

```bash
echo "YXBwdWlv" | base64 -d
appuio
```

Display
In our case `YXBwdWlv` is decoded to `appuio`.

Alternatively, the `oc extract secret/mariadb` command can be used to write the contents of each key into a separate file.
With the parameter `--to=[DIRECTORY]` the directory can be defined in which the values should be written.
Without specification of this parameter the current directory is used.

So with Secrets we can store sensitive information (credentials, certificates, keys, dockercfg, ...) and thus decouple it from the pods.
At the same time, this gives us the possibility to use the same Secrets in multiple containers and thus avoid redundancies.

Secrets can either be mapped into environment variables, as with the MariaDB database above, or mounted directly as files via volumes into a container.

Further information about Secrets can be found in the [official documentation](https://docs.openshift.com/container-platform/latest/nodes/pods/nodes-pods-secrets.html).

## Task: LAB8.2: Connecting the application to the database

By default, our example-spring-boot application uses an H2 memory database.
This can be changed to our new MariaDB service by setting the following environment variables:

- `SPRING_DATASOURCE_USERNAME: appuio`
- `SPRING_DATASOURCE_PASSWORD: appuio`
- `SPRING_DATASOURCE_DRIVER_CLASS_NAME: com.mysql.cj.jdbc.Driver`
- `SPRING_DATASOURCE_URL: jdbc:mysql://[Adresse des MariaDB Service]/appuio?autoReconnect=true`

For the MariaDB service address we can use either its cluster IP (`oc get service`) or its DNS name (`<service>`).
All services and pods within a project can be resolved via DNS.

For example, the value for the variable `SPRING_DATASOURCE_URL` is:

```
Name des Service: mariadb

jdbc:mysql://mariadb/appuio?autoReconnect=true
```

We can now set these environment variables in the DeploymentConfig example-spring-boot.
After the __ConfigChange__ the application is automatically redeployed (ConfigChange is registered as a trigger in the DeploymentConfig).
Due to the new environment variables, the application connects to the MariaDB DB and [Liquibase](http://www.liquibase.org/) creates the schema and imports the test data.

__Note__:
Liquibase is open source.
It is a database independent library to manage and apply database changes.
Liquibase detects at application startup whether DB Changes need to be applied to the database or not.
See Logs.

```bash
SPRING_DATASOURCE_URL=jdbc:mysql://mariadb/appuio?autoReconnect=true
```

__Note__:
MariaDB resolves within your project via DNS query to the cluster IP of the MariaDB service.
The MariaDB database is only accessible within the project.
The service is also accessible via the following name:

```
Projektname = techlab-dockerimage

mariadb.techlab-dockerimage.svc.cluster.local
```

The MariaDB service can be connected via the CLI as follows:

__Note__:
The backslashes (`\`) serve to map the long command more clearly on several lines.
__On Windows__ the so-called multiline delimiter is not the backslash, but in cmd the caret (`^`) and in PowerShell the backtick (`` ` ``).

```bash
oc set env dc example-spring-boot \
    -e SPRING_DATASOURCE_URL="jdbc:mysql://mariadb/appuio?autoReconnect=true" \
    -e SPRING_DATASOURCE_USERNAME=appuio \
    -e SPRING_DATASOURCE_PASSWORD=appuio \
    -e SPRING_DATASOURCE_DRIVER_CLASS_NAME=com.mysql.cj.jdbc.Driver
```

Changing the environment automatically triggers a deployment of the example-spring-boot application.
The pod or container is started with the new environment.

You can view the DeploymentConfig in json using the following command.
The config now also contains the set environment variables:

```bash
oc get dc example-spring-boot -o json
```

```json
...
"env": [
    {
        "name": "SPRING_DATASOURCE_URL",
        "value": "jdbc:mysql://mariadb/appuio?autoReconnect=true&useSSL=false"
    },
    {
        "name": "SPRING_DATASOURCE_USERNAME",
        "value": "appuio"
    },
    {
        "name": "SPRING_DATASOURCE_PASSWORD",
        "value": "appuio"
    },
    {
        "name": "SPRING_DATASOURCE_DRIVER_CLASS_NAME",
        "value": "com.mysql.cj.jdbc.Driver"
    }
],
...
```

The configuration can also be viewed and modified in the Web Console:

- Make sure you are in the Developer view (top left selection).
- Click on Topology
- Select the example-spring-boot DC by clicking on the OpenShift icon
- In the newly opened page window, select "Edit DeploymentConfig" under "Actions" in the upper right corner


### Spring Boot Applikation

Open the application in the browser.

Are the "Say Hello" entries from before still there?

- If yes, why?
- If no, why?

Add a few new "Say Hello" entries.


## Task 2: Secret referencing

We have seen above how OpenShift uses Secrets to decouple sensitive information from the actual configuration and help us avoid redundancy.
Our Springboot application from the previous lab was configured correctly, but the values were redundant and plaintext in the DeploymentConfig.

Let's now adjust the DeploymentConfig example-spring-boot to use the values from the Secrets.
Note the configuration of the containers under `spec.template.spec.containers`.

Using `oc edit dc example-spring-boot -o json` the DeploymentConfig can be edited in JSON format as follows.

```json
...
"env": [
    {
        "name": "SPRING_DATASOURCE_USERNAME",
        "valueFrom": {
            "secretKeyRef": {
                "key": "database-user",
                "name": "mariadb"
            }
        }
    },
    {
        "name": "SPRING_DATASOURCE_PASSWORD",
        "valueFrom": {
            "secretKeyRef": {
                "key": "database-password",
                "name": "mariadb"
            }
        }
    },
    {
        "name": "SPRING_DATASOURCE_DRIVER_CLASS_NAME",
        "value": "com.mysql.cj.jdbc.Driver"
    },
    {
        "name": "SPRING_DATASOURCE_URL",
        "value": "jdbc:mysql://mariadb/appuio?autoReconnect=true"
    }
],
...
```

Now the username and password values are read from the same secret for both the MariaDB pod and the Springboot pod.


### Spring Boot Applikation

Open the application in the browser.

Are the "Say Hello" entries from before still there?

- If yes, why?
- If no, why?

Add a few new "Say Hello" entries.


## Task 3: Connect to DB

As described in Lab [07](07_troubleshooting_ops.md), `oc rsh [POD]` can be used to log into a pod:

```bash
$ oc get pods
NAME                           READY     STATUS             RESTARTS   AGE
example-spring-boot-8-wkros    1/1       Running            0          10m
mariadb-1-diccy                  1/1       Running            0          50m
```

Then log into the MariaDB Pod:

```bash
oc rsh mariadb-1-diccy
```

It is easier to reference it to the pod via DeploymentConfig.

```bash
oc rsh dc/mariadb
```

__Hint__:
If `oc rsh` does not work, open a terminal in the Web Console (Applications -> Pods -> mysql-1-diccy -> Terminal).

Now you can connect to the database using mysql tool:

```bash
mysql -u$MYSQL_USER -p$MYSQL_PASSWORD -h$MARIADB_SERVICE_HOST $MYSQL_DATABASE
```

And with:

```sql
show tables;
```

show all tables.

What does the hello table contain?

<details><summary><b>Hint</b></summary>select * from hello;</details><br/>


## Task 4: Dump einspielen

The task is to import into the MariaDB Pod the [dump](https://raw.githubusercontent.com/appuio/techlab/lab-4/labs/data/08_dump/dump.sql).

__Hint__:
You can use `oc rsync` or `oc cp` to copy local files to a pod.
Alternatively, curl can be used in the MariaDB container.

__Achtung__:
Note that this uses the operating system's rsync command.
On UNIX systems, rsync can be installed using the package manager, on Windows, for example, [cwRsync](https://www.itefix.net/cwrsync).
If it is not possible to install rsync, you can instead log into the pod and download the dump via `curl -O <URL>`.

__Hint__:
Use the tool mysql to import the dump.

__Hint__:
The existing database must be empty beforehand.
It can also be deleted and created again.


### Spring Boot Applikation

Open the application in the browser.

Are the "Say Hello" entries from before still there?

- If yes, why?
- If no, why?

---


## Lösung Task 4

Sync a whole directory (dump).
This contains the file `dump.sql`.
Note also the hint above for the rsync command and the missing trailing slash.

```bash
oc rsync ./labs/data/08_dump mariadb-1-diccy:/tmp/
```

__Hint__:
If `oc rsync` does not work, use `oc cp ./labs/data/08_dump mariadb-1-diccy:/tmp/`.

Log into the MariaDB pod:

```bash
oc rsh dc/mariadb
```

Delete existing database:

```bash
mysql -u$MYSQL_USER -p$MYSQL_PASSWORD -h$MARIADB_SERVICE_HOST $MYSQL_DATABASE
...
mysql> drop database appuio;
mysql> create database appuio;
mysql> exit
```

Import dump:

```bash
mysql -u$MYSQL_USER -p$MYSQL_PASSWORD -h$MARIADB_SERVICE_HOST $MYSQL_DATABASE < /tmp/08_dump/dump.sql
```

What does the hello table contain now?

<details><summary><b>Hint</b></summary>mysql -u$MYSQL_USER -p$MYSQL_PASSWORD -h$MARIADB_SERVICE_HOST $MYSQL_DATABASE<br/>mysql> select * from hello;</details><br/>

__Note__:
You can create the dump as follows:

```bash
mysqldump -u$MYSQL_USER -p$MYSQL_PASSWORD -h$MARIADB_SERVICE_HOST $MYSQL_DATABASE > /tmp/dump.sql
```

---

__Ende Lab 9__

    <p width="100px" align="right"><a href="10_persistent_storage.md">Persistent Storage →</a></p>

[← back to the overview](../README.md)
