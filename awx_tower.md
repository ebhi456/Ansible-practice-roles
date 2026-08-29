# Ansible Tower / AWX — Components, Architecture & Workflow

## 1. What is Ansible Tower?

**Ansible Tower** is a web-based enterprise platform for managing and executing Ansible automation.

Ansible itself is primarily a **command-line automation engine**:

```bash
ansible-playbook deploy.yml
```

Tower adds a centralized platform around Ansible:

```text
                    Ansible
                       │
             ┌─────────┴─────────┐
             │                   │
        CLI / Playbook      Tower / AWX
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                 Web UI                    API
                    │                         │
                    └────────────┬────────────┘
                                 │
                       Centralized Automation
```

With Tower/AWX, teams can manage:

* Playbooks
* Inventories
* Credentials
* Projects
* Job Templates
* Execution Environments
* Schedules
* Workflows
* RBAC
* Job history
* Logs
* Notifications
* API-based automation

---

## Tower vs AWX

| Ansible Tower                      | AWX                                                        |
| ---------------------------------- | ---------------------------------------------------------- |
| Red Hat's enterprise product       | Upstream open-source project                               |
| Commercial/supportable             | Community/open source                                      |
| Enterprise support                 | Community support                                          |
| Production enterprise environments | Labs, learning, development and many internal environments |
| Historically based on AWX          | Upstream project for Tower                                 |

Today, Red Hat's enterprise automation platform is **Ansible Automation Platform (AAP)**, which incorporates the functionality that was historically provided by Ansible Tower.

---

# 2. Why Do We Need AWX/Tower?

Suppose you have 100 servers.

Without AWX:

```text
Developer / DevOps Engineer
        │
        │ SSH
        ▼
   Ansible CLI
        │
        ├── server1
        ├── server2
        ├── server3
        ├── ...
        └── server100
```

You execute:

```bash
ansible-playbook -i inventory deploy.yml
```

This works, but as the organization grows, you need centralized management.

AWX provides:

```text
                    AWX
                     │
       ┌─────────────┼─────────────┐
       │             │             │
    Projects     Inventories    Credentials
       │             │             │
       └─────────────┼─────────────┘
                     │
               Job Templates
                     │
               Workflow Jobs
                     │
              Execution Nodes
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Server1    Server2    Server3
```

### Main Benefits of AWX/Tower

* Centralized automation
* Web-based management
* Credential management
* Role-Based Access Control
* Job scheduling
* Job history
* Centralized logs
* Workflow automation
* REST API integration
* Multiple execution nodes
* Standardized Execution Environments
* Team-based automation

---

# 3. High-Level AWX Architecture

A simplified modern AWX architecture looks like this:

```text
                         USERS
                           │
                    ┌──────┴──────┐
                    │             │
                  Web UI         REST API
                    │             │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │     AWX     │
                    │             │
                    │ Web / API   │
                    │ Task Engine │
                    │ Scheduler   │
                    └──────┬──────┘
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
        PostgreSQL       Redis      Execution Nodes
             │             │             │
             │             │       ┌─────┼─────┐
             │             │       ▼     ▼     ▼
             │             │      Node1 Node2 Node3
             │             │
             └─────────────┴─────────────
```

> The exact deployment architecture can vary depending on the AWX version and deployment method, but these are the major concepts you should understand.

---

# 4. Major AWX Components

Let's understand each component.

---

## 4.1 Web UI

The Web UI is the interface you access through a browser.

For example:

```text
https://awx.example.com
```

From the UI, you can manage:

* Organizations
* Users
* Teams
* Projects
* Inventories
* Credentials
* Job Templates
* Workflow Templates
* Execution Environments
* Jobs
* Schedules

For example:

```text
AWX
 │
 ├── Projects
 ├── Inventories
 ├── Credentials
 ├── Templates
 ├── Workflows
 └── Jobs
```

---

# 5. API

AWX exposes a REST API.

Instead of manually clicking:

```text
Launch Job Template
```

you can trigger AWX through:

```text
CI/CD Pipeline
       │
       ▼
     AWX API
       │
       ▼
 Job Template
       │
       ▼
 Ansible Job
```

For example:

```text
GitHub Actions
      │
      ▼
   AWX API
      │
      ▼
Job Template
      │
      ▼
Deployment
```

This is extremely common in production.

---

# 6. PostgreSQL

PostgreSQL stores AWX's persistent application data.

Conceptually:

```text
                  AWX
                   │
                   ▼
              PostgreSQL
                   │
       ┌───────────┼───────────┐
       │           │           │
     Users       Jobs      Inventories
     Projects    Credentials Templates
```

It stores information such as:

* Users
* Organizations
* Projects
* Inventories
* Credential metadata
* Job Templates
* Jobs
* Workflow definitions
* Execution history
* RBAC information

### Important

**PostgreSQL is not your managed target-server database.**

For example:

```text
AWX PostgreSQL
      │
      └── Stores AWX configuration/history


Target Application
      │
      └── May have MySQL/PostgreSQL/etc.
```

They are completely different things.

---

# 7. Redis

Redis is used for internal messaging and task coordination in the AWX architecture.

Conceptually:

```text
AWX Components
      │
      ▼
    Redis
      │
      ▼
Task / Event Coordination
```

You don't normally interact with Redis directly when creating an Ansible job.

---

# 8. Task Manager / Dispatcher

This is one of the most important concepts.

When you click:

```text
Launch
```

AWX needs to determine:

> Where should this job run?

The AWX task system handles job scheduling and dispatching.

Conceptually:

```text
User
 │
 ▼
AWX
 │
 ▼
Task Manager
 │
 ├── Check job
 ├── Check inventory
 ├── Check credentials
 ├── Check execution environment
 └── Select execution node
          │
          ▼
      Run Ansible
```

---

# 9. Execution Nodes

This is particularly important for a **three-node AWX demonstration**.

Execution nodes are where the actual Ansible automation runs.

For example:

```text
                 AWX Controller
                       │
               Job Scheduling
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
     Execution      Execution    Execution
       Node 1         Node 2       Node 3
          │            │            │
          ▼            ▼            ▼
       Ansible      Ansible      Ansible
          │            │            │
          ▼            ▼            ▼
      Servers       Servers       Servers
```

The execution node contains what is needed to execute the automation, especially the **Execution Environment**.

---

# 10. Execution Environment

This is another very important AWX concept.

An **Execution Environment (EE)** is essentially a container image containing the dependencies required to run your Ansible automation.

For example:

```text
Execution Environment
│
├── Ansible Core
├── Python
├── community.docker
├── amazon.aws
├── community.general
├── Other Python packages
└── Other system dependencies
```

---

## Why Execution Environments Matter

Consider the following scenario.

You have:

```text
Ansible CLI
     │
     ▼
Local Ansible Environment
     │
     └── community.docker
             │
             ▼
         Playbook
             │
             ▼
          SUCCESS
```

But AWX is using an Execution Environment that does not contain the required collection:

```text
AWX
 │
 ▼
Execution Environment
 │
 └── community.docker ❌
          │
          ▼
       Playbook
          │
          ▼
        FAILURE
```

After creating an Execution Environment containing the required collection:

```text
Custom EE
│
├── Ansible
├── Python
└── community.docker ✅
```

and configuring AWX to use it:

```text
AWX
 │
 ▼
Custom Execution Environment
 │
 ▼
community.docker
 │
 ▼
Playbook
 │
 ▼
Success
```

This is a very good real-world example of why Execution Environments matter.

> **Important:** Ansible CLI and AWX may use different execution environments. A collection being installed on the Ansible CLI host does not automatically make it available inside the AWX Execution Environment.

---

# 11. Projects

A **Project** represents the source of your Ansible automation.

Usually, the source is Git.

For example:

```text
GitHub
   │
   │ git clone / pull
   ▼
AWX Project
   │
   ├── playbooks/
   ├── roles/
   ├── inventories/
   └── requirements.yml
```

Example repository:

```text
ansible-project/
│
├── playbooks/
│   ├── deploy.yml
│   └── configure.yml
│
├── roles/
│   ├── nginx/
│   └── docker/
│
└── requirements.yml
```

Example Git repository:

```text
https://github.com/company/ansible-project
```

AWX can synchronize the project repository and use the playbooks from it.

---

# 12. Inventory

Inventory tells Ansible:

> Which machines should I manage?

Example:

```ini
[webservers]
web01
web02

[dbservers]
db01
db02
```

AWX can manage:

* Static inventories
* Dynamic inventories
* Cloud inventories
* Inventory sources
* Inventory groups
* Host variables
* Group variables

For example:

```text
Inventory
│
├── Web Servers
│   ├── web01
│   └── web02
│
└── DB Servers
    ├── db01
    └── db02
```

---

# 13. Credentials

AWX securely stores credentials needed by automation.

### SSH Credential

```text
AWX
 │
 ▼
SSH Credential
 │
 ├── Username
 └── SSH Private Key
```

Used to connect to:

```text
Linux Server
```

Other credential types include:

* Machine credentials
* AWS credentials
* Git credentials
* Vault credentials
* Container registry credentials
* Cloud credentials
* Kubernetes credentials

### Important

You don't normally hard-code passwords or private keys inside playbooks.

Instead:

```text
AWX
 │
 ▼
Credential
 │
 ▼
Job Template
 │
 ▼
Execution Node
 │
 ▼
Target Server
```

---

# 14. Job Template

A **Job Template** combines the resources required to execute an Ansible playbook.

Think of it as:

```text
Job Template
│
├── Project
├── Playbook
├── Inventory
├── Credentials
├── Execution Environment
├── Variables
└── Options
```

Example:

```text
Deploy Application
│
├── Project: devops-ansible
├── Playbook: deploy.yml
├── Inventory: production
├── Credential: prod-ssh
└── EE: custom-ee
```

Then the user can click:

```text
Launch
```

---

# 15. Workflow Template

A **Workflow Template** allows you to chain multiple jobs together.

For example:

```text
               Workflow
                   │
                   ▼
          Provision Infrastructure
                   │
                   ▼
             Configure Servers
                   │
                   ▼
             Deploy Application
                   │
                   ▼
               Run Tests
                   │
             ┌─────┴─────┐
             │           │
          Success       Failure
             │           │
             ▼           ▼
          Notify       Rollback
```

This is very useful for CI/CD.

---

# 16. RBAC

AWX supports **Role-Based Access Control (RBAC)**.

For example:

```text
Organization
│
├── Admin
│    └── Full Access
│
├── Developer
│    └── Launch Specific Jobs
│
└── Operator
     └── Manage Inventory
```

You can control who can:

* View projects
* Modify inventories
* Launch jobs
* Manage credentials
* Administer AWX

This is one of the major reasons enterprises use Tower/AWX instead of simply giving everyone SSH access and Ansible CLI access.

---

# 17. Notifications

AWX can send notifications after jobs.

For example:

```text
Ansible Job
     │
     ├── SUCCESS
     │      │
     │      └── Slack / Email
     │
     └── FAILURE
            │
            └── Slack / Email
```

Notifications can be useful for:

* Deployment success
* Deployment failure
* Workflow completion
* Scheduled jobs
* Operational alerts

---

# 18. Complete AWX Workflow

Now let's look at the entire workflow.

Suppose you want to deploy Docker containers to three servers.

---

## Step 1 — Developer Pushes Code

```text
Developer
    │
    ▼
GitHub
```

---

## Step 2 — AWX Project Syncs

```text
GitHub
   │
   ▼
AWX Project
   │
   ▼
Playbook
```

For example:

```text
deploy.yml
```

---

## Step 3 — User Launches Job Template

```text
User
 │
 ▼
AWX UI
 │
 ▼
Job Template
```

---

## Step 4 — AWX Gathers Configuration

AWX determines:

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

## Step 5 — AWX Schedules the Job

```text
AWX Controller
      │
      ▼
Task Manager
      │
      ▼
Execution Node
```

---

## Step 6 — Execution Environment Starts

```text
Execution Node
      │
      ▼
Execution Environment
      │
      ├── Ansible
      ├── Python
      ├── Collections
      └── Dependencies
```

---

## Step 7 — Ansible Connects to Target Machines

```text
Execution Environment
          │
          │ SSH
          ├──────────────► Server 1
          │
          ├──────────────► Server 2
          │
          └──────────────► Server 3
```

---

## Step 8 — Playbook Executes

Example:

```yaml
- name: Deploy application
  hosts: webservers

  tasks:

    - name: Start container
      community.docker.docker_container:
        name: myapp
        image: nginx:latest
        state: started
```

---

## Step 9 — Results Come Back

```text
Server 1 ───┐
Server 2 ───┼──► Execution Node
Server 3 ───┘
                  │
                  ▼
                 AWX
                  │
                  ▼
              Job Output
```

---

## Step 10 — AWX Stores Job History

```text
AWX
 │
 ├── Job Status
 ├── Output
 ├── Duration
 ├── Failed Tasks
 └── Execution History
```

---

# 19. Complete AWX Architecture

Putting everything together:

```text
                         USERS
                           │
              ┌────────────┴────────────┐
              │                         │
           Web UI                     REST API
              │                         │
              └────────────┬────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │       AWX       │
                  │                 │
                  │ Web/API         │
                  │ Task Manager    │
                  │ Scheduler       │
                  └────────┬────────┘
                           │
             ┌─────────────┼──────────────┐
             │             │              │
             ▼             ▼              ▼
        PostgreSQL        Redis        Projects
             │                            │
             │                            ▼
             │                           Git
             │
             ▼
        AWX Data
             │
             │
             ▼
       Execution Nodes
       ┌──────┼──────┐
       │      │      │
       ▼      ▼      ▼
     Node1  Node2  Node3
       │      │      │
       ▼      ▼      ▼
      EE     EE     EE
       │      │      │
       └──────┼──────┘
              │
              ▼
        Ansible Engine
              │
        ┌─────┼─────┐
        ▼     ▼     ▼
     Server  Server  Server
        1      2      3
```

---

# 20. The Most Important Objects to Remember

For interviews, remember this chain:

```text
Organization
      │
      ▼
   Project
      │
      ▼
   Inventory
      │
      ▼
 Credentials
      │
      ▼
Job Template
      │
      ▼
Execution Environment
      │
      ▼
Execution Node
      │
      ▼
 Ansible Playbook
      │
      ▼
Target Servers
```

> **Important:** This is a conceptual relationship and not a strict execution sequence. A Job Template references resources such as a Project, Inventory, Credentials and Execution Environment.

---

# 21. AWX vs Ansible CLI

This is a very common interview question.

| Ansible CLI                  | AWX / Tower                |
| ---------------------------- | -------------------------- |
| Command-line based           | Web UI + API               |
| Local execution environment  | Centralized execution      |
| Manual execution             | Job Templates              |
| Basic inventory              | Centralized inventory      |
| Credentials handled manually | Credential management      |
| Limited RBAC                 | Enterprise RBAC            |
| Limited job history          | Job history                |
| No centralized dashboard     | Dashboard                  |
| Cron for scheduling          | Built-in schedules         |
| Manual workflow              | Workflow Templates         |
| Local dependencies           | Execution Environments     |
| Individual user oriented     | Team / enterprise oriented |

---

# 22. A Very Important Interview Concept

### Question

> Does AWX execute the playbook itself?

### Answer

> AWX acts as the automation control and orchestration platform. It manages projects, inventories, credentials, Job Templates, scheduling and execution. When a job is launched, AWX schedules the job onto an execution node, where the configured Execution Environment runs Ansible and connects to the target hosts.

A shorter answer:

> **AWX provides the control plane and orchestration, while the Ansible automation is executed on an execution node using the selected Execution Environment.**

---

# 23. Real-World `community.docker` Example

A practical example is the `community.docker` issue.

## Ansible CLI

```text
Ansible CLI
     │
     ▼
Local Ansible Environment
     │
     └── community.docker ✅
             │
             ▼
         Playbook
             │
             ▼
          SUCCESS
```

## AWX Before Custom Execution Environment

```text
AWX
 │
 ▼
Execution Environment
 │
 └── community.docker ❌
          │
          ▼
       Playbook
          │
          ▼
        FAILURE
```

## AWX After Custom Execution Environment

```text
Dockerfile / EE Definition
          │
          ▼
     Build Image
          │
          ▼
     Custom EE Image
          │
          ▼
      Docker Hub
          │
          ▼
        AWX
          │
          ▼
Execution Environment
          │
          ▼
       Job Template
          │
          ▼
        Playbook
          │
          ▼
        SUCCESS
```

This is exactly the kind of real-world troubleshooting scenario you can discuss in an interview.

---

# 24. Easy Way to Remember AWX Architecture

Remember these **8 things**:

```text
1. Web UI / API
       ↓
2. Project
       ↓
3. Inventory
       ↓
4. Credentials
       ↓
5. Job Template
       ↓
6. Execution Environment
       ↓
7. Execution Node
       ↓
8. Target Servers
```

And behind AWX:

```text
PostgreSQL        → Persistent AWX data
Redis             → Internal messaging / coordination
AWX               → Control / orchestration
Execution Node    → Runs automation
Execution Environment → Provides Ansible + dependencies
```

---

# 25. One-Line Explanation of Each Component

| Component             | Purpose                                       |
| --------------------- | --------------------------------------------- |
| AWX                   | Central automation and orchestration platform |
| Web UI                | Manage automation through a browser           |
| REST API              | Programmatically interact with AWX            |
| Project               | Source of Ansible playbooks                   |
| Inventory             | Defines target hosts                          |
| Credentials           | Provides secure authentication                |
| Job Template          | Defines how a playbook should run             |
| Workflow Template     | Chains multiple jobs together                 |
| Execution Environment | Container containing Ansible and dependencies |
| Execution Node        | Runs the Ansible automation                   |
| Task Manager          | Schedules and dispatches jobs                 |
| PostgreSQL            | Stores persistent AWX data                    |
| Redis                 | Internal messaging / task coordination        |
| RBAC                  | Controls user and team permissions            |
| Notifications         | Reports job / workflow results                |

---

# 26. Final Takeaway

The most important concept is:

```text
                    AWX
                     │
             Control / Orchestration
                     │
        ┌────────────┼────────────┐
        │            │            │
     Project     Inventory    Credentials
        │            │            │
        └────────────┼────────────┘
                     │
               Job Template
                     │
             Execution Environment
                     │
               Execution Node
                     │
              Ansible Engine
                     │
                     ▼
              Target Servers
```

In simple terms:

> **Ansible is the automation engine, while AWX/Tower provides centralized management, orchestration, security, scheduling, execution, monitoring and API capabilities around Ansible.**

---

# 27. Quick Interview Summary

If you need to explain AWX in **30–60 seconds** during an interview:

> **AWX is the upstream open-source automation platform for Ansible, while Ansible Automation Platform is Red Hat's enterprise offering. AWX provides a Web UI and REST API for centrally managing Ansible Projects, Inventories, Credentials, Job Templates and Workflows. When a job is launched, AWX schedules it to an Execution Node, where an Execution Environment containing Ansible and the required collections and dependencies runs the playbook against the target hosts. PostgreSQL stores persistent AWX data, while Redis is used for internal task and messaging coordination. AWX also provides RBAC, scheduling, job history, logging and notifications.**
