# dn-solr
Playbooks/Roles used to deploy LucidWorks Fusion (Solr)

# Installation
To install solr using the `site.yml` playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:
```bash
$ git clone --recursive https://github.com/Datanexus/dn-solr
```
That command will pull down the repository and it's submodules (currently the only dependency embedded as a submodule is the dependency on the `https://github.com/Datanexus/common-roles` repository).

# Use
To run the included playbook, change directories to the `dn-solr` subdirectory and run a set of commands that look something like the following (the commands shown here will install the most recent version of the LucidWorks Fusion (Solr)  distribution from the LucidWorks software repository, for example, onto a machine with at the IP address "192.168.34.12"):
```bash
$ export SOLR_URL="https://download.lucidworks.com/fusion-2.4.4.tar.gz"
$ export SOLR_ADDR="192.168.34.12"
$ export SOLR_DIR="/opt/fusion"
$ export SOLR_PACKAGE_LIST='["java-1.8.0-openjdk", "java-1.8.0-openjdk-devel"]'
$ echo "[all]\n${SOLR_ADDR}" > hosts
$ ansible-playbook site.yml --inventory-file hosts --extra-vars "solr_url=${SOLR_URL} \
    solr_addr=${SOLR_ADDR} solr_dir=${SOLR_DIR} solr_package_list=${SOLR_PACKAGE_LIST}"
```
All of the variables shown in this example must be defined for the playbook to run successfully.

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions (and the `site.xml` playbook will not run successfully).  The examples shown above also assume that some (shared-key?) mechanism has been used to provide access to the Kafka host from the Ansible host that the ansible-playbook is being run on (if not, then additional arguments might be required to authenticate with that host from the Ansible host that are not shown in the example `ansible-playbook` commands shown above).

# Deployment via vagrant
The included Vagrantfile can be used to deploy LucidWorks Fusion (Solr) to a VM using `Vagrant`.  From the top-level directory of this repostory a command like the following will (by default) deploy kafka to a CentOS 7 virtual machine running under VirtualBox (assuming that both vagrant and VirtualBox are installed locally, of course):
```bash
$ VAGRANT_DEFAULT_PROVIDER=virtualbox vagrant -s="192.168.34.12" up
```
Note that the `-s` (or the corresponding `--solr-addr`) flag must be used to pass an IP address into the Vagrantfile (this IP address will be used as the IP address of the Solr server that is created by the vagrant command shown above).  The Vagrantfile also includes a definition for a `solr_url` variable that can be updated to point to the gzipped tarfile containing the LucidWorks Fusion (Solr) distribution if you are interested in downloading that file from another source (the Vagrantfile included in the current repository assumes this file can be downloaded from a web server running on the localhost where the `vagrant up` command was run from).
