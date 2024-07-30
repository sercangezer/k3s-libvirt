# ENV['VAGRANT_NO_PARALLEL'] = 'no'
ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'

NODE_ROLES = ["server-0", "server-1", "server-2", "agent-0", "agent-1"]
NODE_BOXES = ['generic/ubuntu2004', 'generic/ubuntu2004', 'generic/ubuntu2004', 'generic/ubuntu2004', 'generic/ubuntu2004']
NODE_CPUS = 4
NODE_MEMORY = 4096
# Virtualbox >= 6.1.28 require `/etc/vbox/network.conf` for expanded private networks 
NETWORK_PREFIX = "192.168.1"

# VirtualBox ortamları için
#ENV["VAGRANT_EXPERIMENTAL"] = "disks"

required_plugins = [ "vagrant-libvirt" ]
plugins_to_install = required_plugins.select { |plugin| not Vagrant.has_plugin? plugin }
if not plugins_to_install.empty?
  puts "Installing plugins: #{plugins_to_install.join(' ')}"
  if system "vagrant plugin install #{plugins_to_install.join(' ')}"
    exec "vagrant #{ARGV.join(' ')}"
  else
    abort "Installation of one or more plugins has failed. Aborting."
  end
end

def provision(vm, role, node_num)
  vm.box = NODE_BOXES[node_num]
  vm.hostname = role
  # We use a private network because the default IPs are dynamically assigned 
  # during provisioning. This makes it impossible to know the server-0 IP when 
  # provisioning subsequent servers and agents. A private network allows us to
  # assign static IPs to each node, and thus provide a known IP for the API endpoint.
  node_ip = "#{NETWORK_PREFIX}.#{160+node_num}"
  # An expanded netmask is required to allow VM<-->VM communication, virtualbox defaults to /32
  #vm.network "private_network", ip: node_ip, netmask: "255.255.255.0"

  vm.network "public_network", ip: node_ip, netmask: "255.255.255.0", mode: "bridge", type: "bridge", dev: "bridge0"

  vm.provision "ansible", run: 'once' do |ansible|
    ansible.compatibility_mode = "2.0"
    ansible.playbook = "playbooks/site.yml"
    ansible.groups = {
      "server" => NODE_ROLES.grep(/^server/),
      "agent" => NODE_ROLES.grep(/^agent/),
      "k3s_cluster:children" => ["server", "agent"],
    }
    ansible.extra_vars = {
      k3s_version: "v1.29.5+k3s1",
      api_endpoint: "#{NETWORK_PREFIX}.160",
      token: "myvagrant",
      # Required to use the private network configured above
      extra_server_args: "--node-external-ip #{node_ip} --flannel-iface eth1", 
      extra_agent_args: "--node-external-ip #{node_ip} --flannel-iface eth1",
      # Optional, left as reference for ruby-ansible syntax
      # extra_service_envs: [ "NO_PROXY='localhost'" ],
      server_config_yaml: <<~YAML
        write-kubeconfig-mode: 644
        disable:
          - traefik
          - servicelb
      YAML
    }
  end
end

Vagrant.configure("2") do |config|
  # Default provider is libvirt, virtualbox is only provided as a backup
  config.vm.provider "libvirt" do |v|
    v.cpus = NODE_CPUS
    v.memory = NODE_MEMORY
  end
  config.vm.provider "virtualbox" do |v|
    v.cpus = NODE_CPUS
    v.memory = NODE_MEMORY
    v.linked_clone = true
  end
  
  NODE_ROLES.each_with_index do |name, i|
    config.vm.define name do |node|
      provision(node.vm, name, i)
    end
  end

end
