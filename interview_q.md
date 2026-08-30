# Ansible Interview Questions & Answers — 3+ Years DevOps Engineer

## Overview

This document contains the top **50 Ansible interview questions** for a DevOps Engineer with **3+ years of experience**.

The questions cover:

* Ansible fundamentals
* Inventory management
* Playbooks and modules
* Variables and precedence
* Facts
* Conditions and loops
* Roles
* Collections
* Ansible Vault
* Production deployments
* Troubleshooting
* AWX / Ansible Tower
* RBAC
* Execution Environments
* CI/CD integration
* Rolling deployments
* Blue-green deployments
* Real-time production scenarios

For a 3+ years DevOps Engineer, interviewers generally expect both **Ansible knowledge and practical production experience**.

---

# 1. What is Ansible and why is it used in DevOps?

Ansible is an open-source automation and configuration-management tool used to automate:

* Server provisioning
* Configuration management
* Application deployment
* Package installation
* Service management
* User management
* Infrastructure operations
* Cloud automation
* Application orchestration

Ansible is agentless and primarily communicates with Linux servers using SSH and Windows servers using WinRM.

### Basic Architecture

```text
                    Ansible Control Node
                            |
                    -------------------
                    |        |        |
                   SSH      SSH      SSH
                    |        |        |
                    v        v        v
                 Server-1 Server-2 Server-3
```

The major advantage is that an Ansible agent is not required on the managed nodes.

---

# 2. What are the major components of Ansible?

Important Ansible components include:

* Control Node
* Managed Nodes
* Inventory
* Playbooks
* Plays
* Tasks
* Modules
* Roles
* Variables
* Facts
* Handlers
* Templates
* Collections
* Plugins
* Ansible Vault
* Execution Environments

### Architecture

```text
Control Node
    |
    +-- Inventory
    +-- Playbooks
    +-- Roles
    +-- Variables
    +-- Vault
    +-- Collections
    |
    +------ SSH ------> Managed Nodes
```

---

# 3. What is an Ansible inventory?

An inventory defines the hosts or groups of hosts that Ansible manages.

Example:

```ini
[webservers]
web01
web02
web03

[appservers]
app01
app02

[dbservers]
db01
db02
```

You can also specify IP addresses:

```ini
[webservers]
10.0.1.10
10.0.1.11
```

Inventory can also contain host variables:

```ini
[webservers]
web01 ansible_host=10.0.1.10 ansible_user=ubuntu
web02 ansible_host=10.0.1.11 ansible_user=ubuntu
```

---

# 4. What is the difference between static and dynamic inventory?

## Static Inventory

A static inventory is manually maintained.

```ini
[web]
web01
web02
web03
```

## Dynamic Inventory

Dynamic inventory obtains infrastructure information dynamically from platforms such as:

* AWS
* Azure
* VMware
* Kubernetes
* Cloud providers

### Example

```text
AWS EC2
   |
   v
Dynamic Inventory
   |
   v
Ansible
   |
   v
EC2 Instances
```

Dynamic inventory is useful when servers are frequently created and terminated.

---

# 5. What is an Ansible playbook?

A playbook is a YAML file containing automation instructions.

Example:

```yaml
---
- name: Install Nginx
  hosts: webservers
  become: true

  tasks:

    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Start nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

A playbook can contain:

* Multiple plays
* Tasks
* Variables
* Conditions
* Loops
* Handlers
* Roles
* Templates

---

# 6. What is an Ansible module?

Ansible modules are reusable units that perform specific operations on managed nodes.

Common modules include:

```text
package
apt
yum
dnf
service
systemd
copy
template
file
user
group
command
shell
uri
get_url
unarchive
lineinfile
replace
debug
```

Example:

```yaml
- name: Install nginx
  ansible.builtin.package:
    name: nginx
    state: present
```

Modules are preferred over manually executing shell commands whenever a suitable module exists.

---

# 7. What is the difference between command and shell?

## command

The `command` module executes a command directly without a shell.

```yaml
- name: Check uptime
  ansible.builtin.command:
    cmd: uptime
```

## shell

The `shell` module executes the command through a shell.

It supports:

* Pipes
* Redirection
* Shell operators
* Environment expansion

Example:

```yaml
- name: Check application errors
  ansible.builtin.shell:
    cmd: cat /var/log/app.log | grep ERROR
```

### Production preference

I prefer Ansible modules wherever possible because they provide better idempotency and predictable behavior.

If no suitable module exists, I use `command` or `shell` carefully.

---

# 8. What is idempotency in Ansible?

Idempotency means running the same automation multiple times should result in the same desired state without unnecessarily changing the system.

Example:

```yaml
- name: Install nginx
  ansible.builtin.package:
    name: nginx
    state: present
```

If nginx is already installed, Ansible will not reinstall it.

The result would typically show:

```text
changed=0
```

### Why is idempotency important?

In production environments, playbooks may be executed repeatedly through:

* CI/CD
* AWX
* Scheduled jobs
* Disaster recovery
* Configuration remediation

Therefore, playbooks should be safe to rerun.

---

# 9. What is the difference between copy and template?

## copy

The `copy` module is generally used for static files.

```yaml
- name: Copy nginx configuration
  ansible.builtin.copy:
    src: nginx.conf
    dest: /etc/nginx/nginx.conf
```

## template

The `template` module uses Jinja2 and supports variables.

```yaml
- name: Configure nginx
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
```

Template:

```jinja2
server {
    listen {{ nginx_port }};
    server_name {{ server_name }};
}
```

Templates are useful when configuration differs between environments.

---

# 10. What are Ansible handlers?

Handlers are tasks that run only when another task notifies them.

Example:

```yaml
- name: Deploy nginx configuration
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Restart nginx

handlers:

  - name: Restart nginx
    ansible.builtin.service:
      name: nginx
      state: restarted
```

If the configuration does not change, the handler will not restart nginx.

This avoids unnecessary service restarts.

---

# 11. What are Ansible variables?

Variables make playbooks reusable and environment-specific.

Example:

```yaml
nginx_port: 8080
app_version: "2.5.1"
environment: production
```

Use:

```yaml
- name: Deploy application
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/app/app.conf
```

Template:

```jinja2
port={{ nginx_port }}
version={{ app_version }}
environment={{ environment }}
```

---

# 12. Explain Ansible variable precedence

Ansible supports variables from many locations.

Common sources include:

```text
Role defaults
Inventory variables
group_vars
host_vars
Play variables
Task variables
Registered variables
Extra variables
```

`--extra-vars` generally has very high precedence.

Example:

```bash
ansible-playbook deploy.yml -e "app_version=3.0"
```

This is useful for passing environment-specific values during CI/CD.

---

# 13. What are group_vars and host_vars?

`group_vars` contains variables applicable to a group.

Example:

```text
inventory/
├── hosts
├── group_vars/
│   ├── webservers.yml
│   └── dbservers.yml
└── host_vars/
    ├── web01.yml
    └── db01.yml
```

Example:

```yaml
# group_vars/webservers.yml

nginx_port: 8080
```

Host-specific configuration can be placed under:

```text
host_vars/
```

---

# 14. What are Ansible facts?

Facts are system information collected from managed nodes.

Examples include:

```text
ansible_hostname
ansible_os_family
ansible_distribution
ansible_memtotal_mb
ansible_processor_vcpus
ansible_default_ipv4
```

Run:

```bash
ansible all -m setup
```

Example:

```yaml
- name: Display operating system
  ansible.builtin.debug:
    var: ansible_distribution
```

---

# 15. How can you disable fact gathering?

Use:

```yaml
gather_facts: false
```

Example:

```yaml
- name: Application deployment
  hosts: appservers
  gather_facts: false

  tasks:

    - name: Deploy application
      ...
```

This can improve execution speed when facts are not required.

---

# 16. How do you use when conditions?

Example:

```yaml
- name: Install nginx on Ubuntu
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_os_family == "Debian"
```

Another example:

```yaml
when: environment == "production"
```

Conditions are useful when the same playbook needs to support different environments or operating systems.

---

# 17. How do you use loops in Ansible?

Example:

```yaml
- name: Install packages
  ansible.builtin.package:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - git
    - curl
```

Loops are useful when the same operation must be performed for multiple values.

---

# 18. How do you handle failures in Ansible?

Ansible provides several mechanisms:

```text
failed_when
changed_when
block
rescue
always
ignore_errors
any_errors_fatal
max_fail_percentage
```

Example:

```yaml
- block:

    - name: Deploy application
      ansible.builtin.command:
        cmd: /opt/app/deploy.sh

  rescue:

    - name: Rollback application
      ansible.builtin.command:
        cmd: /opt/app/rollback.sh

  always:

    - name: Cleanup
      ansible.builtin.file:
        path: /tmp/deployment
        state: absent
```

---

# 19. What is the difference between failed_when and changed_when?

`failed_when` determines whether a task should be considered failed.

Example:

```yaml
failed_when: result.rc != 0
```

`changed_when` determines whether Ansible should report the task as changed.

Example:

```yaml
changed_when: false
```

This is especially useful when using `command` or `shell`.

---

# 20. What is Ansible check mode?

Check mode performs a dry run.

```bash
ansible-playbook deploy.yml --check
```

It helps determine what changes Ansible would make without applying them.

For production changes, I would use check mode where the involved modules support meaningful check-mode behavior.

---

# 21. What is an Ansible role?

A role provides a standardized structure for organizing reusable automation.

Example:

```text
roles/
└── nginx/
    ├── defaults/
    │   └── main.yml
    ├── vars/
    │   └── main.yml
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    ├── files/
    ├── meta/
    └── README.md
```

Roles make automation:

* Reusable
* Maintainable
* Testable
* Easier to understand

---

# 22. What is the difference between defaults and vars in a role?

`defaults/main.yml` contains variables intended to be easily overridden.

Example:

```yaml
nginx_port: 80
```

`vars/main.yml` has higher precedence and is generally used for values that should not normally be overridden.

For configurable application settings, I generally prefer role defaults.

---

# 23. How do you reuse roles?

Example:

```yaml
- name: Configure web servers
  hosts: webservers

  roles:
    - nginx
    - monitoring
```

The same roles can be reused across:

```text
Development
Staging
Production
```

with different inventory variables.

---

# 24. What are Ansible Collections?

Collections package Ansible content such as:

* Modules
* Plugins
* Roles

Example:

```bash
ansible-galaxy collection install community.docker
```

Then:

```yaml
- name: Start nginx container
  community.docker.docker_container:
    name: nginx
    image: nginx:latest
    state: started
```

Collections are important in modern Ansible because many modules are distributed through collections.

---

# 25. What is Ansible Vault?

Ansible Vault is used to encrypt sensitive information.

Create a vault file:

```bash
ansible-vault create secrets.yml
```

Example content:

```yaml
db_username: appuser
db_password: super-secret-password
```

The file is encrypted.

Run:

```bash
ansible-playbook deploy.yml --ask-vault-pass
```

In production, Vault can be integrated with CI/CD or external secret-management systems.

---

# 26. How do you manage production passwords securely?

I would avoid storing plaintext credentials in Git.

Possible approaches include:

```text
Ansible Vault
AWS Secrets Manager
HashiCorp Vault
CyberArk
AWX credentials
Azure Key Vault
Other enterprise secret managers
```

With AWX, credentials can be centrally managed and injected into jobs without exposing them directly in playbooks.

---

# 27. How do you prevent secrets from appearing in Ansible logs?

Use:

```yaml
no_log: true
```

Example:

```yaml
- name: Create application user
  ansible.builtin.user:
    name: appuser
    password: "{{ app_password }}"
  no_log: true
```

This is important when handling:

* Passwords
* API tokens
* SSH keys
* Database credentials

---

# 28. Production Scenario: 100 servers require different applications

### Scenario

You have:

```text
100 production servers
```

You need:

```text
50 servers → Nginx
50 servers → RabbitMQ
```

### Approach

Create inventory groups:

```ini
[nginx]
server01
server02
...
server50

[rabbitmq]
server51
server52
...
server100
```

Create reusable roles:

```text
roles/
├── nginx/
└── rabbitmq/
```

Playbook:

```yaml
- name: Configure nginx
  hosts: nginx
  become: true

  roles:
    - nginx

- name: Configure RabbitMQ
  hosts: rabbitmq
  become: true

  roles:
    - rabbitmq
```

For a cloud environment where servers change frequently, I would use dynamic inventory and metadata/tags rather than manually maintaining the host list.

---

# 29. Production Scenario: Deploy to 500 servers without downtime

I would use rolling deployment.

Example:

```yaml
- name: Rolling deployment
  hosts: appservers
  serial: 10

  tasks:

    - name: Deploy application
      ...
```

Only 10 servers are processed at a time.

Typical workflow:

```text
10 servers
    |
    v
Remove from Load Balancer
    |
    v
Deploy
    |
    v
Health Check
    |
    v
Add back to Load Balancer
    |
    v
Next 10 servers
```

This reduces production risk.

---

# 30. Ansible deployment failed halfway through. What would you do?

First I would identify:

* Failed host
* Failed task
* Error message
* Application logs
* System logs
* Configuration changes

Then I would determine whether the playbook is safely rerunnable.

Because production playbooks should be idempotent, I should ideally be able to rerun the playbook without corrupting the existing environment.

For critical deployments I would implement:

```text
Backup
Health checks
Block/rescue
Rollback
Serial deployment
Monitoring
Notifications
```

---

# 31. One server failed while 99 servers succeeded. What would you do?

I would avoid blindly rerunning the entire deployment.

First identify the failed server.

Example:

```bash
ansible-playbook deploy.yml --limit server42
```

I can also run specific tags:

```bash
ansible-playbook deploy.yml \
  --limit server42 \
  --tags application
```

This reduces production risk.

---

# 32. How do you deploy only to a subset of servers?

Use the `--limit` option.

Single host:

```bash
ansible-playbook deploy.yml --limit server42
```

Multiple hosts:

```bash
ansible-playbook deploy.yml --limit "server42,server43"
```

Group:

```bash
ansible-playbook deploy.yml --limit webservers
```

---

# 33. How would you implement zero-downtime deployment?

Assume:

```text
                Load Balancer
                     |
          -----------------------
          |     |     |     |
        App1  App2   App3   App4
```

Deployment process:

```text
Remove App1 from LB
        |
        v
Deploy new version
        |
        v
Health check
        |
        v
Add App1 back
        |
        v
Process App2
        |
        v
Process App3
        |
        v
Process App4
```

Ansible's `serial` can control the deployment batch size.

---

# 34. How would you implement blue-green deployment?

Blue-green deployment uses two environments.

```text
Blue  → Current Production
Green → New Version
```

Deploy to Green:

```text
Green
  |
  +-- Deploy
  |
  +-- Test
  |
  +-- Health Check
```

Then switch traffic:

```text
Load Balancer
      |
      v
    Green
```

If problems occur:

```text
Load Balancer
      |
      v
     Blue
```

This provides a quick rollback mechanism.

---

# 35. How would you implement rolling deployment?

Example:

```yaml
- name: Rolling deployment
  hosts: appservers

  serial: 10
  max_fail_percentage: 20

  tasks:

    - name: Deploy application
      ...
```

`serial` controls how many hosts are processed at a time.

`max_fail_percentage` helps stop the deployment if too many hosts fail.

---

# 36. Ansible cannot connect to a server. How do you troubleshoot?

First test connectivity:

```bash
ansible all -m ping
```

Then test SSH directly:

```bash
ssh user@server
```

Check:

```text
Network connectivity
Security groups
Firewall
SSH port
SSH key
Username
SSH permissions
known_hosts
Bastion configuration
ansible_user
ansible_host
```

Use verbose mode:

```bash
ansible-playbook deploy.yml -vvv
```

---

# 37. What does UNREACHABLE mean in Ansible?

`UNREACHABLE` generally means Ansible could not establish communication with the managed node.

Common causes:

```text
SSH failure
Wrong IP
Wrong username
Incorrect SSH key
Firewall
Security group
Host is down
DNS issue
Bastion problem
```

Example:

```text
server01 | UNREACHABLE!
```

I would validate SSH connectivity first.

---

# 38. Playbook works manually over SSH but fails from Ansible. What could be wrong?

I would compare the execution environment.

Possible causes:

```text
Different SSH user
Different SSH key
Different PATH
Different environment variables
sudo/become issue
Python interpreter
Proxy/bastion configuration
File permissions
```

I would run:

```bash
ansible-inventory --graph
```

and:

```bash
ansible all -m ping -vvv
```

I would also verify the actual user:

```yaml
ansible_user: ubuntu
```

and privilege escalation:

```yaml
become: true
```

---

# 39. Python is missing on a managed node. Can Ansible still manage it?

Most normal Ansible modules require Python on Linux managed nodes.

A bootstrap approach is to use `raw`, because `raw` does not require Python on the remote host.

Example:

```yaml
- name: Install Python
  ansible.builtin.raw: |
    apt-get update &&
    apt-get install -y python3
```

After Python is installed, normal Ansible modules can be used.

---

# 40. A task takes 20 minutes across 1,000 servers. How would you optimize it?

I would investigate:

* Fact gathering
* Network latency
* Fork count
* Serial value
* Module selection
* Repeated downloads
* Unnecessary tasks
* SSH configuration
* Controller resources

For example:

```ini
[defaults]
forks = 50
```

If facts aren't required:

```yaml
gather_facts: false
```

For large environments I would also consider:

```text
AWX execution nodes
Parallelism
Fact caching
Inventory caching
Artifact repositories
Optimized playbooks
```

---

# 41. What is AWX and how is it different from Ansible?

Ansible is primarily the automation engine.

AWX provides a centralized platform around Ansible.

AWX provides:

```text
Web UI
RBAC
Credentials
Inventories
Projects
Job Templates
Schedules
Workflows
Notifications
Audit/history
Execution Environments
```

### Architecture

```text
Users
  |
  v
 AWX
  |
  +---- Projects
  +---- Inventories
  +---- Credentials
  +---- Job Templates
  +---- Workflows
  +---- Notifications
  |
  v
Execution Environment
  |
  v
Managed Servers
```

---

# 42. What is an AWX Job Template?

A Job Template defines how a playbook should be executed.

It generally contains:

```text
Inventory
Project
Playbook
Credentials
Execution Environment
Variables
Limit
Tags
Verbosity
```

Example workflow:

```text
User
  |
  v
Job Template
  |
  v
Ansible Playbook
  |
  v
Execution Environment
  |
  v
Servers
```

---

# 43. What is an Execution Environment?

An Execution Environment is a container image containing the dependencies required to execute Ansible automation.

Example:

```text
Execution Environment
├── ansible-core
├── Python
├── community.docker
├── boto3
├── AWS SDK
└── Custom dependencies
```

This makes automation more consistent and portable.

Instead of depending on packages installed directly on the AWX host, required dependencies are packaged into the execution environment.

---

# 44. How would you implement RBAC in AWX?

I would use:

```text
Organizations
Teams
Users
Credentials
Inventories
Projects
Job Templates
```

Then assign only the permissions required.

Example:

```text
Development Team
        |
        v
Development Inventory
        |
        v
Development Job Templates


Operations Team
        |
        v
Production Inventory
        |
        v
Production Job Templates
```

I would follow the principle of least privilege.

---

# 45. How would you prevent developers from running production deployments?

I would implement AWX RBAC.

Example:

```text
Developers
   |
   +-- Read/Execute Dev
   |
   X-- No Production Execute


Operations
   |
   +-- Execute Production


Administrators
   |
   +-- Manage AWX
```

Production credentials and inventories should only be accessible to authorized teams.

---

# 46. How would you implement notifications in AWX?

AWX can send notifications for job events.

Typical workflow:

```text
Job Template
      |
      v
Job Execution
      |
      +-------- SUCCESS --------> Email/Notification
      |
      +-------- FAILURE --------> Email/Notification
```

For example, production deployment failure can trigger an email notification to the operations team.

Notifications can be configured for events such as:

```text
Job Started
Job Successful
Job Failed
Job Error
Workflow Successful
Workflow Failed
```

---

# 47. How do you integrate Ansible with Jenkins?

A typical pipeline is:

```text
Developer
    |
    v
Git
    |
    v
Jenkins
    |
    +-- Lint
    +-- Test
    +-- Validate
    |
    v
Ansible
    |
    v
Development
    |
    v
Approval
    |
    v
Production
```

Example Jenkins stage:

```groovy
stage('Deploy') {
    steps {
        sh '''
        ansible-playbook \
          -i inventory/prod \
          deploy.yml
        '''
    }
}
```

In larger organizations, Jenkins may trigger an AWX Job Template through an API rather than running Ansible directly.

---

# 48. How would you implement Git-based Ansible automation?

I would keep Ansible automation in Git.

Example:

```text
ansible/
├── inventories/
│   ├── dev/
│   ├── stage/
│   └── prod/
│
├── roles/
│   ├── nginx/
│   ├── application/
│   ├── rabbitmq/
│   └── monitoring/
│
├── playbooks/
│   ├── site.yml
│   ├── web.yml
│   ├── app.yml
│   └── db.yml
│
├── group_vars/
├── requirements.yml
└── README.md
```

CI/CD pipeline:

```text
Git Commit
    |
    v
Pull Request
    |
    v
Lint
    |
    v
Syntax Check
    |
    v
Security Scan
    |
    v
Testing
    |
    v
Approval
    |
    v
AWX
    |
    v
Deployment
```

---

# 49. How would you design an Ansible repository for a large organization?

A scalable structure could look like:

```text
ansible-project/
│
├── ansible.cfg
├── requirements.yml
│
├── inventories/
│   │
│   ├── dev/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │
│   ├── stage/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │
│   └── prod/
│       ├── hosts.yml
│       └── group_vars/
│
├── playbooks/
│   ├── site.yml
│   ├── web.yml
│   ├── app.yml
│   └── db.yml
│
├── roles/
│   ├── nginx/
│   ├── application/
│   ├── rabbitmq/
│   └── monitoring/
│
├── templates/
├── files/
└── README.md
```

I would also implement:

```text
Git version control
Code reviews
Ansible Lint
Testing
Secret management
AWX
RBAC
Environment separation
CI/CD
Documentation
```

---

# 50. Production Scenario: Ansible changed configuration on 200 servers and caused an outage. How would you prevent it?

This is an important production-level interview question.

I would introduce multiple controls.

## Before Production

```text
Git PR
   |
   v
Code Review
   |
   v
ansible-lint
   |
   v
Syntax Validation
   |
   v
Check Mode
   |
   v
Development
   |
   v
Staging
   |
   v
Approval
```

## Production Deployment

Instead of changing all 200 servers:

```yaml
serial: 5
```

This means only five servers are processed at a time.

Workflow:

```text
5 Servers
    |
    v
Deploy
    |
    v
Health Check
    |
    v
Successful?
   / \
 Yes  No
 |     |
 v     v
Next  Stop
Batch
```

Additional controls:

* Automated health checks
* `serial`
* `max_fail_percentage`
* Load-balancer integration
* Backups
* Rollback strategy
* AWX RBAC
* Production approval workflow
* Monitoring
* Notifications
* Audit logs

The key principle is:

> Never allow a high-risk production change to modify hundreds of servers simultaneously without validation and controlled rollout.

---

# Important Ansible Commands

## Test connectivity

```bash
ansible all -m ping
```

## Check inventory

```bash
ansible-inventory -i inventory --graph
```

## List hosts

```bash
ansible all --list-hosts
```

## Run a playbook

```bash
ansible-playbook -i inventory deploy.yml
```

## Check syntax

```bash
ansible-playbook deploy.yml --syntax-check
```

## Dry run

```bash
ansible-playbook deploy.yml --check
```

## Verbose execution

```bash
ansible-playbook deploy.yml -vvv
```

## Limit execution to one host

```bash
ansible-playbook deploy.yml --limit server01
```

## Run specific tags

```bash
ansible-playbook deploy.yml --tags nginx
```

## Skip tags

```bash
ansible-playbook deploy.yml --skip-tags nginx
```

## List tags

```bash
ansible-playbook deploy.yml --list-tags
```

## List tasks

```bash
ansible-playbook deploy.yml --list-tasks
```

## List hosts

```bash
ansible-playbook deploy.yml --list-hosts
```

## Use extra variables

```bash
ansible-playbook deploy.yml \
  -e "environment=production"
```

## Install a collection

```bash
ansible-galaxy collection install community.docker
```

## Create a role

```bash
ansible-galaxy role init nginx
```

## Encrypt a file

```bash
ansible-vault encrypt secrets.yml
```

## Edit encrypted file

```bash
ansible-vault edit secrets.yml
```

## Decrypt a file

```bash
ansible-vault decrypt secrets.yml
```

---

# Top 20 Questions to Prioritize

If you have limited preparation time, focus heavily on these questions:

1. What is Ansible and how does it work?
2. Explain Ansible architecture.
3. What is inventory?
4. Static vs dynamic inventory.
5. What is idempotency?
6. Explain variables and variable precedence.
7. What are `group_vars` and `host_vars`?
8. What are Ansible facts?
9. `command` vs `shell`.
10. `copy` vs `template`.
11. What are handlers?
12. What are Ansible roles?
13. What is Ansible Vault?
14. How do you troubleshoot `UNREACHABLE`?
15. How do you deploy to 500 servers without downtime?
16. Explain `serial` and rolling deployment.
17. How do you handle deployment failures?
18. Explain AWX architecture.
19. Explain AWX RBAC and Job Templates.
20. Explain Ansible integration with Jenkins/CI/CD.

---

# Production Scenarios You Must Prepare

For a 3+ years DevOps interview, prepare these scenarios particularly well.

## Scenario 1 — 100 Servers

```text
100 Servers
    |
    +---- 50 Nginx
    |
    +---- 50 RabbitMQ
```

Expected concepts:

```text
Inventory Groups
Roles
Playbooks
Variables
Templates
Handlers
Idempotency
```

---

## Scenario 2 — 500 Server Deployment

```text
500 Servers
     |
     v
Rolling Deployment
     |
     v
10 Servers at a Time
     |
     v
Health Check
     |
     v
Next Batch
```

Expected concepts:

```text
serial
max_fail_percentage
health checks
load balancer
rollback
```

---

## Scenario 3 — One Server Failed

```text
100 Servers
     |
     +---- 99 SUCCESS
     |
     +---- 1 FAILED
```

Expected approach:

```bash
ansible-playbook deploy.yml --limit failed-server
```

Don't unnecessarily rerun the entire production deployment.

---

## Scenario 4 — Production Configuration Change

Expected process:

```text
Git
 ↓
Pull Request
 ↓
Code Review
 ↓
Lint
 ↓
Syntax Check
 ↓
Test
 ↓
Staging
 ↓
Approval
 ↓
Production
```

---

## Scenario 5 — Production Outage

Expected controls:

```text
Check Mode
Serial Deployment
Health Checks
Monitoring
Backup
Rollback
RBAC
Approval
Notifications
Audit Logs
```

---

# Interview Answering Strategy

For a 3+ years DevOps Engineer interview, avoid giving only textbook definitions.

Instead of saying:

> "Ansible is an automation tool."

Give a practical answer:

> "In my projects, I use Ansible primarily for configuration management, application deployment and infrastructure automation. I maintain inventories for different environments, use reusable roles, keep sensitive values in Vault or an external secret manager, and integrate the playbooks with CI/CD or AWX. For production deployments, I use controls such as `serial`, health checks and rollback mechanisms so that a deployment failure on one batch doesn't impact the entire fleet."

This demonstrates:

```text
Ansible Knowledge
       +
Production Experience
       +
DevOps Practices
       +
Troubleshooting
       +
Automation Design
```

which is what interviewers generally expect from a **3+ years DevOps Engineer**.
