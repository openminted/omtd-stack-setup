Table of contents
=================
- Hosts:
  - [Executor](#executor)
  - [Editor](#editor)
  - [Cluster Manager](#cluster-manager)
  - [Docker registry](#docker-registry)
  - [Monitor](#monitor)
  - [Cluster nodes](#cluster-nodes)
- How to
  - [add a node](#how-to-add-a-node)
  - [remove a node](#how-to-remove-a-node)
  - [mount a volume](#how-to-mount-a-volume)
  - [add disk space](#how-to-add-disk-space)
  - [reduce disk space](#how-to-reduce-disk-space)

Executor
========
- [Galaxy for execution](#galaxy)
- [NFS Server sharing two folders](#nfs-server)
- [Apache2 as a reverse proxy](#apache2)

Galaxy
------
Typical Galaxy location: `/srv/executor`. In older scripts or deployments,
it may be `/srv/galaxy`.
Logs: `cat /var/log/syslog|grep galaxy`.

Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service galaxy <start|stop|restart|status>
```
or start manually:
```bash
    $ sudo /srv/executor/run.sh
```
If something doesn't work, stop the service and start it manually with
the `run.sh` script, which will check if some code has been modified or
if database migrations have to be applied. It may take a while to load
so you must wait until either the service is up or an error is produced.

To connect to UI, open a browser at `https://<host fqdn>`.

NFS Server
----------
Logs: `cat /var/log/syslog|grep nfs`.

Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service nfs-server <start|stop|restart|status>
```
The NFS Server shares to folders:
- `/srv/executor/database` with thevcluster nodes
- `/srv/executor/tools` with the editor

To check what directories you share with whom:
```bash
    $ sudo showmount -e localhost
```
The exports should be configured in `/etc/export` e.g., to share `/srv/executor/database`
with `12.345.67.89` :
```
    /srv/galaxy/database 83.212.12.345.67.89.96(rw,sync,no_subtree_check,no_root_squash)
```
If you make changes to this file, restart nfs-server afterwards.

Apache2
-------
Configuration: `/etc/apache2/sites-enabled/*`
Modules enabled: `rewrite`, `proxy`, `proxy_http`, `ssl`.
Logs: `/var/log/apache2/`

Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service apache2 <start|stop|restart|status>
```
To handle apache modules:
```bash
    $ sudo a2query -m <module>
    $ sudo a2enmod <module>
    $ sudo a2dismod <module>
```
always restart or reload apache2 afterwards.

On this host, Apache2 acts a reverse proxy redirecting everything to
localhost:8080

Editor
======
- [Galaxy for editing](#galaxy-1)
- [NFS client to mount one folder](#nfs-client)
- [Apache2 as a reverse proxy](#apache2-1)

Galaxy
------
Typical Galaxy location: `/srv/editor`. In older scripts or deployments,
it may be located at `/srv/galaxy`.

Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service galaxy <start|stop|restart|status>
```
or start manually:
```bash
    $ sudo /srv/executor/run.sh
```
If something doesn't work, stop the service and start it manually with
the run.sh script. this script checks if some code has been modified or
if database migrations have to be applied. If that's the case, it may
take a while to load and you need to wait until either the service is up
or an error is produced.

To connect to UI: `https://<host fqdn>/galaxy`
Note: you should be denied access to the system. If you need to get Ui access e.g. for
debugging, go to
/srv/executor/config/galaxy.ini' and turn "use\_remote\_user" to "False". Don't forget to turn it back to "True" when you are finished.

NFS client
----------
Logs: `cat /var/log/syslog|grep nfs`.
```
Make sure the `/srv/editor/tools` directory is mounted as an NFS share.
One way to do this:
```bash
    $ df -h
    Filesystem                     Size  Used Avail Use% Mounted on
    [...]
    <executor IP>:/srv/executor/tools   59G  7.5G   49G  14% /srv/editor/tools
```
If this doesn't work, make sure nfs-client is installed and `/etc/fstab` contains a line like this:
```bash
    <executor IP>:/srv/executor/tools /srv/editor/tools nfs defaults 0 0
```
Check that all elements (executor IP, NFS server directory, NFS client
directory) exist and are correct and mount again:
```bash
    $ mount /srv/editor/tools
```
Apache2
-------
Same as Apache2 for Executor.

Cluster Manager
===============
- [Zookeeper for service discovery](#zookeeper)
- [Chronos for scheduling](#chronos)
- [Mesos master for cluster management](#mesos-master)
- [Troubleshooting](#troubleshooting)

Zookeeper
---------
Logs: `cat /var/log/syslog|grep zookeeper`
Listening to port: `2181`

Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service zookeeper <start|stop|restart|status>
```
How does Mesos now where to find zookeeper:
```bash
    $ cat /etc/mesos/zk
    zk://localhost:2181/mesos
```
Chronos
-------
Logs: `cat /var/log/syslog|grep zookeeper`
Configuration: `/etc/chronos`
Listening to port: `8080`

Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service chronos <start|stop|restart|status>
```
In order to connect to Chronos, Galaxy Executor needs credentials. The
credentials are stored in `/etc/chronos/conf/http_credentials`.

In order for Chronos to connect to Mesos, it needs some other
credentials: a principal and a secret. The principal is stored in
`/etc/chronos/mesos_authentication_principal. The secret is stored in a
file indicated by `/etc/chronos/mesos_authentication_secret_file`
(usually, `/etc/chronos/store/mesos_authentication_secret`). The
principal and secret must match *exactly one* of the credentials in
`/etc/mesos-master/store/credentials.json`.

Make sure all mesos files have NO new line at the end.

If you change any credentials, restart Chronos.

To check if Chronos is running, open
`http://<cluster manager IP or fqdn>:8080` on a browser. You can use
the `http_credentials` to log. In order for the cluster to function
properly, it is not enough for Chronos to be up, it should also be
connected to Mesos master (see next paragraph).

Mesos master
------------
Logs: `/var/log/mesos/`
Configuration: `/etc/mesos-master`
Listening to port: `5050`

Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service mesos-master <start|stop|restart|status>
```
Credentials are stored in a file indicated by
`/etc/mesos-master/credentials` which is typically
`/etc/mesos-master/store/credentials.json`. This file holds credentials
of "principal"/"secret" form in json format. There are stored the
credentials for Chronos as well as for each node.

To check if the mesos master is up, visit
`http://<cluster manager IP or fqdn>:5050` on a browser. You can use
any of the credentials in the credentials file to log. You must be able
to see Chronos connected as a framework (click "Frameworks" on top
menu), and each of the nodes connected as an agent (click "Agents" on
top menu. If you don't see them there, something is wrong.

Troubleshooting
---------------
If you must troubleshoot the cluster manager, you need luck, patience
and a salary raise. If you succeed, you *must* demand a cookie as a
reward.

Most errors are caused by electing the wrong leader. This can happen if
mesos masters installed on the cluster nodes are enabled by mistake and
compete to become leaders. To make sure your system doesn't suffer from
this, log on each cluster node and check if the mesos master is running
(service mesos-master status). If that's the case, stop it
(service mesos-master stop). Make sure only the cluster manager
mesos-master is running. Give it a minute and your issues may get
resolved.

In general, before you go deeper in troubleshooting, try shutting down
the Chronos, mesos master and mesos slaves on cluster manager and every
cluster node. Make sure there are not any mesos masters running on
slaves or mesos slaves running on cluster manager. Then restart them in
this order (order maters) :
1. Mesos master on cluster manager
2. Chronos on cluster manager
3. Mesos slaves on every node

Also, check the credentials on Mesos master. Do they much with the
credentials on the nodes and Chronos?

Another tip: make sure all one-line configuration files in
`/etc/mesos-master` and `/etc/chronos` have no new line at the end. I
kid you not!

Have fun debugging Chronos/Mesos.

Docker registry
===============
- [Docker engine](#docker-engine)
- [Docker registry](#docker-registry)
- [Apache2 as a reverse proxy and password security](#apache2-2)
- [Test if the service works](#test-if-the-service-works)

Docker engine
-------------
Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service docker <start|stop|restart|status>
```
Docker registry
---------------
Runs as a docker container.
Logs: ```$ docker logs registry```

Start, stop, restart or check the status of the service:
```bash
    $ docker <start|stop|restart|inspect> registry
```
or find "registry" when running:
```bash
    $ docker ps
```
Apache2
-------
Configuration: `/etc/apache2/sites-enabled/`
Modules enabled:
- authn_file
- authn_core 
- authz_groupfile
- authz_user
- authz_core
- auth_basic
- access_compat
- headers
- ssl
- proxy
- proxy_http
Logs: `cat /var/log/apache2/`

Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service apache2 <start|stop|restart|status>
```
To handle apache modules:
```bash
    $ sudo a2query -m <module>
    $ sudo a2enmod <module>
    $ sudo a2dismod <module>
```
always restart or reload apache2 afterwards.

In this host, Apache2 acts a reverse proxy redirecting `/v2` to
`localhost:5000/v2`. Also, it guards the `/v2` location with a password.

Test if the service works
-------------------------
From an external machine (e.g., your workstation):
```bash
    $ docker pull hello-world
    $ docker login <docker_registry fqdn>
    username: <registry_username from group_vars/all>
    password: <registry_password from group_vars/all>
    $ docker tag hello-world <docker_registry fqdn>:hello-world
    $ docker push <docker_registry fqdn>:hello-world
    $ docker pull <docker_registry fqdn>:hello-world
```
Typically, the username and password of the registry are defined in
`groups_vars/all` (ansible) as "registry_username" and
"registry_password".

If all these commands complete without an error, the registry works.

Monitor
=======
- [Prometheus for log aggregation and REST API](#prometheus)
- [Grafana for visualization](#grafana)
- [Apache2 as reverse proxy](#apache2-3)

Prometheus
----------
Configuration: `/etc/prometeus/prometheus.yml`
Logs: `/var/log/prometheus/prometheus.log`

Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service prometheus <start|stop|restart|status>
```

Grafana
-------
Configuration: `/etc/grafana/grafana.ini`
Logs: `/var/log/grafana/grafana.log`

Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service grafana-server <start|stop|restart|status>
```

Apache2
-------
Similar Executor setup, with the following details:
Grafana: / --> http://127.0.0.1:3000/
Prometheus: /prometheus/ --> http://127.0.0.1:9090/prometheus/

Cluster nodes
=============
- [Docker engine pulling images from our "docker registry" service](#docker-engine-1)
- [Prometheus node exporter](#prometheus-node-exporter)
- [CAdvisor as a docker container](#cadvisor)
- [NFS client](#nfs-client-2)

Docker engine
-------------
Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service docker <start|stop|restart|status>
```
Test if the engine can pull from the docker registry by trying out the
test in Test if the service works section.

Prometheus node exporter
------------------------
Logs: `cat /var/log/syslog|grep prometheus`.

Start, stop, restart or check the status of the service with systemd:
```bash
    $ sudo service prometheus-node-exporter <start|stop|restart|status>
```

CAdvisor
--------
Logs: `$ docker logs cadvisor`

Start, stop, restart or check the status of the service:
```bash
    $ docker <start|stop|restart|inspect> cadvisor
```

NFS client
----------
Logs: `cat /var/log/syslog|grep nfs`.
```
Make sure the `/srv/executor/datavase` directory is mounted as an NFS share.
One way to do this:
```bash
    $ df -h
    Filesystem                     Size  Used Avail Use% Mounted on
    [...]
    <executor IP>:/srv/executor/database   59G  7.5G   49G  14% /srv/executor/database
```
If this doesn't work, make sure nfs-client is installed and `/etc/fstab` contains a line like this:
```bash
    <executor IP>:/srv/executor/database /srv/editor/database nfs defaults 0 0
```
Check that all elements (executor IP, NFS server directory, NFS client
directory) exist and are correct and mount again:
```bash
    $ mount /srv/executor/database

How to add a node
=================

1.  Provision a new virtual Machine with Debian Jessie (tested) or
    similar (untested).
2.  Make sure you have SSH access to this VM from the host
    running ansible.
3.  Add the VM IP in your hosts (or hosts.production, or whatever
    you use) file inside the ansible script, under the
    `cluster_nodes` section.
4.  Add this in `cluster_nodes` dict which you can find in
    `group\_vars/all`:
    ```json
        "123.45.67.89": {
            principal: "nodeX.omtd",
            secret: "nodeX.secret"
        }
    ```
(replace 123.45.67.89 with the actual VM IP, change the
principal to something unique - if it is the 6th node,use `node6.omtd`,
change the secret to something harder to guess).

5.  Install with ansible:

        $ ansible-playbook -i hosts -l cluster_master,cluster_nodes

How to remove a node
====================

1.  In hosts (or hosts.production or whatever) remove the IP of the node
    from the `cluster_nodes` list
2.  In `group_vars/all` remove the related section (IP, prinsipal
    and secret) from the `cluster_nodes` dict variable
3.  Reset the cluster node with ansible:
    ```bash
        $ ansible-playbook -i hosts -l cluster_master
    ```

The node is not connected to the cluster anymore, you can destroy it or
use it otherwise. If you don't destroy the node, make sure
prometheus-node-exporter does not send scraps to the prometheus server
(e.g., stop the exporter service).

How to mount a volume
=====================
General purpose instructions, after you create a volume on your cloud and attach it on a VM::
```bash
    $ fdisk -l
    $ fdisk /dev/vdb
    <type:> n
    <type:> p
    <type:> w
    `$ mke2fs -t ext4 /dev/vdb1 # note down Filesystem UUID: f090a4e2-ab78-49a6-9be3-8c07161c4f0b
    $ mkdir -p /volumes/nfs1
    $ echo "UUID=f090a4e2-ab78-49a6-9be3-8c07161c4f0b /volumes/nfs1 ext4 errors=remount-ro 0 1" >> /etc/fstab
```

How to add disk space
=====================
The main storage space is an LVM (Logical VoluMe) disk mounted on the
executor VM, which is shared with NFS to every cluster node.
Practically, an LVM is a group of physical volumes which behaves as a
single disk.

Lets see what's inside our nfsstore LVM:
```bash
    $ vgs
      VG       #PV #LV #SN Attr   VSize   VFree
      nfsstore   2   1   0 wz--n- 699.99g    0 
    $ pvs -o+pv_used
      PV         VG       Fmt  Attr PSize   PFree Used
      /dev/vdb   nfsstore lvm2 a--  380.00g    0  380.00g
      /dev/vdc   nfsstore lvm2 a--  320.00g    0  320.00g
```

1.  Create a new volume and attach it on the executor VM, using your
    cloud tools e.g., in ~okeanos:
    ```bash
        $ kamaki volume create --size 200 --name omtd_3 --server-id <executor VM id> --project-id <openinted project id>
    ```
2.  Log in executor and find the new, unformated volume:
    ```bash
        $ fdisk -l
        ...
        Disk /dev/vdd: 200 GiB, 203097383680 bytes, 521088640 sectors
        Units: sectors of 1 * 512 = 512 bytes
        Sector size (logical/physical): 512 bytes / 512 bytes
        I/O size (minimum/optimal): 512 bytes / 512 bytes
        ...
    ```
3.  Prepare the volume for LVM:
    ```bash
        $ pvcreate /dev/vdd
    ```
4.  Extend the volume group and the logical partition:
    ```bash
        $ vgextend nfsstore /dev/vdd
        $ lvextend /dev/nfsstore/nfs_logical /dev/vdd
        $ resize2fs /dev/nfsstore/nfs_logical
    ```

Done!

How to reduce disk space
========================
Make sure you have a backup of everything in `/srv/executor/database` and
then proceed.

List all the physical volumes in LVM:
```bash
    $ pvs -o+pv_used
      PV         VG       Fmt  Attr PSize   PFree Used
      /dev/vdb   nfsstore lvm2 a--  380.00g    0  380.00g
      /dev/vdc   nfsstore lvm2 a--  320.00g    0  320.00g
      /dev/vdd   nfsstore lvm2 a--  200.00g    0  200.00g
```
For demonstration purposes, we assume you want to remove the 200G
volume, thus reducing the VLM storage by 200G. this volume is /dev/vdd.

So, lets remove `/dev/vdd`:
```bash
    $ pvmove /dev/vdd
    $ vgreduce nfsstore /dev/vdd
```
And we are done! You can now use `/dev/vdd` for some other purpose, or
destroy it, if you don't need it anymore.

Explanation: `pvmove` moves all data stored in `/dev/vdd` to the other
physical volumes, while vgreduce removes the volume from LVM.
