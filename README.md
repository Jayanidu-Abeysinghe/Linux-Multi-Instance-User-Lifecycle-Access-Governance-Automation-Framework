# Linux-Multi-Instance-User-Lifecycle-Access-Governance-Automation-Framework
This Framework is an automation system for centrally managing Linux user provisioning, SSH key lifecycle, and fine-grained ACL-based access control across multiple cloud instances. It replaces manual server management with a scalable, JSON-driven governance model featuring centralized tracking and audit logging.


This project helps you:

- Create Linux users remotely
- Remove users safely (process cleanup + home deletion)
- Update user SSH public keys
- Grant/Revoke/Update ACL-based access via permission profiles
- Sync a centralized user registry and action logs to your local machine
- Run the same operation across multiple EC2 instances from one command

---

## 📌 Architecture Overview

```text
Local Machine
  ├── remote-user-manager.sh
  ├── remote-permission-manager.sh
  ├── servers.txt
  ├── pub_keys/
  ├── permissions/
  ├── user_registry.json
  └── logs/user-actions.log
          │
          ▼
Multiple EC2 Instances
  ├── user-manager.sh
  ├── permission-manager.sh
  ├── /home/<admin>/user-onboarding/user_registry.json
  ├── /home/<admin>/user-onboarding/logs/user-actions.log
  └── ACL-protected project directories
```

---

## ✨ Key Features

- **Multi-instance execution** using `servers.txt`
- **Auto admin-user detection** (`ubuntu`, `ec2-user`, `admin`, `root`)
- **SSH key path expansion** for entries like `~/.ssh/key.pem`
- **First-time host-key safe automation** (`StrictHostKeyChecking=accept-new`)
- **Profile-driven ACL permissions** with `setfacl`
- **Registry + log synchronization** from remote to local after each run

---

## 📂 Project Structure

```text
user-onboarding/
├── remote-user-manager.sh
├── remote-permission-manager.sh
├── user-manager.sh
├── permission-manager.sh
├── servers.txt
├── user_registry.json
├── logs/
│   └── user-actions.log
├── pub_keys/
│   └── *.pub
└── permissions/
    └── *.json
```

---

## ⚙️ Prerequisites

### Local machine

- Bash (Linux / macOS / WSL / Git Bash)
- `ssh`, `scp`
- `jq`

### EC2 instances

- Ubuntu / Amazon Linux
- `sudo` privileges for the admin SSH user
- `jq` and `acl` packages installed

Ubuntu install:

```bash
sudo apt update
sudo apt install -y jq acl
```

---

## 📝 Configure `servers.txt`

Format:

```text
<ip-address> <default-user-placeholder> <private-key-path>
```

Example:

```text
13.234.38.78 ubuntu ~/.ssh/linux-user.pem
44.249.138.187 ec2-user ~/.ssh/prod-key.pem
```

Notes:

- Second column is read but not used for login selection (script auto-detects admin user).
- Private key path must exist on the local machine.
- You can add multiple lines; every command runs on all listed servers.

---

## 🔑 Public Keys

Store user public keys in `pub_keys/`.

Example:

```text
pub_keys/devops_jayanidu.pub
```

---

## 👤 User Lifecycle Commands

### 1) Add user

```bash
./remote-user-manager.sh add <username> <public_key_file>
```

Example:

```bash
./remote-user-manager.sh add devops_jayanidu ./pub_keys/jayanidu.pub
```

What happens:

1. Detects reachable admin user on each instance
2. Uploads `user-manager.sh` + target public key
3. Creates Linux user and `authorized_keys`
4. Updates remote registry/log
5. Syncs registry/log back to local

### 2) Remove user

```bash
./remote-user-manager.sh remove <username>
```

### 3) Update user SSH key

```bash
./remote-user-manager.sh update <username> <new_public_key_file>
```

---

## 🔐 Permission Commands

### 1) Grant permissions

```bash
./remote-permission-manager.sh grant <username> <profile1> [profile2 ...]
```

### 2) Revoke permissions

```bash
./remote-permission-manager.sh revoke <username> <profile1> [profile2 ...]
```

### 3) Update permissions (revoke + grant)

```bash
./remote-permission-manager.sh update <username> <profile1> [profile2 ...]
```

Important:

- Profiles must exist locally as `permissions/<profile>.json`.
- Missing profiles are skipped, and command now fails if **none** of the requested profiles are valid.

---

## 📜 Permission Profile Format

Path: `permissions/<profile>.json`

Example:

```json
[
  {
    "path": "/home/ubuntu/folder1",
    "access": ["read"]
  },
  {
    "path": "/home/ubuntu/project-api",
    "access": ["read", "write"]
  }
]
```

Supported access levels:

- `read`
- `write`

---

## 📊 Registry and Logs

### Local synced files

- `user_registry.json`
- `logs/user-actions.log`

### Remote source files

- `/home/<admin>/user-onboarding/user_registry.json`
- `/home/<admin>/user-onboarding/logs/user-actions.log`

Registry example:

```json
[
  {
    "username": "devops_jayanidu",
    "pubkey": "/tmp/jayanidu.pub",
    "created": "2026-03-24_07:35:20",
    "permissions": [
      {
        "area": "folder1",
        "level": ["read"]
      }
    ]
  }
]
```

---

## 🧠 How Multi-Instance Execution Works

Both remote scripts iterate over every server entry using:

```bash
done < servers.txt
```

So one command applies to all configured instances.

---

## 🧪 Quick End-to-End Example

```bash
# 1) Add user
./remote-user-manager.sh add devops_jayanidu ./pub_keys/jayanidu.pub

# 2) Grant folder access profile
./remote-permission-manager.sh grant devops_jayanidu folder1

# 3) Revoke access later
./remote-permission-manager.sh revoke devops_jayanidu folder1

# 4) Rotate SSH key
./remote-user-manager.sh update devops_jayanidu ./pub_keys/new_key.pub

# 5) Remove user
./remote-user-manager.sh remove devops_jayanidu
```

---

## 🚨 Troubleshooting

### `Could not detect a valid admin user`

- Verify instance IP in `servers.txt`
- Verify private key path and file permissions
- Test manually:

```bash
ssh -i ~/.ssh/linux-user.pem ubuntu@<instance-ip>
```

### `SSH key not found for <ip>`

- Fix private key path in `servers.txt`
- Use full path or `~/.ssh/<key>.pem`

### `Profile <name>.json not found locally`

- Create `permissions/<name>.json` before running grant/revoke/update

### User added but no permissions applied

- Ensure profile JSON has valid entries with existing remote paths
- Ensure `acl` package is installed on EC2

---

## 🔒 Security Notes

- SSH key-based access only (no password automation)
- Fine-grained ACL controls with explicit read/write scopes
- Parent traversal handling for directory access
- Action logging and registry syncing for auditability

---

## 📧 Contact

For questions/support:

- Email: amjayniducodeolima@gmail.com

