# ansible-switch-setup

This respository contains Ansible roles and playbook examples to manage the VLANs in multiple layer 2 domains consisting of Cisco IOS and NXOS switches. It has been created and tested under Ansible 2.10.2.

## Inventory

The hosts-file has to be adapted to match your infrastructure first. It is needed to tell Ansible which switches are available on which site, their type and which credentials should be used to login.

Furthermore it is necessary to define a single master-switch per site. This switch will be queried to create a master-vlan file.

Example with two sites (production and testing):

```yaml
---

all:
  children:
    production:
      children:

        ios_switches:
          hosts:
            switch1.prod:
              master_switch: yes
            switch2.prod:
          vars:
            ansible_network_os: cisco.ios.ios
            ansible_become: yes
            ansible_become_method: enable

        nxos_switches:
          hosts:
            switch3.prod:
          vars:
            ansible_network_os: cisco.nxos.nxos
            ansible_become: yes
            ansible_become_method: enable

      vars:
        site_name: PRODUCTION_DOMAIN

    testing:
      children:

        ios_switches:
          hosts:
            switch1.test:
              master_switch: yes
            switch2.testing:
            switch3.testing:
          vars:
            ansible_network_os: cisco.ios.ios
            ansible_become: yes
            ansible_become_method: enable

      vars:
        site_name: TESTING_DOMAIN

  vars:
    ansible_connection: ansible.netcommon.network_cli
    ansible_ssh_user: myusername
    ansible_ssh_pass: mypassword
 ```
## Roles

The following roles can be used in playbooks to perform certain actions on the switches.

- list_switch_inventory
- list_vlans

Their usage is demonstrated in the example playbooks.

### deploy_vlans

## Playbooks

Please note that the playbooks are examples using the generic hosts-file. Make sure to adapt them to match the groups with your own infrastructure and hosts-file.

A playbook for the testing-domain defined in the inventory above could be called `testing_list_vlans.yml` and would look like that:

```yaml
---

- name: "Gather VLANs of all switches of the testing-domain"
  gather_facts: yes
  hosts:
    - testing

  roles:
    - role: list_vlans
```
### Functionality of the example playbooks

#### site_a_list_switch_inventory.yml

This playbook creates a file `data/SITE_A_switch_inventory.yml` that contains an overview of all the switches of the given group in yaml-format.

Example:

```yaml
---

192.168.120.253:
  hostname: basement-02 
  model: WS-C2960G-24TC-L
  serial: FOCxxxxxxxx
  os: IOS 15.0(2)SE2
  image: flash:c2960-lanbasek9-mz.150-2.SE2.bin
```

#### site_a_list_vlans.yml

This playbook creates one file per switch `data/<site_name>/<hostname>_<ip address>.yml` containing information about the configured VLANs in yaml-format.

Example:

```yaml
---
 
vlan_db:
  - vlan_id: 1
    name: default
    state: active
  - vlan_id: 200
    name: LAN
    state: active
  - vlan_id: 300
    name: LAB
    state: active
  - vlan_id: 400
    name: VLAN0400
    state: active
  - vlan_id: 1002
    name: fddi-default
    state: active
  - vlan_id: 1003
    name: token-ring-default
    state: active
  - vlan_id: 1004
    name: fddinet-default
    state: active
  - vlan_id: 1005
    name: trnet-default
    state: active
```

#### site_a_get_vlans_from_master.yml

This playbook creates a file `data/<site_name>/master.yml` containing information about the configured VLANs on the master-swich. The format is the same as shown above. This is the reference for the vlan database of the whole domain. It can be modified in a text editor before deploying the vlans to all switches using the next playbook.

#### site_a_deploy_vlans.yml

This playbook reads the file `data/<site_name>/master.yml` and compares the vlan-databases of all switches in the group to it. Afterwards it lists all the switches where changes are needed and prompts the user whether or not the changes should be made to the switches.
