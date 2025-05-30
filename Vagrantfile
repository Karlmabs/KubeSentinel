
require "yaml"
vagrant_root = File.dirname(File.expand_path(__FILE__))
settings = YAML.load_file "#{vagrant_root}/settings.yaml"

IP_SECTIONS = settings["network"]["control_ip"].match(/^([0-9.]+\.)([^.]+)$/)
# First 3 octets including the trailing dot:
IP_NW = IP_SECTIONS.captures[0]
# Last octet excluding all dots:
IP_START = Integer(IP_SECTIONS.captures[1])
NUM_WORKER_NODES = settings["nodes"]["workers"]["count"]

Vagrant.configure("2") do |config|
  config.vm.provision "shell", env: { "IP_NW" => IP_NW, "IP_START" => IP_START, "NUM_WORKER_NODES" => NUM_WORKER_NODES }, inline: <<-SHELL
      apt-get update -y
      echo "$IP_NW$((IP_START)) controlplane" >> /etc/hosts
      for i in `seq 1 ${NUM_WORKER_NODES}`; do
        echo "$IP_NW$((IP_START+i)) node0${i}" >> /etc/hosts
      done
  SHELL

  if `uname -m`.strip == "aarch64"
    config.vm.box = settings["software"]["box"] + "-arm64"
  else
    config.vm.box = settings["software"]["box"]
  end
  config.vm.box_check_update = true

  config.vm.define "controlplane" do |controlplane|
    controlplane.vm.hostname = "controlplane"
    controlplane.vm.network "private_network", ip: settings["network"]["control_ip"]
    if settings["shared_folders"]
      settings["shared_folders"].each do |shared_folder|
        controlplane.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
      end
    end
    controlplane.vm.provider "virtualbox" do |vb|
        vb.cpus = settings["nodes"]["control"]["cpu"]
        vb.memory = settings["nodes"]["control"]["memory"]
        if settings["cluster_name"] and settings["cluster_name"] != ""
          vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
        end
    end
    controlplane.vm.provision "ansible" do |ansible|
      ansible.playbook = "playbooks/common.yml"
      ansible.verbose = true
      ansible.extra_vars = {
        dns_servers: settings["network"]["dns_servers"].join(" "),
        environment_vars: settings["environment"],
        kubernetes_version: settings["software"]["kubernetes"],
        kubernetes_version_short: settings["software"]["kubernetes"][0..3],
        os_version: settings["software"]["os"]
      }
    end
    controlplane.vm.provision "ansible" do |ansible|
      ansible.playbook = "playbooks/master.yml"
      ansible.verbose = true
      ansible.extra_vars = {
        calico_version: settings["software"]["calico"],
        control_ip: settings["network"]["control_ip"],
        pod_cidr: settings["network"]["pod_cidr"],
        service_cidr: settings["network"]["service_cidr"]
      }
    end
  end

  (1..NUM_WORKER_NODES).each do |i|

    config.vm.define "node0#{i}" do |node|
      node.vm.hostname = "node0#{i}"
      node.vm.network "private_network", ip: IP_NW + "#{IP_START + i}"
      if settings["shared_folders"]
        settings["shared_folders"].each do |shared_folder|
          node.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
        end
      end
      node.vm.provider "virtualbox" do |vb|
          vb.cpus = settings["nodes"]["workers"]["cpu"]
          vb.memory = settings["nodes"]["workers"]["memory"]
          if settings["cluster_name"] and settings["cluster_name"] != ""
            vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
          end
      end
      node.vm.provision "ansible" do |ansible|
        ansible.playbook = "playbooks/common.yml"
        ansible.verbose = true
        ansible.extra_vars = {
          dns_servers: settings["network"]["dns_servers"].join(" "),
          environment_vars: settings["environment"],
          kubernetes_version: settings["software"]["kubernetes"],
          kubernetes_version_short: settings["software"]["kubernetes"][0..3],
          os_version: settings["software"]["os"]
        }
      end
      node.vm.provision "ansible" do |ansible|
        ansible.playbook = "playbooks/node.yml"
      end

      # Only install the dashboard after provisioning the last worker (and when enabled).
      if i == NUM_WORKER_NODES and settings["software"]["dashboard"] and settings["software"]["dashboard"] != ""
        node.vm.provision "ansible" do |ansible|
          ansible.playbook = "playbooks/dashboard.yml"
          ansible.extra_vars = {
            dashboard_version: settings["software"]["dashboard"]
          }
        end
      end
    end

  end

  # New monitoring node configuration
  config.vm.define "monitoring" do |monitoring|
    monitoring.vm.hostname = "monitoring"
    monitoring.vm.network "private_network", ip: IP_NW + "#{IP_START + NUM_WORKER_NODES + 1}"
    
    if settings["shared_folders"]
      settings["shared_folders"].each do |shared_folder|
        monitoring.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
      end
    end
    
    monitoring.vm.provider "virtualbox" do |vb|
      vb.cpus = 2  # Adjust as needed
      vb.memory = 2048  # Adjust as needed
      if settings["cluster_name"] and settings["cluster_name"] != ""
        vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
      end
    end

    # Apply Grafana configuration
    monitoring.vm.provision "ansible" do |ansible|
      ansible.playbook = "playbooks/grafana.yml"
      ansible.verbose = true
    end

  end

end 
