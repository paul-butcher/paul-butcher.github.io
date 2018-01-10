---
layout: post
title: non-conformant hostnames
categories: ansible testing
---

[Verifying ansible inventories]({% post_url 2017-09-05-verifying-ansible-inventories %}), shows how 
to use the match filter to ensure that your hosts conform to the naming convention expected of them.
However, not all hosts in your inventory will necessarily conform to the same convention.

This isn't a problem, as you can simply define a different hostname_pattern for those hosts in their host_vars files.
However, if there are more than one, then it may be better to list out the non-conformists.
Doing so in a separate list in a common group_vars file (such as all) helps with the double-entry-bookkeeping 
aspect of testing, ensuring that you have to make the same mistake twice for it to slip through the net.  It also means that you have a single list, in one place, of all known acceptable violations, even if they are spread across multiple inventories.

To acheive this, the 'fail if this is a bad host' task from [Verifying ansible inventories]({% post_url 2017-09-05-verifying-ansible-inventories %}) can be extended, thus:

```
- name: fail if this is a bad host
  fail: 
    msg: This is a bad host
  when: (not inventory_hostname | match (hostname_pattern)) and hostname_pattern not in verbatim_hostnames
```