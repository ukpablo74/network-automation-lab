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

---

## 11. Add a Raspberry Pi as a second managed device

This extends the lab beyond containers to a real Linux host you can push configuration to over the network. Replace `raspberrypi.local` and `192.168.1.50` below with your Pi's actual hostname and IP.

### Set up the Pi

1. Flash **Raspberry Pi OS Lite (64-bit)** to the SD card with Raspberry Pi Imager. In the Imager's settings (the gear icon), enable SSH, set a username and password, and enter your Wi-Fi details before writing the card. This lets you boot the Pi headless, with no monitor or keyboard needed.
2. Boot the Pi and find its IP address, either from your router's device list or by running this from Ubuntu:

```
ping raspberrypi.local
```

3. Confirm you can reach it over SSH from Ubuntu:

```
ssh pi@raspberrypi.local
```

Accept the host key prompt on first connection, then `exit` back to Ubuntu.

### Let Ansible log in without a password

Generate a key in Ubuntu if you don't already have one, then copy it to the Pi:

```
ssh-keygen -t ed25519 -C "ansible-project"
ssh-copy-id pi@raspberrypi.local
```

Test that it works without asking for a password:

```
ssh pi@raspberrypi.local
```

### Add the Pi to the inventory

Create a second inventory file so the Pi is kept separate from the container lab devices:

`inventories/pi/inventory.yml`

```yaml
all:
  children:
    linux_hosts:
      hosts:
        raspberrypi:
          ansible_host: raspberrypi.local
          ansible_user: pi
          ansible_python_interpreter: /usr/bin/python3
```

Ansible needs Python on the target, which Raspberry Pi OS includes by default.

### Write a playbook for the Pi

`playbooks/pi-setup.yml`

```yaml
- name: Basic configuration for the Raspberry Pi
  hosts: linux_hosts
  become: true
  tasks:
    - name: Update the package cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Make sure useful packages are installed
      ansible.builtin.apt:
        name:
          - curl
          - git
          - net-tools
        state: present

    - name: Set the hostname
      ansible.builtin.hostname:
        name: raspberrypi

    - name: Create a marker file to prove Ansible reached the Pi
      ansible.builtin.copy:
        content: "Configured by Ansible on {{ ansible_date_time.iso8601 }}\n"
        dest: /home/pi/ansible-was-here.txt
```

Run it, pointing at the Pi's inventory specifically since `ansible.cfg` defaults to the containerlab one:

```
ansible-playbook -i inventories/pi/inventory.yml playbooks/pi-setup.yml
```

Check it landed:

```
ssh pi@raspberrypi.local cat /home/pi/ansible-was-here.txt
```

Run the playbook a second time and the apt and copy tasks should show `ok` rather than `changed`, the same idempotency you saw with the routers.

## 12. Add Nornir alongside Ansible

Nornir is a Python automation framework rather than a YAML-based tool like Ansible. Where Ansible describes the desired state declaratively, Nornir gives you a Python script with full control over logic, branching and output, which suits engineers who want to automate in code.

### Install Nornir

In the same virtual environment as Ansible:

```
source .venv/bin/activate
pip install nornir nornir-utils nornir-napalm nornir-netmiko
```

- `nornir-netmiko` handles SSH to network-style CLIs (useful for the Pi treated as a generic Linux/SSH host, or real Cisco gear later).
- `nornir-napalm` is the common choice for multi-vendor network devices; skip it if you are only targeting the Pi over plain SSH for now.

### Set up Nornir's inventory

Nornir keeps its inventory in a small folder of its own, separate from Ansible's.

`nornir/config.yml`

```yaml
inventory:
  plugin: SimpleInventory
  options:
    host_file: "nornir/hosts.yml"
    group_file: "nornir/groups.yml"
runner:
  plugin: threaded
  options:
    num_workers: 5
```

`nornir/groups.yml`

```yaml
linux:
  connection_options:
    netmiko:
      platform: linux
      extras:
        username: pi
```

`nornir/hosts.yml`

```yaml
raspberrypi:
  hostname: raspberrypi.local
  groups:
    - linux
```

Netmiko needs a password or key. Rather than put a password in this file, rely on the SSH key you already copied to the Pi, which Netmiko will use automatically as your Ubuntu user's default key.

### Write a first Nornir script

`nornir/check_uptime.py`

```python
from nornir import InitNornir
from nornir_netmiko import netmiko_send_command
from nornir_utils.plugins.functions import print_result

nr = InitNornir(config_file="nornir/config.yml")

result = nr.run(
    task=netmiko_send_command,
    command_string="uptime",
)

print_result(result)
```

Run it:

```
python nornir/check_uptime.py
```

You should see the Pi's uptime printed with a green `vvvv` header, meaning it succeeded. A red header means the connection or command failed.

### A Nornir script that pushes a small config change

This adds a line to the Pi's message-of-the-day file, so you can see Nornir make a real change:

`nornir/set_motd.py`

```python
from nornir import InitNornir
from nornir_netmiko import netmiko_send_command
from nornir_utils.plugins.functions import print_result

nr = InitNornir(config_file="nornir/config.yml")

result = nr.run(
    task=netmiko_send_command,
    command_string='echo "Managed by Nornir" | sudo tee /etc/motd',
)

print_result(result)
```

```
python nornir/set_motd.py
ssh pi@raspberrypi.local cat /etc/motd
```

Unlike Ansible modules, a raw command like this isn't idempotent by itself. It will happily overwrite the file every run. That trade-off, more control but less built-in safety, is the main practical difference between the two tools day to day.

### When to reach for which tool

- **Ansible**: fits most day-to-day config-push tasks, works well with YAML-based diffs, and has ready-made modules for common jobs, including the `nokia.srlinux` one used earlier in this guide.
- **Nornir**: fits when you need custom Python logic, such as parsing command output, branching based on a device's state, or integrating with other Python libraries and APIs.

Many teams use both: Ansible for standard config pushes, Nornir for bespoke checks, reports, or migrations. Having working examples of each, as in this repo, is worth mentioning on its own.

### Add these to Git

```
git add inventories/pi playbooks/pi-setup.yml nornir
git commit -m "Add Raspberry Pi target and Nornir scripts"
git push
```

Check `git status` before committing. If any inventory file ends up holding a real password, move it out and use Ansible Vault or an environment variable instead of committing it.
