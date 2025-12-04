
# 🛠️ How to Install Ansible Core/CLI on Red Hat Linux

## 👤 User Setup: Admin Access Required
To configure the Ansible Control Server and Managed Servers, you'll need a user with administrative privileges.  
In this example, we’ll use **ansible** as the username.

### ➕ Add User to Admin Group
The default admin group in Red Hat Linux is **wheel**:

```bash
sudo usermod -aG wheel ansible
```

### 🔍 Check Users in the Wheel Group
```bash
groups | grep wheel
```

---

## 🖥️ Terminal Commands: Installing Ansible Core
Open a terminal on your Red Hat Linux system and follow these steps:

### ✅ Step 1: Update Your System
```bash
sudo dnf update -y
```

### 📦 Step 2: Install Ansible Core
```bash
sudo dnf install -y ansible-core
```

### 🔍 Step 3: Verify Installation
```bash
ansible --version
```

### 🧪 Step 4: Test Ansible
```bash
ansible localhost -m ping
```

---

## 🔐 Step 5: Set Up SSH & Generate SSH Key
Generate an SSH key on the Ansible Control Server:

```bash
ssh-keygen -t rsa -b 4096 -C "ansible@yourdomain"
```

Press Enter to accept the default location: `~/.ssh/id_rsa`  
Replace `"ansible@yourdomain"` with your actual username and Control Server IP.

---

## 📤 Step 6: Copy SSH Key to Managed Servers
```bash
ssh-copy-id ansible@yourdomainTarget
```

Replace `"ansible@yourdomainTarget"` with the username and IP of your Managed Server.

---

## 🔗 Step 7: Test SSH Connection
```bash
ssh ansible@yourdomainTarget
```

---

# 🚀 Running Your First Ansible Playbook

## 🗂️ Edit the Hosts File
Use nano to edit the default hosts file:

```bash
nano /etc/ansible/hosts
```

Add your server group at the end:

```ini
[webservers]
192.168.56.10
192.168.56.11
192.168.56.12
```

*(These IPs are examples—replace with your actual server IPs.)*

### ✅ Verify Connectivity
```bash
ansible all -i hosts.ini -m ping
```

---

## 📁 Create a Directory for Playbooks
```bash
mkdir ~/ansible-playbooks
cd ~/ansible-playbooks
```

---

## 📝 Create a Playbook
```bash
touch update-rhel.yml
nano update-rhel.yml
```

Paste the following YAML content:

```yaml
---
- name: Update packages on Red Hat servers
  hosts: webservers
  become: true

  tasks:
    - name: Make sure system is up to date
      yum:
        name: '*'
        state: latest
```

*(The `hosts` value can be changed to match any group defined in your hosts file.)*

---

## ▶️ Run the Playbook
```bash
ansible-playbook update-rhel.yml
```
```

