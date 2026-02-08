# Chef - Infrastructure as Code & Configuration Management

Complete guide to using Chef for infrastructure automation, node management, and configuration deployment at scale.

## Table of Contents

1. [Chef Fundamentals](#chef-fundamentals)
2. [Workstation Setup](#workstation-setup)
3. [Chef Server](#chef-server)
4. [Cookbook Structure](#cookbook-structure)
5. [Recipes & Resources](#recipes--resources)
6. [Attributes & Variables](#attributes--variables)
7. [Templates & Configuration Files](#templates--configuration-files)
8. [Roles & Environments](#roles--environments)
9. [Data Bags](#data-bags)
10. [Testing Cookbooks](#testing-cookbooks)
11. [Node Convergence](#node-convergence)
12. [Advanced Patterns](#advanced-patterns)
13. [Troubleshooting](#troubleshooting)

---

## Chef Fundamentals

Chef is a declarative infrastructure automation platform using Ruby-based configuration files (cookbooks) to define and manage system state.

### Key Concepts

**Cookbook**: Collection of recipes, templates, attributes for a specific application/service
**Recipe**: Sequence of resources defining desired state
**Resource**: Atomic unit of configuration (package, service, file, etc.)
**Node**: Managed machine (server, container, cloud instance)
**Attribute**: Variable defining system configuration
**Data Bag**: Encrypted JSON data store for secrets/configs

### Installation

```bash
# Install Chef Workstation (includes knife, chef-cli, Test Kitchen, Cookstyle)
curl https://omnitruck.chef.io/install.sh | sudo bash -s -- -c workstation -v 23.10

# Verify installation
chef --version
knife --version

# On managed nodes, install Chef Client
curl https://omnitruck.chef.io/install.sh | sudo bash -s -- -c client -v 18.0
```

---

## Workstation Setup

### Directory Structure

```bash
# Create Chef workspace
mkdir -p ~/chef-workspace/cookbooks ~/chef-workspace/roles ~/chef-workspace/data_bags

# Generate cookbook
cd ~/chef-workspace
chef generate cookbook cookbooks/web_server
```

### Knife Configuration

```bash
# Create .chef/knife.rb
knife configure init

# knife.rb example
current_dir = File.dirname(__FILE__)
log_level :info
log_location STDOUT
node_name 'admin'
client_key "#{current_dir}/admin.pem"
chef_server_url 'https://chef-server.example.com/organizations/myorg'
cookbook_path ["#{current_dir}/../cookbooks"]
```

### Connecting to Chef Server

```bash
# Download validator key from Chef Server
knife bootstrap NODE_IP \
  --node-name web-server-01 \
  --user ubuntu \
  --identity-file ~/.ssh/id_rsa \
  --sudo

# List nodes
knife node list

# Show node details
knife node show web-server-01 -F json
```

---

## Chef Server

Central hub for managing cookbooks, nodes, and policies.

### Chef Server Setup (Self-Hosted)

```bash
# Install Chef Server
wget https://packages.chef.io/files/stable/chef-server/15.3.1/ubuntu/20.04/chef-server-core_15.3.1-1_amd64.deb
sudo dpkg -i chef-server-core_*.deb
sudo chef-server-ctl reconfigure

# Create organization
sudo chef-server-ctl org-create myorg "My Organization" --filename myorg-validator.pem

# Create admin user
sudo chef-server-ctl user-create admin Admin User admin@example.com \
  --filename admin.pem --password SecurePass123
```

### Upload Cookbooks to Server

```bash
# From workstation directory
knife cookbook upload web_server -o cookbooks/

# Upload all cookbooks
knife cookbook upload -a -o cookbooks/

# List remote cookbooks
knife cookbook list
```

---

## Cookbook Structure

### Basic Layout

```
web_server/
├── recipes/
│   ├── default.rb          # Main recipe
│   ├── install.rb
│   ├── configure.rb
│   └── deploy.rb
├── attributes/
│   ├── default.rb
│   └── web_server.rb
├── templates/
│   ├── nginx.conf.erb
│   └── app_config.yml.erb
├── files/
│   ├── ssl_cert.pem
│   └── app_data.json
├── test/
│   └── integration/
│       └── default/
│           └── default_test.rb
├── metadata.rb
├── README.md
└── Berksfile
```

### Metadata.rb

```ruby
name 'web_server'
description 'Installs and configures Nginx web server'
version '1.0.0'
chef_version '>= 15.0'
supports 'ubuntu', '~> 20.04'
supports 'centos', '~> 7.0'

depends 'apt', '~> 7.4.0'
depends 'ark', '~> 5.2.0'
depends 'poise-python', '~> 1.7.0'

maintainer 'DevOps Team'
maintainer_email 'devops@example.com'
license 'Apache-2.0'
```

---

## Recipes & Resources

### Basic Recipe Structure

```ruby
# recipes/default.rb
# Update system packages
apt_update 'system' do
  action :update
end

# Install Nginx
package 'nginx' do
  action :install
end

# Start and enable Nginx service
service 'nginx' do
  action [:enable, :start]
end

# Create document root
directory '/var/www/html' do
  owner 'www-data'
  group 'www-data'
  mode '0755'
  recursive true
end
```

### Common Resources

#### Package Management
```ruby
# Install packages
package ['curl', 'wget', 'git'] do
  action :install
end

# Use apt provider explicitly
apt_package 'nodejs' do
  version '16.13.0-1nodesource1'
  action :install
end

# Remove package
package 'apache2' do
  action :remove
end
```

#### File Operations
```ruby
# Create file with content
file '/etc/app/config.conf' do
  content 'key=value\nother=data'
  owner 'root'
  group 'root'
  mode '0644'
  action :create
end

# Copy file from cookbook
cookbook_file '/etc/rsyslog.d/app.conf' do
  source 'rsyslog-app.conf'
  owner 'root'
  group 'root'
  mode '0644'
  action :create
end

# Create directory
directory '/opt/app' do
  owner 'app'
  group 'app'
  mode '0755'
  action :create
end
```

#### Service Management
```ruby
# Enable and start service
service 'postgresql' do
  action [:enable, :start]
end

# Restart on config change (notifies)
service 'nginx' do
  action :nothing
  subscribes :restart, 'template[/etc/nginx/nginx.conf]', :delayed
end

# Subscribe to multiple resources
service 'app' do
  action :restart
  subscribes :restart, ['file[/etc/app/config.yml]', 'package[ruby]'], :delayed
end
```

#### User & Group Management
```ruby
# Create group
group 'app-users' do
  members ['deploy', 'jenkins']
  append true
end

# Create user
user 'deploy' do
  comment 'Deploy User'
  uid 1001
  home '/home/deploy'
  shell '/bin/bash'
  manage_home true
  password_hash '$1$...'  # Shadow password hash
end

# Add user to group
group 'docker' do
  members ['ubuntu']
  append true
end
```

#### Execute Custom Commands
```ruby
# Run shell command
execute 'configure app' do
  command 'python /opt/app/setup.py'
  cwd '/opt/app'
  user 'app'
  group 'app'
  action :run
  only_if { ::File.exist?('/opt/app/setup.py') }
end

# Run only if condition is true
bash 'compile from source' do
  code <<-BASH
    cd /tmp/src
    ./configure --prefix=/opt/app
    make
    make install
  BASH
  not_if { ::File.exist?('/opt/app/bin/myapp') }
end
```

---

## Attributes & Variables

### Attribute Hierarchy (Applied Order)
1. default
2. env_default
3. role_default
4. recipe
5. env_override
6. role_override
7. override

### Default Attributes

```ruby
# attributes/default.rb
default['nginx']['version'] = '1.24.0'
default['nginx']['workers'] = node['cpu']['total']
default['nginx']['worker_connections'] = 1024
default['nginx']['log_level'] = 'warn'
default['nginx']['keepalive_timeout'] = 65
default['app']['port'] = 8080
default['app']['ruby_version'] = '3.0'

# Nested attributes
default['db']['postgresql']['version'] = '14'
default['db']['postgresql']['max_connections'] = 200
default['db']['postgresql']['shared_buffers'] = '256MB'
```

### Using Attributes in Recipes

```ruby
# recipes/default.rb
package "nginx-#{node['nginx']['version']}" do
  action :install
end

template '/etc/nginx/nginx.conf' do
  source 'nginx.conf.erb'
  variables(
    worker_processes: node['cpu']['total'],
    worker_connections: node['nginx']['worker_connections'],
    keepalive: node['nginx']['keepalive_timeout']
  )
  notifies :restart, 'service[nginx]', :delayed
end

# Conditional based on attributes
if node['app']['environment'] == 'production'
  node.override['nginx']['worker_connections'] = 2048
end
```

### Accessing Node Data

```ruby
# System facts
node['hostname']           # hostname
node['fqdn']               # fully qualified domain name
node['ipaddress']          # primary IP
node['cpu']['total']       # CPU count
node['memory']['total']    # Total RAM
node['platform']           # os family (debian, rhel)
node['platform_version']   # os version

# In recipe
Chef::Log.info("Running on #{node['platform']} #{node['platform_version']}")

if node['platform_family'] == 'rhel'
  package 'nginx'
elsif node['platform_family'] == 'debian'
  apt_package 'nginx'
end
```

---

## Templates & Configuration Files

### ERB Template Creation

```erb
# templates/nginx.conf.erb
user www-data;
worker_processes <%= @worker_processes %>;
error_log /var/log/nginx/error.log <%= @log_level %>;

events {
    worker_connections <%= @worker_connections %>;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout <%= @keepalive %>s;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    access_log /var/log/nginx/access.log;

    upstream backend {
    <% @backend_servers.each do |server| %>
        server <%= server['ip'] %>:<%= server['port'] %>;
    <% end %>
    }

    server {
        listen 80;
        server_name <%= @server_name %>;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

### Using Templates in Recipes

```ruby
# recipes/configure.rb
template '/etc/nginx/nginx.conf' do
  source 'nginx.conf.erb'
  owner 'root'
  group 'root'
  mode '0644'
  variables(
    worker_processes: node['cpu']['total'],
    worker_connections: node['nginx']['worker_connections'],
    log_level: node['nginx']['log_level'],
    keepalive: node['nginx']['keepalive_timeout'],
    server_name: node['fqdn'],
    backend_servers: [
      { 'ip' => '10.0.1.10', 'port' => 8080 },
      { 'ip' => '10.0.1.11', 'port' => 8080 }
    ]
  )
  notifies :restart, 'service[nginx]', :delayed
end
```

---

## Roles & Environments

### Roles (Reusable Server Configurations)

```ruby
# roles/web_server.rb
name 'web_server'
description 'Web server role'

run_list(
  'recipe[base]',
  'recipe[nginx]',
  'recipe[nodejs]',
  'recipe[app]'
)

override_attributes(
  'nginx' => {
    'workers' => 8,
    'keepalive_timeout' => 90
  },
  'nodejs' => {
    'version' => '18.0.0'
  },
  'app' => {
    'environment' => 'production'
  }
)
```

### Creating Roles on Chef Server

```bash
# Upload role
knife role from file roles/web_server.rb

# List roles
knife role list

# Show role
knife role show web_server
```

### Environments (Stage Separation)

```ruby
# environments/production.rb
name 'production'
description 'Production environment'

override_attributes(
  'nginx' => {
    'worker_connections' => 2048,
    'log_level' => 'warn'
  },
  'db' => {
    'host' => 'prod-db.example.com',
    'replica_hosts' => ['prod-db-replica-01.example.com']
  },
  'app' => {
    'debug' => false,
    'log_level' => 'error'
  }
)

# environments/staging.rb
name 'staging'
description 'Staging environment'

override_attributes(
  'nginx' => {
    'worker_connections' => 512,
    'log_level' => 'info'
  },
  'db' => {
    'host' => 'staging-db.example.com'
  },
  'app' => {
    'debug' => true,
    'log_level' => 'debug'
  }
)
```

### Uploading to Server

```bash
knife environment from file environments/production.rb
knife environment list
knife environment show production
```

---

## Data Bags

Encrypted JSON storage for sensitive data (passwords, API keys, SSL certs).

### Create Data Bag

```bash
# Create data bag directory
mkdir -p data_bags/secrets

# Create data bag item (JSON)
cat > data_bags/secrets/database.json << 'EOF'
{
  "id": "database",
  "host": "prod-db.example.com",
  "username": "app_user",
  "password": "SuperSecretPassword123!",
  "port": 5432
}
EOF

# Upload to server
knife data bag create secrets
knife data bag from file secrets data_bags/secrets/database.json
```

### Using Data Bags in Recipes

```ruby
# recipes/configure.rb
# Load data bag item
db_config = data_bag_item('secrets', 'database')

template '/etc/app/database.yml' do
  source 'database.yml.erb'
  variables(
    host: db_config['host'],
    username: db_config['username'],
    password: db_config['password'],
    port: db_config['port']
  )
  notifies :restart, 'service[app]', :delayed
end
```

### Encrypted Data Bags

```bash
# Create encrypted data bag with key
knife data bag create --secret-file /etc/chef/encrypted_data_bag_secret secrets prod_api_keys

# In recipe, specify secret location
Chef::Config[:encrypted_data_bag_secret] = '/etc/chef/encrypted_data_bag_secret'
api_keys = data_bag_item('secrets', 'prod_api_keys')
```

---

## Testing Cookbooks

### ChefSpec (Unit Tests)

```ruby
# test/unit/spec/recipes/default_spec.rb
require 'spec_helper'

describe 'web_server::default' do
  platform 'ubuntu', '20.04'

  context 'When all attributes are default' do
    let(:chef_run) do
      ChefSpec::ServerRunner.new(platform: 'ubuntu', '20.04')
        .converge(described_recipe)
    end

    it 'converges successfully' do
      expect { chef_run }.not_to raise_error
    end

    it 'installs nginx package' do
      expect(chef_run).to install_package('nginx')
    end

    it 'starts nginx service' do
      expect(chef_run).to start_service('nginx')
    end

    it 'enables nginx service' do
      expect(chef_run).to enable_service('nginx')
    end

    it 'creates nginx config file' do
      expect(chef_run).to create_template('/etc/nginx/nginx.conf')
        .with(owner: 'root', group: 'root', mode: '0644')
    end
  end
end
```

### InSpec (Integration Tests)

```ruby
# test/integration/default/default_test.rb
describe package('nginx') do
  it { should be_installed }
end

describe service('nginx') do
  it { should be_installed }
  it { should be_enabled }
  it { should be_running }
end

describe file('/etc/nginx/nginx.conf') do
  it { should exist }
  its('owner') { should eq 'root' }
  its('mode') { should cmp '0644' }
end

describe port(80) do
  it { should be_listening }
end

describe http('http://localhost') do
  its('status') { should eq 200 }
end
```

### Run Tests

```bash
# Unit tests (ChefSpec)
rspec test/unit/recipes/default_spec.rb

# Integration tests (Test Kitchen)
kitchen test

# Syntax check
cookstyle recipes/default.rb
```

---

## Node Convergence

### Applying Cookbooks to Nodes

```bash
# Add recipe to node run_list
knife node run_list add web-server-01 "recipe[nginx]"

# Add multiple recipes
knife node run_list add web-server-01 "recipe[base]" "recipe[nginx]" "recipe[app]"

# Add role to node
knife node run_list add web-server-01 "role[web_server]"

# View node run list
knife node run_list show web-server-01

# Trigger convergence on node
knife ssh "name:web-server-01" "sudo chef-client" -x ubuntu

# Converge all nodes (use with caution!)
knife ssh "*:*" "sudo chef-client" -x ubuntu
```

### Chef Client Modes

```bash
# Manual convergence (default)
sudo chef-client

# Converge specific recipe
sudo chef-client -r "recipe[nginx::install]"

# Dry run (no changes)
sudo chef-client --why-run

# Set log level
sudo chef-client -l debug

# Run every 30 minutes (daemon mode)
sudo chef-client -i 1800 -d

# Override attribute
sudo chef-client -o "nginx.workers=16"
```

---

## Advanced Patterns

### Conditional Recipes

```ruby
# recipes/default.rb
# Execute based on platform
if platform_family?('rhel')
  include_recipe 'web_server::redhat'
elsif platform_family?('debian')
  include_recipe 'web_server::debian'
end

# Execute based on attribute
if node['app']['environment'] == 'production'
  include_recipe 'web_server::production'
end

# Execute based on node property
if node['memory']['total'].to_i > 8000
  node.override['nginx']['worker_connections'] = 2048
end
```

### Notifications & Subscriptions

```ruby
# File notifies service to restart
template '/etc/app/config.yml' do
  source 'config.yml.erb'
  notifies :restart, 'service[app]'
end

# Service subscribes to multiple files
service 'app' do
  action :nothing
  subscribes :restart, ['template[/etc/app/config.yml]', 'package[ruby]'], :delayed
end

# Immediate vs Delayed notifications
service 'nginx' do
  notifies :restart, 'service[php-fpm]', :immediately  # Restart immediately
  notifies :run, 'execute[rebuild-cache]', :delayed    # Run at end of Chef run
end
```

### Guards (Only If / Not If)

```ruby
# Execute only if condition is true
execute 'compile from source' do
  command './configure && make && make install'
  cwd '/tmp/build'
  only_if { ::File.exist?('/tmp/build/configure') }
  only_if { node['app']['build_from_source'] }
end

# Execute unless condition is true
package 'nodejs' do
  action :install
  not_if { ::File.exist?('/usr/bin/node') }
end

# Ruby block guard
bash 'setup database' do
  code 'psql -c "CREATE DATABASE myapp;"'
  not_if do
    shell_out('psql -l').stdout.match?('myapp')
  end
end
```

### Looping & Iteration

```ruby
# Loop over array
['vim', 'curl', 'wget', 'git'].each do |pkg|
  package pkg do
    action :install
  end
end

# Loop over attributes
node['users'].each do |name, data|
  user name do
    uid data['uid']
    home "/home/#{name}"
    shell '/bin/bash'
  end
end

# Loop over hash
node['mounts'].each do |mount_point, config|
  mount mount_point do
    device config['device']
    fstype config['fstype']
    action [:mount, :enable]
  end
end
```

---

## Troubleshooting

### Common Issues & Solutions

#### Node Won't Converge
```bash
# Check chef-client logs
sudo tail -f /var/log/chef/client.log

# Run in debug mode
sudo chef-client -l debug

# Check node attributes
knife node show web-server-01 -a

# Verify cookbooks are on server
knife cookbook list
```

#### Template Rendering Errors
```erb
# ERB template issues - Check variable names
Worker Processes: <%= @worker_processes %>
<!-- not @workers -->

# Check for undefined variables
<% if @optional_var %>
  <%= @optional_var %>
<% end %>
```

#### Resource Idempotency Issues
```ruby
# Always make resources idempotent
# ❌ BAD - Runs every convergence
bash 'update config' do
  code 'sed -i "s/old/new/" /etc/config.conf'
end

# ✅ GOOD - Only runs if needed
template '/etc/config.conf' do
  source 'config.conf.erb'
  action :create
end
```

#### Data Bag Access Issues
```bash
# Verify data bag exists on server
knife data bag list
knife data bag show secrets

# Check secret key permissions
sudo ls -la /etc/chef/encrypted_data_bag_secret
# Should be readable by chef-client user
```

### Debugging Commands

```bash
# Knife debugging
knife node show web-server-01 -F json | jq '.override_attributes'

# Chef client with verbose output
sudo chef-client -F doc

# Check cookbook dependencies
knife cookbook metadata web_server

# Validate cookbook syntax
chef exec cookstyle recipes/default.rb

# Test cookbook locally
kitchen converge
kitchen verify
kitchen destroy
```

---

## Best Practices

1. **Keep Recipes Simple** - Single responsibility per recipe
2. **Use Templates** - Don't hardcode configuration files
3. **Leverage Attributes** - Easy to customize without recipe changes
4. **Test Thoroughly** - Unit + Integration tests before production
5. **Document Cookbooks** - README and inline comments
6. **Version Constraints** - Lock cookbook versions in metadata.rb
7. **Idempotent Operations** - Recipes should be safe to run multiple times
8. **Avoid Execute** - Use native resources when possible
9. **Encrypt Sensitive Data** - Use encrypted data bags for secrets
10. **Use Roles** - Group related recipes into roles for consistency

---

*Chef Infrastructure Automation Guide - Enterprise Configuration Management*
