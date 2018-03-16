Deployment guide
================
This document will guide you to deploy the Open MinTeD stack (omtd-stack) on a (virtual) cluster
of ubuntu and debian hosts. It has been deployed and tested on GRNET's "~okeanos" clusters, which
run "Synnefo".

Services
--------
If all services are split as much as possible, then:

Executor Galaxy - executes TDM (Text and Data Mining) tools and workflows, used through REST API
Editor Galaxy - offers an editor for creating and editing workflows
NFS Server - offers a shared folder for the tools between the two galaxies, as well as a shared folder between the Executor Galaxy and the cluster nodes
Chronos scheduler - schedulers the execution of tools
Mesos cluster manager - manages the cluster that executes the tools
Mesos slave
CAdvisor - monitors the execution of tools on the hosts they are executed
Prometheus-node-exporter - exports the results of the monitoring to the Prometheus aggregator
Prometheus - monitors the Mesos slaves and offers the results through a REST API
Grafana - visualazation of Prometheus monitoring
Docker Registry - hosts the images of the TDM tools

Suggested setup per host:

Executor: Executor Galaxy + NFS Server
Editor: Editor Galaxy
Cluster Manager: Chronos scheduler + Mesos master
Cluster Nodes: Mesos slave, CAdvisor, Node-Exporter
Monitoring: Prometheus + Grafana
Tool Registry: Docker Registry

Note: You can start with 2 or 3 nodes. It is easy to scale up (or down) later.

Provision
---------
Make sure to create the following VMs

Executor VM: Ubuntu 16.04 LTS. Resonable resources, with extra volume for the NFS shares, one of which should be as large as possible.
Editor VM: Ubuntu 16.04 LTS with reasonable resources
Cluster Manager: Ubuntu 16.04 LTS  with reasonable resources
Monitoring: Ubuntu 16.04 LTS with reasonable resources
Tool Registry: Debian Jessie with reasonable resources and sizeable disk space (e.g., a few hundred gigs)
Cluster Nodes: Debian Jessie with as much CPU and RAM as possible

Make sure you have your public key in "/root/.ssh/authorized_keys" of each and every of the hosts above. Also, make sure you can log on with ssh with out any obstacles or warnings.

Install with ansible
--------------------
We assume you've got a copy of "omtd-stack-setup" and the IPs of the machines of the previous section.

$ cp hosts hosts.production

edit "hosts.production" so that::
    [nfs_server]
    <Executor IP>

    [docker_registry]
    <Tool Registry IP>

    [nfs_clients]
    <Cluster Node 1 IP>
    <Cluster Node 2 IP>
    ...

    [mesos_masters]
    <Cluster Manager IP>

    [mesos_slaves]
    <Cluster Node 1 IP>
    <Cluster Node 2 IP>
    ...

    [chronos]
    <Cluster Manager IP>

    [executor]
    <Executor IP>

    [editor]
    <Editor IP>

In group_vars/executor::
	chronos_url: <Cluster Manager FQDN with Chronos port>

In groups_vars/editor::
    remote_user_maildomain: <the domain of the user who is going to connect to the editor>
    remote_user_secret: <a long, hard to guess string>

Now, install requirements and run the ansible playbook to install the services. It might take a while::
    $ ansible-galaxy install -r requirements.yaml
    $ ansible-playbook -i hosts.production site.yaml

Editor
------
Ansible takes care of all issues except the HTTP proxy. Setup the HTTP proxy and then go to :ref:`Reverse proxyes and SSL <proxy_ssl>` to setup HTTPS with letsencrypt. If you don't want to use letsencrypt, you can setup HTTPS in another way, which is not covered by the present guide.

On the editor host, make sure '/etc/apache2/vhosts.conf' looks like this::
    DirectoryIndex index.html

    <VirtualHost *:80>
      ServerName 123.45.67.89

      RewriteEngine on
      RewriteEngine on
      RewriteRule ^(.*) http://localhost:8080$1 [P]
    </VirtualHost>

Make sure these apache modules are enabled: ssl, rewrite, proxy, proxy_http::
    $ a2query -m <module>

To enable a disabled module::
    $ a2enmod <module>

Add some HTTPS support (e.g., :ref:`Reverse proxyes and SSL <proxy_ssl>`) and try to connect to the host. You should be redirected to Galaxy, but get a `Access to Galaxy is denied` message, because the editor is configured to accept only remote users from a specific domain (`remote_user_maildomain`), who authenticate themselves with a secret (`remote_user_secret`). All users from this domain can use the editor, as long as their requests contain the secret.

.. _proxy_ssl:
Reverse proxys and SSL
----------------------
Our ansible scripts setup Apache2 as a reverse proxy on the hosts that need a reverse proxy, but only as HTTP. 

At this point, we need SSL on every host machine we use. This may change in the future, at least for some hosts, but for now it is the most straigh-forward way.

In the following we install "let's encrypt" free certificates.

Ubuntu Hosts with apache2::
	$ sudo apt-get install software-properties-common
	$ sudo add-apt-repository ppa:certbot/certbot
	$ sudo apt-get update
	$ sudo apt-get install python-certbot-apache
	$ sudo certbot --apache
		pick "vhosts.conf" and HTTPS only when asked

Make sure the following lines are included in "/etc/apache2/sites-enabled/vhosts-le-ssl.conf"::
	<VirtualHost *:443>
		...
		<Directory>
	        Require all granted
    	</Directory>
    	...
    </VirtualHost>

Resart apache2 and check that the host is redirecting to the correct place::
	$ sudo service apache2 restart
