# ansible-switch-setup

This collection contains Ansible roles and playbook examples to manage the VLANs in multiple layer 2 domains consisting of Cisco IOS and NXOS switches. It has been created and tested under Ansible 2.10.2.

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

### Confidential data

It is highly recommended to encrypt sensitive data like the value of `ansible_ssh_pass`. This can be done using Ansible Vault:

```bash
mboehm21@workstation:~/ansible-switch-setup$ set +o history
mboehm21@workstation:~/ansible-switch-setup$ ansible-vault encrypt_string --name 'ansible_ssh_pass' 'mysecret'
New Vault password: 
Confirm New Vault password: 
ansible_ssh_pass: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          65393764633861386365303762373239343364303435336436373164666262386433306132326565
          3734303261643966386634343431323461376532326236640a313531663864363161633837363930
          33643461656462346439326661313238613134363733653366653865643038326465313639633739
          3737313332613562660a636639663239656633636664323032373833353930386365393334663063
          3932
Encryption successful
mboehm21@workstation:~/ansible-switch-setup$ set -o history
```

Afterwards you can replace the cleartext variable in your inventory file with the encrypted string. When executing playbooks you need to use `--ask-vault-password` to tell Ansible to query your Vault password to decrypt the variable.

## Roles

The following roles can be used in playbooks to perform certain actions on the switches.

- collect_facts
- list_switch_inventory
- list_vlans
- deploy_vlans

Their usage is demonstrated in the example playbooks.

## Playbooks

Please note that the playbooks are examples using the generic hosts-file. Make sure to adapt them to match the groups with your own infrastructure and hosts-file.

A playbook for the testing-domain defined in the inventory above could be called `testing_list_vlans.yml` and would look like that:

```yaml
---

- name: "Gather VLANs of all switches of the testing-domain"
  gather_facts: no
  hosts:
    - testing

  roles:
    - role: collect_facts
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

This playbook creates a file `data/<site_name>/master.yml` containing information about the configured VLANs on the master-swich. The format is the same as shown above. This is the reference for the VLAN database of the whole domain. It can be modified in a text editor before deploying the vlans to all switches using the next playbook.

#### site_a_deploy_vlans.yml

This playbook reads the file `data/<site_name>/master.yml` and compares the VLAN databases of all switches in the group to it. Afterwards it lists all the switches where changes are needed and prompts the user whether or not the changes should be made to the switches.

To avoid service impact due to unintended changes to the VLAN databases it is recommended to compare the current VLANs of each switch before running this playbook. This can be done using the UNIX tool `diff`:

```bash
mboehm21@workstation:~/ansible-switch-setup/data/SITE_A_vlan_db$ diff -u basement-02_192.168.120.253.yml master.yml | grep -e "^[+-] "
-  - vlan_id: 600
-    name: VLAN0400
+  - vlan_id: 500
+    name: VLAN0500
```

In the above example VLAN 600 would be deleted and VLAN 500 would be added to the switch basement-02 when running the playbook.
