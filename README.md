# Automating FHRP with Ansible

![](./images/auto_bgp.PNG)

## Purpose

I was recently going through the IP services chapter of the CCNP ENCOR book and it really sparked my interested in automating HSRP and VRRP configurations. Ive taken the previous CCNP exams and I saw this as a way to stay fresh and keep on labbing as they say.

In this exercise I will look into automating HSRP (Cisco) and VRRP. These two protocols can be used to add redundancy to first hop (gateway) addresses. From a protocol standpoint and configuration wise they are very similar. Something to call out is that VRRP has preemption enabled by default and some terms are different. Active/Standby for HSRP and Master/Backup for VRRP.

## Prerequisites

1. Ansible ≤ 2.9
2. Lab to automate the things
3. Validate Ansible control machine has access to nodes

I have a few basic configurations done in this lab. For example, my Ansible control machine has access to the two nodes acting as distribution switches. I have enabled a tracking command for our lab (DS1 --> IOSv RTR). This track is just checking the Up/Down status of the interface connected to our upstream router. `track 1 interface GigabitEthernet0/2 line-protocol`

## Project Structure

```
juliopdx@librenms:~/repos$ tree ansible_fhrp/
ansible_fhrp/
├── ansible.cfg
├── fhrp.yaml
├── hosts
├── host_vars
│   ├── DS1.yaml
│   └── DS2.yaml
├── README.md
└── templates
    └── fhrp.j2
```

As you can see I didnt go all in with roles on this one. Good example of using a barebones structure to automate a simple configuration. For larger projects or for scale I would highly recommend using roles!

### `hosts`

```
locahost

[network]
DS1 ansible_host=192.168.10.101
DS2 ansible_host=192.168.10.102

[network:vars]
ansible_user=cisco
ansible_ssh_pass=cisco
ansible_network_os=ios
```

### `ansible.cfg`

```
[defaults]
host_key_checking=False
inventory=./hosts
deprecation_warnings=false
pipelining = True
stdout_callback = yaml
[ssh_connection]
pipelining = True
```

## Design Considerations

For this practice run Im using a few caveats. DS1 will be our main layer 3 switch for the SVIs. Whether its VRRP or HSRP. DS1 will be the active device responding as the gateway. The tracking interface mentioned at the start of this file is only configured on DS1. If DS1s upstream connection has issues, its priority will decrement and DS2 will become acive. Preemption will be enabled on all SVIs. When DS1 recovers it will take over as active again.  

### `host_vars/DS1.yaml`

```yaml
vlans:

  - name: 10
    l3: True
    ip_address: 10.0.10.2/24
    hsrp: True
    standby_ip: 10.0.10.1
    priority: 101
    track:
      number: 1 
      decrement: 50

  - name: 20
    l3: True
    ip_address: 10.0.20.2/24
    vrrp: True
    standby_ip: 10.0.20.1
    priority: 101
    track:
      number: 1 
      decrement: 50

  - name: 30

```

Here we have a few VLANs defined. I tried to keep the variables short and sweet. If L3 is defined do this or define that. Notice that `VLAN 30` has pretty much nothing defined. This will make sense in the jinja template and configuration file that is created. Long story short it will just be a simple layer 2 VLAN added to the device. Check out `DS2` variables below and notice there is no tracking defined and priority is set to 99 on all vlans (lower than `DS1` at 101). Oh one more bonus, in VRRP, preemption is enabled by default so we can omit that from the jinja template logic.

### `templates/fhrp.j2`

```jinja
{% for vlan in vlans %}
vlan {{ vlan.name }}
{% if vlan.l3 is defined %}
interface vlan{{ vlan.name }}
  ip address {{ vlan.ip_address | ipaddr('address') }} {{ vlan.ip_address | ipaddr('netmask') }}
  no shutdown
{% if vlan.hsrp is defined %}
  standby {{ vlan.name }} ip {{ vlan.standby_ip }}
  standby {{ vlan.name }} priority {{ vlan.priority }}
  standby 10 preempt
{% if vlan.track is defined %}
  standby {{ vlan.name }} track {{ vlan.track.number }} decrement {{ vlan.track.decrement }}
{% else %}
{% endif %}
{% endif %}
{% if vlan.vrrp is defined %}
  vrrp {{ vlan.name }} ip {{ vlan.standby_ip }}
  vrrp {{ vlan.name }} priority {{ vlan.priority }}
{% if vlan.track is defined %}
  vrrp {{ vlan.name }} track {{ vlan.track.number }} decrement {{ vlan.track.decrement }}
{% else %}
{% endif %}
{% endif %}
{% endif %}
{% endfor %}

```

If you are new to jinja I can see how this may look a bit crazy. We are using a loop at the start to go over our list of VLANs defined in our `host_vars` file for each host. After that we are using a bunch of `if` logic to either do something with the data or just skip over it. For example if I have layer 3 information defined, make an SVI. If I dont then just leave it as is and just make a standard VLAN with no SVI. I will include the output of the file that is generated in the templates folder.

## restart_switches

Restarts switches... and waits :) 