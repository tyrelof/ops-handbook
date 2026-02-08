# Infrastructure Automation & Configuration Management

Enterprise-scale infrastructure automation using declarative configuration management tools. Define infrastructure as code and manage nodes at scale.

## 📚 Core Topics

- [Ansible Automation](cookbooks/ansible.md) - Agentless automation with YAML playbooks
- [Chef Infrastructure](cookbooks/chef.md) - Ruby-based infrastructure as code
- [Puppet Configuration Management](cookbooks/puppet.md) - Declarative node management

## 🎯 Quick Comparison

| Feature | Ansible | Chef | Puppet |
|---------|---------|------|--------|
| **Agent** | Agentless (SSH) | Agent-based | Agent-based |
| **Language** | YAML | Ruby DSL | Puppet DSL |
| **Learning Curve** | Easiest | Moderate | Steeper |
| **Push/Pull** | Push | Pull | Pull |
| **Best For** | Simple, cloud-first | Complex orchestration | Enterprise-scale |
| **Scaling** | 100s of nodes | 1000s of nodes | 10,000s of nodes |

## 🚀 When to Use Each

### Ansible
- Quick provisioning of cloud instances
- Simple configuration changes
- No centralized server needed
- Multi-cloud/hybrid deployments
- Orchestration & application deployment

### Chef
- Complex infrastructure patterns
- Heavy use of data/templates
- Integration with cloud APIs
- Extensive testing requirements
- Large cookbooks/modules

### Puppet
- Large-scale enterprise deployments
- Strict compliance requirements
- Complex node classification
- Long-term infrastructure management
- Established teams with Puppet expertise

## 📦 Cookbooks Available

### 🔧 Ansible Automation & Infrastructure as Code
**[Full Cookbook](cookbooks/ansible.md)**

Agentless infrastructure automation using Python-based task execution over SSH:
- Inventory configuration and group management
- Playbook structure with tasks, handlers, templates
- Variables, facts, and conditional execution
- Roles for modular, reusable automation
- Templates for configuration file generation
- Loops, conditionals, error handling
- Idempotent operations and atomic deployments
- Rolling deployments with load balancer integration
- Security hardening playbook examples
- Testing playbooks (syntax, dry-run, idempotency)
- Common modules (file, package, service, user, git)
- Advanced patterns (handlers, tags, blocks, async)

**Best For**: Cloud infrastructure, application deployment, quick automation

---

### 🍳 Chef Infrastructure as Code
**[Full Cookbook](cookbooks/chef.md)**

Ruby-based infrastructure automation with centralized server management:
- Cookbook structure and metadata
- Resources (package, service, file, user, execute)
- Recipes and attributes
- Templates with ERB variable substitution
- Roles for server configuration profiles
- Environments for stage separation
- Data bags for encrypted secret storage
- Chef Server for centralized management
- Knife CLI for node management
- Testing with ChefSpec (unit) and InSpec (integration)
- Node convergence and runlist management
- Advanced patterns (notifications, guards, loops)

**Best For**: Complex infrastructure, API integrations, enterprise scale

---

### 🎭 Puppet Configuration Management
**[Full Cookbook](cookbooks/puppet.md)**

Declarative configuration management with master-agent architecture:
- Manifest syntax and resource declarations
- Modules and classes with parameterization
- Defined types for reusable constructs
- Facts and variables
- Conditionals and case statements
- Templates with ERB syntax
- Hiera for hierarchical data lookup
- Node classification (site.pp and ENCs)
- Resource relationships and ordering
- Puppet Server and agent architecture
- Testing with rspec-puppet
- PuppetDB for node inventory
- Master-agent pull model

**Best For**: Enterprise-scale deployments, compliance management, large teams

---

## 🎓 Learning Paths

### Path 1: Quick Infrastructure Setup (Ansible)
1. [Ansible Basics](cookbooks/ansible.md#ansible-fundamentals)
2. [Simple Playbooks](cookbooks/ansible.md#playbook-structure)
3. [Inventory Setup](cookbooks/ansible.md#inventory-configuration)
4. [Common Modules](cookbooks/ansible.md#common-modules)
5. [Deploy Your First Playbook](cookbooks/ansible.md#running-playbooks)

### Path 2: Enterprise Automation (Chef)
1. [Chef Fundamentals](cookbooks/chef.md#chef-fundamentals)
2. [Cookbook Structure](cookbooks/chef.md#cookbook-structure)
3. [Resources & Recipes](cookbooks/chef.md#recipes--resources)
4. [Chef Server Setup](cookbooks/chef.md#chef-server)
5. [Roles & Environments](cookbooks/chef.md#roles--environments)

### Path 3: Large-Scale Management (Puppet)
1. [Puppet Fundamentals](cookbooks/puppet.md#puppet-fundamentals)
2. [Manifest Syntax](cookbooks/puppet.md#manifest-syntax)
3. [Classes & Modules](cookbooks/puppet.md#classes--modules)
4. [Hiera Data Management](cookbooks/puppet.md#hiera---hierarchy-lookup)
5. [Node Classification](cookbooks/puppet.md#node-classification)

---

## 🏗️ Common Patterns

### Server Provisioning
All three tools handle basic provisioning:
```
Package Installation → Configuration Files → Service Management → Verification
```

### Multi-Environment Deployment
```
Dev → Staging → Production
```
- **Ansible**: Using inventory groups and extra vars
- **Chef**: Using environments feature
- **Puppet**: Using Hiera hierarchy

### Application Deployment
```
Code Checkout → Dependencies → Configure → Start
```
- **Ansible**: Playbooks with handlers
- **Chef**: Recipes with notifications
- **Puppet**: Classes with relationships

---

## 🔄 Typical Workflows

### Ansible Workflow
```bash
1. Define inventory
2. Write playbooks
3. Test with --check (dry-run)
4. Execute: ansible-playbook playbook.yml
5. Monitor output and logs
```

### Chef Workflow
```bash
1. Generate cookbook
2. Define recipes and resources
3. Test with kitchen test
4. Upload to Chef Server
5. Bootstrap nodes with knife bootstrap
6. Chef client runs on schedule (30 min default)
```

### Puppet Workflow
```bash
1. Write manifests/classes
2. Define in site.pp or ENC
3. Test with puppet apply --noop
4. Upload to Puppet Server
5. Puppet agent pulls and applies (30 min default)
```

---

## 📊 Scale & Complexity Matrix

```
Complexity
    ↑
    │  CHEF ────── PUPPET
    │    ╱
    │  ╱
    │ANSIBLE
    │
    └──────────────────────→ Scale
        Simple    Large
```

- **Ansible**: Best for straightforward tasks, any scale
- **Chef**: Best for complex infrastructure, medium-large scale
- **Puppet**: Best for very large deployments, strict requirements

---

## 🛠️ Tool Features Breakdown

### Control Flow
- **Ansible**: Tasks, handlers, roles, blocks
- **Chef**: Recipes, resources, notifications
- **Puppet**: Classes, defined types, relationships

### Secret Management
- **Ansible**: Vault for encrypted variables
- **Chef**: Encrypted data bags
- **Puppet**: Hiera with encrypted values

### Testing
- **Ansible**: Syntax check, --check mode, molecule
- **Chef**: ChefSpec (unit), InSpec (integration), Test Kitchen
- **Puppet**: rspec-puppet (unit), serverspec/inspec (integration)

### Distribution
- **Ansible**: Push-based, agentless
- **Chef**: Pull-based, client-server
- **Puppet**: Pull-based, client-server

---

## 🎯 Use Cases

### Rapid Cloud Deployment → **Ansible**
- Spin up instances in minutes
- Simple, repeatable configurations
- Multi-cloud flexibility

### Complex Application Stacks → **Chef**
- Multiple interdependent services
- Heavy customization per node
- Ruby scripting for complex logic

### Enterprise Compliance → **Puppet**
- Enforce standards across thousands of nodes
- Audit trail and reporting
- Strict change control

---

## 📚 Related Documentation

- [Linux System Management](../linux/system.md)
- [Windows PowerShell](../windows/core-powershell.md)
- [Docker Container Management](../docker/index.md)
- [Kubernetes Deployments](../kubernetes/index.md)
- [Active Directory Integration](../windows/cookbooks/active-directory-group-policy.md)

---

## 🚀 Next Steps

1. **Choose Your Tool**: Use the comparison table above
2. **Study the Cookbook**: Deep dive into your chosen tool
3. **Build a Lab**: Practice in non-production first
4. **Test Thoroughly**: Use built-in testing frameworks
5. **Document Playbooks/Cookbooks**: Future you will thank you
6. **Scale Gradually**: Start small, add complexity

---

*Infrastructure Automation & Configuration Management - Enterprise Operations*
