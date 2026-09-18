# Ansible for DevOps Interviews (4+ Years Experience)

## 1. Basic Syntax

**Inventory, Playbooks, and Tasks**

*Inventory file (`hosts.ini`)*
```ini
[webservers]
web1.example.com ansible_user=ubuntu
web2.example.com ansible_user=ubuntu

[dbservers]
db1.example.com ansible_user=admin

[production:children]
webservers
dbservers
```

*Basic Playbook (`site.yml`)*
```yaml
---
- name: Setup Web Servers
  hosts: webservers
  become: yes # Run with sudo
  
  tasks:
    - name: Ensure Nginx is installed
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Ensure Nginx is running and enabled
      service:
        name: nginx
        state: started
        enabled: yes
```

---

## 2. Intermediate Examples

**Variables, Handlers, Templates, and Conditionals**
```yaml
---
- name: Configure Application
  hosts: all
  become: yes
  vars:
    app_port: 8080
    env: production

  tasks:
    - name: Create app directory
      file:
        path: /opt/myapp
        state: directory
        owner: appuser
        mode: '0755'

    - name: Deploy configuration template
      template:
        src: app_config.j2
        dest: /opt/myapp/config.yaml
      notify: Restart App # Triggers the handler

    - name: Install debug tools on RedHat only
      yum:
        name: tcpdump
        state: present
      when: ansible_os_family == "RedHat" # Conditional using Facts

  handlers:
    - name: Restart App
      service:
        name: myapp
        state: restarted
```

*Jinja2 Template (`app_config.j2`)*
```yaml
# Managed by Ansible
server:
  port: {{ app_port }}
  environment: {{ env }}
  hostname: {{ ansible_hostname }} # Built-in fact
```

---

## 3. Advanced Examples

**Loops, `register`, Error Handling, and Roles/Collections**
```yaml
---
- name: Advanced User Management
  hosts: all
  become: yes
  
  tasks:
    - name: Create multiple users with loops
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        state: present
      loop:
        - { name: 'alice', groups: 'wheel' }
        - { name: 'bob', groups: 'users' }

    - name: Run a command and capture output
      command: cat /etc/os-release
      register: os_info
      changed_when: false # Idempotency trick: don't mark as "changed" if just reading

    - name: Fail gracefully if unsupported OS
      fail:
        msg: "This playbook only supports Ubuntu!"
      when: "'Ubuntu' not in os_info.stdout"

    - name: Attempt risky operation with rescue
      block:
        - name: Download critical update
          get_url:
            url: https://internal.repo/update.tar.gz
            dest: /tmp/update.tar.gz
      rescue:
        - name: Send alert if download fails
          debug:
            msg: "Download failed! Attempting fallback."
```

---

## 4. Interview Coding Exercises

### Problem 1: Idempotency with Shell Commands
**Task:** You are using the `shell` module to clone a repo. How do you ensure it is idempotent (doesn't run and report "changed" every time)?

**Solution:**
```yaml
- name: Clone custom scripts
  shell: git clone https://github.com/org/scripts.git /opt/scripts
  args:
    creates: /opt/scripts/README.md # Ansible checks if this file exists first.
```
*Alternative (Better Practice):* Use the `git` module instead of `shell`.
```yaml
- name: Clone custom scripts properly
  git:
    repo: https://github.com/org/scripts.git
    dest: /opt/scripts
    version: main
```

### Problem 2: Dynamic Variables based on Environment
**Task:** Deploy a config file where `db_host` is `dev-db.local` for the `dev` group and `prod-db.local` for the `prod` group, without using conditionals in the task.

**Solution:**
Use `group_vars`.
*Directory structure:*
```text
group_vars/
  dev.yml  -> db_host: dev-db.local
  prod.yml -> db_host: prod-db.local
```
*Playbook:*
```yaml
- name: Deploy DB Config
  template:
    src: db.j2
    dest: /etc/db.conf
# Ansible automatically maps the correct `db_host` based on the inventory group the host belongs to.
```

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: Handler Not Firing
**What is wrong here?**
```yaml
tasks:
  - name: Update Nginx Config
    copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf
    notify: restart_nginx

handlers:
  - name: Restart Nginx
    service:
      name: nginx
      state: restarted
```
**Answer:** The `notify` name (`restart_nginx`) does not match the handler's name (`Restart Nginx`). They must match exactly (case-sensitive).

### Broken Configuration 2: Vault Password Issue
**Scenario:** A playbook uses `ansible-vault` for secrets. It runs fine locally but fails in the Jenkins CI/CD pipeline with `ERROR! A vault password must be specified`.
**Solution:** Pass the vault password in Jenkins using a file or environment variable:
`ansible-playbook site.yml --vault-password-file /path/to/.vault_pass` (ensure the file is securely injected by Jenkins credentials).

---

## 6. Common Interview Questions

**Q: "What is Ansible Galaxy and how do Roles work?"**
*Answer:* Ansible Galaxy is a repository for Ansible Roles and Collections. A Role is a way to modularize Ansible code into reusable components. A typical role structure includes folders for `tasks`, `handlers`, `vars`, `defaults`, `templates`, and `files`. `defaults/main.yml` has the lowest precedence (easily overridden), while `vars/main.yml` has higher precedence.

**Q: "Explain Ansible execution strategy (Linear vs Free)."**
*Answer:* By default, Ansible uses the `linear` strategy. It runs Task 1 on *all* hosts, waits for all to finish, then moves to Task 2. The `free` strategy allows each host to run tasks as fast as it can independently, without waiting for other hosts.

**Q: "How do you limit a playbook run to a specific subset of hosts?"**
*Answer:* Use the `--limit` or `-l` flag in the CLI: `ansible-playbook deploy.yml -l dbservers` or `-l web1.example.com`.

---

## 7. Cheat Sheet

| Command / Concept | Usage (4+ Yrs Experience Focus) |
| :--- | :--- |
| `ansible-playbook -i hosts play.yml --check` | Dry run mode. Shows what *would* change. |
| `ansible-playbook -i hosts play.yml --diff` | Shows the actual text differences (diffs) in file changes. |
| `ansible-vault encrypt secret.yml` | Encrypt a file containing variables. |
| `ansible -m setup hostname` | Gather and view all Ansible Facts for a host. |
| `changed_when: false` | Forces a task to never report a 'changed' state (useful for `command`/`shell` reads). |
| `ignore_errors: yes` | Continues playbook execution even if the task fails. |
| `delegate_to: localhost` | Runs the task on the control node rather than the target host (e.g., updating a local LB config). |
