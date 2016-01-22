# -*- mode: ruby -*-
# vi: set ft=ruby :

# Please do not edit this file directly as it lives in the submodule. You can
# customize your Vagrant box by editing the following files :
#
#   * virtualization/parameters.yml for project related parameters
#   * virtualization/playbook.yml for provisioning option
#   * Vagrantfile should you need anything else
#
# If those options are not enough for you, please get in touch so we can
# devise something reusable for everyone !

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

def class_exists?(class_name)
  klass = Module.const_get(class_name)
  return klass.is_a?(Class)
rescue NameError
  return false
end

# Cross-platform way of finding an executable in the $PATH.
#   which('ruby') #=> /usr/bin/ruby
def which(cmd)
  exts = ENV['PATHEXT'] ? ENV['PATHEXT'].split(';') : ['']
  ENV['PATH'].split(File::PATH_SEPARATOR).each do |path|
    exts.each { |ext|
      exe = File.join(path, "#{cmd}#{ext}")
      return exe if File.executable?(exe) && !File.directory?(exe)
    }
  end
  return nil
end

# Some hack to detect the current provider took from https://groups.google.com/forum/#!topic/vagrant-up/XIxGdm78s4I
# We need it to detect LXC because there is currently an issue with LXC
# and ansible_local (see https://github.com/fgrehm/vagrant-lxc/issues/398)
# once this issue is resolved, this can be probably removed.
def get_provider
  provider_index = ARGV.index('--provider')
  if (provider_index && ARGV[provider_index + 1])
     return ARGV[provider_index + 1]
  elsif ARGV.index('--provider=lxc')
     return "lxc"
  end
  return ENV['VAGRANT_DEFAULT_PROVIDER'] || 'virtualbox'
end

# try to support both new and old Vagrantfile format by loading
# the config if this Vagrantfile was called directly
unless class_exists?('CustomConfig')
    puts "It seems you are using the old Vagrantfile format for this framework."
    puts "You should have a look at the docs/migrations.md file to read the"
    puts "steps required to go from the version 0.1.0 to 0.2.0."
    puts "---------------------------------------------------------------------"

    load 'virtualization/VagrantfileExtra.rb'

    # add a 'get' method to our config class (already added in the new format)
    class CustomConfig
        def get(name, default = nil)
            if self.respond_to?(name)
                self.send(name)
            elsif default.nil?
                raise "[CONFIG ERROR] '#{name}' cannot be found and no default provided."
            else
                default
            end
        end
    end
end

custom_config = CustomConfig.new

ansible_provisioner = custom_config.get('ansible_local', false) ? 'ansible_local' : 'ansible'

if get_provider() == 'lxc' and ansible_provisioner == 'ansible_local'
    puts "You are using the LXC provider and selected 'ansible_local'."
    puts "We automatically switched you back to 'ansible' because there"
    puts "currently is an issue : https://github.com/fgrehm/vagrant-lxc/issues/398"
    puts "-------------------------------------------------------------"
    ansible_provisioner = 'ansible'
end

if ansible_provisioner == 'ansible_local'
    Vagrant.require_version ">= 1.8.1"
else
    unless which 'ansible'
        raise "[PROVISIONER ERROR] cannot find ansible on the host. Try using ansible_local."
    end
    Vagrant.require_version ">= 1.6.2"
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    config.vm.box = custom_config.get('vbox_box_name', 'jessie64')
    config.vm.box_url = custom_config.get('vbox_box_url', 'http://vagrantbox-public.liip.ch/liip-jessie64.box')
    config.vm.hostname = custom_config.get('hostname')

    config.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true

    config.ssh.forward_agent = true
    config.ssh.forward_x11 = true

    if Vagrant.has_plugin?("vagrant-hostmanager")
        config.hostmanager.enabled = true
        config.hostmanager.manage_host = true
        config.hostmanager.ignore_private_ip = false
        config.hostmanager.include_offline = true
        config.hostmanager.aliases = custom_config.get('load_aliases', [])
    end

    if Vagrant.has_plugin?("vagrant-cachier")
        # cache the packages for all identical boxes
        config.cache.scope = :box
    end

    config.vm.provider "virtualbox" do |v, override|
        override.vm.network :private_network, ip: custom_config.get('box_ip')
        override.vm.synced_folder ".", "/vagrant", type: "nfs", mount_options: ['noatime', 'noacl', 'proto=udp', 'vers=3', 'async', 'actimeo=1']

        v.memory = 4096
        v.cpus = 2
        v.customize ["modifyvm", :id, "--natdnsproxy1", "off"]
        v.customize ["modifyvm", :id, "--natdnshostresolver1", "off"]

        if Vagrant.has_plugin?("vagrant-cachier")
            # use the same nfs config than above for cache performance
            override.cache.synced_folder_opts type: "nfs", mount_options: ['noatime', 'noacl', 'proto=udp', 'vers=3', 'async', 'actimeo=1']
        end
    end

    config.vm.provider "lxc" do |lxc, override|
        override.vm.box = custom_config.get('lxc_box_name', 'jessie64-lxc')
        override.vm.box_url = custom_config.get('lxc_box_url', 'http://vagrantbox-public.liip.ch/liip-jessie64-lxc.box')

        if Vagrant.has_plugin?("vagrant-hostmanager")
            override.hostmanager.ignore_private_ip = true
        end
    end

    config.vm.provision ansible_provisioner do |ansible|
        ansible.extra_vars = custom_config.get('extra_vars', {})
        ansible.playbook = custom_config.get('playbook', 'virtualization/playbook.yml')

        # force handlers to run even on playbook errors
        ansible.raw_arguments = ['--force-handlers']

        # Update verbosity as needed, multiples 'v' means more verbose
        # ansible.verbose = 'v'

        if ansible_provisioner == 'ansible'
            ansible.host_key_checking = false
            # We can't use controlmaster because it makes SSH use the same
            # connection over and over again. This clashes with the base role which
            # changes sshd configuration. If controlmaster is used, the new sshd
            # configuration is not taken into account
            ansible.raw_ssh_args = ['-o ControlMaster=no']
        else
            # try to install ansible in the box so that we don't have to recreate them
            ansible.install = true
        end
    end

    if ENV['GITLAB_CI'] && ENV['DO_GLOBAL_PROJECTS_CACHE']
        config.vm.synced_folder "/home/gitlab-runner/projects_cache/", "/home/vagrant/.projects_cache"
    end
end
