# karri
A devop tool which will execute a sequence of steps. When re-run it 
remembers which steps have already run and skips them. It will update or rollback steps  as requested.

# Project Status

Currently Karri is an idea which is in design phase.

# Background

This tool is inspired by Liquibase which is a popular tool for managing schema changes in SQL databases. Unlike Liquibase, Karri is general - it executes scripts which can change anything, not just databases.

# Introduction

## Rungs

* are expected to be built in an atomic way, ie either the rung 'up' is fully complete or it fails. Same for 'down'.
* if a rung up fails, the execution of the playbook stops
* rungs have
** up - commands to perform to update a system or components or perform some jobs
** down - commands to reverse the effect of the 'up' 
*** the down cannot be run beofre the up - Karri ensures this via information in the state file
*** the sequence 'up' 'down' must be idempotent - that is the up must be able to be re-run after the down (the external system state may not be the same though). The 'up'-'down-'up' sequence shold be used for testing a playbook to ensure the down works, and that they are idempotent together. 

[TODO]

Notes:

Unlike tools like Ansible or Terraform, there is no library of providers/modules which wrap infrastructure or operating system resources. These may emerge and be wrapped in Karri macros for syntactic ease. However the developer can use their existing knowledge of tool sets such as CLIs, APIs within the Karri framework.

# Command-line

* karri status - prints a report about which tasks are yet to be run
* karri up - executes the rungs in the playbook forward
* karri down - undoes the changes by running the rungs' rollback steps in reverse order

# Playbook File Syntax

The playbooks are written in YAML (and possibly JSON). The `goyamp` system (or variant of) will be used and provides:

* variables
* include of source files
* reusable macros
* loading of data - possibly configuration
* Lua embedding

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
    tags: grub
    up:
      sudo: |
        set -euo pipefail
        sed -i_bu -e '/GRUB_GFXMODE=/c\GRUB_GFXMODE=1024x768' \
                -e '/GRUB_INIT_TUNE=/c\GRUB_INIT_TUNE="410 668 1 668 1 0 1 668 1 0 1 522 1 668 1 0 1 784 2 0 2 392 2"' \
                -e '/GRUB_TIMEOUT=/c\GRUB_TIMEOUT=10' /etc/default/grub
        if [ diff /etc/default/grub /etc/default/grub_bu ] ; then
          update-grub
        fi
      changed_if_not:
        shell: diff /etc/default/grub /etc/default/grub_bu
    down: 
      sudo: |
        if [ diff /etc/default/grub /etc/default/grub_bu ] ; then
          cp /etc/default/grub_bu /etc/default/grub
          update-grub
        fi
      changed_if_not:
        shell: diff /etc/default/grub /etc/default/grub_bu
```

# Rung Transaction Control

In addition to the default behaviour of executing the `up` tasks of rungs, there is also

### When the source code changes

When the code in the `up` clause is read, Karri computes a checksum of the YAML string. This is saved in the state file. If later Karri is run it can detect if the source code been has changed by the authors. The following boolean flags can be set in the tasks:
* run_on_change: (default = true)
* run_always:  (default = false)
* error_on_change: (default = false) 

### Rung Transaction Results

The outcomes of a rung can be:

* changed (boolean) if the rung up succeeded
* failed (boolean)

### The outcome can be explicity set by the rung

* changed_if: 
* changed_if_not:

### Conditions

The `when` or `unless` clause specifies a logical condition in Karri which determines if the rung is to be executed. 
The result of a previous step can be used in a condition, eg
```
  when: grub.changed
```
or if a recovery rung is needed:
```
  when: grub.failed
```


# State Files

The status of a playbook is stored by default in a JSON file in the local file system. [Or maybe a SQLite database]. The following is stored for every rung in every execution

* Rung name
* Rung 'up' tasks source code checksums
* Start & End Date and Time
* Rung outcome changed/success

This information is used by Karri to skip rungs which have already been done, or in the case of rollback to decide which rungs need to be undone.


# The Name
There is a giant tree in Western Australia called the [Gloucester Tree](https://en.wikipedia.org/wiki/Gloucester_Tree). It's 190 feet tall and has a ladder made of steel sikes to the top. ![Gloucester Tree](Gloucester.jpeg "The Gloucester Tree") Each rung in the ladder is like steps in a sequence. The tree is of species Karri (Eucalyptus diversicolor). 

The default branch for this repository is `trunk`, naturally.
