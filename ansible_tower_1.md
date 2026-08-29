# Ansible Tower / AWX – Components, Architecture & Workflow

## 1. What is Ansible Tower?

**Ansible Tower** is a web-based enterprise platform used to centrally manage, execute, schedule, and monitor Ansible automation.

Ansible itself is primarily a command-line automation engine:

```bash
ansible-playbook -i inventory deploy.yml
```

Tower/AWX provides a centralized platform around Ansible with features such as:

* Web UI
* REST API
* Centralized inventory management
* Credential management
* Projects
* Job Templates
* Workflow Templates
* Execution Environments
* Scheduling
* RBAC
* Job history and logs
* Notifications

### Tower vs AWX

| Ansible Tower                               | AWX                                                          |
| ------------------------------------------- | ------------------------------------------------------------ |
| Red Hat enterprise product                  | Upstream open-source project                                 |
| Commercial/supportable                      | Community/open source                                        |
| Enterprise-focused                          | Development, learning, lab and internal automation use cases |
| Part of Red Hat Ansible Automation Platform | Upstream project for Ansible automation                      |

> **Note:** Ansible Tower functionality is now part of **Red Hat Ansible Automation Platform (AAP)**.

---

# 2. Why do we need AWX/Tower?

With Ansible CLI, automation can be executed directly:

```text
Developer / DevOps Engineer
            |
            v
       Ansible CLI
            |
      +-----+-----+
      |     |     |
      v     v     v
   Server1 Server2 Server3
```

This works well for small environments.

However, in enterprise environments we need:

* Centralized automation
* Role-based access control
* Credential management
* Job scheduling
* Job history
* Centralized logs
* API integration
* Workflow orchestration
* Multiple execution nodes
* Standardized execution environments

AWX provides these capabilities.

```text
                       AWX
                        |
       +----------------+----------------+
       |                |                |
    Projects        Inventories      Credentials
       |                |                |
       +----------------+----------------+
                        |
                  Job Templates
                        |
                  Workflow Jobs
                        |
                 Execution Nodes
                        |
              +---------+---------+
              |         |         |
              v         v         v
           Server1   Server2   Server3
```

---

# 3. High-Level AWX Architecture

```text
                         USERS
                           |
              +------------+------------+
              |                         |
              v                         v
           Web UI                    REST API
              |                         |
              +------------+------------+
                           |
                           v
                    +-------------+
                    |     AWX     |
                    |             |
                    | Web / API   |
                    | Task Engine |
                    | Scheduler   |
                    +------+------+
                           |
            +--------------+--------------+
            |              |              |
            v              v              v
       PostgreSQL        Redis         Projects
            |                             |
            |                             v
            |                            Git
            |
            v
        AWX Data

                           |
                           v
                  Execution Nodes
                  +------+------+------+
                  |      |      |      |
                  v      v      v      |
                Node1  Node2  Node3
                  |      |      |
                  v      v      v
                 EE     EE     EE
                  |      |      |
                  +------+------+ 
                           |
                           v
                     Ansible Engine
                           |
              +------------+------------+
              |            |            |
              v            v            v
           Server1      Server2      Server3
```

---

# 4. Major AWX Components

## 4.1 Web UI

The Web UI is the graphical interface used to manage AWX.

From the UI, administrators and users can manage:

* Organizations
* Users
* Teams
* Projects
* Inventories
* Credentials
* Job Templates
* Workflow Templates
* Execution Environments
* Schedules
* Jobs

Example:

```text
AWX
 |
 +-- Projects
 |
 +-- Inventories
 |
 +-- Credentials
 |
 +-- Templates
 |
 +-- Workflows
 |
 +-- Jobs
```

---

# 5. AWX REST API

AWX provides a REST API that allows automation systems to interact with AWX programmatically.

For example:

```text
GitHub Actions
       |
       v
    AWX API
       |
       v
 Job Template
       |
       v
 Ansible Job
```

Instead of manually clicking **Launch**, a CI/CD pipeline can trigger an AWX Job Template using the API.

Typical integrations include:

* GitHub Actions
* Jenkins
* GitLab CI/CD
* Other automation platforms

---

# 6. PostgreSQL

PostgreSQL is used as the persistent database for AWX.

It stores information such as:

* Users
* Organizations
* Projects
* Inventories
* Job Templates
* Workflow Templates
* Job history
* Execution information
* RBAC information
* Configuration metadata

Conceptually:

```text
                    AWX
                     |
                     v
                PostgreSQL
                     |
          +----------+----------+
          |          |          |
        Users       Jobs     Templates
        Projects    History   Inventories
```

### Important

AWX's PostgreSQL database is used to store **AWX application data**.

It is not the same as the database used by an application being deployed.

For example:

```text
AWX PostgreSQL
      |
      +-- AWX configuration
      +-- Job history
      +-- Users
      +-- Templates


Application PostgreSQL
      |
      +-- Application data
      +-- Business records
```

---

# 7. Redis

Redis is used internally for task/message coordination between AWX components.

Conceptually:

```text
AWX Components
      |
      v
    Redis
      |
      v
Task / Event Coordination
```

Redis is an internal infrastructure component and normally isn't something an Ansible user interacts with directly.

---

# 8. Task Manager / Scheduler

When a user launches an AWX job, AWX needs to determine:

* What playbook should run?
* Which inventory should be used?
* Which credentials are required?
* Which Execution Environment should be used?
* Which execution node should run the job?

The AWX task/scheduling system handles this orchestration.

```text
User
 |
 v
AWX
 |
 v
Task Manager / Scheduler
 |
 +-- Check Project
 |
 +-- Check Inventory
 |
 +-- Check Credentials
 |
 +-- Check Execution Environment
 |
 +-- Select Execution Node
 |
 v
Run Ansible Job
```

---

# 9. Execution Nodes

Execution nodes are the systems where Ansible automation actually runs.

For example:

```text
                    AWX
                     |
               Job Scheduling
                     |
          +----------+----------+
          |          |          |
          v          v          v
      Node 1      Node 2      Node 3
          |          |          |
          v          v          v
         EE         EE         EE
          |          |          |
          v          v          v
       Ansible    Ansible    Ansible
          |          |          |
          v          v          v
       Servers    Servers    Servers
```

This is particularly important in a distributed AWX architecture.

The AWX control plane schedules the job, while the execution node performs the actual Ansible execution.

---

# 10. Execution Environment (EE)

An **Execution Environment** is a container image containing everything required to execute Ansible automation.

A typical Execution Environment can contain:

```text
Execution Environment
 |
 +-- Ansible Core
 +-- Python
 +-- Ansible Collections
 +-- Python Packages
 +-- System Packages
 +-- Other Dependencies
```

For example:

```text
Custom Execution Environment
 |
 +-- ansible-core
 +-- community.docker
 +-- amazon.aws
 +-- community.general
 +-- boto3
 +-- Docker SDK
```

This provides consistency between different execution nodes.

---

# 11. Why Execution Environments are Important

Consider an Ansible playbook using:

```yaml
- name: Create Docker container
  community.docker.docker_container:
    name: myapp
    image: nginx:latest
    state: started
```

If `community.docker` is available in the local Ansible CLI environment:

```text
Ansible CLI
     |
     v
Local Environment
     |
     +-- community.docker
     |
     v
Playbook
     |
     v
SUCCESS
```

But AWX may use a different Execution Environment:

```text
AWX
 |
 v
Execution Environment
 |
 +-- community.docker NOT AVAILABLE
 |
 v
Playbook
 |
 v
FAILURE
```

Therefore:

> **Ansible CLI working does not automatically mean that the same playbook will work in AWX.**

The dependencies must exist inside the Execution Environment used by AWX.

---

# 12. Custom Execution Environment Example

A custom Execution Environment can be created with the required Ansible collections.

Conceptually:

```text
Dockerfile / EE Definition
          |
          v
     Build Image
          |
          v
  Custom EE Image
          |
          v
      Docker Hub
          |
          v
        AWX
          |
          v
Execution Environment
          |
          v
     Job Template
          |
          v
      Playbook
```

For example:

```text
Custom EE
 |
 +-- Ansible
 +-- Python
 +-- community.docker
 +-- Docker dependencies
 +-- Other required collections
```

This makes AWX execution predictable and reproducible.

---

# 13. Projects

A **Project** represents the source code containing Ansible automation.

Most commonly, the source is a Git repository.

```text
GitHub
   |
   | Git Sync
   v
AWX Project
   |
   +-- playbooks/
   |
   +-- roles/
   |
   +-- inventories/
   |
   +-- requirements.yml
```

Example project structure:

```text
ansible-project/
|
+-- playbooks/
|   +-- deploy.yml
|   +-- configure.yml
|
+-- roles/
|   +-- nginx/
|   +-- docker/
|
+-- requirements.yml
```

AWX can synchronize the project repository and use the playbooks from it.

---

# 14. Inventory

Inventory defines the hosts that Ansible manages.

Example:

```ini
[webservers]
web01
web02

[dbservers]
db01
db02
```

In AWX:

```text
Inventory
 |
 +-- Web Servers
 |    +-- web01
 |    +-- web02
 |
 +-- DB Servers
      +-- db01
      +-- db02
```

AWX can work with:

* Static inventories
* Dynamic inventories
* Cloud inventory sources
* Inventory groups
* Host variables
* Group variables

---

# 15. Credentials

AWX provides centralized credential management.

Examples include:

* SSH credentials
* Machine credentials
* Git credentials
* AWS credentials
* Kubernetes credentials
* Vault credentials
* Container registry credentials

Example:

```text
AWX
 |
 +-- SSH Credential
 |    +-- Username
 |    +-- SSH Private Key
 |
 +-- AWS Credential
 |
 +-- Git Credential
 |
 +-- Registry Credential
```

Instead of hard-coding credentials in playbooks:

```yaml
password: mypassword
```

credentials can be securely managed by AWX.

---

# 16. Job Template

A **Job Template** defines how a particular Ansible job should be executed.

A Job Template commonly contains:

```text
Job Template
 |
 +-- Project
 |
 +-- Playbook
 |
 +-- Inventory
 |
 +-- Credentials
 |
 +-- Execution Environment
 |
 +-- Variables
 |
 +-- Execution Options
```

Example:

```text
Deploy Application
 |
 +-- Project: devops-ansible
 +-- Playbook: deploy.yml
 +-- Inventory: production
 +-- Credential: production-ssh
 +-- EE: custom-ee
```

The user can then click:

```text
Launch
```

---

# 17. Workflow Template

A Workflow Template allows multiple jobs to be connected together.

Example:

```text
                 Workflow
                    |
                    v
        Provision Infrastructure
                    |
                    v
          Configure Servers
                    |
                    v
          Deploy Application
                    |
                    v
             Run Tests
                    |
              +-----+-----+
              |           |
           Success      Failure
              |           |
              v           v
           Notify      Rollback
```

Workflows are useful for complex automation and CI/CD processes.

---

# 18. Role-Based Access Control (RBAC)

AWX provides Role-Based Access Control.

Example:

```text
Organization
 |
 +-- Admin
 |    |
 |    +-- Full Access
 |
 +-- Developer
 |    |
 |    +-- Launch Specific Jobs
 |
 +-- Operator
      |
      +-- Manage Inventory
```

RBAC can control who can:

* View projects
* Modify projects
* View inventories
* Modify inventories
* Launch jobs
* Manage credentials
* Manage organizations
* Administer AWX

---

# 19. Notifications

AWX can send notifications based on job results.

Example:

```text
Ansible Job
     |
     +----------------+
     |                |
     v                v
 SUCCESS            FAILURE
     |                |
     v                v
  Email/Slack      Email/Slack
```

Notifications can be useful for:

* Deployment success
* Deployment failure
* Workflow completion
* Scheduled jobs
* Operational alerts

---

# 20. Complete AWX Job Workflow

Let's understand what happens when a user launches an Ansible job.

### Step 1 — Developer pushes code

```text
Developer
    |
    v
GitHub
```

---

### Step 2 — AWX synchronizes the Project

```text
GitHub
   |
   v
AWX Project
   |
   v
Ansible Playbook
```

Example:

```text
deploy.yml
```

---

### Step 3 — User launches Job Template

```text
User
 |
 v
AWX Web UI
 |
 v
Job Template
```

---

### Step 4 — AWX gathers required configuration

AWX identifies:

```text
Project
   +
Playbook
   +
Inventory
   +
Credentials
   +
Execution Environment
```

---

### Step 5 — AWX schedules the job

```text
AWX Control Plane
       |
       v
Task Manager / Scheduler
       |
       v
Execution Node
```

---

### Step 6 — Execution Environment is started

```text
Execution Node
       |
       v
Execution Environment
       |
       +-- Ansible
       +-- Python
       +-- Collections
       +-- Dependencies
```

---

### Step 7 — Ansible connects to target hosts

For Linux hosts:

```text
Execution Environment
          |
          | SSH
          |
    +-----+-----+-----+
    |           |     |
    v           v     v
 Server1     Server2 Server3
```

For Windows hosts, Ansible can use WinRM/PSRP depending on the setup.

---

### Step 8 — Playbook executes

Example:

```yaml
- name: Deploy application
  hosts: webservers

  tasks:

    - name: Start Docker container
      community.docker.docker_container:
        name: myapp
        image: nginx:latest
        state: started
```

---

### Step 9 — Results return to AWX

```text
Server1 --------+
Server2 --------+----> Execution Node
Server3 --------+            |
                              v
                             AWX
                              |
                              v
                         Job Output
```

---

### Step 10 — AWX stores job information

AWX records information such as:

```text
Job
 |
 +-- Status
 +-- Start Time
 +-- End Time
 +-- Duration
 +-- Output
 +-- Failed Tasks
 +-- Execution Information
```

This makes it possible to troubleshoot previous automation runs.

---

# 21. Complete AWX Architecture

```text
                              USERS
                                |
                   +------------+------------+
                   |                         |
                   v                         v
                Web UI                    REST API
                   |                         |
                   +------------+------------+
                                |
                                v
                       +----------------+
                       |      AWX       |
                       |                |
                       | Web / API      |
                       | Task Manager   |
                       | Scheduler      |
                       +-------+--------+
                               |
              +----------------+----------------+
              |                |                |
              v                v                v
         PostgreSQL          Redis           Projects
              |                                 |
              |                                 v
              |                                Git
              |
              v
        AWX Application Data

                               |
                               v
                      Execution Nodes
                 +-------------+-------------+
                 |             |             |
                 v             v             v
              Node 1        Node 2        Node 3
                 |             |             |
                 v             v             v
                EE            EE            EE
                 |             |             |
                 +-------------+-------------+
                               |
                               v
                         Ansible Engine
                               |
                  +------------+------------+
                  |            |            |
                  v            v            v
               Server1      Server2      Server3
```

---

# 22. Important AWX Objects

A simple way to remember the important objects is:

```text
Organization
      |
      v
   Project
      |
      v
 Inventory
      |
      v
 Credentials
      |
      v
 Job Template
      |
      v
Execution Environment
      |
      v
Execution Node
      |
      v
 Ansible Playbook
      |
      v
 Target Servers
```

However, these are **references/dependencies of the Job Template**, rather than a strict one-after-another execution sequence.

---

# 23. AWX vs Ansible CLI

| Feature                | Ansible CLI                 | AWX / Tower                       |
| ---------------------- | --------------------------- | --------------------------------- |
| Interface              | CLI                         | Web UI + API                      |
| Playbook execution     | Manual/CLI                  | Job Templates                     |
| Inventory              | Local files/dynamic sources | Centralized                       |
| Credentials            | Manually managed            | Centralized credential management |
| Job history            | Limited                     | Centralized                       |
| Scheduling             | Cron/external tools         | Built-in scheduling               |
| RBAC                   | Limited                     | Yes                               |
| Workflows              | Manual scripting            | Workflow Templates                |
| Execution Environment  | Local environment           | Managed EE                        |
| API                    | Ansible CLI/API ecosystem   | REST API                          |
| Centralized management | No                          | Yes                               |
| Multi-node execution   | Manual configuration        | Execution nodes                   |
| Notifications          | External setup              | Integrated                        |
| Audit/history          | Limited                     | Centralized                       |

---

# 24. Important Interview Question

### Does AWX execute the Ansible playbook itself?

A good answer:

> AWX acts as the control and orchestration platform for Ansible automation. It manages projects, inventories, credentials, job templates, scheduling, workflows and execution environments. When a job is launched, AWX schedules the job onto an execution node, where the configured Execution Environment runs Ansible and connects to the target hosts.

---

# 25. Real-Time Example: `community.docker`

A practical example is the `community.docker` issue.

### Ansible CLI

```text
Ansible CLI
     |
     v
Local Ansible Environment
     |
     +-- community.docker
     |
     v
Playbook
     |
     v
SUCCESS
```

### AWX Before Custom EE

```text
AWX
 |
 v
Execution Environment
 |
 +-- community.docker
 |       |
 |       X  Not Available
 |
 v
Playbook
 |
 v
FAILURE
```

### AWX After Custom EE

```text
Dockerfile / EE Definition
          |
          v
     Build Image
          |
          v
     Custom EE
          |
          +-- Ansible
          +-- Python
          +-- community.docker
          +-- Required Dependencies
          |
          v
       Docker Hub
          |
          v
          AWX
          |
          v
Execution Environment
          |
          v
     Job Template
          |
          v
       Playbook
          |
          v
        SUCCESS
```

This demonstrates an important production concept:

> **The Ansible environment used by the CLI and the Execution Environment used by AWX are separate environments.**

Therefore, required collections and dependencies must be available in the Execution Environment selected by the AWX Job Template.

---

# 26. Key Concepts to Remember

For interviews and real-world troubleshooting, remember these concepts:

```text
AWX
 |
 +-- Web UI / API
 |
 +-- Projects
 |     |
 |     +-- Git repositories
 |
 +-- Inventories
 |     |
 |     +-- Target hosts
 |
 +-- Credentials
 |     |
 |     +-- SSH / Cloud / Git / Vault
 |
 +-- Job Templates
 |
 +-- Workflow Templates
 |
 +-- Execution Environments
 |
 +-- Execution Nodes
 |
 +-- Scheduling
 |
 +-- RBAC
 |
 +-- Notifications
 |
 +-- PostgreSQL
 |
 +-- Redis
```

---

# 27. Simple Memory Diagram

The easiest way to remember the overall flow is:

```text
                 USER
                   |
                   v
              AWX Web UI
                   |
                   v
             Job Template
                   |
       +-----------+-----------+
       |           |           |
       v           v           v
    Project    Inventory   Credentials
       |           |           |
       +-----------+-----------+
                   |
                   v
         Execution Environment
                   |
                   v
            Execution Node
                   |
                   v
             Ansible Engine
                   |
          +--------+--------+
          |        |        |
          v        v        v
       Server1  Server2  Server3
                   |
                   v
              Job Results
                   |
                   v
                  AWX
                   |
                   v
             Job History
```

---

# 28. One-Line Explanation of Each Component

| Component             | Purpose                                       |
| --------------------- | --------------------------------------------- |
| AWX                   | Central automation platform                   |
| Web UI                | Manage automation through browser             |
| REST API              | Programmatically interact with AWX            |
| Project               | Source of Ansible playbooks                   |
| Inventory             | Defines target hosts                          |
| Credentials           | Provides secure authentication                |
| Job Template          | Defines how a playbook should run             |
| Workflow Template     | Chains multiple jobs                          |
| Execution Environment | Container containing Ansible and dependencies |
| Execution Node        | Runs the automation                           |
| Task Manager          | Schedules and dispatches jobs                 |
| PostgreSQL            | Stores AWX persistent data                    |
| Redis                 | Internal messaging/task coordination          |
| RBAC                  | Controls user/team permissions                |
| Notifications         | Reports job/workflow results                  |

---

# 29. Final Takeaway

The most important concept is:

```text
                    AWX
                     |
             Control / Orchestration
                     |
        +------------+------------+
        |            |            |
     Project     Inventory    Credentials
        |            |            |
        +------------+------------+
                     |
               Job Template
                     |
             Execution Environment
                     |
               Execution Node
                     |
              Ansible Engine
                     |
                     v
              Target Servers
```

In simple terms:

> **Ansible is the automation engine, while AWX/Tower provides centralized management, orchestration, security, scheduling, execution, monitoring and API capabilities around Ansible.**

The **Execution Environment** provides the dependencies required by Ansible, the **Execution Node** performs the actual automation, and the **target hosts** are the systems being configured or deployed.
