# Cookbook: Ansible Automation & Infrastructure as Code

> [!NOTE]
> Production Ansible automation for infrastructure at scale.
> Covers: Playbooks, roles, inventory, variables, handlers, and common use cases.

## 1. Ansible Fundamentals

### Installation
```bash
# Linux
pip install ansible
ansible --version

# Or via package manager
apt-get install ansible  # Debian/Ubuntu
yum install ansible      # RHEL/CentOS

# Verify installation
ansible --version
ansible-inventory --list -i inventory/
```

### Inventory File Structure
```yaml
# inventory/production.yml

all:
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
    ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
    
  children:
    webservers:
      hosts:
        web01:
          ansible_host: 192.168.1.10
          app_port: 8080
        web02:
          ansible_host: 192.168.1.11
          app_port: 8080
      vars:
        app_name: myapp
        
    databases:
      hosts:
        db01:
          ansible_host: 192.168.1.20
          db_port: 5432
      vars:
        db_name: production
        
    loadbalancers:
      hosts:
        lb01:
          ansible_host: 192.168.1.5
      vars:
        balance_method: round-robin
```

### Ansible Configuration
```ini
# ansible.cfg

[defaults]
inventory = inventory/production.yml
host_key_checking = False
remote_user = ubuntu
private_key_file = ~/.ssh/id_rsa
gather_facts = True
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

## 2. Basic Playbooks

### Simple Playbook
```yaml
---
- name: Deploy web server
  hosts: webservers
  become: yes
  
  tasks:
    - name: Update package lists
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"
    
    - name: Install packages
      apt:
        name:
          - nginx
          - python3-pip
          - git
        state: present
      when: ansible_os_family == "Debian"
    
    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
    
    - name: Verify service
      uri:
        url: http://localhost
        method: GET
      register: result
      until: result.status == 200
      retries: 3
      delay: 5
```

### Run Playbook
```bash
# Run on specific hosts
ansible-playbook deploy.yml -i inventory/ --limit webservers

# Run with variables
ansible-playbook deploy.yml -e "app_version=1.2.3"

# Run specific tags
ansible-playbook deploy.yml --tags "install,start"

# Dry-run (check mode)
ansible-playbook deploy.yml --check

# Verbose output
ansible-playbook deploy.yml -vvv

# Run on single host
ansible-playbook deploy.yml -i inventory/ --limit "web01"
```

## 3. Variables & Facts

### Playbook Variables
```yaml
---
- name: Use variables
  hosts: all
  
  vars:
    app_name: myapp
    app_version: "1.2.3"
    app_port: 8080
    environment: production
    
  vars_files:
    - vars/common.yml
    - vars/{{ environment }}.yml
  
  tasks:
    - name: Debug variables
      debug:
        msg: "Running {{ app_name }} version {{ app_version }} on port {{ app_port }}"
    
    - name: Use variable in task
      file:
        path: "/opt/{{ app_name }}"
        state: directory
    
    - name: Register variable from command
      shell: uname -a
      register: uname_output
    
    - name: Use registered variable
      debug:
        msg: "System info: {{ uname_output.stdout }}"
```

### Facts
```yaml
---
- name: Use facts
  hosts: all
  
  tasks:
    - name: Print hostname
      debug:
        msg: "Hostname is {{ ansible_hostname }}"
    
    - name: Print OS
      debug:
        msg: "OS is {{ ansible_os_family }} {{ ansible_lsb.major_release | default('unknown') }}"
    
    - name: Print memory
      debug:
        msg: "Memory available: {{ ansible_memtotal_mb }} MB"
    
    - name: Print IP addresses
      debug:
        msg: "IPs: {{ ansible_all_ipv4_addresses }}"
    
    - name: Conditional based on OS
      apt:
        name: nginx
      when: ansible_os_family == "Debian"
```

## 4. Roles

### Role Structure
```
roles/
├── webserver/
│   ├── defaults/
│   │   └── main.yml          # Default variables
│   ├── files/
│   │   └── app.conf          # Static files
│   ├── handlers/
│   │   └── main.yml          # Handlers (restart, reload)
│   ├── meta/
│   │   └── main.yml          # Role metadata, dependencies
│   ├── tasks/
│   │   └── main.yml          # Tasks
│   ├── templates/
│   │   └── nginx.conf.j2     # Jinja2 templates
│   └── vars/
│       └── main.yml          # Role variables
└── database/
    ├── defaults/
    ├── files/
    ├── handlers/
    ├── tasks/
    ├── templates/
    └── vars/
```

### Create Role
```bash
ansible-galaxy init roles/webserver
```

### Role Implementation
```yaml
# roles/webserver/tasks/main.yml
---
- name: Install nginx
  apt:
    name: nginx
    state: present

- name: Deploy config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: restart nginx

- name: Start service
  service:
    name: nginx
    state: started
    enabled: yes
```

```yaml
# roles/webserver/handlers/main.yml
---
- name: restart nginx
  service:
    name: nginx
    state: restarted
```

```yaml
# roles/webserver/defaults/main.yml
---
nginx_port: 80
nginx_user: www-data
nginx_workers: 4
```

```jinja2
# roles/webserver/templates/nginx.conf.j2
user {{ nginx_user }};
worker_processes {{ nginx_workers }};

http {
  server {
    listen {{ nginx_port }};
    server_name {{ inventory_hostname }};
    
    location / {
      proxy_pass http://localhost:{{ app_port }};
    }
  }
}
```

### Use Role in Playbook
```yaml
---
- name: Deploy application stack
  hosts: webservers
  
  roles:
    - role: webserver
      vars:
        nginx_port: 8080
    - role: monitoring-agent
    - role: security-hardening
```

## 5. Control Structures

### Loops
```yaml
---
- name: Use loops
  hosts: all
  
  tasks:
    - name: Create users
      user:
        name: "{{ item.name }}"
        shell: "{{ item.shell }}"
      loop:
        - { name: 'john', shell: '/bin/bash' }
        - { name: 'jane', shell: '/bin/bash' }
        - { name: 'bob', shell: '/bin/sh' }
    
    - name: Install packages
      apt:
        name: "{{ item }}"
      loop:
        - nginx
        - python3-pip
        - git
    
    - name: Create directories
      file:
        path: "/opt/{{ item }}"
        state: directory
      loop: "{{ app_names }}"  # From variables
```

### Conditionals
```yaml
---
- name: Conditional tasks
  hosts: all
  
  tasks:
    - name: Install nginx on Debian
      apt:
        name: nginx
      when: ansible_os_family == "Debian"
    
    - name: Install nginx on RedHat
      yum:
        name: nginx
      when: ansible_os_family == "RedHat"
    
    - name: Start service if installed
      service:
        name: nginx
        state: started
      when:
        - ansible_os_family == "Debian"
        - nginx_installed | default(false)
    
    - name: Multiple conditions
      debug:
        msg: "Deploying to production"
      when:
        - environment == "production"
        - deployment_approved | default(false)
```

### Handlers
```yaml
---
- name: Configuration management
  hosts: all
  
  tasks:
    - name: Update config file
      template:
        src: app.conf.j2
        dest: /etc/app/config
      notify:
        - restart app
        - log restart
    
    - name: Install package
      apt:
        name: app
      notify: restart app
  
  handlers:
    - name: restart app
      service:
        name: app
        state: restarted
    
    - name: log restart
      shell: echo "App restarted at $(date)" >> /var/log/deployments.log
```

## 6. Common Modules

### File Management
```yaml
---
- name: File operations
  hosts: all
  
  tasks:
    # Create directory
    - name: Create app directory
      file:
        path: /opt/app
        state: directory
        owner: app
        group: app
        mode: '0755'
    
    # Copy file
    - name: Copy config
      copy:
        src: files/app.conf
        dest: /etc/app/config
        owner: root
        group: root
        mode: '0644'
    
    # Template (Jinja2)
    - name: Deploy config from template
      template:
        src: app.conf.j2
        dest: /etc/app/config
        backup: yes
    
    # Delete file
    - name: Remove old cache
      file:
        path: /tmp/app-cache
        state: absent
    
    # Create symlink
    - name: Create symlink
      file:
        src: /opt/app/current
        dest: /var/www/app
        state: link
```

### Package Management
```yaml
---
- name: Package operations
  hosts: all
  
  tasks:
    - name: Update all packages
      apt:
        update_cache: yes
        upgrade: dist
      when: ansible_os_family == "Debian"
    
    - name: Install specific version
      apt:
        name: nginx=1.18*
        state: present
    
    - name: Install multiple packages
      apt:
        name:
          - git
          - curl
          - vim
        state: present
    
    - name: Remove package
      apt:
        name: apache2
        state: absent
    
    - name: Ensure latest version
      apt:
        name: nodejs
        state: latest
```

### Service Management
```yaml
---
- name: Service operations
  hosts: all
  
  tasks:
    - name: Start service
      service:
        name: nginx
        state: started
        enabled: yes  # Start on boot
    
    - name: Restart service
      service:
        name: nginx
        state: restarted
    
    - name: Stop service
      service:
        name: nginx
        state: stopped
    
    - name: Check if service is running
      service:
        name: nginx
        state: started
      register: service_status
      ignore_errors: yes
```

### User Management
```yaml
---
- name: User operations
  hosts: all
  become: yes
  
  tasks:
    - name: Create user
      user:
        name: appuser
        shell: /bin/bash
        home: /home/appuser
        groups: sudo
        createhome: yes
    
    - name: Add SSH key
      authorized_key:
        user: appuser
        key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
        state: present
    
    - name: Delete user
      user:
        name: olduser
        state: absent
        remove: yes  # Remove home directory
```

## 7. Advanced Patterns

### Idempotent Deployment
```yaml
---
- name: Deploy application
  hosts: webservers
  
  tasks:
    - name: Check if app directory exists
      stat:
        path: /opt/app
      register: app_dir
    
    - name: Clone repository if not exists
      git:
        repo: https://github.com/example/app.git
        dest: /opt/app
        version: v{{ app_version }}
      when: not app_dir.stat.exists
    
    - name: Update if exists
      git:
        repo: https://github.com/example/app.git
        dest: /opt/app
        update: yes
        version: v{{ app_version }}
      when: app_dir.stat.exists
    
    - name: Install dependencies
      shell: cd /opt/app && npm install
      changed_when: false  # Don't mark as changed if re-run
    
    - name: Start application
      shell: cd /opt/app && npm start
      async: 300  # Run in background, up to 5 min
      poll: 0    # Don't wait for completion
```

### Rolling Deployment
```yaml
---
- name: Rolling deployment
  hosts: webservers
  serial: 1  # One host at a time
  
  pre_tasks:
    - name: Deregister from load balancer
      shell: curl -X POST http://lb.example.com/deregister/{{ inventory_hostname }}
  
  tasks:
    - name: Pull latest code
      git:
        repo: https://github.com/example/app.git
        dest: /opt/app
        version: main
    
    - name: Restart application
      service:
        name: app
        state: restarted
    
    - name: Wait for service to be healthy
      uri:
        url: http://localhost:8080/health
        method: GET
      register: result
      until: result.status == 200
      retries: 5
      delay: 10
  
  post_tasks:
    - name: Register with load balancer
      shell: curl -X POST http://lb.example.com/register/{{ inventory_hostname }}
```

## 8. Testing & Validation

### Syntax Check
```bash
# Validate playbook syntax
ansible-playbook deploy.yml --syntax-check

# Validate inventory
ansible-inventory -i inventory/ --list
```

### Dry Run
```bash
# Check what would change
ansible-playbook deploy.yml --check

# Check with diff
ansible-playbook deploy.yml --check --diff
```

### Idempotent Testing
```bash
# Run twice - should see no changes on second run
ansible-playbook deploy.yml
ansible-playbook deploy.yml  # Should show 0 changed

# If not idempotent, tasks will show "changed"
```

## 9. Common Use Cases

### Security Hardening
```yaml
---
- name: Harden server
  hosts: all
  become: yes
  
  tasks:
    - name: Update system
      apt:
        update_cache: yes
        upgrade: dist
    
    - name: Configure firewall
      ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop: [22, 80, 443]
    
    - name: Disable SSH password auth
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PasswordAuthentication'
        line: 'PasswordAuthentication no'
      notify: restart sshd
    
    - name: Install fail2ban
      apt:
        name: fail2ban
        state: present
    
    - name: Enable automatic security updates
      apt:
        name: unattended-upgrades
        state: present
  
  handlers:
    - name: restart sshd
      service:
        name: sshd
        state: restarted
```

### Monitoring Stack Deployment
```yaml
---
- name: Deploy Prometheus + Grafana
  hosts: monitoring
  
  roles:
    - prometheus
    - grafana
    - alertmanager
  
  post_tasks:
    - name: Configure Prometheus scrape targets
      template:
        src: prometheus.yml.j2
        dest: /etc/prometheus/prometheus.yml
      notify: restart prometheus
```

## 10. Best Practices

| Practice | Implementation |
|----------|-----------------|
| **Idempotency** | Tasks should be safe to run multiple times |
| **Error handling** | Use `ignore_errors`, `failed_when`, `rescue` |
| **Validation** | Use `--check` before production runs |
| **Versioning** | Pin versions in playbooks |
| **Notifications** | Use handlers for conditional actions |
| **Modularization** | Use roles for reusability |
| **Documentation** | Add comments and descriptions |
| **Secrets** | Use ansible-vault for passwords |

---

**Essential Commands**:
```bash
ansible-playbook deploy.yml                    # Run playbook
ansible-playbook deploy.yml --check            # Dry-run
ansible-playbook deploy.yml --tags "install"   # Run specific tags
ansible all -m ping                            # Test connectivity
ansible-vault create secrets.yml               # Encrypt secrets
ansible-galaxy init roles/myrole               # Create role
```

**This foundation enables infrastructure as code at enterprise scale.**
