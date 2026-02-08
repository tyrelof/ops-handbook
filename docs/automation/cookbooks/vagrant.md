# Vagrant - Local Lab & Development Environment

Complete guide to using Vagrant for spinning up local multi-VM environments, provisioning with code, and testing infrastructure at scale.

## Table of Contents
1. [Installation](#installation)
2. [Vagrantfile Basics](#vagrantfile-basics)
3. [Box Management](#box-management)
4. [Networking](#networking)
5. [Provisioning](#provisioning)
6. [Ansible Integration](#ansible-integration)
7. [Advanced Patterns](#advanced-patterns)
8. [Troubleshooting](#troubleshooting)

---

## Installation

### Install Vagrant & VirtualBox

```bash
# Ubuntu 20.04
sudo apt update
sudo apt install vagrant virtualbox virtualbox-ext-pack

# Verify
vagrant --version
vboxmanage --version

# macOS
brew install vagrant virtualbox

# Windows
# Download from hashicorp.com and virtualbox.org
```

---

## Vagrantfile Basics

### Minimal Vagrantfile

Create `Vagrantfile` in project directory:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-20.04"
  config.vm.hostname = "lab-vm-01"
  
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus = 2
  end
  
  # Sync folder
  config.vm.synced_folder ".", "/vagrant", type: "virtualbox"
  
  # Provisioning
  config.vm.provision "shell", inline: "apt update && apt install -y git curl"
end
```

Commands:
```bash
vagrant up           # Create and boot VM
vagrant ssh          # SSH to VM
vagrant suspend      # Pause VM
vagrant resume       # Resume VM
vagrant halt         # Shutdown gracefully
vagrant destroy      # Delete VM
```

---

## Box Management

### Add/List Boxes

```bash
# Add box
vagrant box add bento/ubuntu-20.04

# List installed boxes
vagrant box list

# Remove box
vagrant box remove bento/ubuntu-20.04

# Update boxes
vagrant box update
```

### Popular Boxes

- `bento/ubuntu-20.04` - Ubuntu 20.04
- `bento/centos-7` - CentOS 7
- `generic/rhel8` - RHEL 8
- `hashicorp/bionic64` - Ubuntu 18.04

---

## Networking

### Private Network (Host-Only)

```ruby
config.vm.network "private_network", ip: "10.0.10.10"
```

Access from host at `10.0.10.10`; VMs can communicate on private network.

### Multiple Machines with Custom Network

```ruby
Vagrant.configure("2") do |config|
  
  (1..3).each do |i|
    config.vm.define "web#{i}" do |node|
      node.vm.box = "bento/ubuntu-20.04"
      node.vm.hostname = "web-#{i}"
      node.vm.network "private_network", ip: "10.0.10.#{10 + i}"
      
      node.vm.provider "virtualbox" do |vb|
        vb.memory = 1024
        vb.cpus = 1
      end
    end
  end
  
  config.vm.define "db" do |node|
    node.vm.box = "bento/ubuntu-20.04"
    node.vm.hostname = "db-01"
    node.vm.network "private_network", ip: "10.0.30.10"
    
    node.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
      vb.cpus = 2
    end
  end
  
end
```

Commands:
```bash
vagrant up                    # Create all VMs
vagrant up web1              # Create only web1
vagrant ssh web1             # SSH to specific VM
vagrant status               # List all VMs
```

---

## Provisioning

### Shell Provisioning

```ruby
config.vm.provision "shell", inline: <<-SHELL
  apt update
  apt install -y python3 python3-pip git curl
  pip3 install ansible
SHELL
```

Or from file:
```ruby
config.vm.provision "shell", path: "scripts/bootstrap.sh"
```

### File Provisioning

```ruby
config.vm.provision "file", source: "app/", destination: "/home/vagrant/app"
config.vm.provision "file", source: "./config.yml", destination: "/tmp/config.yml"
```

### Run Provisioning After Boot

```bash
vagrant provision          # Re-run all provisioners
vagrant provision web1     # Specific VM
vagrant reload --provision # Reboot and provision
```

---

## Ansible Integration

### Provision with Ansible

```ruby
Vagrant.configure("2") do |config|
  
  config.vm.define "web1" do |node|
    node.vm.box = "bento/ubuntu-20.04"
    node.vm.hostname = "web-01"
    node.vm.network "private_network", ip: "10.0.10.10"
  end
  
  config.vm.define "db" do |node|
    node.vm.box = "bento/ubuntu-20.04"
    node.vm.hostname = "db-01"
    node.vm.network "private_network", ip: "10.0.30.10"
  end
  
  # Run Ansible playbook after all VMs are created
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbooks/site.yml"
    ansible.inventory_path = "inventory/hosts"
    ansible.limit = "all"
    ansible.verbose = "v"
  end
  
end
```

Create `playbooks/site.yml`:
```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: Update packages
      apt:
        update_cache: yes
      when: ansible_os_family == "Debian"

- hosts: web
  become: yes
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
    - name: Start Nginx
      service:
        name: nginx
        state: started
        enabled: yes

- hosts: db
  become: yes
  tasks:
    - name: Install PostgreSQL
      apt:
        name: postgresql
        state: present
    - name: Start PostgreSQL
      service:
        name: postgresql
        state: started
        enabled: yes
```

Create `inventory/hosts`:
```
[web]
web1 ansible_host=10.0.10.10 ansible_user=vagrant ansible_private_key_file=.vagrant/machines/web1/virtualbox/private_key

[db]
db ansible_host=10.0.30.10 ansible_user=vagrant ansible_private_key_file=.vagrant/machines/db/virtualbox/private_key

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

---

## Advanced Patterns

### Template Vagrantfile (Multi-Environment)

```ruby
require 'yaml'

config_file = File.expand_path("../config.yml", __FILE__)
if File.exist?(config_file)
  config = YAML.load_file(config_file)
else
  config = { 'vms' => [], 'network' => {} }
end

Vagrant.configure("2") do |config|
  
  config['vms'].each do |vm|
    config.vm.define vm['name'] do |node|
      node.vm.box = vm['box']
      node.vm.hostname = vm['hostname']
      node.vm.network "private_network", ip: vm['ip']
      
      node.vm.provider "virtualbox" do |vb|
        vb.memory = vm['memory']
        vb.cpus = vm['cpus']
      end
    end
  end
  
end
```

Create `config.yml`:
```yaml
network:
  subnet: 10.0.10.0/24

vms:
  - name: web1
    box: bento/ubuntu-20.04
    hostname: web-01
    ip: 10.0.10.10
    memory: 1024
    cpus: 1
    
  - name: web2
    box: bento/ubuntu-20.04
    hostname: web-02
    ip: 10.0.10.11
    memory: 1024
    cpus: 1
    
  - name: db
    box: bento/ubuntu-20.04
    hostname: db-01
    ip: 10.0.30.10
    memory: 2048
    cpus: 2
```

### Snapshots (Save/Restore State)

```bash
# Save snapshot
vagrant snapshot save web1 "before-deploy"

# List snapshots
vagrant snapshot list

# Restore snapshot
vagrant snapshot restore web1 "before-deploy"

# Delete snapshot
vagrant snapshot delete web1 "before-deploy"
```

### Box Customization

```ruby
# Download box and customize
vagrant box add --name my-custom-box ./custom.box

# Share box among projects
# Set VAGRANT_HOME=/shared/vagrant
export VAGRANT_HOME=/shared/vagrant/data
vagrant up
```

---

## Troubleshooting

### VMs Won't Boot

```bash
# Check VirtualBox logs
cat ~/.vagrant.d/logs/vboxmanage.*

# Increase CPU/memory
# Edit Vagrantfile, increase vb.memory
vagrant reload

# Try different provider
vagrant up --provider=libvirt
```

### Network Issues

```bash
# Ping from host to VM
ping 10.0.10.10

# Ping from VM to host
vagrant ssh web1
ping 10.0.1.1  # host gateway (default for private networks)

# Check iptables/firewall on host
sudo iptables -L -n | grep 10.0.10
```

### Provisioning Failures

```bash
# Re-run provisioning with debug
vagrant provision web1 --debug

# SSH and check logs
vagrant ssh web1
tail -f /var/log/syslog
```

---

## Best Practices

1. **Version Control**: Commit Vagrantfile and playbooks; use .gitignore for .vagrant/
2. **Snapshots**: Save before major changes; easy rollback
3. **Performance**: Disable synced_folder if not needed; use rsync for large files
4. **Isolation**: Each project in separate directory; don't share Vagrant state
5. **Documentation**: Document network layout, roles, access details
6. **Testing**: Use same provisioning code for local + production
7. **Cleanup**: `vagrant destroy` to free disk; boxes can be large (1+ GB)

---

*Vagrant local lab setup guide for testing infrastructure, automation, and deployments.*
