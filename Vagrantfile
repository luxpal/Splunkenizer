# -*- mode: ruby -*-
# vi: set ft=ruby :

# Specify minimum Vagrant version and Vagrant API version
Vagrant.require_version '>= 1.6.0'
VAGRANTFILE_API_VERSION = '2'

require 'yaml'
dir = File.dirname(File.expand_path(__FILE__))
config_file = "splunk_config.yml"
config_dir = "config"

if !File.directory?("#{dir}/#{config_dir}")
  FileUtils.mkdir_p("#{dir}/#{config_dir}")
end

if !File.file?("Vagrantfile")
  print "ERROR: Run the command from the top directory, where 'Vagrantfile' is located!\n"
  exit 2
end

if !File.file?("#{dir}/#{config_dir}/#{config_file}")
  print "ERROR: Please copy a config file from the examples dir to #{dir}/#{config_dir}/#{config_file} to continue\n"
  exit 2
end

# Read YAML file with instance information (box, CPU, RAM, IP addresses)
# Edit the config file to change VM and environment configuration details
settings = YAML.load_file("#{dir}/#{config_dir}/#{config_file}")

# Check for Splunk baseconfig apps
check_base = Dir.glob(dir+"/"+settings['splunk_basics']['splunk_baseconfig_dir']+"/*/org_all_indexer_base")
check_cluster = Dir.glob(dir+"/"+settings['splunk_basics']['splunk_baseconfig_dir']+"/*/org_cluster_indexer_base")
if check_base.length < 1 or check_cluster.length < 1
  print "ERROR: Please download the Splunk baseconfig apps here and extract it into #{dir}/#{settings['splunk_basics']['splunk_baseconfig_dir']} \n\n"
  print "Configurations Base:      https://splunk.app.box.com/ConfigurationsBase\n"
  print "Configurations Cluster:   https://splunk.app.box.com/ConfigurationsCluster\n"
  exit 2
end

# Check for Splunk installer archives
settings['splunk_environments'].each do |splunkenv|
  ['splunk','splunkforwarder'].each do |splunk_arch|
    check_arch = Dir.glob(dir+"/"+settings['splunk_basics']['splunk_software_dir']+"/"+splunk_arch+"-"+splunkenv['splunk_version']+"-*Linux-x86_64.tgz")
    if check_arch.length < 1
      print "ERROR: Splunk Enterprise version #{splunkenv['splunk_version']} missing in #{dir}/#{settings['splunk_basics']['splunk_software_dir']}\n\n"
      print "Download "+splunk_arch+"-"+splunkenv['splunk_version']+"-...-Linux-x86_64.tgz at https://www.splunk.com\n"
      exit 2
    end
  end
  # Check for Splunk license file
  if !splunkenv['splunk_license_file'].nil?
    if !File.file?(dir+"/"+settings['splunk_basics']['splunk_software_dir']+"/"+splunkenv['splunk_license_file'])
      print "ERROR: Cannot find license file #{settings['splunk_basics']['splunk_software_dir']+"/"+splunkenv['splunk_license_file']} \n\n"
      print "Comment variable 'splunk_license_file' in #{dir}/#{config_dir}/#{config_file} if no license available\n"
      exit 2
    end
  end
end

# create ansible inventory from the config file
groups = {}
idxc_list = {}
idxc_sites = {}
shc_list = {}
# Create inventory host groups
settings['splunk_hosts'].each do |splunk_host|
  splunk_host['roles'].each do |role|
    if groups.has_key?("role_"+role)
      groups["role_"+role].push(splunk_host['name'])
    else
      groups["role_"+role] = [splunk_host['name']]
    end
    var_obj = {}
    # Add the management nodes here as well, since the role_* groups are not completely populated from the beginning
    ['deployment_server','monitoring_console', 'license_master'].each do |role_def|
      if role == role_def
        # Check if license file is available for license master
        if role_def == "license_master" and settings['splunk_environments'].find {|i| i["splunk_env_name"] == splunk_host['splunk_env']}["splunk_license_file"].nil?
          print "ERROR: Role 'license_master' on Splunk host '#{splunk_host['name']}' is not allowed if no license file available! \n\n"
          exit 2
        end
        var_obj["splunk_"+role_def] = splunk_host['name']
        groupname = "splunk_env_"+splunk_host['splunk_env'] +":vars"
        if groups.has_key?(groupname)
          groups[groupname] = groups[groupname].merge(var_obj)
        else
          groups[groupname] = var_obj
        end
      end
    end
    # Create indexer cluster member list
    if role == 'indexer' and splunk_host['idxcluster']
      if idxc_list.has_key?(splunk_host['idxcluster'])
        if not idxc_list[splunk_host['idxcluster']].include?(splunk_host['name'])
          idxc_list[splunk_host['idxcluster']].push(splunk_host['name'])
        end
      else
        idxc_list[splunk_host['idxcluster']] = [splunk_host['name']]
      end
      if splunk_host['site']
        if idxc_sites.has_key?(splunk_host['idxcluster'])
          if not idxc_sites[splunk_host['idxcluster']].include?(splunk_host['site'])
            idxc_sites[splunk_host['idxcluster']].push(splunk_host['site'])
          end
        else
          idxc_sites[splunk_host['idxcluster']] = [splunk_host['site']]
        end
      end
    end
    # Create search head cluster member list
    if role == 'search_head' and splunk_host['shcluster']
      if shc_list.has_key?(splunk_host['shcluster'])
        if not shc_list[splunk_host['shcluster']].include?(splunk_host['name'])
          shc_list[splunk_host['shcluster']].push(splunk_host['name'])
        end
      else
        shc_list[splunk_host['shcluster']] = [splunk_host['name']]
      end
    end
  end

  # Splunk environment and cluster groups
  ['splunk_env','idxcluster','shcluster'].each do |var|
    tmparray = []
    if splunk_host[var].kind_of?(Array)
      tmparray = splunk_host[var]
    else
      tmparray = [splunk_host[var]]
    end
    tmparray.each do |item|
      if splunk_host[var]
        groupname = var+"_"+item
        if groups.has_key?(groupname)
          groups[groupname].push(splunk_host['name'])
        else
          groups[groupname] = [splunk_host['name']]
        end
      end
    end
  end
end

# Cleanup old group_vars files
Dir['ansible/group_vars/**/*'].select{|f| File.file?(f) }.each do |filepath|
  File.delete(filepath) if File.basename(filepath) != "dynamic.yml"
end

# Create general group_vars
if !settings['general'].nil?
  File.open("ansible/group_vars/all/general.yml", "w") do |f|
    f.write(settings['general'].to_yaml)
  end
end

# Create splunk_basics group_vars
if !settings['splunk_basics'].nil?
  File.open("ansible/group_vars/all/splunk_basics.yml", "w") do |f|
    f.write(settings['splunk_basics'].to_yaml)
  end
end

# Create splunk environment group_vars
if !settings['splunk_environments'].nil?
  settings['splunk_environments'].each do |splunkenv|
    group_name = "splunk_env_"+splunkenv['splunk_env_name']
    env_vars = group_name+":vars"
    if groups.has_key?(env_vars)
      splunkenv = splunkenv.merge(groups[env_vars])
    end
    File.open("ansible/group_vars/#{group_name}.yml", "w") do |f|
      f.write(splunkenv.to_yaml)
    end
  end
end

# Create splunk indexer cluster group_vars
if !settings['splunk_idxclusters'].nil?
  settings['splunk_idxclusters'].each do |idxcluster|
    group_name = "idxcluster_"+idxcluster['idxc_name']
    if idxc_sites[idxcluster['idxc_name']]
      idxcluster['idxc_available_sites'] = idxc_sites[idxcluster['idxc_name']]
    end
    idxcluster['idxc_members'] = idxc_list[idxcluster['idxc_name']]
    File.open("ansible/group_vars/#{group_name}.yml", "w") do |f|
      f.write(idxcluster.to_yaml)
    end
  end
end

# Create splunk search head cluster group_vars
if !settings['splunk_shclusters'].nil?
  settings['splunk_shclusters'].each do |shcluster|
    group_name = "shcluster_"+shcluster['shc_name']
    shcluster['shc_captain'] = shc_list[shcluster['shc_name']].last
    shcluster['shc_members'] = shc_list[shcluster['shc_name']]
    File.open("ansible/group_vars/#{group_name}.yml", "w") do |f|
      f.write(shcluster.to_yaml)
    end
  end
end

# If dynamic IPs are used, calculate the start number
if !settings['general']['start_ip'].nil?
  ip_array = settings['general']['start_ip'].split(".")
  base_ip = ip_array[0..2].join(".")
  start_num = ip_array[3].to_i
end

# Create inventory host vars
host_vars = {}
splunk_host_list = []
settings['splunk_hosts'].each do |splunk_host|
  per_host_vars = {}
  ['ip_addr', 'site'].each do |var|
    if !settings['general']['start_ip'].nil?
      # Keep the ip, if defined
      if splunk_host['ip_addr'].nil?
        splunk_host['ip_addr'] = base_ip+"."+start_num.to_s
      end
    end
    if splunk_host[var]
      var_obj = {}
      var_obj[var] = splunk_host[var]
      per_host_vars=per_host_vars.merge(var_obj)
    end
  end
  host_vars[splunk_host['name']] = per_host_vars
  start_num += 1
  hosts_entry = {}
  hosts_entry['name'] = splunk_host['name']
  hosts_entry['ip_addr'] = splunk_host['ip_addr']
  splunk_host_list.push(hosts_entry)
end

# Write out hosts content to build /etc/hosts file
splunk_hosts = {}
splunk_hosts['splunk_hosts'] = splunk_host_list
File.open("ansible/group_vars/all/hosts.yml", "w") do |f|
  f.write(splunk_hosts.to_yaml)
end

# Ansible settings
if !settings['ansible'].nil?
  if settings['ansible'].has_key?('verbose')
    ansible_verbose = settings['ansible']['verbose']
  end
end

# Create and configure the specified systems
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  # Loop through YAML file and set per-VM information
  settings['splunk_hosts'].each do |server|
    config.vm.define server['name'] do |srv|

      # Setting hostname
      srv.vm.hostname = server['name']

      # Box image to use
      srv.vm.box = settings['virtualbox']['box']

      # Don't check for box updates
      #srv.vm.box_check_update = false

      # Splunk need some time to shutdown 
      srv.vm.boot_timeout = 600
      srv.vm.graceful_halt_timeout = 600

      # Install vbguest additions
      config.vbguest.no_install = true
      if !settings['virtualbox']['install_vbox_additions'].nil? and settings['virtualbox']['install_vbox_additions']
        if !Vagrant.has_plugin?("vagrant-vbguest")
          print "ERROR: You need to install the plugin vagrant-vbguest, to be able to mount local folders in guests.\n"
          exit 2
        end
        config.vbguest.no_install = false       
      end

      # Turn off vbox guest addition installation, if no folders shared
      if !settings['virtualbox']['synced_folder'].nil? and settings['virtualbox']['synced_folder'].length > 0
        if config.vbguest.no_install == true
          print "ERROR: 'install_vbox_additions' must be set to 'true' to enable synced folders!\n"
          exit 2
        end
      end

      # Disable default shared folder
      srv.vm.synced_folder '.', '/vagrant', disabled: true

      if Vagrant.has_plugin?("vagrant-vbguest") and config.vbguest.no_install == false
        # Add custom synced folders, if vboxsf is available
        if !settings['virtualbox']['synced_folder'].nil?
          settings['virtualbox']['synced_folder'].each do |folder|
          srv.vm.synced_folder "#{folder['source']}", "#{folder['target']}"
          end
        end

        # Use Caching for package deployments
        if Vagrant.has_plugin?("vagrant-cachier")
          config.cache.scope = :box
        end
      end
  
      srv.vm.network :private_network, ip: server['ip_addr']
      #srv.vm.network "forwarded_port", guest: 8000, host: 8000, auto_correct: true

      # Set per-server VirtualBox provider configuration/overrides
      srv.vm.provider 'virtualbox' do |vb, override|
        #override.vm.box = server['box']['vb']
        vb.memory = settings['virtualbox']['memory']
        vb.cpus = settings['virtualbox']['cpus']
        if server.has_key?('virtualbox')
          vb.memory = server['virtualbox']['memory'] if !server['virtualbox']['memory'].nil?
          vb.cpus = server['virtualbox']['cpus'] if !server['virtualbox']['cpus'].nil?
        end
        # Fix clock skew problem
        vb.customize ["setextradata", :id, "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled", "1"]
      end

      srv.vm.provision "ansible" do |ansible|
        ansible.compatibility_mode = "2.0"
        ansible.groups = groups
        ansible.host_vars = host_vars
        ansible.skip_tags = ['packages','fix_clock_skew']
        if !ansible_verbose.nil?
          ansible.verbose = ansible_verbose
        end
        ansible.playbook = "ansible/site.yml"
      end
    end
  end
end
