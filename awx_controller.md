# AWX + Ansible + Docker Deployment

## 1. Project Overview

This project demonstrates how to use **AWX** to automate server configuration and Docker application deployment using **Ansible**.

The project uses:

* AWX
* Ansible
* Ansible Execution Environments
* Ansible `community.docker` collection
* Docker
* Docker Hub
* GitHub
* Ubuntu EC2 instances
* SSH-based Ansible connectivity

The final workflow is:

```text
                         GitHub
                            |
                            v
                       AWX Project
                            |
                            v
                      Job Template
                            |
                            v
              Custom Execution Environment
              ebhi456/awx-ee-community-docker
                            |
                            v
                         Ansible
                            |
                  SSH to worker servers
                       /          \
                      /            \
                     v              v
                 Worker 1       Worker 2
                     |              |
                     v              v
                   Docker         Docker
                     |              |
                     v              v
                  Nginx          Nginx
```

---

# 2. Project Objectives

The main objectives were:

1. Configure worker servers using Ansible.
2. Install Docker on the worker servers.
3. Verify Docker installation.
4. Deploy an Nginx Docker container using Ansible.
5. Store the Ansible project in GitHub.
6. Configure AWX to use the GitHub repository.
7. Understand AWX Execution Environments.
8. Create a custom Execution Environment containing `community.docker`.
9. Push the custom Execution Environment image to Docker Hub.
10. Configure AWX to use the custom Execution Environment.
11. Successfully deploy the Docker application through AWX.

---

# 3. Infrastructure

The environment contains one control node and two worker nodes.

```text
Control Node
--------------------------------
Private IP: 10.0.4.167
OS: Ubuntu
Purpose:
- Ansible control node
- AWX host
- Docker
- kubectl
- ansible-builder


Worker 1
--------------------------------
Private IP: 10.0.6.152
Hostname: ip-10-0-6-152
OS: Ubuntu
Purpose:
- Docker host
- Nginx container


Worker 2
--------------------------------
Private IP: 10.0.7.198
Hostname: ip-10-0-7-198
OS: Ubuntu
Purpose:
- Docker host
- Nginx container
```

---

# 4. Project Directory Structure

The Ansible project was organized as follows:

```text
~/ansible/
│
├── ansible.cfg
├── inventory
│
└── awx-demo/
    ├── server_setup.yml
    └── docker_deploy.yml
```

The custom Execution Environment was created separately:

```text
~/ansible/awx-ee/
│
├── execution-environment.yml
├── collections/
│
├── context/
│
└── .venv/
```

---

# 5. Configure Ansible Control Node

The Ansible configuration file was:

```bash
cat ~/ansible/ansible.cfg
```

Example:

```ini
[defaults]
inventory = /home/ubuntu/ansible/inventory
private_key_file = /home/ubuntu/new_ssh_key
host_key_checking = False
```

The inventory contained the worker servers.

Example:

```ini
[workers]
worker1 ansible_host=10.0.6.152 ansible_user=ubuntu
worker2 ansible_host=10.0.7.198 ansible_user=ubuntu
```

---

# 6. SSH Connectivity

Initially, Ansible could not connect to the worker servers.

The error was:

```text
Permission denied (publickey)
```

Example:

```text
worker1 | UNREACHABLE!
ubuntu@10.0.6.152: Permission denied (publickey).

worker2 | UNREACHABLE!
ubuntu@10.0.7.198: Permission denied (publickey).
```

The problem was related to the SSH private key being unavailable from the location where the command was being executed.

For example, running this from a worker server:

```bash
ssh -i ~/new_ssh_key ubuntu@10.0.7.198
```

resulted in:

```text
Warning: Identity file /home/ubuntu/new_ssh_key not accessible:
No such file or directory.
```

The important lesson was:

> The private SSH key must exist on the Ansible control node and Ansible must be able to access it.

After correcting the SSH configuration/key availability, connectivity was verified with:

```bash
ansible all -i ~/ansible/inventory -m ping
```

Expected result:

```text
worker1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

worker2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

# 7. Server Configuration Playbook

The first playbook configured the worker servers.

File:

```text
awx-demo/server_setup.yml
```

The playbook performed tasks such as:

* Gathering facts
* Displaying hostname
* Updating apt cache
* Installing required packages
* Installing Docker
* Starting Docker
* Adding the Ubuntu user to the Docker group
* Verifying Docker
* Checking server connectivity

The playbook was executed from the control node:

```bash
ansible-playbook -i ~/ansible/inventory ~/ansible/awx-demo/server_setup.yml
```

The playbook completed successfully:

```text
PLAY RECAP

worker1 : ok=10 changed=4 unreachable=0 failed=0
worker2 : ok=10 changed=4 unreachable=0 failed=0
```

Docker version displayed:

```text
Docker version 29.1.3
```

---

# 8. Verify Docker on Worker Nodes

Docker was also verified directly on the workers.

Example:

```bash
ssh worker1
docker --version
docker ps
```

and:

```bash
ssh worker2
docker --version
docker ps
```

This confirmed that Docker was installed and running.

---

# 9. Docker Deployment Playbook

The second playbook was created:

```text
awx-demo/docker_deploy.yml
```

Its purpose was to deploy Nginx using Docker.

The playbook used the Ansible collection:

```text
community.docker
```

The deployment performed:

1. Pull Nginx image.
2. Run Nginx container.
3. Verify the container.
4. Display container status.

The playbook was tested from the Ansible CLI:

```bash
ansible-playbook -i ~/ansible/inventory ~/ansible/awx-demo/docker_deploy.yml
```

It successfully deployed Nginx to both worker nodes.

Example result:

```text
TASK [Pull Nginx Docker image]

changed: [worker1]
changed: [worker2]


TASK [Run Nginx container]

changed: [worker1]
changed: [worker2]


TASK [Verify Nginx container]

ok: [worker1]
ok: [worker2]
```

The resulting container was:

```text
CONTAINER ID   IMAGE          STATUS       PORTS
xxxxxxxxxxxx   nginx:latest   Up           0.0.0.0:8080->80/tcp
```

Container name:

```text
ansible-nginx
```

---

# 10. Verify community.docker Collection

The Docker deployment playbook uses modules from:

```text
community.docker
```

On the Ansible control node:

```bash
ansible-galaxy collection list
```

showed:

```text
Collection                               Version
---------------------------------------- -------
community.docker                         5.2.2
community.library_inventory_filtering_v1 1.1.5
```

Therefore, the playbook worked correctly when executed directly through the Ansible CLI.

---

# 11. Configure AWX

AWX was deployed in Kubernetes.

The AWX namespace was:

```text
awx
```

The AWX resources were verified:

```bash
kubectl get pods -n awx
```

Example:

```text
awx-demo-postgres-15-0
awx-demo-task-xxxxxxxxxx
awx-demo-web-xxxxxxxxxx
awx-operator-controller-manager-xxxxxxxxxx
```

AWX resource:

```bash
kubectl get awx -n awx
```

Result:

```text
NAME
awx-demo
```

---

# 12. Configure GitHub Repository in AWX

The Ansible project was already stored in GitHub.

The repository contained:

```text
awx-demo/
│
├── server_setup.yml
└── docker_deploy.yml
```

The AWX Project was configured to use the Git repository.

AWX periodically updates the project source tree.

During project synchronization, AWX displayed information such as:

```text
Update source tree if necessary

TASK [Update project using git]

changed: [localhost]

TASK [Set the git repository version]

ok: [localhost]

TASK [Repository Version]

Repository Version 566b3573d0b978f00029bbafa1b7a717e50178f8
```

This confirmed that AWX successfully pulled the GitHub repository.

---

# 13. First AWX Docker Deployment Failure

When the Docker deployment playbook was executed through AWX, it failed.

The error was:

```text
ERROR! couldn't resolve module/action 'community.docker.docker_image'.
```

This was confusing because the playbook worked from the CLI.

The reason was an important AWX concept:

> The Ansible CLI and AWX do not necessarily use the same execution environment.

---

# 14. Why CLI Worked but AWX Failed

On the control node:

```bash
ansible-galaxy collection list
```

showed:

```text
community.docker 5.2.2
```

Therefore:

```text
Ansible CLI
    |
    +-- community.docker
    |
    +-- docker_deploy.yml
    |
    +-- SUCCESS
```

However, AWX executes jobs inside an **Execution Environment**.

The AWX job was initially using:

```text
AWX EE (24.6.1)
```

The AWX Execution Environment did not contain the required `community.docker` collection in the environment being used for the job.

Therefore:

```text
AWX
 |
 +-- AWX EE
       |
       X community.docker
       |
       X docker_deploy.yml
```

Result:

```text
couldn't resolve module/action
'community.docker.docker_image'
```

---

# 15. Important Concept: Execution Environment

An AWX Execution Environment is essentially a container image containing the tools required to execute Ansible jobs.

It can contain:

```text
Ansible
Python
Ansible Collections
Python packages
System packages
Other required dependencies
```

The worker nodes do NOT need the `community.docker` collection.

This is important.

The collection is required where the Ansible playbook executes.

The architecture is:

```text
AWX
 |
 +-- Execution Environment
       |
       +-- Ansible
       +-- community.docker
       |
       +-- SSH
             |
             +-- Worker 1
             |
             +-- Worker 2
```

The workers only need Docker because Docker is the target application runtime.

---

# 16. Create Custom Execution Environment

Because the standard AWX EE did not contain the required collection, a custom EE was created.

Directory:

```bash
mkdir -p ~/ansible/awx-ee
cd ~/ansible/awx-ee
```

A Python virtual environment was created:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

`ansible-builder` was installed inside the virtual environment.

After installation:

```bash
ansible-builder --version
```

Result:

```text
3.1.1
```

---

# 17. Download Required Ansible Collections

The required collection was:

```text
community.docker:5.2.2
```

It was downloaded using:

```bash
ansible-galaxy collection download community.docker:5.2.2
```

The download also included:

```text
community.library_inventory_filtering_v1:1.1.5
```

The collection archives were stored under:

```text
collections/
```

Example:

```text
collections/
├── community-docker-5.2.2.tar.gz
├── community-library_inventory_filtering_v1-1.1.5.tar.gz
└── requirements.yml
```

---

# 18. Important Collection Requirements Lesson

During the build process, an incorrect requirements file caused errors.

Ansible Galaxy collection requirements must use valid collection names such as:

```yaml
collections:
  - name: community.docker
    version: "5.2.2"
```

They should not incorrectly use an archive filename as the collection name:

```yaml
# Incorrect
- name: community-docker-5.2.2.tar.gz
```

The valid collection name is:

```text
community.docker
```

---

# 19. Custom Execution Environment Configuration

The final Execution Environment configuration was based on the required Ansible collection.

The goal was to create an image containing:

```text
community.docker 5.2.2
```

The resulting image was:

```text
awx-ee-community-docker:1.0
```

---

# 20. Build the Custom Execution Environment

The image was built using:

```bash
ansible-builder build --tag awx-ee-community-docker:1.0
```

The build initially encountered several problems, including:

* Docker socket permission problems
* Incorrect collection requirements
* Local collection archive path problems
* Build context problems
* Galaxy dependency resolution problems

These were corrected by ensuring the collection archives were included correctly in the build context and that the generated Dockerfile installed the collection from the correct location.

---

# 21. Verify Custom Execution Environment

After successful build:

```bash
docker images | grep awx-ee-community-docker
```

Result:

```text
awx-ee-community-docker:1.0
```

The image was approximately:

```text
1.19GB
```

The most important verification was:

```bash
docker run --rm awx-ee-community-docker:1.0 \
  ansible-galaxy collection list | grep community.docker
```

Result:

```text
community.docker  5.2.2
```

This confirmed that the custom Execution Environment contained the required collection.

---

# 22. Push Custom EE to Docker Hub

Instead of using Amazon ECR, Docker Hub was selected for learning and demonstration purposes.

The image was tagged:

```bash
docker tag awx-ee-community-docker:1.0 \
  ebhi456/awx-ee-community-docker:1.0
```

Then pushed:

```bash
docker push ebhi456/awx-ee-community-docker:1.0
```

The push completed successfully.

The image digest was:

```text
sha256:ca6a881abcca031c03b6cf8728fe1b681ddaf0bae44154c158d62ea807597a1d
```

---

# 23. Verify Docker Hub Image

The image was pulled again:

```bash
docker pull ebhi456/awx-ee-community-docker:1.0
```

Docker confirmed:

```text
Status: Image is up to date
```

The collection was verified again:

```bash
docker run --rm ebhi456/awx-ee-community-docker:1.0 \
  ansible-galaxy collection list | grep community.docker
```

Result:

```text
community.docker  5.2.2
```

---

# 24. Verify Ansible Version in the Custom EE

The image was also inspected:

```bash
docker run --rm ebhi456/awx-ee-community-docker:1.0 \
  ansible --version
```

This showed the Ansible version packaged inside the custom image.

This demonstrated an important principle:

> An AWX Execution Environment has its own Ansible runtime and dependencies.

The host's Ansible installation is not automatically used by the AWX job.

---

# 25. Configure Custom EE in AWX

In AWX:

```text
Administration
    |
    +-- Execution Environments
            |
            +-- Add
```

A custom Execution Environment was configured using the Docker Hub image:

```text
ebhi456/awx-ee-community-docker:1.0
```

The standard:

```text
AWX EE (24.6.1)
```

was replaced for the Docker deployment job with the custom EE.

---

# 26. Update Job Template

The Docker deployment Job Template was configured to use:

```text
AWX EE Community Docker
```

instead of:

```text
AWX EE (24.6.1)
```

The important configuration became:

```text
Job Template
     |
     +-- Project: GitHub Ansible Project
     |
     +-- Inventory: Worker Inventory
     |
     +-- Execution Environment:
             AWX EE Community Docker
```

---

# 27. Successful AWX Deployment

After changing the Job Template to the custom Execution Environment, the Docker deployment was executed again.

AWX successfully started the job.

The output included:

```text
Identity added:
/runner/artifacts/11/ssh_key_data
```

Then:

```text
PLAY [Deploy Docker Application]
```

The worker servers successfully gathered facts:

```text
TASK [Gathering Facts]

ok: [worker1]
ok: [worker2]
```

The Nginx deployment then proceeded successfully.

The previous error:

```text
couldn't resolve module/action
'community.docker.docker_image'
```

was resolved.

---

# 28. Final Architecture

The final working architecture is:

```text
                         +----------------+
                         |     GitHub     |
                         | Ansible Repo   |
                         +-------+--------+
                                 |
                                 |
                                 v
                         +-------+--------+
                         |      AWX       |
                         |                |
                         | Project        |
                         | Job Template   |
                         +-------+--------+
                                 |
                                 v
                    +------------+-------------+
                    | Custom Execution        |
                    | Environment             |
                    |                         |
                    | Ansible                 |
                    | community.docker 5.2.2  |
                    +------------+-------------+
                                 |
                                 |
                              SSH
                         /           \
                        /             \
                       v               v
              +--------+------+ +------+--------+
              |   Worker 1    | |    Worker 2   |
              | 10.0.6.152    | | 10.0.7.198    |
              |               | |                |
              | Docker        | | Docker         |
              | Nginx         | | Nginx          |
              +---------------+ +----------------+
```

---

# 29. Files Used in the Project

## Ansible Configuration

```text
ansible.cfg
```

## Inventory

```text
inventory
```

## Server Configuration

```text
awx-demo/server_setup.yml
```

Purpose:

```text
Configure worker servers
Install Docker
Start Docker
Verify Docker
```

## Docker Deployment

```text
awx-demo/docker_deploy.yml
```

Purpose:

```text
Pull Nginx image
Run Nginx container
Verify container
Display container status
```

## Custom Execution Environment

```text
awx-ee/execution-environment.yml
```

Purpose:

```text
Define dependencies required by the custom AWX EE.
```

---

# 30. Useful Commands

## Test Ansible Connectivity

```bash
ansible all -i ~/ansible/inventory -m ping
```

## Run Server Configuration

```bash
ansible-playbook \
  -i ~/ansible/inventory \
  ~/ansible/awx-demo/server_setup.yml
```

## Run Docker Deployment

```bash
ansible-playbook \
  -i ~/ansible/inventory \
  ~/ansible/awx-demo/docker_deploy.yml
```

## List Collections

```bash
ansible-galaxy collection list
```

## Check Docker

```bash
docker ps
```

## Check Worker Docker Containers

```bash
ssh worker1 docker ps
ssh worker2 docker ps
```

## Check AWX Pods

```bash
kubectl get pods -n awx
```

## Check AWX Resource

```bash
kubectl get awx -n awx
```

## Check AWX Task Deployment

```bash
kubectl get deployment -n awx awx-demo-task
```

---

# 31. Troubleshooting

## Issue 1: SSH Permission Denied

Error:

```text
Permission denied (publickey)
```

Cause:

The Ansible control node did not have access to the correct private SSH key.

Solution:

Verify:

```bash
ls -l /home/ubuntu/new_ssh_key
```

Test:

```bash
ssh -i /home/ubuntu/new_ssh_key \
  -o IdentitiesOnly=yes \
  ubuntu@10.0.6.152
```

and:

```bash
ssh -i /home/ubuntu/new_ssh_key \
  -o IdentitiesOnly=yes \
  ubuntu@10.0.7.198
```

Then test:

```bash
ansible all -i ~/ansible/inventory -m ping
```

---

# 32. Issue 2: community.docker Not Found in AWX

Error:

```text
couldn't resolve module/action
'community.docker.docker_image'
```

Cause:

The Ansible CLI had the collection installed, but the AWX Execution Environment did not.

Solution:

Create a custom Execution Environment containing:

```text
community.docker
```

Then configure the AWX Job Template to use the custom EE.

---

# 33. Issue 3: Worker Does Not Need community.docker

The `community.docker` collection does not need to be installed on the worker nodes.

The collection is used by Ansible on the execution side.

The worker only needs:

```text
Docker
SSH access
Python
```

Architecture:

```text
AWX Custom EE
      |
      | community.docker
      |
      | SSH
      v
Worker
      |
      | Docker Engine
      v
Container
```

---

# 34. Issue 4: Docker Socket Permission

While building the Execution Environment, Docker returned:

```text
permission denied while trying to connect to the Docker API
at unix:///var/run/docker.sock
```

The user running the build did not have Docker group membership.

The groups were checked:

```bash
groups
```

The user was added to the Docker group:

```bash
sudo usermod -aG docker ubuntu
```

A new login/session was required before the group membership became active.

Verify:

```bash
groups
```

Expected:

```text
ubuntu ... docker
```

Then:

```bash
docker ps
```

should work without `sudo`.

---

# 35. Issue 5: Collection Archive Build Problems

The custom EE build initially failed because collection archive files were not available in the Docker build context.

The important lesson is:

> Files referenced by `COPY` in a Dockerfile must exist inside the Docker build context and must not be excluded by `.dockerignore`.

The final build successfully included the collection archive and installed:

```text
community.docker 5.2.2
```

---

# 36. Key DevOps Concepts Learned

## Ansible Control Node

The machine where Ansible commands/playbooks are executed.

```text
Control Node
     |
     +-- Ansible
     +-- Inventory
     +-- SSH keys
```

## Managed Nodes

Servers controlled by Ansible.

```text
Worker 1
Worker 2
```

## Ansible Collection

Collections provide modules, plugins and other Ansible content.

Example:

```text
community.docker
```

## Execution Environment

A container image containing the runtime and dependencies required to execute an Ansible job.

## AWX Project

Connects AWX to the source repository.

## AWX Job Template

Defines how an Ansible job should be executed.

## AWX Workflow

Can chain multiple Job Templates together.

---

# 37. CLI vs AWX Execution

One of the most important lessons from this project:

### CLI

```text
Ansible CLI
    |
    +-- Host Ansible installation
    |
    +-- Host collections
    |
    +-- community.docker
    |
    +-- Playbook
```

### AWX

```text
AWX
 |
 +-- Job Template
       |
       +-- Execution Environment
              |
              +-- Ansible
              +-- Collections
              +-- Python
              +-- Dependencies
              |
              +-- Playbook
```

Therefore:

> A playbook working from the Ansible CLI does not automatically guarantee that it will work inside AWX.

The AWX Execution Environment must contain all required dependencies.

---

# 38. Why Custom Execution Environments Are Useful

Custom EEs are useful when a project requires:

* Specific Ansible versions
* Specific Ansible collections
* Python libraries
* System packages
* Cloud SDKs
* Docker modules
* Kubernetes modules
* Terraform-related tooling
* Database libraries
* Organization-specific automation dependencies

For example:

```text
Custom EE
│
├── Ansible
├── community.docker
├── kubernetes.core
├── amazon.aws
├── Python libraries
└── Other dependencies
```

This makes AWX jobs reproducible.

---

# 39. Docker Hub Usage

Docker Hub was used to store the custom Execution Environment image:

```text
ebhi456/awx-ee-community-docker:1.0
```

The advantage is that AWX can pull the same image whenever the Job Template executes.

This is useful in real environments because the Execution Environment becomes a versioned artifact.

Example:

```text
awx-ee-community-docker:1.0
awx-ee-community-docker:1.1
awx-ee-community-docker:2.0
```

This also makes rollback easier.

---

# 40. Current Project Status

The following items have been successfully completed:

```text
[✓] Ubuntu control node
[✓] Worker 1 configured
[✓] Worker 2 configured
[✓] SSH connectivity
[✓] Ansible inventory
[✓] Ansible configuration
[✓] Docker installation
[✓] Docker service configuration
[✓] Docker verification
[✓] Nginx deployment using Ansible
[✓] GitHub repository
[✓] AWX installation
[✓] AWX Project
[✓] AWX Job Template
[✓] Identified AWX EE dependency issue
[✓] Created custom Execution Environment
[✓] Added community.docker
[✓] Built custom EE
[✓] Verified community.docker in custom EE
[✓] Pushed custom EE to Docker Hub
[✓] Added custom EE to AWX
[✓] Updated Job Template
[✓] Successful Docker deployment through AWX
```

---

# 41. Recommended Next Phase

The next phase can make this project more production-like.

Recommended workflow:

```text
GitHub
   |
   v
GitHub Actions
   |
   +-- Validate Ansible
   +-- Run tests
   +-- Build application image
   +-- Push image to Docker Hub
   |
   v
AWX
   |
   +-- Workflow
   |
   +-- Server Configuration
   |
   +-- Docker Deployment
   |
   +-- Deployment Verification
```

The AWX workflow can eventually become:

```text
START
  |
  v
Server Configuration
  |
  | SUCCESS
  v
Docker Deployment
  |
  | SUCCESS
  v
Application Verification
  |
  | SUCCESS
  v
END
```

This would provide a stronger real-world demonstration of:

```text
Git
GitHub
GitHub Actions
Ansible
AWX
Execution Environments
Docker
Docker Hub
Linux
CI/CD
Infrastructure Automation
```

---

# 42. Interview Explanation

A concise explanation of this project for an interview:

> I implemented an AWX-based Ansible automation setup where AWX pulls playbooks from GitHub and executes them against Ubuntu worker nodes. Initially, the Docker deployment worked from the Ansible CLI but failed in AWX because the AWX Execution Environment did not contain the `community.docker` collection. I created a custom Ansible Execution Environment using ansible-builder, included the required collection, built and tested the image, and pushed it to Docker Hub. I then configured AWX to use the custom Execution Environment in the Job Template. After that, AWX successfully executed the Docker deployment playbook and deployed Nginx containers across both worker nodes.

---

# 43. Final Outcome

The project successfully demonstrates:

```text
Source Control
      |
      v
    GitHub
      |
      v
     AWX
      |
      v
Execution Environment
      |
      v
    Ansible
      |
      v
   SSH Automation
      |
      +-------------+
      |             |
      v             v
  Worker 1       Worker 2
      |             |
      v             v
   Docker         Docker
      |             |
      v             v
    Nginx         Nginx
```

The most important lesson is that **AWX Execution Environments provide the runtime dependencies for Ansible jobs**. Installing an Ansible collection on the control node is not sufficient when AWX executes the playbook inside a separate Execution Environment.
