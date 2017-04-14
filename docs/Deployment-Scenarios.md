# Example deployment scenarios

There are a three basic deployment scenarios that are supported by this playbook. In the first two scenarios (shown below) we'll walk through the deployment of Fusion to a single node and the deployment of a multi-node Fusion cluster using a static inventory file. Finally, in the third scenario, we will show how the same multi-node Fusion cluster deployment shown in the second scenario could be performed using the dynamic inventory scripts for both AWS and OpenStack instead of a static inventory file.

## Scenario #1: deploying Fusion to a single node
While this is the simplest of the deployment scenarios that are supported by this playbook, it is more than likely that deployment of Fusion to a single node is really only only useful for very small workloads or deployments of simple test environments. Nevertheless, we will start our discussion with this deployment scenario since it is the simplest.

If we want to deploy Fusion to a single node with the IP address "192.168.34.22", we would start by creating a very simple inventory file that looks something like the following:

```bash
$ cat single-node-inventory

192.168.34.22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_node_private_key'

$ 
```

Note that in this inventory file the `ansible_ssh_host` and `ansible_ssh_port` parameters will take their default values since they aren't specified for the single host entry in the inventory file. Once we've built our static inventory file, we would simply run an `ansible-playbook` command that looks something like this to perform a single-node Fusion deployment to that node:

```bash
$ ansible-playbook -i single-node-inventory -e "{ host_inventory: ['192.168.34.22'] }" site.yml
```

This command will download the Lucidworks Fusion distribution from the default URL defined in the [vars/solr.yml](../vars/solr.yml) file (the main Lucidworks download site), unpack that distribution file into the `/opt/lucidworks` directory on that host (the default value for the location to unpack the distribution into), configure the `fusion` instance on that host, enable the `fusion` service to start on boot, and (re)start the `fusion` service. Note that the `fusion` service is actually a wrapper for a number of services that make up the Lucidworks Fusion distribution, and for a single-node deployment that set of services will include a bundled `zookeeper` instance that will be started as part of the process of starting up the various `fusion` services.

## Scenario #2: deploying a multi-node Fusion cluster
If you are using this playbook to deploy a multi-node Fusion cluster, then the configuration becomes a bit more complicated. The playbook assumes that if you are deploying a Fusion cluster you will also want to have a multi-node Zookeeper ensemble associated with that cluster. Furthermore, to ensure that the resulting pair of clusters are relatively tolerant of the failure of a node, we highly recommend that the deployments of these two clusters (the Fusion cluster and the Zookeeper ensemble) be made to separate sets of nodes. For the playbook in this repository, this separation of the Zookeeper ensemble from the Fusion cluster is made by assuming that in deploying a Fusion cluster we will be configuring the nodes of that cluster to work with a multi-node Zookeeper ensemble that has **already been deployed and configured separately** (and we provide a separate role and playbook, both of which are defined in the DataNexus [dn-zookeeper](https://github.com/DataNexus/dn-zookeeper) repository, that can be used to manage the process of deploying and configuring the necessary Zookeeper ensemble).

So, assuming that we've already deployed a three-node Zookeeper ensemble separately and that we want to deploy a three node Fusion cluster, let's walk through what the commands that we'll need to run look like. In addition, let's assume that we're going to be using a static inventory file to control our Fusion deployment. The static inventory file that we will be using for this example looks like this:

```bash
$ cat test-cluster-inventory
# example inventory file for a clustered deployment

192.168.34.28 ansible_ssh_host= 192.168.34.28 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/solr_cluster_private_key'
192.168.34.29 ansible_ssh_host= 192.168.34.29 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/solr_cluster_private_key'
192.168.34.30 ansible_ssh_host= 192.168.34.30 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/solr_cluster_private_key'

$
```

To correctly configure our Fusion cluster to talk to the Zookeeper ensemble, the playbook will need to connect to the nodes that make up the associated Zookeeper ensemble and collect information from them, and to do so we'll have to pass in the information that Ansible will need to make those connections to the playbook. We do this by passing in a separate inventory file (the `zookeeper_inventory_file` for the deployment) that contains the inventory information for the members of the Zookeeper ensemble we will be associating with this Fusion cluster. For the purposes of this example, let's assume that our `zookeeper_inventory_file` looks something like this:

```bash
$ cat zookeeper-inventory
# example inventory file for a clustered deployment

192.168.34.18 ansible_ssh_host= 192.168.34.18 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.19 ansible_ssh_host= 192.168.34.19 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.20 ansible_ssh_host= 192.168.34.20 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'

$
```

To deploy Fusion to the three nodes in our static inventory file, we'd run a command that looks something like this:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      host_inventory: ['192.168.34.28', '192.168.34.29', '192.168.34.30'], \
      cloud: vagrant, data_iface: eth0, api_iface: eth1, \
      zookeeper_inventory_file: './zookeeper-inventory', \
      solr_url: 'http://192.168.34.254/lucidworks-fusion/fusion-3.0.0.tar.gz', \
      yum_repo_url: 'http://192.168.34.254/centos', solr_data_dir: '/data' \
    }" site.yml
```

Alternatively, rather than passing all of those arguments in on the command-line as extra variables, we can make use of the *local variables file* support that is built into this playbook and construct a YAML file that looks something like this containing the configuration parameters that are being used for this deployment:

```yaml
cloud: vagrant
data_iface: eth0
api_iface: eth1
zookeeper_inventory_file: './zookeeper-inventory'
solr_url: 'http://192.168.34.254/lucidworks-fusion/fusion-3.0.0.tar.gz'
yum_repo_url: 'http://192.168.34.254/centos'
solr_data_dir: '/data'
```

and then we can pass in the *local variables file* as an argument to the `ansible-playbook` command; assuming the YAML file shown above was in the current working directory and was named `test-cluster-deployment-params.yml`, the resulting command would look somethin like this:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      host_inventory: ['192.168.34.28', '192.168.34.29', '192.168.34.30'], \
      local_vars_file: 'test-cluster-deployment-params.yml' \
    }" site.yml
```

Once the playbook run is complete, we can simply browse to the Fusion UI on one of the nodes in our cluster (eg. http://192.168.34.28:8764/) to test and make sure that our Fusion cluster is up and running. Do keep in mind that it does take some time for the Fusion UI service to start after the playbook run completes, so be patient).

## Scenario #3: deploying a Fusion cluster via dynamic inventory
In this section we will repeat the multi-node cluster deployment that we just showed in the previous scenario, but we will use the dynamic inventory scripts provided in the [common-utils](../common-utils) submodule to control the deployment of our Fusion cluster to an AWS or OpenStack environment rather than relying on a static inventory file.

To accomplish this, the we have to:

* Tag the instances in the AWS or OpenStack environment that we will be configuring as a Fusion cluster with the appropriate `Tenant`, `Project`, `Domain`, and `Application` tags. Note that for all of the nodes we are targeting in our playbook runs, we will assign an `Application` tag of `solr`
* Have already deployed a Zookeeper ensemble into this environment; this ensemble should be tagged with the same  `Tenant`, `Project`, `Domain` tags that were used when tagging the instances that will make up our Fusion cluster (above); obviously the  `Application` for the Zookeeper ensemble nodes would be `zookeeper`
* Once all of the nodes that will make up our cluster had been tagged appropriately, we can run `ansible-playbook` command similar to the `ansible-playbook` command shown in the previous scenario; this will deploy Fusion to the cluster nodes and configure them to talk to the each other through the associated Zookeeper ensemble

In terms of what the commands look like, lets assume for this example that we've tagged our seed nodes with the following VM tags:

* **Tenant**: labs
* **Project**: projectx
* **Domain**: preprod
* **Application**: solr

The `ansible-playbook` command used to deploy Fusion to our nodes and configure them as a cluster in an OpenStack environment would then look something like this:

```bash
$ ansible-playbook -i common-utils/inventory/osp/openstack -e "{ \
        host_inventory: 'meta-Application_solr:&meta-Cloud_osp:&meta-Tenant_labs:&meta-Project_projectx:&meta-Domain_preprod', \
        application: solr, cloud: osp, tenant: labs, project: projectx, domain: preprod, \
        ansible_user: cloud-user, private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        solr_data_dir: '/data' \
    }" site.yml
```

The playbook uses the tags in this playbook run to identify the nodes that make up the associated Zookeeper ensemble (which must be up and running for this playbook command to work) and the target nodes for the playbook run, installs the Lucidworks Fusion distribution on the target nodes, and configures them as a new Fusion cluster. 

In an AWS environment, the command would look something like this:

```bash
$ ansible-playbook -i common-utils/inventory/aws/ec2 -e "{ \
        host_inventory: 'tag_Application_solr:&tag_Cloud_aws:&tag_Tenant_labs:&tag_Project_projectx:&tag_Domain_preprod', \
        application: solr, cloud: aws, tenant: labs, project: projectx, domain: preprod, \
        ansible_user: cloud-user, private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        solr_data_dir: '/data' \
    }" site.yml
```

As you can see, this command is basically the same command that was shown for the OpenStack use case; it only differs slightly in terms of the name of the inventory script passed in using the `-i, --inventory-file` command-line argument, the value passed in for `Cloud` tag (and the value for the associated `cloud` variable), and the prefix used when specifying the tags that should be matched in the `host_inventory` value (`tag_` instead of `meta-`). In both cases the result would be a set of nodes deployed as a Fusion cluster, with the number of nodes in the cluster determined (completely) by the tags that were assigned to the VMs in the cloud environment in question.
