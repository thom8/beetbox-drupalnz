# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

Vagrant.require_version '>= 1.8.0'

cwd = File.dirname(File.expand_path(__FILE__))
config_dir = '.beetbox/'
project_config = "#{cwd}/#{config_dir}config.yml"
local_config = "#{cwd}/#{config_dir}local.config.yml"

# Default vagrant config.
vconfig = {
  'vagrant_box' => 'beet/box',
  'vagrant_box_version' => '>= 0.2.0',
  'vagrant_ip' => '0.0.0.0',
  'vagrant_memory' => 1024,
  'vagrant_cpus' => 2,
  'beet_home' => '/beetbox',
  'beet_base' => '/var/beetbox',
  'beet_domain' => cwd.split('/').last.gsub(/[\._]/, '-') + ".local"
}

if !File.exist?(project_config)
  # Create default config file.
  require 'fileutils'
  FileUtils::mkdir_p config_dir
  File.open(project_config, "w+") {|f| f.write("---\nbeet_domain: #{vconfig['beet_domain']}\n") }
end

pconfig = YAML::load_file(project_config) || nil
vconfig = vconfig.merge pconfig if !pconfig.nil?

# Merge local.config.yml
if File.exist?(local_config)
  lconfig = YAML::load_file(local_config) || nil
  vconfig = vconfig.merge lconfig if !lconfig.nil?
end

# Replace variables in YAML config.
vconfig.each do |key, value|
  while vconfig[key].is_a?(String) && vconfig[key].match(/{{ .* }}/)
    vconfig[key] = vconfig[key].gsub(/{{ (.*?) }}/) { |match| match = vconfig[$1] }
  end
end

hostname = vconfig['beet_domain']
branches = ['beetbox']
current_branch = 'beetbox'

Vagrant.configure("2") do |config|

  # Multidev config.
  if vconfig['beet_mode'] == 'multidev'
    branches = %x(git branch | tr -d '* ').split(/\n/).reject(&:empty?)
    branches.unshift("beetbox")
    current_branch = %x(git branch | grep '*' | tr -d '* \n')
    vconfig['vagrant_ip'] = "0.0.0.0"
    branch_prefix = true
  end

  # Check for plugins and attempt to install if not (Windows only).
  if Vagrant::Util::Platform.windows?
    %x(vagrant plugin install vagrant-hostsupdater) unless Vagrant.has_plugin?('vagrant-hostsupdater')
    raise 'Your config requires hostsupdater plugin.' unless Vagrant.has_plugin?('vagrant-hostsupdater')
    if vconfig['vagrant_ip'] == "0.0.0.0"
      %x(vagrant plugin install vagrant-auto_network) unless Vagrant.has_plugin?('vagrant-auto_network')
      raise 'Your config requires auto_network plugin.' unless Vagrant.has_plugin?('vagrant-auto_network')
    end
  end

  branches.each do |branch|
    active_node = (branch == current_branch) ? true : false
    config.vm.define branch, autostart: active_node, primary: active_node do |node|

      node.vm.box = vconfig['vagrant_box']
      node.vm.box_version = vconfig['vagrant_box_version']
      node.vm.hostname = (branch_prefix) ? "#{branch}.#{hostname}" : hostname
      node.ssh.insert_key = false
      node.ssh.forward_agent = true

      # Network config.
      if vconfig['vagrant_ip'] == "0.0.0.0" && Vagrant::Util::Platform.windows?
        node.vm.network :private_network, :ip => "0.0.0.0", :auto_network => true
      elsif vconfig['vagrant_ip'] == "0.0.0.0"
        node.vm.network :private_network, :type => "dhcp"
      else
        node.vm.network :private_network, ip: vconfig['vagrant_ip']
      end

      # Synced folders.
      node.vm.synced_folder ".", vconfig['beet_base'],
        type: "nfs",
        id: "beetbox"

      if vconfig['beet_debug']
        node.vm.synced_folder "./provisioning", "#{vconfig['beet_home']}/provisioning",
          type: "nfs",
          id: "debug"
        debug_mode = "BEET_DEBUG=true"
      end

      # Upload vagrant.config.yml
      node.vm.provision "project_config", type: "file" do |s|
       s.source = project_config
       s.destination = "~/vagrant.config.yml"
      end

      # Upload local.config.yml
      if File.exist?(local_config)
        node.vm.provision "local_config", type: "file" do |s|
         s.source = local_config
         s.destination = "~/local.config.yml"
        end
      end

      # Provision box
      beet_sh = "#{vconfig['beet_home']}/provisioning/beetbox.sh"
      remote_sh = "https://raw.githubusercontent.com/beetboxvm/beetbox/master/provisioning/beetbox.sh"
      local_provision = "sudo chmod +x #{beet_sh} && #{debug_mode} #{beet_sh}"
      remote_provision = "sudo apt-get -qq update && curl -fsSL #{remote_sh} | sudo bash"
      node.vm.provision "ansible", type: "shell" do |s|
        s.privileged = false
        s.inline = "if [ -f #{beet_sh} ]; then #{local_provision}; else #{remote_provision}; fi"
      end

      # VirtualBox.
      node.vm.provider :virtualbox do |v|
        v.name = "#{node.vm.hostname}.#{Time.now.to_i}"
        v.memory = vconfig['vagrant_memory']
        v.cpus = vconfig['vagrant_cpus']
        v.linked_clone = true
        v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
        v.customize ["modifyvm", :id, "--ioapic", "on"]
      end
    end
  end
end

# Create local drush alias.
if File.directory?("#{Dir.home}/.drush")

  alias_file = "#{Dir.home}/.drush/"+hostname+".aliases.drushrc.php"
  if ARGV[0] == "destroy"
    File.delete(alias_file) if File.exist?(alias_file)
  else
    require 'erb'
    class DrushAlias
      attr_accessor :hostname, :uri, :key, :root
      def template_binding
        binding
      end
    end

    template = <<ALIAS
<?php

$aliases['<%= @hostname %>'] = array(
   'uri' => '<%= @uri %>',
   'remote-host' => '<%= @uri %>',
   'remote-user' => 'vagrant',
   'ssh-options' => '-i <%= @key %> -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no',
   'root' => '<%= @root %>',
);
ALIAS

    alias_file = File.open(alias_file, "w+")
    da = DrushAlias.new
    da.hostname = hostname
    da.uri = hostname
    da.key = "#{Dir.home}/.vagrant.d/insecure_private_key"
    da.root = vconfig['beet_web'] ||= vconfig['beet_root'] ||= vconfig['beet_base']
    alias_file << ERB.new(template).result(da.template_binding)
    alias_file.close
  end
end
