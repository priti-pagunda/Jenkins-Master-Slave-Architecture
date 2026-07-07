# Master-Slave Architecture using Jenkins

A Jenkins CI/CD lab demonstrating the **master-slave (controller-agent)** architecture. A Jenkins **Controller** (running on an AWS EC2 Ubuntu instance) delegates a build job over SSH to a Jenkins **Agent** node, which clones and builds a Spring Boot Java project.

> **Note:** Jenkins now officially calls this the **Controller-Agent** model rather than Master-Slave. The repo title keeps the classic terminology for lab/assignment purposes.

## 🏗️ Architecture

```
                ┌─────────────────────┐
                │   Jenkins Controller │
                │   (Built-In Node)    │
                │   EC2 - Ubuntu       │
                └──────────┬───────────┘
                           │ SSH (port 22)
                           │
                 ┌─────────▼─────────┐
                 │   Agent: node-1   │
                 │   EC2 - Ubuntu    │
                 │   Label: node-1   │
                 └───────────────────┘
```

## ⚙️ Prerequisites

- 2x Ubuntu EC2 instances (ap-south-1) — one for the Jenkins Controller, one as the Agent
- Jenkins installed on the Controller (`Jenkins 2.555.2`)
- Java (OpenJDK 21) installed on the Agent node
- Git installed on the Agent node
- SSH access (port 22) open between Controller and Agent security groups

## 🔌 Plugins Used

Installed via **Manage Jenkins → Plugins**:

| Plugin | Purpose |
|---|---|
| SSH Build Agents | Launches agents over SSH using Jenkins' Java-based SSH implementation |
| SSH Credentials Plugin | Stores SSH credentials (username + private key) in Jenkins |
| Apache Mina SSHD (Core & Common) | Underlying SSH library used by the above plugins |

<img width="1920" height="1080" alt="01-ssh-plugins-installed" src="https://github.com/user-attachments/assets/76548edc-deb0-48c0-b568-0fe202c1fef4" />


## 🔧 Setup Steps (as performed)

### 1. Generate SSH Key Pair
On the controller (or locally), generate a key in PEM format to avoid `libcrypto` key-format errors with newer OpenSSH keys:
```bash
ssh-keygen -t rsa -b 4096 -m PEM -f node-1-key
```
Add the **public key** to `~/.ssh/authorized_keys` on the `node-1` agent server.

### 2. Add SSH Credential in Jenkins
**Manage Jenkins → Credentials → System → Global credentials → Add Credentials**
- Kind: `SSH Username with private key`
- Scope: `Global`
- ID: `node-1-key`
- Description: `node-1-key`
- Username: `ubuntu`
- Private Key: **Enter directly** → paste the full private key

<img width="1920" height="1080" alt="02-add-ssh-credential" src="https://github.com/user-attachments/assets/2f93fab2-d419-4134-abcb-770934b4e7d7" />


### 3. Create the Agent Node
**Manage Jenkins → Nodes → New Node**
- Name: `node-1`
- Description: `This node will perform a git clone operation`
- Number of executors: `1`
- Remote root directory: `/home/ubuntu`
- Labels: `node-1`
- Launch method: Launch agents via SSH
  - Host: (agent's EC2 public IP)
  - Credentials: `node-1-key`
  - Host Key Verification Strategy: `Known hosts file Verification Strategy`

<img width="1920" height="1080" alt="03-configure-node-1" src="https://github.com/user-attachments/assets/0cc850f7-8668-47fa-969a-253810b700c2" />


Once saved, `node-1` appears in **Manage Jenkins → Nodes** alongside the Built-In Node and connects successfully.

<img width="1920" height="1080" alt="04-nodes-list" src="https://github.com/user-attachments/assets/152e3aaf-48ac-4f7b-abe1-d67595945618" />


### 4. Create the Job
Job name: `pull-job-on-node-1`
- General → Description: `It will clone all the project files on the node`
- General → ✅ **Restrict where this project can be run** → Label Expression: `node-1`
- Source Code Management → Git → Repository URL:
  `https://github.com/iamtruptimane/springboot-java-project.git`

<img width="1920" height="1080" alt="05-job-restrict-node-1" src="https://github.com/user-attachments/assets/f32e3055-1876-4720-8244-23873fc07380" />


### 5. Build and Verify
Run the job. Console output confirms the repo is cloned directly onto `node-1`:
```
Building remotely on node-1 in workspace /home/ubuntu/workspace/pull-job-on-node-1
Cloning repository https://github.com/iamtruptimane/springboot-java-project.git
Checking out Revision 870db26f9b7f12ac10e4cae488df1f7bb9fad1d7 (refs/remotes/origin/main)
Commit message: "added readme files"
Finished: SUCCESS
```

<img width="1920" height="1080" alt="06-console-output-success" src="https://github.com/user-attachments/assets/a2a6431a-b0f4-44d9-ba79-6bf2aab36aa8" />


On the agent (`node-1`) terminal, the cloned project is visible under the workspace:
```bash
ubuntu@node-1:~/workspace/pull-job-on-node-1$ ls
README.md  img  mvnw  mvnw.cmd  pom.xml  src
```

<img width="1920" height="1080" alt="08-agent-terminal-cloned-files" src="https://github.com/user-attachments/assets/1d42c2bb-09ba-42d9-8ad3-1e30ac826ec1" />
<img width="1920" height="1080" alt="07-agent-terminal-workspace" src="https://github.com/user-attachments/assets/c3b343d0-934d-4d50-b643-5479714c9b9a" />


## 🐞 Troubleshooting

**`libcrypto` key format error in the SSH agent launcher:**
Newer `ssh-keygen` defaults to the OpenSSH key format, which some Jenkins SSH agent implementations can't parse. Regenerate the key in PEM format instead:
```bash
ssh-keygen -t rsa -b 4096 -m PEM
```
Then re-create the credential in Jenkins (**Enter directly**, pasting the full key including the `-----BEGIN`/`-----END` lines) rather than editing the existing one.

## 📁 Repo Structure

```
.
├── README.md
    └── screenshots/
        ├── 01-ssh-plugins-installed.png
        ├── 02-add-ssh-credential.png
        ├── 03-configure-node-1.png
        ├── 04-nodes-list.png
        ├── 05-job-restrict-node-1.png
        ├── 06-console-output-success.png
        ├── 07-agent-terminal-workspace.png
        └── 08-agent-terminal-cloned-files.png
```

✅ Project Outcome

Objective: Set up a Jenkins Master-Slave (Controller-Agent) architecture where the Jenkins controller delegates build jobs to a remote agent node over SSH.
Result: Successfully achieved

Jenkins Controller running on EC2 (Ubuntu)✅ Done<br>
SSH plugins installed (SSH Build Agents, SSH Credentials, Mina SSHD)✅ Done<br>
SSH key format issue (libcrypto error) resolved via PEM-format key regeneration✅ Resolved<br>
SSH credential (node-1-key) added to Jenkins✅ Done<br>
Agent node node-1 configured and connected via SSH✅<br>
ConnectedJob pull-job-on-node-1 created, restricted to run only on node-1✅ Done<br>
Job executed a Git clone of springboot-java-project remotely on the agent✅ Build #1 — SUCCESS<br>
Verified cloned files physically present in agent's workspace✅ Confirmed

