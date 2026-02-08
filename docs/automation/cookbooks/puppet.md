# Puppet - Configuration Management & Infrastructure Automation

Complete guide to using Puppet for declarative infrastructure automation, node classification, and configuration management.

## Table of Contents

1. [Puppet Fundamentals](#puppet-fundamentals)
2. [Architecture](#architecture)
3. [Installation](#installation)
4. [Manifest Syntax](#manifest-syntax)
5. [Resources](#resources)
6. [Classes & Modules](#classes--modules)
7. [Variables & Facts](#variables--facts)
8. [Conditionals & Expressions](#conditionals--expressions)
9. [Templates](#templates)
10. [Hiera - Hierarchy Lookup](#hiera---hierarchy-lookup)
11. [Node Classification](#node-classification)
12. [Relationships & Ordering](#relationships--ordering)
13. [Testing](#testing)
14. [Troubleshooting](#troubleshooting)

---

## Puppet Fundamentals

Puppet is a declarative configuration management tool that uses a Domain-Specific Language (DSL) to define desired system state.

### Key Concepts

**Manifest**: Puppet configuration files (`.pp`) defining desired state
**Resource**: Atomic unit of system configuration
**Module**: Reusable collection of manifests and data
**Catalog**: Compiled set of resources for a specific node
**Fact**: System information (OS, CPU, memory, etc.) collected from nodes
**Hiera**: Hierarchical data lookup system for configuration
**PuppetDB**: Centralized database storing node facts and reports

### Declarative vs Imperative

```puppet
# Puppet is DECLARATIVE - you define desired state
# Puppet figures out HOW to achieve it

# Desired state: Nginx is installed and running
package { 'nginx':
  ensure => 'installed',
}

service { 'nginx':
  ensure => 'running',
  enable => true,
}

# Puppet handles:
# - Installing nginx package
# - Starting the service
# - Enabling auto-start on boot
# - Restarting if config changes
```

---

## Architecture

### Puppet Architecture Modes

```
AGENT-SERVER MODE (Most Common)
┌──────────────┐                    ┌─────────────────┐
│              │──── HTTPS ────────>│                 │
│ Puppet Agent │  Request Catalog   │ Puppet Master   │
│  (Node)      │<──── HTTPS ────────│  (Server)       │
│              │  Return Catalog    │                 │
└──────────────┘                    └─────────────────┘
    Every 30 minutes                Compiles Manifests
    (configurable)                  Stores Reports

STANDALONE MODE (No Server)
┌──────────────┐
│              │
│ Puppet Agent │ Apply local manifests directly
│  (Node)      │ No server communication
│              │
└──────────────┘
```

### Master Components

```
Puppet Master Server
├── Manifest Files (/etc/puppetlabs/code/environments/production/)
├── Modules (/etc/puppetlabs/code/modules/)
├── SSL Certificates (mutual authentication)
├── PuppetDB (node facts, catalogs)
├── Reports Storage
└── Hiera Data Files
```

---

## Installation

### Install Puppet Master (Server)

```bash
# Ubuntu 20.04
wget https://apt.puppetlabs.com/puppet8-release-focal.deb
sudo dpkg -i puppet8-release-focal.deb
sudo apt update
sudo apt install puppetserver

# Start service
sudo systemctl start puppetserver
sudo systemctl enable puppetserver

# Verify
sudo /opt/puppetlabs/bin/puppet master --version

# Check logs
sudo tail -f /var/log/puppetlabs/puppetserver/puppetserver.log
```

### Install Puppet Agent (Node)

```bash
# Ubuntu 20.04
wget https://apt.puppetlabs.com/puppet8-release-focal.deb
sudo dpkg -i puppet8-release-focal.deb
sudo apt update
sudo apt install puppet-agent

# Configure agent
sudo /opt/puppetlabs/bin/puppet config set server puppet.example.com

# Start service
sudo systemctl start puppet
sudo systemctl enable puppet

# Check logs
sudo journalctl -u puppet -f
```

### Bootstrap Agent to Master

```bash
# On master - sign agent certificate
sudo /opt/puppetlabs/bin/puppet cert list
sudo /opt/puppetlabs/bin/puppet cert sign web-server-01.example.com

# On agent - trigger run
sudo /opt/puppetlabs/bin/puppet agent -t

# Verify connectivity
sudo /opt/puppetlabs/bin/puppet agent --test --verbose
```

---

## Manifest Syntax

### Basic Structure

```puppet
# manifests/site.pp - Main manifest file

# Resource declaration
resource_type { 'title':
  attribute => value,
  attribute => value,
}

# Example: Ensure nginx package is installed
package { 'nginx':
  ensure => 'present',
}

# Service resource
service { 'nginx':
  ensure    => 'running',
  enable    => true,
  subscribe => Package['nginx'],  # Restart if package changes
}

# File resource
file { '/etc/nginx/nginx.conf':
  ensure  => 'file',
  owner   => 'root',
  group   => 'root',
  mode    => '0644',
  content => template('nginx/nginx.conf.erb'),
}
```

### Puppet DSL Basics

```puppet
# Comments
# Single line comment
/* Multi-line
   comment */

# Variables (must start with $)
$nginx_workers = 4
$server_name = "web.example.com"
$ports = [80, 443, 8080]

# String interpolation
$message = "Hello ${::hostname}"
$file_path = "/etc/app/${server_name}.conf"

# Arrays
$packages = ['curl', 'wget', 'git', 'vim']
$first = $packages[0]  # 'curl'
$count = $packages.length

# Hashes
$db_config = {
  'host'     => 'db.example.com',
  'port'     => 5432,
  'username' => 'app_user',
  'password' => 'secret',
}
$host = $db_config['host']

# Boolean values
$debug = true
$enable_ssl = false
```

---

## Resources

### Common Resources

#### Package Management
```puppet
# Install package
package { 'nginx':
  ensure => 'present',
}

# Install specific version
package { 'postgresql':
  ensure => '14.3-1.pgdg100+1',
}

# Install multiple packages
package { ['curl', 'wget', 'git']:
  ensure => 'present',
}

# Remove package
package { 'apache2':
  ensure => 'absent',
}
```

#### File Management
```puppet
# Create/manage file with content
file { '/etc/app/config.yml':
  ensure  => 'file',
  owner   => 'app',
  group   => 'app',
  mode    => '0640',
  content => 'key: value\nother: data',
}

# Create directory
file { '/var/www/html':
  ensure => 'directory',
  owner  => 'www-data',
  group  => 'www-data',
  mode   => '0755',
}

# Recursively manage directory
file { '/opt/app':
  ensure  => 'directory',
  owner   => 'app',
  group   => 'app',
  mode    => '0755',
  recurse => true,  # Set ownership recursively
}

# Create symlink
file { '/usr/local/bin/myapp':
  ensure => 'link',
  target => '/opt/app/bin/myapp',
}

# Copy file from module
file { '/etc/nginx/nginx.conf':
  ensure => 'file',
  source => 'puppet:///modules/nginx/nginx.conf',
}
```

#### Service Management
```puppet
# Enable and start service
service { 'nginx':
  ensure => 'running',
  enable => true,
}

# Stop service
service { 'apache2':
  ensure => 'stopped',
  enable => false,
}

# Restart on config change
service { 'postgresql':
  ensure    => 'running',
  enable    => true,
  subscribe => File['/etc/postgresql/postgresql.conf'],
}
```

#### User & Group Management
```puppet
# Create group
group { 'app-users':
  ensure => 'present',
  gid    => 1001,
}

# Create user
user { 'deploy':
  ensure     => 'present',
  uid        => 1001,
  gid        => 1001,
  home       => '/home/deploy',
  managehome => true,
  shell      => '/bin/bash',
  password   => sha512_password('SecurePassword123'),  # Hashed
}

# Add user to group
user { 'ubuntu':
  ensure => 'present',
  groups => ['docker', 'sudo'],
}
```

#### Exec Resource
```puppet
# Run arbitrary command
exec { 'build app':
  command => '/usr/bin/make',
  cwd     => '/opt/app',
  user    => 'app',
  group   => 'app',
  onlyif  => '/bin/test -f /opt/app/Makefile',
}

# Unless condition (opposite of onlyif)
exec { 'initialize database':
  command => '/usr/local/bin/init_db.sh',
  unless  => '/usr/bin/test -f /var/lib/app/db.initialized',
}

# Refresh when notified
exec { 'reload nginx':
  command     => '/usr/sbin/nginx -s reload',
  refreshonly => true,
}
```

---

## Classes & Modules

### Module Structure

```
nginx/
├── manifests/
│   ├── init.pp           # Main class definition
│   ├── install.pp
│   ├── config.pp
│   ├── service.pp
│   └── params.pp         # Default parameters
├── templates/
│   ├── nginx.conf.erb
│   ├── site.conf.erb
│   └── ssl.conf.erb
├── files/
│   ├── mime.types
│   └── ssl_cert.pem
├── hiera.yaml
├── metadata.json
└── README.md
```

### Class Definition

```puppet
# manifests/init.pp
class nginx (
  String $version = 'latest',
  Integer $workers = $facts['processors']['count'],
  Integer $worker_connections = 1024,
  String $server_name = $facts['fqdn'],
) {
  # Include related classes
  include nginx::install
  include nginx::config
  include nginx::service

  # Class relationships
  Class['nginx::install']
    -> Class['nginx::config']
    -> Class['nginx::service']
}
```

### Class Parameters

```puppet
# Parameterized class with defaults
class apache (
  String $version = 'present',
  Array[String] $mods = [],
  Array[String] $configs = [],
  Boolean $enable_ssl = false,
) {
  package { 'apache2':
    ensure => $version,
  }

  $mods.each |$mod| {
    exec { "enable module ${mod}":
      command => "/usr/sbin/a2enmod ${mod}",
      unless  => "/bin/grep -q LoadModule.*${mod}.so /etc/apache2/mods-enabled/*",
    }
  }
}
```

### Include Classes

```puppet
# manifests/site.pp
node 'web.example.com' {
  # Simple include
  include nginx
  include nodejs

  # Include with parameters
  class { 'postgresql':
    version => '14',
    port    => 5432,
  }
}
```

### Defined Types (Reusable Constructs)

```puppet
# manifests/vhost.pp
define nginx::vhost (
  String $server_name = $title,
  String $document_root = "/var/www/${title}",
  Integer $port = 80,
  Boolean $ssl = false,
  String $ssl_cert = undef,
  String $ssl_key = undef,
) {
  file { "${document_root}":
    ensure => 'directory',
    owner  => 'www-data',
    group  => 'www-data',
    mode   => '0755',
  }

  file { "/etc/nginx/sites-available/${server_name}.conf":
    ensure  => 'file',
    content => template('nginx/vhost.conf.erb'),
  }

  file { "/etc/nginx/sites-enabled/${server_name}.conf":
    ensure => 'link',
    target => "/etc/nginx/sites-available/${server_name}.conf",
  }
}

# Usage
nginx::vhost { 'example.com':
  document_root => '/var/www/example.com',
  port          => 443,
  ssl           => true,
  ssl_cert      => '/etc/nginx/ssl/example.com.crt',
  ssl_key       => '/etc/nginx/ssl/example.com.key',
}
```

---

## Variables & Facts

### Built-in Facts

```puppet
# System facts available in all manifests
$facts['os']['family']              # debian, redhat, etc
$facts['os']['name']                # Ubuntu, CentOS, etc
$facts['os']['release']['major']    # Version: 20, 8, etc
$facts['hostname']                  # Short hostname
$facts['fqdn']                      # Fully qualified domain name
$facts['ipaddress']                 # Primary IP address
$facts['ipaddress_eth0']            # Interface specific IP
$facts['processors']['count']       # Number of CPUs
$facts['memory']['system']['total'] # Total RAM
$facts['kernelversion']             # Kernel version

# Legacy fact syntax (still works)
$::operatingsystem
$::operatingsystemrelease
$::ipaddress
```

### Using Facts

```puppet
# Conditional based on OS family
case $facts['os']['family'] {
  'debian': {
    $nginx_package = 'nginx'
    $service_manager = 'systemd'
  }
  'redhat': {
    $nginx_package = 'nginx'
    $service_manager = 'systemd'
  }
  default: {
    fail("Unsupported OS: ${facts['os']['family']}")
  }
}

# Set worker processes based on CPU count
$nginx_workers = $facts['processors']['count'] > 4 ? {
  true  => 8,
  false => $facts['processors']['count'],
}

# Compute port based on hostname
$app_port = $facts['hostname'] =~ /^prod/ ? {
  true  => 8080,
  false => 3000,
}
```

### Custom Facts

```puppet
# lib/facter/custom_facts.rb
Facter.add(:app_version) do
  setcode do
    if File.exist?('/opt/app/version.txt')
      File.read('/opt/app/version.txt').strip
    else
      'unknown'
    end
  end
end

# Usage in manifest
if $facts['app_version'] =~ /^1\.[0-9]+/ {
  notify { "Running legacy app version ${facts['app_version']}": }
}
```

---

## Conditionals & Expressions

### If/Else Statements

```puppet
# Simple if
if $facts['os']['family'] == 'debian' {
  notify { 'Using Debian-based system': }
}

# If/else
if $facts['os']['family'] == 'debian' {
  package { 'nginx': ensure => 'present' }
} else {
  package { 'httpd': ensure => 'present' }
}

# If/elsif/else
if $facts['ipaddress'] =~ /^10\./ {
  notify { 'Private network detected': }
} elsif $facts['ipaddress'] =~ /^172\./ {
  notify { 'Secondary private network': }
} else {
  notify { 'Public network': }
}
```

### Case Statements

```puppet
case $facts['os']['family'] {
  'debian': {
    $pkg_manager = 'apt'
    $db_package = 'postgresql'
  }
  'redhat': {
    $pkg_manager = 'yum'
    $db_package = 'postgresql-server'
  }
  /windows/: {
    $pkg_manager = 'chocolatey'
    $db_package = 'postgresql'
  }
  default: {
    fail("OS not supported: ${facts['os']['family']}")
  }
}

package { $db_package: ensure => 'present' }
```

### Selectors (Ternary Operator)

```puppet
# Syntax: condition ? true_value : false_value
$ssl_enabled = $facts['networking']['primaryeth']['fqdn'] =~ /^prod/ ? true : false

$worker_count = $facts['processors']['count'] > 4 ? {
  true  => 8,
  false => $facts['processors']['count'],
}

$log_level = $environment == 'production' ? {
  true  => 'warn',
  false => 'debug',
}
```

---

## Templates

### ERB Template Syntax

```erb
<!-- templates/nginx.conf.erb -->
user www-data;
worker_processes <%= @worker_processes %>;
pid /run/nginx.pid;

events {
    worker_connections <%= @worker_connections %>;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout <%= @keepalive_timeout %>s;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    <%# Conditional block -%>
    <% if @enable_ssl -%>
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    <% end -%>

    <%# Iterate over upstream servers -%>
    upstream backend {
    <% @backend_servers.each do |server| -%>
        server <%= server %>;
    <% end -%>
    }

    server {
        listen 80;
        server_name <%= @server_name %>;
    }
}
```

### Using Templates in Manifests

```puppet
file { '/etc/nginx/nginx.conf':
  ensure  => 'file',
  owner   => 'root',
  group   => 'root',
  mode    => '0644',
  content => epp('nginx/nginx.conf.erb', {
    'worker_processes'  => 4,
    'worker_connections' => 2048,
    'keepalive_timeout'  => 65,
    'enable_ssl'        => true,
    'server_name'       => 'web.example.com',
    'backend_servers'   => ['10.0.1.10:8080', '10.0.1.11:8080'],
  }),
}
```

---

## Hiera - Hierarchy Lookup

Central configuration data management system.

### Hiera Directory Structure

```
/etc/puppetlabs/code/
├── hiera.yaml
└── data/
    ├── common.yaml
    ├── nodes/
    │   ├── web-01.example.com.yaml
    │   └── db-01.example.com.yaml
    ├── environments/
    │   ├── production.yaml
    │   └── staging.yaml
    └── os/
        ├── debian.yaml
        └── redhat.yaml
```

### Hiera Configuration

```yaml
# hiera.yaml
version: 5

defaults:
  datadir: /etc/puppetlabs/code/data
  data_hash: yaml_data

hierarchy:
  - name: "Per-node data"
    path: "nodes/%{trusted.certname}.yaml"
  
  - name: "Environment data"
    path: "environments/%{server_facts.environment}.yaml"
  
  - name: "OS family"
    path: "os/%{facts.os.family}.yaml"
  
  - name: "Common data"
    path: "common.yaml"
```

### Hiera Data Files

```yaml
# data/common.yaml
nginx::version: 'latest'
nginx::worker_processes: 4
nginx::worker_connections: 1024

postgresql::version: '14'
postgresql::port: 5432

# data/nodes/web-01.example.com.yaml
nginx::worker_processes: 8
nginx::worker_connections: 2048

# data/environments/production.yaml
app::debug: false
app::log_level: 'warn'
db::enable_replication: true

# data/environments/staging.yaml
app::debug: true
app::log_level: 'debug'
db::enable_replication: false
```

### Accessing Hiera Data

```puppet
# Use Hiera lookup in classes
class nginx {
  $version = lookup('nginx::version')
  $workers = lookup('nginx::worker_processes')
  
  package { 'nginx':
    ensure => $version,
  }
}

# Lookup with default value
$timeout = lookup('nginx::timeout', Integer, 'first', 65)

# Array merge
$packages = lookup('packages', Array, 'unique', [])
```

---

## Node Classification

### Site Manifest (site.pp)

```puppet
# manifests/site.pp

# Default node
node default {
  notify { "Unconfigured node: ${facts['fqdn']}": }
}

# Specific nodes
node 'web-01.example.com' {
  include nginx
  include nodejs
  include logstash
}

node 'db-01.example.com' {
  include postgresql
  include pgbackrest
}

# Multiple nodes with regex
node /^web-\d+\.example\.com$/ {
  include nginx
  include php
  include memcached
}

# Node groups by role
node 'web-01.example.com', 'web-02.example.com', 'web-03.example.com' {
  include web_role
}
```

### External Node Classifier (ENC)

```bash
#!/bin/bash
# /usr/local/bin/node_classifier.sh
# Custom script that returns node configuration

HOSTNAME=$1

case $HOSTNAME in
  web-*)
    cat <<EOF
---
classes:
  nginx:
    worker_processes: 8
  nodejs:
    version: 18.0.0
parameters:
  environment: production
EOF
    ;;
  db-*)
    cat <<EOF
---
classes:
  postgresql:
    version: '14'
parameters:
  environment: production
EOF
    ;;
esac
```

---

## Relationships & Ordering

### Resource Relationships

```puppet
# Notify relationship (restart on change)
package { 'nginx':
  ensure => 'present',
  notify => Service['nginx'],  # Restart nginx if package changes
}

service { 'nginx':
  ensure => 'running',
  enable => true,
}

# Subscribe relationship (listen for changes)
service { 'nginx':
  ensure    => 'running',
  enable    => true,
  subscribe => File['/etc/nginx/nginx.conf'],  # Restart if config changes
}

file { '/etc/nginx/nginx.conf':
  ensure  => 'file',
  content => template('nginx/nginx.conf.erb'),
}
```

### Dependency Arrows

```puppet
# Before/require syntax
file { '/var/www/html':
  ensure  => 'directory',
  before  => Service['nginx'],  # Create directory BEFORE starting nginx
}

service { 'nginx':
  ensure => 'running',
  require => Package['nginx'],  # Ensure package installed BEFORE starting service
}

# Chaining arrows
Package['nginx'] -> File['/etc/nginx/nginx.conf'] -> Service['nginx']

# Complex chains
Class['nginx::install']
  -> Class['nginx::config']
  -> Class['nginx::service']
```

---

## Testing

### Unit Tests with rspec-puppet

```ruby
# spec/classes/nginx_spec.rb
require 'spec_helper'

describe 'nginx' do
  on_supported_os.each do |os, os_facts|
    context "on #{os}" do
      let(:facts) { os_facts }
      
      context 'with default parameters' do
        it { is_expected.to compile.with_all_deps }
        it { is_expected.to contain_package('nginx').with_ensure('present') }
        it { is_expected.to contain_service('nginx').with_ensure('running') }
        it { is_expected.to contain_file('/etc/nginx/nginx.conf').with_owner('root') }
      end

      context 'with custom parameters' do
        let(:params) do
          {
            'version' => '1.18',
            'worker_processes' => 8,
          }
        end

        it { is_expected.to contain_package('nginx').with_ensure('1.18') }
      end
    end
  end
end
```

### Integration Tests with Puppet Apply

```bash
# Test manifest directly
sudo puppet apply --modulepath=/opt/puppet/modules /tmp/test.pp

# Test with verbose output
sudo puppet apply --verbose --debug /tmp/test.pp

# Dry run (no changes)
sudo puppet agent --noop --verbose
```

---

## Troubleshooting

### Common Issues & Debugging

```bash
# Check Puppet syntax
puppet parser validate manifests/site.pp

# Agent logs
sudo journalctl -u puppet -f
tail -f /var/log/puppetlabs/puppet/puppet-agent.log

# Master logs
sudo journalctl -u puppetserver -f
tail -f /var/log/puppetlabs/puppetserver/puppetserver.log

# Test agent run
sudo puppet agent --test --verbose --debug

# Check catalog compilation
sudo puppet agent --test --trace

# List certificate requests
sudo puppet cert list

# Revoke certificate (if needed)
sudo puppet cert revoke web-server-01.example.com
sudo puppet cert clean web-server-01.example.com
```

### Fact-Related Issues

```puppet
# Check available facts on node
puppet facts

# Use facts in debug
notify { "Debug: ${facts['os']['family']}": }

# Handle fact changes
if $facts['processors']['count'] > 4 {
  notify { "Multi-core system detected": }
}
```

### Resource Compilation Errors

```puppet
# ❌ Error: Duplicate resource
file { '/etc/config':
  ensure => 'directory',
}
file { '/etc/config':  # ERROR - Same resource declared twice
  ensure => 'directory',
}

# ✅ Solution: Use unique titles
file { '/etc/config':
  ensure => 'directory',
}
file { '/etc/config/app':
  ensure => 'directory',
}

# Resource not found
service { 'nginx':
  subscribe => File['/etc/nginx/nginx.conf'],  # Ensure File resource exists
}
```

### Hiera Lookup Failures

```bash
# Debug Hiera lookups
puppet lookup --debug nginx::version

# Test Hiera hierarchy
puppet lookup nginx --hierarchy

# Check YAML syntax
ruby -e "require 'yaml'; YAML.load_file('data/common.yaml')"
```

---

## Best Practices

1. **Idempotent Resources** - Safe to run multiple times
2. **Avoid Exec Resources** - Use native resources when possible
3. **Use Hiera** - Separate data from code
4. **Test Thoroughly** - Unit and integration tests
5. **Document Modules** - README and inline comments
6. **Version Control** - Track all manifests
7. **Naming Conventions** - Consistent class/resource names
8. **Relationships** - Define explicit ordering
9. **Classes Over Defined Types** - Prefer classes for large configurations
10. **Fail Fast** - Catch errors early with validation

---

*Puppet Configuration Management Guide - Enterprise Infrastructure Automation*
