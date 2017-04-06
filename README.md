Mesos Cluster Setup
===================

This ansible playbook setup a mesos cluster with high availability
based on this [guide](https://www.digitalocean.com/community/tutorials/how-to-configure-a-production-ready-mesosphere-cluster-on-ubuntu-14-04).
Also it setups a development environment for running a galaxy instance.

# Inventory Configuration

You have to configure your inventory to specify the hosts belonging
to every group. This playbook requires four groups of hosts:

- `nfs_server`: An nfs server.
- `nfs_clients`: List of hosts which share a directory with nfs sever.
- `mesos_masters`: The list of mesos_masters to manage the cluster.
  One of them is the leader master elected by Zookeeper and the rest
  are standby.
- `mesos_slaves`: The list of mesos_slaves nodes.
- `chronos`: Hosts that run chronos framework. Note that `chronos` is
   not supported on Ubuntu 16.04.
- `galaxy`: A host running Galaxy instance.

Perhaps, you may also want to specify additional group variables
based on your host, e.g. change the `ansible_python_interpreter`
variable to use python3.


# Setup
Install any required playbooks:

```console
sudo ansible-galaxy install -r requirements.yaml
```
Setup cluster and Galaxy:

```console
ansible-playbook -i hosts site.yaml
```

# Use Case: Run SADI workflow on a Mesos cluster

First, add the following variables to the `group_vars/galaxy`:

```
galaxy_repo: ssh://phab-vcs-user@phab.dev.grnet.gr:222/diffusion/GALAXY/galaxy.git
galaxy_version: feature-chronos
```

These variables specify the repository of galaxy from which ansible will clone and
the branch to checkout (e.g. `feature-chronos`). Ensure that you have
access to the phabricator from the corresponding host.


Then, export the following variable to forward ssh agent to your host:
```
export  ANSIBLE_SSH_ARGS="-o ForwardAgent=yes"
```

On the host which runs galaxy, create the following job configuration on
`<galaxy_repo>/config/job_conf.xml` (Do not forget to change the parameter
of `chronos-master` to point to the host which runs your chronos):

```xml
<?xml version="1.0"?>
<!-- A sample job config that explicitly configures job running the way it is configured by default (if there is no explicit config). -->
<job_conf>
    <plugins>
        <plugin id="chronos" type="runner" load="galaxy.jobs.runners.chrons:ChronosJobRunner" workers="4">
            <param id="chronos-master">{{ chronos-hostname }}:8080</param>
            <param id="volume">/root/galaxy</param>
            <param id="owner">galaxy@foo.test</param>
        </plugin>
        <plugin id="local" type="runner" load="galaxy.jobs.runners.local:LocalJobRunner" workers="4"/>
    </plugins>
    <handlers>
        <handler id="main"/>
    </handlers>
    <destinations default="local_dest">
        <destination id="chronos_dest" runner="chronos">
            <param id="docker_enabled">true</param>
            <param id="docker_memory">512</param>
            <param id="docker_cpu">2</param>
        </destination>
        <destination id="local_dest" runner="local">
            <param id="docker_enabled">true</param>
            <param id="docker_sudo">false</param>
        </destination>
    </destinations>
    <tools>
        <tool id="SADI-Docker-sadi_client" destination="chronos_dest"/>
        <tool id="SADI-Docker-RDFSyntaxConverter" destination="chronos_dest"/>
        <tool id="SADI-Docker-mergeRDFgraphs" destination="chronos_dest"/>
        <tool id="SADI-Docker-rapper" destination="chronos_dest"/>
        <tool id="SADI-Docker-tab2rdf" destination="chronos_dest"/>
    </tools>
</job_conf>
```

Then restart galaxy and run workflow as described
[here](https://github.com/mikel-egana-aranguren/SADI-Docker-Galaxy).


# Defaults variables per role.

Below, there is a list of default variables per role. You can override them by
setting your variables on your group_vars directory.

## Role: nfs

```yaml
# NFS export options.
export_options: rw,sync,no_subtree_check,no_root_squash
```

## Role: common

```yaml
# Mesosphere apt repo.
mesosphere_repo: "deb http://repos.mesosphere.com/{{ ansible_distribution | lower }} {{ ansible_distribution_release }} main"
```

## Role: mesos_masters

```yaml
# Size of Zookeeper Quorum
quorum_size: 1
```

## Role: chronos

```yaml
chronos_http_port: 8080
chronos_https_port: 9443
```

## Role: galaxy

```yaml
# Directory to host galaxy code.
galaxy_directory: /srv/galaxy

# Directory which holds object store data. This directory is mounted to an nfs
# server.
object_store_directory: /root/galaxy_dir

# Directory which is the galaxy job working directory. It's parent is
# `object_store_directory`.
job_working_directory: jobs_directory

# Directory which holds datasets of workflows. It's parent is
# `object_store_directory`.
files_directory: files

# tmp directory of jobs. It's parent is `object_store_directory`.
tmp_directory: tmp

# Galaxy remote repo.
galaxy_repo: https://github.com/galaxyproject/galaxy

# Version of the repository to check out. A branch, a tag or a commit hash.
galaxy_version: release_16.10

# True is SADI workflow should be installed.
install_sadi: True

# Directory to host SADI code.
sadi_directory: /root/SADI
```
