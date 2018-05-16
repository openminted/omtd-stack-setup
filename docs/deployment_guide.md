Deployment guide
================

This document will guide you to deploy the Open MinTeD stack
(omtd-stack) on a (virtual) cluster of ubuntu and debian hosts. It has
been deployed and tested on GRNET's "~okeanos" clusters, which run
"Synnefo".

1. [Services](#services)
2. [Provision](#provision)
3. [Install with ansible](#install-with-ansible)
4. [Executor](#executor)
5. [LVM for executor](#lvm-for-executor)
6. [Editor](#editor)
7. [Mesos and Chronos](#mesos-and-chronos)
8. [Cluster nodes](#cluster-nodes)
9. [Reverse proxy and SSL](#reverse-proxy-and-ssl)
10. [SSL without Proxy](#ssl-without-proxy)

Services
--------

If all services are split as much as possible, then::

- Executor Galaxy - executes TDM (Text and Data Mining) tools and workflows, used through REST API
- Editor Galaxy - offers an editor for creating and editing workflows
- NFS Server - offers a shared folder for the tools between the two galaxies, as well as a shared folder between the Executor Galaxy and the cluster nodes
- Chronos scheduler - schedulers the execution of tools
- Mesos cluster manager - manages the cluster that executes the tools
- Mesos slave - handles execution on slave nodes
- CAdvisor - monitors the execution of tools on the hosts they are executed
- Prometheus-node-exporter - exports the results of the monitoring to the Prometheus aggregator
- Prometheus - monitors the Mesos slaves and offers the results through a REST API
- Grafana - visualization of Prometheus monitoring
- Docker Registry - hosts the images of the TDM tools

Suggested setup per host::

- Executor: Executor Galaxy, NFS server
- Server Editor: Editor Galaxy
- Cluster Manager: Chronos scheduler, Mesos master
- Cluster Nodes: Mesos slave, CAdvisor, Node-Exporter
- Monitoring: Prometheus + Grafana
- Tool Registry: Docker Registry

Note: You can start with 2 or 3 nodes. It is easy to scale up (or down)
later.

Provision
---------

Make sure to create the following VMs

- Executor VM: Ubuntu 16.04 LTS. Resonable resources, with at least one extra volume
for the NFS shares (may use LVM).
- Editor VM: Ubuntu 16.04 LTS with reasonable resources
- Cluster Manager: Ubuntu 16.04 LTS with reasonable resources
-  Monitoring: Ubuntu 16.04 LTS with reasonable resources
-  Tool Registry: Debian Jessie with reasonable resources and sizable disk space (e.g., a few hundred gigs)
-  Cluster Nodes: Debian Jessie with backports and as much CPU and RAM as possible.

Make sure you have your public key in `/root/.ssh/authorized_keys` of
each and every of the hosts above. Also, make sure you can log on with
ssh with out any obstacles or warnings.

Install with ansible
--------------------

We assume you've got a copy of `omtd-stack-setup` repository and the IPs of the
machines of the previous section.

Make a copy of `hosts`:
```bash
    $ cp hosts hosts.production
```
Edit "hosts.production" so that:
```yaml
    [nfs_server]
    <Executor IP>

    [docker_registry]
    <Tool Registry IP>

    [cluster_master]
    <Cluster Manager IP>

    [cluster_nodes]
    <Cluster Node 1 IP>
    <Cluster Node 2 IP> ...

    [executor]
    <Executor IP>

    [editor]
    <Editor IP>

    [monitor]
    <Monitor IP>
```
Edit `group_vars/all` to set credentials for interservice communication. Make sure to replace passwords and secrets with new ones that are long and hard to guess.

```yaml
    chronos_http_user: chronosadmin
    chronos_http_password: chronospassword

    chronos_principal: "chronos.omtd"
    chronos_secret: "chronos.secret"

    cluster_nodes:
        "83.212.107.159": {
            principal: "node1.omtd",
            secret: "node1.secret"

        }, "83.212.107.160": {
            principal: "node2.omtd",
            secret: "node2.secret"
        }
```

In `group_vars/executor`:
```yaml
    galaxy_admin: <email of the galaxy admin user>
```
In `group_vars/editor`:
```yaml
    remote_user_maildomain: <the domain of the user who is going to connect to the editor>
    remote_user_secret: <a long, hard to guess string>
```
Now, install requirements and run the ansible playbook to install the services. It might take a while:
```bash
    $ ansible-galaxy install -r requirements.yaml $ ansible-playbook -i hosts.production site.yaml
```

Executor
--------
First go to Reverse proxy and SSL <proxy_ssl> to setup HTTP (and HTTPS).

Now, make sure the following lines are included in `/etc/apache2/sites-enabled/vhosts-le-ssl.conf`:
```xml
    <VirtualHost *:443>
        ...
        <Directory>
        Require all granted
        </Directory>
        ...
    </VirtualHost>
```
Restart apache2.

Test if you can reach your host through HTTP(s). You should be able to
reach Galaxy.

Galaxy requires to create an admin user first. To do this, you must
change the Galaxy configuration to allow users to be created.

In /srv/executor/config/galaxy.ini find and commend out the following line:
```allow_user_creation = False```

Restart galaxy:
```$ service galaxy restart```

Connect to Galaxy through the web UI (just the fqdn of the host). On the top menu click `Login or Register` > `Register`. Fill out the form. The email field should be in the admin_users field in `/etc/executor/config/galaxy.ini`.

Uncommend `allow_user_creation = False` and restart galaxy. You are good to go.

LVM for executor
----------------
You can hide the database volume as an LVM. This will allow easy (but
manual) storage scaling (see the operations chapter).

1.  Install lvm:
```bash
    $ apt install lvm
```
2.  Find the unused physical volume(s) and prepare it(them) :
```bash
    $ fdisk -l
    ...
    Disk /dev/vdb: 380 GiB, 408021893120 bytes, 796917760 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    ...
    $ pvcreate /dev/vdb [/dev/vdc [...]]
```
3.  Create a new LVM group nfsstore, then create a logical partition and
    format it:
```bash
    $ vgcreate nfsstore /dev/vdb [/dev/vdc [...]]
    $ lvcreate -l 100%FREE -n nfs_logical nfsstore
    $ mke2fs -t ext4 /dev/nfsstore/nfs_logical
```
4.  Mount it:
```bash
    $ echo "/dev/nfsstore/nfs_logical /srv/executor/database ext4 defaults,nofail 0 0">>/etc/fstab
    $ mount -a
```
Make sure the target directory /srv/executor/database exists (in some
deployments it is /srv/galaxy/database).

Editor
------

First go to Reverse proxy and SSL <proxy_ssl> to setup HTTP (and HTTPS).

Then, try to connect to the host (just the IP). You should be redirected
to Galaxy, but get rejected with this message:

    Access to Galaxy is denied.

This is because the editor is configured to accept only remote users
from a specific domain (remote_user_maildomain), who authenticate
themselves with a secret (remote_user_secret). All users from this
domain can use the editor, as long as their requests contain the secret.

Mesos and Chronos
-----------------

Everything is set up, but it is good to secure communication with SSL.
You can do that with letsencrypt, if you follow the instructions in
SSL without proxy <just_ssl>.

Cluster nodes
-------------

Everything is set up, but it is good to secure communication with SSL.
You can do that with letsencrypt, if you follow the instructions in
SSL without proxy <just_ssl>.

Reverse proxy and SSL
---------------------
Our ansible scripts setup Apache2 as a reverse
proxy on the hosts that need a reverse proxy, but only as HTTP.

On the editor host, make sure '/etc/apache2/vhosts.conf' looks like this:
```xml
    DirectoryIndex index.html
   
    <VirtualHost *:80>
        ServerName 123.45.67.89

        RewriteEngine on RewriteRule ^(.*)
        [http://localhost:8080$1](http://localhost:8080$1) [P]
    </VirtualHost>
```

Make sure these apache modules are enabled: ssl, rewrite, proxy, proxy_http:
```$ a2query -m <module>```
To enable a disabled module:
```$ a2enmod <module>```

Restart apache2:
```$ service apache2 restart```

At this point, thinks should work well without SSL, but that is going to
change in the following lines.

First, remove the RewriteRule line from `/etc/apache2/sites-enabled/vhosts.conf`

In the following we install "let's encrypt" free certificates. If you
don't want to use these, you must figure some other way to setup your
HTTPS proxy.

In Debian:
```bash
    $ echo 'deb <http://ftp.debian.org/debian> jessie-backports main' |
    sudo tee /etc/apt/sources.list.d/backports.list
    $ sudo apt-get update
    $ sudo apt-get install python-certbot-apache -t jessie-backports
```

In Ubuntu:
```bash
    $ sudo apt-get install software-properties-common
    $ sudo add-apt-repository ppa:certbot/certbot
    $ sudo apt-get update
    $ sudo apt-get install certbot
```

Automatically set up certificates:
```bash
    $ sudo certbot --apache
    :   pick "vhosts.conf" and HTTPS only when asked
```
Restart apache2 and check that the host is redirecting to the correct place:
```$ sudo service apache2 restart```

SSL without Proxy
-----------------
In Debian:
```bash
    $ echo 'deb <http://ftp.debian.org/debian> jessie-backports main' |
    sudo tee /etc/apt/sources.list.d/backports.list
    $ sudo apt-get update
    $ sudo apt-get install certbot -t jessie-backports
```
In Ubuntu:
```bash
    $ sudo apt-get install software-properties-common
    $ sudo add-apt-repository ppa:certbot/certbot
    $ sudo apt-get update
    $ sudo apt-get install certbot
```
Now, install certbot and get the certificates. Make sure to replace example.com with your domain:
```bash
    $ sudo certbot certonly --standalone -d example.com
    $ sudo certbot renew
```
