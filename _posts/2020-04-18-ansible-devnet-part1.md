---
layout: post
title: "Ansible and Cisco DevNet: Part 1. Getting Started"
date: 2020-04-18
author: "Christopher Hannan"
---

## Table of Contents

* [Objectives](#objectives)
* [Prerequisites](#prerequisites)
* [Guide](#guide)
* [Next Steps](#next-steps)

## Objectives

* Install and verify the default Ansible settings
* Create a new Ansible environment

## Prerequisites

> **Note:** This guide assumes that you already have a control node that can run Ansible, with Python installed and access to the internet.

## Guide

### Step 1: Install Ansible

Listed below are the installation steps for both Ubuntu and macOS. For other distributions, please follow Ansible's [Installation Guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html "Installing Ansible").

**Ubuntu 18.04**

```bash
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

**macOS**

```bash
sudo easy_install pip
sudo pip install ansible
```

### Step 2: Verify the Ansible Install

When Ansible has installed, verify the default configuration by running the `ansible --version` command.

```bash
cphannan@ubuntu:~$ ansible --version
ansible 2.8.6
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/cphannan/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.6/dist-packages/ansible
  executable location = /usr/local/bin/ansible
  python version = 3.6.9 (default, Apr 18 2020, 01:56:04) [GCC 8.4.0]
```

The output shows that for my installation of Ansible:

* The version I am running is 2.8.6
* The default configuration file is **/etc/ansible/ansible.cfg**
* The version of Python used is 3.6.9

View the contents of the default **ansible.cfg** file.

```bash
cphannan@ubuntu:~$ cat /etc/ansible/ansible.cfg
[defaults]
#inventory       = /etc/ansible/hosts
#library         = ~/.ansible/plugins/modules:/usr/share/ansible/plugins/modules
#module_utils    = ~/.ansible/plugins/module_utils:/usr/share/ansible/plugins/module_utils
#remote_tmp      = ~/.ansible/tmp
#local_tmp       = ~/.ansible/tmp
#forks           = 5
#poll_interval   = 0.001
#ask_pass        = False
#transport       = smart
```

The output shows that for the default Ansible configuration:

* The inventory it will use is **/etc/ansible/hosts**

The source of the **ansible.cfg** file can be in several places and Ansible will search for it in the following order, using the first file found:

* `ANSIBLE_CONFIG` (environment variable if set)
* `ansible.cfg` (in the current directory)
* `~/.ansible.cfg` (in the home directory)
* `/etc/ansible/ansible.cfg`

>**Source:** https://docs.ansible.com/ansible/latest/reference_appendices/config.html

We will now create a new directory for the Ansible environment and then create a custom **ansible.cfg** file that will override the defaults above.

### Step 3: Build the base directory layout

This guide will just focus on populating the **dev/** inventory but the Ansible directory we create follows the alternative layout from Ansible's [best practice](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html "Ansible Best Practices") for directory layouts. We will have a separate directory for our production and development inventories, each with their own respective group and host var files.

```bash
cphannan@ubuntu:~$ mkdir netauto
cphannan@ubuntu:~$ cd netauto
cphannan@ubuntu:~/netauto$
cphannan@ubuntu:~/netauto$ mkdir inventories
cphannan@ubuntu:~/netauto$ mkdir inventories/dev
cphannan@ubuntu:~/netauto$ mkdir inventories/dev/group_vars
cphannan@ubuntu:~/netauto$ mkdir inventories/dev/host_vars
cphannan@ubuntu:~/netauto$ mkdir roles
```

### Step 4: Create a custom ansible.cfg file

Use `nano` to create the new **ansible.cfg** file and paste in the below parameters:

```bash
cphannan@ubuntu:~/netauto$ nano ansible.cfg
```

```ini
[defaults]
stdout_callback = yaml
timeout = 60
deprecation_warnings = False
host_key_checking = False
retry_files_enabled = True

[persistent_connection]
connect_timeout = 60
command_timeout = 60
```

### Step 5: Add the Cisco DevNet sandboxes to the dev/ inventory

[Cisco DevNet](https://developer.cisco.com/site/sandbox/ "Cisco DevNet") have always-on sandboxes for Cisco IOS, IOS-XR, NX-OS and ACI which we will add to the **dev/** inventory. Providing our control node can access the devices via SSH, we could add our own lab equipment (virtual or physical) to the same inventory for testing network automation against.

In the example below, the **dev/** inventory file defines the groups `ios`, `iosxr`, `nxos`, `aci` and a "group of groups" called `always_on`. 

Use `nano` to create the new **hosts** file in **inventories/dev/**.

```bash
cphannan@ubuntu:~/netauto$ nano inventories/dev/hosts
```

Paste in the below information for the Cisco DevNet hosts:

```ini
[always_on]
ios-xe-mgmt.cisco.com
sbx-iosxr-mgmt.cisco.com
sbx-nxos-mgmt.cisco.com


[ios]
ios-xe-mgmt.cisco.com ansible_host=64.103.37.51


[iosxr]
sbx-iosxr-mgmt.cisco.com ansible_host=64.103.37.3


[nxos]
sbx-nxos-mgmt.cisco.com ansible_host=64.103.37.14


[aci]
sandboxapicdc.cisco.com
```

### Step 6: Populate the group variables for the dev/ inventory

Group variables are used to apply shared variables to hosts in the same inventory group. Create the new "group_var" files and paste in the parameters.

**For the IOS group:**

```bash
cphannan@ubuntu:~/netauto$ nano inventories/dev/group_vars/ios.yml 
```

```yaml
---

ansible_network_os: ios

ansible_become: yes
ansible_become_method: enable
```

**For the IOS-XR group:**

```bash
cphannan@ubuntu:~/netauto$ nano inventories/dev/group_vars/iosxr.yml 
```

```yaml
---

ansible_network_os: iosxr
```

**For the NX-OS group:**

```bash
cphannan@ubuntu:~/netauto$ nano inventories/dev/group_vars/nxos.yml 
```

```yaml
---

ansible_network_os: nxos
```

**For the ACI group:**

```bash
cphannan@ubuntu:~/netauto$ nano inventories/dev/group_vars/aci.yml 
```

```yaml
---

ansible_network_os: aci
```

### Step 7: Populate the host variables for the dev/ inventory

Host variables are used to assign variables to a single host. As each Cisco DevNet sandbox has its own specific connection parameters and login details, we will create host specific folders and variable files for each of the sandboxes.

> **Note:** These credentials for the Cisco DevNet sandboxes are publicly available so for simplicity we will store them as host variables in plaintext. If this was our production environment we could encrypt these credential variables at rest using Ansible's [Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html "Ansible Vault") feature.

**For the IOS host:**

```bash
cphannan@ubuntu:~/netauto$ mkdir inventories/dev/host_vars/ios-xe-mgmt.cisco.com
cphannan@ubuntu:~/netauto$ nano inventories/dev/host_vars/ios-xe-mgmt.cisco.com/ios-xe-mgmt.cisco.com.yml
```

```yaml
---

ansible_user: "root"
ansible_password: "D_Vay!_10&"
ansible_port: 8181
```

**For the IOS-XR host:**

```bash
cphannan@ubuntu:~/netauto$ mkdir inventories/dev/host_vars/sbx-iosxr-mgmt.cisco.com
cphannan@ubuntu:~/netauto$ nano inventories/dev/host_vars/sbx-iosxr-mgmt.cisco.com/sbx-iosxr-mgmt.cisco.com.yml
```

```yaml
---

ansible_user: "admin"
ansible_password: "C1sco12345"
ansible_port: 8181
```

**For the NX-OS host:**

```bash
cphannan@ubuntu:~/netauto$ mkdir inventories/dev/host_vars/sbx-nxos-mgmt.cisco.com
cphannan@ubuntu:~/netauto$ nano inventories/dev/host_vars/sbx-nxos-mgmt.cisco.com/sbx-nxos-mgmt.cisco.com.yml
```

```yaml
---

ansible_user: "admin"
ansible_password: "Admin_1234!"
ansible_port: 8181
```

**For the ACI host:**

```bash
cphannan@ubuntu:~/netauto$ mkdir inventories/dev/host_vars/sandboxapicdc.cisco.com
cphannan@ubuntu:~/netauto$ nano inventories/dev/host_vars/sandboxapicdc.cisco.com/sandboxapicdc.cisco.com.yml
```

```yaml
---

ansible_user: "admin"
ansible_password: "ciscopsdt"
```

### Step 8: Verify the Ansible directory layout

The ansible environment's directory structure should now look like this:

```ini
cphannan@ubuntu:~/netauto$ tree 
.
├── ansible.cfg
└── inventories
    └── dev
        ├── group_vars
        │   ├── aci.yml
        │   ├── iosxr.yml
        │   ├── ios.yml
        │   └── nxos.yml
        ├── hosts
        └── host_vars
            ├── ios-xe-mgmt.cisco.com
            │   └── ios-xe-mgmt.cisco.com.yml
            ├── sandboxapicdc.cisco.com
            │   └── sandboxapicdc.cisco.com.yml
            ├── sbx-iosxr-mgmt.cisco.com
            │   └── sbx-iosxr-mgmt.cisco.com.yml
            └── sbx-nxos-mgmt.cisco.com
                └── sbx-nxos-mgmt.cisco.com.yml

8 directories, 10 files
```

## Next Steps

We now have a solid Ansible directory structure, with the Cisco DevNet sandboxes for multiple OS types defined in a dedicated development inventory.

[Part 2](/2020/06/06/ansible-devnet-part2/) of this guide will cover creating a simple playbook to retrieve information from the Cisco DevNet sandboxes.
