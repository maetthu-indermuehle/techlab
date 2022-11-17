# Cron Jobs in OpenShift

Kubernetes brings the concept of jobs and cron jobs.
This makes it possible to execute certain tasks once (job) or at a certain time (cron job).

A possible selection of use cases:

- Create a database backup every 23:12 and save it to a mounted PVC.
- Generate reports once
- Cleanup job which cleans up old data
- Asynchronous sending of emails

## Job

Jobs, unlike a deployment that is tracked by the Replication Controller, run a Pod once until the command completes.
A job creates a pod and executes the defined operation or command.
This does not necessarily have to be just one Pod, but can also include several.
If a job is deleted, the pods started (and ended) by the job are also deleted.

A job can therefore be used, for example, to ensure that a pod is reliably executed until it is completed.
If a pod fails, for example because of a node error, the job starts a new pod.

More information about jobs can be found in the [OpenShift Documentation](https://docs.openshift.com/container-platform/latest/nodes/jobs/nodes-nodes-jobs.html).

## Cron Jobs

An OpenShift Cron job is nothing more than a resource that creates a job at defined times, which in turn starts a Pod as usual to execute a command.

More information about cron jobs can be found on the same [OpenShift documentation page](https://docs.openshift.com/container-platform/latest/nodes/jobs/nodes-nodes-jobs.html) as the jobs.

## Task: Job für MariaDB-Dump erstellen

Similar to [Lab-Task 9.4](../labs/09_database.md), we now want to create a dump of the running MariaDB database, but without having to log into the pod.

For this example we use the Spring Boot example from [Lab 4](../labs/04_deploy_dockerimage.md), `[USERNAME]-dockerimage`.

<details><summary><b>Hint</b></summary>oc project [USERNAME]-dockerimage</details>

First, let's take a look at the job resource we want to create.
It can be found at [resources/job_mariadb-dump.yaml](resources/job_mariadb-dump.yaml).
Under `.spec.template.spec.containers[0].image` we see that we use the same image as for the running database itself.
However, we do not start a database afterwards, but want to execute a `mysqldump` command, as listed under `.spec.template.spec.containers[0].command`.
To do this, we use the same environment variables as in the database deployment, to be able to define hostname, user or password within the pod.

If the job fails, it should be restarted, this is defined by the `restartPolicy`.
The job should be tried 3 times in total (`backoffLimit`).

So now let's create our job:

```bash
oc create -f ./additional-labs/resources/job_mariadb-dump.yaml
```

Let's check if the job was successful:

```bash
oc describe jobs/mariadb-dump
```

We can look at the executed pod as follows:

```bash
oc get pods
```

To output all pods belonging to a job in machine-readable form, the following command can be used, for example:

```bash
oc get pods --selector=job-name=mariadb-dump --output=jsonpath={.items..metadata.name}
```

To check if the job was successful, the logs of the Pod can be read.

```bash
oc logs $(oc get pods --selector=job-name=mariadb-dump --output=jsonpath={.items..metadata.name})
```


## Task: Set up cron job

In the previous task, we only instantiated a job that creates a database dump once.
Now we want to make sure that this database dump is executed once a night.

For this purpose, we now create a resource of the type Cron Job. The cron job should execute a job at 23:12 every day, which creates and saves a database dump.

```bash
oc create -f ./additional-labs/resources/cronjob_mariadb-dump.yaml
```

Now let's take a look at this cron job:

```bash
oc get cronjob mariadb-backup -o yaml
```

__Important__:
Note that backups in particular must be monitored and checked by restore tests.
This logic can be integrated into the command to be executed, for example, or taken over by a monitoring tool.
In the test cron job, the dump is written to the `/tmp` directory.
For productive use, this should be a mounted persistent volume.

Try to answer the following questions:

- When was the last time the cron job was run?
- Was the backup successful?
- Could the data be restored successfully?

__Ende Lab Cron Jobs__

[← back to the overview](../README.md)

