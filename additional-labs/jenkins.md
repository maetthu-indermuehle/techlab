# Jenkins Pipelines on OpenShift

With Jenkins Pipelines on OpenShift, it is possible to map complex CI/CD processes in a fully integrated way. In this lab we will show how to work with Jenkins Pipelines to build, test and deploy applications to the different stages in a controlled manner.

The official documentation can be found at [Pipeline build](https://docs.openshift.com/container-platform/latest/builds/build-strategies.html#builds-strategy-pipeline-build_build-strategies) or [OpenShift 3 Pipeline Builds](https://docs.openshift.com/container-platform/3.11/dev_guide/dev_tutorials/openshift_pipeline.html).

The OpenShift platform, since version 4, relies on Tekton for the integrated pipeline builds.
However, these Tekton-based pipelines are only available as a Technology Preview feature [OpenShift 4 Pipeline Builds](https://docs.openshift.com/container-platform/latest/pipelines/understanding-openshift-pipelines.html).

## Basic principle

OpenShift Jenkins pipelines are based on Jenkins pipelines, which act fully integrated with OpenShift. So you have to create BuildConfigs of type `jenkinsPipelineStrategy` which again reference a Jenkins pipeline.

## LAB: Create and run a simple OpenShift pipeline.

To understand how OpenShift pipelines work, let's create a pipeline directly as a first step.

Let's create a new project named `[USER]-buildpipeline`.

<details><summary>tip</summary>oc new-project [USER]-buildpipeline</details><br/>

Now we deploy Jenkins to our project:

```bash
oc get template/jenkins-ephemeral -n openshift -o yaml | oc process --param MEMORY_LIMIT=2Gi --param DISABLE_ADMINISTRATIVE_MONITORS=true -f - | oc apply -f -
```

We create the corresponding BuildConfig with the following command, which directly contains the JenkinsFile, i.e. the pipeline. A second BuildConfig is also created. This contains the BuildConfig for the actual application that we want to deploy as part of this pipeline. In this example, a simple PHP application:

```bash
oc apply -f ./additional-labs/resources/simple-jenkins-pipeline.yaml
```

While OpenShift is working, we still look at the configuration file used: [additional-labs/resources/simple-jenkins-pipeline.yaml](resources/simple-jenkins-pipeline.yaml)

Based on the BuildConfig, OpenShift automatically deploys an integrated Jenkins instance. Let's take a look at this in the Web Console. There is a running Jenkins instance in the project after successful deployment, which is exposed via a route. Access to Jenkins via the route is secured using OpenShift OAuth. Retrieve the URL of the route.

The first time you invoke an OpenShift Jenkins, you will be presented with an OpenShift authentication mask.

![Jenkins OAuth2 Login](../images/jenkins_oauth2_login.png "Jenkins OAuth2 Login")

After that, an authorization screen will be displayed asking you for authorization permissions.
See the image below.

![Jenkins OAuth2 permissions](../images/jenkins_oauth2_permissions.png "Jenkins OAuth2 permissions")

Accept them and go to the next screen by clicking 'Allow selected permissions'.

The next screen should be the famous/classic (or infamous) Jenkins welcome screen.

![Jenkins welcome](../images/pipeline-jenkins-overview.png "Jenkins welcome")

The pipeline from BuildConfig is visible in the folder of our OpenShift project.
It was automatically synchronized and created.

Back in the OpenShift Web Console we can now view the pipeline via Builds --> *appuio-sample-pipeline*.

__Note__: The 'Pipeline build strategy deprecation' is displayed here. As explained in the introduction, OpenShift now relies on Tekton.

To start the pipeline we click on Actions --> Start Build

![Start Pipeline](../images/openshift-pipeline-start.png)

In doing so, OpenShift starts the Jenkins job and synchronizes it accordingly into the pipeline view.

Builds --> *appuio-sample-pipeline* --> Builds Tab --> *appuio-sample-pipeline-1*

![Run Pipeline](../images/openshift-pipeline-run.png)

The same information can be seen in the Jenkins GUI.

![Run Pipeline Jenkins](../images/openshift-pipeline-jenkins-view-run.png)

Our example here contains a test pipeline that illustrates the general principle. However, Jenkins pipelines offer full flexibility to map complex CI/CD pipelines.

```groovy
def project=""
node {
    stage('Init') {
        project = env.PROJECT_NAME
    }
    stage('Build') {
        echo "Build"
        openshift.withCluster() {
            openshift.withProject() {
                def builds = openshift.startBuild("application");
                builds.logs('-f')
                timeout(5) {
                    builds.untilEach(1) {
                    return (it.object().status.phase == "Complete")
                    }
                }
            }
        }
    }
    stage('Test') {
        echo "Test"
        sleep 2
    }
}
node ('maven') {
    stage('DeployDev') {
        echo "Deploy to Dev"
        sleep 5
    }
    stage('PromoteTest') {
        echo "Deploy to Test"
        sleep 5
    }
    stage('PromoteProd') {
        echo "Deploy to Prod"
        sleep 5
    }
}
```

Dhe test pipeline consists of six pipeline stages, which are executed on two Jenkins slaves. The `Build` pipeline stage is currently the only programmed stage. It starts the image build for our application in the current OpenShift project and waits until it is successfully completed.

The last three steps are conditionally executed by the node selector `node ('maven') { ... }` are executed on a Jenkins slave named `maven`. OpenShift dynamically starts a Jenkins slave pod using the Kubernetes plugin and executes these build stages on that slave accordingly.

The `maven` slave is preconfigured by Jenkins in the Kubernetes Cloud plugin. Further down in the Custom Slaves chapter, you will learn how to use custom slaves to run pipelines on them.

## BuildConfig Optionen

In the previous lab, we specified the JenkinsFile directly in the BuildConfig of type `jenkinsPipelineStrategy`. Alternatively, the Jenkins pipeline file can also be stored in the BuildConfig via the Git repository.

```yaml
kind: "BuildConfig"
apiVersion: "v1"
metadata:
  name: "appuio-sample-pipeline"
spec:
  source:
    git:
      uri: "https://github.com/appuio/simple-openshift-pipeline-example.git"
  strategy:
    jenkinsPipelineStrategy:
      jenkinsfilePath: Jenkinsfile
```

## Jenkins OpenShift Plugins

The Jenkins dynamically deployed by OpenShift is fully coupled with OpenShift through a set of OpenShift Jenkins plugins. On the one hand, resources within the project can be accessed directly, on the other hand, dynamic slaves can be set up through appropriate labeling. Furthermore, a corresponding service account (`jenkins`) is also created. Rights can be assigned accordingly via this service account.

Additional information can be found here: <https://docs.openshift.com/container-platform/latest/cicd/builds/build-strategies.html#builds-strategy-pipeline-build_build-strategies>

### OpenShift Jenkins Pipeline

With the OpenShift Jenkins Client Plugin, it is easy to communicate directly with the OpenShift cluster and implement complex CI/CD pipelines as Jenkins files:

```Groovy
openshift.withCluster() {
    openshift.withProject() {
        echo "Hello from project ${openshift.project()} in cluster ${openshift.cluster()}"
    }
}
```

The plugin works as a wrapper to the `oc client`. With the plugin all functions of the client are available.

More information can be found at <https://github.com/openshift/jenkins-client-plugin/blob/master/README.md>.

### OpenShift Jenkins Sync Plugin

The OpenShift Jenkins Sync plugin keeps BuildConfig and Jenkins jobs in sync. It also allows dynamic creation and definition of Jenkins slaves via ImageStream; more on this below.

### Kubernetes Plugin

The Kubernetes plugin is used to dynamically launch the Jenkins slaves in the OpenShift project.

### Custom Slaves

Custom Jenkins slaves can be easily integrated into the build. To do this, the corresponding slaves must be created as ImageStreams and provided with the label `role=jenkins-slave`. These are then automatically registered as pod templates in Jenkins for the Kubernetes plugin. So pipelines can now use `node ('customslave'){ ... }` to run parts of their builds on the corresponding custom slaves.

#### LAB: Use Custom Jenkins Slave as Build Slave

To use custom images as Jenkins slaves you have to define a label `role=jenkins-slave` on the ImageStream.

Based on the label, the Jenkins Sync plugin then synchronizes the configuration for the Kubernetes plugin as pod templates. When Jenkins is started, ImagesStreams labeled in this way are synced and created in Jenkins as Kubernetes Pod Templates.

Let's now create a custom Jenkins slave in our project. To illustrate this, we will use an image of Docker Hub that acts as a Maven Jenkins slave:

```bash
oc import-image custom-jenkins-slave --from=docker.io/openshift/jenkins-slave-maven-centos7 --confirm
```

With this command we have created an ImageStream to this image. We inspect it via the Web Console or with the oc tool.

<details><summary>Tipp</summary>Administrator Web Console: Builds -> Image Streams -> custom-jenkins-slave<br/>oc Tool: oc describe is/custom-jenkins-slave</details><br/>

Afterwards we define by means of the label `role` that it is a Jenkins slave:

```bash
oc label is custom-jenkins-slave role=jenkins-slave
```

After a maximum of 5 minutes, the custom slave is available in Jenkins. The sync mechanism runs only every 5 minutes.

Let's have a look at the configuration in the Jenkins configuration menu: https://[jenkins-route]/configureClouds/.

Our `custom-jenkins-slave` is visible in the pod templates:

![CustomSlave Pipeline Jenkins](../images/pipeline-jenkins-custom-podtemplate.png)

Now the `custom-jenkins-slave` can be used in the pipeline with the following addition in the JenkinsFile:

```groovy
node ('custom-jenkins-slave') {
    stage('custom stage') {
        echo "Running on the custom jenkins slave"
    }
}
```

Extend your pipeline with this stage either in the Jenkins job, the Web Console or in BuildConfig.

__Note__:
The easiest way is to extend the pipeline in the Jenkins GUI.
Select the job `appuio-sample-pipeline` and click on `Configure`.
There you can change the code of the pipeline in an editor window.
Just insert the `custom-jenkins-slave` code snippet at the end of the pipeline.
Start a new build with `Build Now`.

Check that the new stage has been executed.

This mechanism can be used to set up and use any Jenkins slave, including self-built ones. Out-of-the-box a `maven` and `gradle` as well as a `NodeJS` slave are available.

If you want to build your own Jenkins slaves, you should base them on the `openshift/jenkins-slave-base-centos7` image.

## LAB: Multi-Stage Deployment

Next, we now want to further expand our pipeline and tackle the deployment of the application on the different stages (dev, test, prod).

For a multi-stage deployment on OpenShift, the following setup has proven to be best practice:

- One build project, CI/CD environment Jenkins, image builds, S2I builds, ...
- Per stage (dev, test, ..., prod) one project containing the running pods and services, which allows us to keep all environments practically identical

We have already set up the build project above (`[USER]-buildpipeline`). The next step is to create the projects for the different stages:

- [USER]-pipeline-dev
- [USER]-pipeline-test
- [USER]-pipeline-prod

<details>
    <summary>Tipp</summary>
    oc new-project [USER]-pipeline-dev<br/>
    oc new-project [USER]-pipeline-test<br/>
    oc new-project [USER]-pipeline-prod
</details><br/>

Now we have to give the `puller` service account from the corresponding projects per stage the necessary rights so that the built images can be pulled.

```bash
oc policy add-role-to-group system:image-puller system:serviceaccounts:[USER]-pipeline-dev -n [USER]-buildpipeline
oc policy add-role-to-group system:image-puller system:serviceaccounts:[USER]-pipeline-test -n [USER]-buildpipeline
oc policy add-role-to-group system:image-puller system:serviceaccounts:[USER]-pipeline-prod -n [USER]-buildpipeline
```

Furthermore, we now need to give the `jenkins` service account from the `[USER]-buildpipeline` project permissions so that it can create resources in the stage projects.

```bash
oc policy add-role-to-user edit system:serviceaccount:[USER]-buildpipeline:jenkins -n [USER]-pipeline-dev
oc policy add-role-to-user edit system:serviceaccount:[USER]-buildpipeline:jenkins -n [USER]-pipeline-test
oc policy add-role-to-user edit system:serviceaccount:[USER]-buildpipeline:jenkins -n [USER]-pipeline-prod
```

Next, we create the applications in the stage projects. For this, we define a tag in the ImageStream, which is to be deployed. But first, we need to create the corresponding tags that will be deployed:

```bash
oc tag [USER]-buildpipeline/application:latest [USER]-buildpipeline/application:dev
oc tag [USER]-buildpipeline/application:dev [USER]-buildpipeline/application:test
oc tag [USER]-buildpipeline/application:test [USER]-buildpipeline/application:prod
```

### Creating the applications

In each stage, we now create the application based on the previously created ImageStream.

Attention, the applications must be created in the correct project.

<details>
    <summary>Tipp</summary>
    dev: oc new-app [USER]-buildpipeline/application:dev -n [USER]-pipeline-dev<br/>
    test: oc new-app [USER]-buildpipeline/application:test -n [USER]-pipeline-test<br/>
    prod: oc new-app [USER]-buildpipeline/application:prod -n [USER]-pipeline-prod
</details><br/>

After that we can still expose the application.

<details>
    <summary>Tipp</summary>
    dev: oc create route edge --service=application -n [USER]-pipeline-dev<br/>
    test: oc create route edge --service=application -n [USER]-pipeline-test<br/>
    prod: oc create route edge --service=application -n [USER]-pipeline-prod
</details><br/>

In the pipeline, we can now promote and deploy the corresponding image to the appropriate stage by setting a specific tag on the built application's image stream, e.g. `application:dev`.

Customize your pipeline either in the Jenkins job, the Web Console or in BuildConfig as follows (set the values for the variables `dev_project`, `test_project`, `prod_project` accordingly):

```groovy
def project=""
def dev_project="[USER]-pipeline-dev"
def test_project="[USER]-pipeline-test"
def prod_project="[USER]-pipeline-prod"
node {
    stage('Init') {
        project = env.PROJECT_NAME
    }
    stage('Build') {
        echo "Build"
        openshift.withCluster() {
            openshift.withProject() {
                def builds = openshift.startBuild("application")
                builds.logs('-f')
                timeout(5) {
                    builds.untilEach(1) {
                    return (it.object().status.phase == "Complete")
                    }
                }
            }
        }
    }
    stage('Test') {
        echo "Test"
        sleep 2
    }
}
node ('maven') {
    stage('DeployDev') {
        echo "Deploy to Dev"
        openshift.withCluster() {
            openshift.withProject() {
                // Tag the latest image to be used in dev stage
                openshift.tag("$project/application:latest", "$project/application:dev")
            }
            openshift.withProject(dev_project) {
                // trigger Deployment in dev project
                def dc = openshift.selector('deploy', "application")
                dc.rollout().status()
            }
        }
    }
    stage('PromoteTest') {
        echo "Deploy to Test"
        openshift.withCluster() {
            openshift.withProject() {
                // Tag the dev image to be used in test stage
                openshift.tag("$project/application:dev", "$project/application:test")
            }
            openshift.withProject(test_project) {
                // trigger Deployment in test project
                def dc = openshift.selector('deploy', "application")
                dc.rollout().status()
            }
        }
    }
    stage('ApproveProd') {
        input message: 'Deploy to production?',
        id: 'approval'
    }
    stage('PromoteProd') {
        echo "Deploy to Prod"
        openshift.withCluster() {
            openshift.withProject() {
                // Tag the test image to be used in prod stage
                openshift.tag("$project/application:test", "$project/application:prod")
            }
            openshift.withProject(prod_project) {
                // trigger Deployment in prod project
                def dc = openshift.selector('deploy', "application")
                dc.rollout().status()
            }
        }
    }
}
```

Run the pipeline again and see how the built application is now deployed from stage to stage.

![Run Pipeline Jenkins](../images/openshift-pipeline-rollout.png)

## Jenkins Pipeline Language

See <https://github.com/puzzle/jenkins-techlab> for a corresponding hands-on lab on the Jenkins pipeline language. The syntax is described [here](https://jenkins.io/doc/book/pipeline/syntax/).

## Deployment of resources and configuration

So far we have only deployed the application using our deployment pipeline. How can we now use our pipeline to deploy resources (deployments, routes, cronjobs...) as well?

Similarly, above we used the command `oc new-app [USER]-buildpipeline/application:stage -n [USER]-pipeline-stage` to create the necessary resources in the background. In our case `service` and `deployment`.

These resources are configuration, which can develop analogous to the actual application and should also be deployed via CI/CD pipeline.

The configuration values of the environment variables for the configuration of the actual application illustrate this necessity, e.g. to be able to establish the connection to a mail server from the application.

If the parameters in this configuration change on an environment, e.g. user name or password, these values are to be deployed by means of pipeline.

As a basic principle you can imagine it like this:

- Resources are managed as files (`yaml` or `json`) in Git.
- Within the deployment pipeline, these resources are applied to the Kubernetes cluster accordingly

It can even go as far as creating resources that do not yet exist via pipeline, thus setting up the entire environment at the push of a button.

In the Jenkins pipeline, this is then done via the `openshift.apply(r)` command, where the variable `r` corresponds to the corresponding resource.


---

__End__


[← back to the overview](../README.md)

