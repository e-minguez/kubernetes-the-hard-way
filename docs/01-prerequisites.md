# Prerequisites

## OpenStack Platform

This tutorial leverages the [OpenStack Platform](https://www.openstack.org/) to
streamline provisioning of the compute infrastructure required to bootstrap a
Kubernetes cluster from the ground up.

The OpenStack Platform environment requires a project (tenant) to store the
required instances that are to install Kubernetes. This project requires
ownership by a user and role of that user to be set to member.

> Those steps are required to be executed by an OpenStack administrator

Create a project (tenant) that is to store the OpenStack instances

```
openstack project create kubernetes-the-hard-way
```

Create an OpenStack Platform user that has ownership of the previously created project:

```
openstack user create --password <password> <username>
```

Set the role of the user:

```
openstack role add --user <username> --project kubernetes-the-hard-way member
```

## OpenStack CLI

### Install the OpenStack CLI

Follow the OpenStack CLI [documentation](https://docs.openstack.org/newton/user-guide/common/cli-install-openstack-command-line-clients.html) to install and configure the `openstack`
command line utility in your workstation.

Verify the OpenStack CLI version is 3.12.0 or higher:

```
openstack --version
```

### Set environment variables using the OpenStack RC file

To set the required environment variables for the OpenStack command-line clients, you must create an environment file called an OpenStack rc file, or openrc.sh file. If your OpenStack installation provides it, you can download the file from the OpenStack dashboard as an administrative user or any other user. This project-specific environment file contains the credentials that all OpenStack services use.

When you source the file, environment variables are set for your current shell. The variables enable the OpenStack client commands to communicate with the OpenStack services that run in the cloud.

Follow the OpenStack CLI [documentation](https://docs.openstack.org/newton/user-guide/common/cli-set-environment-variables-using-openstack-rc.html) to properly configure your `openstack` CLI.

After downloading or creating the OpenStack RC file, it is required to use it
by sourcing it:

```
. ~/openstack/openrc.sh
```
Test if working by trying to gather the instances running:

```
openstack server list
```

### OpenStack keypair
OpenStack Platform uses cloud-init to place an ssh public key on each instance as it is created to allow ssh access to the instance. OpenStack Platform expects the user to hold the private key.

In order to generate a keypair use the following command:

```
openstack keypair create k8s-the-hard-way > ~/.ssh/k8s.pem
```

Once the keypair is created, set the permissions to 600 thus only allowing the owner of the file to read and write to that file.

```
chmod 600 ~/.ssh/k8s.pem
```

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with `synchronize-panes` enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable synchronize-panes by pressing `ctrl+b` followed by `shift+:`. Next type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
