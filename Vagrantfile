# -*- mode: ruby -*-
# vi: set ft=ruby :

# Function to prompt for input with a default value
def prompt(message, default = '')
  print "#{message} [#{default}]: "
  input = STDIN.gets.chomp
  input.empty? ? default : input
end

# Get user inputs
CLUSTER_NAME = prompt("Enter cluster name", "k3s-dev")
GITHUB_REPO = prompt("Enter GitHub repository (org/repo)")
GITHUB_TOKEN = prompt("Enter GitHub token")

# VM configurations
VM_MEMORY = 4096
VM_CPUS = 2
DISK_SIZE = "30G"

Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  
  # Provider-specific configurations
  config.vm.provider :libvirt do |libvirt|
    libvirt.memory = VM_MEMORY
    libvirt.cpus = VM_CPUS
    libvirt.storage :file, :size => DISK_SIZE
    libvirt.graphics_type = "none"
  end

  # Master node configuration
  config.vm.define "#{CLUSTER_NAME}" do |master|
    master.vm.hostname = "#{CLUSTER_NAME}-master"
    
    # Network configuration
    master.vm.network "private_network", type: "dhcp"

    # Provisioning script
    master.vm.provision "shell", inline: <<-SHELL
      # Update system
      apt-get update
      apt-get upgrade -y

      # Install required packages
      apt-get install -y curl git

      # Install K3s
      curl -sfL https://get.k3s.io | sh -

      # Wait for K3s to be ready
      while ! kubectl get node 2>/dev/null; do
        echo "Waiting for K3s to be ready..."
        sleep 5
      done

      # Get kubeconfig and save it
      mkdir -p /vagrant
      cp /etc/rancher/k3s/k3s.yaml /vagrant/kubeconfig
      chmod 600 /vagrant/kubeconfig
      chown vagrant:vagrant /vagrant/kubeconfig
      export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
      echo "export KUBECONFIG=/home/vagrant/kubeconfig" >> /home/vagrant/.bashrc
      sed -i "s/127.0.0.1/$(hostname -I | awk '{print $1}')/g" /vagrant/kubeconfig
      
      # Install Flux
      curl -s https://fluxcd.io/install.sh | bash

      # Bootstrap Flux
      export GITHUB_TOKEN="#{GITHUB_TOKEN}"
      export GITHUB_USER=$(echo "#{GITHUB_REPO}" | cut -d '/' -f1)
      export GITHUB_REPO=$(echo "#{GITHUB_REPO}" | cut -d '/' -f2)
      # fix : apiextensions.k8s.io/v1: Get "http://localhost:8080/apis/apiextensions.k8s.io/v1": dial tcp 127.0.0.1:8080: connect: connection refused
      flux bootstrap github \
        --owner=$GITHUB_USER \
        --repository=$GITHUB_REPO \
        --branch=main \
        --path=clusters/#{CLUSTER_NAME} \
        --personal \
        --components-extra=image-reflector-controller,image-automation-controller

      echo "K3s cluster is ready!"
      echo "Kubeconfig is available at: /vagrant/kubeconfig"
      echo "To use kubectl externally, run: export KUBECONFIG=/path/to/kubeconfig"
    SHELL
  end

  # Disable default synced folder
  config.vm.synced_folder ".", "/vagrant", type: "rsync"
end
