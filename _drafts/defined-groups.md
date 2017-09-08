---
layout: post
title: Defined Groups
---

A mistake that may occur in an Ansible codebase is the omission of necessary groups.  A playbook can run completely successfully, but the applications still do not work, because (for example), there is no database server.  It would be helpful to guard against running playbooks against such deficient inventories.

A play, running locally, with a task to compare the groups in the inventory against a list of required groups, could act as such a guard.