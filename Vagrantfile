# -*- mode: ruby -*-
# vi: set ft=ruby :

prefix = "cluster"

# this has to be same network as lmu vzd ansible
base_network = "172.16.0"

manager_group = "manager"

servers = {
  manager_group => {
    "instances" => 1,
    "cpus" => 2,
    "memory" => 4096,
    "ipoffset" => 20
  },
  "agent" => {
    "instances" => 2,
    "cpus" => 2,
    "memory" => 2048,
    "ipoffset" => 30
  }
}

all_vars = {
  "service_user" => "{{ ansible_user }}",
}

Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.
  # General Provider Settings

  # Use SSH-Key of host machine
  config.ssh.forward_agent = true

  # Disable synced folder
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.box = "generic/debian12"

  config.vm.provider "virtualbox" do |vb|
    # Disable GUI --> No GUI Window will pop up on start, could be opened on virtual Box app via double click on machine.
    vb.gui = false

    # special VirtualBox Customizations:
    vb.customize ["modifyvm", :id,
                  "--nicpromisc1", "allow-all",
                  "--nicpromisc2", "allow-all",
                  #"--cpuexecutioncap", "50",
                  # Make a Structure within Virtualbox to group Machines
                  "--groups", "/Vagrant/K3s"
                 ]
  end

  config.vm.provider "libvirt" do |lv|
    lv.cpu_mode = 'host-model'
    lv.cpu_model = 'qemu64'
  end

  host_vars = {}

  servers.each do |groupname, group|
    group["instances"].times { |i|
      name = "#{prefix}-#{groupname}-#{i}"
      config.vm.define name, primary: false, autostart: true do |node|

        node.vm.hostname = name
        #node.vm.box = "debian/bullseye64"
        #node.vm.box = "generic/debian11"

        node.vm.provider "virtualbox" do |vb|
          vb.name = name
          vb.cpus = group["cpus"]
          vb.memory = group["memory"]
        end

        node.vm.provider "libvirt" do |lv|
          lv.title = name
          lv.cpus = group["cpus"]
          lv.memory = group["memory"]
        end

        ipaddr = "#{base_network}.#{group["ipoffset"] + i}"
        #puts "assigning private address #{ipaddr} to #{name}"
        node.vm.network "private_network", ip: ipaddr, netmask: "255.255.255.0"
        host_vars[name] = {"main_ip" => ipaddr}
        if groupname != "quorum"
          host_vars[name]["advertised_ip"] = ipaddr
        end

        if groupname == manager_group and i == 0
          #puts "Port forwarding for #{name} try to forward ports 9092/30777/30779. (#{groupname} + #{i})"
          node.vm.network :forwarded_port, guest:6443, host: 6443
          node.vm.network :forwarded_port, guest_ip: "172.16.0.20", guest:8080, host: 8080
          node.vm.network :forwarded_port, guest_ip: "172.16.0.20", guest:3000, host: 3000
        end
      end
    }
  end

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "vagrant.yml"
    ansible.host_vars = host_vars
    # ansible.limit = "all"

    # verbosity
    # ansible.verbose = "v"
    # ansible.verbose = "vv"
    # ansible.verbose = "vvv"

    # create inventory (like hosts.ini)
    groups = {
      "all" => [],
      "all:vars" => all_vars
    }
    servers.each do |groupname, group|
      # puts "groupname: #{groupname}"
      # set initial_manager
      if groupname == manager_group
        groups["all:vars"]["initial_manager"] = "#{base_network}.#{group["ipoffset"]}"
      end
      # fill inventory
      groupdata = []
      groups[groupname] = groupdata
      group["instances"].times { |i|
        servername = "#{prefix}-#{groupname}-#{i}"
        groupdata.append(servername)
        groups["all"].append(servername)
      }
    end
    ansible.groups = groups
    # puts ansible.groups
  end
end
