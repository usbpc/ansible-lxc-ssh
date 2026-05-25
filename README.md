# ansible-lxc-ssh
Ansible connection plugin using ssh + lxc-attach

![GitHub Workflow Status](https://github.com/andreasscherbaum/ansible-lxc-ssh/actions/workflows/test.yml/badge.svg)
![GitHub Workflow Status](https://github.com/andreasscherbaum/ansible-lxc-ssh/actions/workflows/black.yml/badge.svg)

[![GitHub Open Issues](https://img.shields.io/github/issues/andreasscherbaum/ansible-lxc-ssh.svg)](https://github.com/andreasscherbaum/ansible-lxc-ssh/issues)
[![GitHub Stars](https://img.shields.io/github/stars/andreasscherbaum/ansible-lxc-ssh.svg)](https://github.com/andreasscherbaum/ansible-lxc-ssh)
[![GitHub Forks](https://img.shields.io/github/forks/andreasscherbaum/ansible-lxc-ssh.svg)](https://github.com/andreasscherbaum/ansible-lxc-ssh)

## Description

This plugin allows to use Ansible on a remote server hosting LXC containers,
without having to install SSH servers in each LXC container.

The plugin connects to the host using SSH, then uses `lxc` or `lxc-attach` to enter the
container.

For LXC version 1 this means the SSH connection must login as `root`, otherwise
`lxc-attach` will fail.

For LXC version 2 this means that the user must either login as `root` or must be
in the `lxc` group in order to execute the `lxc` command.

If you are looking for Proxmox support, there's a fork: [ansible-pct-ssh](https://github.com/jbrubake/ansible-pct-ssh):

## Configuration

Add to `ansible.cfg`:
```
[defaults]
connection_plugins = /path/to/connection_plugins/lxc_ssh
```

Then, modify your `hosts` file to use the `lxc_ssh` transport:
```
container ansible_host=server ansible_connection=lxc_ssh lxc_host=container
```

`lxc_container=container` also works for setting the LXC container name.

## Fork

This is a fork from the forked plugin plugin:

[ansible-lxc-ssh by Andreas Scherbaum](https://github.com/andreasscherbaum/ansible-lxc-ssh)

This fork adds the option `host_become_method`. It is inspired by the 
[`proxmox_become_method` parameter from proxmox_pct_remote connection plugin](https://docs.ansible.com/projects/ansible/latest/collections/community/proxmox/proxmox_pct_remote_connection.html#parameter-proxmox_become_method).

The default for `host_become_method` is an empty string and dosen't change the behaviour
of the plugin at all. Setting it to `sudo` prefixes all `lxc-attach` commands with `sudo`.

For it to work the user ansible connects with needs to have password less access 
to execute `sudo lxc-attach`. Assuming the connection user is `ansible` this can
be configured using the following task:

```yaml
- name: Add sudoers entry for ansible user
  ansible.builtin.copy:
    content: 'ansible ALL = (root) NOPASSWD: /usr/bin/lxc-attach'
    dest: /etc/sudoers.d/ansible_lxc
    owner: root
    group: root
    mode: '0440'

```

## How to create a container

The following is an extract from a Playbook which creates a container. First the hosts.cfg:

```
[containers]
web ansible_host=physical.host lxc_host=web host_become_method=sudo
```

The Playbook:

```
# deploy the container
- hosts: containers
  become: yes
  # the container is not up, nothing to gather here
  gather_facts: False
  # files on the host system are changed,
  # creating multiple containers in parallel might cause a race condition
  serial: 1

  tasks:
  - name: Create LXD Container
    become: True
    lxd_container:
      name: "{{ inventory_name }}"
      state: started
      source:
        type: image
        mode: pull
        server: https://cloud-images.ubuntu.com/releases
        protocol: simplestreams
        alias: 16.10/amd64
      profiles: ['default']
      wait_for_ipv4_addresses: true
      timeout: 600
    register: container_setup
    delegate_to: "{{ ansible_host }}"
    #delegate_facts: True
```

The actual container creation is redirected to the `ansible_host`, also fact gathering is turned off because the container is not yet live. It might be a good idea to create the containers one by one, hence the serialization. In my case I also setup ssh access and hostname resolution during the container setup - this does not work well when run in parallel for multiple containers.
