# Description

The objective of this project is to be able to push, remove, and view configurations safely on network devices by executing Ansible Playbooks via an Ansible managed node.

The desired network topology is as follows:

\<Insert Network Diagram Here\>

For more visibility into the topology, you can view the desired configurations if they were to be applied via the command line on each device [here]().

# Setting up the Devices

As per the topology design, to allow connectivity between the Ansible management node and the network devices, it is required to provide the configurations present [here]() for the network devices.

For the Ansible management node, the Linux distribution used in this project is CentOS Stream 9. Steps required to configure the system is as follows:

- We’ll start with the minimal install.

- Initially, we’ll need to disable default routing on ens192.

```console
nmcli con modify ens192 ipv4.never-default yes
nmcli connection up ens192
```

- Then, we’ll update our repos.

```console
sudo yum update -y
```

- We’ll install the following packages:
    - net-tools
    - vim
    - git
    - epel-release
    - ansible
    - tree

```console
sudo yum install <packagename> -y
```

- Next, we’ll install paramiko and ansible-pylibssh.

```console
sudo yum install python-devel -y
sudo yum install libffi-devel -y
sudo yum install openssl-devel -y
sudo yum install python-pip -y
pip install paramiko
pip install --upgrade pip
pip install --user ansible-pylibssh
```

- Next, we’ll apply the LEGACY cryptographic policy in order to allow proper SSH communications to and from the network devices.

```console
sudo update-crypto-policies --set LEGACY
sudo reboot
```

- Next, we’ll create a user-specific SSH RSA key to use with both our GitHub repo and network devices.

```console
ssh-keygen -t RSA
```

>Note: Determine the path to be /home/malaa/.ssh/malaa and use a passphrase.

- Next, we’ll have to enable one of the ciphers that work with IOS devices for some of the devices (for example R1) by editing the ~/.ssh/config file as follows: THIS STEP MAY NOT BE NECESSARY
 
- Next, we’ll upload the public key to GitHub and clone the repository via the SSH URL using git clone \<url\>.

- Finally, connect to each device via SSH to establish authenticity and fill the ~/.ssh/known_hosts file.


Now that all the devices are ready, it’s time to configure Ansible itself.

# Setting up Ansible

- **inventory** file

```txt
[cisco_routers]
R1 ansible_host=10.10.90.1

[core_cisco_switches]
Core1 ansible_host=10.10.90.201

[dist_cisco_switches]
Dist1 ansible_host=10.10.90.202

[access_cisco_switches]
AS1 ansible_host=10.10.90.203
AS2 ansible_host=10.10.90.204
AS3 ansible_host=10.10.90.205

[cisco_switches:children]
core_cisco_switches
dist_cisco_switches
access_cisco_switches
```

- **ansible.cfg** file

```ini
[defaults]
inventory = inventory
remote_user = malaa
action_warnings = false
```

- **group_vars/all.yml** file

```yaml
ansible_connection: ansible.netcommon.network_cli
ansible_network_os: cisco.ios.ios
state_typeA_global: merged
prepend_info: None

users:
  malaa:
    privilege: 15
    secret: 9
    hashed_password: $9$jRNqe2Di2gdJIt$/IIA8SceE97JMLahfTU/0LQvMld/Sokg/tmFtM0zDm.
    disabled: username #Set to no username to remove.
    sshkey: "{{ lookup('file','~/.ssh/malaa.pub') }}"
    state: present #Set to absent to remove the ssh key for this user.
  test:
    privilege: 15
    secret: 9
    hashed_password: $9$iMshSdwwDdrTE2$.qzRj/zrcOrVAHr3B2J7e9jQv.cU.gKx2C3x/YEUIe6
    disabled: username #Set to no username to remove.
    sshkey: "{{ lookup('file','~/.ssh/test.pub') }}"
    state: present #Set to absent to remove the ssh key for this user.
```

>Note: Variables other than ansible_connection and ansible_network_os will be explained later.

Now we’re ready to dive into our playbooks.

# Playbooks:

## initial_config_playbook.yml

>**IOS modules used here are:** ios_config

This playbook creates a snapshot of the initial configuration that allows connectivity between the Ansible management node and the network devices. The aim of this playbook is to create a backup that can be later restored and can be used for comparison after applying configurations using other playbooks.

>Note: You have to run this playbook before applying any configurations, otherwise, it’ll be inaccurate.

```yaml
---

- hosts: all
  gather_facts: false
  tasks:

    - name: Backing up initial configurations
      cisco.ios.ios_config:
        backup: true
        backup_options:
          filename: "{{inventory_hostname}}_IC.cfg"
          dir_path: backup/initial_confs
```

You can run this playbook using the command:

```console
ansible-playbook initial_config_playbook.yml -k
```

## RO_config_playbook.yml

>*IOS modules used here are:* ios_config

This playbook's main task is to configure DHCP and L3 Interfaces on Cisco routers. It imports both the dhcp and l3_interfaces roles which rely on iterating to configure the routers. Iterations are applied on dictionaries containing the required configurations that are present in a host_vars file named after the router’s ansible_host name provided in the inventory file.

*RO_config_playbook.yml*

```yaml
- hosts: cisco_routers
  gather_facts: false
  tasks:
    - name: Configure DHCP pools
      tags: dhcp, d_dhcp
      import_role:
        name: dhcp

    - name: Configure L3_Interfaces
      tags: l3_interfaces, d_l3_interfaces
      import_role:
        name: l3_interfaces
```

*roles/dhcp/tasks/main.yml*

```yaml
#If you're going to prepend any lines with no, uncomment vars and prepend_info

- name: Configure DHCP pools
  #vars:
    #prepend_info: Some
  cisco.ios.ios_config:
    before: "ip dhcp excluded-address {{ item.value.excluded_address }}"
    lines:
      - "network {{ item.value.network_id }}"
      - "default-router {{ item.value.default_router_ip }}"
      - dns-server 8.8.4.4
    parents: "ip dhcp pool {{ item.key }}"
  loop: "{{ pools | dict2items }}"
  register: dhcp_conf
```

*roles/l3_interfaces/tasks/main.yml*

```yaml
#If you're going to prepend any lines with no, uncomment vars and prepend_info
- name: Enable L3_Interfaces
  #vars:
    #prepend_info: Some
  cisco.ios.ios_config:
    lines:
      - no shutdown
    parents: interface FastEthernet0/0

- name: Configure L3_Interfaces
  cisco.ios.ios_config:
    lines:
      - "encapsulation dot1Q {{ item.value.encapsulation}}"
      - "ip address {{ item.value.ip_address}}"
    parents: "interface {{ item.key }}"
  loop: "{{ interfaces | dict2items }}"
  register: l3_interfaces_conf
```

*host_vars/R1.yml*

```yaml
pools:
  Servers:
    excluded_address: "10.10.100.1"
    default_router_ip: "10.10.100.1"
    network_id: "10.10.100.0 255.255.255.0"
  IT:
    excluded_address: "10.10.90.1"
    default_router_ip: "10.10.90.1"
    network_id: "10.10.90.0 255.255.255.0"
  OP:
    excluded_address: "10.10.110.1"
    default_router_ip: "10.10.110.1"
    network_id: "10.10.110.0 255.255.255.0"

interfaces:
  FastEthernet0/0.10:
    encapsulation: "10"
    ip_address: "10.10.100.1 255.255.255.0"
  FastEthernet0/0.20:
    encapsulation: "20"
    ip_address: "10.10.90.1 255.255.255.0"
  FastEthernet0/0.30:
    encapsulation: "30"
    ip_address: "10.10.110.1 255.255.255.0"
```

>Note: If you want to add more pools or more interfaces, simply follow the same formatting. If you also want to add more Cisco Routers, simply add them to the inventory file while following the same formatting and copy the R1 host variables file, rename it to -for example- R2.yml, and edit as required.

### Additional Capabilites:

You can view the full playbook [here](), however, you'll notice that the playbook is much larger, this is because it has been appended with a couple capabilites that allows the admin to:

1. Confirm changes before applying them:

This is done via the RO_confirmation role which uses the debug module to provide information and the pause module to offer the choice of continuing to run the playbook -therefore applying the changes- or cancelling.

*roles/RO_confirmation/tasks/main.yml*

```yaml
- name: Displaying DHCP Configuration Information
  tags: dhcp, d_dhcp
  debug:
    msg:
      - "{{ prepend_info }} of the commands here are prepended with no. The values without any prepend are as shown."
      - "ip dhcp pool {{ item.key }}"
      - "ip dhcp excluded-address {{ item.value.excluded_address }}"
      - "network {{ item.value.network_id }}"
      - "default-router {{ item.value.default_router_ip }}"
      - "dns-server 8.8.8.8"
  loop: "{{ pools | dict2items }}"

- name: Displaying L3_Interfaces Configuration Information
  tags: l3_interfaces, d_l3_interfaces
  debug:
    msg:
      - "{{ prepend_info }} of the commands here are prepended with no. The values without any prepend are as shown."
      - "interface FastEthernet0/0"
      - "no shutdown"
      - "interface {{ item.key }}"
      - "encapsulation dot1Q {{ item.value.encapsulation}}"
      - "ip address {{ item.value.ip_address}}"
  loop: "{{ interfaces | dict2items }}"

- name: Confirmation
  tags: always
  pause:
    prompt: "You're about to apply the previous configurations. To abort click CTRL+C then A, to continue, click Enter."
```

The prepend_info variable gives the reader awareness of whether any of the configurations in the dhcp or l3_interfaces roles are appended with no, therefore, prompting them to check these roles before applying the configuration.

>Note: If you’re going to reconfigure any of the tasks in the playbook or add new ones, you’ll have to reconfigure the confirmation role as well -whether adding more tasks in it or reconfiguring ones-.

2. Backup current running configuration before applying:

*RO_config_playbook.yml*

```yaml
    - name: Backup Configuration
      tags: always
      cisco.ios.ios_config:
        backup: true
```

3. Save changes to the startup configuration after applying:

This is done via the save role which also uses the pause module offer the choice of continuing to run the playbook -therefore saving the configuration to the startup- or cancelling.

*roles/save/tasks/main.yml*

```yaml
- name: Prompt to Save Configurations
  pause:
    prompt: "You're about to save configurations. To abort click CTRL+C then A, to continue, click Enter."

- name: Save Configurations
  cisco.ios.ios_config:
    config:
      lines: "file prompt quiet" #Removes confirmation upon save in network devices.
    save_when: always
```

4. Provide a way to debug changes in two different methods:

Method 1: Using the debug module to view returned values which provide many different info depending on the module and parameters.

*RO_config_playbook.yml*

```yaml
    - name: Debug
      tags: never, debug
      block:
        - name: Debug DHCP pools
          tags: debug, d_dhcp
          debug:
            var: dhcp_conf

        - name: Debug L3_Interfaces
          tags: debug, d_l3_interfaces
          debug:
            msg:
              - "{{ l3_interfaces_conf }}"
              - "Interface FastEthernet0/0 has been set to administratively up"
```

Method 2: Using the compare_before_task and compare_after_task tasks which make use of the ios_config module capabilities to provide a more pleasing comparison that shows what was added and what was removed.

*tasks/compare_before_task.yml*

```yaml
- name: Backup for Comparison (Before)
  cisco.ios.ios_config:
    backup: true
    backup_options:
      filename: "{{ inventory_hostname }}_cmp_before.cfg"
      dir_path: backup/compare_confs
```

*tasks/compre_after_task.yml*

```yaml
- name: Compare with Previous Configuration
  block:
    - name: Backup for Comparison (After)
      cisco.ios.ios_config:
        backup: true
        backup_options:
          filename: "{{ inventory_hostname }}_cmp_after.cfg"
          dir_path: backup/compare_confs
    - name: Compare with Previous Configuration
      cisco.ios.ios_config:
        running_config: "{{ lookup('file', 'backup/compare_confs/{{ inventory_hostname }}_cmp_before.cfg') }}"
        diff_against: intended
        intended_config: "{{ lookup('file', 'backup/compare_confs/{{ inventory_hostname }}_cmp_after.cfg') }}"
```

### Tags Design:

The only task that will always run is the Backup Configuration task. There are tasks that will always run but can be cancelled. These tasks are ones related to confirmation and comparison as they have a tag other than always allowing you to use --skip-tags \<other tag name\> to skip them.

There are tasks that will never run unless specified. These tasks are ones related to debugging and the Save Configurations task as they have a tag other than never allowing you to use --tags <all or required tags\> \<other tag name\>.

You can specify which tasks to run from the core tasks of the playbook by specifying the tag assigned to it. You can also specify if you wish to view debugging information related to it.

>Note: This also applies to the SW_config_playbook and the users_config_playbook.

### Executing the Playbook:

>Note: This section applies to the SW_config_playbook and the users_config_playbook with the only differences being in the name of the executed playbook and some of the tags -such as dhcp and d_dhcp-.

With tags in mind, you can for example run the following commands to execute the playbook:

```console
ansible-playbook RO_config_playbook.yml -k --diff
```

This will run all the tasks without debugging or saving. Confirmation and comparison tasks run normally.

```console
ansible-playbook RO_config_playbook.yml -k --diff --tags all,save
```

This will run all the tasks without debugging but configurations will be saved. Confirmation and comparison tasks run normally.

```console
ansible-playbook RO_config_playbook.yml -k  --tags all,save --skip-tags confirmation
```

This will run the dhcp task without debugging and configurations will be saved. Confirmation and comparison tasks run normally.

```console
ansible-playbook RO_config_playbook.yml -k --diff --tags dhcp,save
```

This will run the dhcp task without debugging and configurations will be saved. Confirmation for the dhcp task only runs and comparison tasks run normally.

```console
ansible-playbook RO_config_playbook.yml -k --diff --tags d_dhcp
```

This will run the dhcp task with debugging and configurations won’t be saved. Confirmation for the dhcp task only runs and comparison tasks run normally.

>Note:
>- You can use any combination of tags but it’s not recommended to use the always or never tags as their assignment wasn’t designed around them being skipped or specifically specified during execution.
>- You can also use --limit \<hostname\>, however, we only have one Cisco router, therefore, it isn't useful in this scenario. This is useful on the other playbooks though.

### Removing Configurations:

If you wish to remove configurations, you have to manually append 'no' to lines under the ios_config module as it doesn't support states.

Another method you can use is to rollback to the initial configuration using the restore_config_playbook -discussed later-.

## SW_config_playbook.yml

IOS modules used here are: ios_config, ios_vlans, ios_l2_interfaces, ios_l3_interfaces

This playbook's main task is to configure VLANs, L2 Interfaces, and L3 Management Interfaces on Cisco switches. It imports  the vlans, l2_interfaces, mgmt_interfaces roles which rely on iterating to configure the switches. Iterations are applied on dictionaries containing the required configurations that are present in a host_vars file named after the each switch's ansible_host name provided in the inventory file.

*SW_config_playbook.yml*

```yaml
- hosts: cisco_switches
  gather_facts: false
  tasks:
    - name: Configure VLANs
      tags: vlans, d_vlans
      import_role:
        name: vlans #Role has module states configured to merged by default, to override, change the state_typeA_global variable in group_vars/all.yml

    - name: Configure L2_Interfaces
      tags: l2_interfaces, d_l2_interfaces, tr_d_l2_interfaces, acc_d_l2_interfaces
      import_role: #Role has module states configured to merged by default, to override, change the state_typeA_global variable in group_vars/all.yml
        name: l2_interfaces

    - name: Configure Management Interfaces
      tags: mgmt_interfaces, d_mgmt_interfaces
      import_role: #Role has module states configured to merged by default, to override, change the state_typeA_global variable in group_vars/all.yml
        name: mgmt_interfaces
```

*roles/vlans/tasks/main.yml*

```yaml
- name: Configure VLANs
  cisco.ios.ios_vlans:
    config:
      - name: "{{ item.value.name }}"
        vlan_id: "{{ item.value.vlan_id }}"
    state: "{{ item.value.state_typeA }}"
  loop: "{{ vlans | dict2items }}"
  register: vlans_conf
```

*roles/l2_interfaces/tasks/main.yml*

```yaml
- name: Configure Trunk L2_Interfaces
  cisco.ios.ios_l2_interfaces:
    config:
      - name: "{{ item.key }}"
        mode: trunk
        trunk:
          allowed_vlans: "{{ item.value.allowed_vlans }}"
          encapsulation: dot1q
    state: "{{ item.value.state_typeA }}"
  loop: "{{ l2_interfaces | dict2items }}"
  when: item.value.mode == "trunk"
  register: trunk_conf

- name: Configure Access L2_Interfaces
  cisco.ios.ios_l2_interfaces:
    config:
      - name: "{{ item.key }}"
        mode: access
        access:
          vlan: "{{ item.value.vlan }}"
    state: "{{ item.value.state_typeA }}"
  loop: "{{ l2_interfaces | dict2items }}"
  when: item.value.mode == "access"
  register: access_conf
```

*roles/mgmt_interfaces/tasks/main.yml*

```yaml
- name: Configure Management Interfaces
  cisco.ios.ios_l3_interfaces:
    config:
      - name: vlan 20
        ipv4:
          - address: "{{ mgmt_ip_address}}"
    state: merged
  register: mgmt_interfaces_conf

- name: Enable Management Interfaces
  cisco.ios.ios_config:
    lines:
      - no shutdown
    parents: interface vlan 20
```

*host_vars/AS2.yml*

```yaml
vlans:
  VLAN10_Servers:
    name: Servers
    vlan_id: 10
    state_typeA: "{{ state_typeA_global }}"
  VLAN20_IT:
    name: IT
    vlan_id: 20
    state_typeA: merged

l2_interfaces:
  Ethernet0/0:
    mode: trunk
    allowed_vlans: 10,20
    state_typeA: merged
  range Ethernet0/1-3:
    mode: access
    vlan: 10
    state_typeA: "{{ state_typeA_global }}"
  range Ethernet1/0-3:
    mode: access
    vlan: 10
    state_typeA: "{{ state_typeA_global }}"

mgmt_ip_address: "10.10.90.204 255.255.255.0"
```

To view the rest of the host variables files, click [here]()

>Note:
>- If you want to add more vlans or more interfaces, simply follow the same formatting. If you also want to add more Cisco Switches, simply add them to the inventory file while following the same formatting and copy the host variables file of an existing switch -which one depends on the type of switch you're installing- then rename and edit as required.

You can view the full playbook [here]() with the appended capabilites mentioned in RO_config_playbook. The only difference lies in the confirmation role.

*roles\SW_confirmation\tasks\main.yml*

```yaml
- name: Displaying VLANs Configuration Information
  tags: vlans, d_vlans
  debug:
    msg:
      - "State is set to {{ item.value.state_typeA }}"
      - "vlan {{ item.value.vlan_id }}"
      - "name {{ item.value.name }}"
  loop: "{{ vlans | dict2items }}"

- name: Displaying L2_Trunk_Interfaces Configuration Information
  tags: l2_interfaces, d_l2_interfaces, tr_d_l2_interfaces
  debug:
    msg:
      - "State is set to {{ item.value.state_typeA }}"
      - "interface {{ item.key }}"
      - "switchport trunk encapsulation dot1q"
      - "switchport mode trunk"
      - "switchport trunk allowed vlan {{ item.value.allowed_vlans }}"
  loop: "{{ l2_interfaces | dict2items }}"
  when: item.value.mode == "trunk"

- name: Displaying L2_Access_Interfaces Configuration Information
  tags: l2_interfaces, d_l2_interfaces, acc_d_l2_interfaces
  debug:
    msg:
      - "State is set to {{ item.value.state_typeA }}"
      - "interface {{ item.key }}"
      - "switchport mode access"
      - "switchport access vlan {{ item.value.vlan }}"
  loop: "{{ l2_interfaces | dict2items }}"
  when: item.value.mode == "access"

- name: Displaying Management Interfaces Configuration Information
  tags: mgmt_interfaces, d_mgmt_interfaces
  debug:
    msg:
      - "State is set to merged"
      - "interface vlan 20"

- name: Confirmation
  tags: always
  pause:
    prompt: "You're about to apply the previous configurations. To abort click CTRL+C then A, to continue, click Enter."
```

### Removing Configurations:

Removing configurations is easier/more automated in this playbook due to the capability of using states lying in the modules it uses.

For more on how different states operate, click [here]().

States are assigned a variable state_typeA_global whose value is set to merged by default in the all.yml group variables file to prevent any changes from deleting whole parts of the configuration already existing on the switches. This variable is referenced by each host's variables file in a way that is designed to prevent any changes from affecting connectivity to the network devices.

If you wish remove whole parts of the configuration of switches, the recommended method is to change the state to deleted by changing the value of the state_TypeA_global variable then executing the playbook while limiting the effect to a certain host and to certain tasks by using tags.

You also have the option to do minute deletions by changing the dict.key.value.state_typeA in each host variable's file.

So for example, you could remove the configuration of the L2 interfaces by applying the command:

```console
ansible-playbook SW_config_playbook.yml -k --limit AS2 --tags l2_interfaces --diff
```

Another method you can use is to rollback to the initial configuration using the restore_config_playbook -discussed later-, however, you should keep in mind that VLANs do not show up on the startup or running-config, therefore, won't be stored in the initial configuration backups taken, therefore, they will not be deleted using that method.
