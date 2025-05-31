# MN-521-Assignment-2
# ansible.cfg
[defaults]
inventory = ./inventory/hosts.ini
host_key_checking = False
timeout = 30
retry_files_enabled = False
forks = 20
gathering = smart

[privilege_escalation]
become = True
become_method = enable
become_user = root

[ssh_connection]
pipelining = True
control_path = %(directory)s/%%h-%%r
persistent_command_timeout = 60

[defaults]
# Ensure modules for network devices are auto‐loaded
nocows = 1
[cisco_ios]
sw1 ansible_host=192.168.100.11
r1  ansible_host=192.168.100.12
sw2 ansible_host=192.168.100.13

[juniper_junos]
srx ansible_host=192.168.100.21

[all:vars]
ansible_user=admin
ansible_password=cisco123
ansible_become_password=cisco123
---
# Common variables for all devices:
ansible_network_os: null   # Will be overridden per group
ansible_connection: network_cli
ansible_user: admin
ansible_password: cisco123
ansible_become: yes
ansible_become_method: enable
ansible_become_password: cisco123
---
- name: “Configure VLANs and Trunks on Cisco IOS switches”
  hosts: cisco_ios
  gather_facts: no
  connection: network_cli
  vars:
    vlan_list:
      - id: 10
        name: WEB
      - id: 20
        name: APP
    # Define trunk port mapping for each switch
    switch_trunks:
      sw1:
        - interface: GigabitEthernet0/1    # to R1
          trunk_vlans: "10,20"
        - interface: GigabitEthernet0/2    # to SW2
          trunk_vlans: "10,20"
        - interface: GigabitEthernet0/3    # to SRX
          trunk_vlans: "10,20"
      sw2:
        - interface: GigabitEthernet0/1    # to SW1
          trunk_vlans: "10,20"
      r1: []    # R1 handled in router_config.yml
  tasks:

    - name: “Create VLANs on each switch”
      cisco.ios.ios_vlan:
        vlan_id: "{{ item.id }}"
        name: "{{ item.name }}"
        state: present
      loop: "{{ vlan_list }}"

    - name: “Configure Trunk Ports (802.1Q) on switches”
      vars:
        port_config: "{{ switch_trunks[inventory_hostname] }}"
      cisco.ios.ios_interface:
        name: "{{ item.interface }}"
        mode: trunk
        trunk_allowed_vlans: "{{ item.trunk_vlans }}"
        state: present
      loop: "{{ port_config }}"

    - name: “Set native VLAN (optional) to VLAN 1 on trunk ports”
      vars:
        port_config: "{{ switch_trunks[inventory_hostname] }}"
      cisco.ios.ios_interface:
        name: "{{ item.interface }}"
        native_vlan: 1
        state: present
      loop: "{{ port_config }}"
---
- name: “Configure Router Interfaces and OSPF on R1”
  hosts: r1
  gather_facts: no
  connection: network_cli
  vars:
    # Physical interfaces and subinterfaces
    router_interfaces:
      - name: GigabitEthernet0/0.10
        encapsulation: dot1q 10
        ip_address: 10.0.10.1
        mask: 255.255.255.0
      - name: GigabitEthernet0/0.20
        encapsulation: dot1q 20
        ip_address: 10.0.20.1
        mask: 255.255.255.0
      # Optional: uplink interface to upstream ISP
      - name: GigabitEthernet0/1
        ip_address: 203.0.113.2
        mask: 255.255.255.252

    ospf_process: 1
    ospf_networks:
      - network: 10.0.10.0
        wildcard: 0.0.0.255
        area: 0
      - network: 10.0.20.0
        wildcard: 0.0.0.255
        area: 0
      - network: 203.0.113.0
        wildcard: 0.0.0.3
        area: 0

  tasks:

    - name: “Ensure Router Subinterfaces Exist and Have IPs”
      cisco.ios.ios_config:
        lines:
          - "interface {{ item.name }}"
          - "encapsulation {{ item.encapsulation }}"
          - "ip address {{ item.ip_address }} {{ item.mask }}"
        parents: []
      loop: "{{ router_interfaces }}"

    - name: “Enable OSPF and Advertise Networks”
      cisco.ios.ios_config:
        parents: [router ospf {{ ospf_process }}]
        lines:
          - "network {{ item.network }} {{ item.wildcard }} area {{ item.area }}"
      loop: "{{ ospf_networks }}"
## templates/srx_firewall.j2

## Define interfaces with VLAN subunits
interfaces {
  ge-0/0/0 {
    unit 0 {
      family ethernet-switching;
    }
    unit 10 {
      vlan-id 10;
      family inet {
        address 10.0.10.254/24;
      }
    }
    unit 20 {
      vlan-id 20;
      family inet {
        address 10.0.20.254/24;
      }
    }
  }
}

## Define security zones
security {
  zones {
    security-zone VLAN10 {
      interfaces {
        ge-0/0/0.10;
      }
    }
    security-zone VLAN20 {
      interfaces {
        ge-0/0/0.20;
      }
    }
    security-zone untrust {
      interfaces {
        ge-0/0/1.0;   ## uplink to R1 or upstream
      }
    }
  }

  ## Policies: allow VLAN10 → untrust and VLAN20 → untrust
  policies {
    from-zone VLAN10 to-zone untrust {
      policy permit-vlan10-out {
        match {
          source-address any;
          destination-address any;
          application any;
        }
        then {
          permit;
        }
      }
    }
    from-zone VLAN20 to-zone untrust {
      policy permit-vlan20-out {
        match {
          source-address any;
          destination-address any;
          application any;
        }
        then {
          permit;
        }
      }
    }
    ## Optional: allow VLAN10 <-> VLAN20 (inter-VLAN) if needed
    from-zone VLAN10 to-zone VLAN20 {
      policy vlan10-to-vlan20 {
        match {
          source-address any;
          destination-address any;
          application any;
        }
        then {
          permit;
        }
      }
    }
  }
}
#sample output
PLAY [Configure Juniper SRX Firewall for VLAN Zones] **************************

TASK [Load SRX configuration from template] ************************************
changed: [srx]

PLAY RECAP *********************************************************************
srx                        : ok=1    changed=1    unreachable=0    failed=0

real    0m7.125s
user    0m0.176s
sys     0m0.036s

