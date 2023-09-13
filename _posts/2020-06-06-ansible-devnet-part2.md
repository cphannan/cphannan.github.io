---
layout: post
title: "Ansible and Cisco DevNet: Part 2. Writing a simple playbook"
date: 2020-06-06
author: "Christopher Hannan"
---

## Table of Contents

* [Objectives](#objectives)
* [Prerequisites](#prerequisites)
* [Guide](#guide)
* [Takeaways](#takeaways)

## Objectives

* Write a simple playbook to retrieve facts from network devices
* Test the new playbook against Cisco's DevNet sandboxes

An Ansible playbook is written in YAML and orchestrates the playbook 'run'. The top section specifies what devices are targeted for this run and the tasks list is an ordered set of instructions to execute on each device.

We will now write a playbook that utilises the Ansible [ios_facts](https://docs.ansible.com/ansible/latest/modules/ios_facts_module.html "ios_facts") module, used to collect a base set of facts from Cisco IOS devices.

## Prerequisites

> **Note:** This guide assumes that you have already worked through the Getting Started steps covered in [Part 1](/2020/04/18/ansible-devnet-part1/).

## Guide

### Step 1: Create a playbook

In the root of the `netauto/` directory, create a new file named `play_gather-facts.yml`.

```bash
cphannan@ubuntu:~/netauto$ touch play_gather-facts.yml
```

YAML accepts single-line comments and I have included comments in this playbook to break down what each line is doing. These comments will be omitted when Ansible executes the playbook.

Use nano or a text-editor of your choice to paste in the below information:

```yaml
---                                                  ## signals the start of a YAML file

- name: Play - IOS audit report                      ## name for the playbook, included in the output of a playbook run
  hosts: ios                                         ## list of one or more inventory groups, separated by colons
  connection: network_cli                            ## plugin for a CLI connection to network devices
  gather_facts: no                                   ## 'no' is required for Ansible versions prior to 2.9

  tasks:                                             ## start of the tasks list, executed from top to bottom on each network device in the hosts list

  - name: Task 1 - Gathering IOS facts               ## name for task 1, included in the output of a playbook run
    ios_facts:                                       ## ios_facts is the Ansible module we are invoking
    register: facts                                  ## capturing the output from the task to the variable 'facts'

  - debug: var=facts                                 ## debug is the Ansible module we are invoking for task 2, which will print the return information from the task 1
```

### Step 2: Run the playbook

Our playbook is now ready to run and the command below tells Ansible that we want to run the playbook `play_gather-facts.yml` against the `/dev` inventory. Because we have listed `ios` as the target hosts list in the playbook, only the `[ios]` list from our `/dev` inventory file will be targeted; specifically, the DevNet IOS sandbox device `ios-xe-mgmt.cisco.com`.

Issue the Ansible command below:

```bash
cphannan@ubuntu:~/netauto$ ansible-playbook play_gather-facts.yml -i inventories/dev/
```
---

```bash
chhannan@ubuntu:~/netauto$ ansible-playbook play_gather-facts.yml -i inventories/dev/

PLAY [Play - IOS audit report] ************************************************************************************************************************************************************************************

TASK [Task 1 - Gathering IOS facts] *******************************************************************************************************************************************************************************
ok: [ios-xe-mgmt.cisco.com]

TASK [debug] ******************************************************************************************************************************************************************************************************
ok: [ios-xe-mgmt.cisco.com] => 
  facts:
    ansible_facts:
      ansible_net_all_ipv4_addresses:
      - 9.9.9.9
      - 1.1.1.4
      - 1.2.3.4
      - 10.10.1.5
      - 13.13.13.13
      - 89.89.89.89
      .
      .
      <output truncated for readability>
      .
      .
      ansible_net_hostname: csr1000v
      ansible_net_image: bootflash:packages.conf
      ansible_net_iostype: IOS-XE
      ansible_net_memfree_mb: 2068652
      ansible_net_memtotal_mb: 2392443
      ansible_net_model: CSR1000V
      ansible_net_neighbors: {}
      ansible_net_python_version: 2.7.17
      ansible_net_serialnum: 98BH7LU5M3H
      ansible_net_system: ios
      ansible_net_version: 16.09.03
      .
      .
      <output truncated for readability>

PLAY RECAP ********************************************************************************************************************************************************************************************************
ios-xe-mgmt.cisco.com      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

The Play Recap shows that this playbook ran successfully and made no changes to the end device(s), it simply used the _ios_facts_ module to retrieve information in key-value pairs. 

* **Task 1:** used the _ios_facts_ module to gather a base set of facts from each device, registering the output to the variable _"facts"_.
* **Task 2:** (debug:) a temporary information gathering task that used the [debug](https://docs.ansible.com/ansible/latest/modules/debug_module.html "debug") module to print the _"facts"_ variable registered in Task 1.

This information will be used in the next step to generate a report.

### Step 3: Add reporting tasks to the playbook

We will now edit the playbook, removing the debug task and creating new tasks that:

* Create a folder to store our device reports.
* Use a Jinja2 template to dynamically generate a report file for each device.
* Consolidate each individual device report into a single report.

Use nano or a text-editor of your choice to update the playbook `play_gather-facts.yml` with the below tasks:

```yaml
---                                                  ## signals the start of a YAML file

- name: Play - IOS audit report                      ## name for the playbook, included in the output of a playbook run
  hosts: ios                                         ## list of one or more inventory groups, separated by colons
  connection: network_cli                            ## plugin for a CLI connection to network devices
  gather_facts: no                                   ## 'no' is required for Ansible versions prior to 2.9

  tasks:                                             ## start of the tasks list, executed from top to bottom on each network device in the hosts list

  - name: Task 1 - Gathering IOS facts               ## name for task 1, included in the output of a playbook run
    ios_facts:                                       ## ios_facts is the Ansible module we are invoking

  - name: Task 2 - Creating a reports folder (if not already present)
    file:                                            ## file is the Ansible module we are invoking
      name: reports                                  ## the name to give the new directory
      state: directory                               ## parameter of the file module, creates a directory
    run_once: true                                   ## instructs Ansible to only run this task once for the batch of hosts targeted in the playbook

  - name: Task 3 - Generating the device reports
    template:                                        ## template is the Ansible module we are invoking
      src: ios_report.j2                             ## parameter of the template module, path to the Jinja2 template on our Ansible control node
      dest: reports/{{ inventory_hostname }}.md      ## parameter of the template module, location to render the Jinja2 template
  
  - name: Task 4 - Consolidating the device reports
    assemble:                                        ## assemble is the Ansible module we are invoking
      src: reports/                                  ## parameter of the assemble module, path to the directory containing the individual device reports
      dest: reports/ios_network_report.md            ## parameter of the assemble module, the file to create using the concatenation of the individual device reports
    delegate_to: localhost                           ## runs this task on our Ansible control node
    run_once: true
```

If we were to run this playbook now it would fail at task 3 as we have not yet created the Jinja2 template referenced in that task _(ios_report.j2)_.

```bash
chhannan@ubuntu:~/netauto$ ansible-playbook play_gather-facts.yml -i inventories/dev/

PLAY [Play - IOS audit report] ************************************************************************************************************************************************************************************

TASK [Task 1 - Gathering IOS facts] *******************************************************************************************************************************************************************************
ok: [ios-xe-mgmt.cisco.com]

TASK [Task 2 - Creating a reports folder (if not already present)] ************************************************************************************************************************************************
changed: [ios-xe-mgmt.cisco.com]

TASK [Task 3 - Generating the device reports] *********************************************************************************************************************************************************************
fatal: [ios-xe-mgmt.cisco.com]: FAILED! => changed=false 
  msg: |-
    Could not find or access 'ios_report.j2'
    Searched in:
            /netauto/templates/ios_report.j2
            /netauto/ios_report.j2
            /netauto/templates/ios_report.j2
            /netauto/ios_report.j2 on the Ansible Controller.
    If you are using a module and expect the file to exist on the remote, see the remote_src option
  to retry, use: --limit @/netauto/play_gather-facts.retry

PLAY RECAP ********************************************************************************************************************************************************************************************************
ios-xe-mgmt.cisco.com      : ok=2    changed=1    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0

chhannan@ubuntu:~/netauto$ 
```

### Step 4: Create the Jinja2 template referenced in Task 3

We will now use the key-value pairs retrieved in Step 2 of this guide to create a template of how we want our report file to look and what information it should hold about each device targeted in the playbook run. The template will use the keys from facts gathered by the _ios_facts_ module as variables to dynamically render a device report.

In the root of the `netauto/` directory, create a new file named `ios_report.j2`.

```bash
cphannan@ubuntu:~/netauto$ touch ios_report.j2
```

Use nano or a text-editor of your choice to paste in the below template:

```jinja2
---
**{{ inventory_hostname.upper() }}**

Model: {{ ansible_net_model }}

Serial Number: {{ ansible_net_serialnum }}

Software Type: {{ ansible_net_iostype }}

Version: {{ ansible_net_version }}

Image: {{ ansible_net_image }}
```

> **Note:** we manipulate the {{ inventory_hostname }} variable by applying the .upper() filter to make the rendered output of this variable uppercase.

### Step 5: Run the updated playbook

```bash
chhannan@ubuntu:~/netauto$ ansible-playbook play_gather-facts.yml -i inventories/dev/

PLAY [Play - IOS audit report] ************************************************************************************************************************************************************************************

TASK [Task 1 - Gathering IOS facts] *******************************************************************************************************************************************************************************
ok: [ios-xe-mgmt.cisco.com]

TASK [Task 2 - Creating a reports folder (if not already present)] ************************************************************************************************************************************************
changed: [ios-xe-mgmt.cisco.com]

TASK [Task 3 - Generating the device reports] *********************************************************************************************************************************************************************
changed: [ios-xe-mgmt.cisco.com]

TASK [Task 4 - Consolidating the device reports] ******************************************************************************************************************************************************************
changed: [ios-xe-mgmt.cisco.com -> localhost]

PLAY RECAP ********************************************************************************************************************************************************************************************************
ios-xe-mgmt.cisco.com      : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

The Play Recap shows that this playbook ran successfully and 3 of the 4 tasks registered changes:

* **Task 1:** no changes -  used the _ios_facts_ module to gather a base set of facts from each device.
* **Task 2:** changed - created a `reports/` directory. Subsequent runs of this playbook would register no changes for this task.
* **Task 3:** changed - used the Jinja2 template to render device reports in the `/reports` directory. Subsequent runs of this playbook would register no changes for this task until we either: 
  * modified the template
  * updated our target inventory to include new hosts
* **Task 4:** changed - creates a new "master" report that concatenates all of the device reports into a single report file. Subsequent runs of this playbook would register no changes for this task until we either: 
  * modified the template
  * updated our target inventory to include new hosts

> **Note:** In this example we only target a single IOS device so the individual report and the "master" report will be identical. If we were to add more devices to the `[ios]` group in our `/dev` inventory, the report generated in Task 4 (_ios_network_report.md_) would contain every device's individual report.

### Step 6: View the rendered reports

From the root of the `netauto/` directory we should now see a `reports/` directory that contains the two report files:

```bash
chhannan@ubuntu:~/netauto$ tree 
.
├── ansible.cfg
├── inventories
│   └── dev
│       ├── group_vars
│       │   ├── aci.yml
│       │   ├── iosxr.yml
│       │   ├── ios.yml
│       │   └── nxos.yml
│       ├── hosts
│       └── host_vars
│           ├── ios-xe-mgmt.cisco.com
│           │   └── ios-xe-mgmt.cisco.com.yml
│           ├── sandboxapicdc.cisco.com
│           │   └── sandboxapicdc.cisco.com.yml
│           ├── sbx-iosxr-mgmt.cisco.com
│           │   └── sbx-iosxr-mgmt.cisco.com.yml
│           └── sbx-nxos-mgmt.cisco.com
│               └── sbx-nxos-mgmt.cisco.com.yml
├── ios_report.j2                                  ## jinja2 template created in step 4
├── play_gather-facts.yml                          ## playbook created in step 1
└── reports                                        ## reports directory created in task 2 of playbook
    ├── ios_network_report.md                      ## device reports created in task 3 of playbook
    └── ios-xe-mgmt.cisco.com.md                   ## master report created in task 4 of playbook

9 directories, 14 files
```

And the rendered Markdown reports would look like this:

---
**IOS-XE-MGMT.CISCO.COM**

Model: CSR1000V

Serial Number: 98BH7LU5M3H 

Software Type: IOS-XE

Version: 16.09.03

Image: bootflash:packages.conf

## Takeaways

* **Benefits of network automation:** this post should hopefully highlight the power of Ansible and Jinja2 templates for quickly generating inventory reports. Although we only targeted a single device, this could be scaled up to 100s or 1000s of devices.

* **Flexibility  of network automation:** the Jinja2 template we created and the file extension we used in Tasks 3 and 4 of the playbook resulted in our reports being generated in Markdown. This could easily be changed to output our reports to .csv by changing the file extensions in tasks 3 and 4 to .csv and amending the Jinja2 template to look like this:

**ios_report.j2**

```jinja2
Hostname,Model,Serial Number,Software Type,Version,Image
{{ inventory_hostname }},{{ ansible_net_model }},{{ ansible_net_serialnum }},{{ ansible_net_iostype }},{{ ansible_net_version }},{{ ansible_net_image }}
```

* **Future improvements:** 
  * this playbook could be expanded to include other network OSs such as IOS-XR and NXOS.
  * this playbook could be turned in to a dedicated "reports" role that generates reports for your multi-vendor, multi-OS network estate.
  * as of Ansible 2.9, we could enable facts gathering within the play definition (top section) of the playbook `gather_facts: yes` and remove the need to include it as task 1. Ansible will then gather facts from the devices targeted in the `hosts` key.

The code from this post is available on [Github](https://github.com/cphannan/netauto "Github"):