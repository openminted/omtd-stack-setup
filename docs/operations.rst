Executor
========
Services run:
- Galaxy for execution
- NFS Server sharing two folders
- Apache2 as a reverse proxy

Galaxy
------
Typical Galaxy location: `/srv/executor`.
In older scripts or deployments, it may be located at `/srv/galaxy`.
Logs: `cat /var/log/syslog|grep galaxy`.

Start, stop, restart or check the status of the service with systemd::

    $ sudo service galaxy <start|stop|restart|status>

or start manually::

    $ sudo /srv/executor/run.sh

If something doesn't work, stop the service and start it manually with the `run.sh` script. this script checks if some code has been modified or if database migrations have to be applied. If that's the case, it may take a while to load and you need to wait until either the service is up or an error is produced.

To connect to UI: https://<host fqdn>

NFS Server
----------
Logs: `cat /var/log/syslog|grep nfs`.

Start, stop, restart or check the status of the service with systemd::

    $ sudo service nfs-server <start|stop|restart|status>

The NFS Server shares to folders:
- `/srv/executor/database` with the cluster nodes
- `/srv/executor/tools` with the editor

To check what directories you share with whom::

    $ sudo showmount -e localhost

The exports should be configured in `/etc/export`, e.g. to share `database` with `12.345.67.89` ::

    /srv/galaxy/database 83.212.12.345.67.89.96(rw,sync,no_subtree_check,no_root_squash)

If you make changes to this file, restart nfs-server afterwards.

Apache2
-------
Configuration: `/etc/apache2/sites-enabled/*`
Modules enabled: `rewrite`, `proxy`, `proxy_http`, `ssl`.
Logs: `cat /var/log/apache2/*.log`

Start, stop, restart or check the status of the service with systemd::

    $ sudo service apache2 <start|stop|restart|status>

To handle apache modules::

    $ sudo a2query -m <module>
    $ sudo a2enmod <module>
    $ sudo a2dismod <module>

always restart or reload apache2 afterwards.

In this host, Apache2 acts a reverse proxy redirecting everything to localhost:8080

Editor
======
Services run:
- Galaxy for editing
- NFS client to mount one folder
- Apache2 as a reverse proxy

Galaxy
------
Typical Galaxy location: `/srv/editor`.
In older scripts or deployments, it may be located at `/srv/galaxy`.

Start, stop, restart or check the status of the service with systemd::

    $ sudo service galaxy <start|stop|restart|status>

or start manually::

    $ sudo /srv/executor/run.sh

If something doesn't work, stop the service and start it manually with the `run.sh` script. this script checks if some code has been modified or if database migrations have to be applied. If that's the case, it may take a while to load and you need to wait until either the service is up or an error is produced.

To connect to UI: https://<host fqdn>/galaxy
Note: you should be denied access to the system. If you need to get Ui access e.g. for debugging, go to `/srv/executor/config/galaxy.ini' and turn "use_remote_user" to "False". Don't forget to turn it back to "True" when you are finished.

NFS client
----------
Logs: `cat /var/log/syslog|grep nfs`.

Start, stop, restart or check the status of the service with systemd::

    $ sudo service nfs-client <start|stop|restart|status>

Make sure the `/srv/editor/tools` directory is mounted as an NFS share. One way to do this::

    $ df -h
    Filesystem                     Size  Used Avail Use% Mounted on
    [...]
    <executor IP>:/srv/executor/tools   59G  7.5G   49G  14% /srv/editor/tools

If this doesn't work, make sure `/etc/fstab` contains a line like this::

    <executor IP>:/srv/executor/tools /srv/editor/tools nfs defaults 0 0

Check that all elements (executor IP, NFS server directory, NFS client directory) exist and are correct and mount again::

    mount /srv/editor/tools

Apache2
-------
Same as Apache2 for Executor.


Cluster Manager
===============
Services run:
- Zookeeper for service discovery
- Chronos for scheduling
- Mesos master for cluster management

Zookeeper
---------
Logs: `cat /var/log/syslog|grep zookeeper`
Listening to port: 2181

Start, stop, restart or check the status of the service with systemd::

    $ sudo service zookeeper <start|stop|restart|status>

How does Mesos now where to find zookeeper::

    $ cat /etc/mesos/zk
    zk://localhost:2181/mesos

Chronos
-------
Logs: `cat /var/log/syslog|grep zookeeper`
Configuration: `/etc/chronos`
Listening to port: 8080

Start, stop, restart or check the status of the service with systemd::

    $ sudo service chronos <start|stop|restart|status>

In order to connect to Chronos, Galaxy Executor needs credentials. The credentials are stored in `/etc/chronos/conf/http_credentials`.

In order for Chronos to connect to Mesos, it needs some other credentials: a principal and a secret. The principal is stored in `/etc/chronos/mesos_authentiation_principal`. The secret is stored in a file indicated by `/etc/chronos/mesos_authentication_secret_file` (usually, `/etc/chronos/store/mesos_authentication_secret`). The principal and secret must match *exactly one* of the credentials in `/etc/mesos-master/store/credentials.json`.

Make sure all mesos files have NO new line at the end.

If you change any credentials, restart Chronos.

To check if Chronos is running, open `http://<cluster manager IP or fqdn>:8080` on a browser. You can use the "http_credentials" to log. In order for the cluster to function properly, it is not enough for Chronos to be up, it should also be connected to Mesos master (see next paragraph).

Mesos master
------------
Logs: `/var/log/mesos/*`
Configuration: `/etc/mesos-master`
Listening to port: 5050

Start, stop, restart or check the status of the service with systemd::

    $ sudo service mesos-master <start|stop|restart|status>

Credentials are stored in a file indicated by `/etc/mesos-master/credentials` which is typically '/etc/mesos-master/store/credentials.json'. This file holds credentials of "principal"/"secret" form in json format. There are stored the credentials for Chronos as well as for each node.

To check if the mesos master is up, visit `http://<cluster manager IP or fqdn>:5050` on a browser. You can use any of the credentials in the credentials file to log. You must be able to see Chronos connected as a framework (click "Frameworks" on top menu), and each of the nodes connected as an agent (click "Agents" on top menu. If you don't see them there, something is wrong.

Troubleshooting
---------------
If you must troubleshoot the cluster manager, you need luck, patience and a salary raise. If you succeed, you *must* demand a cookie as a reward.

Most errors are caused by electing the wrong leader. This can happen if mesos masters installed on the cluster nodes are enabled by mistake and compete to become leaders. To make sure your system doesn't suffer from this, log on each cluster node and check if the mesos master is running (`service mesos-master status`). If that's the case, stop it (`service mesos-master stop`). Make sure only the cluster manager mesos-master is running. Give it a minute and your issues may get resolved.

In general, before you go deeper in troubleshooting, try shutting down the Chronos, mesos master and mesos slaves on cluster manager and every cluster node. Make sure there are not any mesos masters running on slaves or mesos slaves running on cluster manager.
Then restart them in this order (order maters) :
- 1: Mesos master on cluster manager
- 2: Chronos on cluster manager
- 3: mesos slaves on every node

Also, check the credentials on Mesos master. Do they much with the credentials on the nodes and Chronos?

Another tip: make sure all one-line configuration files in '/etc/mesos-master' and '/etc/chronos' have no new line at the end. I kid you not!

Have fun debugging Chronos/Mesos.

Docker registry
===============
Services run:
- Docker engine
- Docker registry
- Apache2 as a reverse proxy and passwor security

Docker engine
-------------
Start, stop, restart or check the status of the service with systemd::

    $ sudo service docker <start|stop|restart|status>

Docker registry
---------------
Runs as a docker container.
Logs: `docker logs registry`

Start, stop, restart or check the status of the service with systemd::

    $ docker <start|stop|restart|inspect> registry

or find "registry" when running::

    $ docker ps

Apache2
-------
Configuration: `/etc/apache2/sites-enabled/*`
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
Logs: `cat /var/log/apache2/*.log`

Start, stop, restart or check the status of the service with systemd::

    $ sudo service apache2 <start|stop|restart|status>

To handle apache modules::

    $ sudo a2query -m <module>
    $ sudo a2enmod <module>
    $ sudo a2dismod <module>

always restart or reload apache2 afterwards.

In this host, Apache2 acts a reverse proxy redirecting `/v2` to `localhost:5000/v2`. Also, it guards the `/v2` location with a password.

Monitor
=======
Services run:
- Prometheus for log aggregation and REST API
- Grafana for visualization
- Apache2 as reverse proxy

Prometheus
----------
Configuration: `/etc/prometeus/prometheus.yml`
Logs: `/var/log/prometheus/prometheus.log`

Start, stop, restart or check the status of the service with systemd::

    $ sudo service prometheus <start|stop|restart|status>

Grafana
-------
Configuration: `/etc/grafana/grafana.ini`
Logs: `/var/log/grafana/grafana.log`

Start, stop, restart or check the status of the service with systemd::

    $ sudo service grafana-server <start|stop|restart|status>

Apache2
-------
Similar Executor setup, with the following details:
Grafana: / --> http://127.0.0.1:3000/
Prometheus: /prometheus/ --> http://127.0.0.1:9090/prometheus/

Cluster nodes
=============
Services run:
- Docker engine logged to "docker registry"
- Prometheus node exporter
- CAdvisor as a docker container

Docker engine
-------------

Prometheus node exporter
------------------------

CAdvisor
--------

How to add a node
=================

How to remove a node
====================

How to add disk space
=====================

How to reduce disk space
========================
