# karri
A devop tool which will execute a sequence of steps. When re-run it 
remembers which steps have already run and skips them. It will update or rollback steps  as requested.

# Project Status

Currently Karri is an idea which is in design phase.

# Background

This tool is inspired by Liquibase which is a popular tool for managing schema changes in SQL databases. Unlike Liquibase, Karri is general - it executes scripts which can change anything, not just databases.

# Introduction

TODO

# Command-line

* karri status - prints a report about which tasks are yet to be run
* karri up - executes the rungs in the playbook forward
* karri down - undoes the changes by running the rungs' rollback steps in reverse order

# Playbook File Syntax

The playbooks are written in YAML (and possibly JSON). 

## Example playbook

```
#
# Comments
---
- name: Install my workstation stuff
  define:
    $theuser: birchb
    $theipaddress: 192.168.0.140/24    
    $dns_servers: 8.8.8.8,8.8.4.4      
    $gateway: 192.168.0.1             

  rungs:
  - name: grub
    description: Configure Grub boot
    # refer to https://breadmaker.github.io/grub-tune-tester/ , https://gist.github.com/ArtBIT/cfb030c0791b42330381acce33f82ca0
    tags: grub
    block:  
    - name: config
      description: Edit Grub Config
      up:
        sudo: |
          set -euo pipefail
          sed -i_bu -e '/GRUB_GFXMODE=/c\GRUB_GFXMODE=1024x768' \
                  -e '/GRUB_INIT_TUNE=/c\GRUB_INIT_TUNE="410 668 1 668 1 0 1 668 1 0 1 522 1 668 1 0 1 784 2 0 2 392 2"' \
                  -e '/GRUB_TIMEOUT=/c\GRUB_TIMEOUT=10' /etc/default/grub
      down: 
        sudo: |
          cp /etc/default/grub_bu /etc/default/grub
      
    - name: update
      description: Update grub boot
      when: grub.config.changed
      sudo: update-grub

      
    - name: update
      description: Update grub boot
      when: grub.config.changed
      sudo: update-grub

  - name: Download Hosts file
    get_url:
      url: http://192.168.0.130/etc/hosts # TODO externalise 192.168.0.130 is promus
      force: yes
      dest: /etc/hosts
      owner: root
      group: root
      mode: u=rw,g=r,o=r

  - name: Add '{{ ansible_facts['nodename'] }}' to hosts
    lineinfile:
      path: /etc/hosts
      regexp: '{{ item.regex }}'
      line: '{{ item.line }}' 
    with_items:
      - regex: '127.0.0.1' 
        line: 127.0.0.1 localhost {{ ansible_facts['nodename'] }}
      - regex: '127.0.1.1'
        line: 127.0.1.1 {{ ansible_facts['nodename'] }}

  - name: Remove unwanted packages
    apt:
      state: absent
      autoremove: true
      name:
      - hv3
      - orca # conflicts with orca the sequencer

  - name: System admin tools
    apt:
      state: latest
      update_cache: true
      name:
      - apt-file
      - aptitude
      - dnsutils 			# dig, nslookup
      - edac-utils			# edac-util
      - gnome-disk-utility 	# gnome-disks
      - gparted
      - gpg
      - htop
      - hwinfo
      - ncdu
      - rsync
      - sudo
      - tree
      
  - name: Static IP Address # https://michlstechblog.info/blog/linux-set-a-static-fixed-ip-with-network-manager-cli/
    ansible.builtin.shell: |
      set -euo pipefail
      nmcli
      (cd /etc/NetworkManager/system-connections ; awk '{print FILENAME":",$0}' * )
      interface="$(nmcli -t con show | awk -F : '/ethernet/{print $NF}')"
      conn="$(nmcli -t con show | awk -F : '/ethernet/{print $1}' )"
      nmcli con down "$conn"
      nmcli con mod "$conn" \
        ipv4.addresses  {{theipaddress}} \
        ipv4.gateway "{{gateway}}" \
        ipv4.dns "{{dns_servers}}" \
        ipv4.method "manual"
      nmcli con up "$conn"
    args:
      executable: /bin/bash
  

  - name: check output
    ansible.builtin.shell: |
      nmcli
  
  - name: Host-specific firmware #TODO from inventory
    apt:
      state: latest
      update_cache: true
      name:
      - firmware-amd-graphics

  - name: Adding '{{ theuser }}' to sudo group
    user:
      name: '{{ theuser }}'
      groups: sudo
      append: yes

  - name: Allow '{{ theuser }}' to have passwordless sudo
    lineinfile:
      dest: /etc/sudoers
      state: present
      regexp: '^{{theuser}}'
      line: '{{theuser}} ALL=(ALL) NOPASSWD: ALL'
      validate: visudo -cf %s

  - name: Security
    include_tasks: security.yaml
    tags: security

  - name: Docker
    include_tasks: docker.yaml
    tags: docker

  - name: qemu
    include_tasks: qemu.yaml
    tags: qemu

  - name: Internet Tools
    include_tasks: internet.yaml
    tags: internet

  - name: Development Tools
    include_tasks: development.yaml
    tags: development

  - name: Genyris
    include_tasks: genyris.yaml
    tags: genyris

  - name: Goland
    include_tasks: goland.yaml
    tags: goland

  - name: Games
    include_tasks: games.yaml
    tags: games
```


# The Name
There is a giant tree in Western Australia called the [Gloucester Tree](https://en.wikipedia.org/wiki/Gloucester_Tree). It's 190 feet tall and has a ladder made of steel sikes to the top. ![Gloucester Tree](Gloucester.jpeg "The Gloucester Tree") Each rung in the ladder is like steps in a sequence. The tree is of species Karri (Eucalyptus diversicolor). 

The default branch for this repository is `trunk`, naturally.
