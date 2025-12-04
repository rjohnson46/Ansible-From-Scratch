# 🛠 Ansible/AWX Installation & Removal Guide

**Platform:** Red Hat Enterprise Linux (RHEL)  
**Tools:** Ansible Core, Kubernetes, AWX Operator

---

## 📋 Requirements
- RHEL system with subscription enabled  
- `ansible` user created and added to `sudo`, `wheel`, or `admins` group  
- `/etc/sudoers` configured for passwordless sudo access  
- Python 3 and pip installed  
- Kubernetes cluster available  

---

## 🧰 Install Ansible Core on RHEL

### ✅ Prerequisites
- RHEL 8 or 9 system  
- Sudo privileges  
- Subscription manager configured  

### 📦 Installation Steps
```bash
# Step 1: Update system
sudo dnf update -y

# Step 2: Install Ansible Core from AppStream
sudo dnf install -y ansible-core

# Step 3: Verify installation
ansible --version
```

---

## 🚀 AWX Installation Playbook

### 📄 Create Playbook File
```bash
touch awx-install.yml
```

### 📦 Playbook Content
```yaml
---
- name: Install AWX
  hosts: localhost
  become: yes
  vars:
    awx_namespace: awx

  tasks:
    - name: Download Kustomize
      ansible.builtin.shell:
        cmd: curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
        creates: /usr/local/bin/kustomize

    - name: Move Kustomize to /usr/local/bin
      ansible.builtin.shell:
        cmd: mv kustomize /usr/local/bin
      args:
        creates: /usr/local/bin/kustomize

    - name: Create namespace
      ansible.builtin.shell:
        cmd: kubectl create namespace {{ awx_namespace }} --dry-run=client -o yaml | kubectl apply -f -

    - name: Generate AWX resource file
      ansible.builtin.copy:
        dest: "./awx.yaml"
        content: |
          apiVersion: awx.ansible.com/v1beta1
          kind: AWX
          metadata:
            name: awx
          spec:
            service_type: nodeport
            nodeport_port: 30060

    - name: Fetch latest AWX Operator release
      ansible.builtin.shell:
        cmd: curl -s https://api.github.com/repos/ansible/awx-operator/releases/latest | grep tag_name | cut -d '"' -f 4
      register: release_tag
      changed_when: false

    - name: Create kustomization.yaml
      ansible.builtin.copy:
        dest: "./kustomization.yaml"
        content: |
          apiVersion: kustomize.config.k8s.io/v1beta1
          kind: Kustomization
          resources:
            - github.com/ansible/awx-operator/config/default?ref={{ release_tag.stdout }}
            - awx.yaml
          images:
            - name: quay.io/ansible/awx-operator
              newTag: {{ release_tag.stdout }}
          namespace: {{ awx_namespace }}

    - name: Apply Kustomize config
      ansible.builtin.shell:
        cmd: kustomize build . | kubectl apply -f -
```

### ▶️ Run Playbook
```bash
ansible-playbook awx-install.yml
```

---

## 🔍 Post-Install Checks
```bash
# Watch logs
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager -n awx

# Check pods
kubectl get pods -n awx

# Check services
kubectl get svc -n awx

# Get AWX admin password
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" -n awx | base64 --decode ; echo

# Change AWX password
kubectl -n awx exec -it awx-web-<pod-name> -- awx-manage changepassword admin

# Get host IP
hostname -I

# Access AWX in browser
http://<host-ip>:30060
```

---

## 🧹 AWX Removal Playbook

### 📄 Create Removal File
```bash
touch awx-remove.yml
```

### 🧼 Playbook Content
```yaml
---
- name: Remove AWX
  hosts: localhost
  become: yes
  tasks:
    - name: Delete AWX deployment
      shell: kubectl delete deployment awx-operator-controller-manager -n awx
      ignore_errors: yes

    - name: Delete service account
      shell: kubectl delete serviceaccount awx-operator-controller-manager -n awx
      ignore_errors: yes

    - name: Delete role binding
      shell: kubectl delete rolebinding awx-operator-awx-manager-rolebinding -n awx
      ignore_errors: yes

    - name: Delete role
      shell: kubectl delete role awx-operator-awx-manager-role -n awx
      ignore_errors: yes

    - name: Scale deployments to zero
      shell: kubectl scale deployment --all --replicas=0 -n awx
      ignore_errors: yes

    - name: Delete AWX deployments
      shell: kubectl delete deployments.apps/awx-web deployments.apps/awx-task -n awx
      ignore_errors: yes

    - name: Delete statefulsets
      shell: kubectl delete statefulsets.apps/awx-postgres-13 -n awx
      ignore_errors: yes

    - name: Delete services
      shell: kubectl delete service/awx-operator-controller-manager-metrics-service service/awx-postgres-13 service/awx-service -n awx
      ignore_errors: yes

    - name: Get PVC name
      command: kubectl get pvc -n awx -o custom-columns=:metadata.name --no-headers
      register: pvc_output
      ignore_errors: yes

    - name: Delete PVC
      command: kubectl -n awx delete pvc {{ pvc_output.stdout }}
      when: pvc_output.stdout != ""
      ignore_errors: yes

    - name: Get PV name
      command: kubectl get pv -n awx -o custom-columns=:metadata.name --no-headers
      register: pv_output
      ignore_errors: yes

    - name: Delete PV
      command: kubectl -n awx delete pv {{ pv_output.stdout }}
      when: pv_output.stdout != ""
      ignore_errors: yes

    - name: Delete namespace
      shell: kubectl delete namespace awx
      ignore_errors: yes
```

### ▶️ Run Removal
```bash
ansible-playbook awx-remove.yml
```

---

## 📁 Persistent Projects Path Fix

### 🛠️ Updated Install Playbook Snippet
```yaml
vars:
  awx_namespace: awx
  project_directory: /var/lib/awx/projects
  storage_size: 2Gi
```

### 📦 Add PV and PVC Resources
```yaml
- name: Generate PV and PVC
  ansible.builtin.copy:
    dest: "{{ item.dest }}"
    content: "{{ item.content }}"
  loop:
    - dest: "./pv.yml"
      content: |
        apiVersion: v1
        kind: PersistentVolume
        metadata:
          name: awx-projects-volume
        spec:
          accessModes:
            - ReadWriteOnce
          persistentVolumeReclaimPolicy: Retain
          capacity:
            storage: {{ storage_size }}
          storageClassName: awx-projects-volume
          hostPath:
            path: {{ project_directory }}
    - dest: "./pvc.yml"
      content: |
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: awx-projects-claim
        spec:
          accessModes:
            - ReadWriteOnce
          volumeMode: Filesystem
          resources:
            requests:
              storage: {{ storage_size }}
          storageClassName: awx-projects-volume
```

### 📝 Update AWX Resource File
```yaml
spec:
  service_type: nodeport
  nodeport_port: 30060
  projects_persistence: true
  projects_existing_claim: awx-projects-claim
```
```

