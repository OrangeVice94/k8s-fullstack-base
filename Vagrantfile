# -*- mode: ruby -*-
# vi: set ft=ruby :

# Network: 192.168.56.0/24 (VirtualBox host-only)
NETWORK_PREFIX = "192.168.56"

MACHINES = [
  { name: "master1",  ip: "#{NETWORK_PREFIX}.10", ram: 4096, cpu: 4 },
  { name: "worker1",  ip: "#{NETWORK_PREFIX}.11", ram: 3072, cpu: 1 },
  { name: "worker2",  ip: "#{NETWORK_PREFIX}.12", ram: 3072, cpu: 1 },
  { name: "nginx1",   ip: "#{NETWORK_PREFIX}.20", ram: 1024, cpu: 1 },
  { name: "ansible1", ip: "#{NETWORK_PREFIX}.30", ram: 1024, cpu: 1 },
]

# Generate a shared SSH key pair for the ansible user (stored in .vagrant/)
KEY_DIR  = File.join(__dir__, ".vagrant", "ansible_ssh")
KEY_PATH = File.join(KEY_DIR, "id_rsa")
PUBKEY_PATH = File.join(KEY_DIR, "id_rsa.pub")

unless File.exist?(KEY_PATH)
  FileUtils.mkdir_p(KEY_DIR)
  system("ssh-keygen -t rsa -b 4096 -f \"#{KEY_PATH}\" -N \"\" -q")
end

PUBKEY = File.read("#{KEY_PATH}.pub").strip

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"

  # Disable default /vagrant sync (requires Guest Additions, not included in this box)
  config.vm.synced_folder ".", "/vagrant", disabled: true

  MACHINES.each do |machine|
    config.vm.define machine[:name] do |node|
      node.vm.hostname = machine[:name]
      node.vm.network "private_network", ip: machine[:ip]

      node.vm.provider "virtualbox" do |vb|
        vb.name   = machine[:name]
        vb.memory = machine[:ram]
        vb.cpus   = machine[:cpu]
        vb.linked_clone = true
      end

      # Create ansible user and inject SSH key on all VMs
      node.vm.provision "shell", inline: <<-SHELL
        # Create ansible user with sudo privileges
        if ! id "ansible" &>/dev/null; then
          useradd -m -s /bin/bash ansible
          echo "ansible ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible
          chmod 0440 /etc/sudoers.d/ansible
        fi

        # Configure SSH authorized key
        mkdir -p /home/ansible/.ssh
        echo "#{PUBKEY}" > /home/ansible/.ssh/authorized_keys
        chmod 700 /home/ansible/.ssh
        chmod 600 /home/ansible/.ssh/authorized_keys
        chown -R ansible:ansible /home/ansible/.ssh
      SHELL

      # ansible1: install Ansible, place private key, sync project
      if machine[:name] == "ansible1"
        # Copy the private key into ansible1
        node.vm.provision "file",
          source: KEY_PATH,
          destination: "/tmp/ansible_id_rsa"

        node.vm.provision "file",
          source: PUBKEY_PATH,
          destination: "/tmp/ansible_id_rsa.pub"

        node.vm.provision "shell", inline: <<-SHELL
          # Install Ansible
          apt-get update -qq
          apt-get install -y -qq ansible > /dev/null 2>&1

          # Place private key for ansible user
          mv /tmp/ansible_id_rsa /home/ansible/.ssh/id_rsa
          mv /tmp/ansible_id_rsa.pub /home/ansible/.ssh/id_rsa.pub
          
          chmod 600 /home/ansible/.ssh/id_rsa
          chown ansible:ansible /home/ansible/.ssh/id_rsa

          chmod 644 /home/ansible/.ssh/id_rsa.pub
          chown ansible:ansible /home/ansible/.ssh/id_rsa.pub

          # Add target hosts to known_hosts to avoid first-connect prompts
          su - ansible -c '
            for ip in 192.168.56.10 192.168.56.11 192.168.56.12 192.168.56.20; do
              ssh-keyscan -H "$ip" >> ~/.ssh/known_hosts 2>/dev/null
            done
          '
        SHELL

        # Sync project via rsync (no Guest Additions needed)
        node.vm.synced_folder ".", "/home/ansible/project",
          type: "rsync",
          rsync__exclude: [".git/", ".vagrant/", "Conversacion/"]

        node.vm.provision "shell", inline: <<-SHELL
          chown -R ansible:ansible /home/ansible/project
        SHELL
      end
    end
  end
end
