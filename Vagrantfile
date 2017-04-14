# (c) 2016 DataNexus Inc.  All Rights Reserved
# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'optparse'
require 'resolv'

# monkey-patch that is used to leave unrecognized options in the ARGV
# list so that they can be processed by underlying vagrant command
class OptionParser
  # Like order!, but leave any unrecognized --switches alone
  def order_recognized!(args)
    extra_opts = []
    begin
      order!(args) { |a| extra_opts << a }
    rescue OptionParser::InvalidOption => e
      extra_opts << e.args[0]
      retry
    end
    args[0, 0] = extra_opts
  end
end

# a function that is used to parse Ansible (static) inventory files and
# return a list of the node addresses contained in the file
def addr_list_from_inventory_file(inventory_file)
  first_field_list = []
  File.open(inventory_file, 'r') do |f|
    f.each_line do |line|
      # grab the first field from each line
      first_field_list << line.gsub(/\s+/, ' ').strip.split(" ")[0]
    end
  end
  # return the entries that look like IP addresses (skipping the rest)
  # and only return the unique values in the resulting list
  first_field_list.select { |addr| (addr =~ Resolv::IPv4::Regex) }.uniq
end

# initialize a few values
options = {}
VALID_ZK_ENSEMBLE_SIZES = [3, 5, 7]

# vagrant commands that include these commands can be run without specifying
# any IP addresses
no_ip_commands = ['version', 'global-status', '--help', '-h']
# vagrant commands that only work for a single IP address
single_ip_commands = ['status', 'ssh']
# vagrant command arguments that indicate we are provisioning a cluster (if multiple
# nodes are supplied via the `--solr-list` flag)
provisioning_command_args = ['up', 'provision']
no_zk_required_command_args = ['destroy']
not_provisioning_flag = ['--no-provision']

optparse = OptionParser.new do |opts|
  opts.banner    = "Usage: #{opts.program_name} [options]"
  opts.separator "Options"

  options[:solr_list] = nil
  opts.on( '-s', '--solr-list A1,A2[,...]', 'Solr address list (multi-node commands)' ) do |solr_list|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-s=192.168.1.1')
    options[:solr_list] = solr_list.gsub(/^=/,'')
  end

  options[:inventory_file] = nil
  opts.on( '-i', '--inventory-file FILE', 'Zookeeper (Ansible) inventory file' ) do |inventory_file|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-i=/tmp/zookeeper_inventory')
    options[:inventory_file] = inventory_file.gsub(/^=/,'')
  end

  options[:solr_path] = nil
  opts.on( '-p', '--path SOLR_DIR', 'Path where the distribution should be installed' ) do |solr_path|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-p=192.168.1.1')
    options[:solr_path] = solr_path.gsub(/^=/,'')
  end

  options[:solr_url] = nil
  opts.on( '-u', '--url SOLR_URL', 'URL the distribution should be downloaded from' ) do |solr_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-u=http://localhost/tmp.tgz')
    options[:solr_url] = solr_url.gsub(/^=/,'')
  end

  options[:local_solr_file] = nil
  opts.on( '-l', '--local-file FILE', 'Local (gzipped tar)file containing Fusion (Solr) distribution' ) do |local_solr_file|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-l=/tmp/fusion-2.4.4.tar.gz')
    options[:local_solr_file] = local_solr_file.gsub(/^=/,'')
  end

  options[:solr_data_dir] = nil
  opts.on( '-d', '--data DATA_DIR', 'Data directory for Solr' ) do |solr_data_dir|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-r="/data"')
    options[:solr_data_dir] = solr_data_dir.gsub(/^=/,'')
  end

  options[:yum_repo_url] = nil
  opts.on( '-y', '--yum-url URL', 'Local yum repository URL' ) do |yum_repo_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-y=http://192.168.1.128/centos')
    options[:yum_repo_url] = yum_repo_url.gsub(/^=/,'')
  end

  options[:local_vars_file] = nil
  opts.on( '-f', '--local-vars-file FILE', 'Local variables file' ) do |local_vars_file|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-f=/tmp/local-vars-file.yml')
    options[:local_vars_file] = local_vars_file.gsub(/^=/,'')
  end

  options[:reset_proxy_settings] = false
  opts.on( '-c', '--clear-proxy-settings', 'Clear existing proxy settings if no proxy is set' ) do |reset_proxy_settings|
    options[:reset_proxy_settings] = true
  end

  opts.on_tail( '-h', '--help', 'Display this screen' ) do
    print opts
    exit
  end

end

begin
  optparse.order_recognized!(ARGV)
rescue SystemExit => e
  exit
rescue Exception => e
  print "ERROR: could not parse command (#{e.message})\n"
  print optparse
  exit 1
end

# check remaining arguments to see if the command requires
# an IP address (or not)
ip_required = (ARGV & no_ip_commands).empty?
# check the remaining arguments to see if we're provisioning or not
provisioning_command = !((ARGV & provisioning_command_args).empty?) && (ARGV & not_provisioning_flag).empty?
# and to see if multiple IP addresses are supported (or not) for the
# command being invoked
single_ip_command = !((ARGV & single_ip_commands).empty?)
# and to see if a zookeeper inventory must also be provided
no_zk_required_command = !(ARGV & no_zk_required_command_args).empty?

if options[:solr_url] && !(options[:solr_url] =~ URI::regexp)
  print "ERROR; input solr URL '#{options[:solr_url]}' is not a valid URL\n"
  exit 3
end

if options[:local_solr_file] && !File.file?(options[:local_solr_file])
  print "ERROR; input local solr file '#{options[:local_solr_file]}' is not a local file\n"
  exit 3
end

if options[:solr_url] && options[:local_solr_file]
  print "ERROR; the solr-url option and the local-solr-file options cannot be combined\n"
  exit 2
end

if options[:inventory_file] && !File.file?(options[:inventory_file])
  print "ERROR; the if a zookeeper list is defined, a zookeeper inventory file must also be provided\n"
  exit 2
end

# if a local variables file was passed in, check and make sure it's a valid filename
if options[:local_vars_file] && !File.file?(options[:local_vars_file])
  print "ERROR; input local variables file '#{options[:local_vars_file]}' is not a local file\n"
  exit 3
end

# if we're provisioning, then the `--solr-list` flag must be provided and either contain
# a single node (for single-node deployments) or multiple nodes in a comma-separated list
# (for multi-node deployments) that define a valid solr cluster
solr_addr_array = []
if provisioning_command || ip_required
  if !options[:solr_list]
    print "ERROR; IP address must be supplied (using the `-s, --solr-list` flag) for this vagrant command\n"
    exit 1
  else
    solr_addr_array = options[:solr_list].split(',').map { |elem| elem.strip }.reject { |elem| elem.empty? }
    if solr_addr_array.size == 1
      if !(solr_addr_array[0] =~ Resolv::IPv4::Regex)
        print "ERROR; input Solr IP address #{solr_addr_array[0]} is not a valid IP address\n"
        exit 2
      end
    elsif !single_ip_command
      # check the input `solr_addr_array` to ensure that all of the values passed in are
      # legal IP addresses
      not_ip_addr_list = solr_addr_array.select { |addr| !(addr =~ Resolv::IPv4::Regex) }
      if not_ip_addr_list.size > 0
        # if some of the values are not valid IP addresses, print an error and exit
        if not_ip_addr_list.size == 1
          print "ERROR; input Solr IP address #{not_ip_addr_list} is not a valid IP address\n"
          exit 2
        else
          print "ERROR; input Solr IP addresses #{not_ip_addr_list} are not valid IP addresses\n"
          exit 2
        end
      end
      # when provisioning a multi-node Solr cluster, we **must** have an associated zookeeper
      # ensemble consisting of an odd number of nodes greater than three, but less than seven
      # (any other topology is not supported, so an error is thrown)
      if provisioning_command && solr_addr_array.size > 1 && !no_zk_required_command
        if !options[:inventory_file]
          print "ERROR; A zookeeper inventory file must be supplied (using the `-i, --inventory-file` flag)\n"
          print "       containing the (static) inventory file for an existing Zookeeper ensemble when\n"
          print "       provisioning a Solr cluster\n"
          exit 1
        else
          # parse the inventory file that was passed in and retrieve the list of host addresses from it
          zookeeper_addr_array = addr_list_from_inventory_file(options[:inventory_file])
          # and check to make sure that an appropriate number of zookeeper addresses were
          # found in the inventory file (the size of the ensemble should be an odd number
          # between three and seven)
          if !(VALID_ZK_ENSEMBLE_SIZES.include?(zookeeper_addr_array.size))
            print "ERROR; only a zookeeper cluster with an odd number of elements between three and\n"
            print "       seven is supported for multi-node solr deployments; the defined cluster\n"
            print "       #{zookeeper_addr_array} contains #{zookeeper_addr_array.size} elements\n"
            exit 5
          end
          # finally, we need to make sure that the machines we're deploying solr to are not the same
          # machines that make up our zookeeper ensemble (the zookeeper ensemble must be on a separate
          # set of machines from the solr cluster)
          same_addr_list = zookeeper_addr_array & solr_addr_array
          if same_addr_list.size > 0
            print "ERROR; the solr cluster cannot be deployed to the same machines that make up\n"
            print "       the zookeeper ensemble; requested clusters overlap for the machines at\n"
            print "       #{same_addr_list}\n"
            exit 7
          end
        end
      end
    end
  end
end

# if a yum repository address was passed in, check and make sure it's a valid URL
if options[:yum_repo_url] && !(options[:yum_repo_url] =~ URI::regexp)
  print "ERROR; input yum repository URL '#{options[:yum_repo_url]}' is not a valid URL\n"
  exit 6
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
if solr_addr_array.size > 0
  Vagrant.configure("2") do |config|
    proxy = ENV['http_proxy'] || ""
    no_proxy = ENV['no_proxy'] || ""
    proxy_username = ENV['proxy_username'] || ""
    proxy_password = ENV['proxy_password'] || ""
    if Vagrant.has_plugin?("vagrant-proxyconf")
      if $proxy
        config.proxy.http               = $proxy
        config.vm.box_download_insecure = true
        config.vm.box_check_update      = false
      end
      if $no_proxy
        config.proxy.no_proxy           = $no_proxy
      end
      if $proxy_username
        config.proxy.proxy_username     = $proxy_username
      end
      if $proxy_password
        config.proxy.proxy_password     = $proxy_password
      end
    end

    # Every Vagrant development environment requires a box. You can search for
    # boxes at https://atlas.hashicorp.com/search.
    config.vm.box = "centos/7"
    # config.vm.box = "iamseth/rhel-7.3"
    config.vm.box_check_update = false

    # loop through all of the addresses in the `solr_addr_array` and, if we're
    # creating VMs, create a VM for each machine; if we're just provisioning the
    # VMs using an ansible playbook, then wait until the last VM in the loop and
    # trigger the playbook runs for all of the nodes simultaneously using the
    # `site.yml` playbook
    solr_addr_array.each do |machine_addr|
      # Customize the amount of memory on the VM
      config.vm.provider "virtualbox" do |vb|
        vb.memory = "8192"
      end
      config.vm.define machine_addr do |machine|
        # Create a two private networks, which each allow host-only access to the machine
        # using a specific IP.
        if machine_addr
          # configure a private network with the input address
          config.vm.network "private_network", ip: machine_addr
          # and configure a second private network based on that address
          # (by simply shifting the third octet up by one)
          split_addr = machine_addr.split('.')
          api_addr = (split_addr[0..1] + [(split_addr[2].to_i + 10).to_s] + [split_addr[3]]).join('.')
          config.vm.network "private_network", ip: api_addr
          # if this is the last machine in the list, then define a couple of
          # CIDR blocks that can be used to differentiate between these two networks
          # (where the first is the data network and the second is the api network);
          # note we're assuming that a (set of) '\24' network address(es) was(were)
          # passed in by the user and that all nodes are on the same '\24' network
          # in this block of code
          if machine_addr == solr_addr_array[-1]
            data_cidr = (split_addr[0..2] + ['0']).join('.') + '/24'
            api_cidr = (split_addr[0..1] + [(split_addr[2].to_i + 10).to_s] + ['0']).join('.') + '/24'
          end
        end
        # if it's the last node in the list if input addresses, then provision
        # all of the nodes simultaneously (if the `--no-provision` flag was not
        # set, of course)
        if machine_addr == solr_addr_array[-1]
          # now, use the playbook in the `site.yml' file to provision our
          # nodes with Solr (and configure them as a cluster if there
          # is more than one node)
          machine.vm.provision "ansible" do |ansible|
            # set the limit to 'all' in order to provision all of machines on the
            # list in a single playbook run
            ansible.limit = "all"
            ansible.playbook = "site.yml"
            ansible.groups = {
              solr: solr_addr_array
            }
            ansible.extra_vars = {
              proxy_env: {
                http_proxy: proxy,
                no_proxy: no_proxy,
                proxy_username: proxy_username,
                proxy_password: proxy_password
              },
              # data_iface: 'eth1',
              # api_iface: 'eth2',
              iface_description_array: [
                { as_var: 'data_iface', type: 'cidr', val: data_cidr },
                { as_var: 'api_iface', type: 'cidr', val: api_cidr },
              ],
              yum_repo_url: options[:yum_repo_url],
              local_solr_file: options[:local_solr_file],
              host_inventory: solr_addr_array,
              reset_proxy_settings: options[:reset_proxy_settings],
              zookeeper_inventory_file: options[:inventory_file],
              # overrides a few values from the 'vars/solr.yml' file that
              # set options for the JVM we can't satisfy when testing locally
              solr_java_ops: "-Xmx2g -Xss256k",
              ui_java_ops: "-Xmx512m",
              connectors_java_ops: "-Xmx1g -Xss256k",
              # set the cloud type to 'vagrant' (which will trigger the use
              # of static inventory in the playbook)
              cloud: "vagrant"
            }
            # if defined, set the 'extra_vars[:solr_url]' value to the value that was passed in on
            # the command-line (eg. "https://10.0.2.2/fusion-2.4.4.tar.gz")
            if options[:solr_url]
              ansible.extra_vars[:solr_url] = options[:solr_url]
            end
            # if defined, set the 'extra_vars[:solr_dir]' value to the value that was passed in on
            # the command-line (eg. "/opt/solr")
            if options[:solr_path]
              ansible.extra_vars[:solr_dir] = options[:solr_path]
            end
            # if defined, set the 'extra_vars[:solr_data_dir]' value to the value that was passed
            # in on the command-line
            if options[:solr_data_dir]
              ansible.extra_vars[:solr_data_dir] = options[:solr_data_dir]
            end
            # if defined, set the 'extra_vars[:local_vars_file]' value to the value that was passed in
            # on the command-line (eg. "/tmp/local-vars-file.yml")
            if options[:local_vars_file]
              ansible.extra_vars[:local_vars_file] = options[:local_vars_file]
            end
          end     # end `machine.vm.provision "ansible" do |ansible|`
        end     # end `if machine_addr == solr_addr_array[-1]`
      end     # end `config.vm.define machine_addr do |machine|`
    end     # end `solr_addr_array.each do |machine_addr|`
  end     # end `Vagrant.configure ("2") do |config|`
end     # end `if solr_addr_array.size > 0`
