# dn-solr
Playbooks/Roles used to deploy LucidWorks Fusion (Solr)

# Installation
To install solr using the `site.yml` playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:

```bash
$ git clone --recursive https://github.com/Datanexus/dn-solr
```

That command will pull down the repository and it's submodules (currently the only dependency embedded as a submodule is the dependency on the `https://github.com/Datanexus/common-roles` repository).

# Using this role to deploy Solr
The `site.yml` file at the top-level of this repository pulls in a set of default values for the parameters that are needed to deploy an instance of the LucidWorks Fusion (Solr) distribution to a node from the `vars/solr.yml` file.  The contents of that file currently look like this:

```yaml
# (c) 2016 DataNexus Inc.  All Rights Reserved
#
# Defaults that are necessary for all deployments of
# LucidWorks Fusion (Solr)
---
application: solr
# the URL that should be used to download the Fusion (Solr) distribution
# (Fusion is distributed as a gzipped tarfile)
solr_url: "https://download.lucidworks.com/fusion-3.0.0.tar.gz"
# the directory where the solr data will be written
solr_data_dir: /opt/lucidworks/data
# the interface Fusion (Solr) should listen on when running
solr_iface: eth0
# the directory that the distribution should be unpacked into
solr_dir: "/opt/lucidworks"
# the packages that need to be installed for Solr nodes (the JRE and JDK
# packages from the OpenJDK project)
solr_package_list: ["java-1.8.0-openjdk", "java-1.8.0-openjdk-devel"]
# used to install Fusion (Solr) from a local (to the Ansible node) gzipped
# tarfile (if it exists)
local_solr_file: ""
```

This default configuration defines default values for all of the parameters needed to deploy an instance of Solr to a node, including defining reasonable defaults for the URL that the Fusion distribution should be downloaded from, the directory that the gzipped tarfile containing the Fusion distribution should be unpacked into, and the packages that must be installed on the node for Fusion to run.  To deploy Solr to a node the IP address "192.168.34.12" using the role in this repository, one would simply run a command that looks like this:

```bash
$ export SOLR_ADDR="192.168.34.12"
$ ansible-playbook -i "${SOLR_ADDR}," -e "{ host_inventory: ['${SOLR_ADDR}']}" site.yml
```

This will download the distribution from the main LucidWorks Fusion download site onto the host with an IP address of 192.168.34.12, unpack the gzipped tarfile that it downlaoded form that site into the `/opt/lucidworks` directory on that host, and install/configure the Solr server locally on that host.

Unfortunately, the main LucidWorks download site is often quite busy (and very slow as a result), so in our experience it has been a much better option to download the gzipped tarfile containing the Fusion distribution from that site once, then post it on a local webserver (one that is only available internally) and download it from the nodes we are setting up as Solr servers from that internal web server.  This can be accomplished by also including a definition for the `solr_url` parameter in the extra variables that are passed into the Ansible playbook run.  In that case, the command shown above ends up looking something like this:

```bash
$ export SOLR_URL="https://10.0.2.2/fusion-3.0.0.tar.gz"
$ export SOLR_ADDR="192.168.34.12"
$ ansible-playbook -i "${SOLR_ADDR}," -e "{ host_inventory: ['${SOLR_ADDR}'] \
    solr_url: '${SOLR_URL}'}" site.yml
```

(assuming that the file is available directly from a web-server that is accessible from the Solr server we are building via the IP address 10.0.2.2).  The path that the Solr distribution is unpacked into can also be overriden on the command-line in a similar fashion (using the `solr_dir` variable) if the default values shown in the `vars/solr.yml` file (above) proves to be problematic.

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions (and the `site.xml` playbook will not run successfully).  The examples shown above also assume that some (shared-key?) mechanism has been used to provide access to the Kafka host from the Ansible host that the ansible-playbook is being run on (if not, then additional arguments might be required to authenticate with that host from the Ansible host that are not shown in the example `ansible-playbook` commands shown above).

# Deployment via vagrant
A Vagrantfile is included in this repository that can be used to deploy Solr to a VM using the `vagrant` command.  From the top-level directory of this repository a command like the following will (by default) deploy Solr to a CentOS 7 virtual machine running under VirtualBox (assuming that both vagrant and VirtualBox are installed locally, of course):

```bash
$ vagrant -s="192.168.34.12" up
```

Note that the `-s` (or the corresponding `--solr-list`) flag must be used to pass an IP address into the Vagrantfile (this IP address will be used as the IP address of the Solr server that is created by the vagrant command shown above).

## Additional vagrant deployment options
While the `vagrant up` command that is shown above can be used to easily deploy Solr to a node, the Vagrantfile included in this distribution also supports separating out the creation of the virtual machine from the provisioning of that virtual machine using the Ansible playbook contained in this repository's `site.yml` file. To create a virtual machine without provisioning it, simply run a command that looks something like this:

```bash
$ vagrant -s="192.168.34.12" up --no-provision
```

This will create a virtual machine with the appropriate IP address ("192.168.34.12"), but will skip the process of provisioning that VM with an instance of Solr using the playbook in the `site.yml` file.  To provision that machine with a Solr instance, you would simply run the following command:

```bash
$ vagrant -s="192.168.34.12" provision
```

That command will attach to the named instance (the VM at "192.168.34.12") and run the playbook in this repository's `site.yml` file on that node (resulting in the deployment of an instance the LucidWorks Fusion distribution of Solr to that node).

It should also be noted here that while the commands shown above will install Solr with a reasonable default configuration from a standard location, there are two additional command-line parameters that can be used to override the default values that are embedded in the `vars/solr.yml` file that is included as part of this repository:  the `-u` (or corresponding `--url`) flag and the `-p` (or corresponding `--path`) flag.  The `-u` flag can be used to override the default URL that is used to download the LucidWorks Fusion (Solr) distribution (which points back to the main LucidWorks distribution site), while the `-p` flag can be used to override the default path (`/opt/lucidworks`) that that the LucidWorks Fusion gzipped tarfile is unpacked into during the provisioning process.

As an example of how these options might be used, the following command will download the gzipped tarfile containing the LucidWorks Fusion distribution from a local web server, rather than downloading it from the main LucidWorks distribution site and unpack the downladed gzipped tarfile into the `/opt/solr` directory when provisioning the VM with an IP address of `192.168.34.12` with an instance of the LucidWorks Fusion distribution:

```bash
$ vagrant -k="192.168.34.12" -p="/opt/solr" -u="https://10.0.2.2/fusion-3.0.0.tar.gz" provision
```

Obviously, this option could prove to be quite useful in situations were we are deploying the distribution from a datacenter environment (where access to the internet may be restricted, or even unavailable).