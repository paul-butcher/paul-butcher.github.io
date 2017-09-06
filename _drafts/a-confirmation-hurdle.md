---
layout: post
title: A Confirmation Hurdle
categories: ansible
---


```yaml
### Request confirmation
# As a safeguard against accidentally running a playbook against production,
# This play pauses for confirmation if the play is being run without any tags.

- name: request confirmation to continue
  hosts: all
  any_errors_fatal: true
  gather_facts: no
  tags: ["untagged"]

  tasks:
  - name: Wait for user confirmation if this play is about to do something dangerous
    pause:
       prompt: "You have not specified any tags.
       This is going to change the state of the
       following hosts:\n
       {{ansible_play_hosts| join(',\n ')}}.\n
       Are you sure you want to continue?"
    when:
    - not jfdi | default(False)| bool
    - '"production" in hostvars[inventory_hostname]["inventory_dir"]'

  - name: Tell the user that they have chosen to continue
    debug:
      msg: "On your head be it!"
    when:
    - '"production" in hostvars[inventory_hostname]["inventory_dir"]'
    
```