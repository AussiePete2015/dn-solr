# Supported configuration parameters
The playbook in the [site.yml](../site.yml) file in this repository pulls in a set of default values for many of the configuration parameters that are needed to deploy Fusion from the [vars/solr.yml](../vars/solr.yml) file. The parameters defined in that file define a reasonable set of defaults for a fairly generic Fusion deployment, either to a single node or a cluster, including defaults for the URL that the Fusion distribution should be downloaded from, the directory that the Fusion nodes should store their data in, and the packages that must be installed on the node before the `fusion` service can be started.

In addition to the defaults defined in the [vars/solr.yml](../vars/solr.yml) file, there are a larger set of parameters that can be used to either control the deployment of Fusion to the nodes that will make up a cluster during an `ansible-playbook` run or to configure those Fusion nodes once the installation is complete. In this section, we summarize these options, breaking them out into:

* parameters used to control the `ansible-playbook` run
* parameters used during the deployment process itself, and
* parameters used to configure our Fusion nodes once Fusion has been installed locally.

Each of these sets of parameters are described in their own section, below.

## Parameters used to control the playbook run
The following parameters can be used to control the `ansible-playbook` run itself, defining things like how Ansible should connect to the nodes involved in the playbook run, which nodes should be targeted, where the Fusion distribution should be downloaded from, which packages must be installed during the deployment process, and where those packages should be obtained from:

* **`ansible_ssh_private_key_file`**: the location of the private key that should be used when connecting to the target nodes via SSH; this parameter is useful when there is one private key that is used to connect to all of the target nodes in a given playbook run
* **`ansible_user`**: the username that should be used when connecting to the target nodes via SSH; is useful if the same username is used when connecting to all of the target nodes in a given playbook run
* **`solr_url`**: the URL that the Fusion distribution should be downloaded from
* **`inventory_type`**: how the inventory is being managed (either 'static' or 'dynamic')
* **`cloud`**: if the inventory is being managed dynamically, this parameter is used to indicate the type of target cloud for the deployment (either `aws` or `osp`); this controls how the [build-app-host-groups](../common-roles/build-app-host-groups) common role retrieves the list of target nodes for the deployment
* **`local_solr_file`**: used to pass in the local path (on the Ansible host) to a Fusion distribution file; the named file will be uploaded to the target hosts and unpacked into the `solr_dir` directory.
* **`private_key_path`**: used to define the directory where the private keys are maintained when the inventory for the playbook run is being managed dynamically; in these cases, the scripts used to retrieve the dynamic inventory information will return the names of the keys that should be used to access each node, and the playbook will search the directory specified by this parameter to find the corresponding key files. If this value is not specified then the current working directory will be searched for those keys by default
* **`proxy_env`**: a hash map that is used to define the proxy settings to use for downloading distribution files and installing packages; supports the `http_proxy`, `no_proxy`, `proxy_username`, and `proxy_password` fields as part of this hash map
* **`reset_proxy_settings`**: used to reset any HTTP/YUM proxy settings that may have been made in a previous playbook run back to the defaults (no proxy); this is useful when a proxy was incorrectly set in a previous playbook run and the user wants to return to a "no-proxy" setup in the current playbook run
* **`yum_repo_url`**: used to set the URL for a local YUM mirror. This parameter is only used for CentOS-based deployments; when deploying Fusion to RHEL-based nodes this parameter is silently ignored and the RHEL package repositories defined locally on the node will be used for any packages installed during the deployment process

## Parameters used during the deployment process
These parameters are used to control the deployment process itself, defining things like where the distribution should be unpacked, the user/group that should be used when unpacking the distribution and starting the `fusion` service, and which packages to install.

* **`solr_dir`**: the path to the directory that the Fusion distribution should be unpacked into; defaults to `/opt/lucidworks` if unspecified
* **`solr_package_list`**: the list of packages that should be installed on the Fusion nodes; typically this parameter is left unchanged from the default (which installs the OpenJDK packages needed to run Fusion), but if it is modified the default, OpenJDK packages must be included as part of this list or an error will result when attempting to start the `fusion` service
* **`solr_user`**: the username under which Fusion should be installed and run; defaults to `lucidworks`
* **`solr_group`**: the name of the user group under which Fusion should be installed and run; defaults to `lucidworks`

## Parameters used to configure the Fusion nodes
These parameters are used configure the Fusion nodes themselves during a playbook run, defining things like the interfaces that Kakfa should be listening on for requests and the directory where Fusion should store its data.

* **`data_iface`**: the name of the interface that the Fusion nodes in the cluster should use when talking with each other and when talking to the Zookeeper ensemble. This interface typically corresponds to a private or management network, with no customer access. An interface of this name must exist for the playbook to run successfully, and if unspecified a value of `eth0` is assumed
* **`api_iface`**: the name of the interface that the Fusion nodes should listen on for user requests. This network corresponds to a public network since customers will use this interface to access the data in the Fusion cluster. An interface of this name must exist for the playbook to run successfully, and if unspecified a value of `eth0` is assumed
* **`iface_description_array`**: this parameter can be used in place of the `data_iface` and `api_iface` parameters described above, and it provides users with the ability to specify a description of these two interfaces rather than identifying them by name (more on this, below)
* **`solr_data_dir`**: the name of the directory that Fusion should use to store its data; defaults to `/opt/lucidworks/data` if unspecified. If necessary, this directory will be created as part of the playbook run
* **`default_gc_type`**: the Java garbage collection algorithm that should be used by the JVM used to run the various Java processes that are started by the `fusion` service; can be `cms`, `g1`, `throughput`/`parallel`, or `serial`/`none`. More information on these algorithms can be found [here](https://dzone.com/articles/garbage-collectors-serial-vs-0); this parameter defaults to a value of `g1` (the "Garbage First" algorithm), which is designed to better support applications with large Java heap sizes, if left unspecified
* **`solr_java_ops`**: the JVM options that should be set for the `solr` service; defaults to `-Xms16g -Xmx16g -Xss256k` if unspecified
* **`ui_java_ops`**: the JVM options that should be set for the `ui` service; defaults to `-Xms1g -Xmx1g` if unspecified
* **`connectors_java_ops`**: the JVM options that should be set for the `connectors` service; defaults to `-Xms3g -Xmx3g -Xss256k` if unspecified
* **`open_files`**: used to set the maximum number of file descriptors for the `fusion` service, defaults to `32768` (32K) if unspecified
* **`processes_number`**:  used to set the maximum number of proceses for the `fusion` service, defaults to `65536` (64K) if unspecified

## Interface names vs. interface descriptions
For some operating systems on some platforms, it can be difficult (if not impossible) to determine the names of the interfaces that should be passed into the playbook using the `data_iface` and `api_iface` parameters that we described, above. In those situations, the playbook in this repository provides an alternative; specifying those interfaces using the `iface_description_array` parameter instead.

Put quite simply, the `iface_description_array` lets you specify a description for each of the networks that you are interested in, then retrieve the names of those networks on each machine in a variable that can be used elsewhere in the playbook. To accomplish this, the `iface_description_array` is defined as an array of hashes (one per interface), each of which include the following fields:

* **`type`**: the type of description being provided, currently only the `cidr` type is supported
* **`val`**: a value describing the network in question; since only `cidr` descriptions are currently supported, a CIDR value that looks something like `192.168.34.0/24` should be used for this field
* **`as_var`**: the name of the variable that you would like the interface name returned as

With these values in hand, the playbook will search the available networks on each machine and return a list of the interface names for each network that was described in the `iface_description_array` as the value of the fact named in the `as_var` field for that network's entry. For example, given this description:

```json
    iface_description_array: [
        { as_var: 'data_iface', type: 'cidr', val: '192.168.34.0/24' },
        { as_var: 'api_iface', type: 'cidr', val: '192.168.44.0/24' },
    ]
```

the playbook will return the name of the network that matches the CIDR `192.168.34.0/24` as the value of the `data_iface` fact and the name of the network that matches the CIDR `192.168.34.0/24` as the value of the `api_iface` fact. These two facts can then be used later in the playbook to correctly configure the nodes to talk to each other and listen on the proper interfaces for user requests.

It should be noted that if you choose to take this approach when constructing your `ansible-playbook` runs, a matching entry in the `iface_description_array` must be specified for both the `data_iface` and `api_iface` networks, otherwise the default value of `eth0` will be used for these facts (and the playbook run may result in nodes that are at best misconfigured; if the `eth0` network does not exist then the playbook will fail to run altogether).
