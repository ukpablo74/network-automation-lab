# Network automation lab: WSL2 + Containerlab + Ansible

Two Nokia SR Linux routers running as containers inside Ubuntu on WSL2, configured with Ansible and edited from VS Code. No VMware or CML licence needed.

This guide starts from a fresh Windows 11 machine and builds everything, including the Git repository. Follow the steps in order.

## What you will end up with

- Ubuntu running on WSL2, edited from VS Code
- Docker Engine and Containerlab running two Nokia SR Linux routers
- Ansible playbooks that read system info, configure an interface link, and back up router configs
- The whole project stored in a GitHub repository

## Requirements

- Windows 11 with virtualization enabled in the BIOS
- A GitHub account (free)
- About 5 GB of free disk space

---

## 1. Install WSL2 with Ubuntu

Open **PowerShell as Administrator** and run:

```
wsl --install -d Ubuntu
```

Restart when asked. Open **Ubuntu** from the Start menu and create a Linux username and password. Then check it is running as WSL 2:

```
wsl --list --verbose
```

The VERSION column should say 2. If the install fails, check that virtualization is enabled in your BIOS.

From here on, all commands run **inside the Ubuntu terminal** unless stated otherwise. Update Ubuntu first:

```
sudo apt update && sudo apt upgrade -y
```

## 2. Set up VS Code with WSL

1. Install VS Code on Windows: https://code.visualstudio.com
2. In VS Code, install the **WSL** extension (Microsoft).
3. Optional extensions: **Ansible** (Red Hat) and **YAML**.
4. In the Ubuntu terminal, create the project folder and open it in VS Code:

```
mkdir ~/ansible-project
cd ~/ansible-project
code .
```

The bottom-left corner of VS Code should say **WSL: Ubuntu**. Open the built-in terminal (Ctrl+`) and it will run inside Ubuntu. Use that terminal for the rest of this guide.

## 3. Install Docker Engine inside Ubuntu

```
curl -fsSL https://get.docker.com | sudo sh
```

The script shows a WSL warning and pauses for 20 seconds. Wait for it; do not press Ctrl+C. Then:

```
sudo usermod -aG docker $USER
sudo service docker start
```

Close the terminal and open a new one so the group change applies (or run `newgrp docker`). Check it:

```
docker version
```

You should see both a **Client** and a **Server** section. If the Server section is missing or Docker can't connect, run `sudo service docker start` again. On WSL you may need to run that each time you open Ubuntu.

## 4. Install Containerlab

```
bash -c "$(curl -sL https://get.containerlab.dev)"
containerlab version
```

A version number and logo means it worked.

## 5. Create the Git repository

### Tell Git who you are

```
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### Create the project structure

From inside `~/ansible-project`:

```
mkdir -p labs inventories/clab/host_vars playbooks
```

### Add a .gitignore

This keeps the virtual environment, generated lab files and backups out of Git:

```
cat > .gitignore <<'EOF'
.venv/
clab-*/
backups/
__pycache__/
EOF
```

### Initialise the repo

```
git init -b main
```

### Connect it to GitHub

Install the GitHub CLI, sign in, and create the remote repository:

```
sudo apt install -y gh
gh auth login
gh repo create network-automation-lab --private --source=. --remote=origin
```

`gh auth login` walks you through signing in with your browser. If the apt version of `gh` is too old, follow the install steps at https://cli.github.com/packages. Alternatively, create an empty repository on github.com (no README) and run the two commands GitHub shows under "push an existing repository".

Start with `--private`. You can make it public later, after checking that no real passwords are in it (see the last section).

## 6. Install Ansible in a virtual environment

```
sudo apt install -y python3-venv
python3 -m venv .venv
source .venv/bin/activate
pip install ansible jmespath
ansible-galaxy collection install nokia.srlinux
```

Your prompt should now start with `(.venv)`. Run `source .venv/bin/activate` again in every new terminal.

## 7. Create the project files

In VS Code, create each file below at the path shown and paste in the contents. Indentation matters in YAML, so copy exactly.

### `labs/first.clab.yml` (the Containerlab topology)

```yaml
name: first
topology:
  nodes:
    r1:
      kind: nokia_srlinux
      image: ghcr.io/nokia/srlinux
    r2:
      kind: nokia_srlinux
      image: ghcr.io/nokia/srlinux
  links:
    - endpoints: ["r1:e1-1", "r2:e1-1"]
```

### `ansible.cfg`

```ini
[defaults]
inventory = inventories/clab/inventory.yml
deprecation_warnings = False
```

### `inventories/clab/inventory.yml`

```yaml
all:
  children:
    srl:
      hosts:
        clab-first-r1:
        clab-first-r2:
      vars:
        ansible_connection: ansible.netcommon.httpapi
        ansible_network_os: nokia.srlinux.srlinux
        ansible_user: admin
        ansible_password: "NokiaSrl1!"
        ansible_httpapi_use_ssl: true
        ansible_httpapi_validate_certs: false
```

`admin` / `NokiaSrl1!` is the published default login for the free Nokia lab image. Never put real device passwords in a repository.

### `inventories/clab/host_vars/clab-first-r1.yml`

```yaml
interface_ip: 172.31.0.1/30
```

### `inventories/clab/host_vars/clab-first-r2.yml`

```yaml
interface_ip: 172.31.0.2/30
```

The file names must match the hostnames in the inventory exactly.

### `playbooks/site.yml` (read system info and set a location)

```yaml
- name: First SR Linux playbook
  hosts: srl
  gather_facts: false
  tasks:
    - name: Read system info
      nokia.srlinux.get:
        paths:
          - path: /system/information
            datastore: state
      register: info

    - name: Show it
      ansible.builtin.debug:
        var: info.result

    - name: Set the location
      nokia.srlinux.config:
        update:
          - path: /system/information
            value:
              location: "Home lab"
```

### `playbooks/interfaces.yml` (configure the link between the routers)

```yaml
- name: Configure the link between the routers
  hosts: srl
  gather_facts: false
  tasks:
    - name: Enable ethernet-1/1 and set its IP address
      nokia.srlinux.config:
        update:
          - path: /interface[name=ethernet-1/1]
            value:
              admin-state: enable
              subinterface:
                - index: 0
                  admin-state: enable
                  ipv4:
                    admin-state: enable
                    address:
                      - ip-prefix: "{{ interface_ip }}"
          - path: /network-instance[name=default]
            value:
              interface:
                - name: ethernet-1/1.0
```

The second update attaches the interface to the default routing instance. Without it the port has an address but cannot reply to pings.

### `playbooks/backup.yml` (save each router's config to a file)

```yaml
- name: Back up router configs
  hosts: srl
  gather_facts: false
  vars:
    backup_dir: "{{ playbook_dir }}/../backups"
  tasks:
    - name: Get the running config
      nokia.srlinux.get:
        paths:
          - path: /
            datastore: running
      register: running_config

    - name: Make sure the backup folder exists
      ansible.builtin.file:
        path: "{{ backup_dir }}"
        state: directory
        mode: "0755"
      delegate_to: localhost
      run_once: true

    - name: Save the config to a file
      ansible.builtin.copy:
        content: "{{ running_config.result | to_nice_json }}\n"
        dest: "{{ backup_dir }}/{{ inventory_hostname }}.json"
        mode: "0644"
      delegate_to: localhost
```

## 8. Run the lab

Deploy the routers. The first run downloads the Nokia image (a few hundred MB), so it can take a few minutes:

```
sudo containerlab deploy -t labs/first.clab.yml
```

You should get a table showing `clab-first-r1` and `clab-first-r2` with IP addresses. Then run the playbooks:

```
ansible-playbook playbooks/site.yml
ansible-playbook playbooks/interfaces.yml
ansible-playbook playbooks/backup.yml
```

Both routers should report `ok` or `changed`. Run any playbook a second time and you should see `changed=0`. That is idempotency: Ansible only changes what differs.

### Check the result on a router

```
ssh admin@clab-first-r1
```

The password is `NokiaSrl1!`. Inside the router CLI:

```
info system information
info interface ethernet-1/1
ping 172.31.0.2 network-instance default
```

You should see the location and IP address that Ansible set, and ping replies from the other router. Type `quit` to leave the router (`exit` does nothing at the top level).

## 9. Save your work to GitHub

```
git add .
git status
```

Read the `git status` list first. You should not see `.venv`, `clab-first` or `backups`. Then:

```
git commit -m "Containerlab and Ansible network lab"
git push -u origin main
```

After any later change, `git add .`, `git commit -m "what changed"` and `git push` update the repository.

## 10. Tear down and rebuild

```
sudo containerlab destroy -t labs/first.clab.yml
```

The router config is lost when the lab is destroyed, but rebuilding takes minutes:

```
sudo containerlab deploy -t labs/first.clab.yml
ansible-playbook playbooks/site.yml
ansible-playbook playbooks/interfaces.yml
```

## Troubleshooting

| Problem | Fix |
| --- | --- |
| `docker: command not found` | Docker Engine is not installed in Ubuntu. Redo step 3. |
| `permission denied` on the Docker socket | Open a new terminal, or run `newgrp docker`. If it persists, run `wsl --shutdown` in PowerShell and reopen Ubuntu. |
| Docker can't connect to the daemon | Run `sudo service docker start`. |
| `externally managed environment` error from pip | Use the virtual environment in step 6. |
| Playbook says `interface_ip` is undefined | The `host_vars` file names must match the inventory hostnames exactly. |
| Ping fails between routers | Re-run `playbooks/interfaces.yml`. The interface must be attached to the default network instance. |
| Router hostnames don't resolve | Use the IP addresses from the `containerlab deploy` table. |

## Before making the repo public

Run this from the project folder to look for credentials:

```
grep -rniE "password|secret|token" --exclude-dir=.venv --exclude-dir=.git .
```

The Nokia default login is fine to share. Anything for real or personal devices is not. For real secrets, use Ansible Vault (`ansible-vault encrypt`) or keep those files out of Git with `.gitignore`.

## Related

A separate Raspberry Pi lab, using Ansible against a real Linux host over SSH, is in [ansible-raspberry-pi-lab](https://github.com/ukpablo74/ansible-raspberry-pi-lab).
