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
