# Lab 2: OpenShift CLI installation 

In this lab, we will work together to install and configure the `odo` CLI tool so that we can then take our first steps on the OpenShift Techlab platform.

## `oc`

The __oc client__ provides an interface to OpenShift.

### Installation

According to the [official documentation](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html#cli-installing-cli_cli-developer-commands) the __oc client__ can be downloaded from the __Infrastructure Provider__ , or simpler from the following URL: <https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest/>

__Hint__:
Alternatively, the binary can be installed using the following commands in the terminal on linux:

```bash
curl -fsSL https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/linux/oc.tar.gz | sudo tar xfz - -C /usr/bin
```

### bash Command Completion (optional)

This step is optional and does not work on Windows. For Command Completion to work on macOS, the `bash-completion` package must be installed via `brew`, for example.

`oc` offers a Command Completion, which can be installed according to the [documentation](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/configuring-cli.html#cli-enabling-tab-completion_cli-configuring-cli).

__Hint__:
Alternatively, the Bash Command Completion can be installed using the following commands in the terminal:

```bash
oc completion bash | sudo tee /etc/bash_completion.d/oc_bash_completion
```

__Ende Lab 2__

<p width="100px" align="right"><a href="03_first_steps.md">First Steps on the platform →</a></p>

[← back to the overview](../README.md)
