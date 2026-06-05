# Module 15 - Configuration Management with Ansible

This repository contains a demo project created as part of my **DevOps studies** in the [TechWorld with Nana – DevOps Bootcamp](https://www.techworld-with-nana.com/devops-bootcamp).

**Demo Project:** Automate Node.js application Deployment

**Technologies used:** Ansible, Node.js, DigitalOcean, Linux

**Project Description:**

- Create Server on DigitalOcean
- Write Ansible Playbook that installs necessary technologies, creates Linux user for an application and deploys a NodeJS application with that user

---

## Prerequisites

- **Ansible** installed on your local control machine. Ansible runs *from here* — it connects to the server over SSH; nothing is installed on the target ahead of time.

macOS (Homebrew):
```sh
brew install ansible
```

With pip (any OS with Python ≥ 3.9):
```sh
pip install ansible
```

Verify the install and see which Python interpreter Ansible uses:
```sh
ansible --version
```

- **Node.js & npm** on your local machine, used to build the application package with `npm pack`.
- An **SSH key pair** (e.g. `~/.ssh/id_rsa`) whose public key is added to the droplet.
- The **`community.general`** collection, which provides the `npm` module used by this playbook:
```sh
ansible-galaxy collection install community.general
```

> ℹ️ `apt`, `user`, `unarchive`, `command`, and `shell` ship with `ansible-core`. The `npm` module lives in the separate `community.general` collection, so it must be installed before running the playbook.

---

### Overview

![](./images/overview.png)


### Create Server on DigitalOcean

Droplet configuration:

| Setting     | Value   |
| ----------- | ------- |
| Image       | Ubuntu  |
| CPU Option  | Regular |
| vCPU        | 1       |
| RAM         | 1 GB    |
| Disk        | 25 GB   |

![](./images/create-droplet.png)

> Ansible is **agentless** — it manages the server purely over SSH. The only requirement on the target is a Python interpreter, which Ubuntu ships with by default. Confirm the path (e.g. `/usr/bin/python3.12`) and set it as `ansible_python_interpreter` in your inventory.

- Copy the public IP address of the new droplet.


### Write the Ansible Playbook

The playbook installs the required technologies, creates a dedicated Linux user for the application, and deploys the Node.js app as that user. See [deploy-node.yaml](deploy-node.yaml) for the full playbook.

#### 1. Create the inventory (`hosts`) file

```sh
cp hosts.example hosts
```

```conf
webserver ansible_host=<DROPLET-IP> ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_rsa ansible_python_interpreter=/usr/bin/python3.12
```

| Variable                       | Purpose                                            |
| ------------------------------ | -------------------------------------------------- |
| `ansible_host`                 | Public IP of the droplet                           |
| `ansible_user`                 | SSH user Ansible connects as (`root`)              |
| `ansible_ssh_private_key_file` | Private key matching the key added to the droplet  |
| `ansible_python_interpreter`   | Python on the target used to run modules           |

> Host key checking is disabled in [ansible.cfg](ansible.cfg) (`host_key_checking = False`) so the first connection to a fresh droplet doesn't prompt for fingerprint confirmation.

#### 2. Externalize values in `project-vars`

Shared values live in a [project-vars](project-vars) file and are pulled into each play with `vars_files`, keeping the playbook DRY:

```yaml
location: nodejs-app
version: 1.0.0
user_name: app-user
destination: "/home/{{user_name}}"
```

#### 3. Install Node.js and npm — `apt` module

```yaml
- name: Install node and npm
  hosts: webserver
  tasks:
    - name: Update apt repo and cache
      apt: update_cache=yes force_apt_get=yes cache_valid_time=3600
    - name: Install nodejs and npm
      apt:
        pkg:
          - nodejs
          - npm
```

> 📘 [`apt` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
> - `update_cache=yes` runs `apt-get update` before installing (refreshes the package lists).
> - `cache_valid_time=3600` skips that refresh if the cache is younger than 3600 s, avoiding redundant updates on repeated runs.
> - `force_apt_get=yes` forces `apt-get` instead of `aptitude`.
> - Passing a **list** to `pkg` installs all packages in a single transaction — much more efficient than looping the module once per package.

#### 4. Build & copy the app — `npm pack` + `unarchive` module

Build the deployable tarball from the `nodejs-app` directory:

```sh
cd nodejs-app
npm i
npm pack
```

`npm pack` produces `nodejs-app-1.0.0.tgz` — the exact artifact npm would publish.

```yaml
- name: Unpack the nodejs tar file
  unarchive:
    src: "{{location}}/nodejs-app-{{version}}.tgz"
    dest: "{{destination}}"
```

> 📘 [`unarchive` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html)
> - With the default `remote_src: no`, the archive is taken from the **control machine** and copied to the target before being unpacked. Use `remote_src: yes` when the archive already exists on the server (or is a URL).
> - `dest` must already exist on the target.
> - Requires `tar`/`unzip` on the target host.
> - Add `creates:` to make it idempotent — the unpack is skipped if the given path already exists.

Run the playbook:
```sh
ansible-playbook -i hosts deploy-node.yaml
```

![](./images/deploy-node-1.png)

Check the result on the server:

![](./images/check-server.png)


#### 5. Install dependencies & start the app — `npm` + `command` modules

```yaml
- name: Install dependencies
  npm:
    path: "{{destination}}/package"

- name: Start the application
  command:
    chdir: "{{destination}}/package/app"
    cmd: node server
  async: 1000
  poll: 0
```

> 📘 [`npm` module](https://docs.ansible.com/ansible/latest/collections/community/general/npm_module.html) (in `community.general`)
> - With `path` set and no `name`, it installs the dependencies declared in that directory's `package.json` (equivalent to running `npm install` there).

> 📘 [`command` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html)
> - `chdir` changes into the directory before executing; `cmd` is the command to run.
> - `command` does **not** run through a shell, so `|`, `>`, `<`, `;`, `&`, and `*` are **not** interpreted. Prefer it over `shell` when you don't need those features — it's more secure.
> - It is **not idempotent**: it runs every time unless guarded with `creates:`/`removes:`.

> ⚡ **Why `async` + `poll: 0`?** `node server` is a long-running process that never returns. Without async, the task would block forever and the playbook would hang. [Asynchronous "fire-and-forget"](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_async.html) starts the process and moves on:
> - `async: 1000` — allow up to 1000 s of runtime before Ansible would terminate it.
> - `poll: 0` — don't wait for the result; start it and continue to the next task.

Run again:
```sh
ansible-playbook -i hosts deploy-node.yaml
```

#### 6. Verify the app is running — `shell` module

```yaml
- name: Ensure app is running
  shell: ps aux | grep node
  register: app_status
- debug: msg={{app_status.stdout_lines}}
```

> The `shell` module is used here (not `command`) precisely because the task uses a **pipe** (`|`). `register` captures the task's output into `app_status`, and `debug` prints it — a handy pattern for inspecting results during a run.

Check Node status on the server directly:
```sh
ps aux | grep node
```
![](./images/ps-node-server.png)

![](./images/app-status.png)


#### 7. Create a dedicated Linux user — `user` module + privilege escalation

Running the app under its own unprivileged user (`app-user`) instead of `root` is a security best practice.

```yaml
- name: Create new linux user for node app
  hosts: webserver
  vars_files:
    project-vars
  tasks:
    - name: Create linux user
      user:
        name: "{{user_name}}"
        comment: App User
        group: admin
      register: user_creation_result
    - debug: msg={{user_creation_result}}
```

> 📘 [`user` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html)
> - `group` sets the user's **primary** group and must reference a group that already exists; `groups` (with `append: yes`) would set **supplementary** groups instead.
> - On Ubuntu the group that grants sudo rights is typically `sudo` — adjust `group`/`groups` to match the access the app user actually needs.

Deploy the app **as that user** by adding `become` parameters to the deploy play:

```yaml
- name: Deploy nodejs app
  hosts: webserver
  become: True
  become_user: "{{user_name}}"
  vars_files:
    project-vars
  tasks:
    ...
```

> 📘 [Privilege escalation (`become`)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_privilege_escalation.html)
> - `become: true` activates privilege escalation; `become_user` selects the target user (defaults to `root`). Note that setting `become_user` alone does **not** imply `become: true`.
> - Here we connect as `root` and then *drop down* to `app-user`. This is the recommended pattern — escalating from `root` to an unprivileged user avoids the temp-file permission issues Ansible runs into when **both** the connection user and `become_user` are unprivileged.

Run the full playbook:
```sh
ansible-playbook -i hosts deploy-node.yaml
```

![](./images/become-user.png)
