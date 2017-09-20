---
layout: post
title: Local templating in Ansible
categories: ansible testing
---

Using [delegate_to](http://docs.ansible.com/ansible/latest/playbooks_delegation.html#delegation) 
with your templates you can gain even more confidence in your production configurations, 
before sending them off into the real world.

In a [checking Ansible vars]({% post_url 2017-08-29-checking-ansible-vars %}), 
I described some ways to gain greater confidence in the comprehensiveness of your 
production definitions in Ansible, without actually having to connect to the production estate.

