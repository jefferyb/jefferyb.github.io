---
layout: post
title:  "Getting started with OpenShift"
desc: "Install & Getting started with OpenShift/Kubernetes"
keywords: "OpenShift,Kubernetes,docker,install,blog"
date: 2017-01-10
categories: [OpenShift,Kubernetes,Docker,Linux]
tags: [OpenShift,Kubernetes,docker]
icon: fa-code
---

# Intro

> **OpenShift Origin** is a distribution of Kubernetes optimized for continuous application development and multi-tenant deployment. Origin adds developer and operations-centric tools on top of Kubernetes to enable rapid application development, easy deployment and scaling, and long-term lifecycle maintenance for small and large teams. - [OpenShift Origin](https://github.com/openshift/origin)

The instructions below are used on CentOS 7 using the [OpenShift Ansible Installer](https://github.com/openshift/openshift-ansible).

# OpenShift all-in-one cluster

If instead you'd like to just try out OpenShift on a single node instead, you can set up OpenShift with the `oc cluster up`. Please visit
[https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md)
to starts a local OpenShift all-in-one cluster with a configured registry, router, image streams, and default templates.

In short on Linux:

1. Install Docker with your platform's package manager.
2. Configure the Docker daemon with an insecure registry parameter of `172.30.0.0/16`
   - On RHEL and Fedora, edit the `/etc/sysconfig/docker` file and add or uncomment the following line:
     ```
     INSECURE_REGISTRY='--insecure-registry 172.30.0.0/16'
     ```

   - After editing the config, restart the Docker daemon.
     ```
     $ sudo systemctl restart docker
     ```

   - On Ubuntu, edit `/etc/systemd/system/multi-user.target.wants/docker.service`, find `/usr/bin/dockerd -H fd://` and replace it with:
     ```
     /usr/bin/dockerd --insecure-registry 172.30.0.0/16 -H fd://
     ```

   - After editing the config, restart the Docker daemon.
     ```
     $ sudo systemctl daemon-reload
     $ systemctl restart docker
     ```

3. Download the Linux `oc` binary from
   [openshift-origin-client-tools-VERSION-linux-64bit.tar.gz](https://github.com/openshift/origin/releases)
   and place it in your path.

   > Please be aware that the 'oc cluster' set of commands are only available in the 1.3+ or newer releases.


4. Open a terminal with a user that has permission to run Docker commands and run:
   ```
   $ oc cluster up
   ```
     or with a few more settings, like if you want your data to persist
   ```
   $ oc cluster up --public-hostname $(ip route get 1 | awk '{print $NF;exit}') \
                   --host-config-dir='/opt/origin/openshift.local.config' \
                   --host-data-dir='/opt/origin/openshift.local.data' \
                   --host-volumes-dir='/opt/origin/openshift.local.volumes' \
                   --host-pv-dir='/opt/origin/openshift.local.pv' \
                   --use-existing-config
   ```
   You can set `--public-hostname` with your hostname or IP Address. I used `$(ip route get 1 | awk '{print $NF;exit}')` to get the IP address there...

   Once it's up, you can try it out:

   ```bash
   # stand up a local cluster
   oc cluster up

   # create a project (Kubernetes namespace) for your app
   oc new-project example

   # stand up a CakePHP + MySQL example application using one of the preconfigured OpenShift templates
   oc new-app cakephp-mysql-example

   # Watch the build logs for the CakePHP source code
   oc logs -f bc/cakephp-mysql-example

   # See a summary of the app
   oc status

   # Follow the deployment logs for CakePHP
   oc logs -f dc/cakephp-mysql-example

   # Browse to the route (exposed via the router)
   curl http://cakephp-mysql-example-example.10.1.2.2.xip.io
   ```

   [![Watch the full asciicast](https://raw.githubusercontent.com/openshift/origin/master/docs/openshift-intro.gif)](https://asciinema.org/a/49402)


To stop your cluster, run:
```
$ oc cluster down
```

# The Servers

The following info is a short version of a document by [Dusty Mabe](http://www.projectatomic.io/blog/author/dustymabe/), which can be found here [Installing an OpenShift Origin Cluster on Fedora 25 Atomic Host: Part 1](http://dustymabe.com/2016/12/07/installing-an-openshift-origin-cluster-on-fedora-25-atomic-host-part-1/) and [http://dustymabe.com/2016/12/12/installing-an-openshift-origin-cluster-on-fedora-25-atomic-host-part-2/](http://dustymabe.com/2016/12/12/installing-an-openshift-origin-cluster-on-fedora-25-atomic-host-part-2/) so for a more in depth, please visit those links...

In this example, I'll be using 3 nodes running CentOS 7... Once you have your nodes, collect your Public and Private IP addresses.

Role | Public IPv4 | Private IPv4
-----|-------------|-------------
*master,etcd* | 35.185.26.245 | 10.142.0.2
*worker* | 104.196.156.116 | 10.142.0.3
*worker* | 104.196.99.84 | 10.142.0.4

# The Installer

Here's a screencast of the installation *You can pause, copy and paste from the screencast video if you want*

<script type="text/javascript" src="https://asciinema.org/a/99046.js" id="asciicast-99046" async></script>

OpenShift uses Ansible v2.2 or greater to manage nodes in the cluster.

```bash
# First, we need to clone the OpenShift Ansible repo
$ git clone https://github.com/openshift/openshift-ansible.git
$ cd openshift-ansible
# Create an inventory file, openshift_inventory
$ cat > openshift_inventory <<EOF
# Create an OSEv3 group that contains the masters and nodes groups
[OSEv3:children]
masters
nodes
etcd

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
ansible_user=jefferyb
ansible_become=true
deployment_type=origin
containerized=true
openshift_release=v1.3.2
openshift_router_selector='router=true'
openshift_registry_selector='registry=true'
openshift_master_default_subdomain=104.196.99.84.xip.io

# enable htpasswd auth
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]
openshift_master_htpasswd_users={'admin': '$apr1$zgSjCrLt$1KSuj66CggeWSv.D.BXOA1', 'user': '$apr1$.gw8w9i1$ln9bfTRiD6OwuNTG5LvW50'}

# host group for masters
[masters]
35.185.26.245 openshift_public_hostname=35.185.26.245 openshift_hostname=10.142.0.2

# host group for etcd, should run on a node that is not schedulable
[etcd]
35.185.26.245

# host group for worker nodes, we list master node here so that
# openshift-sdn gets installed. We mark the master node as not
# schedulable.
[nodes]
35.185.26.245   openshift_hostname=10.142.0.2 openshift_schedulable=false
104.196.156.116 openshift_hostname=10.142.0.3 openshift_node_labels="{'router':'true','registry':'true'}"
104.196.99.84   openshift_hostname=10.142.0.4 openshift_node_labels="{'router':'true','registry':'true'}"
EOF

```

Make sure that you change things around, like
* The IP addresses to match yours
* ansible_user=jefferyb - The user that you use to connect
* openshift_release=v1.3.2 - The version of Origin to run
* openshift_master_htpasswd_users - Users you want to create

The default passwords used above are

  * *admin* -> **OriginAdmin**
  * *user* -> **OriginUser**

Generate the passwords by running `htpasswd` on the command line like so:

```bash
$ htpasswd -bc /dev/stdout admin OriginAdmin
Adding password for admin user
admin:$apr1$zgSjCrLt$1KSuj66CggeWSv.D.BXOA1
$ htpasswd -bc /dev/stdout user OriginUser
Adding password for user user
user:$apr1$.gw8w9i1$ln9bfTRiD6OwuNTG5LvW50
```

# Executing the Installer

First, let's make sure that we can contact the servers...

```bash
$ ansible -i openshift_inventory nodes -m ping
35.185.26.245 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
104.196.99.84 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
104.196.156.116 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

You can add ` --private-key=~/.ssh/private_key.pem` if you're using a different key file...

If it's all and good, then we run the installer...

```bash
$ ansible-playbook -i openshift_inventory playbooks/byo/config.yml
Using /etc/ansible/ansible.cfg as config file
....
....  snip all of the output
....
PLAY RECAP *********************************************************************
104.196.156.116               : ok=162  changed=49   unreachable=0    failed=0   
35.185.26.245                : ok=540  changed=150  unreachable=0    failed=0   
104.196.99.84             : ok=159  changed=49   unreachable=0    failed=0   
localhost                  : ok=15   changed=9    unreachable=0    failed=0

```

You should have your system up and running, but before we get started, we need to give `admin` user we created earlier, the cluster admin role in OpenShift.
The command is `oadm policy add-cluster-role-to-user cluster-admin admin` or `oc adm policy add-cluster-role-to-user cluster-admin admin`, using the `oc` command.

```bash
$ ansible -i openshift_inventory masters -a '/usr/local/bin/oadm policy add-cluster-role-to-user cluster-admin admin'
35.185.26.245 | SUCCESS | rc=0 >>
```

# oc Command & bash auto completion

Download the Linux `oc` binary from [openshift-origin-client-tools-VERSION-linux-64bit.tar.gz](https://github.com/openshift/origin/releases) and place it in your path.

```bash
$ wget https://github.com/openshift/origin/releases/download/v1.3.2/openshift-origin-client-tools-v1.3.2-ac1d579-linux-64bit.tar.gz
$ tar zxvf openshift-origin-client-tools-v1.3.2-ac1d579-linux-64bit.tar.gz
$ sudo mv openshift-origin-client-tools-v1.3.2-ac1d579-linux-64bit/oc /usr/local/bin/
$ echo "curl -G https://raw.githubusercontent.com/openshift/origin/master/contrib/completions/bash/oc > /etc/bash_completion.d/oc" | sudo bash
```

# Login
And now we are ready to log in as either the `admin` or `user` users using `oc login https://35.185.26.245:8443` from the command line or visiting the web frontend at `https://35.185.26.245:8443`.

```bash
$ oc login --insecure-skip-tls-verify -u admin -p OriginAdmin https://35.185.26.245:8443
Login successful.

You have access to the following projects and can switch between them with 'oc project <projectname>':

  * default
    kube-system
    logging
    management-infra
    openshift
    openshift-infra

Using project "default".
Welcome! See 'oc help' to get started.
```

Let's look around:

```bash
$ oc get nodes
NAME           STATUS                     AGE
10.142.0.3    Ready                      9h
10.142.0.2   Ready,SchedulingDisabled   9h
10.142.0.4   Ready                      9h

$ oc get pods -o wide
NAME                       READY     STATUS    RESTARTS   AGE       IP             NODE
docker-registry-3-hgwfr    1/1       Running   0          9h        10.129.0.3     10.142.0.3
registry-console-1-q48xn   1/1       Running   0          9h        10.129.0.2     10.142.0.3
router-1-nwjyj             1/1       Running   0          9h        10.142.0.3    10.142.0.3
router-1-o6n4a             1/1       Running   0          9h        10.142.0.4   10.142.0.4

$ oc get svc
NAME               CLUSTER-IP       EXTERNAL-IP   PORT(S)                   AGE
docker-registry    172.30.2.89      <none>        5000/TCP                  9h
kubernetes         172.30.0.1       <none>        443/TCP,53/UDP,53/TCP     9h
registry-console   172.30.147.190   <none>        9000/TCP                  9h
router             172.30.217.187   <none>        80/TCP,443/TCP,1936/TCP   9h

$ oc get routes
NAME               HOST/PORT                                        PATH      SERVICES           PORT               TERMINATION
docker-registry    docker-registry-default.104.196.99.84.xip.io              docker-registry    5000-tcp           passthrough
registry-console   registry-console-default.104.196.99.84.xip.io             registry-console   registry-console   passthrough

```

# Running an Application as a Normal User

Now we can try to run an application as a normal user

```bash
# Login
$ oc login --insecure-skip-tls-verify -u user -p OriginUser https://35.185.26.245:8443                                                                                        
Login successful.

You do not have any projects. You can try to create a new project, by running

    oc new-project <projectname>


# Create a new project/namespace, myproject
$ oc new-project myproject
Now using project "myproject" on server "https://35.185.26.245:8443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git

to build a new example application in Ruby.


# Try out a ruby application with source code
$ oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git
--> Found Docker image ecd5025 (10 hours old) from Docker Hub for "centos/ruby-22-centos7"

    Ruby 2.2
    --------
    Platform for building and running Ruby 2.2 applications

    Tags: builder, ruby, ruby22

    * An image stream will be created as "ruby-22-centos7:latest" that will track the source image
    * A source build using source code from https://github.com/openshift/ruby-ex.git will be created
      * The resulting image will be pushed to image stream "ruby-ex:latest"
      * Every time "ruby-22-centos7:latest" changes a new build will be triggered
    * This image will be deployed in deployment config "ruby-ex"
    * Port 8080/tcp will be load balanced by service "ruby-ex"
      * Other containers can access this service through the hostname "ruby-ex"

--> Creating resources with label app=ruby-ex ...
    imagestream "ruby-22-centos7" created
    imagestream "ruby-ex" created
    buildconfig "ruby-ex" created
    deploymentconfig "ruby-ex" created
    service "ruby-ex" created
--> Success
    Build scheduled, use 'oc logs -f bc/ruby-ex' to track its progress.
    Run 'oc status' to view your app.


# Check the status of the application
$ oc status
In project myproject on server https://35.185.26.245:8443

svc/ruby-ex - 172.30.213.94:8080
  dc/ruby-ex deploys istag/ruby-ex:latest <-
    bc/ruby-ex source builds https://github.com/openshift/ruby-ex.git on istag/ruby-22-centos7:latest
      build #1 running for 26 seconds
    deployment #1 waiting on image or update

1 warning identified, use 'oc status -v' to see details.


# After some time, you can see that the build is done and that the deployment is running
$ oc status
In project myproject on server https://35.185.26.245:8443

svc/ruby-ex - 172.30.213.94:8080
  dc/ruby-ex deploys istag/ruby-ex:latest <-
    bc/ruby-ex source builds https://github.com/openshift/ruby-ex.git on istag/ruby-22-centos7:latest
    deployment #1 running for 6 seconds

1 warning identified, use 'oc status -v' to see details.


# And after some time, a Pod is running
$ oc status
In project myproject on server https://35.185.26.245:8443

svc/ruby-ex - 172.30.213.94:8080
  dc/ruby-ex deploys istag/ruby-ex:latest <-
    bc/ruby-ex source builds https://github.com/openshift/ruby-ex.git on istag/ruby-22-centos7:latest
    deployment #1 deployed about a minute ago - 1 pod

1 warning identified, use 'oc status -v' to see details.


# List the pods
$ oc get pods
NAME              READY     STATUS      RESTARTS   AGE
ruby-ex-1-build   0/1       Completed   0          13m
ruby-ex-1-mo3lb   1/1       Running     0          11m


# Now we need to expose the service so that we can access it from the outside
$ oc expose svc/ruby-ex
route "ruby-ex" exposed
$ oc get route/ruby-ex
NAME      HOST/PORT                                 PATH      SERVICES   PORT       TERMINATION
ruby-ex   ruby-ex-myproject.104.196.99.84.xip.io             ruby-ex    8080-tcp   

```

Now you can point the browser to `ruby-ex-myproject.104.196.99.84.xip.io`

For more information, please check [oc by example](https://github.com/openshift/origin/blob/master/docs/generated/oc_by_example_content.adoc) and [OpenShift Command-Line Interface](https://github.com/openshift/origin/blob/master/docs/cli.md)
