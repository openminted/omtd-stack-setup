Mesos Cluster Setup
===================

This ansible playbook setup a mesos cluster with high availability
based on this [guide](https://www.digitalocean.com/community/tutorials/how-to-configure-a-production-ready-mesosphere-cluster-on-ubuntu-14-04).

# Inventory Configuration

You have to configure your inventory to specify the hosts belonging
to every group. This playbook requires two groups of hosts:

- `mesos_masters`: The list of mesos_masters to manage the cluster.
  One of them is the leader master elected by Zookeeper and the rest
  are standby.
- `mesos_slaves`: The list of mesos_slaves nodes.

Perhaps, you may also want to specify additional group variables
based on your host, e.g. change the `ansible_python_interpreter`
variable to use python3.


# Setup
Run ansible-playbook -i hosts site.yaml
