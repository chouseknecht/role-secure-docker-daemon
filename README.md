[![Build Status](https://travis-ci.org/chouseknecht/adb-up-role.svg?branch=master)](https://travis-ci.org/chouseknecht/adb-up-role)

# adb-up-role

Create an [Atomic Developer Bundle (ADB)](https://github.com/projectatomic/adb-atomic-developer-bundle) virtual machine configured to work with [Ansible Container](https://github.com/ansible/ansible-container).

Specifically, it performs the following tasks:

- Downloads the [OpenShift Vagrantfile](https://raw.githubusercontent.com/projectatomic/adb-atomic-developer-bundle/master/components/centos/centos-openshift-setup/Vagrantfile), and updates it with the desired OpenShift version
- Creates a new VM
- Installs require Vagrant plugins
- Downloads and installs the latest OpenShift client
- On the host machine, adds an entry to */etc/hosts* for *openshift.adb*, and points it to the IP address of the new VM
- Generates new client and server certificates for the Docker daemon running on the VM. Certificates are installed on the VM, and copied to the host. The new certs are created with *openshift.adb* as the server name, and the IP address as an alternate name.
- Restarts the Docker daemon on the VM
- Restarts the OpenShift cluster
- On the host, creates *setenv.sh*, a script you can source to set your shell environment to use the Docker daemon running inside the new VM
- Grants cluster admin access to the *openshift-dev* account
- Logs into OpenShift as user *openshift-dev*, and set the project to *default* 

As described below in *Running the Role*, use this role by running the included playbook, [adb-up.yml](./files/adb-up.yml). 

## Supported Platforms 

This role has been tested on the following platforms:

- OSX 
- Red Hat Linux

## Prerequisites

Before you can use this role, you need to have the following installed:

- [Ansible 2.1+](https://github.com/ansible/ansible)
- Virtualbox 5.0.26+
- Vagrant 1.9.1+

You will also need *sudo* access to your local machine in order to install the *oc* client, and update */etc/hosts*.  

If you want to deploy an Ansible Container project to the OpenShift cluster, you'll need Ansible Container 0.3.0. See [Installing from Source](http://docs.ansible.com/ansible-container/installation.html#running-from-source), if you need assistance.

## Running the Role

The best way to use this role is to create a new project directory, and then copy and run the playbook, [adb-up.yml](./files/adb-up.yml), included with the role. The following, example demonstrates this.

But first, a couple notes:

- When running the playbook, be sure to include the *--ask-sudo-pass* option, and then provide your password at the prompt.
- The following example uses the environment variable *ANSIBLE_ROLES_PATH*. You can also set this path using an *ansible.cfg* file. View [Ansible roles_path](http://docs.ansible.com/ansible/intro_configuration.html#roles-path), for more information.

```
# Install the role to your ANSIBLE_ROLES_PATH
$ ansible-galaxy install chouseknecht.adb-up

# Create a new project directory in your home directory
$ mkdir ~/adb 

# Set the working directory to the new directory
$ cd ~/adb

# Copy the included playbook
$ cp $ANSIBLE_ROLES_PATH/chouseknecht.adb-up/files/adb-up.yml . 

# Run the playbook
$ ansible-playbook adb-up.yml --ask-sudo-pass
```
Click the following image to watch a video showing how to run the role:

[![Running the role](https://github.com/chouseknecht/adb-up-role/blob/images/images/adb_up_role.png)](https://youtu.be/8HFKxZSP5A8)

After the role completes you will have a running VM that hosts a Docker daemon, and an OpenShift cluster. To access the cluster, open ``https://openshift.adb:8443/console`` in a browser. The username is ``openshift-dev``, and the password is ``devel``.

To use the Docker daemon, source the script ``setenv.sh`` that was added to your project directory. This will set the DOCKER* environment variables in your shell.

## Deploying your Ansible Container project 

Start by first sourcing the script, *setenv.sh*, which was generated by the role. It will set DOCKER* environment variables in your shell to point the Docker daemon running in the VM. You'll then build your project images using the Docker daemon found on the VM, generate your deployment playbook and role, and then run the playbook. 

In the following example we'll create a new project, install the Container Enabled role [jenkins-container](https://galaxy.ansible.com/awasilyev/jenkins-container/), and deploy the Jenkins service to our local OpenShift cluster.

**NOTE**: to run this example, you will need to install Ansible Container 0.3.0. See [Installing from Source](http://docs.ansible.com/ansible-container/installation.html#running-from-source), if you need assistance.

```
# Set the DOCKER* variables 
$ source setenv.sh 

# Create a new project folder
$ mkdir jenkins 

# Set the working directory 
$ cd jenkins 

# Init the project 
$ ansible-container init 

# Install the jenkins-container role 
$ ansible-container install awasilyev.jenkins-container

# Build the images on the ADB virtual machine
$ ansible-container --no-selinux build 

# Generate the deployment playbook and role 
$ ansible-container --no-selinux shipit openshift --local-images

# Set the working directory to ansible
$ cd ansible

# Run the shipit playbook
$ ansible-playbook shipit-openshift.yml
```

To view the project, use a browser to log into the OpenShift console at `https://openshift.adb:8443/console`. The username is `openshift-dev`, and the password is `devel`. If you followed the example above, you will see a `jenkins` project. Click on `jenkins` to open the project overview.

Click the following image to watch a video of the Jenkins service deployment:

[![Deploying Jenkins](https://github.com/chouseknecht/adb-up-role/blob/images/images/jenkins-deployment.png)](https://youtu.be/FQY8hQ-cB1c)

### A couple of notes:

- The *--no-selinux* option is new in v0.3.0. It stops the additon of the 'Z' option to volumes automatically attached to the build container. In this case, it should not be included on the project directory, which gets mounted as */ansible-container*. 

- The *--local-images* option used with `shipit` is new in v0.3.0 as well. It forces the use of images local to the Docker daemon, rather than pulling them from the engine's default source, which in this case would be Docker Hub. 

## Role Variables
adb_server_certpath: /etc/docker
> Where to place generated service certificates on the VM.

adb_client_certpath: ./.vagrant/machines/default/virtualbox/docker
> Where to place generated client certificates on the host.

adb_hostname: openshift.adb
> The name to give add to the host's */etc/hosts* file and associate witht th VM.

adb_passphrase: opensesame
> Password used when generating the certifiates.

adb_country: US
> Country to use for certificate signing requests.

adb_state: North Carolina
> State to use for certificate signing requests.
 
adb_locality: Durham
> Locality to use for certificate signing requests.

adb_organization: Acme Corp
> Organization name to use for certificate signing requests.

adb_restart_docker: yes
> Restart the Docker daemon after generating new certificates. 

adb_force_create: yes
> Destroy any pre-existing VM found in the project directory. 

adb_openshift_version: v1.3.2
> Version of OpenShift to install on the VM. 

adb_github_url: https://api.github.com/repos
> Where to get the OpenShift client from.

openshift_repo: openshift/origin
> Where to get the OpenShift client from.

openshift_client_dest: /usr/local/bin
> Where to install the OpenShift client. If you change this, be sure to specify a directory that's included in your PATH environment variable.

openshift_force_client_install: yes
> Overwrite any existing `oc` binary found in the *openshift_client_dest* path.

## Known issues

### Ruby gem error

If you're running on OSX (and possibly other platforms), you'll see the following message:

```
Ignoring eventmachine-1.0.9.1 because its extensions are not built.  Try: gem pristine eventmachine --version 1.0.9.1
```

This seems to be coming from the [Landrush](https://github.com/vagrant-landrush/landrush) plugin. See issue [#292](https://github.com/vagrant-landrush/landrush/issues/292). And, it seems safe to ignore. The VM runs fine, and `vagrant` commands work.

### Using service-manager to set the envirionment

If you read the ADB documentation, you'll see references to the command `vagrant service-manager env`. Do **NOT** use this command. It generates new certificates when you run it, and the certificates will not be bound to the *openshift.adb* host. They're bound to *example.com*, which works fine for the `docker` command, but breaks Ansible Container.

Rather than using service-manager, use the command `source setenv.sh`. This will execute the script generated by this role, setting the DOCKER variables in your environment to point to the Docker daemon running inside the ADB virtual machine.

### Missing a persistent volume

There seems to be no documentation at the ADB project for how to create persistent volumes. In fact, there is an [open issue](https://github.com/projectatomic/adb-atomic-developer-bundle/issues/117) describing the difficulty in creating one while SELinux is enabled for the Docker daemon. It does however present a complicated workaround, which I have not yet tested.


## License

[Apache v2](./LICENSE)

## Author

[@chouseknecht](https://github.com/chouseknecht)
