# Lab 11: Code Changes through Webhook Trigger Rebuild on OpenShift

In this lab, we will demonstrate the Docker build workflow with an example and you will learn how to automatically start a build and deployment of the application on OpenShift with a push to the Git repository.


## Task 1: Preparation Github account and fork

### Github Account

To be able to make changes to the source code of our sample application, you need your own GitHub account.
Set up an account at <https://github.com/> if you don't already have one.


### Example project fork

__Example project__: <https://github.com/appuio/example-php-docker-helloworld>

Go to the [GitHub project page](https://github.com/appuio/example-php-docker-helloworld) and [fork](https://help.github.com/articles/fork-a-repo/) the project.

![Fork](../images/lab_09_fork_example.png)

You now have a fork of the Example project at `https://github.com/[YourGitHubUser]/example-php-docker-helloworld` that you can extend as you wish.


## Deploying your own fork

Create a new project named `[USERNAME]-example4`.

<details><summary><b>Hint</b></summary>oc new-project [USERNAME]-example4</details><br/>

Create a new app for your fork with the following configuration:

* Name: `appuio-php-docker-ex`
* Build Strategie: `docker`
* Git Repository: `https://github.com/[YourGithubUser]/example-php-docker-helloworld.git`

__Note__:
Replace `[YourGithubUser]` with the name of your GitHub account.

<details>
  <summary>Create application command</summary>
  oc new-app --as-deployment-config https://github.com/[YourGithubUser]/example-php-docker-helloworld.git --strategy=docker --name=appuio-php-docker-ex
</details><br/>

Using the `--strategy=docker` parameter, we now explicitly tell the `oc new-app` command to search for a Dockerfile in the specified Git repository and use it for the build.

Now expose the service of the application.

<details><summary>Create Route Command</summary>oc create route edge --service=appuio-php-docker-ex</details><br/>


## Task 2: Set up webhook on GitHub

When the app was built, webhooks were defined directly in the BuildConfig (bc).
You can copy these webhooks from the Web Console.
To do this, go to the "appuio-php-docker-ex" build via the "Builds" menu item.
You will now see the webhooks on the page below.

![Webhook](../images/lab_09_webhook_ocp4.png)

Copy the [GitHub Webhook URL](https://developer.github.com/webhooks/) by right clicking on "Copy URL with Secret".

In your GitHub project, click Settings:
![Github Webhook](../images/lab_09_webhook_github1.png)

Click Webhooks:
![Github Webhook](../images/lab_09_webhook_github2.png)

Add a webhook:
![Github Webhook](../images/lab_09_webhook_github3.png)

Now paste the copied GitHub Webhook URL:
![Github Webhook](../images/lab_09_webhook_github4.png)

From now on, all pushes to your GitHub repository trigger a build and then deploy the code changes directly to the OpenShift platform.


## Task 3: Customize code

Clone your Git repository and change to the code directory:

```bash
git clone https://github.com/[YourGithubUser]/example-php-docker-helloworld.git
cd example-php-docker-helloworld
```

Adjust the file `./app/index.php` for example on line 56:

```bash
vim app/index.php
```

![Github Webhook](../images/lab_09_codechange1.png)

```html
    <div class="container">

      <div class="starter-template">
        <h1>Hallo <?php echo 'OpenShift Techlab'?></h1>
        <p class="lead">APPUiO Example Dockerfile PHP</p>
      </div>

    </div>
```

Push your Change:

```bash
git add .
git commit -m "Update hello message"
git push
```

Alternatively, you can edit the file directly on GitHub:
![Github Webhook](../images/lab_09_edit_on_github.png)

Once you have pushed the changes, OpenShift starts a build of the new source code.

```bash
oc get builds
```

and then deployed the changes.

## Task 4: Rollback

With OpenShift, different software states can be enabled and disabled by simply launching a different version of the image.

The `oc rollback` and `oc rollout` commands are used for this.

To perform a rollback, you need the name of the DeploymentConfig:

```bash
oc get dc

NAME                   REVISION   DESIRED   CURRENT   TRIGGERED BY
appuio-php-docker-ex   4          1         1         config,image(appuio-php-docker-ex:latest)
```

You can now use the following command to roll back to the previous version:

```bash
oc rollback appuio-php-docker-ex
#3 rolled back to appuio-php-docker-ex-1
Warning: the following images triggers were disabled: appuio-php-docker-ex:latest
  You can re-enable them with: oc set triggers dc/appuio-php-docker-ex --auto
```

As soon as the deployment of the old version is done, you can check via your browser if the original headline __Hello APPUiO__ is displayed again.

__Hint__:
Automatic deployments of new versions are now disabled for this application to prevent unwanted changes after rollback. To re-enable automatic deployment, run the following command:

```bash
oc set triggers dc/appuio-php-docker-ex --auto
```

---

__Ende Lab 11__

<p width="100px" align="right"><a href="12_template.md">Application templates →</a></p>

[← back to the overview](../README.md)
